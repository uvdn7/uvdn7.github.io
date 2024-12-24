---
layout: post
title: RCU vs. Hazard Pointer
tags:
- cpp
- operating-system
---

These two concepts are very similar. Both are techniques to provide near linear scalability for readers for a read-often-write-rare workload.

RCU is mostly used in the Linux kernel because it can be implemented very efficiently with specific config and hardware information. E.g. `synchronize_rcu()` is a no-op in `tiny-rcu`. (reference [https://github.com/torvalds/linux/blob/f40ddce8/kernel/rcu/tiny.c#L146](https://github.com/torvalds/linux/blob/f40ddce8/kernel/rcu/tiny.c#L146)). With a kernel with preemption disabled, rcu read-side critical section can be zero-overhead as well.

