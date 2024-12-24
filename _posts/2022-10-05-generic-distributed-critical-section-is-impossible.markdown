---
layout: post
title: Generic Fault Tolerant Distributed Critical Section is Impossible
date: '2022-10-05 03:16:03'
tags:
- distributed-system
- lock
---

In this post, we will show that a generic distributed critical section can't satisfy both safety and liveness properties with even a single faulty client process.

## System Model

We assume a non-real-time system with no bound on process time or network delay.

In the system, there exists a lock service which satisfies the safety property – it will not grant the same lock to more than one client at any time. We treat the lock service as a black box and assume that it is _always_ available. This is obviously not attainable in practice. As we will try to demonstrate later, even with a strong model (e.g. an always available lock service), a generic distributed critical section is still impossible.

There are `p` (`p > 1`) client processes trying to acquire the same lock to enter a critical section. The critical section is essentially distributed among all the client processes. The pseudo code on client looks like:

    lock.acquire()
    # inside the critical section
    lock.release()

## Generic distributed critical section

We assume all the work in the critical section can be completed in a finite amount of time on a _non-faulty_ client process. A faulty client process can be network partitioned, crashed, or stalled. Generic distributed critical section needs to satisfy the following properties:

- Safety – at most one client can enter the critical section at any time
- Liveness – when a client attempts to acquire the lock, it would eventually succeed 

## A single faulty client

Say, a faulty process A is holding the lock. Because it's faulty, there's no guarantee that it will release the lock eventually. This rules out all non-preemptive protocols, as if all other clients wait for the lock indefinitely, it would obviously violate the liveness property. With a preemptive protocol, another client can acquire the lock from the lock server before the previous lock owner explicitly releases the lock. Because the previous lock owner didn't explicitly release the lock, it can still be active in the critical section, executing instructions – meaning there can be two active client processes in the critical section, violating the safety property. No finite amount of wait is enough to ensure that the previous lock owner is no longer in the critical section. To preempt a lock, it requires a promise from the previous lock owner to _not_ do something. There's no guarantee to get this promise in a bounded (however long) amount of time. The faulty client process will eventually be out of the critical section, but it's impossible to know when, in order to coordinate the lock handoff – because we assume a non-real-time system.

## Consensus

Generic critical section is a harder problem than consensus. If one solves fault tolerant critical section, it can be used to build a fault tolerant consensus protocol trivially. But not the other way around.

Lock services, e.g. [Chubby](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwjs7b6JkMj6AhWFHjQIHcRGC4UQFnoECBIQAQ&url=https%3A%2F%2Fresearch.google.com%2Farchive%2Fchubby-osdi06.pdf&usg=AOvVaw1OIHckC-w_kgQKUF1ml1R9), requires clients to plumb lock sequence (a monotonically increasing id) to stateful observers so they can discard old messages. It's not "generic" by the definition of this post. As it requires control of the stateful observers. What if we are sending an SMS message in the critical section?

