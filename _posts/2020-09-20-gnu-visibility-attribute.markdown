---
layout: post
title: GNU Visibility Attribute
date: '2020-09-20 22:20:47'
tags:
- cpp
---

When going through the [folly](https://github.com/facebook/folly) code base, sometimes, you see definitions like

    class FOLLY_EXPORT OptionalEmptyException : public std::runtime_error {
    ...

`FOLLY_EXPORT` is defined as

    #define FOLLY_EXPORT __attribute__ (( __visibility__ ("default")))

It sets the visibility attribute of symbol `OptionalEmptyException` to "default". What does it do exactly, and how it works?

## Static Library

We will start with a very simply example. We have a file `foo.cpp`,

    int foo(int x) {
      return x*x;
    }
    

And a simple `main.cpp` that calls `foo`. If we compile `main.cpp` and `foo.cpp` and inspect the object files,

    clang++ -c main.cpp foo.cpp
    nm -C main.o
    0000000000000000 T main
                     U foo(int)
    nm -C foo.o
    0000000000000000 T foo(int)
                     

`nm` is a utility that displays symbols from object files. `-C` dismangles the symbols, so it's more readable (`foo(int)` vs `_Z3fooi`).

Symbol `foo` is what we are interested in here. In `main.o`, `foo` is U (undefined). In `foo.o`, `foo` is T (a symbol in .text section and externally visible).

Now let's add a visibility attribute to `foo`.

    int __attribute__ ((visibility("hidden"))) foo(int x) {
      return x*x;
    }
    

    clang++ foo.cpp
    nm -C foo.o
    0000000000000000 T foo(int)

`foo` is still public. If we make a binary out of main.o and foo.o, it will run just fine. This is because `foo.o` is a static library, with static linkage. `foo` is a "public" symbol as we statically link `foo.o` into our final binary.

## Dynamic Library

So the visibility attribute doesn't affect static library and it must affect dynamic library. We first build a shared library,

    clang++ -shared -fpic foo.cpp -o libfoo.so

Now `foo` becomes a local symbol, hence not visible outside of the shared library itself.

    nm -C libfoo.so
    0000000000004020 b completed.0
                     w __cxa_finalize@@GLIBC_2.2.5
    0000000000001020 t deregister_tm_clones
    0000000000001090 t __do_global_dtors_aux
    0000000000003e18 d __do_global_dtors_aux_fini_array_entry
    0000000000004018 d __dso_handle
    0000000000003e20 d _DYNAMIC
    0000000000001100 t _fini
    00000000000010e0 t frame_dummy
    0000000000003e10 d __frame_dummy_init_array_entry
    0000000000002050 r __FRAME_END__
    0000000000004000 d _GLOBAL_OFFSET_TABLE_
                     w __gmon_start__
    0000000000002000 r __GNU_EH_FRAME_HDR
    0000000000001000 t _init
                     w _ITM_deregisterTMCloneTable
                     w _ITM_registerTMCloneTable
    0000000000001050 t register_tm_clones
    0000000000004020 d __TMC_END__
    00000000000010f0 t foo(int) # <--- notice the lower case t
    

Now if we try to link the object files together, clang would complain that it cannot find `foo`.

    clang++ main.cpp libfoo.so
    /usr/bin/ld: /tmp/main-6da6a7.o: in function `main':
    main.cpp:(.text+0x15): undefined reference to `foo(int)'
    clang-10: error: linker command failed with exit code 1 (use -v to see invocation)
    

Usually ` __attribute__ ((visibility("default")))` is used in combination with `-fvisibility=hidden` linker flag. The latter would make "hidden" the default visibility for all symbols of a shared library. Back to the original example from folly,

    class FOLLY_EXPORT OptionalEmptyException : public std::runtime_error {
    ...

Exceptions, if visible outside of the shared library itself, needs to be marked with default visibility explicitly (if `-fvisibility=hidden` is set). People usually set `-fvisibility` to reduce shared library size and load time.

## How Dynamic Library Works

Dynamic library (a.k.a shared library) is lazily loaded at runtime. It has the benefit of sharing the dynamic library memory section across multiple processes. This means a binary that depends on a shared library is incomplete.

    clang++ main.cpp libfoo.so
    objdump -D a.out -t -s -M intel

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/09/Screenshot_2020-09-20_15-13-08.png" class="kg-image" alt loading="lazy"></figure>

This is where foo gets called.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/09/Screenshot_2020-09-20_15-14-14.png" class="kg-image" alt loading="lazy"></figure>

It looks up the Global Offset Table, where it would find out that it needs to load libfoo.so to resolve symbol foo. As expected, string "libfoo.so" is in the final binary.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/09/Screenshot_2020-09-20_15-17-37.png" class="kg-image" alt loading="lazy"></figure><!--kg-card-begin: markdown-->
## Summary

To summarize, ` __attribute__ ((visibility("default")))` is often used in a shared library, and expected to be compiled with `-fvisibility=hidden` (though it doesn't have to). Symbols with "default" visibility will always be public outside of the shared library itself. You would often find it set on public interfaces of a shared library (including exceptions).

<!--kg-card-end: markdown-->