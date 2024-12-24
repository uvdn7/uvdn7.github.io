---
layout: post
title: atomic_thread_fence
date: '2021-04-15 04:38:02'
tags:
- cpp
---

Just like you can have a `std::atomic` synchronizes two threads with each other with release-acquire semantic, you can also have Fence-Fence, Atomic-Fence, Fence-Atomic synchronizations. C++ reference has very detailed documentation about when there exists a valid [synchronizes-with](https://preshing.com/20130823/the-synchronizes-with-relation/) relationship, [https://en.cppreference.com/w/cpp/atomic/atomic\_thread\_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence).

Rust's memory model is a word-to-word copy of C++'s, and it has pretty good documentations e.g. [https://doc.rust-lang.org/std/sync/atomic/fn.fence.html](https://doc.rust-lang.org/std/sync/atomic/fn.fence.html) is a good example of Fence-Fence synchronization.

<!--kg-card-begin: markdown-->

    Thread 1 Thread 2
    
    fence(Release); A --------------
    x.store(3, Relaxed); X --------- |
                                   | |
                                   | |
                                   -------------> Y if x.load(Relaxed) == 3 {
                                        |-------> B fence(Acquire);
                                                         ...
                                                     }
    

<!--kg-card-end: markdown-->

The aforementioned ordering can be established as well with release-store and acquire-load of an `atomic`. Why do we need `atomic_thread_fence` at all? Especially it requires an `atomic` variable to work correctly to begin with?

Basically it's like a batch annotation. `fence(release)` marks _all_ following stores with `release` semantic. The following [quote from C++ reference](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence) is saying effectively the same thing.

> While an atomic store-release operation prevents all preceding writes from moving past the store-release, an `atomic_thread_fence` with `memory_order_release` ordering prevents all preceding writes from moving past all subsequent stores.

