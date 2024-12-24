---
layout: post
title: Stack overflow
date: '2019-09-11 23:01:21'
tags:
- cpp
- glibc
---

I recently ran into a program crash. There were some mysteries at the beginning of the debugging process.

#### Mysteries

**Signal handler not called**  
The program crashed with a coredump, according to which, it crashed with SIGSEGV. The mystery is that the code has registered a signal handler, which should get called when SIGSEGV happens and dump useful information in the log. The handler didn't get called.

**Putting `if(false)` mysteriously fixed the crash**  
There's a block of code, that if I put `if(false)` around it, the program would stop crashing. Weird enough if I put `if(foo())` around it, where foo is defined as

    inline bool foo() {return false;}

, the program would still crash.

**Stack size didn't seem over the limit**  
From the coredump, the difference between the stackpointer(`$sp`) of the top frame and the stackpoint of the bottom frame is about 540KB which is less than the stack size limit in place (600KB).

#### Mysteries resolved

At the end, with helps from a few of my friends, we found that the program crashed because it overflowed the stack. The reason why the signal handler was not called is because we have already overflowed the buffer, we can't push more frames onto the stack. The reason why `if(false)` worked but not `if(foo())` is because in certain compiler optimization mode, it's smart enough to know that the code in `if(false)` would never get executed, so the frame size is smaller. But it's not smart enough to know that the same applies to `if(foo())`, even though it's defined in the same translation unit as a free function.

But why the stackpointers in the coredump doesn't add up? So I wrote a small program to reproduce the issue.

###### Example (stackoverflow.cpp)

    void* foo(void*) {
      char a[1024];
      cout << "foo" << endl;
      foo(nullptr);
      return nullptr;
    }
    
    __attribute__ (( __section__ (".failsafe"))) static void
    signalhandler(int, siginfo_t*, void* ptr) {
        cerr << "signal handler called" << endl;
    }
    
    int main() {
      pthread_attr_t attr;
      ::pthread_attr_init(&attr);
      size_t stacksize = 17000;
      
      struct sigaction action;
      memset(&action, 0, sizeof(action));
      action.sa_sigaction = signalhandler;
      action.sa_flags = SA_SIGINFO;
    
      if (sigaction(SIGSEGV, &action, nullptr) < 0) {
        cerr << "failed to register signal handler" << endl;
      }
      
      pthread_t thread;
      pthread_create(&thread, &attr, foo, nullptr);
      pthread_join(thread, nullptr);
      return 0;
    }

This is a simple program that does the followings:

1. create a thread with stack size limited to 17KB
2. run `foo` on the newly created thread and recursive infinitely until it overflows the stack

If you run the program with `clang++ --std=c++11 stackoverflow.cpp -pthread && ./a.out`, you would be able to reproduce the crash. If you look at the coredump, it would look like:

    Program terminated with signal SIGSEGV, Segmentation fault.
    #0 0x0000000000400ae2 in foo () at stackoverflow.cpp:13
    13 cout << "foo" << endl;
    [Current thread is 1 (Thread 0x7f5b4a7ae940 (LWP 46278))]
    (gdb) bt
    #0 0x0000000000400ae2 in foo () at stackoverflow.cpp:13
    #1 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #2 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #3 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #4 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #5 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #6 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #7 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #8 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #9 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #10 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #11 0x0000000000400b0d in foo () at stackoverflow.cpp:14
    #12 0x00007f5b49b5cdd5 in start_thread () from /lib64/libpthread.so.0
    #13 0x00007f5b49885ead in clone () from /lib64/libc.so.6

Notice that it only printed "foo" 11 times and crashed on the 12th time. `foo` has a frame size about 1KB, and our stack size limit is set to 17KB. We are missing about 5KB memory. Where did it go?

I then tried a different syscall `pthread_attr_setstack` instead of `pthread_attr_setstacksize`.

      void *addr = ::mmap(nullptr, stacksize, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANON, -1, 0);
      int rv = ::pthread_attr_setstack(&attr, addr, stacksize);

With this, I got 15 iterations before the crash. This suggests it has something to do with alignment.

Later someone pointed out that if I do

      void* sp;
      int rv = posix_memalign(&sp, sysconf(_SC_PAGESIZE), stacksize);
      pthread_attr_setstack(&attr, sp, stacksize);

I can actually get 18 iterations before the crash.

So it turns out `pthread` [trims the memory size](https://code.woboq.org/userspace/glibc/nptl/allocatestack.c.html#528) if `stackaddr` is not set.

    528: size &= ~__static_tls_align_m1;

This explains why when we use `pthread_attr_setstack` with `stackaddr`, we can get more iterations! It's because our stack size is actually bigger in these cases.

So next time, if you see a mysterious crash, check the stackpointer in the coredump. If the stack size is close to the limit, it crashed probably due to stack overflow.

<!--kg-card-end: markdown-->