---
layout: post
title: Linear scalable read-write lock
date: '2020-07-01 23:59:40'
tags:
- cpp
- folly
- lock
---

The basic concept of a read-write lock is simple. It allows multiple readers to access the resource simultaneously, but at most one thread can have exclusive ownership of the lock (a.k.a write lock). It's supposed to be an optimization, comparing to simple mutex (e.g. `std::mutex`). As in theory, multiple readers do not contend. Naively, read performance should scale linearly. However, this is not the case in practice for most read-write locks, due to cacheline pingpong-ing.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2020/07/Screen-Shot-2020-07-01-at-2.21.45-PM.png" class="kg-image" alt loading="lazy"><figcaption>CPU cache</figcaption></figure><!--kg-card-begin: markdown-->

Let's say N threads are running on N cores, all trying to get shared ownership of the mutex (read lock). Because we need to keep a count of the number of active readers (to exclude writers), each `lock_shared()` call needs to mutate the mutex state.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/07/Screen-Shot-2020-07-01-at-2.27.11-PM.png" class="kg-image" alt loading="lazy"></figure>

And CPU needs to keep cache coherent. It needs to invalidate cachelines in other cores of the same address.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/07/Screen-Shot-2020-07-01-at-2.29.29-PM.png" class="kg-image" alt loading="lazy"></figure>

Now if Core 1 wants to get the lock in shared mode, it will get a cache miss. An L1 cache hit takes about 4 CPU cycles. Main memory access usually takes 20x more CPU cycles if not more. In Jeff Dean's famous latency table, L1 cache hit takes about 0.5 ns, and main memory access is about 100 ns. In this way, the performance doesn't scale linearly as you throw more readers to the data. By increasing the number of threads, you often end up with 10x power assumption and maybe 20% performance improvment over single reader.

<!--kg-card-begin: markdown-->
## How `folly::SharedMutex` solves the problem

`folly` is Facebook's C++ open source library. [`folly::SharedMutex`](https://github.com/facebook/folly/blob/master/folly/SharedMutex.h) solves this problem by storing active readers in a global static storage. It stripes the shared storage, so each reader takes a whole cacheline. It tries to have a sticky routing from a given core to the slot in the shared memory for reducing search overhead. Each slot stores the address of the mutex. In this way, when another reader comes in, it only needs to mutate the shared storage, and leaves the mutex state unchanged.

Voilà! No more cacheline pingpong-ing!

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2020/07/Screen-Shot-2020-07-01-at-2.47.35-PM.png" class="kg-image" alt loading="lazy"></figure><!--kg-card-begin: markdown-->

However, it does come with a downside. Because _all_ deferred readers share the same block of static memory, this means

> acquisition of any lock in write mode can conflict with acquisition of any lock in shared mode  
> ([https://github.com/facebook/folly/blob/master/folly/SharedMutex.h](https://github.com/facebook/folly/blob/master/folly/SharedMutex.h))

E.g. A writer, of a given mutex, might have most of the deferred reader array pre-fetched in memory, but readers of other mutex might cause the cacheline to be invalidated. And most of the time the write and the read are not even contending on the same lock object. But the cacheline gets invalidated anyway.

The benefit of SharedMutex is most significant when there are many concurrent readers, and there are actual contention. For cases where actual concurrent reads is rare, spin lock works better. That's why `folly::SharedMutex` actually degenerates to an inlined read-write lock (which stores the reader count inline) when there are no concurrent readers detected. This is done dynamically at runtime.

## Details

The devil is in the details.

#### How to block

How do we block when we need to, E.g. reader waiting for an active writer, or writer waiting for multiple outstanding readers? You don't want to call [FUTEX\_WAIT](https://man7.org/linux/man-pages/man2/futex.2.html) every time you block, as system calls are expensive. `folly::SharedMutex` blocks in following order

1. spin with `pause` instruction
2. `std::this_thread::yield()`
3. FUTEX\_WAIT

If there's no contention or the contention is short lived, spinning wait is the fastest, with no system calls. `PAUSE` instruction is used to avoid expensive pipeline flush when exiting the loop. ([https://c9x.me/x86/html/file\_module\_x86\_id\_232.html](https://c9x.me/x86/html/file_module_x86_id_232.html))

If it exhausts the spinning wait tries, it moves on to [sched\_yield()](https://linux.die.net/man/2/sched_yield). It will give the other thread opportunities to release the lock and avoid burning through too many CPU cycles busy waiting.

At the very end, it calls FUTEX\_WAIT if both spin-wait and yield are exhausted. All the states related to a given mutex is encoded in the 32bit futex and the external deferred reader array.

#### How lock/unlock works

All the synchronizations required are done based on a single 32bit atomic futex. A common pattern of how lock works looks like:

1. block until `state` shows no contention with `acquire` semantic
2. if the lock is inlined, perform CAS on the atomic state. Only return, when CAS succeeds (except this is try\_lock, or timeout is allowed).
3. if the lock is not inlined
  - if not already set, perform CAS on the atomic state to block other exclusive operations (e.g. set the "has\_deferred\_reader" bit in the state for `lock_shared()`, if not already set)
  - create deferred reader slot in global static memory
  - read state again with `acquire` semantic
  - if the "has\_deferred\_reader" bit is still set, return success
  - retry the whole operation

Setting the "has\_deferred\_reader" bit will block the acquisition of the lock in exclusive mode. However it's possible that after a deferred reader slot is written, the previous last deferred reader exists the lock and clears the "has\_deferred\_reader" bit. Or even worse, some writer might have even started acquiring the lock in exclusive mode. That's why we need to read the atomic state again to check the "has\_deferred\_reader" before we can return from `lock_shared()`. If we have the deferred reader registered, and "has\_deferred\_reader" is set in the state, it's impossible for another writer to successfully get exclusive ownership of the lock. So we can safely return.

For `lock()` (lock exclusive), it sometimes needs to wait for all the deferred readers to exit the lock. But for deferred readers, we are not keeping a refcount inline anymore. So `folly::SharedMutex` in this case, would move all the deferred readers back inline, and then `FUTEX_WAIT` on the state until the refcount reaches zero.

Unlock just calls `FUTEX_WAKE`. But again, the devil is in the details. Whom do we wake up? How many threads do we wake up?

#### Whom to wake up

If there are multiple writers waiting for the mutex, when the last reader exits, it's most efficient to just wake up one writer instead of all of them. This is done using a special bit from the atomic state. Only the first writer waiter will set this bit, and only this thread will be waken up.

On the other hand, if there are multiple readers waiting for the active writer to exit the lock, all the them will be waken up at once.

#### What if we run out of slots

If no slots are available, readers are inlined. They are refcounted inside the 32bit atomic futex.

## Benchmark

You can run the [benchmark](https://github.com/facebook/folly/blob/master/folly/test/SharedMutexTest.cpp) on your own hardware. On my machine (32KB L1 cache per core, dual-socket, 24 cores with hyperthreading enabled), `folly::SharedMutex` is better than other mutexes in most of the cases, and in some cases significantly faster. When it's slower than other mutexes, it's usually only few ns slower.

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2020/07/Screen-Shot-2020-07-01-at-4.57.26-PM.png" class="kg-image" alt loading="lazy"><figcaption>benchmark</figcaption></figure>