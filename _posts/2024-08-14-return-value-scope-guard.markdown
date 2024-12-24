---
layout: post
title: Access a function's return value in C++ scope guard
date: '2024-08-14 19:29:17'
tags:
- cpp
- coroutine
---

It is best to _avoid_ accessing a function's return value in a scope guard or we need to be really careful about the return value's lifetime.

### Happy case

Scope guard is a simple RAII concept in C++ that, in most cases, just works.

    struct ScopeGuard {
        ScopeGuard(std::vector<int>& v): v_(v) {}
        ~ScopeGuard() {
            std::cout << "scope guard destructed with vector size: " << v_.size() << std::endl;
        }
        std::vector<int>& v_;
    };
    
    void foo() {
        std::vector<int> v{1,2,3};
        auto g = ScopeGuard(v);
    }

We get "vector size: 3". Everything is well. However, when the scope guard starts accessing the return-value of the function, things become spicy.

    std::vector<int> foo_nrvo() {
        // -fno-elide-constructors
        std::vector<int> v{1,2,3};
        auto g = ScopeGuard(v);
        return v;
    }

With the same scope guard, if we use it to print out the size of a vector, when the vector is also being returned from the function. What do we get from the `std::out` in this case? The correct answer is that it depends.

### It depends

Before we dive into what happens, we need to be clear about one thing - that is the ordering of operations when a function returns. It's well-defined by the [standard](https://en.cppreference.com/w/cpp/language/return) (since C++11), that "the copy-initialization of the result of the function call" happens before "the destruction of local variables". This means the scope guard is destructed _after_ the return value is "copy initialized". It makes sense, because if local variables are destroyed first, we have nothing useful to return in many cases.

Get back to the previous example of what would the scope guard see. We are in the territory of [NRVO](https://en.cppreference.com/w/cpp/language/copy_elision)(named return value optimization) a non-mandatory copy/move elision optimization specified by the language. The compiler has the freedom to elide the copy/move entirely in this case (or not); and it affects what the scope guard sees.

If we run the previous code with g++ with no special compiler options, we will get "vector size: 3". This is because the compiler is eliding the copy entirely; `v` is actually living on the caller's stack frame. As a result, it's still alive (with size 3) when the scope guard is destructed later. If we specify `-fno-elide-constructors` to disable the copy-elision optimization, we get "vector size: 0" because `v` gets _moved_ to the variable in the caller's stack frame holding the return value of `foo_nrvo()`, and the scope guard is accessing a _moved_ `v` .

This is not the worst part. In many cases, compiler will have copy elision enabled. It is reasonable to expect NRVO if you know the compiler you are using and the compiler options you are setting. The worst part is that the behavior can change based on if it's a normal function or a coroutine.

### Coroutine and the lack of NRVO

Now we have gotten used to NRVO and expected scope guard to "just work", only to realize that there's no NRVO for coroutine. It is unlikely that it would ever have NRVO for coroutine. This is because the "return" behavior of a coroutine is not specified by the language, but rather left to be implemented by any library that implements the [C++ coroutine TS](https://en.cppreference.com/w/cpp/language/coroutines). Hence it's not reasonable to expect the _language_ to reason about the lifetime of the return value of a coroutine and perform copy elision. Unlike normal functions, where the language can just allocate the return value in the caller's frame, the language has no idea what can be done to the coroutine return value. The coroutine's promise type dictates what happens to the value after `co_return` , via the `return/yield_value` hook – the return value is intercepted.

All this is to say a scope guard would work with normal function and might break when migrated to coroutine – if the scope guard is accessing the return value. And this mental model split will likely continue because it seems unlikely to have NRVO ever supported for coroutine.

