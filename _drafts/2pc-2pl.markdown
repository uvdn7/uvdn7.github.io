---
layout: post
title: Two Phase Commit (2PC) is an application of Two Phase Locking (2PL)
tags:
- distributed-system
- database
---

- lock is about extracting a promise (on the other hand, a promise is more general than a lock)
- the same proof for 2PL providing serializability can be used for proving why 2pc provides serializability; because 2pc is an application of 2pl ([https://courses.cs.washington.edu/courses/cse444/16sp/lectures/lecture14-transactions-locking.pdf](https://courses.cs.washington.edu/courses/cse444/16sp/lectures/lecture14-transactions-locking.pdf)) 2pc is trivially serializable because it's an application of 2PL 
- distributed system and database are inseparable because they are both concurrent systems ([https://jepsen.io/consistency](https://jepsen.io/consistency)) 
- 2pc is C2PL + S2PL (locks are taken before txn starts, hence no deadlocks – a property from C2PL, and it only releases locks at the end when the txn is already committed or rolledback – a feature of S2PL)
