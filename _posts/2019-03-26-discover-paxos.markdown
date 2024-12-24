---
layout: post
title: Discover Paxos via 2pc
date: '2019-03-26 04:33:37'
tags:
- paxos
- 2pc
---

## 2pc and Paxos are solving the same problem

Two Phase Commit, a.k.a 2pc, is a very well-known and intuitive algorithm, often used for coordinating transactions. Paxos is a well-known consensus algorithm, often used for replication on a stateful service. Paxos is less intuitive and harder to grasp.

Atomic commit is a classic 2pc use case. E.g. you want to commit a transaction touching different MySQL shards. You do it by locking all the rows in the first phase and committing the transaction in the second phase.

Leader election is a classic use case of Paxos. E.g. you want all replicas to agree on which host is the leader, that should be taking writes. You run single-decree Paxos among all the replicas, and you will get a leader chosen.

Their use cases look different, but fundamentally, 2pc and Paxos are solving exactly the same problem -- consensus, which is just a fancy word for different parties agreeing on something. In distributed transaction, you want different MySQL shards to have _consensus_ on the decision of a transaction, commit or abort. In leader election, you want all replicas to have _consensus_ on who's the new leader.

In this post, we will try to solve the consensus problem using 2pc as a starting point. Our goal is to meet both **Liveness** and **Correctness** requirements. At the end, we will "discover" Paxos. Hopefully this exploration will help us grok Paxos.

> Liveness - some value will eventually be chosen as long as majority of the hosts are available.
> 
> Correctness - one and only one value can be chosen.

I will stick with Paxos terminologies in this post. I am going to stick with `proposer` and `acceptor`. Proposer is called "coordinator" in 2pc. Acceptor can be called "participants" in 2pc or voter in some cases.

## Use 2pc for leader election

We need everyone to agree on who's the new leader, right? Simple!  
**Proposal 1**

1. [Proposer: prepare] Lock _all_ acceptors.
2. [Acceptor: prepare] Reply YES if there's no existing lock. NO otherwise.
3. [Proposer: accept] If _all_ acceptors are locked, send ACCEPT message telling everyone who's the new leader. Otherwise, abort and unlock all acceptors.
4. [Acceptor: accept] Simply learn about the chosen value.

This is **correct**. But the protocol can't make any progress as long as a single acceptor is not responsive. This doesn't meet our **Liveness** requirement.

Proposer actually doesn't need to wait for _all_ acceptors. It only needs majority votes anyway.

**Proposal 2**

1. [Proposer: prepare] Try lock _all_ acceptors.
2. [Acceptor: prepare] Reply YES if there's no existing lock. NO otherwise.
3. [Proposer: accept] Send ACCEPT message to all as long as the proposer has majority votes. Otherwise, abort and unlock all acceptors.
4. [Acceptor: accept] Simply learn about the chosen value.

This is already not the classic 2pc anymore. The proposer can commit a value that's rejected by some of the acceptors. In other words, we have to overwrite reservations in ACCEPT phase in some cases. From now on, let's call it "prepared" instead of "locked".

Notice that once leader election result is accepted on a single acceptor, the leader is _elected_ (or chosen).

This is correct, as racing proposals will always fail due to lock contention. It's better than proposal-1, but it can't tolerate proposer failures. It's still not fault-tolerant and doesn't meet our **Liveness** requirement.

## Distribute the knowledge

2pc is a simple protocol because it has a single source of truth regarding to the decision (value chosen), which is controlled by the coordinator/proposer.

Let's consider a case, where the proposer got majority votes, and crashed just before it was able to send out any ACCEPT messages. Technically, majority acceptors are already "under control", whatever value to be proposed will be chosen. But we are stuck because the proposer died before it can communicate that information to anyone.

> Considering a scenario of total three participants. Things would be left in state: (`prepared`, `prepared`, `not-prepared`).

_We can't rely on a single original proposer to finish the protocol. When the original proposer died, information of the proposed candidate (the value) died with him. We need to distribute enough knowledge to acceptors in phase one, so that another proposer can come along, learn about the state of the world, and finish things off._

The most intuitive approach is to ask proposer to send proposed value in phase one and acceptors store the information locally. Essentially, we are trying to make it like (`preparedValue=Lincoln`, `preparedValue=Lincoln`, `preparedValue=None`).

Then it's possible for `proposer-2` to send `ACCEPT Lincoln` upon seeing (`preparedValue=Lincoln`, `preparedValue=Lincoln`) from two different acceptors. Notice that `proposer-2` would want to commit Lincoln instead of a different value e.g. Seward. This is because `proposer-2` might be in a race with `proposer-1`, who's still in the middle of proposing Lincoln.

