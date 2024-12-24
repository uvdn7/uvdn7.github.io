---
layout: post
title: State Machine and Synchronization
date: '2018-04-21 22:47:13'
tags:
- distributed-system
---

_This is the second of two notes of Lamport's Time-Clock paper<sup class="footnote-ref"><a href="#fn1" id="fnref1">[1]</a></sup>. The first one is [here]( __GHOST_URL__ /time-and-order/)._

## The problem

We will be focusing on how to use Logical Clock to solve an actual problem. The problem is to _grant the resource to a process, that_

1. a process which has been granted the resource must release it before it's granted to another process (mutex)
2. different requests for the resource must be granted in the other in which they are made \*\*
3. if every process which is granted the resource eventually releases it, then every request is eventually granted

## A centralized solution

The requirement(2) is non-trivial to satisfy. The most naive solution would be having a centralized master that assigns the resource to processes upon receiving the request. But it doesn't actually satisfy (2). Let's say _P0_ made two requests, _R0_ to the master and _R1_ to _P1_. _P1_ after receiving _R1_, made _R2_ to the master requesting for the resource. _R2_ can be received before _R0_. If the master naively assigns the resource upon receiving a request, (2) will be violated.

This can be solved with Logical Clock, which gives a partial ordering of all events. We can define a global ordering of all events that's consistent with the partial ordering by breaking ties in a deterministic way (e.g. `t:Pi < t:Pj iff i < j`). We can modify the previous naive algorithm a bit. Now each process not only needs to request resource from the master, it also keeps sending heartbeats to the master. Let's say `t0:p0` is the first request. `t0` will be the smallest request timestamp received by the master. Once the master has received messages, including heartbeats from all processes that have timestamp greater than `t0`, it can safely grant the resource to _P0_. The master will then wait until _P0_ releases the resource and then repeat the process. The master doesn't need to store all the heartbeat messages from all processes. It only needs to track the _high watermark_ from every process and if the resource is currently available or not.

This works. But it has a few downsides.

- it's a centralized algorithm
- sending heartbeats can take a significant portion of the network bandwidth comparing to the actual `request` and `release` commands

## A distributed solution

![Paper.Sketches.1]( __GHOST_URL__ /content/images/2018/04/Paper.Sketches.1.png)

1. Every process sends _request resource_ (timestamped) to everyone, and stores the request locally.
2. Upon receiving the _request_, each process stores the request and sends an ACK (timestamped) to the sender.
3. The resource is granted to a _process Pi_, when (i) `req:Pi:Ti` is in the local store and `Ti` is the smallest timestamp for a resource request locally (ii) `Pi` has received messages from all other processes with `T > Ti`
4. To release the resource, _Pi_ sends _release resource_ (timestamped) to everyone, and removes any `req:Pi` locally stored.
5. Upon receiving the _release_, each process also removes any `req:Pi` from the local store.

![Paper.Sketches.2]( __GHOST_URL__ /content/images/2018/04/Paper.Sketches.2.png)

_(There are a few assumptions made to make describing the algorithm simpler. E.g. messages are sent and received in order.)_

It's pretty obvious that it satisfies all three requirements.

## State Machine

If we generalize the solution above a little bit, it's not hard to see that it's actually a generic method for achieving synchronization in distributed system (without failures).

You can imagine that each process simply broadcasts timestamped _commands_ to everyone. And everyone sends back _ACK_. Each process only _executes_ a _command_ when it satisfies the (3) above. In this way, all processes will eventually execute all commands in exactly the same order. If we model each process as a state machine, they all start with the same initial state. Then they will go through exactly the same sequence of state transitions due to applying (or executing) the same sequence of commands. And eventually end up in exactly the same end state, which achieves synchronization. They all do that independently without any centralized coordination.

Lamport likes to reason about things (either distributed system or a program) using state machine. It's used in the Paxos, and now in [TLA+](https://lamport.azurewebsites.net/tla/tla.html). They both originated from the state machine idea in this paper, the same old, Time-Clock paper.

* * *
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item">
<p>Leslie Lamport. Time, clocks and the ordering of events in a distributed system. <em>Communications of the AMC</em>, 21(7):558-565, July 1978. <a href="#fnref1" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section><!--kg-card-end: markdown-->