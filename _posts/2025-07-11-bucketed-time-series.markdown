---
layout: post
title: Integer Division in Bucketed Timeseries
tags:
- cpp
---

This post is about a cool piece of integer math code in `folly::BucketedTimeSeries`.

## Context
Folly, Facebook's open-source C++ library, has a class template called `folly::BucketedTimeSeries`, which handles stats aggregation over a duration. For example, one might want to track the number of page views on a website over time. The user can have a `BucketedTimeSeries` for a minute worth of duration, split into 60 buckets. For each page view, we add `1` to the time series. The time-series keeps track of the total number of page views in 60 one-second buckets. If two counter bumps happen within the same bucket, they will be aggregated inside the same bucket. As time moves forward, the time series serves as a rolling window with stats collected in the last minute.

## First Integer (Floor) Division 
To maintain this ring buffer of buckets, one needs to map a timestamp to an index into the ring buffer. The math is

```c++
auto timeIntoCurrentCycle = (time.time_since_epoch() % duration_);
return timeIntoCurrentCycle.count() * buckets_.size() / duration_.count();
```

Nothing too crazy here. Folly is open-sourced. The code can be found [here](https://github.com/facebook/folly/blob/main/folly/stats/BucketedTimeSeries-inl.h#L360). It is worth noting that `duration_.count()` is an integer (`duraion_` is of type `std::chrono::duration`), as well as `buckets_.size()`. So integer division (specifically floor division) takes place here. It rounds down (towards zero) when duration is not divisible by bucket size. That seems reasonable.

## Second Integer (Ceiling) Division
The code sometimes needs to figure out the start time of a given bucket - e.g. the start time of the current (latest) bucket. It is useful e.g. to tell if a new sample still falls into the existing bucket, by simply checking if `now < nextBucketStartTime`. We have seen how to map a timestamp into a bucket index. Here, essentially, we are dealing with a mapping of the opposite direction - mapping a bucket index into time. And here's the code for doing that ([link](https://github.com/facebook/folly/blob/main/folly/stats/BucketedTimeSeries-inl.h#L369-L426))
```c++
(bucket_idx * duration + buckets_.size() - 1)/ bucket_size + durationStart;
```

`(m + n -1)/n` is just performing ceiling division for `m/n`. So the code is essentially
```c++
ceil(bucket_idx * duration / (float)bucket_size) + durationStart;
```
(For simplicity, `duration = duration_.count()` and `bucket_size = buckets_.size()` for the rest of the post.)

`durationStart` captures how many whole durations (`N*duration_.count()`) that has passed. As it's a ring buffer, we are just looking for a `bucket_idx/bucket_size` fraction of a single duration, which is the start time for the bucket. The question is why is it doing ceiling division here? Is it really consistent with the previous floor division? 

## Proof
Turns out the correctness is guaranteed by an invariant `bucket_size <= duration_.count()` - the number of buckets cannot be greater than the number of ticks, which makes sense. Given the invariant, we can prove the correctness of the second ceiling division.

We call the first function that maps timestamp to bucket index `getBucketIdx`. We call the second function that maps a bucket idx to timestamp `getBucketStartTime`.  `startTime` acquired from `getBucketStartTime(idx)` is correct iff `getBucketIdx(startTime)` returns the same bucket index `idx`, and `getBucketIdx(startTime-1)` returns the previous bucket
index `idx-1`. 

`getBucketIdx(getBucketStartTime(idx)) = floor(ceil(bucket_idx *  duration / bucket_size) * bucket_size / duration)`

If duration is divisible by `bucket_size`, the expression equals `bucket_idx`. Otherwise, `ceil(bucket_idx * duration / bucket_size) * bucket_size` is strictly greater than `bucket_idx * duration` and less than
`bucket_idx * duration + bucket_size`.  Since `duration >= bucket_size`, and we perform floor division for `getBucketIdx`, the final expression still equals `bucket_idx`.

`getBucketIdx(getBucketStartTime(idx) - 1) = floor((ceil(bucket_idx *  duration / bucket_size) - 1)* bucket_size / duration)`

`(ceil(bucket_idx * duration / bucket_size) - 1) * bucket_size` falls in `[bucket_idx * duration - bucket_size, bucket_idx * duration)`. The left equality can be achieved when duration is divisible by `bucket_size`. It will always be strictly less than `bucket_idx * duration` again because of the `bucket_size <= duration` invariant. Hence the expression equals `bucket_idx - 1`.
equals bucket_idx - 1.

For posterity, the proof is later committed as comment [here](https://github.com/facebook/folly/blob/main/folly/stats/BucketedTimeSeries-inl.h#L392-L417).
