---
layout: post
title: Atomic Queue
tags:
- cpp
---

This post is about [folly](https://github.com/facebook/folly)'s [AtomicQueue](https://github.com/facebook/folly/blob/main/folly/io/async/AtomicNotificationQueue.h#L194). It is a lock-free MPSC(multiple-producer-single-consumer) queue that only wakes up the consumer when necessary. It is mainly used as the notification queue for `folly::EventBase`, as a way to pass work from external threads to be executed on the EventBase thread. In this context, the EventBase is the consumer, and it always runs on the same thread. There can be many producers requesting work to be done inside the EventBase. The main loop of `folly::EventBase` basically does [two things](https://github.com/facebook/folly/blob/main/folly/io/async/EventBase.cpp#L631-L657)
* Call libevent's `event_base_loop()` and block on `epoll_wait` if there are no pending tasks be executed. If some event handlers are ready, the callbacks are executed inline (in `event_base_loop()`). The callbacks can schedule more work to be done on the EventBase and they are called LoopCallbacks.
* Execute various tasks (drain the notification queue, aka external tasks, and loop callbacks, aka internal tasks).

If the EventBase is blocked on epoll when someone adds a task to the notification queue, we would need to wake up the EventBase thread, otherwise the task will wait indefinitely. The wake up mechanism is done by firing an event that is registered by the notification queue with the EventBase. It makes sense since the only way to wake up a thread waiting on epoll, is to trigger some fd event.

The problem is that we do not want to make a system call for every single task that's enqueued to the notification queue. That's a lot of overhead. Despite of there could be multiple producers pushing tasks to the queue at the same time, we want to wake up (make system calls) as few times as possible. The ideal case is 0 time when the consumer (the EventBase) is not even blocked. If it is blocked, we should wake it up no more than once. This is what `folly::AtomicQueue` does.

## Contract
* The consumer should "arm" the queue (call `arm` in this case) when it's abouto go to sleep/wait/block. `arm()` either returns more items for the consumer to process (and call `arm()` later), or an empty list, in which case the consumer can go to sleep.
* One of the producers, on `push`, is responsible to wake up the consumer. This is achieved by the return value of `push` (`true` means the queue was armed before)

The specific sleep/wake-up mechanism is user defined. The `AtomicQueue` is only facilitating the communication. The implementation is pretty simple actually. It is the classic lock-free implementation of a queue using linked list, using `std::atomic` for the head address. `folly::AtomicQueue` has three state, all represented by the single atomic. `1` means the queue is armed, `0` means the queue is empty, otherwise the queue is non-empty. Notice that the queue can't be armed, and non-empty at the same time. This is achieved by the contract for the `arm()` function. 

At first glance, the contract/interface is pretty odd (returning armed on `push()`, returning items on `arm()`). But actually it makes perfect sense. Since it's trying to solve the problem of only waking up the consumer when needed, the consumer's state (armed or not) must be encoded in the queue (calling `arm()` by consumer) and it must be discoverable by the producers (the return value of `push()`). So that only the first `push()` would wake an armed consumer.

One more thing, since the AtomicQueue doesn't make any assumption of how the consumer will go to sleep and how producer wakes it up. It is important to make sure the wake-up is not lost to the classic race condition
1. consumer calls `arm()` and the queue is armed; but before the consumer actually goes to sleep
2. producer calls `push()` and attempts to wake up the consumer
3. consumer goes to sleep, and misses the wakeup

It is not a problem for EventBase here because epoll will always pick up the event. If synchronization primitives like `folly::Baton` is used, where the consumer waits on `baton.wait();` and the producer performs `baton.post();` it would work as well. 
