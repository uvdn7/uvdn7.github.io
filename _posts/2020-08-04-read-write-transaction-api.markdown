---
layout: post
title: Read-Write Transaction API
date: '2020-08-04 20:17:40'
tags:
- transaction
---

There are multiple ways you can approach a Read-Write transaction API. MySQL's transaction API is fully conversational that comes with great flexibility and power. However it can be easily abused causing long lock duration, especially in an OLTP workload.

Most of the choices related to transactions boil down to pessimistic vs. optimistic concurrency control.

## Pessimistic Read-Write Transaction

Spanner takes the pessimistic approach.

> Like Bigtable, writes that occur in a transaction are buffered at the client until commit. As a result, reads in a transaction do not see the effects of the transaction’s writes. This design works well in Spanner because a read returns the timestamps of any data read, and uncommit- ted writes have not yet been assigned timestamps.  
> Reads within read-write transactions use wound-wait [33] to avoid deadlocks. The client issues reads to the leader replica of the appropriate group, which acquires read locks and then reads the most recent data. While a client transaction remains open, it sends keepalive messages to prevent participant leaders from timing out its transaction. When a client has completed all reads and buffered all writes, it begins two-phase commit.

## Optimistic Read-Write Transaction

Optimistic read-write transaction looks exactly like read + multi-CAS. In this case, unlike its pessimistic counterpart, reads do not hold locks. The final CAS will only succeed if all the records read don't change in the scope of this transaction.

Optimistic RW transaction is simpler to implement but it doesn't guarantee that a client can always make progress when there are contentions. Pessimistic RW transaction requires more complex logic on the client and has a longer lock duration in general.

<!--kg-card-end: markdown-->