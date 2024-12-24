---
layout: post
title: Paxos at its heart is very simple
date: '2018-11-14 15:34:20'
tags:
- distributed-system
- paxos
- consensus
---

> The Paxos algorithm, when presented in plain English, is very simple. -- Leslie Lamport 2001 <sup class="footnote-ref"><a href="#fn1" id="fnref1">[1]</a></sup>

I am going to try explain the single decree Paxos in a way, hopefully, that's easy to understand.

To this day, IMO, [John Ousterhout's lecture on Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=) is still THE BEST out there. The material in this post borrows heavily from Prof. Ousterhout's lecture. With John Ousterhout's agreement, I used some of his slides to help explain Paxos.

## Consensus

Paxos as a protocol, is trying to solve the consensus problem. Put in plain English, it's trying to solve a problem that a system will only _choose_ **one** value when one or more servers can _propose_ different value(s).

The real world example would be something like an election. People can vote for different president. But only _one_ president can be _chosen_.

Paxos is designed to deal with fail-stop/crash (non-Byzantine) and message delay/loss. Also there's a liveness requirement that _some proposed value is eventually chosen as long as majority of the servers are running_. This is to avoid uninteresting solutions like never-accept-any-value. If you do nothing, you will do nothing wrong. Similarly, you don't want more than one presidents be elected and you want to have _a_ president eventually as long as people are voting.

On a side note, to be honest, I think trying to use voting as an example to explain Paxos, in the original Part-Time Parliament<sup class="footnote-ref"><a href="#fn2" id="fnref2">[2]</a></sup> paper is very natural. But somehow it didn't work very well. Lamport also said it himself in an [interview](https://softwareengineeringdaily.com/2016/02/26/distributed-systems-with-leslie-lamport/) that it's a total failure, which is unfortunate.

## Single Acceptor

![Screen-Shot-2018-11-13-at-11.17.43-PM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-13-at-11.17.43-PM.png)  
With a cluster of `2F+1` servers, what happens if we always make one certain box to accept proposals. And whatever proposal being accepted by that box is chosen.

The problem of this approach is that a single down box can make the service unavailable. But the requirement says the service needs to be up and running as long as majority of the servers are running (`F+1` in this case).

Notice that this is effectively a single master replication model. And this is exactly why downtime is inevitable with master-slave replication.

## Quorum

So if having a single acceptor doesn't work, maybe we should wait until multiple servers have accepted a proposal before we claim a value being _chosen_.

How many?

If the acceptors are fixed and it's `<=F`, again, we fail to tolerate `F` crashed servers. If the acceptors can be any servers and it's `<=F`, in a cluster of `2F+1` servers, we might have two different proposals going to two disjoint set of acceptors causing two values be chosen.

So the number of acceptors has to be `>=F+1`. Since fewer the acceptors the better, let's just go with `F+1`. And it can be any `F+1` servers in the cluster.

We have already made some progress! **If any proposal's accepted by `F+1` servers, it's chosen.**

## Accepting proposals

Now let's see how acceptors should decide if to accept a proposal or not.

_Accept first proposal only?_  
 ![Screen-Shot-2018-11-13-at-11.17.27-PM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-13-at-11.17.27-PM.png)

It wouldn't work, because we can end up never have any value chosen like the case above. This violates liveness requirement.

It doesn't make sense to say accept second proposal or anything like that because you never know if there will ever be a second proposal.

So we have to accept the first proposal, but maybe not first proposal only.

_Accept every proposals?_  
 ![Screen-Shot-2018-11-13-at-11.31.11-PM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-13-at-11.31.11-PM.png)  
This would violate the safety requirement as we can end up have more than one value chosen.

## Core

Here comes the trick, which I think is THE MOST interesting part of Paxos.

> Once a value has been chosen, future proposals must propose the same value.

This means a two-phase protocol would be needed. In the first phase, proposers need to figure out if any value has already been chosen. And propose in the second phase.

If you are a distributed system veteran you must have already noticed one key word in the previous statement. And it's "future". I reviewed [Time and Order]( __GHOST_URL__ /time-and-order/) before. Distributed system is all about ordering. "future" here implies ordering (happen-before relationship). So we have to **order** all proposals. Then in the example below, we can reject `red` because we know `blue` is chosen and `red` is an old proposal.

![Screen-Shot-2018-11-13-at-11.39.37-PM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-13-at-11.39.37-PM.png)

#### Ballot Number

An easy way to order proposals is to have an unique ballot number for each proposal in the form of `<seq_id, server_id>`. In order to guarantee ordering, `seq_id` has to be persisted cross crash/restart. So it has to be durably stored locally.

## Paxos

Now we are all set up to describe the real two-phase Paxos protocol. We will call the first phase the `prepare` phase and second phase the `accept` phase. `prepare` phase serves two purposes in one round trip.

1. learn about any value that's already chosen
2. prevent acceptors accepting any older proposals (very much like the RESERVE step in Two Phase Commit)

![Screen-Shot-2018-11-13-at-11.53.52-PM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-13-at-11.53.52-PM.png)  
Acceptor has to durably store a few things

- `minProposal`
- `acceptedProposal`
- `acceptedValue`  
_Notice that instead of "broadcast to all servers", it should be "broadcast to at least F+1 servers". And proposer should send the accept message to the same set of F+1 servers as in the prepare phase._

Let's take a close look at each step.

#### Prepare phase

In this phase, proposer tries to get a _promise_ from acceptor to not accept any proposals older than his. At the same time, acceptors will inform the proposers what values have already been accepted so far.

In step `3` and `4`, a proposer will change its proposal to the latest accepted proposal from acceptors it talked to. It seems to be very conservative because you might get an accepted proposal that's not chosen and the proposer ends up giving up his proposal. **But that's OK** since we only care about consensus. In the first phase, it's hard for the proposer to know if a value is _chosen_ or not. An accepted value might not be chosen. But a chosen value must be accepted by a majority of the servers! And acceptors reject older proposals (step `6`). With two invariants, we know that the latest accepted proposal returned to the proposal in phase one either has already been chosen, or no value has been chosen at the moment. Either way, it's safe to propose that value.

_Why keep track of `minProposal`?_  
There can be two racing proposals in prepare phase while nothing has been accepted yet. This means they will both move to accept phase. Thanks to the total ordering we have for all proposals and the `F+1` requirement, at least one acceptor will see the newer proposal and reject the `accept` message from the losing/older proposal. The invariant here is that `minProposal >= acceptedProposal`.

![Screen-Shot-2018-11-14-at-12.28.11-AM]( __GHOST_URL__ /content/images/2018/11/Screen-Shot-2018-11-14-at-12.28.11-AM.png)

This is probably the most interesting case, as the second proposer didn't see the previous accepted value and a value was not chosen at the time in prepare phase. In this case, it will go ahead and send `accept(Y)` to other servers. Because `S3` has _promised_ that it will only accept proposals later than `4.5`, it rejects `X`. If we don't keep track of `minProposal`, `X` will be accepted by `S3`. Then both `X` and `Y` would have been accepted by three servers and considered chosen, which violates the safety requirement.

There you have it. This is the single decree Paxos. That's being said, implementing Paxos has been proven to be very very hard<sup class="footnote-ref"><a href="#fn3" id="fnref3">[3]</a></sup>.

* * *
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item">
<p><em>Paxos made simple</em> <a href="https://lamport.azurewebsites.net/pubs/paxos-simple.pdf">https://lamport.azurewebsites.net/pubs/paxos-simple.pdf</a> <a href="#fnref1" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn2" class="footnote-item">
<p><em>The Part-Time Parliament</em> <a href="https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf">https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf</a> <a href="#fnref2" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn3" class="footnote-item">
<p><em>Paxos made live</em> <a href="https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf">https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf</a> <a href="#fnref3" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section><!--kg-card-end: markdown-->