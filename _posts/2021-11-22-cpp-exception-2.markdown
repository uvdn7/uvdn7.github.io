---
layout: post
title: C++ exception (2) — throwing an exception
date: '2021-11-22 05:29:55'
tags:
- exception
- cpp
---

This is the second post of a series that I am making on C++ exceptions.

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-1/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (1) — zero-cost exception handling</div>
<div class="kg-bookmark-description">This is the first post of a series I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making on C++ exceptions. C++ exception (2) — throwing an exceptionThis is the second post of a series that I am making</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1498084393753-b411b2d26b34?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDF8fGZhc3R8ZW58MHx8fHwxNjM3MzM0MjYy&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure><figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-2/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (2) — throwing an exception</div>
<div class="kg-bookmark-description">This is the second post of a series that I am making on C++ exceptions. The following assumes you have read the first post already. Now we are warmed up with some assembly reading. Let the fun begin. What happens when test doesn’t pass and it throws an exception? func(</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1522978413910-e3889a1343db?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDJ8fHRocm93fGVufDB8fHx8MTYzNzUxNDYzMQ&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure><figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-3/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (3) – catching an exception</div>
<div class="kg-bookmark-description">This is the third post of a series that I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1590502160462-58b41354f588?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDN8fGNhdGNofGVufDB8fHx8MTYzNzU1OTc5MA&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure>

The following assumes you have read the [first post]( __GHOST_URL__ /cpp-exception-1/) already.

Now we are warmed up with some assembly reading. Let the fun begin. What happens when `test` doesn't pass and it throws an exception?

<!--kg-card-begin: markdown-->

    func(bool): # @func(bool)
            ...
            test byte ptr [rbp - 1], 1
            je .LBB0_3 # not jumping this time
            mov edi, 4 # asking __cxa_allocate_exception to allocate one with 4 bytes (to store "1")
            call __cxa_allocate_exception
            ...

<!--kg-card-end: markdown-->

What is this `__cxa_allocate_exception` and what does it do? To understand it, we need to learn a bit about the Itanium C++ ABI.

## Itanium C++ ABI

The [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/) is a language specific (obviously) [ABI]( __GHOST_URL__ /abi/).

> The Itanium C++ ABI is an ABI for C++. &nbsp;As an ABI, it gives precise rules for implementing the language, ensuring that separately-compiled parts of a program can successfully interoperate.

Today we will focus on the [exception handling section](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html). It's part of C++ ABI because we want to be able to catch an exception thrown from a separately compiled library. There needs to be a contract between binaries describing how exceptions should be handled. It has a set of APIs which must be available on any Itanium compatible platforms.

#### Base APIs

The following `_Unwind_*` base APIs are language agnostic. They help perform basic functionalities of exception handling – e.g. actually unwind the stack. Although they have C interfaces, any language conforming to the System V AMD64 ABI can invoke these functions just fine.

<!--kg-card-begin: markdown-->

      _Unwind_RaiseException,
      _Unwind_Resume,
      _Unwind_DeleteException,
      _Unwind_GetGR,
      _Unwind_SetGR,
      _Unwind_GetIP,
      _Unwind_SetIP,
      _Unwind_GetRegionStart,
      _Unwind_GetLanguageSpecificData,
      _Unwind_ForcedUnwind

<!--kg-card-end: markdown-->

`libunwind` is the most popular (mostly language agnostic) implementation of these APIs (and more). There are actually two implementations of `libunwind`. One is the "[official](https://github.com/libunwind/libunwind)"/nongnu continuation of HP's `libunwind`. The other is from LLVM, which Apple made most of the contributions. [LLVM's `libunwind`](https://bcain-llvm.readthedocs.io/projects/libunwind/en/latest/) focuses on implementing the base APIs listed above. I will use code from LLVM's `libunwind` implementation for this post.

The following is the actual [implementation](https://github.com/llvm-mirror/libunwind/blob/master/src/UnwindLevel1.c#L348-L369) of `_Unwind_RaiseException` from `libunwind`.

<!--kg-card-begin: markdown-->

    
    /// Called by __cxa_throw. Only returns if there is a fatal error.
    _LIBUNWIND_EXPORT _Unwind_Reason_Code
    _Unwind_RaiseException(_Unwind_Exception *exception_object) {
      _LIBUNWIND_TRACE_API("_Unwind_RaiseException(ex_obj=%p)",
                           (void *)exception_object);
      unw_context_t uc;
      unw_cursor_t cursor;
      __unw_getcontext(&uc);
    
      // Mark that this is a non-forced unwind, so _Unwind_Resume()
      // can do the right thing.
      exception_object->private_1 = 0;
      exception_object->private_2 = 0;
    
      // phase 1: the search phase
      _Unwind_Reason_Code phase1 = unwind_phase1(&uc, &cursor, exception_object);
      if (phase1 != _URC_NO_REASON)
        return phase1;
    
      // phase 2: the clean up phase
      return unwind_phase2(&uc, &cursor, exception_object);
    }

<!--kg-card-end: markdown-->

You might have heard about personality routine, which is basically a set of callbacks that are language specific which an unwind library can invoke. For `libstdc++` and `libc++`, the personality routine is called `__gxx_personality_v0`.

#### Unwind process

As you can see from the above code snippet. `_Unwind_RaiseException` has two phases – `unwind_phase1` and `unwind_phase2`, dictated by the Itanium ABI. The first phase walks the stack and tries to find a frame that can handle the exception. It just tries to find a handler in phase one. No action is performed. The stack is not yet actually unwound. If no handler was found, the program should terminate. The second phase performs the actual unwind and cleanup. Here's the [code](https://github.com/llvm-mirror/libunwind/blob/3e6ec2ae9afaa3683269b690612f84d907943ea2/src/Unwind-seh.cpp#L344-L360) in `libunwind` about what happens when the first phase fails to find a handle.

<!--kg-card-begin: markdown-->

    /// Called by \c __cxa_throw(). Only returns if there is a fatal error.
    _LIBUNWIND_EXPORT _Unwind_Reason_Code
    _Unwind_RaiseException(_Unwind_Exception *exception_object) {
      _LIBUNWIND_TRACE_API("_Unwind_RaiseException(ex_obj=%p)",
                           (void *)exception_object);
    
      // Mark that this is a non-forced unwind, so _Unwind_Resume()
      // can do the right thing.
      memset(exception_object->private_, 0, sizeof(exception_object->private_));
    
      // phase 1: the search phase
      // We'll let the system do that for us.
      RaiseException(STATUS_GCC_THROW, 0, 1, (ULONG_PTR *)&exception_object);
    
      // If we get here, either something went horribly wrong or we reached the
      // top of the stack. Either way, let libc++abi call std::terminate().
      return _URC_END_OF_STACK;
    }

<!--kg-card-end: markdown-->

So if an exception is not caught, according to the Itanium ABI, the stack is actually not unwound, which means no cleanup is performed. This can be easily verified by a simple program that throws an uncaught exception. Interestingly, C++ standard explicitly says unwind behavior in this case is [implementation dependent](https://en.cppreference.com/w/cpp/error/terminate).

> an [exception is thrown](https://en.cppreference.com/w/cpp/language/throw) and not caught (it is implementation-defined whether any stack unwinding is done in this case)

Basically C++, the language, doesn't require stack to be unwound when an exception is not caught. You can have your C++ ABI that actually unwinds the stack on uncaught exception, and still be consistent with the C++ standard. But your C++ ABI won't be compatible with the Itanium ABI. In practice, it's unlikely to matter but I found this difference interesting.

I wonder why the Itanium ABI designed a two-phase unwind process. The documentation says it allows other languages to support [resumptive exception handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html). [Here](http://christian.heinleins.net/apples/except/) Christian provided an example of what resumptive exception handling could look like. It's not a feature that C++ supports. It sounds like a cool feature. Maybe one day it will be added to C++, as if the language is not complicated enough.

It's also interesting that `libunwind` itself actually doesn't terminate the program. It just returns `_URC_END_OF_STACK`and lets `libc++abi` call `std::terminate`.

#### `__cxa_*` C++ APIs

`libc++abi` is LLVM's implementation (`libsupc++` within `libstdc++` from GCC serves similar functionality) of the C++ ABI as part of the Itanium ABI. The C++ ABI covers a range of things e.g. the layout of exception objects, control transfer, etc. But we will just focus on the APIs. C++ ABI functions all start with `__cxa_` prefix. If you remember from our [assembly dump](__GHOST_URL__/cpp-exception-1/) from the first post, we have seen a few of them already — `__cxa_allocate_exception`, ` __cxa_throw`, `__ cxa_begin_catch` and ` __cxa_end_catch`. C++ ABI is built on top of the base unwind APIs. E.g. `__ cxa_throw` calls `_Unwind_RaiseException`.

It's interesting that `libc++abi` has APIs such as `__cxa_new_handler`, that is not present in `libstdc++`. I found an[email from Apple](https://itanium-cxx-abi.github.io/cxx-abi/cxx-abi-dev/archives/2010-May/002322.html) explaining why it's added. First of all, `libc++abi` is introduced so that libraries built with `libc++` and `libstdc++` can interoperate. But there's a problem when it comes to `std::set_new_handler` (and a few others), where it assumes there exists a global handle. Now if a library compiled with `libstdc++` calls `std::set_new_handler` to function A, another library compiled with `libc++` calls `std::set_new_handler` to function B, and these two libraries are linked together. There's no contract about how they can consolidate the conflict. One solution is to factor out the actual implementation of `std::set_new_handler` and link the implementation with the two libraries only once, which is essentially what Apple proposed and implemented in `libc++abi`.

## Throwing an exception

Now we can get back to the original code where an exception is thrown.

<!--kg-card-begin: markdown-->

    func(bool): # @func(bool)
            ...
            test byte ptr [rbp - 1], 1
            je .LBB0_3
            mov edi, 4
            call __cxa_allocate_exception
            mov rdi, rax
            mov dword ptr [rdi], 1
            mov esi, offset typeinfo for int
            xor eax, eax
            mov edx, eax
            call __cxa_throw

<!--kg-card-end: markdown-->

It calls ` __cxa_allocate_exception` to get a 4-byte-sized exception object. It then initializes the higher 4 bytes of the exception object with `1`, and the lower 4 bytes with `typeinfo` representing `int` – we need the `typeinfo` when it comes to checking if an exception can be caught by a handler or not. This is why C++ exceptions need to be copy/move constructable. When you do `throw ex;`, `ex` is _always_ moved/copied to this another exception object that's allocated by `__ cxa_allocate_exception` first. For most exceptions, chances are it's doing a copy.

Then it calls `__cxa_throw` with `rdi` (first function argument) set to be the initialized exception object. This kick-starts the two-phase unwind process we described earlier. It has to walk the stack at runtime, because it's dynamic, which is slow. Because frame pointer by default is omitted, `libunwind` depends on information in DWARF (if you are on Linux) to unwind the stack, which means more page misses and an even slower unwind process.

So when people say "C++ exceptions are slow", what they really meant is that _throwing an exception in C++ is slow_.

