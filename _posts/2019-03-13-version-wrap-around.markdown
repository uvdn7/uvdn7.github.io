---
layout: post
title: Compare Small Versions that Wrap Around
date: '2019-03-13 05:31:22'
tags:
- version
- cache
---

## Why use small versions

Almost all stateful services (data store, cache, etc.) have versions stored along with each piece of data. The version can be in the form of a timestamp, version (a.k.a logical clock), or even better -- Hybrid Logical Clock.

If it's a data store, HLC is almost always better than timestamp or version. Because HCL is strictly more powerful than either timestamp or version and data stores are usually OK with sparing a few extra bytes to store it.

But there are legitimate cases that one might prefer versions. E.g. you might want to store a small version (2 bytes) in cache instead of HLC. Because a cache service is most likely memory-bound and spending a few extra bytes for every piece of data in cache can be very expensive. Especially if you are mostly caching small data.

## Small Version Comparison

#### It's not straightforward

Let's say we want to only use **two bytes** (16 bits) for each version field. With 16 bits, it can only represent numbers up to `65535`. What happens if you bump the version more than 65535 times? It _overflows_ or wraps around. It goes from `65535` back to `0` again. In normal sense, `0` is less than `65535`. But it shouldn't be the case here. Version is used for ordering events, the even which bumped the version from `65535` to `0`, happened after the event at version `65535`. So, you want `0` to be greater than `65535`. You want `0 > 65535` to return true.

If a stateful service considers version `0` is less than version `65535`, either it is already broken and people just don't realize it, or it will break in the future.

#### Circle instead of axis

The versions instead of increasing on an axis, they go around on a circle.  
 ![](/assets/version1.png)

Comparing two versions on an axis is simple. The one on the right is the greater one. How do we compare two versions on a circle? We can't.

Let's consider the following example. Which one is greater? `A` or `B`?  
 ![](/assets/version2.png)

If we consider `A` is greater, which means it happened _after_ `B`, the gap between `A` and `B` is bigger. If considering `B` greater than `A`, the gap is smaller.  
 ![](/assets/version3.png)
We cannot compare `A` and `B` without making assumptions. But if we assume that we are only comparing events that happened _close_ to each other, we know `B` should be greater than `A`. This is a reasonable assumption to make as versions are often only used to order recent events during a racing window. It's unlikely that more than `2^15` events happened in a small racing window.

#### How to compute it

We are effectively comparing the distance from `B` to `A` and the distance from `A` to `B`. Because it's a circle, if `B` to `A` distance is more than `2^15`, `B` must happen after `A` hence greater.

    B > A iff B + D == A, where D >= 2^15
    B + D == A iff A - B >= 2^15
    A - B >=2^15 iff int16_t(A - B) < 0

So, at the very end, in C++, `version_cmp(A, B)` can be written as `int16_t(A - B) > 0` -- an elegant one liner.

This is how small version comparison works.

<!--kg-card-end: markdown-->
