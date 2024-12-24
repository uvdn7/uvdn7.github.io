---
layout: post
title: Beyond Causal Consistency
---

## Beyond Causal Consistency

Linearizability is expensive and often require special hardware to have good performance at scale. A popular design choice made by systems like [CockroachDB](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/) is to capture causality between operations and settle for Causal Consistency.

I think we can do better than Causal Consistency. If &nbsp;

