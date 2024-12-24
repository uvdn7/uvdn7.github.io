---
layout: post
title: What Good Are Impossibility Results
date: '2023-04-06 13:37:12'
tags:
- distributed-system
- theory
---

Here is the section 3.5 of Nancy Lynch's 1989 paper – [A Hundred Impossibility Proofs for Distributed Computing](http://groups.csail.mit.edu/tds/papers/Lynch/podc89.pdf), which is profound:

> What good are impossibility results, anyway? They don't seem very useful at first, since they don't allow computers to do anything they couldn't previously.

> Most obviously, impossibility results tell you when you should stop trying to device or improve an algorithm. This information can be useful both for theoretical research and for systems development work.

> It is probably true that most systems developers, even when confronted with the proved impossibility of what they're trying to do, will still keep trying to do it. This doesn't necessarily mean that they are obstinate, but rather that they have some flexibility in their goals. E.g., if they can't accomplish something absolutely, maybe they can settle for a solution that works with "sufficiently high probability". In such a case, the effect of the impossibility result might be to make a system developer clarify his/her claims about what the system accomplishes.

> Proving impossibility results causes us to take a very analytical approach to understanding the area. It causes us to state carefully exactly what assumptions (about the execution environment and the problems) the results depend on. This sort of detailed information does not normally arise from or algorithm or system development work alone.

> Sometimes impossibility proofs lead to interesting work on ways of getting around the inherent limitation. For example, many randomized algorithms have been produced in order to get around the inherent cost previously proved for deterministic and nondeterministic algorithms. Examples include Ben-Or's asynchronous fault-tolerant consensus algorithm in [19], Itai and Rodeh's randomized algorithms for leader election without identifiers in [66] and Feldman and Micali's fast algorithm for synchronous consensus [52].

> The close connections between impossibility proofs and modeling means that impossibility results help in the development of formal models. Models produced for impossibility proofs have many nice features, as I discussed earlier. They are not only useful for proving impossibility results; they also have other uses, such as specification and verification of algorithms and software.

> Finally, I think that an understanding of impossibility results in an area is an important part of understanding the fundamental ideas of that area.

