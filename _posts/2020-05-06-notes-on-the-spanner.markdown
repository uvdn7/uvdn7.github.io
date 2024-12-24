---
layout: post
title: Notes on the Google Spanner Paper
date: '2020-05-06 19:30:34'
tags:
- spanner
- time
- distributed-system
- database
---

This is my notes on the paper: [Spanner: Google’s Globally-Distributed Database](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=4&cad=rja&uact=8&ved=2ahUKEwiM8NSx3p_pAhUOpp4KHRwAAoUQFjADegQIBBAB&url=https%3A%2F%2Fresearch.google.com%2Farchive%2Fspanner-osdi2012.pdf&usg=AOvVaw0jTMltcXSUju43NRB29vPi). I will first summarize the paper and try to explain how it works. At the end, I will list my opinion and questions about Spanner.

### What is Spanner

It's a globally replicated database that hides _sharding_ and _replication_. So as a customer of Spanner, it feels as if all your reads and writes are sent to a single running database instance. And even better, it's fault tolerant, so the service is still available under failures. Spanner provides this illusion, even though in reality, your data is too large to fit in a single physical host, and there are multiple copies of your data.

### How does Spanner do it

Most parts of Spanner are actually very traditional database and replication setups. E.g. A _Paxos group_, mentioned in the paper, is nothing more than a db shard. It just provides better availability comparing to a master-slave replicated database, because of Paxos. Spanner runs distributed transaction across multiple shards (a.k.a Paxos groups) by using two phase commit, specifically the Percolator style two phase commit. Again, nothing new.

But these alone don't hide _sharding_ and _replication_ from users. Users would still observe replication when they read stale data from a slave. They could still observe shards when they get a torn read (partial effect of a committed transaction). The sauce that makes Spanner almost magical is _TrueTime_.

> the linchpin of Spanner's feature set is TrueTime

### TrueTime API and what we can do with it

TrueTime provides a very simple API to applications.

| Method | Returns |
| --- | --- |
| TT.now() | interval: [earliest, latest] |
| TT.after(t) | true if `t` has definitely passed |
| TT.before(t) | true if `t` has definitely not arrived yet |

