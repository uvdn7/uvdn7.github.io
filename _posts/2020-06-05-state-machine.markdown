---
layout: post
title: Every Computer System is a State Machine – TLA+ series (1)
date: '2020-06-05 06:01:30'
tags:
- tla
- state-machine
---

This is the first post of a series of posts I plan to make about TLA+. This entire series is based on Lamport's [TLA+ tutorial](https://lamport.azurewebsites.net/video/videos.html).

#### What is a computer

First, let's take a look at a very philosophical question -- "what is a computer?". I am writing this post on my computer, which has 32 GB of RAM and 8 2.4 GHz cores. Most likely you are reading this post on your computer as well (or smartphone, which is also a computer). We are surrounded by computers nowadays (smart phones, laptops, game consoles, smart speaker, smart home devices, etc.). But **what is a computer**?

Is computer the thing on your desk right now with billions of transistors? Well, that describes _the_ computer on your desk, or more generally a class of computers of this kind. But this certainly doesn't describe all computers. If I just throw a billion transistors in a trash can, the trash can is not a computer.

Is computer a set of transistors wired up in a very specific way? This can describe computers of certain architecture, maybe e.g. x86. But this certainly doesn't describe all computers of different architectures. We need abstraction. Maybe we should not look at the physical presence of a computer. But instead, describe and define a computer by what it can do. You might have guessed it already, **a computer is a [Turing Machine](https://en.wikipedia.org/wiki/Turing_machine)**. And Turing Machine describes and defines all possible behaviors of all computers in the world.

#### What is a Turing Machine

Ben Eater has a very good video explaining [Turing Machine](https://www.youtube.com/watch?v=AqNDk_UJW4k). So any algorithm, program, distributed system, the most complicated system interaction you can ever imagine, can be described as a state machine, which can be implemented by a Turing Machine. This makes sense intuitively as well. Computer, physically, is nothing more than a storage of various states, and combinational logic based on all the states (Program Counter, register values, RAM, Carry Flag, etc.) for state transition.

#### View every system as a state machine

Anything computable can be done by the Turing Machine. And Turing Machine can be described as a state machine. So naturally, a state machine is sufficient to describe anything computable. Behaviors can be described in programming languages, but they are often too concrete. State machine is very flexible and easy-to-use at describing system behaviors.

Let's look at the following tiny example:

    a = get_random_number()
    a += 1

We can describe this program behavior as a state machine of two variables: `a` and `pc` (stands for program counter). An example run can be \<a:10, pc:start\> -\> \<a:11, pc:middle\> -\> \<a:11, pc:done\>. The state machine describing its behavior would look something like:

    if pc == start:
        a = get_random_number()
    else if pc == middle:
        a += 1
    else:
        halt

You should not read this like python which executes the program from top to bottom. And in fact it's not a program that you run at all. It's a spec and it just specifies the state transitions for this state machine.

#### State machine is simpler

Python (replace with your favorite language) can also be used to describe program behaviors. It's just that states in a program are much harder to manage and describe because they are represented differently. For the simple python program above, there are "hidden" states like heap, stack, program counter, etc. States in a state machine are just variables. It's consistent, which makes it easier to use for describing behaviors.

And TLA+ is the language along with a set of tools to specify system behaviors (either on a single host or distributed).

<!--kg-card-end: markdown-->