**Proposal 3**

1. [Proposer: prepare] Send `PREPARE Lincoln` to all acceptors.
2. [Acceptor: prepare] Set `preparedValue` if not set. Reply with `preparedValue`. Notice we don't overwrite existing `preparedValue`.
3. [Proposer: accept] If the same value is prepared by the majority, send ACCEPT message with the value chosen to all.
4. [Acceptor: accept] Simply learn about the chosen value.

Consider the following case:

1. Proposer-1 proposed value `Lincoln`.
2. Majority of acceptors set `preparedValue=Lincoln`.
3. Proposer-1 died.
4. Proposer-2 came along and proposed `Seward`.
5. It heard back two prepared value of `Lincoln` instead. Proposer-2 then sent `ACCEPT Lincoln` to all.

#### Need more metadata

In this way, we can tolerate proposer failures. **But it's not correct.** In step-5 above, what happens if proposer-2 saw (`preparedValue=Lincoln`, `preparedValue=Seward`), two responses out of total three acceptors? He doesn't have enough information to pick which one to commit, since the third acceptor can be either in state `preparedValue=Lincoln` or `preparedValue=Seward`. **We need more metadata to help us pick a winner between Lincoln and Seward.**

#### Chosen as soon as prepared by the majority

Notice that after step-2 above, `Lincoln` is already considered chosen when majority have `preparedValue=Lincoln`, as no other value can be possibly chosen. So, the "commit" in step-5 is not needed. **We are wasting a phase.**

    propose - value chosen - learn
    <------- p1 ----------><--p2-->

Even more, **we _must_ be able to call a value chosen, as early as when it's prepared on the majority of the acceptors.** This about this for a second. Prepare phase is used to get some promise from acceptor (otherwise it's not useful at all). E.g. "do not accept any values from anyone except me". Let's try proof by contradiction. Notice that we are not being explicit about what the promise is.

Assume we can _only_ call a value chosen after commit/accept message is seen by at least one acceptor. This implies proposer can't trust the promises from acceptors. (If majority voters promised they would vote for Lincoln, and you trust them, at that moment, Lincoln is already chosen.) Then the following case is possible:

1. proposer-1 sent `PREPARE Lincoln` to all.
2. proposer-1 got promises from majority acceptors.
3. proposer-1 only managed to send `ACCEPT Lincoln` to one out of three acceptors.
4. proposer-2 came along. Sent `PREPARE Seward` to all.
5. proposer-2 got promises from the remaining two acceptors that haven't seen Lincoln being accepted. _Because promises are not reliable._
6. proposer-2 sent `ACCEPT Seward` to all.

Now both Lincoln and Seward are chosen. BAD!

We got two key insights here.

1. We need more metadata to help us pick a winner from two prepared values.
2. We must be able to consider a value chosen when it's prepared on the majority of the acceptors. So, we are wasting a phase.

#### Proposal Id

Let's pick metadata. The simplest metadata to be associated with each proposal is a `proposalId`. Let's assume we have a globally unique and totally ordered `proposalId` associated with each proposal.

> This is not hard to implement as we don't require this total order to be consistent with physical time. The simplest implementation of such `proposalId` is a tuple of `(timestamp, proposer-hostname)`.

With such metadata, upon seeing (`<preparedValue=Lincoln, preparedProposalId=2>`, `<prepraedValue=Seward, preparedProposalId=1>`), in the [Proposer: accept] step, it can simply pick the one with the higher `proposalId`. Or can it? We can only pick the value of the higher `proposalId`, if it's **safe** , meaning state (`<preparedValue=Lincoln, preparedProposalId=2>`, `<preparedValue=Seward, preparedProposalId=1>`, `<preparedValue=Seward, preparedProposalId=1>`) **has to be impossible**.

How can we make it impossible for the third acceptor to be in `<preparedValue=Seward, preparedProposalId=1>` state? Most intuitively, We want the presence of `<preparedValue=Lincoln, preparedProposalId=2>` to ensure us that majority of the acceptors don't have a different `proposalId` stored. If we want the presence of `preparedProposalId=2` to carry this meaning, it must be the second phase of a two-phase protocol. The first phase, `PRE-PREPARE`, getting a promise from each acceptor that they will not "prepare" any proposals other than `2`, and the second phase of `PREPARE <value=Lincoln, proposalId=2>`. Sounds familiar? Isn't this exactly proposal-2? This is 2pc all over again. And we know it can't tolerate proposer failures. We are back to square one if we go down this route. We need to relax something here.

