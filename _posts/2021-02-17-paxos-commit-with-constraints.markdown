---
layout: post
title: Paxos Commit with Constraints
date: '2021-02-17 17:50:50'
tags:
- paxos
- 2pc
- calvin
---

 **Correction: The following description of Paxos Commit is inaccurate. The algorithm is still correct but it's not the same Paxos Commit proposed in the paper. 4/25/2021.**

Lamport proposed [Paxos Commit](https://lamport.azurewebsites.net/video/consensus-on-transaction-commit.pdf) algorithm as an alternative to 2PC for achieving Transaction Commit. It's a very natural application of the Paxos algorithm (if you have read [this post](/understanding-paxos/), where I pointed out Paxos is effectively itself a read-modify-write transaction protocol.) In Paxos Commit, the transaction decision (to execute/"commit" it or not) is distributed among the participants. This means if majority of the participants decide to commit, the decision of this transaction is _chosen_ to be _commit_. It doesn't cares about what the minority says.

There's one subtle difference between 2PC and Paxos Commit that is not addressed in Lamport's paper or anywhere I can find. It's about how it enforces constraints. Taking databases for example, values are sometimes associated with constraints (e.g. y can never to less than 0, x can never be greater than 1, etc.). When you are running transactions that touches values with constraints, the correctness of 2pc is straightforward, because every participant in a transaction has veto power. It makes sure all invariants are honored before moving to the commit phase.

But how does this work at all for Paxos Commit? Paxos Commit distributes the _decision_ of a transaction (_commit_ or _abort_) to the participants (a.k.a RM/Resource Managers, Acceptors). When `x` or `y` is involved in a Paxos Commit, the decision can be _commit_ while `x` can't commit without violating its own constraint. What's going on here? Did Lamport make a mistake?

Of course not. You can actually break Transaction Commit into two parts

1. consensus of executing a transaction (all-or-nothing)
2. correctly executing a transaction without violating constraints

2PC solves both problem at the same time. Paxos Commit only solves the first problem. i.e. with Paxos Commit, all participants will _agree_ on whether to _execute_ a transaction or not. But it doesn't say about how to execute the transaction correctly to avoid violating constraints.

OK. But how do we solve #2 indenpendently? This is exactly what the [Calvin paper](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) is about. The gist of it is that if we can agree on #1 (either all or nobody will execute a transaction), we can solve #2. Dr. Abadi actually wrote a [blog post](http://dbmsmusings.blogspot.com/2019/01/its-time-to-move-on-from-two-phase.html) about exactly how this would enable us to move away from 2pc. The main idea is that once all participants agreed on executing a transaction (#1), they can figure out all the dependencies and constraints independently. E.g. participants can query the host owning key `x` to see if it executed transaction-id-42. It does require keeping extra metadata for transactions at least for a short period of time. Dr.Abadi himself confirmed this on HN as well.

![](/assets/paxos-commit.png)

### Basically, all-or-nothing(Paxos Commit) + CAS(compare-and-set) replaces 2PC.
