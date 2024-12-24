---
layout: post
title: SeqLock, atomic_thread_fence, std::function altogether
tags:
- cpp
---

I ran into an issue that some code has Undefined Behavior due to data race on non trivial storage (not `std::memcpy`-safe). The context is [https://github.com/facebook/folly/commit/68104e033d8c6cf24c7d8aa52d2b411dd964ac23](https://github.com/facebook/folly/commit/68104e033d8c6cf24c7d8aa52d2b411dd964ac23).

It was caused by the fact that `std::function` is not trivially copyable (`std::is_trivially_copyable`). Naturally, I went on a big tangent to understand

- `std::atomic_thread_fence` and its usage ([http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1478r1.html](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1478r1.html))
- SeqLock
- benign data race
- `std::function` and how it works (Type Erasure)
- what it means to be trivially copyable

## Let's start with `atomic_thread_fence`

Just like you can have two `std::atomic`s synchronize with each other with release-acquire semantic, you can have Fence-Fence, Atomic-Fence, Fence-Atomic synchronizations. C++ reference has very detailed documentation about when there exists a valid synchronize-with relationship, [https://en.cppreference.com/w/cpp/atomic/atomic\_thread\_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence).

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

Basically it's like a batch annotation. `fence(release)` marks all following stores with `release` semantic. The following [quote from C++ reference](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence) is saying effectively the same thing.

> While an atomic store-release operation prevents all preceding writes from moving past the store-release, an `atomic_thread_fence` with `memory_order_release` ordering prevents all preceding writes from moving past all subsequent stores.

## SeqLock

[https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-6.html](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-6.html) has a good intro about the topic.

Writers will hold exclusive locks. But readers would be lock free except that they could fail sometimes. The idea of a Sequence Lock is simple. There will be a single counter that writers would bump every time they update the data. Readers will read the counter before _and_ after the data fetch. If there's both reads return the same value, it means the data hasn't been changed during the data fetch, hence success otherwise, the read operation needs to be retried.

While it's easy to describe the algorithm in English, it requires a lot of attentions to details when translating it to C++ due to CPU and Compiler reordering. First of all, the only reason SeqLock is even useful to begin with is that the data we are interested in, is not an `atomic`. We don't have atomic memory access to the data.

The first section of [http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1478r1.html](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1478r1.html) provides a good C++ example.

<!--kg-card-begin: markdown-->

    do {
      seq1 = seq_no.load(memory_order_acquire);
      data = shared_data;
      atomic_thread_fence(memory_order_acquire);
      int seq2 = seq_no.load(memory_order_relaxed);
    } while (seq1 != seq2 || seq1 & 1);
    use data;

<!--kg-card-end: markdown-->

* * *

Best resource on C++ Type Erasure [https://quuxplusone.github.io/blog/2019/03/18/what-is-type-erasure/](https://quuxplusone.github.io/blog/2019/03/18/what-is-type-erasure/)

