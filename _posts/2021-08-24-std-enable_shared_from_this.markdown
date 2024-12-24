---
layout: post
title: When would someone use std::enable_shared_from_this
date: '2021-08-24 03:29:51'
tags:
- cpp
---

`std::enable_shared_from_this` allows a class to have a valid `shared_ptr` of `this`. Simply adding a member function that returns `shared_ptr<T>(this)` is susceptible to double-free. But when would you use such a feature? It's a relevant question because, next time when you see `std::enable_shared_from_this` in other people's class, you would have a pretty good idea of what they are trying to do.

I think it's because the class's member function interacts with other threads (or eventbases) that need `this` to be alive. It's usually a separate _thread_ or _eventbase_ because otherwise when a member function is sending `this` to another object, `this` is obviously alive and can outlive the interaction. So a common scenario is that the class is managing (or co-managing) a thread-pool, or eventbase, e.g. for scheduling async jobs.

