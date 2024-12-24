---
layout: post
title: Non-blocking 2pc
date: '2019-02-19 06:17:03'
tags:
- 2pc
- distributed-system
- consensus
---

## 2pc is a Blocking Protocol

Two Phase Commit is a blocking protocol. It blocks when Coordinator is not available. Not only the transaction cannot make progress. Other transactions that conflict with the same set of keys are also blocked.

## Non-blocking 2pc Alternative

Daniel Abadi proposed a non-blocking alternative for 2pc in his blog post [It’s Time to Move on from Two Phase Commit](http://dbmsmusings.blogspot.com/2019/01/its-time-to-move-on-from-two-phase.html). It's a little light on details of the proposed algorithm. I will try to elaborate the algorithm here.

#### Example

2pc is a locking protocol, which is very useful for constructing read-modify-write transactions. The classic example would be transferring money from two bank accounts. Say we want to perform a transaction wiring `N` dollars from `X`'s bank account to `Y`'s. If there's less than `N` bucks on `X`'s balance, nothing should happen. The work needs to be performed of each resource managers would be:

    // for X
    IF BALANCE_X >= N:
        BALANCE_X -= N
    ELSE:
        ABORT

    // for Y
    BALANCE_Y += N

#### 2pc Solution

2pc solution is straightforward.

1. In the `PREPARE` round, the Coordinator sends `RESERVE` to `X` and `Y`. `X` and `Y` should send back `YES` if it can perform the operation and at the same time **lock** the balance to prevent updates from any racing transactions. In this specific case, `X` would send back `YES` only if it has a balance more than `N` dollars.
2. When the Coordinator collected two `YES`s from `X` and `Y`, it sends a `COMMIT` command to `X` and `Y`, which will perform mutations on both bank accounts.

Notice that in theory, here, as long as `X`'s balance is more than `N`, the transaction should always commit. But in practice, this is **not** always the case. The server handling `Y`'s account can be slow, or crashed even at the time, and the Coordinator with only one vote, must send a `ROLLBACK` command to both.

_But do we have to rollback in this case?_

This is the **key** insight that inspired Prof. Abadi's blog post.

#### Deterministic Transaction

Before we talk about the algorithm, we need to store more metadata in order to perform non-blocking read-modify-write transactions.

`account_balance_history`

| account\_id | timestamp | balance |
| --- | --- | --- |
| X | t<sub>x1</sub> | M |
| X | t<sub>x0</sub> | 0 |
| Y | t<sub>y0</sub> | 0 |

`account_balance`

| account\_id | timestamp | balance |
| --- | --- | --- |
| X | t<sub>x1</sub> | N + M |
| Y | t<sub>y0</sub> | M |

Here we store multiple versions of each key, at different `timestamp`. Here the `timestamp` is like the one in [RAMP transaction](/ramp/). It's globally unique. And it provides a total ordering of all mutations on a single key. It can be easily achieved by making it a tuple of `(timestamp, client_host_name)`. `account_balance_history` stores the account balance for transaction at time `timestamp` but before executing the transaction. `account_balance` is the materialized account balance view after transaction execution. E.g. according to the tables above, there's a transaction at t<sub>x1</sub> to add `N` dollars to `X`'s balance. (For the following I am going to assume the timestamps from clients are monotonically increasing, this is easy to achieve by using HLC.)

Here's the non-blocking deterministic read-modify-write algorithm.  
Client picks a transaction timestamp `t`. Then send two mutations to `X` and `Y`.

    // for X
    IF BALANCE_X >= N:
        INSERT INTO account_balance_history (X, t, BALANCE_X)
        UPDATE account_balance SET balance=BALANCE_X - N, timestamp=t where account_id=X

    // for Y
    BAlANCE_X = SELECT balance from account_balance_history where timestamp=t AND account_id=X
    IF BALANCE_X >= N:
        INSERT INTO account_balance_history (Y, t, BALANCE_Y)
        UPDATE account_balance SET balance=BALANCE_Y + N, timestamp=t where account_id=Y

Putting in English, `Y` reads `X`'s balance at `t` to figure out the decision of the transaction instead of relying on a Coordinator. Notice that with this algorithm, there's no Coordinator; hence no blocking. The transaction **has to** commit as long as `X` has a balance more than `N`. Even if `Y` crashed at the time, it needs to perform the `+N` after it restarts. In this way, the "read" part of the read-modify-write is moved to one of the resource managers. "modify-writes" are done in-place.

## Summary

Dr. Abadi's idea is good. But it's not the only way of avoiding the Coordinator-block problem. E.g. with Percolator style of 2pc, you store the transaction decision with the underlying resource. Then the availability of the transaction decision is the same as the availability of the resource you are updating. Essentially, parties in a transaction need to learn about the decision in some way. If they learn it from a single source of truth, e.g. the coordinator or other participants in the transaction, it's essentially 2pc. If they learn it from a quorum of hosts, it then becomes Paxos.

<!--kg-card-end: markdown-->
