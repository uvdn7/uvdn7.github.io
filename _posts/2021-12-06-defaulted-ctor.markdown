---
layout: post
title: "`defaulted` constructor in C++"
date: '2021-12-06 00:27:59'
tags:
- cpp
---

You have seen `Foo() = default;`. It declares a default constructor (duh!). But what does it really do, and when is it actually useful? "When is it useful" is a very interesting question in my opinion because next time when you see `= default;` in other people's code, you would understand why it's there. It's all about communicating intentions and making code self documented.

First of all, a default constructor is the one that gets called when no arguments are passed in – `T{}`. For class/struct `T`, if you have constructor(s) of any kind defined explicitly, compiler will stay out of your way; you get what you see.

<!--kg-card-begin: markdown-->

    class Foo {
        // as long as there is at least one user-defined ctor
        // compiler will not do anything about ctors (including
        // default ctor)
        Foo(int a) {}
    };

<!--kg-card-end: markdown-->

If no constructor is defined by user, compiler will _declare_ one for you. The invariant is that a_ny_ class have at least one constructor declaration.

<!--kg-card-begin: markdown-->

    class Foo {
      // compiler will declare Foo() for you
    };

<!--kg-card-end: markdown-->

Now `Foo()` is automatically declared; but it's not necessarily _defined_. If `class Foo` has a reference member (e.g. `int& a`), `Foo` obviously can't be default constructed (there's a longer list of rules [here](https://en.cppreference.com/w/cpp/language/default_constructor), if you are in the mood of reading legal-ish documents). Other examples include e.g. what if there's one member that can't be default constructed. In these cases, the compiler will `delete` the default constructor – same as `Foo() = delete;`. If no issues are found, the compiler will define a default constructor for you that is equivalent to an empty function.

This is nice, especially for a templated class that you can't determine ahead of time if `T` can be default constructed or not. This is cool! But what if you defined one constructor? You will loose the implicitly declared constructor from the compiler! That's what `Foo() = default;` is for. You can enforce the compiler to perform the aforementioned implicit thingie while having your own constructors.

