---
layout: post
title: Static  storage duration and `folly::Singleton`
tags:
- cpp
- folly
---

When it comes to singleton in C++, Meyers Singleton is probably the most widely adopted. It looks something like,

<!--kg-card-begin: markdown-->

    MyClass& getInstance() {
        static auto myObj = MyClass();
        return myObj;
    }

<!--kg-card-end: markdown-->

It's great. It's lazily initialized, it doesn`t suffer from the [SIOF](https://en.cppreference.com/w/cpp/language/siof) problem even if another dependent static object is defined in another translation unit. But how about destruction?

* * *

singleton vault is the class that holds all the singletons

singleton holder

registrationComplete() is called in folly/Init.cpp (where it handles other singleton setup stuff)

> all folly::Singletons are destroyed after main function returns, which is before the destruction of any Meyers Singletons

how's function local static objects destructed

- objects with static storage duration are cleaned up at std::exit [https://en.cppreference.com/w/cpp/utility/program/exit](https://en.cppreference.com/w/cpp/utility/program/exit)

> Returning from the [main function](https://en.cppreference.com/w/cpp/language/main_function), either by a `return` statement or by reaching the end of the function performs the normal &nbsp;function termination (calls the destructors of the variables with &nbsp;automatic [storage durations](https://en.cppreference.com/w/cpp/language/storage_duration)) and then executes `std::exit`, passing the argument of the return statement (or ​0​ if implicit return was used) as `exit_code`.

which is called after `main` returns

- [https://stackoverflow.com/questions/19744250/what-happens-to-a-detached-thread-when-main-exits](https://stackoverflow.com/questions/19744250/what-happens-to-a-detached-thread-when-main-exits)
- basically it's not fun to figure out. Let's not use detach, and have clean shutdown process.

singleton base is not templated itself so the single vault can keep a vector of base objects

the creation of the `Singleton` object triggers the registration

how it handles "fork"

- `detail::AtFork::registerHandler`

what's leaky singleton all about?

- it calls Meyers singleton leaked

Why do we need to support `fork`? Well, web servers share `listen()` socket, so one can have multiple running web server processes on the same host, binding to one port.

