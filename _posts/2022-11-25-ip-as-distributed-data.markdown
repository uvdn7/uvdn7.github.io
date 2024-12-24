---
layout: post
title: IP as distributed data in the cloud
date: '2022-11-25 21:32:57'
tags:
- ip
- anycast
- unicast
- database
- distributed-system
- network
---

I previously wrote a post on reasoning [DNS as a distributed database](/dns/). In the same spirit, today, let's take a look at IP as distributed data (the Internet would be a distributed database in this analogy). This is inspired by a very interesting blog post from Cloudflare, [https://blog.cloudflare.com/cloudflare-servers-dont-own-ips-anymore/](https://blog.cloudflare.com/cloudflare-servers-dont-own-ips-anymore/), talking about how they save egress IP addresses.

IP addresses are just names. Most commonly, a single IP address is mapped to a single physical host. This name-\>host map is stored and distributed among a set of routers. Conceptually, every time one routes a datagram with destination address X, it's conceptually performing a database lookup for key X against this massive distributed database (what we call the Internet). This analogy is not perfect (and it doesn't have to be) as IP is for routing packets instead of storing/fetching values. However, at a certain angle, "sending a datagram to IP address X" is no different than "fetching key X from a distributed system" as _both have to resolve where X is located at_.

In most cases, an IP address is analogous to a single key in our database analogy, as both are mapped to a single entity (an IP address -\> a single host, a key -\> single database row). With Anycast, many hosts can announce the same IP address. This is analogous to a replicated database, with the same value for a given key replicated on multiple hosts. Anycast is similar to "reading" from a replicated database, as you can read from any replica. But how about "writes"? Usually, leaving paxos aside for a second, writes are done against a single database primary. The same applies to egress IP addresses as well. If a host is fetching data from the Internet, the reply must be routed to that specific host. Writes are often the bottleneck for a distributed database, as you can't easily replicate away the scaling problems. Cloudflare was facing a similar scaling problem for egress IP addresses as well.

One way to look at the problem described in the Cloudflare blog post is that it's fundamentally a scaling challenge. We need a unique egress IP address to make sure the reply is routed correctly. Assigning multiple unique IP address to each host (the motivation is described in the blog post) doesn't scale, as IPv4 addresses are expensive (and #tags \* #servers is a large number).

If you have a database that's bottlenecked on write throughput, sharding the database is a very common solution (which comes with a set of challenges). Turns out _the same sharding idea applies to IP as well_. Once you shard a database, a table no longer lives on a single physical host anymore. Similarly, an egress IP can be sharded among a set of hosts, given we have a way to route the datagram to the correct place. Similar to distributing a table across a cluster of hosts, Cloudflare is distributing an IP across multiple hosts – that's how they saved egress IP addresses.

> What we chose instead, is splitting an egress IP across servers by **a port range**.

This is essentially saying, "sharding an IP with port as sharding keys". &nbsp;In this way, an IP is no longer mapped to a single host, or a fixed set of hosts. It's stored in the cloud, abstracted away from the client/sender. Just like when you have a table in DynamoDB, it's not mapped to any physical hosts. It's distributed; everything is in the cloud. IP as well.

