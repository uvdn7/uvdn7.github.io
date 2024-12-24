---
layout: post
title: Fiber-compatible Future
date: '2021-10-19 13:54:05'
tags:
- folly
- fiber
- future
- async
---

`folly`'s Future API is Fiber-compatible.

> Calling get() on a folly::Future object will only suspend the calling fiber-task. It won't block the system thread, letting it process other tasks. – [https://github.com/facebook/folly/blob/master/folly/fibers/README.md](https://github.com/facebook/folly/blob/master/folly/fibers/README.md)

But how does it actually work? E.g. what happens when calling `folly::futures::sleep(1).get()` inside a Fiber?

When you call `get()` on a future, it will first `wait()` [https://github.com/facebook/folly/blob/master/folly/futures/Future-inl.h#L2259-L2265](https://github.com/facebook/folly/blob/master/folly/futures/Future-inl.h#L2259-L2265) until the promise is fulfilled.

<!--kg-card-begin: markdown-->

    template <class T>
    Try<T> SemiFuture<T>::getTry() && {
      wait();
      auto future = folly::Future<T>(this->core_);
      this->core_ = nullptr;
      return std::move(std::move(future).result());
    }

<!--kg-card-end: markdown-->

`wait()` is implemented by waiting on a `baton` of `FutureBatonType` which is `typedef`ed to `folly::fibers::Baton` ([https://github.com/facebook/folly/blob/master/folly/futures/Future-inl.h#L2078-L2099](https://github.com/facebook/folly/blob/master/folly/futures/Future-inl.h#L2078-L2099)).

<!--kg-card-begin: markdown-->

    template <class FutureType, typename T = typename FutureType::value_type>
    void waitImpl(FutureType& f) {
      if (std::is_base_of<Future<T>, FutureType>::value) {
        f = std::move(f).via(&InlineExecutor::instance());
      }
      // short-circuit if there's nothing to do
      if (f.isReady()) {
        return;
      }
    
      Promise<T> promise;
      auto ret = convertFuture(promise.getSemiFuture(), f);
      FutureBatonType baton;
      f.setCallback_([&baton, promise = std::move(promise)](
                         Executor::KeepAlive<>&&, Try<T>&& t) mutable {
        promise.setTry(std::move(t));
        baton.post();
      });
      f = std::move(ret);
      baton.wait();
      assert(f.isReady());
    }

<!--kg-card-end: markdown-->

This code says

- if `f` is a child of `Future<T>` type, just execute the future [inline](https://github.com/facebook/folly/blob/master/folly/executors/InlineExecutor.h#L42)
- it schedules a callback on the future, which `post`s the `baton` when ready
- it then waits on the baton

Now it's clear that why `folly::Future` is fiber-compatible. The short answer is that because it uses `fiber::baton` for waiting. When a `fiber::baton` is waited upon, it will try to get the thread-local fiber manager `FiberManager::getFiberManagerUnsafe()` if it has one. Then it can perform cooperative multi-tasking.

