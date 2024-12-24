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

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2022/04/IMG_4707.jpg" class="kg-image" alt loading="lazy" width="2000" height="1500" srcset=" __GHOST_URL__ /content/images/size/w600/2022/04/IMG_4707.jpg 600w, __GHOST_URL__ /content/images/size/w1000/2022/04/IMG_4707.jpg 1000w, __GHOST_URL__ /content/images/size/w1600/2022/04/IMG_4707.jpg 1600w, __GHOST_URL__ /content/images/size/w2400/2022/04/IMG_4707.jpg 2400w" sizes="(min-width: 720px) 720px"></figure>

The instruction decoder sends a control signal to the flag register, so it knows when to set the register and when not to. In return, the flag register affects the next instruction to be decoded and executed. It's a physical feedback loop exists in every digital system.

If we go a level deeper, and look at a latch – e.g. an SR latch, there's a feedback loop as well. Storing a single bit in a register requires a feedback loop.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2022/04/RS-and-or-flip-flop.png" class="kg-image" alt loading="lazy" width="318" height="152"></figure>

It's even on a children's book.

<figure class="kg-card kg-image-card"><img src=" __GHOST_URL__ /content/images/2022/04/F1A7A9F8-CAAF-4777-96CB-6CCB3AEC63C3.JPG" class="kg-image" alt loading="lazy" width="2000" height="2667" srcset=" __GHOST_URL__ /content/images/size/w600/2022/04/F1A7A9F8-CAAF-4777-96CB-6CCB3AEC63C3.JPG 600w, __GHOST_URL__ /content/images/size/w1000/2022/04/F1A7A9F8-CAAF-4777-96CB-6CCB3AEC63C3.JPG 1000w, __GHOST_URL__ /content/images/size/w1600/2022/04/F1A7A9F8-CAAF-4777-96CB-6CCB3AEC63C3.JPG 1600w, __GHOST_URL__ /content/images/size/w2400/2022/04/F1A7A9F8-CAAF-4777-96CB-6CCB3AEC63C3.JPG 2400w" sizes="(min-width: 720px) 720px"></figure>

Feedback loop is hard to deal with, and adds a lot of complexity. It is also exactly the feedback loop that empowers and enables powerful and complex systems. Without feedback loops there will be no digital computers, or modern economy, or democracy. We need to learn how to live with this complexity, and try our best to manage it.

