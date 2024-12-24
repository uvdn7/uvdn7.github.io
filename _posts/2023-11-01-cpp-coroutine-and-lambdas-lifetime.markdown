---
layout: post
title: C++ coroutine and lambda's lifetime
date: '2023-11-01 02:12:18'
tags:
- cpp
- coroutine
---

We rarely need to worry about lambda's lifetime more than any other objects in C++ until we are dealing with coroutine at the same time. folly's wiki has a good example of why we need to be careful about lambda's lifetime – [https://github.com/facebook/folly/blob/main/folly/experimental/coro/README.md#lambdas](https://github.com/facebook/folly/blob/main/folly/experimental/coro/README.md#lambdas). The tl;dr is that when you have a lambda that returns a coroutine handle, we need to make sure the lambda itself outlives the coroutine handle.

Lambda is just an object with operator overload. The function body (code) has static lifetime. So the only thing that can go wrong is the state captured by the lambda. In other words, if the lamda body does not access the captured variables, the coroutine should work just fine even if the lambda object has already been destroyed. On paper it checks out because the code is always there, and there won't be any machine code generated to access the captures (which are already gone). So it is as if the lambda object were still alive.

      int a = 42;
      auto foo = [a]() -> folly::coro::Task<int> {
        co_return a;
      }(); // <-- the lambda gets destroyed at the ;
      auto res = folly::coro::blockingWait(std::move(foo));

As expected, we got use after free

    ==4058644==ERROR: AddressSanitizer: stack-use-after-scope on address 0x7f4df7be2f70 at pc 0x0000002c3276 bp 0x7ffc6f62b920 sp 0x7ffc6f62b918

Now let's try not accessing the capture

      int a = 42;
      auto foo = [a]() -> folly::coro::Task<int> {
        co_return 42;
      }(); // <-- the lambda still gets destroyed at the ;
      auto res = folly::coro::blockingWait(std::move(foo));

The lambda is still destroyed at the semicolon but this program runs just fine, because the assembly it generates will never access the lambda object – just the code.

