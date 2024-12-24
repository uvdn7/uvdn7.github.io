---
layout: post
title: run folly tests lol
---

clang++ -std=c++17 -O3 -DNDEBUG ../folly/experimental/test/RefCountBenchmark.cpp ../folly/Benchmark.cpp -lgflags -lglog -lpthread -lfolly -ldouble-conversion -lboost\_regex -ldl -lfmt -I ../ -o ref\_count\_bench && ./ref\_count\_bench

