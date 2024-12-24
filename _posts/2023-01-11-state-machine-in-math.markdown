---
layout: post
title: Describing State Machines using Math – TLA+ series (2)
date: '2023-01-11 18:24:27'
tags:
- tla
---

Previous post can be found here – [https://blog.the-pans.com/state-machine/]( __GHOST_URL__ /state-machine/). The entire series is based on Lamport's [TLA+ tutorial](https://lamport.azurewebsites.net/video/videos.html).

In the previous [post]( __GHOST_URL__ /state-machine/), we talked about any system can be described as a state machine. What language should we use to describe state machines? TLA+ uses mathematics, which is precise and forces the users to reason about systems abstractly. &nbsp;

## Describing state machines using math

Let's describe this program (from the previous example), in math. This is what the original program looks like.

<figure class="kg-card kg-code-card"><pre><code class="language-python">a = get_random_number(0, 1000)
a += 1</code></pre>
<figcaption>original program</figcaption></figure>

This is how the state machine looks like described in code (pc stands for program counter).

<figure class="kg-card kg-code-card"><pre><code class="language-python">if pc == start:
    a = get_random_number(0, 1000)
else if pc == middle:
    a += 1
else:
    halt</code></pre>
<figcaption>state machine described in code</figcaption></figure>

This is how the state machine looks like described in math.

<!--kg-card-begin: markdown-->

    
     (pc = "start") ∧ (a' ∈ {0, .., 1000}) ∧ (pc' = "middle")
     ∨
     (pc = "middle") ∧ (a' = a + 1) ∧ (pc' = "done")

<!--kg-card-end: markdown-->

This is not code. `=` is not assignment but equality instead. We're writing a formula relating a, pc, a', and pc'. The formula evaluates to either `true` or `false` based on the value of a, pc, a', and pc'.

For example, for a given state – &nbsp;`a = 1, a' = 4, pc = "middel", pc' = "done"`, the formula would be evaluated to `false` because `a' != a + 1` .

TLA+ supports the following syntactic sugar to make things look more like a bullet list.

<!--kg-card-begin: markdown-->

    
     ∨ ∧ (pc = "start")
       ∧ (a' ∈ RANDOM_NUMBER_SET)
       ∧ (pc' = "middle")
     ∨ ∧ (pc = "middle")
       ∧ (a' = a + 1)
       ∧ (pc' = "done")

<!--kg-card-end: markdown-->
## A complete spec

Usually a state machine has two parts – an initial state, and a state transition formula. Here's the complete TLA+ spec for the previous example.

<!--kg-card-begin: markdown-->

    EXTENDS Integers
    VARIABLES a, pc
    
    
    Init == (pc = "start") /\ (a = 0)
    
    Next == \/ /\ pc = "start"
               /\ a' \in 0..1000 
               /\ pc' = "middle"
            \/ /\ pc = "middle"
               /\ a' = a + 1
               /\ pc' = "done"

<!--kg-card-end: markdown-->

The pretty printed version looks like the following.

![tla](/assets/tla2.png)
