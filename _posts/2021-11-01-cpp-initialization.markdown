---
layout: post
title: "`struct Foo f;` and `struct Foo f{};` are very different"
date: '2021-11-01 14:04:28'
tags:
- cpp
---

Compare the following two programs, and can you predict what the outputs would be for each?

    // program #1
    struct Foo {
        int x;
    };
    
    void stack() {
        int x = 3;
    }
    
    void bar() {
        struct Foo foo{};
        cout << foo.x << endl;
    }
    
    int main() {
        stack();
        bar();
        return 0;
    }

    // program #2
    struct Foo {
        int x;
    };
    
    void stack() {
        int x = 3;
    }
    
    void bar() {
        struct Foo foo;
        cout << foo.x << endl;
    }
    
    int main() {
        stack();
        bar();
        return 0;
    }

<!--kg-card-end: markdown-->

The only difference between the two is `struct Foo foo{};` vs `struct Foo foo;`. Program #1 would print `0`, while program #2 would print `3` – on gcc11 at least (clang13 produces `4198992`). Credit to my wonderful co-worker for the stack based example.

## Default Initialization

_Default Initialization doesn't always initialize._

`struct Foo f;` performs [Default Initialization](https://en.cppreference.com/w/cpp/language/default_initialization). Since `Foo` here is a POD type, and not an array type, the standard states,

> otherwise, no initialization is performed: the objects with automatic storage duration (and their subobjects) contain indeterminate values.

So `Foo f;` _is_ an initialization, `f` _is_ initialized. Foo's constructor does get called. However, it's _default_ initialized, which in this case, _doesn't_ perform initialization over the storage.

## Direct List Initialization

`struct Foo f{};` on the other hand is a [Direct List Initialization](https://en.cppreference.com/w/cpp/language/list_initialization). The standard states,

> Otherwise, if `T` is an [aggregate type](https://en.cppreference.com/w/cpp/language/aggregate_initialization), [aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization) is performed.

`struct Foo` is an aggregate type. Hence in this case, [aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization) is performed. Given our initializer list is empty, this is what should happen accordingly to the standard,

> If the number of initializer clauses is less than the number of members and bases (since C++17) or initializer list is completely empty, the remaining members and bases (since C++17) are initialized by their [default member initializers](https://en.cppreference.com/w/cpp/language/data_members#Member_initialization), if provided in the class definition, and otherwise (since C++14) &nbsp;copy-initialized from empty lists, in accordance with the usual [list-initialization](https://en.cppreference.com/w/cpp/language/list_initialization) rules (which performs value-initialization for non-class types and non-aggregate classes with default constructors, and aggregate initialization for aggregates). If a member of a reference type is one of these remaining members, the program is ill-formed.

We didn't provide a default member initializer for `x` in this case. That is we didn't write `int x{42};` inside the definition of `Foo`. `int` is of non-class type, so in this case, it should be [value-initialized](https://en.cppreference.com/w/cpp/language/value_initialization) (which in this case means [zero-initialized](https://en.cppreference.com/w/cpp/language/zero_initialization)).

Just in this short blog post alone, just trying to understand the difference between `Foo f;` and `Foo f{};`, we have already came across the following initializations – Default Initialization, Direct List Initialization, Aggregate Initialization, Default Member Initialization, List Initialization, Value Initialization, Zero Initialization.

