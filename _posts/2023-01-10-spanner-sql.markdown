---
layout: post
title: 'Notes on the Spanner: becoming a SQL system paper'
date: '2023-01-10 18:53:36'
tags:
- database
---

I have wanted to read the paper, [Spanner: becoming a SQL system](https://research.google/pubs/pub46103/) from SIGMOD 2017 for a while and finally got a chance to finish reading it. I have mixed feelings. The paper mostly focuses on the DQL aspect of SQL and only briefly mentioned DML and locking.

I like how the paper goes through several practical techniques in building a distributed SQL query engine.  
- It compiles an optimized query plan by pulling up the Distributed Union operator (and hence pushing down other relational operators such as Filters).  
- It rewrites the query plan aggressive towards using Apply-style joins, and even requires WHERE clauses into multiple correlated self-joins. And the Apply operator itself can be batched and distributed efficiently.  
- It supports a parallel consumer API and promises the result would be the same unordered set as if it's consumed by a single client.  
- Query/key range extraction is done by leveraging a data structure called the Filter Tree, which seems to encode the filter logic with optional pruning.  
- It supports query restart by leveraging "restart tokens" to prevent duplication and ensure forward progress. There's not a lot of details about how it actually works.  
- Testing is done with a random query generation tool which targets the AST.  
- They were in the middle of replacing Spanner's storage engine with a columnar engine, with PAX layout.

They explicitly try to make Spanner work well for both OLTP and OLAP workloads, which is ambitious given Stonebraker famously said that one size does not fit all.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2023/01/324649554_725029758887107_1118418123568972738_n.png" class="kg-image" alt loading="lazy" width="996" height="664" srcset=" __GHOST_URL__ /content/images/size/w600/2023/01/324649554_725029758887107_1118418123568972738_n.png 600w, __GHOST_URL__ /content/images/2023/01/324649554_725029758887107_1118418123568972738_n.png 996w" sizes="(min-width: 720px) 720px"></figure>

The paper has no experiments or benchmark results, presumably because Ressi, their columnar store, was not rolled out yet. Although the paper advocates reducing burdens on customers, it seems to be non-trivial for the customers to set up the schemas optimally and write queries that can be optimized and distributed for better performance, because   
- Distributed Union pull-up only works when the operator is "partitionable";  
- Multiple-consumer API only works when the query plan is root-partitionable;  
- Table-interleaving requires a parent-child relationship between tables, and a child can't belong to multiple parents;  
- complex conditions are excluded from range extraction.

As a result, Spanner engineers have to review schema designs from users.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2023/01/324969681_933601757544951_7352823040086432700_n.png" class="kg-image" alt loading="lazy" width="940" height="450" srcset=" __GHOST_URL__ /content/images/size/w600/2023/01/324969681_933601757544951_7352823040086432700_n.png 600w, __GHOST_URL__ /content/images/2023/01/324969681_933601757544951_7352823040086432700_n.png 940w" sizes="(min-width: 720px) 720px"></figure>

I have never used Spanner myself. I am curious about people's experience from using Cloud Spanner.

