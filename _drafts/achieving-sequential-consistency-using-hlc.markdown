---
layout: post
title: Providing Sequential Consistency in a distributed system using HLC
---

## What is Sequential Consistency

Leslie Lamport defined sequential consistency in his 1979 paper [How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/How-to-Make-a-Multiprocessor-Computer-That-Correctly-Executes-Multiprocess-Programs.pdf). He uses “sequentially consistent” to imply…

> … the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

([https://jepsen.io/consistency/models/sequential](https://jepsen.io/consistency/models/sequential))

> Informally, sequential consistency implies that operations appear to take place in some total order, and that that order is consistent with the order of operations on each individual process. ([https://jepsen.io/consistency/models/sequential](https://jepsen.io/consistency/models/sequential))

## Why do we care
<figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-05-at-5.59.12-PM.png" class="kg-image" alt loading="lazy" width="1080" height="810" srcset=" __GHOST_URL__ /content/images/size/w600/2023/01/Screen-Shot-2023-01-05-at-5.59.12-PM.png 600w, __GHOST_URL__ /content/images/size/w1000/2023/01/Screen-Shot-2023-01-05-at-5.59.12-PM.png 1000w, __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-05-at-5.59.12-PM.png 1080w" sizes="(min-width: 720px) 720px"><figcaption>https://jepsen.io/consistency</figcaption></figure>

Sequential Consistency is stronger than Causal Consistency but weaker than Linearizability.

## Sequential Consistency via HLC

## TLA+
