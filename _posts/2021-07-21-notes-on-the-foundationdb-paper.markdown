---
layout: post
title: How FoundationDB works and why it works
date: '2021-07-21 15:29:41'
tags:
- foundationdb
- consistency
- database
---

FoundationDB is a very impressive database. Its [paper](https://www.foundationdb.org/files/fdb-paper.pdf) won the best industry paper award in SIGMOD'21. In this post, I will explain, in detail, how FDB works and discuss a few very interesting design choices they made. It's a dense paper packed with neat ideas. Many details (sometimes even proof of correctness) are not included in the paper. I added proof wherever necessary.

## What is FoundationDB?

It's a _non-sharded_, _strict serializable_, fault tolerant, _key-value_ store that supports point writes, reads and range reads. Notice that it provides a key-value API (not SQL). It's also not sharded, meaning the entire key space is essentially on one logical shard. That's it. Once you have a strict serializable key-value store, you can layer a SQL engine and secondary indexes on top. A strict serializable (can be relaxed if needed obviously) key-value store is the foundation (a smaller reusable component), upon which you can build distributed databases almost<sup class="footnote-ref"><a href="#fn1" id="fnref1">[1]</a></sup> however you want. This is a _great_ design choice.

## Bold Claims

The paper makes a few jaw-dropping claims in the intro.

- FDB is strict serializable and lock-free. How is this possible?
  - me: Don't you have to take locks to guarantee order?
- For a global FDB deployment, it can avoid x-region write latency.
  - me: Hmm... OK. It's possible if you make some assumptions about failure domains.
- FDB can tolerate _f_ failures with only _f+1_ replicas.
  - me: You gotta be kidding me!

We will come back to all these jaw-dropping claims. If these claims don't excite you, this post and the paper are probably not for you.

## Architecture

![Screen-Shot-2021-07-05-at-11.01.20-AM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-05-at-11.01.20-AM.png)

FDB takes the idea of microservice to the extreme (they call it decoupling). Almost every single functionality is handled by a distinct service (stateful or stateless). Embrace yourself for many names for different roles.

#### Control Plane

The most important Control Plane service in the diagram is the _Coordinator_. It basically stores small metadata about the entire FDB deployment/configuration (e.g. the current epoch, which gets bumped every time a reconfiguration/recovery takes place). _ClusterController_ monitors the health of all servers (presumably via heartbeats, as it's not mentioned in the paper).

#### Data Plane

FDB breaks the Data Plane into three parts, Transaction System, Log System and Storage System. Log System and Storage System are mostly what you would expect – a distributed log and a disaggregated, sharded storage. The Transaction System is the most interesting one, among which the _Sequencer_ is the most critical service.

## Transaction Management

1. A client is always connected to a Proxy, which communicates with FDB internal services.
2. The Proxy will get a read-version in the form of HLC (hybrid logical clock) from the _Sequencer_. Notice that there's only one (not logical) physical _Sequencer_ in the system, which serves as a single point of serialization (ordering). The _Sequencer_ conceptually has the following API, which keeps producing monotonically increasing HLCs(or LSNs).  
 ![Screen-Shot-2021-07-17-at-11.58.41-PM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-17-at-11.58.41-PM.png) The singularity of the _Sequencer_ is critical, as it provides _the_ ordering of all relevant events of an entire FDB deployment. HLC and LSN are interchangeable for the rest of this post.
3. The client then issues reads to _Storage_ with the specific read-version, acquired from the _Sequencer_. Storage System will return data at the specific point-in-time indicated by the read-version (an HLC) – MVCC.
4. The client then asks _Sequencer_ again for a commit-version.
5. Next, it sends the write-set _and_ the read-set to the _Resolver_. _Resolvers_ are range-sharded and they keep a short history of recent writes. Here, it detects if there are any conflicts (i.e. if the data read earlier in the transaction has changed or not, between `[read-version, commit-version]`).
6. If no conflict is detected, commit the writes to the Log System with commit-version. Once enough ACKs have been received from the logs, the _Proxy_ returns success to the client.

It's basically Optimistic Concurrency Control (OCC), very straightforward. In this example, it actually doesn't even need MVCC. Because if there are conflicts, i.e. what the client read has changed, the transaction will be either aborted or retried anyway. MVCC support in FDB is there to support snapshot reads, and read-only transactions.

_Edit: Bhaskar Muppana, one of the authors of the paper, pointed [one mistake](https://www.linkedin.com/feed/update/urn:li:activity:6823635315471863808?commentUrn=urn%3Ali%3Acomment%3A%28activity%3A6823635315471863808%2C6826392929918160896%29) I had in the following paragraph. It has been corrected. Thanks, Bhaskar._

The reason why _Proxy_ is needed is that it allows reading uncommitted writes within a transaction, which is a very common semantic provided by almost all relational databases. It's achieved by buffering uncommitted writes locally on the Proxy servers, and merging these writes with storage data for reads on the same keys. Client (notice not _Proxy_) caches uncommitted writes to support read-uncommitted-writes in the same transaction. This type of read-repair is only feasible for a simple k/v data model. Anything slightly more complicated, e.g. a graph data model, would introduce a significant amount of complexity. Caching is done on the client, so read queries can bypass the entire transaction system. Reads can be served either locally from client cache or from storage nodes.

_Proxy_ sits between clients and the transaction system. Its responsibility is to serve as a semi-stateful "client". E.g. it remembers the _KCV_ to aid recovery; it performs batching (of requests from multiple clients) for reducing _Sequencer_ qps.

_Resolvers_ are range-sharded, hence they can perform conflict checks in parallel. That means _Proxy_ needs to wait to hear back from all _Resolvers_ involved in each transaction. How does a _Resolver_ get recently _committed_ keys and their versions? It doesn't actually. Otherwise, it would require a distributed transaction between the _Log_ and the _Resolver_. The workaround is that _Resolvers_ keep a set of recent **write-attempts** and their versions. It's possible that a key appears to be recently written in a _Resolver_ while in fact its transaction is rolled back, leading to false positives. FDB claims the false positives are manageable because:

1. most workloads fall onto one _Resolve_ (hence it would know if it needs to record the commit-attempt or not)
2. the MVCC window is 5 seconds. Waiting it out is not the end of the world, for conflicts are only around a single key – partial unavailability.

How about racing commits? E.g. A Resolver can first admit a transaction with read-version `Vr` and commit-version `Vc`. Later, another transaction with commit-version `Vr < Vc2 < Vc` can hit the Resolver. In this case, the log servers will make sure to accept Vc2 first before accepting Vc, preserving linearizability. All messages in all log servers are sorted in LSN (See 2.4.2, "Similarly, StorageServers pull log data from LogServers in increasing LSNs as well.").

_Resolver_ doesn't need locks to detect conflicts. That's why FDB claims itself to be "lock-free". I found this mention of "lock-free" misleading. In order to realize strict serializability, locks must be used _somewhere_ in the system to implement this total ordering. The _Sequencer_ might have locks inside (not mentioned in the paper). Or even if there's only a single IO thread in the _Sequencer_, the kernel must use locks to maintain a queue of network packets. Even atomic memory operations are essentially implemented by locks in hardware. Without locks, ordering _can't_ be achieved.

I know you are probably thinking about what if _Sequencer_ or _Resolver_ goes down. We will get there.

## Logging Protocol

![Screen-Shot-2021-07-16-at-11.57.52-AM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-16-at-11.57.52-AM.png)

FDB's logging protocol, at first glance, seems pretty straightforward. It has a group of logs, and a separate group of storage servers with distinct replication factors and topologies – another example of physical decoupling in FDB. If you dive a little deeper, you will notice that things are much more interesting.

There are multiple _LogServers_. Remember, FDB provides an abstraction of one logical shard (non-sharded) database. Commonly, people use logs for sequencing (e.g. Raft), in which case, each log is mapped to a logical shard. So how does FDB provide a single shard abstraction with multiple logs? The answer is **FDB decouples _sequencing_ from _logging_**. There is only one _Sequencer_ in the system, which maps to the single shard abstraction (ordering). It enables FDB to have multiple logs, as they do _not_ perform the job of ordering, which logs in other systems usually do. This design is just brilliant. Whoever performs the job of sequencing must be singular. You will get higher throughput and lower latency by having a dedicated role just doing sequencing, and _nothing_ else. In contrast, systems like Raft couple sequencing and logging, which inherently pay a performance cost. In FDB, the _Sequencer_ is physically a singleton – there's a single process. We will talk about failure recovery very soon. Here's what the logging protocol looks like,

- _Proxy_ broadcasts a message to **all** _LogServers_. Each message will have a common header including `LSN`, `prevLSN` and `KCV`. `KCV` is the last known committed version from the _Proxy_. It should always be less than the `LSN` (`KCV < LSN`), as the transaction associated with `LSN` is still in progress (hence not committed). `KCV` captures an incomplete local view of the world, which will be used later for the recovery process.
- Not all broadcast messages include data. E.g. key `A` is stored on storage shard `S`. _Coordinator_ maintains an affinity map (each storage server has one preferred LogServer). In this case, data will only be included in the message sent to shard `S`'s preferred LogServer (and potentially additional LogServers to satisfy replication requirements). All other messages to other LogServers will only include the common header.
- Once the data is in the log, it will apply to the storage node. It's not super interesting, although there are some performance improvements mentioned in the paper.

There are a few interesting details above. _Proxy_ broadcasts a message to "all" logs. Why all logs instead of only the replicas for the key? Because all messages are applied in order on each log. There can be no gaps. Also, as you will see later, that it's critical for the recovery process. Another related question is why sending messages without data to other logs? This is a performance optimization, as only logs associated with the underlying storage nodes need to have data stored.

Because FDB decouples sequencing from logging, now committing to the logs becomes a distributed transaction problem – e.g. what if log-1 has some data stored, while log-2 doesn't, for an aborted transaction. Does FDB need to perform 2pc? No, actually.

FDB here basically adopts the [Calvin](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiEw8fzoPLxAhX6KFkFHU1eB14QFjAAegQIBRAD&url=http%3A%2F%2Fcs.yale.edu%2Fhomes%2Fthomson%2Fpublications%2Fcalvin-sigmod12.pdf&usg=AOvVaw0XE8ck-WBihvUiGgAnzr2H) approach. It's the same idea that Prof. Abadi [posted](http://dbmsmusings.blogspot.com/2019/01/its-time-to-move-on-from-two-phase.html). The keyword is _deterministic_. We don't need 2pc here, because _Proxy_ writing to multiple logs is performed _blindly_ (a.k.a unconditionally). Sure, errors can happen. If they are transient e.g. timeout, _Proxy_ can always try again – it will never be blocked by another log commit. This is, by the way, why FDB can afford letting _Proxies_ broadcast each message to _all_ logs.

In this case, the only failure scenario that can actually block log commit and require rollback, is when _LogServers_ actually fail (e.g. crash). We will talk about the recovery process soon.

Notice that in the diagram above, there are 5 logs and 10 storage nodes (and one _Sequencer_ not in the diagram). `#StorageNode > #Logs > #Sequencer` is not a coincidence. A single _Sequencer_ can support multiple logs, as a _Sequencer_ is much more lightweight (not having to write to disk). A single log again can support multiple storage nodes, as logs are relatively cheaper than storage (you can prune logs, sequential writes are cheaper, etc.).

## Recovery Process

Finally, the real fun begins. What happens if the _Sequencer_ fails? _Resolvers_ fail? _LogServers_ fail?

1. Failures are detected by the _ClusterController_ (via heartbeats according to the [source](https://github.com/apple/foundationdb/blob/master/design/recovery-internals.md)), who explicitly triggers the recovery process.
2. The _Sequencer_ locks and reads the previous states from _Coordinators_. It can be a new _Sequencer_ instance, running the recovery process, if the previous _Sequencer_ is in a faulty state. That's why we need to lock the _Coordinator_ here, to avoid multiple _Sequencers_ making topology changes at the same time. The paper doesn't talk about what happens if the recovery Sequencer itself crashes, leaving the Coordinators locked forever. This is a classic consensus problem. Given the Coordinators are already running Paxos, this is not a hard problem to solve. E.g. You can think of each recovery itself being a Paxos Proposal. If a recovery Sequencer died halfway, another Sequencer can always come and discover the previous chosen value if any, or propose a new epoch – starting a new recovery process.
3. Previous states include information about all old _LogServers_. The _Sequencer_ then stops old logs from accepting new transactions. The _Sequencer_ knowing it's running a recovery process, obviously won't accept new transactions. But if there's another old _Sequencer_ instance running, it can still accept new transactions. Hence we need to stop the old logs to avoid potential dataloss. Another reason for contacting the old _LogServers_ is to discover the end of the redo log (the last committed LSN).
4. The _Sequencer_ recruits a new set of _Proxies_, _Resolvers_, and _LogServers_.
5. Write the new system states to the _Coordinators_, and release the lock.
6. Finally, the new _Sequencer_ starts accepting new transactions.

Unlike other fault tolerance algorithms, e.g. Paxos, **FDB's recovery process has _downtime_** – from the shutdown of old _LogServers_ to when the new _Sequencer_ starts accepting new transactions. One can argue that this is not really _fault tolerant_. I personally think it's totally fine to have partial downtime to heal a system (given a short one, which is the case for FDB), e.g. unavailability for a single shard. It's not like systems such as Paxos or Raft have 100% availability even when the simple majority of hosts are always up and running. However FDB has only _one_ logical shard. Its recovery process causes downtime for the entire deployment and the entire key space.

![Screen-Shot-2021-07-19-at-11.33.17-AM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-19-at-11.33.17-AM.png)

P50 downtime for FDB's recovery process is 3.08s. That means no one can write to FDB at all during this window. The paper argues clients can still read from storage nodes unaffected.

> During the recovery, read-write transactions were temporarily blocked and were retried after timeout. However, client reads were not impacted, because they are served by StorageServers.

This statement seems not completely accurate. Snapshot reads are unaffected. But if _Sequencer_ is down, no read-only transactions can be processed either. FDB is read-heavy. It's very reasonable to assume that most of the production read workloads are snapshot reads (instead of read-only transactions). Claiming minimum client visible impact is reasonable.

If a system allows downtime, it can tolerate a lot of replicas going down without affecting it's availability. E.g. With the simplest leader-follower setup, a single leader can have any number of followers (let's say 10). In this case, with a total replication factor of 11, in theory, we can tolerate up to 10 failures without a single second of downtime.

_Proxies_ are easy to recover, as they are mostly stateless (just keeping track of a `KCV` locally). _Resolvers_ are stateful, but it's safe to discard all _Resolvers_ data from the previous epoch. Because it's impossible to have conflicts spanning across two epochs, as you can't have read-write transactions that span two epochs (as a completely new set of Proxies are recruited). That leaves us with the logs and the _Sequencer_, which are harder to recover. We just need to know the last committed LSN from the old logs to start a new _Sequencer_ instance. Let's take a closer look at the recovery process of the _LogServers_.

#### Log Recovery

![Screen-Shot-2021-07-20-at-3.15.38-PM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-20-at-3.15.38-PM.png)

For a LogServer deployment with _m_ hosts and replication factor _k_, here's how the algorithm looks like:

- Each log maintains `DV` (Durable Version, the maximum persisted LSN) and `KCV` (Known last committed version, the latest committed version known from the _Proxy_). Both _DV_ and _KCV_ are piggy-backed on each message from Proxy to the logs.
- The _Sequencer_ attempts to stop _all_ old logs.
- After hearing back from _m-k+1_ logs, the _Sequencer_ knows the previous epoch has committed transactions **at least** up to `max(KCVs)`. We call `max(KCVs)` here `PEV` (Previous Epoch's end Version). The new _Sequencer_ will start its epoch from `PEV+1`.

It seems pretty straightforward. However, the paper mentions another variable `RV` (Recovery Version) to be `min(DVs)`, and it talks about copying data between `[PEV+1, RV]` from the old logs to new logs and discarding logs after _RV_. This is for,

1. providing failure atomicity when log commit fails partially (in which case, we need to perform rollback)
2. preserving (healing replication factor) for data that are committed at LSN \> PEV

#### Proof of Correctness

_The paper didn't include any proof for the correctness of the recovery process. I will explain and prove its correctness here._

Notice that any log message arrived at the logs have already passed the conflict detection, hence safe to be durably stored. Especially given the fact that data stored might have already been read by another client, we can and should err on the side of preserving more of the old logs except when it's not safe to do so. We know for sure that all transactions before _PEV_ are committed. We just need to find the _tail_ of the redo log, where

1. partial failures might be happening, for which we must rollback, and
2. no clients have ever seen the data yet (because it's the tail)

Let's take a closer look at this diagram.

![Screen-Shot-2021-07-20-at-11.16.41-PM]( __GHOST_URL__ /content/images/2021/07/Screen-Shot-2021-07-20-at-11.16.41-PM.png)

For it to be correct (that we can safely discard data after _RV_ and preserve data before _RV_), the following conditions need to be met:

1. DV<sub>m</sub>'s cross-range-shard transaction cannot be partially committed (no rollback required).
2. No clients can read any data written after DV<sub>m</sub>.

Let's try proof by contradiction on #1 and #2.

What if DM<sub>m</sub>'s transaction were partially committed (specifically, rollback is required)? Notice that "partial" here specifically means cross-range-shard transaction, e.g. a transaction that sets both key A and key B, key A is stored with data in its logs, but key B is not. In this case, we have to rollback A. Logs can tolerate _k-1_ failures (_k_ being the data replication factor). If at least one surviving B replica has the data for transaction DM<sub>m</sub>, no rollback is required because we have all the data for T<sub>m</sub>, for which we will finish the commit process (heal the replication factor). Otherwise, let's say _k-1_ B replicas that got DM<sub>m</sub> data crashed, there must be at least one B replica up and running that has not received the DM<sub>m</sub> message. On that log, its _DV_ must be smaller than DM<sub>m</sub>. Contradiction.

Now let's move on to #2. Notice that `DV >= KCV` is true locally for each log, because `LSN > KCV` is true on each _Proxy_. Log local _DV_ can sometimes be the same as _KCV_ because reads from the _Proxies_ don't set _DV_ but can bring log local _KCV_ up to date. Even better, `max(KCVs) <= min(DVs)` should hold always, thanks to the fact that each log commit requires broadcasting the message to _all_ logs. This answers our question earlier why Proxies need to broadcast each log message to _all_ logs. Now it's obvious why #2 holds, because FDB can only serve reads based on _KCV_ (and not _DV_). So for any _DV \> DV<sub>m</sub>_, those transactions must not be committed.

With #1 and #2 combined, we can be confident that we can safely treat _RV_ as the tail of the redo log. The presence of a _KCV_ on a log, implies that the associated transaction is fully replicated (having _k_ copies). It's not covered in the paper, but conceptually we can think of these messages having been durably flushed to storage nodes (safe to discard from the logs). For messages between `[PEV+1, RV]`, they _might_ be committed transactions. It's safe to err on the side of treating them as committed, as all writes to the logs have already passed conflict detection. These cases would be indistinguishable from reply timeouts. But due to the lack of information, we don't know for sure if messages in the `[PEV+1, RV]` window have been durably stored in storage nodes or not. The easiest way to handle it is to start the new epoch at _PEV+1_ and replay messages from `[PEV+1, RV]` from the old logs.

## Replication Summary

Given the extreme decoupling, each component has its own replication strategy.

- _Coordinator_ runs on Paxos, which can tolerate _f_ failures with a cluster size of _2f+1_.
- _LogServers_ can tolerate _f_ failures for replication factor _f+1_, which is orthogonal to the number of servers. Storage nodes share a similar strategy. E.g. there can be a total of 5 logs, with a replication factor of 3, which can tolerate 2 failures.

## Those Claims at the Beginning

I think FDB is a great system, probably the best k/v store. However, I do think those bold claims at the beginning of the paper are, frankly, misleading.

- "FDB provides strict serializability while being lock-free". Well, to be more precise, it's meant to say the conflict detection is lock-free. _Sequencer_ is not lock-free for example.
- "For global FDB deployment, it can avoid x-region write latency." Well, it's based on semi-sync in a local region in a different failure domain. It requires assumptions about failure domains.
- "FDB can tolerate _f_ failures with only _f+1_ replicas." Well, this only applies to logs and storage nodes. If simply majority of the _Coordinator_ fail, FDB is totally hosed for example.

## Summary

FDB is probably the best k/v store for regional deployment out there. It's rock solid thanks to its simulation framework. The FDB team, in my opinion, made all the best design choices, which is just remarkable. The paper is dense. A lot of important details are only briefly mentioned or even omitted, which is probably due to the paper length limit. Congratulations to the FoundationDB team.

* * *
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item">
<p>You are still bound by the underlying scaling bottleneck of FDB's singletons, mostly the Sequencer. <a href="#fnref1" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section><!--kg-card-end: markdown-->