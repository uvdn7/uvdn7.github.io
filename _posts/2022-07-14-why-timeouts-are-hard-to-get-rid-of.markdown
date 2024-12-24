---
layout: post
title: Why TIMEOUTs are hard to get rid of
date: '2022-07-14 22:21:26'
tags:
- timeout
- distributed-system
---

For the entirety of this post, we assume a non-[real-time system](https://en.wikipedia.org/wiki/Real-time_operating_system). First of all, why do we have TIMEOUT as a type of replies? What if we don't? This means, in some cases, either client or server (or both) can be "blocked" for an unbounded amount of time. With async IO, we don't necessarily have to block any threads, but the "work" is blocked regardless. The keyword here is _unbounded_. This means you don't have an explicit limit – this is a problem. Saying a system without explicit limits is like a guy not knowing how much he can drink until he eventually passes out. Allowing unbounded waiting means the work has to be queued up somewhere, and eventually, it can hit some physical limit (e.g. out of memory), and cause failures.

Now, we added some explicit limit to the system, so we don't have an unbounded queue; but do we have to return TIMEOUTs to the client? Returning TIMEOUTs to clients is, to be fair, not ideal. Especially for write requests (mutations), they may or may not have succeeded. The client doesn't know when looking at a TIMEOUT reply. However it's very hard to hide TIMEOUTs from clients.

In order to hide TIMEOUTs from clients, it implies that we need to figure out whether the request actually succeeded or not. Say there's a mutation "set x=UUID1", and the request timed out. In order to find out if the timed-out request actually succeeded or not, we must query the database, who is the only source that can give us extra information. To make the situation even simpler, we assume there are no racing writes, and we are the only client writing to the database. Say, the database returns x=UUID2. Does this mean the x=UUID1 mutation actually failed? Not really, because the request can still be in-flight in the network, or being processed in the database, etc. Can we wait Y seconds, and try again? But how long should Y be?

Let's say you have a system where each component (servers, switches, databases, etc.) supports killing requests that are created more than Y seconds ago. E.g. if at time T, the database sees a mutation initiated from time T - Y, it can just return an explicit error. Can we then say, if we wait Y seconds, and try to read the database again, we can then tell if the original x=UUID1 mutation succeeded or not?

Not really. Let's examine this statement more closely – "if at time T, the database sees a mutation initiated from time T - Y, it can just return an explicit error". The database obviously has to perform this check before making a commit decision. Say the request passes the check, then it moves to commit the change. The time duration between the "TTL" check and actual commit can be in theory unbounded (because we are not talking about a real-time system). The thread can be descheduled from the CPU after the check but before the commit. Not to mention clock skew. So Y in theory is unbounded.

