---
layout: post
title: How C++ `typeid` operator works
date: '2021-02-13 17:56:00'
tags:
- cpp
- assembly
---

C++ language include an operator called `typeid` ([https://en.cppreference.com/w/cpp/language/typeid](https://en.cppreference.com/w/cpp/language/typeid)).

> Queries information of a type.  
> Used where the dynamic type of a polymorphic object must be known and for static type identification.

It gives you information about the type of an object, as long as it's available.

    int a = 0;
    std::string tname = typeid(a).name(); // tname == "i"

How it works?

## Static Type

For static types, like the example above, compiler has all the information to know the evaluation result of `typeid(a)`. Let's take at look at the assembly.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2021/02/Screenshot-from-2021-02-13-10-14-57.png" class="kg-image" alt loading="lazy"></figure><!--kg-card-begin: markdown-->

`call 0x555555555160 <_ZNKSt9type_info4nameEv>` is where it calls the `name()` member function of the result `std::type_info` object from `typeid()` evaluation of the expression. We are interested in the `this` pointer and where the `std::type_info` object lives.

`this` is stored in `%rdi` register before a member function call. It's coming from `%rax` which comes from a `%rip` relative address. Now we know the `type_info` object of interest lives at `0x7ffff7f827a0`, the program looks up the address by looking at what's stored in `[rip+0x2eal]`. `%rip` register stores the program counter (`$pc` in GDB). A process's virtual memory layout looks like

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/02/memory-layout.png" class="kg-image" alt loading="lazy"><figcaption>(from https://www.thegeekstuff.com/2012/03/linux-processes-memory-layout/)</figcaption></figure><!--kg-card-begin: markdown-->

Program counter (`%rip`) should always be pointing at the TEXT section of the virtual memory of a running process. So in this case, adding an offset of `0x2eal`, it's now pointing at the section for storing global data.

To summarize, for static types, compiler has global variables (`std::type_info`) initialized. At places where the code calls `typeid`, the machine code just gets the corresponding `std::type_info` global objects that are already initialized. Done.

## Dynamic Type
<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2021/02/Screenshot-from-2021-02-13-09-32-52.png" class="kg-image" alt loading="lazy"></figure><!--kg-card-begin: markdown-->

Here we have an example with dynamic types. Let's check the assembly.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2021/02/Screenshot-from-2021-02-13-10-14-12.png" class="kg-image" alt loading="lazy"></figure><!--kg-card-begin: markdown-->

Similarly, notice the call to `<_ZnKSt9type_info4nameEv>`. We are interested in where the assembly got `%rdi` (`this`) from.

`%rbp` stores the stack base pointer. Accordingly to the calling convention, `[rbp-xxxx]` is how a machine gets locally variables for the current function frame. `[rbp-0x20]` must be getting the only locally variable (`foo` in this case).

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/02/stackframe1.png" class="kg-image" alt loading="lazy"><figcaption>(from https://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86)</figcaption></figure><!--kg-card-begin: markdown-->

Notice `mov rcx, QWORD PTR [rcx]`. It took the first 8 byte of `foo`, and treated it as an address. It's the address to the vtable.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/02/Screenshot-from-2021-02-13-10-07-42.png" class="kg-image" alt loading="lazy"><figcaption>The first 8 byte of foo is the address to class Bar's vtable</figcaption></figure><!--kg-card-begin: markdown-->

Now `%rcx` is the address of Bar's vtable. `[rcx-0x8]` is how it got the address to the `std::type_info` that we are interested in. This is called RTTI (Runtime Type Information, [https://en.wikipedia.org/wiki/Run-time\_type\_information](https://en.wikipedia.org/wiki/Run-time_type_information)). It's kept as a pointer to `std::type_info` in vtable. The overhead is minimum (no more expensive than an additional virtual method).

You can disable RTTI, and compiler will produce an error if you accidentally use RTTI in your code.

    ~/p/how-typeid-op-works ❯❯❯ clang++ -std=c++17 test-typeid.cpp -g -fno-rtti
    test-typeid.cpp:15:10: error: use of typeid requires -frtti
      return typeid(foo).name();
             ^
    

<!--kg-card-end: markdown-->