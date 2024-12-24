---
layout: post
title: Linearizability is more than Capturing Causality Everywhere
date: '2023-01-03 18:28:42'
tags:
- distributed-system
---

> Linearizability is one of the strongest single-object consistency models, and implies that every operation appears to take place atomically, in some order, consistent with the real-time ordering of those operations ([https://jepsen.io/consistency/models/linearizable](https://jepsen.io/consistency/models/linearizable)).

Quite often people make the mistake of over-simplifying linearizability as the ability of being causally consistent even in the existence of out-of-band communication channels. It's tempting to just call linearizability as read-everyone's-writes; except that it's not entirely accurate.

## Read-everyone's-writes

Let's first provide a precise definition of &nbsp;"read-everyone's-writes". It's a generalization of read-your-write, which says one should be able to read what he/she wrote earlier. Read-your-write, as a property, is not concerned about a scenario when the same person sends two concurrent operations, one read and one write because they are not causally related.

_Precisely, in a system which provides read-everyone's-write, if a read happens after a successful acknowledgement of a write, the read should see a state at-or-later than the write._

Notice that we didn't say the write needs to be acknowledged by a client. However, the property is only concerned about the scenarios when we know for sure that the read happens after the acknowledgement of the write – the two operations are causally related. For example, if we only send the read operation after the client receives an ACK from the write, we know for sure that the read happens after the write, because there's a causality between the operations.

## An Example

We send two concurrent writes W(A) and W(B) to a system that provides the property "read-everyone's-writes" by the aforementioned definition. While W(A) and W(B) are inflight, we send two more concurrent read operations R(A) and R(B). There are no other communication channels between us (the client) and the database (e.g. we can't tail the binlog to find out if W(A) happens first or not).

In this example, R(A) &nbsp;observes W(A) but not W(B) while R(B) observes W(B) but not W(A). This doesn't violate the read-everyone's-writes property. R(A) implies that W(A) happens before W(B), because it sees W(A) but not W(B); while R(B) implies exactly the opposite.

This can happen in a system that provides read-everyone's-writes, but it can never happen in a Linearizable system, where there is a total order of all operations.

