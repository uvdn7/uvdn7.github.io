---
layout: post
title: Bet using a token bucket
date: '2022-10-06 14:11:17'
tags:
- retry
- distributed-system
---

_This post is a slightly expanded version of my earlier [tweet](https://twitter.com/uvdn7/status/1552322815365357569)._

In software systems, "betting" is actually a common practice. One of the most common bets people place is _retry_. When the first attempt doesn't work, we can just "try it again". Retry is a bet – you are betting on the next attempt being successful. You have a cost to pay, when you lose the bet. The "cost" is additional load to a likely already overloaded system. At this point, a positive feedback loop can be formed (overload -\> retry -\> overload), and bring the system to its knees. &nbsp;

Marc Brooker has a very nice blog post talking about using token buckets as a retry strategy – [https://brooker.co.za/blog/2022/02/28/retries.html](https://brooker.co.za/blog/2022/02/28/retries.html). With an adaptive retry strategy, when the system is generally healthy (low error rate), we can retry requests and provide higher availability; when the system is overloaded or unhealthy (high error rate), it would adapt to a lower retry rate to avoid further overloading the system.

The same idea can be extended to any "bets", not just retries. I would like to propose a token bucket based betting strategy. Namely, we deposit `X` tokens on successful bets, and take `Y` tokens on each placed bet.

I want to share a real world example where this type of betting strategy is useful. Say, a client polls the database periodically to get a sense of its staleness. Say the database is at position 100 (GTID in MySQL can be a real world example) when the client last polled 10 seconds ago. We know the database position is always moving forward in time, at an unpredictable pace. Now you have a client query requesting the database position to be \>= 101. You have a bet to make. You can bet on the db to be actually ahead of position 101 now (10 seconds after last time it was polled). Extra work is the cost of losing the bet. E.g. you might still have to failover to the primary db after "wasting" a query to the local db replica.

By using the token bucket betting strategy, we can reduce the number of primary db failover requests in steady state (when local db is not lagging much behind the primary), _and_ bound the added load when local db replica's replication lag is abnormally high.

