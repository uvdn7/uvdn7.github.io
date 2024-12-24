---
layout: post
title: Synchronized Multi-Append
date: '2020-08-04 00:35:29'
tags:
- cross-shard-transaction
- hlc
- spanner
---

Hybrid Logical Clock is a nice primitive for keeping ordering. Many databases nowadays use HLC to version incoming writes (appends to binlog/WAL/redo-log). It's usually one monotonically increasing HLC per shard/database. When dealing with cross shard transactions, it would be very nice if we can have the same commit HLC for all participants across multiple databases. This provides benefits like:

- write transactions can be totally ordered globally
- supports snapshot reads cross shards

The challenge is to maintain synchronized HLCs while at the same time keeping it monotonically increasing per shard. Notice that the same challenge applies to systems like Spanner with TrueTime, because we are not concerned about the external consistency aspect of a single shard but synchronization between multiple shards.

## How Spanner solves the problem

> A non-coordinator-participant leader first acquires write locks. It then chooses a prepare timestamp that must be larger than any timestamps it has assigned to previous transactions (to preserve monotonicity), and logs a prepare record through Paxos. Each participant then notifies the coordinator of its prepare timestamp.  
> The coordinator leader also first acquires write locks, but skips the prepare phase. It chooses a timestamp for the entire transaction after hearing from all other participant leaders. The commit timestamps must be greater or equal to all prepare timestamps (to satisfy the constraints discussed in Section 4.1.3), greater than TT.now().latest at the time the coordinator received its commit message, and greater than any timestamps the leader has assigned to previous transactions (again, to preserve monotonicity). The coordinator leader then logs a commit record through Paxos (or an abort if it timed out while waiting on the other participants).  
> Before allowing any coordinator replica to apply the commit record, the coordinator leader waits until TT.after(s), so as to obey the commit-wait rule described in Section 4.1.2.

TL;DR, a transaction participant will pick a timestamp in the future to satisfy monotonicity and keep commit timestamps in sync across multiple participants.

The same idea works just as well for HLC-based systems.

<!--kg-card-end: markdown-->