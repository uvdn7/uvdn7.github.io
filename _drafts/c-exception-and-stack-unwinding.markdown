---
layout: post
title: C++ Exception and Stack nwinding
---

Questions

- why people always say C++ exception is expensive?
- Zero-cost exceptions
- How do C++ exceptions work?
- How it compares to other languages e.g. Rust &nbsp;[https://news.ycombinator.com/item?id=13272474](https://news.ycombinator.com/item?id=13272474) &nbsp;(and this btw, is why rust would never be faster than C++) Rust doesn't have exceptions.
- How does Java handle exceptions
- what is libunwind and what does it do
- does this have anything to do with the fact that GCC and Clang omits frame pointer?
- C++ exception handling and database read/write optimization (throw is like the rare write path)
- Itanium ABI (c++ agnostic) vs. C++ ABI
- how unwind-step works? DWARF frame section DW\_CFA (dwarf canonical frame point)
- If it depends on DWARF for symbols, would it still work with split-dwarf?
- How does the assembly looks like?
- What is ABI
- what is \_ \_ cxa\_ prefix (www.codesourcery.com/public/cxx-abi/) 

## First principle

- read the assembly and RFM

## reference 

- [https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)
- [https://stackoverflow.com/questions/307610/how-do-exceptions-work-behind-the-scenes-in-c/307716#307716](https://stackoverflow.com/questions/307610/how-do-exceptions-work-behind-the-scenes-in-c/307716#307716) 
- [https://news.ycombinator.com/item?id=13273422](https://news.ycombinator.com/item?id=13273422)
- [https://eli.thegreenplace.net/2012/08/13/how-statically-linked-programs-run-on-linux](https://eli.thegreenplace.net/2012/08/13/how-statically-linked-programs-run-on-linux) 
- AMD64 ABI
- calling conventions and why [https://blog.aaronballman.com/2011/04/the-importance-of-calling-conventions/](https://blog.aaronballman.com/2011/04/the-importance-of-calling-conventions/) 
