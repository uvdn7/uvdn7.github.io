---
layout: post
title: Caches should always be close to clients
tags:
- cache
---

The title of this post might be an example of truism, but it might be less obvious sometimes. Let's start with an example. Say you started with a fairly common website setup – a web server, a cache, and a database. Over time, your website becomes very popular, you added more database shards, and cache replicas and scaled the website horizontally. Things are fairly straightforward until you realize that cache invalidation has a super-linear cost curve as a function of the number of cache replicas.

`cache_invalidation_count = mutation_count * number_of_cache_replicas`

Let's make a fairly reasonable assumption that the number of cache replicas grow linearly to the number of mutations. Invalidation count hence is O(n^2) – n being the number of cache replicas.

Practically, you can't scale your website unbounded in a single physical data center, which has its physical limits (e.g. limited compute, power, etc.). And running everything in a single physical data center is not a good idea from a disaster readiness perspective. So, usually you will spread your client workload, web servers and cache replicas across multiple physical data centers. Say, you have