`TT.after()` and `TT.before()` are just helper functions. The only functionality that TrueTime API provides is the time interval `[earliest, latest]` when you query `TT.now()`. It "guarantees" that the absolute current time is within the interval. (I put quotes on "guarantee" because it's not really a guarantee. I will get to it later.)

Before we dig into how TrueTime works. Let's see what we can do with this seemingly simple but very powerful API. You can hide replication by providing linearizability. Combining single shard linearizability with distributed transactions (two phase commit) and snapshot isolation, you can hide sharding as well. TrueTime, combined with the fact that Spanner stores multiple versions of the same key, can achieve both.

If there's single keyword about distributed system, it's _ordering_. Weak consistency guarantees can all be phrased in terms of violations of some ordering. Spanner provides external consistency (a.k.a linearizability). This means, all events' timestamps in the system are consistent with the global external total ordering. This means if event `A` happened in Spanner, and you saw it. You called Bob (out of band communication) about it. Bob is guaranteed to observe the system in a state that's at-or-after `A` and never before `A` (unless you explicitly asked for a snapshot at a time in the past). And Spanner does this by using TrueTime.

There are two ways disordering can happen in a system.

1. write transaction A happens _after_ write transaction B, but it has a lower timestamp. (read-write transaction)
2. A read transaction A happens _after_ write transaction B, but it doesn't see the effect of B. (read-only transaction)

(Notice the usage of "after" above. It captures out-of-band communication as well, in which case Spanner can't reason about the causality between events. And this is where Spanner really shines.)

It first defined a few events for a given transaction.

| event name | note |
| --- | --- |
| e:start | transaction starts (when client starts a transaction) |
| e:server | server receives a commit message for the transaction |
| e:commit | server commits the transaction |

| timestamp name | note |
| --- | --- |
| `abs(e)` | the absolutely timestamp of a given event. We would never know what it is |
| s | transaction commit timestamp |

##### Read-write transaction

Spanner resolves #1 by guaranteeing `abs(e1:commit) < abs(e2:start) => s1 < s2`. Or in English, if a transaction A starts after transaction B was committed, it will have a greater commit timestamp. It can be achieved by having `s1 < abs(e1:commit) < abs(e2:start) < abs(e2:server) < s2`.

`s1 < abs(e1:commit)` is what's called Commit Wait. In English, it means after you pick a commit timestamp `s`, you don't actually perform the commit until you know `s` has definitely passed. You wait it out.

`abs(e2:server) < s2` can be achieved by calling `TT.now()` after `e:server`, and make `s` at least `TT.now().latest`, so `s` has definitely not arrived yet. Notice that we used TrueTime twice here. We first advanced `s` after `e:server` to the `latest` and waited for `s` to pass. So on average, for each commit, we are spending twice the average expected time difference between TrueTime and Absolute Time, waiting.

In the Spanner paper, transaction A and transaction B are mostly explained in the context of running on two different Paxos groups. Branch off topic a little bit, the same technical can be used for resolving multi-master write conflicts. That is, two writes to the same shard in a multi-master setting can have a consistent ordering as well. But it does introduce other challenges, e.g. when serving reads, you don't know if you are missing a multi-master write locally or not because a multi-master write can be in the middle of being replicated.

##### Read-only transaction

Spanner resolves #2 by effectively issuing reads with timestamp `TT.now().latest`. And Spanner server can return a consistent snapshot at the timestamp without locking (because Spanner stores multiple versions of the same key at different timestamps). In order to know, if a non-leader Spanner is sufficiently up to date to serve the read at a given timestamp, it needs to keep track of all the pending transactions to make sure it doesn't miss some transactions from the reply. This is tracked by a low watermark called `t:safe` in the paper. The paper says read-only transactions are lock-free, which is true because of MVCC. But reads can be blocked by replication lag and pending transactions.

### How TrueTime works

It's essentially built on top of both GPS clocks and Atomic clocks. Google picks two different clocks because it's very unlikely that they share same failure modes and have coordinated failures. They also have very good network infrastructure to keep the TrueTime interval small. But even with all these, there's really no proof that the absolute time always falls in the TrueTime interval. Local clock can drift more than expected between time sync intervals. Network latency can have outliers. Spanner's observation is that CPUs are 6 times more likely than bad clocks. That is to say, the CPU having a bad value in a register, or took a wrong jump based on a CPU flag, is 6 times more likely than TrueTime being wrong. Since we are building databases on not 100% reliable CPUs anyway, TrueTime is as trustworthy as the rest hardware we depend on. Nothing is 100% with computers.

### My personal opinion and questions

Spanner by and large is a very traditional distributed database, except the usage of TrueTime which was novel at the time. It provides very strong and nice (it can't be better) consistency guarantees to users. TrueTime itself and Spanner are more about great engineering achievements than distributed system theory breakthroughs.

The last section of the paper talks about benchmarks and use cases (e.g. F1). The 2pc latency metrics in the paper look good but the numbers were collected in the setup that client to server latency is about 1ms. Once leaders are more widely spread, round trip latency can easily dominate the 2pc latency.

The F1 use case demonstrates that once you have strong consistency and cross shard transaction support, client can build secondary indexes themselves and using the transaction primitives to keep them up-to-date. F1 also maintains a consistent materialized view of some data locally by taking a snapshot at certain timestamp and apply replication deltas on top. This is where TrueTime again really shines. It's made possible because sharding is totally hidden from users.

It's not mentioned in the paper how Spanner handles coordinated failures. Having everything tied to a clock is one way to have coordinated and cascading failures. It's also mentioned in the paper that unavailability of time-master can cause datacenter-wide increase in TrueTime interval. This means when time-master is down, all transactions in the datacenter will start to take longer at roughly the same time. Eventually this might cross some timeout threshold and causing failover and overload cascade. It does seem like this can be an operational nightmare. It's mentioned in the paper as well how Spanner works hard to reduce the TrueTime interval. It does seem to support the observation that Spanner is very sensitive to TrueTime quality.

The paper only provided mean latency and std dev of the latency for read operations. Spanner seems slow. But it might be the fastest we can get with this nice feature set. Specifically, for each transaction, you need to wait out the TrueTime uncertainty interval (2 \* avg difference between TrueTime and absolutely time). And read-only transactions can be blocked by replication and pending transactions.

Overall this is a great paper. Like the authors said, it's the first system that provides external consistency at a global scale.

<!--kg-card-end: markdown-->