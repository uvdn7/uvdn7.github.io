---
layout: post
title: Integer Division in Bucketed Timeseries
tags:
- cpp
---

Folly, Facebook's open-source C++ library, has a class template called `folly::BucketedTimeSeries`, which handles stats aggregation over a duration. For example, one can have a `BucketedTimeSeries` for a minute
