---
layout: post
title: API for Assigning Timestamp
date: '2020-05-08 07:06:24'
tags:
- clock
- time
- distributed-system
---

This is a follow-up post of my [previous one]( __GHOST_URL__ /notes-on-the-spanner/) about Spanner. There I talked about how Spanner uses TrueTime API to achieve external consistency at global scale and hide sharding and replication from users.

In this post, I want to discuss different options of assigning timestamps and their usefulness to applications. But first, let's take a look at how can you get a timestamp with bounded uncertainty. You can't reply on software if you want to have an accurate clock. As long as the clock synchronization happens in software, you can't have an accurate bound of the uncertainty. Even if the daemon runs in kernel space, there can be times that memory pressure is high, and the code can be waiting for IO, etc. It's impossible to have a bound on the execution time of the daemon. Notice that we are talking about milliseconds, if not microseconds or nanoseconds. Having a bound of one minute e.g. is not very helpful. That's why [Precision Time Protocol](https://en.wikipedia.org/wiki/Precision_Time_Protocol) (or PTP) requires hardware support, i.e. timestamps are minted by network cards. And PTP clock synchronization is done usually, if not exclusively, in hardware. Once you bypass software, time synchronization, especially various latency estimations are much easier. Hence tighter bound on clock uncertainty.

## Hardware minting timestamps

In PTP, network cards mint timestamps. This is a useful feature for applications. E.g. I might want all packets sent by my application to carry the timestamps when they leave the network card. But when database commits locally, it doesn't involve network card. Can we expand on that, what if flash card, spinning disk can mint timestamps as well? Then we need to keep these hardware's clocks in sync as well. How useful is this? Well, not very useful. The commit log of a shard is usually ordered by the commit timestamp. But we only have a set of time intervals `[earliest, latest]` for each committed entry. And the interval can change as clock sync takes place. How can you order `[10, 20]` and `[12, 15]`? Which one happens first?

## TrueTime style API

Now you have hardware that keeps your clock very accurate, with high confident uncertainty bound, how do you use it? In Spanner, the TrueTime API uses a pull model, where applications ask for `TT.now()` at certain point in time during the execution of the application program. However, there can be a delay between when the kernel gets the TrueTime and when the function returns to user space and the value available to application code. So when the code gets back the `TT.now()` it's always a little stale already. It's `now()` in the past. Well how useful is this then? Turns out, it's very useful, as long as the application code be careful about when it calls `TT.now()`.

Spanner calls `TT.now()` in the following sequence:

    `e:server` -> `TT.now()` -> chose a commit timestamp

All Spanner needs in this case, is just that commit timestamp being larger than `abs(e:server)`. So picking commit timestamp to be `>= TT.now().latest` still works. (This is the start condition in the Spanner paper.)

For commit wait, Spanner doesn't perform the commit (after a timestamp has been chosen) until it’s definitely in the past. That is:

    Commit timestamp chosen -> `TT.now()` -> perform commit

The causality still works here. Even if `TT.now()` value is stale. It doesn’t affect correctness.

Another case when Spanner uses TT, is the read transaction, which is stamped at `TT.now().latest`. As long as this timestamp is picked after the RO transaction starts, it’s all fair game. Let’s say in the extreme case, when TT.now() doesn’t return after it sets the value in kernel for more than 10 minutes. The RO transaction itself will be blocked and running for 10 minutes also.

For external consistency, Spanner only guarantees that if `start:2` \> `commit:1`, `s:2 > s:1` (In English, if the absolute start time of transaction 2 happens after absolute commit time of transaction 1, transaction 2's commit time must be greater than transaction 1's). Same thing applies to reads. In this case, `start:read` is not greater than e.g. a write transaction just committed. So the transaction timestamp for this RO transaction doesn’t have to be greater than the write transaction. Hence, it doesn’t need to reflect/get the latest writes.

## Summary

Even though `TT.now()` value can be stale. But as long as application calls it at the right place, it's still very useful. Causality is your friend. Even though the TrueTime API is very simply. But it's very powerful and elegant in design.

<!--kg-card-end: markdown-->