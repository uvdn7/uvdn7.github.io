---
layout: post
title: Unary Plus
date: '2020-03-10 16:36:11'
tags:
- cpp
---

A few times I ran into the following issue

<!--kg-card-begin: markdown-->

    uint8_t a = 0;
    std::cout << "a = " << a << std::endl;

Guess what would it print?

    a = 

What happened here is that it printed NULL (0) in ASCII. `uint8_t` is just a typedef of `unsigned char`.

    typedef unsigned char uint8_t; //https://code.woboq.org/gtk/include/stdint.h.html#uint8_t

In the past, I worked around the problem by casting the variable to e.g. `uint16_t` and be done with it. Then I found out this:

    uint8_t a = 0;
    std::cout << "a = " << +a << std::endl;

    a = 0

You can print `+a`, which would promote the type to an integral type. Then you get `0` printed out.

According to the reference ([https://en.cppreference.com/w/cpp/language/operator\_arithmetic](https://en.cppreference.com/w/cpp/language/operator_arithmetic)),

> The built-in unary plus operator returns the value of its operand. The only situation where it is not a no-op is when the operand has integral type or unscoped enumeration type, which is changed by integral promotion, e.g, it converts char to int or if the operand is subject to lvalue-to-rvalue, array-to-pointer, or function-to-pointer conversion.

<!--kg-card-end: markdown-->