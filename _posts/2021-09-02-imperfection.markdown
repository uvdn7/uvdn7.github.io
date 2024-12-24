---
layout: post
title: There's Nothing 100% in Computer Science
date: '2021-09-02 01:36:53'
tags:
- computer-science
---

When it comes to correctness guarantees, there are few communities in Computer Science (software) more obsessed with them than the database and distributed system communities. They better be serious about correctness because everyone's bank account balances depend on it.

You have heard about terms like Paxos, consensus, fault tolerance, but still there's nothing that's 100% in Computer Science. When it comes to fault tolerance, e.g. Paxos, it's only designed to tolerate certain failure mode - crash stop.

When you actually examine any database, distributed system protocol, etc., you will find that everyone of them, at the end of the day, depends on at least one lock working on one process. The "lock" probably depends on a futex, which depends on the Linux kernel implementation, which depends some instruction, which depend on atomic memory access, which depends on one transistor working as expected. If _one_ P-N junction on _one_ computer mishaves _once_, technically, it can cause arbitrary impact on upper layer applications.

No matter what fancy "AI" applications one can achieve with computers; ultimately they are just man-made machines and they are not perfect.

