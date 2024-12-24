---
layout: post
title: FLP to Paxos is like CAP to Spanner
---

_Spanner pushes the limit of CAP in the same way that Paxos pushes the limit of FLP. Mostly because not being CAP-available or not guaranteed to terminate by the definition in FLP is just not a big deal in practice._

* * *

FLP is such a beautiful theorem with a concise and elegant [proof](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwjcwKX855_6AhXIEmIAHU_6ANwQFnoECAUQAQ&url=https%3A%2F%2Fgroups.csail.mit.edu%2Ftds%2Fpapers%2FLynch%2Fjacm85.pdf&usg=AOvVaw3cwr00WJuxyxJUTcm4rELk). The entire paper reasons distributed systems as state machines. If you like TLA+, this model will come very naturally. The theorem states that no consensus protocol is "guaranteed" to terminate with just one faulty process. We will explain it in details later about the conditions under which the theorem applies, and what it means to have one faulty process.

As much as I like the theorem and its proof, I don't think it's very useful in practice – in a sense that it's unlikely we will find a situation in which we will say X is impossible because of FLP (or it can be reduced to FLP).

## Proof in English

For a consensus protocol to work, the state will eventually transit from a bivalent one to an univalent one. However, there doesn't exist a message (state transition) that's decisive – moving all possible states before it from a bivalent one to univalent ones. This means, there exists some previous states/steps without `e` that `e(c)` is still bivalent. This holds for any `e`.

Then we just need to keep playing those previous steps repeatedly (plus `e`); and the state machine will never reach a decision.

## Paxos

Paxos is published after FLP.

