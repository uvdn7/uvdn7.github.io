---
layout: post
title: When and How to Invalidate Cache
date: '2022-06-11 23:29:34'
tags:
- cache
- consistency
- invalidation
---

_You can leave comments on HN – [https://news.ycombinator.com/item?id=31709501](https://news.ycombinator.com/item?id=31709501)._

In this post, I will talk about one way to figure out _when_ to invalidate cache entries. I will use a specific setup as an example, which should still be general enough, and at the end of the post, I will briefly talk about how to generalize it even more.

So does this blog post solve anything new or is the "solution" here novel in any sense? **No**. All I did is to treat cache as a distributed system (as it deserves). Does the cache invalidation solution here work from a protocol's perspective (in a sense that can be proved by TLA+)? **Yes**. I am happy to write a TLA+ spec for it if people are interested. Let me know.

The first half of the blog post describes a potential solution for solving a specific cache invalidation problem. The second half of the blog post lists tradeoffs when we try to generalize the solution. You can still invalidate cache but sometimes at a huge cost (unbounded write amplification, huge computational overhead, etc.). It's not that it can't be done in those cases; but it might not worth it, depending on your use case, workload and requirement. So, yes, you can figure out _when_ and _how_ to invalidate caches (given that you treat it as a distributed system problem), but you might not want to actually do it for various reasons.

## Example

Say a system has following components (a very typical web application setup):

- Many stateless clients (web servers)
- one cache server (say Memcache)
- one database (say Postgres)

In the database we store a table (a set of [relations](/relational/)) called `friends`. And its schema looks something like:

<!--kg-card-begin: markdown-->

    +---------------------+------------------+------+-----+---------+-------+
    | Field | Type | Null | Key | Default | Extra |
    +---------------------+------------------+------+-----+---------+-------+
    | user_id | bigint unsigned | NO | PRI | 0 | |
    | friend_id | bigint unsigned | NO | PRI | 0 | |
    +---------------------+------------------+------+-----+---------+-------+

<!--kg-card-end: markdown-->

Now in cache we want to store a set of relations of `friends-of-friends` because in our application we have code like `getFriendsofFriends($bob)`. It's very common/natural to cache a list of friends-of-friends keyed by user-id in services like Redis or Memcache. Say the key in Memcache is user id (e.g. Bob's user id), and the value is a list of user ids who are friends of Bob's friends. That's it.

## Definition

- I will define "cache" here as _any_ materialized view of what's in the source of truth in this post, and assume this definition in this post unless otherwise specified. 
- By "invalidate" I meant the action of updating/dropping relevant cache entries when the source of truth mutates, so no stale data is stored in cache indefinitely.

## Requirements

When you are using cache, by definition, you are dealing with a distributed system – there is the cache, and the source of truth. Since you are dealing with a distributed system, we need to be clear about the expected state/knowledge distribution. Too many systems approach cache in an ad-hoc way, without realizing they are dealing with a distributed system, leading to the idea that figuring out when to invalidate cache is uniquely hard. If one runs a query against Postgres, and stores the data locally on its client cache, without the database knowing, it's obvious that invalidating the local client cache can't always be done correctly. E.g. another client can come and mutate Postgres underneath without the first client knowing. Postgres doesn't know the first client cached any data with its earlier query, etc.

<figure class="kg-card kg-embed-card"><blockquote class="twitter-tweet">
<p lang="en" dir="ltr">That's inherently a difficult problem, because it requires very clear thinking about that balance between the desired system properties and the costs of coordination. Too many systems approach that in an ad-hoc way, leading to the idea that cache invalidation is uniquely hard.</p>— Marc Brooker (@MarcJBrooker) <a href="https://twitter.com/MarcJBrooker/status/1534944338341310470?ref_src=twsrc%5Etfw">June 9, 2022</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</figure>

In order to deal with a distributed system correctly, the cache and the database need to be aware of each other. That is to say, the database needs to know that some queries are being cached; and the cache needs to know where the data is coming from (and especially how mutations are ordered – if there is a single keyword in distributed systems, it would be _order_).

More specifically, cache data schema must be made known to the entire data system (not just the cache itself – a single running process somewhere). It's obvious that without these requirements, reliable cache invalidation can't be done generally (one might figure out an ad-hoc solution for a specific use case).

## Mutations / Write Transactions

E.g. someone comes and `INSERT`s a new friendship between Bob and Alice:

<!--kg-card-begin: markdown-->

    BEGIN;
    
    INSERT INTO friends (user_id, friend_id)
    VALUES
      ($bob, $alice), ($alice, $bob);
    
    COMMIT;

<!--kg-card-end: markdown-->

