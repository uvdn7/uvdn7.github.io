---
layout: post
title: Learning C++ Memory Model from a Distributed System's Perspective
date: '2021-10-23 23:32:52'
---

_If C++ standard were reworded using distributed system terms, it might be more readable._

## Your single machine is actually a distributed system in disguise

Multiple cachelines inside your multi-core machine make a distributed system. Cacheline coherence is 100% a distributed system problem. C++ tries to provide a high level abstraction for writing software that runs on a distributed system via [std::memory\_order](https://en.cppreference.com/w/cpp/atomic/memory_order). It makes a lot of sense to expose ordering as an API, because if there's a single key word in distributed system, it would be _ordering_.

This abstraction makes so much sense that rust just [copied it word for word](https://doc.rust-lang.org/nomicon/atomics.html), even though it's pretty convoluted. Just to give you an idea about how convoluted it is, there are _ **many** types of happens-before_ in C++20 standard – sequenced-before, dependency-ordered-before, inter-thread happens-before, happens-before, simply happens-before, strongly happens-before, and coherence-ordered-before. Happens-before relationship _defines_ a partial order. So given these aforementioned happens-before relationships, there are a few types of orderings – release-consume ordering, release-acquire ordering, sequentially-consistent ordering, etc..

In this post, we will try to make sense of C++'s memory model by looking at it as a distributed database.

## std::memory\_order

There're

- memory\_order\_relaxed
- memory\_order\_consume 
- memory\_order\_acquire
- memory\_order\_release
- memory\_order\_acq\_rel
- memory\_order\_seq\_cst

Notice that these settings mostly are _not_ describing the requirements on the atomic operation you are performing. They are mostly for non-atomic memory accesses _around_ the atomic operation (except `memory_order_seq_cst`). All accesses (reads and writes) on a single `std::atomic` are consistent with the modification order; notice not the evaluation order.

I am going to assume `memory_order_consume` and consume operation do not exist in the standard (they are discouraged by the standard anyway). That would make things a little bit more manageable.

## Happens-before

The `happens-before` relationship defined in the standard is actually quite simple (if you compile the formal definitions into common sense). It's a partial order that's consistent with

- [evaluation order](https://en.cppreference.com/w/cpp/language/eval_order) (e.g. A is evaluated before, a.k.a sequenced-before, B) – a static relationship
- dependency order (e.g. A [synchronizes-with](https://preshing.com/20130823/the-synchronizes-with-relation/) B) – a runtime relationship

## synchronizes-with

`memory_order_relaxed` is not a synchronization operation. E.g. in the following [example](https://en.cppreference.com/w/cpp/atomic/memory_order#Sequentially-consistent_ordering),

<!--kg-card-begin: markdown-->

    // Thread 1:
    r1 = y.load(std::memory_order_relaxed); // A
    x.store(r1, std::memory_order_relaxed); // B
    // Thread 2:
    r2 = x.load(std::memory_order_relaxed); // C 
    y.store(42, std::memory_order_relaxed); // D

<!--kg-card-end: markdown-->

, it is allowed to produce `r1 == r2 == 42`. A is sequenced-before B; C is sequenced-before D accordingly to the evaluation order – a static relationship. During program execution, D can precede A and B can precede C in modification order – a runtime relationship. This can happen because there's no synchronization for `memory_order_relaxed`. To put it another way, `synchronizes-with` **brings two partial orderings, static order (evaluation order) and runtime order, together**.

A thread can write to an atomic with relaxed semantic, another thread trying to read it can see either the state before the write or after the write. This is basically eventual-consistency in an asynchronously replicated distributed system. The write from thread A needs to _propagate_ in a sense to the other cachelines. &nbsp;

All other operations establish synchronizes-with relationships. One of the most useful one is the release-acquire ordering.

> If an atomic store in thread A is tagged memory\_order\_release and an atomic load in thread B from the same variable is tagged memory\_order\_acquire, all memory writes (non-atomic and relaxed atomic) that _happened-before_ the atomic store from the point of view of thread A, become _visible side-effects_ in thread B. That is, once the atomic load is completed, thread B is guaranteed to see everything thread A wrote to memory.

Think of it as a mutex.

Now if you always write to an atomic using release operation, and read from it using acquire operation, you will always see the _latest_ value. From a distributed k/v store's perspective, you are getting the same effect as always doing critical read (or reading from the primary) – while the actual implementation can differ.

## `memory_order_seq_cst`

This is the strongest and also the default memory order. It orders memory the same way as release/acquire. Besides, if you apply `memory_order_seq_cst` to multiple `std::atomic`s, there exists a total ordering that's consistent with the modification order of all the atomics involved. Basically, you get linearizability across _all_ these atomics, _if you stick with `memory_order_seq_cst`_. Within this subset of atomics, you have global linearizability – think of all writes and reads are going to a single MySQL primary instance, unlike acquire/release that reads and writes just need to go to the primary DB that owns its key.

## 
