---
layout: post
title: "`folly::Indestructible`"
date: '2021-05-07 23:11:26'
tags:
- cpp
---

`folly::Indestructible<T>` is a class template that makes a `static` variable well, indestructible. Notice that it's meant for `static` variables in the Meyers Singleton pattern. If it's for heap allocated memory, it would just be called memory leak instead of "indestructible".

It boils down to making a destructor of `T` not actually destroy the underlying storage. The trick used by `folly` is to use `union`.

<!--kg-card-begin: markdown-->

    union Storage {
      T value;
      Storage() : value() {}
      ~Storage() {}
    };

<!--kg-card-end: markdown-->

From cppreference [https://en.cppreference.com/w/cpp/language/destructor](https://en.cppreference.com/w/cpp/language/destructor)

> For both user-defined or implicitly-defined destructors, after the body &nbsp;of the destructor is executed, the compiler calls the destructors for &nbsp;all non-static **non-variant members** of the class, in reverse order of &nbsp;declaration, then it calls the destructors of all direct non-virtual &nbsp;base classes

Because `value` is a variant member of the union, it won't be destructed if `Storage s;` is destructed.

