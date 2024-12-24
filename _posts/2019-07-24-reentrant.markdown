---
layout: post
title: Re-entrant vs. Thread-safe
date: '2019-07-24 00:30:22'
tags:
- thread-safe
- re-entrant
- cpp
---

These two are very different concepts but can be confusing. re-entrant is used to describe a function in a single-threaded environment, thread-safe in multi-threaded environment on the other hand. A function can be both re-entrant and thread-safe, either re-entrant or thread-safe, or neither.

A function is re-entrant, if it can be interrupted in the middle of an execution, and it's safe to call the same function again in the same thread. The second execution of the function can finish after the first one. Notice how this differs from recursing a function, as in recursion, the latter execution always finishes before the former execution. Re-entering a function is a generalization of recursing a function.

![](/assets/reenter.png)

A function is thread-safe, if multiple threads can execute the same function at the same time safely.

## Examples

#### Not thread-safe, not re-entrant

    int tmp;
    int add10(int a) {
      tmp = a;
      return tmp + 10; // <--- interrupt here
    }

It's not thread-safe because there's a data-race because multiple threads can access `tmp` at the same time. It's not re-entrant because for example:

1. Call `add10(1)`.
2. On the line of return, there is an interrupt, and the signal handler calls `add10(2)`.
3. The `add10(2)` call sets global variable `tmp` to `2`.
4. Now after the signal handler, the first call `add10(1)` resumes and returns `12` instead of `11`. Wrong!

#### Not thread-safe, but re-entrant

    int tmp;
    int add10(int a) {
      tmp = a;
      return a + 10;
    }

It's en-entrant. But it's not thread-safe because the data-race on the shared variable `tmp`. This is a silly example as `tmp` does nothing other than creating a data-race here. But you get the idea.

#### Thread-safe but not re-entrant

    thread_local int tmp;
    int add10(int a) {
      tmp = a;
      return tmp + 10;
    }

It's thread-safe thanks to `thread_local`. But it's not re-entrant just like the first example.

#### Thread-safe and re-entrant

    int add10(int a) {
      return a + 10;
    }

## Why not thread-safe and re-entrant for all?

It's possible to make all functions `thread-safe`. But it comes with performance cost. It's not even possible to make all functions re-entrant. E.g. most of the functions in `libevent` are not re-entrant. As long as a function accesses memories out of its stack, it can be non-reentrant. And there are legit reasons why a function has to access some shared memory, e.g. `malloc` and `free`.

<!--kg-card-end: markdown-->
