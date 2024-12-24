---
layout: post
title: When `__cxa_pure_virtual` is just a different flavor of SEGFAULT
date: '2024-07-15 20:09:22'
tags:
- cpp
---

A program crashed during shutdown, with message `pure virtual method called` which came from `__cxa_pure_virtual` – where a pure virtual function points to, in its vtable. Its implementation involves calling `std::terminate` which calls `std::abort` which by default throws a SIGABRT, and crashing the program. Now, why did pure virtual method got called?

This usually happens when someone intends to do polymorphism inside a constructor or destructor.

    class Base {
    public:
        Base() {fn();} // thinking it would be calling the Derived's `fn()`
        // the same happens with dtor
        // virtual ~Base() {fn();}
        virtual void fn() = 0;
    };
    
    class Derived : public Base{
    public:
        void fn() {}
    };
    
    int main() {
        Derived d;
        return 0;
    }

Running this code will lead to `std::terminate` being called\*. Like most object oriented languages, polymorphism is disabled inside the constructor and destructor for sanity. The interesting bit is how C++ disables polymorphism.

Turns out C++ mutates the vptr (vtable pointer) to make it point to the current class' vtable to effectively disable polymorphism – when it's inside the constructor and destructor. This can be verified from gdb with the following simple program.

    class Base {
      public:
        virtual int foo() = 0;
        virtual ~Base() {
          ; // empty statement just for setting breakpoints
        }
    };
    class Derived : public Base {
      public:
        ~Derived() {
          ; // empty statement just for setting breakpoints
        }
        int foo() override {return 42;}
    };
    int main() {
      Derived d;
      return 0;
    }

By setting a breakpoint in `~Base()`, we can see the vptr being mutated to point to the Base class from the Derived.

![gdb_screenshot](/assets/pure_virtual_gdb_screenshot.png)

This means if a Derived object is accessed (e.g. a virtual method being called upon), when it is _being_ destructed, specifically when it has executed its own destructor and it is _executing_ the Base class' destructor (polymorphism disabled), and when the Base class' method happens to be pure virtual, we will get "pure virtual method called".

Combined with the fact that the program only crashes on shutdown, the theory is that it's just a different flavor of the good old SEGFAULT. Except that instead of accessing an object (a piece of memory) that has been freed (or invalid), in this case, it's accessing an object that is **being destructed** – it was partially destructed.

\* The exact behavior appears to be linker dependent. In Clang e.g. the compiler produces a warning and LLD succeeds in linking the program, and produces a runtime error. While GCC produces a compiler warning but ld &nbsp;refuses to link.

