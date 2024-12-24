---
layout: post
title: C++ exception (1) — zero-cost exception handling
date: '2021-11-21 17:06:50'
tags:
- exception
- cpp
---

This is the first post of a series I am making on C++ exceptions.

When people say "C++ exceptions are slow", what they really meant is that throwing an exception in C++ is slow. C++'s exception handling has zero-cost when there are no exceptions. Let's look at a very simple example.

<!--kg-card-begin: markdown-->

    class Foo {
        public:
            ~Foo() {}
    };
    
    class Bar {
        public:
            ~Bar() {}
    };
    
    void func(bool b) {
        Bar bar{}; // <--- Bar::~Bar() should be called on stack unwinding
        if (b) {
            throw 1;
        }
    }
    
    int main() {
        try {
          Foo foo{}; // <--- Foo::~Foo() should be called on stack unwinding
          func(false);
        } catch (...) {
            
        }
        return 0;
    }

<!--kg-card-end: markdown-->

I put `throw 1;` in the code, even though it never throws; so we can see the generated machine code (compiled on clang 13 with -std=c++17 without optimization). Here's the assembly (in Intel syntax), which we will read one piece at a time.

<!--kg-card-begin: markdown-->

    func(bool): # @func(bool)
            push rbp
            mov rbp, rsp
            sub rsp, 32
            mov al, dil
            and al, 1
            mov byte ptr [rbp - 1], al
            test byte ptr [rbp - 1], 1
            je .LBB0_3
            mov edi, 4
            call __cxa_allocate_exception
            mov rdi, rax
            mov dword ptr [rdi], 1
            mov esi, offset typeinfo for int
            xor eax, eax
            mov edx, eax
            call __cxa_throw
            jmp .LBB0_5
            mov rcx, rax
            mov eax, edx
            mov qword ptr [rbp - 16], rcx
            mov dword ptr [rbp - 20], eax
            lea rdi, [rbp - 8]
            call Bar::~Bar() [base object destructor]
            jmp .LBB0_4
    .LBB0_3:
            lea rdi, [rbp - 8]
            call Bar::~Bar() [base object destructor]
            add rsp, 32
            pop rbp
            ret
    .LBB0_4:
            mov rdi, qword ptr [rbp - 16]
            call _Unwind_Resume@PLT
    .LBB0_5:
    Bar::~Bar() [base object destructor]: # @Bar::~Bar() [base object destructor]
            push rbp
            mov rbp, rsp
            mov qword ptr [rbp - 8], rdi
            pop rbp
            ret
    main: # @main
            push rbp
            mov rbp, rsp
            sub rsp, 32
            mov dword ptr [rbp - 4], 0
            xor edi, edi
            call func(bool)
            jmp .LBB2_1
    .LBB2_1:
            lea rdi, [rbp - 8]
            call Foo::~Foo() [base object destructor]
            jmp .LBB2_4
            mov rcx, rax
            mov eax, edx
            mov qword ptr [rbp - 16], rcx
            mov dword ptr [rbp - 20], eax
            lea rdi, [rbp - 8]
            call Foo::~Foo() [base object destructor]
            mov rdi, qword ptr [rbp - 16]
            call __cxa_begin_catch
            call __cxa_end_catch
    .LBB2_4:
            xor eax, eax
            add rsp, 32
            pop rbp
            ret
    Foo::~Foo() [base object destructor]: # @Foo::~Foo() [base object destructor]
            push rbp
            mov rbp, rsp
            mov qword ptr [rbp - 8], rdi
            pop rbp
            ret

<!--kg-card-end: markdown-->
## Zero-cost exception handling

Let's start from the simplest ones.

<!--kg-card-begin: markdown-->

    .LBB0_5:
    Bar::~Bar() [base object destructor]: # @Bar::~Bar() [base object destructor]
            push rbp # store the caller frame pointer on stack
            mov rbp, rsp # set the frame pointer for the callee
            mov qword ptr [rbp - 8], rdi # store `this` pointer on stack (passed in by DRI, dictated by AMD64 ABI)
            pop rbp # restore the frame pointer
            ret

