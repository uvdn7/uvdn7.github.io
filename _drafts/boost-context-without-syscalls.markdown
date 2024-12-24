---
layout: post
title: "`boost::context` without syscalls"
tags:
- boost
- fiber
---

For a long time, I assumed fibers uses `setcontext(2)/getcontext(2)` syscalls for switching context. `folly` uses `boost::context` and `boost::fibers`; here is an example.

<!--kg-card-begin: markdown-->

    
    int main() {
      folly::EventBase evb;
      auto& fiberManager = folly::fibers::getFiberManager(evb);
      folly::fibers::Baton baton;
      fiberManager.addTask([&]() {
        std::cout << "Task 1: start from thread id " << std::this_thread::get_id()
                  << std::endl;
        baton.wait();
        std::cout << "Task 1: after baton.wait()" << std::endl;
      });
    
      fiberManager.addTask([&]() {
        std::cout << "Task 2: start from thread id " << std::this_thread::get_id()
                  << std::endl;
        baton.post();
        std::cout << "Task 2: after baton.post()" << std::endl;
      });
    
      evb.loop();
    
      return 0;
    }

<!--kg-card-end: markdown-->

`strace` shows no `setcontext(2)/getcontext(2)` is getting called. A much smarter ex-colleague and friend told me that it's implemented in assembly – and of course he's right – grep for `.global jump_fcontext`.

Now it begs the question why are `setcontext(2)` and `getcontext(2)` system calls?