#### Relax the promise

Remember we said `Lincoln` was considered chosen when prepared by majority of the acceptors? We want it to be **safe** to propose `<preparedValue=Lincoln, preparedProposalId=2>` instead of `<preparedValue=Seward, preparedProposalId=1>`. What **safety** means here is not necessarily `Lincoln` be the chosen value. We only need the guarantee that it's _impossible_ for `Seward` to be the chosen value. **We need the presence of `preparedProposalId=2` to ensure us that majority of the acceptors promised not to prepare proposals with `proposalId < 2`.**

This implies we allow them to prepare proposals with different values as long as the `proposalId`s are higher. This is the side-effect of the relaxation, we made.

Here's what we have so far.  
**Proposal 4**

1. [Proposer: pre-prepare] Generate `proposalId`. Send to all acceptors.
2. [Acceptor: pre-prepare] Maintain a `minProposalId` locally. Bump it when the incoming `proposalId` is higher.
3. [Proposer: prepare] Send `<preparedValue=Lincoln, proposalId=pid>` to all acceptors.
4. [Acceptor: prepare] Maintain `preparedValue` and `preparedProposalId`. Set both if `proposalId >= minProposalId`. Otherwise reject the PREPARE message.
5. [Proposer: accept] After successfully prepared by majority of the acceptors, send `ACCEPT Lincoln` to all acceptors.
6. [Acceptor: accept] Simply learn about the chosen value.

#### Add back the state

Well there is a problem. In PREPARE phase, acceptors no longer send back the `preparedValue` any more. Remember, we added that in proposal-3, so that a different proposer can finish off what's left behind by the previous proposer. We did that for safety reason.

E.g. the state of the world can be

    (`<preparedValue=Lincoln, preparedProposalId=2>`, `<preparedValue=Lincoln, preparedProposalId=2>`, <preparedValue=Seward, preparedProposalId=1>)

Apparently, `Lincoln` is chosen because it's prepared on the majority of the acceptors. Let's say proposer-1 died before any one learned about the chosen value. Proposer-2 came along, and proposed `<preparedValue=Chase, proposalId=3>`. `Chase` would be chosen! This violates the **safty** requirement. We need to add that back. The only possible place is step-2.

Remember we were also kind of wasting a phase anyway. Names like `PRE-PREPARE` is confusing. Let's "shift up a phase" and rename a few things.

> `PRE-PREPARE` phase =\> `PREPARE` phase  
> `PREPARE` phase =\> `ACCEPT` phase  
> `preparedProposalId` =\> `acceptedProposalId`  
> `preparedValue` =\> `acceptedValue`

    // shift up a phase
    propose - value chosen - learn
    <---p1--><-----p2---->

Now as long as a value is "accepted" (instead of calling it prepared) by the majority, the value is chosen. Let's not worry about the phase of letting acceptors know which value is chosen, it's trivial anyway. Now we have:

**Proposal 5**

1. [Proposer: prepare] Generate `proposalId`. Send to all acceptors.
2. [Acceptor: prepare] Maintain a `minProposalId`. If incoming `proposalId` is higher, bump the `minProposalId`. If there is already an accepted proposal, reply `(acceptedProposalId, acceptedValue)`. (Notice that this is exactly the same as step-2 in proposal-3, except it doesn't keep track of `proposalId`s.)
3. [Proposer: accept] Wait to hear back from majority acceptors. If there are any accepted values, pick the value with the highest proposalId. And replace the original proposal value to that. Send ACCEPT message to all.
4. [Acceptor: accept] If the incoming ACCEPT message's `proposalId` is lower than the `minProposalId`, reject it. Otherwise, set `acceptedProposalId`, and `acceptedValue`.
5. [Proposer: post-accept] If proposal gets accepted by majority, the value is chosen.

We pick the value from proposals with the highest `proposalId`, as discussed in proposal-3 to help us pick a winner. And the response `(acceptedProposalId=2, acceptedValue=Lincoln)` ensures that `(acceptedProposalId=1)` won't be present in majority of the acceptors. Because proposer-1 had the promise from the majority of not accepting proposals with `proposalId < 2`. And **this promise is distributed among the acceptors and picked up by proposer-2.** This makes committing proposals with `proposalId=2` safe.

## Paxos "discovered"

Proposal-5 above, is exactly Paxos. Well, I hope this makes it easier for us to grok Paxos.

<!--kg-card-end: markdown-->