---
layout: post
title: Proper Boost Installation on OSX
date: '2019-02-12 21:51:08'
tags:
- cpp
- boost
---

`boost` is THE C++ library out there. Many features were introduced in `boost` before adopted in C++ standard. I want to have it installed on my local Mac laptop, so I can easily play with it.

## `homebrew` formula not working

`brew install boost` is the first thing I did and it looked like it "worked". But it didn't install things like `libboost_fiber`. It's simply missing from `/usr/local/Cellar/boost/<boost_version>/lib`. It's not obvious why that's the case by reading the [formula](https://github.com/Homebrew/homebrew-core/blob/master/Formula/boost.rb). I don't know Ruby as well. `homebrew` does't seem to be the best option here, especially, I want to have more control over the installation.

## System default c++ compiler

The default compiler that comes with XCode is `clang`. You can find `gcc`, `g++`, `clang`, `clang++` driver binaries in `/usr/bin`. They all do pretty much the same thing. There's no difference between `g++` and `clang++` for example.

    /u/bin ❯❯❯ g++ -x c++ -v -E /dev/null
    Apple LLVM version 10.0.0 (clang-1000.11.45.5)
    Target: x86_64-apple-darwin17.7.0
    Thread model: posix
    InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
     "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang" -cc1 -triple x86_64-apple-macosx10.13.0 -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -E -disable-free -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mthread-model posix -mdisable-fp-elim -fno-strict-return -masm-verbose -munwind-tables -target-cpu penryn -dwarf-column-info -debugger-tuning=lldb -target-linker-version 409.12 -v -resource-dir /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0 -stdlib=libc++ -fdeprecated-macro -fdebug-compilation-dir /usr/bin -ferror-limit 19 -fmessage-length 123 -stack-protector 1 -fblocks -fencode-extended-block-signature -fobjc-runtime=macosx-10.13.0 -fcxx-exceptions -fexceptions -fmax-type-align=16 -fdiagnostics-show-option -fcolor-diagnostics -o - -x c++ /dev/null
    clang -cc1 version 10.0.0 (clang-1000.11.45.5) default target x86_64-apple-darwin17.7.0
    ignoring nonexistent directory "/usr/include/c++/v1"
    #include "..." search starts here:
    #include <...> search starts here:
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1
     /usr/local/include
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0/include
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
     /usr/include
     /System/Library/Frameworks (framework directory)
     /Library/Frameworks (framework directory)
    End of search list.
    # 1 "/dev/null"
    # 1 "<built-in>" 1
    # 1 "<built-in>" 3
    # 373 "<built-in>" 3
    # 1 "<command line>" 1
    # 1 "<built-in>" 2
    # 1 "/dev/null" 2

vs.

    /u/bin ❯❯❯ clang++ -x c++ -v -E /dev/null
    Apple LLVM version 10.0.0 (clang-1000.11.45.5)
    Target: x86_64-apple-darwin17.7.0
    Thread model: posix
    InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
     "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang" -cc1 -triple x86_64-apple-macosx10.13.0 -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -E -disable-free -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mthread-model posix -mdisable-fp-elim -fno-strict-return -masm-verbose -munwind-tables -target-cpu penryn -dwarf-column-info -debugger-tuning=lldb -target-linker-version 409.12 -v -resource-dir /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0 -stdlib=libc++ -fdeprecated-macro -fdebug-compilation-dir /usr/bin -ferror-limit 19 -fmessage-length 123 -stack-protector 1 -fblocks -fencode-extended-block-signature -fobjc-runtime=macosx-10.13.0 -fcxx-exceptions -fexceptions -fmax-type-align=16 -fdiagnostics-show-option -fcolor-diagnostics -o - -x c++ /dev/null
    clang -cc1 version 10.0.0 (clang-1000.11.45.5) default target x86_64-apple-darwin17.7.0
    ignoring nonexistent directory "/usr/include/c++/v1"
    #include "..." search starts here:
    #include <...> search starts here:
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1
     /usr/local/include
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0/include
     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
     /usr/include
     /System/Library/Frameworks (framework directory)
     /Library/Frameworks (framework directory)
    End of search list.
    # 1 "/dev/null"
    # 1 "<built-in>" 1
    # 1 "<built-in>" 3
    # 373 "<built-in>" 3
    # 1 "<command line>" 1
    # 1 "<built-in>" 2
    # 1 "/dev/null" 2

Well, which C++ standards does it support?

    /u/bin ❯❯❯ clang++ -x c++ -std=c++20 -v -E /dev/null
    ...
    error: invalid value 'c++20' in '-std=c++20'
    note: use 'c++98' or 'c++03' for 'ISO C++ 1998 with amendments' standard
    note: use 'gnu++98' or 'gnu++03' for 'ISO C++ 1998 with amendments and GNU extensions' standard
    note: use 'c++11' for 'ISO C++ 2011 with amendments' standard
    note: use 'gnu++11' for 'ISO C++ 2011 with amendments and GNU extensions' standard
    note: use 'c++14' for 'ISO C++ 2014 with amendments' standard
    note: use 'gnu++14' for 'ISO C++ 2014 with amendments and GNU extensions' standard
    note: use 'c++17' for 'ISO C++ 2017 with amendments' standard
    note: use 'gnu++17' for 'ISO C++ 2017 with amendments and GNU extensions' standard
    note: use 'c++2a' for 'Working draft for ISO C++ 2020' standard
    note: use 'gnu++2a' for 'Working draft for ISO C++ 2020 with GNU extensions' standard

Nice, it supports up to C++20!

## Install Boost

Follow the instructions from Boost website, [https://www.boost.org/doc/libs/1\_69\_0/more/getting\_started/unix-variants.html#easy-build-and-install](https://www.boost.org/doc/libs/1_69_0/more/getting_started/unix-variants.html#easy-build-and-install). Since `boost` is a head-only library, which means you get most of the features without having to "install" anything. But I want `fiber`, which depends on `Boost.context`; so I have to build it.

However when you simply do `b2 install` it will print you something like

    Performing configuration checks
    
        - default address-model : 64-bit
        - default architecture : x86
        - C++11 mutex : no
        - lockfree boost::atomic_flag : yes
        - Boost.Config Feature Check: cxx11_auto_declarations : no
        - Boost.Config Feature Check: cxx11_constexpr : no
        - Boost.Config Feature Check: cxx11_defaulted_functions : no
        - Boost.Config Feature Check: cxx11_final : no
        - Boost.Config Feature Check: cxx11_hdr_mutex : no
        - Boost.Config Feature Check: cxx11_hdr_tuple : no
        - Boost.Config Feature Check: cxx11_lambdas : no
        - Boost.Config Feature Check: cxx11_noexcept : no
        - Boost.Config Feature Check: cxx11_nullptr : no
        - Boost.Config Feature Check: cxx11_rvalue_references : no

Wait, I thought we support all the way to c++20? Well not by default. We have to set `cxxflags` so the compiler would know to use the latest standard.

`./b2 cxxflags="-std=c++14" install` looks much better

    Performing configuration checks
    
        - default address-model : 64-bit
        - default architecture : x86
        - C++11 mutex : yes
        - lockfree boost::atomic_flag : yes
        - Boost.Config Feature Check: cxx11_auto_declarations : yes
        - Boost.Config Feature Check: cxx11_constexpr : yes
        - Boost.Config Feature Check: cxx11_defaulted_functions : yes
        - Boost.Config Feature Check: cxx11_final : yes
        - Boost.Config Feature Check: cxx11_hdr_mutex : yes
        - Boost.Config Feature Check: cxx11_hdr_tuple : yes
        - Boost.Config Feature Check: cxx11_lambdas : yes
        - Boost.Config Feature Check: cxx11_noexcept : yes
        - Boost.Config Feature Check: cxx11_nullptr : yes
        - Boost.Config Feature Check: cxx11_rvalue_references : yes
        - Boost.Config Feature Check: cxx11_template_aliases : yes
        - Boost.Config Feature Check: cxx11_thread_local : yes
        - Boost.Config Feature Check: cxx11_variadic_templates : yes

## Build

Download `simple.cpp` from [boost/fiber/examples](https://github.com/boostorg/fiber/blob/develop/examples/simple.cpp). And build it with

    $ clang++ -std=c++14 -I /usr/local/opt/boost/include simple.cpp -L /usr/local/opt/boost/lib -lboost_fiber -lboost_context
    $ 

Hooray!

<!--kg-card-end: markdown-->