Both `.LBB0_5` and `Bar::~Bar()` symbols store the code for `Bar::~Bar()`. We see the [common prologue and epilogue](https://en.wikipedia.org/wiki/Function_prologue_and_epilogue) on x86.The code is the same for `Foo::~Foo()`. Now let's take a look at `func`.

    func(bool): # @func(bool)
            push rbp
            mov rbp, rsp
            sub rsp, 32
    mov al, dil # dil is the lower 8 bits from RDI(http://www.tortall.net/projects/yasm/manual/html/arch-x86-registers.html) Here, it's passing the `bool` param to AL
            and al, 1
            mov byte ptr [rbp - 1], al
            test byte ptr [rbp - 1], 1
            je .LBB0_3 # jump to .LBB0_3 if `b` is not 1/true
            mov edi, 4
            call __cxa_allocate_exception
            mov rdi, rax
            mov dword ptr [rdi], 1
            mov esi, offset typeinfo for int
            xor eax, eax
            mov edx, eax
            call __cxa_throw
            jmp .LBB0_5
            mov rcx, rax
            mov eax, edx
            mov qword ptr [rbp - 16], rcx
            mov dword ptr [rbp - 20], eax
            lea rdi, [rbp - 8]
            call Bar::~Bar() [base object destructor]
            jmp .LBB0_4

<!--kg-card-end: markdown-->

After the initialization of `bar`, if `b` is `false`, the code will just jump to `.LBB0_3`which calls Bar's destructor, unwinds the stack pointer `rsp` and returns.

<!--kg-card-begin: markdown-->

    .LBB0_3:
            lea rdi, [rbp - 8] # store `this` in rdi (dictated by AMD64 ABI)
            call Bar::~Bar() [base object destructor] # call the dtor
            add rsp, 32 # unwind the stack
            pop rbp # restore caller's frame pointer
            ret # return

<!--kg-card-end: markdown-->

Now let's take a look at `main`.

<!--kg-card-begin: markdown-->

    main: # @main
            push rbp
            mov rbp, rsp
            sub rsp, 32
            mov dword ptr [rbp - 4], 0 # looks useless
            xor edi, edi # set edi to zero (again dictated by AMD64 ABI)
            call func(bool)
            jmp .LBB2_1
    .LBB2_1:
            lea rdi, [rbp - 8] # store `this` foo's addr in rdi
            call Foo::~Foo() [base object destructor]
            jmp .LBB2_4
            ...
            
    .LBB2_4:
            xor eax, eax # set eax (return value) to zero
            add rsp, 32 # unwind the stack
            pop rbp # restore caller's frame pointer
            ret        

<!--kg-card-end: markdown-->

The reason why it uses `xor eax eax` to set a register to 0 is for efficiency reason. It produces shorter opcode and enables the processor to perform register renaming. Notice the `mov dword ptr [rbp - 4], 0` line in main. It initializes the next four bytes on the stack to zero, which we are not really using anyway. If you switch from clang to gcc, this line would disappear (with the same flags).

If you follow the labels from `main`, to `.LBB2_1`, to `.LBB2_4`, all they are doing is just calling `func`, destructing `foo`, and returning 0. The code runs _as if the exception handling logic is not even there_ to begin with, when it's not thrown. This is known as **zero-cost exception handling**. It's an implementation detail for programming languages. E.g. C++ and Java have zero-cost exception handling. [CPython 3.11](https://bugs.python.org/issue40222) is thinking about adding zero-cost exception handling support.

You might be thinking, we are not even handling an exception in this code, of course it's zero-cost. How can we call it exception handling to begin with? Well, we actually did handle the exception, it just didn't throw. We can rewrite the code as,

<!--kg-card-begin: markdown-->

    int foo(bool b) {
      if (b) {return -1;} // exceptional case
      else {return 0;}
    } 
    
    int main() {
      int ret = foo(false);
      if (ret != 0) { // error handling }
    }

<!--kg-card-end: markdown-->

Notice that even if `func` never returns `-1` here, `main` needs to check the return value (handling the exception). A smart compiler can optimize this away in this case. But in practice, `func` will error out _sometimes_. So `main` _always_ needs to check the return value of `func` because it's possible that it needs to handle the error case. This check, a form of exception handling, is not free, even though 99% of the time there are no errors. This is essentially how Python implements its exception handling. It checks if there's an exception PyErr all the time, and invokes the handling logic if needed. CPython's exception handling (before 3.11) is not zero-cost. With C++ exception, on the other hand, it's free (zero-cost), when there are no exceptions (if you discount iTLB misses). There are binary layout optimizers such as [BOLT](https://github.com/facebookincubator/BOLT) that can even further optimize the physical layout of the code to minimize iTLB misses.

This strategy is the equivalent of optimistic concurrency control in database. As you might have guessed, it comes with a cost. When an optimistic prediction is wrong — when an exception is actually thrown — it's much more expensive.

