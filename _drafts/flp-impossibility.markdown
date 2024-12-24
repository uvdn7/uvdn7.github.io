---
layout: post
title: FLP – get your bivalent booster today
tags:
- distributed-system
---

At the beginning of the [FLP](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwjSgLaxl6f6AhV7GFkFHfrQAJQQFnoECAUQAQ&url=https%3A%2F%2Fgroups.csail.mit.edu%2Ftds%2Fpapers%2FLynch%2Fjacm85.pdf&usg=AOvVaw3cwr00WJuxyxJUTcm4rELk) paper, it says

> ... no completely asynchronous consensus protocol can **tolerate** even a single unannounced process death.

> ... the stopping of a single process at an inopportune time can cause any distributed commit protocol to fail to reach agreement. Thus, this important problem has **no robust solution** without further assumptions about the computing environment or still greater restrictions on the kind of failures to be tolerated!

These sentences, taken out of context, can be misleading. But what's the definition of a "robust solution" and "tolerate"?

In this post, we will review the famous, and beautiful FLP paper. We will cover what does FLP really say, and its consice and beautiful proof.

## Definitions and assumptions

The paper was written, and the theorem was proved by mathmaticians. We need to be very precise about the definitions and the assumptions to just understand what the theorem is saying, not to mention the proof.

The paper assumes the network is completely asynchronous – messages can take arbitrarily long to be delivered, and in any order. This is a fair assumption matches that happens in the [real world]( __GHOST_URL__ /why-timeouts-are-hard-to-get-rid-of/) (at least most of the real world not on real time systems and network). The paper assumes a non-real-time system. So any time-out based protocols are not considered (as they are not reliable). Crash stop is the only failure mode considered – it's indistinguishable from network partition, so they can be considered equivalent in the scope of this paper.

The paper focuses on the simplest consensus problem – agreeing on a value in `{0, 1}` (deciding on either 0 or 1) among a group of processes. The correctness property requires that processes that do make a decision agree on the same value. The liveness property requires that _some_ processes eventually make a decision. It's the liveness property that's proven to be unattainable for _any_ consensus protocols with the aforementioned assumptions, with a very narrow definition. E.g. Paxos can terminiate with probability 1 in practice, but it's not considered to satisfy the FLP-liveness property.

## System Model

The paper essentially models any distributed system as a state machine, without loss of generality. If you are familiar with TLA+, this notion will come very naturally.

Essentially, we will think of the entire distributed system as a set of process states (optionally with a global message buffer). The entire distributed system moves in a state machine triggered by messages sent and recieved by participants in the system. Each state transition is atomic. The atomicity clearly doesn't hold in practice. You can easily have two processes having state changes in parallel (or the same process processing two messages at the same time). However, it doesn't matter from a modeling perspective, as we can model the two state changes one after another without affecting our reasoning about the correctness. Either the two messages are affecting the same value/state, which ordering is required, in which case, nothing should happen in parallel. Or the two messages are affecting independent values/states, in which case, arbitrary ordering of the state changes doesn't affect the outcome of our reasoning.