Our goal here is to figure out when and how to invalidate cache entries (friends-of-friends) _reliably_. Postgres has this feature supporting [returning data from modified rows](https://www.postgresql.org/docs/current/dml-returning.html). It's handy in this case, as we want to know the user-ids, who's friends-of-friends list could have changed. Now the write transaction would look something like this:

<!--kg-card-begin: markdown-->

    BEGIN;
    
    INSERT INTO friends (user_id, friend_id)
    VALUES
      ($bob, $alice), ($alice, $bob) RETURNING user_id;
    
    COMMIT;

<!--kg-card-end: markdown-->

In this simple example, it's very easy to figure out the user\_ids that are mutated (you can achieve the same by having a `SELECT` after the `INSERT`). But in more complex queries, the `RETURNING` feature from Postgres becomes very handy.

Now when the db client executes this transaction it will receive a list of `user_id`s, whose friendship has changed. Now not only Alice's and Bob's friends-of-friends lists need to be updated in cache but also Alice's friends' and Bob's friends'. So once we have the `user_ids`, we need to run a `select` query to get all the users affected by this mutation:

<!--kg-card-begin: markdown-->

    SELECT friend_id
    FROM friends
    WHERE user_id in $user_ids;

<!--kg-card-end: markdown-->

Now we have a complete list of user ids to invalidate. That the order of operations is:

- client starts a transaction
- client runs any mutations as needed
- client collects `user_id`s whose cache entries need to be invalidated
- store the list of `user_id`s durably in a separate table or store it in binlog (e.g. MySQL) – must be done in the same DB transaction
- client commits the transaction
- invalidate the cache based on `user_id`s

In this case, even if the client crashes after DB commit, you can have a tailer that tails the invalidation log, and applies cache invalidation asynchronously and reliably. The invalidation only requires at-least-once delivery semantic. A very naive implementation of the tailer can look something like:

- read the `to-be-invalidated` table
- send cache invalidation
- upon acknowledgement, truncate the `to-be-invalidated` table (up-to where cache invalidations have been sent)

There can be cases when the same cache entries be invalidated multiple times; and it's OK, as long as cache invalidations are sent _after_ DB commits. There is another classic race when on a miss, client goes upstream to get the data. And when the data/reply is in network, someone else changed the database, and the new invalidation hits the cache server first. This problem can be solved by leases in memcache or versioning each cache item explicitly.

This works when there's only one database (single shard, single primary).

What's nice is that the steps of "collecting user\_ids and invalidating cache" can be encapsulated in your DB client, and so you don't have to worry about someone forgetting to invalidate the cache. You notice that the DB client in this case is aware of the cache schema (so it knows to collect user\_ids, etc.). This is where the earlier requirement kicks in,

> cache data schema must be made known to the entire data system.

In practice, whoever adds a new cache schema, the same person should update the db client for handling cache invalidation. One might notice that the `RETURNING` clause is expected to be attached to every mutation, and question how generic and practical this is. If you are familiar with MySQL and [RBR](https://dev.mysql.com/doc/refman/8.0/en/replication-rbr-usage.html), think about it this way, we are essentially trying to get a set of RBR events _synchronously._ So the only requirement here is that for every mutation, `RETURNING` updated rows for each table (relations). It's a function of the existing tables in your database, and independent of the cache schemas. It's very generic.

You might have noticed that what we are doing here is essentially "transacting" over DB and cache. It's not an ACID transaction but it's essentially an always-commit transaction on the cache side. The reason why it can be more relaxed than cross-system/cross-shard db transaction is because what's in cache is purely a materialized view of what's in the database. So there can't be conflicts in a sense that DB commits a write, and cache says there's a conflict and I can't invalidate the cache (which can happen in the case of x-system/x-shard db transactions).

## Generalization

Now let's generalize the previous setup – single cache, single database.

**What if the cache schema is complicated?**

Say, we have a cache that stores friends-of-friends who live in US. So besides the `friends` table, we now have a `home` table where stores where each user lives. Now updates to either `friends` or `home` can affect our cache. Remember, one key requirement we listed earlier is that cache schemas are made known to the entire data system.

Let's only focus on the changed rows, as only changed `friends` relations and/or changed `home` relations can affect our cache. If `friends` relations are changed, the same "transform" function we talked earlier in the post applies mostly unchanged (besides adding an additional join and filter with the `home` table). We just need to add logic to collect invalidation keys, when `home` relations are changed. It can look something like this:

<!--kg-card-begin: markdown-->

    BEGIN;
    
    UPDATE home SET country = 'US' WHERE user_id = $bob RETURNING user_id;
    
    COMMIT;

<!--kg-card-end: markdown-->

Now on `home` relation updates, we will get back a list of `user_id`s whose home countries are changed, which can affect our "friends-of-friends who live in US" cache. Then the collection query can look something like:

<!--kg-card-begin: markdown-->

    SELECT friends.user_id
    FROM
    (
        SELECT friends.user_id as uid
        FROM friends
        WHERE friend_id in $user_ids -- notice that we didn't filter on country, as any country change (move to or out of US) should trigger invaildation
    ) AS user_with_friends_who_moved
    JOIN friends
    ON friends.friend_id = user_with_friends_who_moved.uid;

<!--kg-card-end: markdown-->

Now we have a complete list of invalidate keys – user\_ids whose friends-of-friends have moved. The gist is that what's stored in cache is also just a set of relations. Once we know its schema, figuring out a list of keys that need to be updated, within the same DB transaction, is absolutely doable. Does it work? Yes (if by "work" it means if it can reliably invalidate cache). Is it always a good idea? Certainly not. Just look at the query here. It has unbounded fanout; and it slows down your transaction. Cache invalidation is not free.

**What if I have a lot of cache replicas, and a lot of cache schemas and data?**

There is this write amplification (to cache) that happens asynchronously. If you have a lot of cache replicas (not uncommon, since caches are commonly introduced for scaling read traffic), the write amplification is pretty terrible. It slows the write transaction down, and it tends to be error prone (higher error rate, transaction timeout, lock timeout, etc.). This shouldn't be surprising because cache stores a denormalized materialized view and shares similar characteristics as table indices on the update path; hence share the write amplification cost similarly as well.

One option for reducing write amplification is to push more compute to the read path. E.g. here you can cache user -\> friends list, and do "join"s on the read path. The result won't be a snapshot (unless you do point-in-time reads against the database).

**What if I have a read-only db replica?**

Here be dragons. After a cache is invalidated (after db mutation), if you have a db replica (always somewhat behind the db primary), there's always a chance that cache can fill its data from a stale db replica. A concrete example would look like this:

- DB has data A
- client updates A -\> B and invalidates the cache
- DB replica still has data A
- cache fills from DB replica and puts A back in
- DB replica catches up and updates A -\> B
- now cache will have A (a stale data) indefinitely

I will not go into details but there are a few ways to solve this. E.g.

- send cache invalidation with version data
- keep track of version data in cache, so it never goes back in time (e.g. after processing B's invalidate, it should never go back to A again)
- use leases (supported in memcache)

**What if I have multiple DB shards, and I have a cache that materializes data from multiple DB shards?**

The solution mostly remains the same, the only problem you need to solve is again ordering (which is the single most important keyword in distributed systems). Depending on your system, you might be able to use vector clocks, or if you have Spanner (lucky you), you can just use TrueTime to resolve the ordering issues so cache states would never go back in time.

**What if I cache data from multiple sources, files, databases, replies, etc.?**

When you use cache, you are dealing with a distributed system. If you cache data from files _and_ databases _and_ replies from other services, you have to reason all of them together as a distributed system. If you are in this situation, and it requires cache invalidation, chances are some ad-hoc reasoning happened in this past. I will quote [Marc Brooker](https://twitter.com/MarcJBrooker/status/1534944338341310470) here,

> Too many systems approach that in an ad-hoc way, leading to the idea that cache invalidation is uniquely hard. .... ad-hoc reasoning about distributed designs frequently leads to failure.

**What if I don't store data in Postgres?**

When designing a cache, you need to be aware of the features that your database supports. This is where the earlier requirement kicks in:

> In order to deal with a distributed system correctly, the cache and the database need to be aware of each other.

When you are dealing with caches, you are dealing with a distributed system. Treat it as a distributed system. If your database doesn't support returning modified data synchronously, but supports binlog (e.g. MySQL), what you can do is

- tail the binlog on mutations (where you can collect mutated user\_ids)
- send invalidation from the tailer

Here be dragons. We need to run selection logic e.g. find out the invalidation keys when someone changes his address. That requires a snapshot point-in-time read at the time of when the write transaction is executed. Quite a few databases offer this capabilities (it's not too expensive if the database only needs to keep history for a few minutes, when this snapshots are required in this case). So if you want to move the logic of collecting invalidation keys to be asynchronous (outside of the db transaction), the database must support point-in-time snapshot reads in general. Depending on the cache schema, versioning primitives you have, and concrete use cases, it's entirely possible that you can keep your caches updated without a database that supports point-in-time snapshots. But if we are talking about general cases, a DB with point-in-time (recent) snapshot reads is needed.

**How about write amplifications?**

Now in order to keep caches updated, you might be paying a huge cost due to write amplification (async or sync); and it can be unbounded. E.g. if I make friends with Alice who has a million friends, a million cache entries at least would be updated.

This is where we need to make tradeoffs, depending on the workload (read and write). Sometimes, using TTL (completely getting rid of the invalidation) would help save capacity.

**What if the source of truth of the data is not in a database?**

Let's assume data is stored in just plain files. If data is immutable, there's no invalidation needs to be performed. So we are good. If data in files can be mutated, and you are not using a database, you must provide a subset of database features to synchronize accesses at the very least. From there you can bootstrap "ordering" of your data mutation stream, and you can order cache invalidation events in the same total order.

## Closing

Once you treat caching as a distributed system problem (as it is), it's about tradeoffs. You tradeoff between cache consistency and coordination overhead that works better for your workload and requirements. What's hard about caching in general is about making this tradeoff. Now once you have made a tradeoff, and having the most brilliant cache invalidation protocol implemented (say it's provably correct via TLA+), caches still tend to be inconsistent because handling cache invalidations correctly is very hard. This [post](https://engineering.fb.com/2022/06/08/core-data/cache-invalidation/) goes into details about why _implementing_ cache invalidation is hard and how to manage the complexity.

