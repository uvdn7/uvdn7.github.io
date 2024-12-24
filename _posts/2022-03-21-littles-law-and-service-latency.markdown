---
layout: post
title: Little's Law and Service Latency
date: '2022-03-21 15:24:59'
tags:
- queueing-theory
- latency
---

Queueing Theory is very relevant to Computer Science as queues are all over the places – instructions to CPUs, packets to NICs or switches, requests to servers, etc. Little's Law (L=λW) is a really cool theorem because it requires very few conditions which implies its wide application. It works on subsystems of a system; it makes no assumption of the arrival distribution, service order, etc. The only requirement for the equation to hold is that the system must be stationary. A [stationary process](https://en.wikipedia.org/wiki/Stationary_process) is a stochastic process whos CDF is not a function of time. E.g. constant white noise is a stationary process, while queries to the IRS website is certainly not one.

Most software engineering teams working on infrastructure maintain services (e.g. MySQL). Most likely your workload is not a stationary process, but in practice, it might be close enough that Little's Law still applies. There is only one way to find out – verify it using the stats you have. We can measure the number of requests that are "being processed" – in the system(L), measure the client QPS (query per second) – arrival rate (λ) and average latency – how long each request is in the system (W); and see if L=λW holds.

Here, we need to be _precise_ in terms of defining the scope of the system, where we can test if Little's Law applies. Let's say we draw a box around the client, and say that's the system we are studying (you can have a different definition), meaning,

- L is the number of requests that are being processed by _all_ the client;
- λ is the client QPS (query per second);
- W is the average latency for a client request to be processed.

Notice that we are not studying the server _at all._

Measuring the client QPS is usually pretty straightforward, just be careful if you are performing batching. Once we have a precise definition of the system scope, measuring the number of requests in the system (being processed) should be straightforward as well (again be careful about batching). Latency measurement however, can be a little tricky, as its accuracy can suffer from various factors, such as,

- async IO
- threading model
- schedule delay
- etc.

All these factors contribute to latency variance. E.g. a busy host can cause higher latency. But notice that's not the issue here, as long as the actual latency is measuring the time when a request is _in the system_. The best place to measure latency, for the purpose of this study, is the same places where you keep track of the number of requests in your system.

<!--kg-card-begin: markdown-->

    inFlightRequestCounter++;
    startTime = now();
    
    // processing request
    
    inFlightRequestCounter--;
    endTime = now();
    

<!--kg-card-end: markdown-->

If there are more steps/instructions between where you keep track of the requests in your system and where you measure latency, the result will be inaccurate.

Now you have the stats, look at **average** over a long period of time (as Little's Law works on average stats), you can see if you can approximate your workload to a stationary process, where Little's Law applies. If it works well enough, you now have a powerful tool that can describe the behavior of your system. E.g. if average latency doubles, while the qps stays the same, the number of inflight requests (from client's perspective) will double (not just increase, but double).

