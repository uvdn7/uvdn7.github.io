---
layout: post
title: Project Management is a Concurrency Control Problem
date: '2023-01-30 19:28:24'
tags:
- database
---

_The target audience of this post is the database community, and especially those who take an interest in the OLTP workload._

When multiple people or multiple teams are working on the same service, the same code base, even the same code path, we need project management to keep track of dependencies, progress, and deliverables. I just realized recently that project management is a database concurrency control problem.

Each project is analogous to a database transaction. When you make a design for a project, it's targeting a fixed product – a snapshot. You want the Serializability isolation level for your project management, otherwise, the final product might not work properly. We can borrow ideals from established concurrency control solutions to solve our project management problem.

#### Pessimistic concurrency control 

When executing a project, you can use two-phase-locking, and block other people/teams from changing the components you are touching; and only release the lock when the project launches – analogous to when a transaction commits.

#### Optimistic concurrency control

On (merge) conflicts, retry the project and risk starvation.

#### MVCC

This is loosely analogous to keeping multiple versions of the product with different feature sets as they are being built.

In this analogy, a TPM is like a scheduler that sequences transaction execution. A good TPM can help maximize concurrency, and throughput and minimize contention.

