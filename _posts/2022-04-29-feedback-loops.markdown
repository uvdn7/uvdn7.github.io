---
layout: post
title: Feedback Loops
date: '2022-04-29 13:55:24'
tags:
- feedback-loops
- hardware
---

Any complex systems e.g. an economy seem to have feedback loops. Digital systems are no exceptions. Feedback loops add significant amount of complexity to a system, which makes it harder to manage. However, a philosophical point might be that it's exactly the presence of feedback loops that makes a powerful system.

To make digital computers, the ones that depend on 0s and 1s, Turing Complete, a conditional jump instruction is indispensable. We need an additional flag register to support conditional jump. As pointed out by [Ben Eater](https://www.youtube.com/watch?v=ObnosznZvHY), adding the flag register in hardware, creates a feedback loop in the computer.

![feedback](/assets/feedback_loop.jpg)

The instruction decoder sends a control signal to the flag register, so it knows when to set the register and when not to. In return, the flag register affects the next instruction to be decoded and executed. It's a physical feedback loop exists in every digital system.

If we go a level deeper, and look at a latch – e.g. an SR latch, there's a feedback loop as well. Storing a single bit in a register requires a feedback loop.

![sr_latch](/assets/sr-latch.png)

It's even on a children's book.

![latch](/assets/latch.JPG)

Feedback loop is hard to deal with, and adds a lot of complexity. It is also exactly the feedback loop that empowers and enables powerful and complex systems. Without feedback loops there will be no digital computers, or modern economy, or democracy. We need to learn how to live with this complexity, and try our best to manage it.

