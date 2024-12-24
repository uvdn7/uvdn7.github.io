---
layout: post
title: Debugging Random Program Behavior
date: '2019-02-05 19:53:47'
tags:
- cpu
- debug
- assembly
- gdb
---

I was looking at a weird coredump the other day. From the core, the program was trying to write to virtual address `0x6` and crashed on `memcpy`.

<!--kg-card-begin: markdown-->

There's a piece of code looks like

    if (a == 1) {
        do_foo();
    } else {
        do_bar();
    }

And from the coredump, `a` is indeed `1`. However the execution took the `else` branch and crashed on `memcpy` in `do_bar`.

<!--kg-card-end: markdown-->
1. Disassembled the code in `gdb` using `disassemble /s`. It's so much better than just `disassemble` or even `disassemble /m`. With link time optimization, more functions are being inlined, which makes reading plain assembly harder. `disassemble /s` would annotate each block of instructions with reference back to the source file. It helps understand the assembly much easier.
2. Read the assembly and find where the bogus address came from. It boiled down to a single instruction `lea 0x2(%r14) %r15`, where `%r15` was supposed to be set to `%r14+2` but it's set to `0x6` instead. `gdb` is able to provide register values for each frame by [unwinding the stack](https://gnu.wildebeest.org/blog/mjw/2007/08/23/stack-unwinding/). So it looks like some kind of CPU/firmware bug.
3. It's running E5-2680 v4, at microcode version `0xb000014` according to `/proc/cpuinfo`.
4. It's Broadwell according to [https://ark.intel.com/products/91754/Intel-Xeon-Processor-E5-2680-v4-35M-Cache-2-40-GHz-](https://ark.intel.com/products/91754/Intel-Xeon-Processor-E5-2680-v4-35M-Cache-2-40-GHz-).
5. Got BIOS version `Version: F06_3B06` from `dmidecode`.
6. Once you have enough keywords, Google is your friend. And... there you go [https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=842796](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=842796)! Looks like I should try update the firmware.
