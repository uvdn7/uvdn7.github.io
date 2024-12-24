---
layout: post
title: C++ Type Erasure and `std::function`
date: '2021-04-15 15:49:20'
tags:
- cpp
---

You have heard about Type Erasure of C++. You probably know `std::function` is a classic usage of the pattern. But what is Type Erasure in C++? It's effectively trying to achieve Duck Typing, which is common in languages like Python.

<!--kg-card-begin: markdown-->

    
    # Notice that there's no relationship between Bar1 or Bar2
    class Bar1:
      def doSomething():
        print("bar1")
        
    class Bar2:
      def doSomething():
        print("bar2")
    
    def foo(bar):
      # works on _any_ bar that has `doSomething()` defined
      bar.doSomething()
    
    # you can do things like
    foo(Bar1())
    foo(Bar2())
    for x in [Bar1(), Bar2()]:
      foo(x)

<!--kg-card-end: markdown-->

So we want to do something like this in C++, without forgoing type safety!

## Need to define the interface

If we are not forgoing type safety, we need an interface definition. Something that describes the expectation on the argument. e.g. it has a `doSomething` method defined. Easy.

<!--kg-card-begin: markdown-->

    class AbstractBar {
      virtual void doSomething() = 0;
      virtual ~AbstractBar() = default; // always have a virtual destructor
    };
    
    
    void foo(AbstractBar& bar);

<!--kg-card-end: markdown-->

Now `foo` takes any bars that inherit from `AbstractBar` class. This means we need make `Bar1` and `Bar2` inherit from `Abstractbar`, which can very quickly spin out of hand (one parent class for each interface). Not to mention that Bar1 and Bar2 might be defined in third party libraries that we can't change.

## Template

What's the first thing that comes to mind? Template!

<!--kg-card-begin: markdown-->

    template <typename T>
    void foo(T bar) {
      bar.doSomething(); // done
    }

<!--kg-card-end: markdown-->

The downside of this is that now as a library you expose your implementation to all users (because of template) and if there are a lot of callsites, and it slows down the compilation time (sometimes dramatically). There's also no types erased here. Not the Duck Typing we wanted. We want the type T to be actually captured/erased. E.g. we want to have a `vector` of _things_ that we can call `doSomething` upon, and they can have different behaviors (different underlying types).

## Erasing Types

What's THE hammer that can solve most problems in computer science? Yes, **another level of indirection**.

Something like ...

<!--kg-card-begin: markdown-->

    class Bar : public AbstractBar {
    public:
      template <typename T>
      Bar(T t) {}
      void doSomething() override {
        t_.doSomething(); // hmm... but how do we store t_
      }
    };
    
    void foo(Bar bar);

<!--kg-card-end: markdown-->

If we have a `Bar` class that has a templated constructor, it can _capture/erase_ the type T. And if we make `Bar` a subclass of `AbstractBar` that seems to solve the problem. Well not quite. We need to store `t_` of type `T`, which can only be done with a templated class.

## Take three

Well, let's just add _another_ level of indirection.

<!--kg-card-begin: markdown-->

    template <typename T>
    class BarWrapper : public AbstractBar {
    public:
      explicit BarWrapper(T t): t_(t) {}
      void doSomething() override {
        t_.doSomething(); // no vtable lookup here
      }
    private:
      T t_;
    };
    
    class Bar {
    public:
      template <typename T>
      Bar(T t) {
        // type captured/erased here
        t_ = std::make_unique<BarWrapper<T>>(t);
      }
      void doSomething() {
        t_->doSomething(); // vtable lookup here
      }
    private:
      std::unique_ptr<AbstractBar> t_; // now we solved the problem of storing t_ without knowing T
    };
    
    void foo(Bar bar);

<!--kg-card-end: markdown-->

This is it. Now if we pass our `Bar1` to `foo`

- it will first implicitly construct a `Bar` object with a pointer to `BarWrapper<Bar1>`
- when `bar.doSomething()` is called inside `foo`, it will trigger vtable lookup and find `BarWrapper<Bar1>::doSomething`
- which then calls `Bar1::doSomething` which is exactly what we want

## Summary

Well that's a lot of levels of indirection. But the result is beautiful. `void foo(Bar bar)`, which accepts _anything_ that affords `doSomething` (Duck Typing in C++). When `Bar1` gets converted to `Bar`, its type got erased. We can have a `std::vector<Bar>` that stores wildly different things as long as they can afford the `doSomething` action, and it would work as you expect.

I learned a lot about Type Erasure from Author O'Dwyer's post – [https://quuxplusone.github.io/blog/2019/03/18/what-is-type-erasure/](https://quuxplusone.github.io/blog/2019/03/18/what-is-type-erasure/). I highly recommend his posts and cppconf videos as well.

