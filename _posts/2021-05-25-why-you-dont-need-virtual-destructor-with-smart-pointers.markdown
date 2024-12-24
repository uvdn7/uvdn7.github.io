---
layout: post
title: Why you don't need virtual base destructor with smart pointers
date: '2021-05-25 02:25:40'
tags:
- cpp
---

    
    struct Foo {
        // virtual ~Foo() {};
        int a;
    };
    
    struct Bar : public Foo {
        ~Bar() {std::cout << "bar dtor" << std::endl;};
    };
    
    int main() {
        std::shared_ptr<Foo> f = std::make_shared<Bar>();
        //Foo* f = new Bar();
        return 0;
    }

<!--kg-card-end: markdown-->

In this example, the `shared_ptr` version would work as you expect. The raw pointer version will, however, not call Bar's destructor because Foo's dtor is not declared `virtual`. The question is why the `shared_ptr` version works?

Smart pointers work by managing a separate control block. It not only stores a ref count in the control block but also a deleter, a callable that gets called when ref count reaches zero. Multiple `shared_ptr`s of the same underlying resource must share the control block.

In the example above, the deleter of the control block is set when a `shared_ptr<Bar>` was created. Hence the deleter will always delete a Bar object, calling Bar's destructor, no matter what happens. It has nothing to do with virtual dispatch. Hence it doesn't require Foo to have a virtual destructor.

## libc++'s implementation

Let's take a look at `libc++`'s implementation for the aforementioned behavior.

1. [https://github.com/llvm/llvm-project/blob/main/libcxx/include/\_\_memory/shared\_ptr.h#L453](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L453) Create a new `shared_ptr` from a raw pointer. 
2. [https://github.com/llvm/llvm-project/blob/main/libcxx/include/\_\_memory/shared\_ptr.h#L670](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L670) It uses the default deleter of `_Yp` (Bar here).
3. [https://github.com/llvm/llvm-project/blob/main/libcxx/include/\_\_memory/shared\_ptr.h#L840](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L840) When it copies to `shared_ptr<Foo>`, the same control block is passed over.

Notice how it always takes the pointer type and use that to create the deleter here [https://github.com/llvm/llvm-project/blob/main/libcxx/include/\_\_memory/shared\_ptr.h#L670](https://github.com/llvm/llvm-project/blob/main/libcxx/include/__memory/shared_ptr.h#L670). This means the following would work as expected,

<!--kg-card-begin: markdown-->

    auto f = std::shared_ptr<Foo>(new Bar());

<!--kg-card-end: markdown-->