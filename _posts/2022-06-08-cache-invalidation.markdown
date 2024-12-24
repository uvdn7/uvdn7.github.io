---
layout: post
title: Cache Invalidation
date: '2022-06-08 17:43:34'
tags:
- cache
- consistency
---

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://engineering.fb.com/2022/06/08/core-data/cache-invalidation/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Cache made consistent: Meta’s cache invalidation solution</div>
<div class="kg-bookmark-description">When it comes to cache invalidation, we believe we now have an effective solution to bridge the gap between theory and practice.</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src="https://engineering.fb.com/wp-content/themes/code-fb-com/favicon.ico" alt=""><span class="kg-bookmark-author">Engineering at Meta</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://engineering.fb.com/wp-content/uploads/2022/06/CacheMadeConsistentHero.png" alt=""></div></a></figure>

We wrote a post on Meta/FB's engineering blog on how we manage the complexities of cache invalidation, and make caches more consistent along the way. I believe the methodology described should be applicable to most if not all invalidation-based caches. Cache invalidation might no longer be a hard thing in computer science. By cache invalidation, I am using the following definition from Wikipedia

> Cache invalidation is a process in a computer system whereby entries in a cache are replaced or removed.

It's the smallest unit of function that "cache invalidation" must perform, and nothing more. &nbsp;It assumes we already know when and what to invalidate. The second step – actually doing the cache invalidate locally, remains very challenging in practice. And that's what the post explains and addresses.

Based on the discussions on [Hacker News](https://news.ycombinator.com/item?id=31671252), people have different perceptions about what Phil Karlton was referring to by "cache invalidation" in his famous quote – There are only two hard things in computer science: cache invalidation and naming things. Some people believe cache invalidation is hard because it's hard to figure out _when_ to invalidate. Let's talk about it. Before I start, I want to say it has been an absolute pleasure to have intelligent public civil discourse on HN; and I treasure it. **I should have applied a narrower and more specific definition of cache invalidation in the blog post. I apologize for any confusions it caused.**

EDIT: I share exactly the same view of Marc's [https://twitter.com/MarcJBrooker/status/1534944338341310470](https://twitter.com/MarcJBrooker/status/1534944338341310470) except that I didn't name the tradeoff between coordination and consistency as a cache invalidation problem. This post was made before his Tweet and it's a coincidence that what I talked about here happens to be a verbose version of what Marc put in a few tweets. I wouldn't have written this post if I had saw his tweets, as I couldn't do it any better.

* * *

In its most generic form, a cache can store arbitrary materialization from any data source. There would be dependencies. E.g. `y = f(x)`; if `x` changes, `y`, in cache, needs to be changed/invalidated as well. However, this problem is not unique to cache. E.g. you can store a table of "friends" (x in our example) in one database, and another table "friends-of-friends" (y in our example) in the same database. Now to keep both tables consistent, one essentially needs to update them transactionally (in one ACID transaction would be the easiest), which implies both schemas are made known to whoever is interacting with the database. I will say the `y = f(x)` problem is **not** what makes _cache_ invalidation hard. If you are specifically interested in _when_ and _how_ to invalidate a cache reliably, this [post](/when-and-how-to-invalidate-cache/) might be helpful.

What's unique to cache, unlike a database, is that data can be cached everywhere and anywhere. A client can come run arbitrary queries and store the materialization on client. No schema about the cached data is made known to the data source. In this case, it's not about if it's a hard problem or not. In the most generic form of cache and data source, it appears to be obvious that cache invalidation can't possibly work in this setup because

- Client comes and goes at will, there's no deterministic cluster membership (e.g. a known list of clients that should be invalidated)
- Data stored in client is _not_ schematized, as it's not made known to the data source. There's no generic way to figure out if a mutation would affect a certain cache entry on client or not. Of course, one can just invalidate everything, but client membership is still indeterministic. 

My point is that it appears to be obvious that a solution to this problem – knowing who/when to invalidate cache, in its _most generic form_ (i.e. indeterministic clients, unschematized queries, etc.), doesn't exit. I assume people won't refer to a clearly unsolvable problem is hard. It would be like saying, "making a piece of paper black and white at the same time is hard". Based on this assumption, I will say this specific flavor of an unsolvable problem (mostly due to a lack of information) is _not_ what makes cache invalidation _hard_.

Now let's talk about a more specific version of the problem, when the schema (or query used) for cache data is made known to the data source/invalidation pipeline. In that case, when updating the data source, in order to keep caches consistent, you essentially need to transact (cross system transaction) on both the data source and cache(s). Usually cache has more number of replicas, I don't think running this type of transactions synchronously is practical at scale.

What happens if we don't transact on both systems (the data source, and cache) synchronously? Well, now whenever the asynchronous update pipeline performs the computation, it's done against a moving data source (not a snapshot of when the write was committed). Now let's say the data source is Spanner, which provides point-in-time snapshots. On Spanner commit you can get a commit time (TrueTime) back. Now using that commit time, to read the data and compute cache update asynchronously can be done. Because the materialization in cache don't take writes themselves, so updates to them are essentially performed blindly and can be ordered by the commit time (TrueTime).

This is still pretty useful, as the only requirement we added is that the schema (query used) for cache data is made known to the data source (e.g. database). This is hard to build, but _doable_. **If this is your definition of "cache invalidation", I agree it's hard; and the blog post linked doesn't address this problem.** And the difficulty is coming from the fact that you are dealing with a distributed system, the moment you start using cache.

I would also like to point out that this architecture has a hidden cost – unbounded write amplification. Because solving the "when/who" to invalidate problem is essentially equivalent to doing "joins/filters (building a relation in general)" on the write/invalidation path. The number of cache entries you need to invalidate can be unbounded (not bounded by the number of indices, but a function of data in the database instead). I imagine it would be hard to provision the invalidation pipeline. This is where it gets into trade-offs. E.g. by just caching the primary index and secondary indices (not relations) you can get very far as well (build a relation on read). But it does depend on the workload; and here it lies the tradeoffs.

At the end of the day, after we figure out "when/who" to invalidate on mutation, we still need to actually process the invalidate on the cache server (if you read earlier paragraphs, you would see why I excludes caching on client here). It's actually not too hard to figure out "when/who" to invalidate if the data model is relatively simple.

The action of literally processing cache invalidate requests on cache host is very hard to get right. When you talk about cache in this context, you are dealing with a fault tolerant distributed system. Things can go wrong in a million different ways for a distributed system. It's easy to _feel_ like this is an easy problem. But it's actually a very hard problem (as explained in details in the linked blog), and any cache service with invalidation _has_ to deal with it. This is what, I think, makes cache invalidation hard; and _this_ is what I claim we have managed in the linked blog post. I believe it benefits all invalidate-based cache services, because, again, the process of actually processing an invalidate request can't be avoided.

If I learned anything from this public civil discourse on Hacker News is that

- we should avoid putting the power/responsibility of cache invalidation in the hands of cache users but rather in cache service owners (and database owners);
- we should provide guard rails via better abstractions, simpler data models (e.g. a graph data model);
- we should avoid caching relations (separate materializations) but prefer caching indices instead, so the write/invalidation amplification is bounded. 

Once stepped outside of these practices, sometimes for good reasons, like using `goto`, things tend to get out of hand pretty quickly.

