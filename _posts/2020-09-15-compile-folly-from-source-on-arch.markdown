---
layout: post
title: Compile `folly` from source on Arch Linux
date: '2020-09-15 06:15:07'
tags:
- folly
- cmake
---

You first need to install all the dependencies as documented on the [folly](https://github.com/facebook/folly) github page,

- fmt
- googletest,
- boost
- gflag
- glog

These are pretty straightforward, as you can build them just fine by following the github page for each project respectively.

However I ran into some issues when building `folly` from source. I did the standard,

    mkdir bin
    cd bin
    cmake ..
    make -j 8

However it complains that gflag is redefined.

    /home/lupan/folly/folly/portability/GFlags.h:66:7: error: redefinition of ‘class google::FlagSaver’           
       66 | class FlagSaver {};                                                                                   
          | ^ ~~~~~~~~                                                                                       
    In file included from /usr/local/include/glog/logging.h:86,                                                   
                     from /home/lupan/folly/folly/concurrency/UnboundedQueue.h:23,                                
                     from /home/lupan/folly/folly/executors/StrandExecutor.h:20,                                  
                     from /home/lupan/folly/folly/executors/StrandExecutor.cpp:17:                                
    /usr/local/include/gflags/gflags.h:278:23: note: previous definition of ‘class google::FlagSaver’             
      278 | class GFLAGS_DLL_DECL FlagSaver {                                                                     
          |       

`folly/portability/GFlags.h` depends on a macro called `FOLLY_HAVE_LIBGFLAGS` to tell if gflag is available or not. But we clearly have just installed gflag. If you grep this macro, you see that it's in the generated `folly-config.h` header file.

Turns out what happened was that I ran `cmake` once before gflag was properly installed in the `folly` directory, and it put a generated `folly-config.h` in `folly/` directly. Because the file is in the `.gitignore` list, I missed it from `git status`. That's why even after `gflag` is properly installed, the `folly-config.h` in `bin` directory was always masked by the previous generated one in `folly/` directory.

The fix is simple. I just deleted the `bin/` directory and ran `cmake` in `folly/` which overwrote the previous bad `folly-config.h` file, and fixed all the problems.

<!--kg-card-end: markdown-->
