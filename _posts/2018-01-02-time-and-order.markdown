---
layout: post
title: Time and Order
date: '2018-01-02 07:32:36'
tags:
- distributed-system
---

_This is the first of two notes of Lamport's Time-Clock paper<sup class="footnote-ref"><a href="#fn1" id="fnref1">[1]</a></sup>._

### Abstract

Order is a more basic concept than Time. And it's critical to how we reason. However physical time, which even though gives us total ordering on all events, cannot be observed within the system. So, we introduced a Logical Clock which satisfies Clock Condition, which is observable in the system.

### Time

Time is something we experience in our physical world. E.g. we saw there's day and night, and we call the interval between two instants when the sun is at its highest "a day". We noticed certain repetitive physical processes that repeat themselves in equal intervals (e.g. the interval between two consecutive instants when a pendulum reaches its highest) and we use that to be certain basic unit to quantify time i.e. a second. And a clock was made. `Clock` (or a function from `event` to `time`) was made from our observation of the physical world.

When we say an event _A_ happened at time _3:15pm_, what we actually mean is that when the event happened, our clock read time _3:15pm_. And if another event _B_ happened at time _3:14pm_, we can say it happened _before_ _A_. Assuming all the observers are stationary to each other, we all share the same notion of time and space.

But we don't need to read clock to say event _B_ happened before event _A_. The experience of `ordering` came before the `clock`. In fact, if we rephrase the "definition" of a "second", "a second is the time between the instant the pendulum is at its highest and the one right `after`". **The concept of `order` in which events occur is more basic than the concept of `time`.**

### Order

`Ordering` (order here is the same as `happen-before`/`happen-after` relationship between events) is fundamental to our way of thinking in system. E.g. you can only withdraw $500 `before` your bank balance drop below $500. Your remaining balance `after` the transaction will be whatever was there subtract $500. We have total ordering for events in a single process system, consistent with our experience of time. All events in everyone's notion of time and space are totally ordered by time. Hence, it's easier for us to reason about a single process system than a distributed system.

In a distributed system, it's hard for all events to be total ordered in a way that's consistent with physical time, _that is observable in the system_. E.g. event `A` in `p0` happened before event `B` in `p1`, but it's hard for a process in the system to observe (or verify) this ordering. (It's possible if all processes in the system have perfectly synchronized clocks. But the condition itself is hard to satisfy.) If the ordering is not observable in the system, the information cannot be used in the system. Hence, the `happened-before` relationship, consistent with physical time, is not very useful.

### Happened-before

The approach we are going to take is to first define a partial order that's consistent with physical time, which is observable. On top of it, we define a total order, which is consistent with the partial order, which is also observable. Notice that this doesn't mean the total order is consistent with the total order from physical time.

So, we have to define a `happened-before` relationship that's observable in the system, in order for it to be useful.

We define `happened-before` (denoted as `->`) as a binary relation _over all events in a system_ that satisfies

- if _A_ and _B_ happened on the same process and _A_ happened before _B_ (notice the "before" here is the _a priori_ total ordering within a process), `A -> B`
- if _A_ is the sending of a message on _p0_ and _B_ is the receiving of the same message on _p1_, `A -> B`
- if `A -> B` and `B -> C`, then `A -> C`

According to the definition of [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set), you can see that `->` is an irreflexive partial order. Obviously, this partial order is consistent with the total ordering of physical time. Because the sending of a message must happen `before` the receiving of the message under the total order of physical time.

If we want to achieve the goal above, we need to make sure that this `order` is observable in the system. We achieve this by introducing `clocks` to the system. Clock is a function of event to time (doesn't have to be physical time). Let's say event _A_ happened at time `C(A)` and _B_ happened at time `C(B)`.

If we can _create_ a `clock` that satisfies

> if `A->B` then `C(A) < C(B)`

then the partial ordering we defined is observable _in the system_ using this clock. Remember this is the main reason why we can't use physical clock to order events at the first place.

> if `A->B` then `C(A) < C(B)`

is called **Clock Condition**

Asking the converse condition to hold would be too strict (as it would be hard to come up with a clock that satisfies the condition). It would mean two concurrent events must have happened at same time.

The algorithm to satisfy the clock condition is pretty straightforward ([Logical Clock](https://en.wikipedia.org/wiki/Lamport_timestamps)).

* * *
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item">
<p>Leslie Lamport. Time, clocks and the ordering of events in a distributed system. <em>Communications of the AMC</em>, 21(7):558-565, July 1978. <a href="#fnref1" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section><!--kg-card-end: markdown-->