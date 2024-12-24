---
layout: post
title: Build `folly::coro` with GCC
date: '2021-05-31 06:19:17'
tags:
- folly
- coroutine
- cpp
---

You have heard about Coroutine in C++, and you want to use it. There're two coroutine implementations that are considered most mature - `cppcoro` and `folly::coro`. They are written by the same guy - Lewis Baker. He's brilliant, and you shoud watch his cppcon talk on [structured concurrency](https://www.youtube.com/watch?v=1Wy5sq3s2rg) if you haven't already. `folly::coro` provides more features than `cppcoro`, so let's try it out.

First, let's have a toy program.

<!--kg-card-begin: markdown-->

    #include <folly/experimental/coro/Task.h>
    #include <folly/experimental/coro/BlockingWait.h>
    #include <folly/futures/Future.h>
    #include <folly/executors/GlobalExecutor.h>
    #include <folly/init/Init.h>
    #include <iostream>
    
    folly::coro::Task<int> slow() {
      std::cout << "before sleep" << std::endl;
      co_await folly::futures::sleep(std::chrono::seconds{1});
      std::cout << "after sleep" << std::endl;
      co_return 1;
    }
    
    int main(int argc, char** argv) {
      folly::init(&argc, &argv);
      folly::coro::blockingWait(
          slow().scheduleOn(folly::getGlobalCPUExecutor().get()));
      return 0;
    }
    

<!--kg-card-end: markdown-->

Our goal is to compile this and run the program successfully.

## Build `folly` with GCC and stdlibc++

First of all, you want to get `folly` the library and build it. Folly doesn't have a stable ABI, so you can just "install" it and among all of its dependencies into your project directory. Now after you've cloned the [github repo](https://github.com/facebook/folly), before you run the `./build.sh` command (according to the README), we need to do a few more steps. That's why even if you already have folly built and installed, you would likely have to do it again with proper compiler flags.

First, switch your compiler to GCC, if you are using Clang. The steps with Clang are slightly different.

<!--kg-card-begin: markdown-->

    export CC=/usr/bin/gcc
    export CXX=/usr/bin/g++

We also need to add the following flag to GCC so it will build `folly` with coroutine support.

    export CXXFLAGS=-fcoroutine # it would be -fcoroutine-ts for clang

Then run

    ./build.sh --install-dir /your/project/dir/folly/

The script will download all folly dependencies, e.g. boost, fmt, gflags, etc. and build folly with GCC.

<!--kg-card-end: markdown-->
## Build our program with GCC and stdlibc++
<!--kg-card-begin: markdown-->

    g++ coro.cpp -std=c++17 -fcoroutines -I./lib/folly/include ./gcc-folly/folly/folly/lib/libfolly.a -lpthread -lm -ldl -lglog -lgflags -levent -ldouble-conversion ./gcc-folly/folly/fmt-jw5zFPx9bcovXWARXzsj4Dm5Slb5s8WHOv-hm0osfrA/lib64/libfmt.a -lunwind -liberty -lboost_context

I have gcc 10. And the above command compiles and successfully links against libstdc++ that comes with gcc 10. Notice that I staticly linked the `fmt` that was downloaded with the folly build script. Otherwise I got linker errors. I guess `fmt` doesn't have a stable ABI either.

<!--kg-card-end: markdown-->

And voila!

<!--kg-card-begin: markdown-->

    ~/p/coro ❯❯❯ ./a.out
    before sleep
    after sleep

<!--kg-card-end: markdown-->
## Failed attampt at building `folly` with Clang and libc++

First of all, you want to delete everything if your folly directory that's listed in `.gitignore`. Specifically, `folly-config.h` which during the build process sets many macros. You don't want your previous GCC build to set macros that won't work for your clang build.

<!--kg-card-begin: markdown-->

    export CC=/usr/bin/clang
    export CXX=/usr/bin/clang++
    export CXXFLAGS="-stdlib=libc++ -fcoroutines-ts -I/usr/local/lib/c++/v1"
    export LDFLAGS="-lc++abi"

The exact command might differ on your computer. The point is to construct the compiler and linker flags that folly can be properly compiled (e.g. use libc++ and its standard headers, and libc++abi, etc.)

<!--kg-card-end: markdown-->

Now folly is built with Clang. Let's build our program.

<!--kg-card-begin: markdown-->

    ~/p/coro ❯❯❯ clang++ coro.cpp -std=libc++ -std=c++2a -fcoroutines-ts -I./clang-folly/folly/include -I/usr/local/include/c++/v1 -lpthread -lm -ldl -lglog -lgflags -levent -ldouble_conversion ./clang-folly/fmt-DykEYK9RumG6QnGTKWi-w5f6emSSD
    hccE-zkSgDUDJI/lib64/libfmt.a -lunwind -liberty -lboost_context -lc++abi 2>&1                                                                                                                                                                 
    In file included from coro.cpp:1:                                                                                                                                                                                                             
    In file included from ./clang-folly/folly/include/folly/experimental/coro/Task.h:19:                                                                                                                                                          
    In file included from /usr/local/include/c++/v1/exception:81:                                                                                                                                                                                 
    In file included from /usr/local/include/c++/v1/cstdlib:85:                                                                                                                                                                                   
    In file included from /usr/local/include/c++/v1/stdlib.h:93:                                                                                                                                                                                  
    /usr/bin/../lib64/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../include/c++/10.2.0/stdlib.h:38:12: error: no member named 'abort' in namespace 'std'                                                                                             
    using std::abort;                                                                                                                                                                                                                             
          ~~~~~ ^  

<!--kg-card-end: markdown-->

`stdlib.h` from libc++ has `include_next <stdlib.h>` which means it will try to include the next `stdlib.h` header from the system include path. Let's add `-v` flag to Clang.

<!--kg-card-begin: markdown-->

    #include <...> search starts here:                                                                                                                                                                                                            
     ./clang-folly/folly/include                                                                                                                                                                                                                  
     /usr/local/include/c++/v1                                                                                                                                                                                                                    
     /usr/bin/../lib64/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../include/c++/10.2.0                                                                                                                                                              
     /usr/bin/../lib64/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../include/c++/10.2.0/x86_64-pc-linux-gnu                                                                                                                                          
     /usr/bin/../lib64/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../include/c++/10.2.0/backward                                                                                                                                                     
     /usr/local/include                                                                                                                                                                                                                           
     /usr/lib/clang/11.1.0/include                                                                                                                                                                                                                
     /usr/include 

<!--kg-card-end: markdown-->

So the "next" `stdlib.h` comes from gcc here. Let's try remove that by setting `-stdlib++-isystem` flag of Clang to `/usr/local/include/c++/v1`, where it has the libc++ headers. Here's what I've got,

<!--kg-card-begin: markdown-->

    ~/p/coro ❯❯❯ clang++ coro.cpp -std=libc++ -std=c++17 -fcoroutines-ts -I./clang-folly/folly/include -stdlib++-isystem /usr/local/include/c++/v1 -lpthread -lm -ldl ./clang-folly/glog-nY6-9ZKI5XRaJI_NUlE6f0fGTadcztXmsJ3jiy0ev3o/lib64/libglog.so ./clang-folly/gflags-VRES6w4l-hYsLHiL7oeMF11RzCanjyGDFLXG4bsP2v8/lib/libgflags.a -levent ./clang-folly/double-conversion-ofzjwl4IFoHrti1JqsbRSL-zDJzjsTRYzMzaXHpAbfY/lib/libdouble-conversion.a ./clang-folly/fmt-DykEYK9RumG6QnGTKWi-w5f6emSSDhccE-zkSgDUDJI/lib64/libfmt.a -lunwind -liberty -lboost_context -lc++abi ./clang-folly/folly/lib/libfolly.a /usr/local/lib/libc++.a 2>&1 | head
    /usr/bin/ld: ./clang-folly/folly/lib/libfolly.a(Demangle.cpp.o): in function `folly::demangle(char const*, char*, unsigned long)':
    /home/lupan/folly/folly/Demangle.cpp:165: undefined reference to `cplus_demangle_v3_callback'
    /usr/bin/ld: ./clang-folly/folly/lib/libfolly.a(LogConfigParser.cpp.o): in function `double_conversion::DoubleToStringConverter::ToShortest(double, double_conversion::StringBuilder*) const':
    /tmp/fbcode_builder_getdeps-ZhomeZlupanZfollyZbuildZfbcode_builder/installed/double-conversion-ofzjwl4IFoHrti1JqsbRSL-zDJzjsTRYzMzaXHpAbfY/include/double-conversion/double-conversion.h:158: undefined reference to `double_conversion::DoubleToStringConverter::ToShortestIeeeNumber(double, double_conversion::StringBuilder*, double_conversion::DoubleToStringConverter::DtoaMode) const'
    /usr/bin/ld: ./clang-folly/folly/lib/libfolly.a(LogConfigParser.cpp.o): in function `double_conversion::DoubleToStringConverter::ToShortestSingle(float, double_conversion::StringBuilder*) const':
    /tmp/fbcode_builder_getdeps-ZhomeZlupanZfollyZbuildZfbcode_builder/installed/double-conversion-ofzjwl4IFoHrti1JqsbRSL-zDJzjsTRYzMzaXHpAbfY/include/double-conversion/double-conversion.h:163: undefined reference to `double_conversion::DoubleToStringConverter::ToShortestIeeeNumber(double, double_conversion::StringBuilder*, double_conversion::DoubleToStringConverter::DtoaMode) const'
    /usr/bin/ld: ./clang-folly/folly/lib/libfolly.a(LogConfigParser.cpp.o): in function `std:: __1::enable_if<std::is_floating_point<double>::value&&IsSomeString<std::__ 1::basic_string<char, std:: __1::char_traits<char>, std::__ 1::allocator<char> > >::value, void>::type folly::toAppend<std:: __1::basic_string<char, std::__ 1::char_traits<char>, std::__1::allocator<char> >, double>(double, std::__1::basic_string<char, std:: __1::char_traits<char>, std::__ 1::allocator<char> >*, double_conversion::DoubleToStringConverter::DtoaMode, unsigned int)':
    /home/lupan/folly/folly/Conv.h:657: undefined reference to `double_conversion::DoubleToStringConverter::ToFixed(double, int, double_conversion::StringBuilder*) const'
    /usr/bin/ld: /home/lupan/folly/folly/Conv.h:662: undefined reference to `double_conversion::DoubleToStringConverter::ToPrecision(double, int, double_conversion::StringBuilder*) const'
    /usr/bin/ld: ./clang-folly/folly/lib/libfolly.a(Conv.cpp.o): in function `folly::Expected<float, folly::ConversionCode> folly::detail::str_to_floating<float>(folly::Range<char const*>*)':
    

<!--kg-card-end: markdown-->

It doesn't make much sense to me, given I am linking the same double\_conversion that folly is built with. Anyway, maybe later.

