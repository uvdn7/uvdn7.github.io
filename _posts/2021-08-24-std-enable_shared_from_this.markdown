---
layout: post
title: When would someone use std::enable_shared_from_this
date: '2021-08-24 03:29:51'
tags:
- cpp
---

`std::enable_shared_from_this` allows a class to have a valid `shared_ptr` of `this`. Simply adding a member function that returns `shared_ptr<T>(this)` is susceptible to double-free. But when would you use such a feature? It's a relevant question because, next time when you see `std::enable_shared_from_this` in other people's class, you would have a pretty good idea of what they are trying to do.

Double free itself is not the main problem that motivates `std::enable_shared_from_this`. One can always have a public static method for creating instances of `T`, and enforces that the external world only interacts with the object via a single `shared_ptr` control block. The problem is that sometimes, inside a member function of the object, we need to create a `shared_ptr` sharing the same existing control block. The most common pattern is that a class registers some kind of callback, as a member function, that later when triggered, it would require `this` to be alive. Practically, this can happen when you dispatch some work to an executor, or register some handler.
