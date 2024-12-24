---
layout: post
title: TLA+ series (3)
---

> TLA+ has constructs for writing theorems and formal proofs of those theorems.

> TLAPS is a tool for checking those proofs.

`constant RM` in TLA+ every value is a set.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-11-at-5.18.04-PM.png" class="kg-image" alt loading="lazy" width="2000" height="343" srcset=" __GHOST_URL__ /content/images/size/w600/2023/01/Screen-Shot-2023-01-11-at-5.18.04-PM.png 600w, __GHOST_URL__ /content/images/size/w1000/2023/01/Screen-Shot-2023-01-11-at-5.18.04-PM.png 1000w, __GHOST_URL__ /content/images/size/w1600/2023/01/Screen-Shot-2023-01-11-at-5.18.04-PM.png 1600w, __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-11-at-5.18.04-PM.png 2378w" sizes="(min-width: 720px) 720px"></figure><figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-11-at-5.19.13-PM.png" class="kg-image" alt loading="lazy" width="1656" height="630" srcset=" __GHOST_URL__ /content/images/size/w600/2023/01/Screen-Shot-2023-01-11-at-5.19.13-PM.png 600w, __GHOST_URL__ /content/images/size/w1000/2023/01/Screen-Shot-2023-01-11-at-5.19.13-PM.png 1000w, __GHOST_URL__ /content/images/size/w1600/2023/01/Screen-Shot-2023-01-11-at-5.19.13-PM.png 1600w, __GHOST_URL__ /content/images/2023/01/Screen-Shot-2023-01-11-at-5.19.13-PM.png 1656w" sizes="(min-width: 720px) 720px"></figure>

TCTypeOK is an invariant.

[RM -\> ...] is a set, and rmState is an element in the set. rmState is a single array. so [RM- \> ...] represents _all_ possible arrays indexed by elements of RM.

