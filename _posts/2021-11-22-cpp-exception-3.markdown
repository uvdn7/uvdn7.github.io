---
layout: post
title: C++ exception (3) – catching an exception
date: '2021-11-22 20:52:46'
tags:
- exception
- cpp
---

This is the third post of a series that I am making on C++ exceptions.

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-1/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (1) — zero-cost exception handling</div>
<div class="kg-bookmark-description">This is the first post of a series I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making on C++ exceptions. C++ exception (2) — throwing an exceptionThis is the second post of a series that I am making</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1498084393753-b411b2d26b34?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDF8fGZhc3R8ZW58MHx8fHwxNjM3MzM0MjYy&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure><figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-2/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (2) — throwing an exception</div>
<div class="kg-bookmark-description">This is the second post of a series that I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making on C++ exceptions. C++ exception (1) — zero-cost exception handlingThis is the first post of a series I am making</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1522978413910-e3889a1343db?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDJ8fHRocm93fGVufDB8fHx8MTYzNzUxNDYzMQ&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure><figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href=" __GHOST_URL__ /cpp-exception-3/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">C++ exception (3) – catching an exception</div>
<div class="kg-bookmark-description">This is the third post of a series that I am making on C++ exceptions. https://blog.the-pans.com/cpp-exception-1/ https://blog.the-pans.com/cpp-exception-2/ Personality RoutineTo understand how an exception is caught in C++, we need to understand a little more about personality routine. Personality…</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src=" __GHOST_URL__ /favicon.ico" alt=""><span class="kg-bookmark-author">Lu's blog</span><span class="kg-bookmark-publisher">Lu Pan</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://images.unsplash.com/photo-1590502160462-58b41354f588?crop=entropy&amp;cs=tinysrgb&amp;fit=max&amp;fm=jpg&amp;ixid=MnwxMTc3M3wwfDF8c2VhcmNofDN8fGNhdGNofGVufDB8fHx8MTYzNzU1OTc5MA&amp;ixlib=rb-1.2.1&amp;q=80&amp;w=2000" alt=""></div></a></figure>
## Personality Routine

To understand how an exception is caught in C++, we need to understand a little more about _personality routine_. Personality routine at a high level is basically a language specific callback that allows an unwind library (language agnostic) to perform the unwind process. The Itanium ABI specifies the personality routine interface, which looks like the following. (The personality routine implemented in `libc++` and `libstdc++` (`libc++abi` and `libsupc++` to be precise) is called `__gxx_personality_v0`.)

<!--kg-card-begin: markdown-->

        _Unwind_Reason_Code (*__personality_routine)
    	    (int version,
    	     _Unwind_Action actions,
    	     uint64 exceptionClass,
    	     struct _Unwind_Exception *exceptionObject,
    	     struct _Unwind_Context *context);

<!--kg-card-end: markdown-->

As you might have guessed, `_Unwind_Reason_Code` is an `enum` that lets the personality routine communicate the result back to `libunwind` (or other implementation of the base unwind APIs). E.g. `_URC_END_OF_STACK` means no handler was found by the personality routine, `_URC_HANDLER_FOUND` means a handler was found, etc. `version` here is used to make sure `libunwind` and `__gxx_personality_v0` are speaking the same protocol. `_Unwind_Action` is another `enum` that defines the set of jobs a personality routine needs to support (e.g. phase-one search, phase-two cleanup, etc.). The `exceptionClass` is not very interesting in this context; we will ignore it for now. `_Unwind_Exception` is the type for exception objects.

<!--kg-card-begin: markdown-->

        struct _Unwind_Exception {
    	    uint64 exception_class;
    	    _Unwind_Exception_Cleanup_Fn exception_cleanup;
    	    uint64 private_1;
    	    uint64 private_2;
        };

<!--kg-card-end: markdown-->

All four fields are mostly not useful for us actually. The `exception_class` is mostly not very interesting. The `exception_cleanup` function pointer is only useful if you expect a different runtime to clean up the exception object (e.g. catch an exception thrown from Java in C++). We are not supposed to use `private_1` or `private_2` at all. Where does the actual information about the C++ exception go? E.g. the C++ exception type, etc.?

> ... a typical runtime such as the C++ runtime will add language-specific information used to process the exception. This is expected to be a contiguous area of memory after the `_Unwind_Exception` object ... &nbsp;([https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html))

That makes sense. Since C++ exception allocation needs to go through the C++ ABI ` __cxa_allocate_exception` anyway, C++ runtime can allocate a few more bytes then and `__ gxx_personality_v0` can safely assume there are more bytes at the end of `_Unwind_Exception` for C++ specific information.

In terms of `_Unwind_Context`, it's an opaque struct. `struct _Unwind_Context;` is all what it is. We don't need a `struct` definition if it's opaque. It's entirely language specific. The C++ ABI can use it however it wants.

Here's what it actually looks like in `libc++abi`,

<!--kg-card-begin: markdown-->

    // https://github.com/llvm-mirror/libcxxabi/blob/master/src/cxa_personality.cpp#L945-L966
    
    ...
    __gxx_personality_v0
    #endif
    #endif
                        (int version, _Unwind_Action actions, uint64_t exceptionClass,
                         _Unwind_Exception* unwind_exception, _Unwind_Context* context)
    {
        if (version != 1 || unwind_exception == 0 || context == 0)
            return _URC_FATAL_PHASE1_ERROR;
    
        bool native_exception = (exceptionClass & get_vendor_and_language) ==
                                (kOurExceptionClass & get_vendor_and_language);
        scan_results results;
        if (actions & _UA_SEARCH_PHASE)
        {
        ...

<!--kg-card-end: markdown-->
## Install the personality routine

Now we have the interface and the implementation in `libc++abi` (or `libsupc++`). We need to hook the personality routine and the unwind library up. This is done through the `.eh_frame` section in DWARF. Basically frame information and personality routines are stored in `.eh_frame` and that's where `libunwind` finds them. The frame information helps `libunwind` find the previous frame, especially in the absence of frame pointers. DWARF itself is very complicated, and I dare not look at it.

## Catch an exception

We've roughly covered ` __cxa_allocate_exception` and `__ cxa_throw` which are both built on top of the base unwind API. Now the last two ` __cxa_*` functions in our code are `__ cxa_begin_catch` and `__cxa_end_catch`. Both take an exception object pointer. The main job for this pair of functions is to maintain reference count on the exception object. Refcount is needed because the same exception can be re-thrown, and handled by another exception handler — a catch clause.

As part of the unwind process, `libunwind` would work with the personality routine in `libc++abi`, read DWARF in the object file, it will find an exception handler (if it exists) and transition control (set program counter) to the exception handler. The actual transition requries setting registers and managing context. `libunwind` provides a set of APIs for these tasks.

At a very high level, this is how C++ exception works.

