---
layout: post
title: Rust's early vs. late lifetime binding
date: '2024-03-02 03:15:41'
tags:
- rust
---

Lifetime in many ways are similar to types but also different. The goal of this post is to explain my mental model of rust's lifetime binding works, when it takes place, and a trick to force late lifetime binding when needed.

Here's a motivating example that I will use throughout this post.

<!--kg-card-begin: markdown-->

    // ========== concrete.rs ===========
    // Foo<'_> implements ValueTrait defined in generic.rs
    struct Foo<'a> {
        value: &'a str
    }
    impl<'a> ValueTrait for Foo<'a> {
        fn get_value(&self) -> &'a str {
            self.value
        }
    }
    
    // Here's a concrete implementation of how to build a Foo with &str
    // with lifetime annotation explicitly spelled out
    fn build<'b>(s: &'b str) -> Foo<'b> {
        Foo {value: s}
    }
    
    
    // ======== generic.rs =========
    // A trait that's implemented by a few different structs
    trait ValueTrait {
        fn get_value(&self) -> &str;
    }
    
    // A generic function that works for different types that implement 
    // the ValueTrait and different `build` functions.
    fn generic_function<???>(build) {...}
    

<!--kg-card-end: markdown-->

The goal is to figure out how to write the function signature of `generic_function`.

## Take 1 – the straight forward approach
<!--kg-card-begin: markdown-->

The most straight forward approach is to write `generic_function` as

    fn generic_function<T, F>(build: F) 
    where F: Fn(&str) -> T,
    T: ValueTrait {...}

And the callsite would look like the following, with the concrete `build` function,

    generic_function(build);

<!--kg-card-end: markdown-->

The code doesn't compile. Here's the playground [link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=e98f4530ba45351bc8c255986b97f5b1). It produces the following not-very-helpful error message.

<!--kg-card-begin: markdown-->

       Compiling playground v0.0.1 (/playground)
    error[E0308]: mismatched types
      --> src/main.rs:29:5
       |
    29 | generic_function(build);
       | ^^^^^^^^^^^^^^^^^^^^^^^ one type is more general than the other
       |
       = note: expected struct `Foo<'_>`
                  found struct `Foo<'_>`
    note: the lifetime requirement is introduced here
      --> src/main.rs:20:22
       |
    20 | where F: Fn(&str) -> T,
       | ^
    

<!--kg-card-end: markdown-->

Isn't `Foo<'_>` the same as `Foo<'_>`? What is going on here?

## Take 2 – specialize and work our way back

It works fine if we remove the generic over `T`. The following compiles ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=0952847e1c785b3c13ff84bd462381f9))

<!--kg-card-begin: markdown-->

    fn generic_function(build: impl Fn(&str) -> Foo) {

<!--kg-card-end: markdown--><!--kg-card-begin: markdown-->

The above desugars to

    fn generic_function<F>(build: F)
    where F: for <'a> Fn(&'a str) -> Foo<'a> // here we uses the syntax for HRTB (https://doc.rust-lang.org/nomicon/hrtb.html)

which still compiles (given that I desugared it correctly).

<!--kg-card-end: markdown-->

Now let's add the generic argument `T` into the desugared version ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=127b3aff5644b7222e8ddcf7ee9f0457)).

<!--kg-card-begin: markdown-->

    fn generic_function<T, F>(build: F) 
    where F: for <'a> Fn(&'a str) -> T<'a>
    {

<!--kg-card-end: markdown-->

The compiler is not happy but with a different error message.

<!--kg-card-begin: markdown-->

       Compiling playground v0.0.1 (/playground)
    error[E0109]: lifetime arguments are not allowed on type parameter `T`
      --> src/main.rs:20:36
       |
    20 | where F: for <'a> Fn(&'a str) -> T<'a>
       | - ^^ lifetime argument not allowed
       | |
       | not allowed on type parameter `T`
       |
    note: type parameter `T` defined here
      --> src/main.rs:19:21
       |
    19 | fn generic_function<T, F>(build: F) 
       | ^
    

<!--kg-card-end: markdown-->

The error message here would make more sense later; but for now it's kind of weird (and `rustc --explain E0109` is not helpful). But it's OK, we can remove `<'a>` from `T<'a>` in the return position of the trait bound (this would later turn out to be the key).

<!--kg-card-begin: markdown-->

    fn generic_function<T, F>(build: F) 
    where F: for <'a> Fn(&'a str) -> T,
    T: ValueTrait
    {

<!--kg-card-end: markdown-->

This time ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=4382f9c480f152e1a359f583fed76d96)) it gives us a more useful error message, because we gave one lifetime a name,

<!--kg-card-begin: markdown-->

    error[E0308]: mismatched types
      --> src/main.rs:29:5
       |
    29 | generic_function::<Foo, _>(build);
       | ^^^^^^^^^^^^^^^^^^^^^^^^^^ one type is more general than the other
       |
       = note: expected struct `Foo<'a>`
                  found struct `Foo<'_>`

<!--kg-card-end: markdown-->

It's saying we are expecting `Foo<'a>` which makes sense due to the trait bound on `F`, but we are providing `Foo<'_>` from the callsite. In the first attempt, we got error message saying "expecting `Foo<'_>` but got `Foo<'_>`; these two `'_` s are different. One is the unresolved lifetime from the HRTB `where F: for <'_> Fn(&'_ str) -> T`, while the other is from the callsite `generic_function::<Foo<'_>, _>`. It only becomes obvious now, after we give the HRTB lifetime a different name.

## Early vs. late lifetime binding 

At a high level there are three steps performed by the rust compiler before it performs the borrow checking,

1. desugaring
2. monomorphism 
3. lifetime binding

Turns out a "more correct" version should be

1. desugaring
2. monomorphism + **early** lifetime binding
3. **late** lifetime binding

More details can be found at the appendix of [RFC3216](https://rust-lang.github.io/rfcs/3216-closure-lifetime-binder.html#appendix-late-bound-regions-early-bound-regions-and-region-variables) and [Niko's blog post](https://smallcultfollowing.com/babysteps/blog/2013/10/29/intermingled-parameter-lists/). The gist of it is that if a type (say `T`) is needed for monomorphism (e.g. stamping out an instance of a generic function), the lifetime is bound at the same time (early). Otherwise, the compiler would perform late lifetime binding, meaning that it would happily leave the lifetime stay generic, and only resolve it when it's used.

Now the earlier error messages start to make more sense. In the case of

<!--kg-card-begin: markdown-->

    fn generic_function<T, F>(build: F) 
    where F: for <'a> Fn(&'a str) -> T,
    T: ValueTrait
    {

<!--kg-card-end: markdown-->

the compiler needs to know `T` in order to stamp out the code for the function; hence it performs early binding for the lifetime - `T` becomes `Foo<'_>` from the caller. After the code is stamped out, the compiler starts late lifetime binding. HRTB's lifetime is late bound. At the place where we actually call `build`, in the body of the generic function, it binds `'a` to the regional scope, which means `T` gets resolved to `Foo<'a>`. Conflict! That's exactly the error message we got earlier.

Remember earlier the compiler doesn't allow us to write `T<'a>` in the return position of the trait bound?

<!--kg-card-begin: markdown-->

    fn generic_function<T, F>(build: F) 
    where F: for <'a> Fn(&'a str) -> T<'a>
    {

<!--kg-card-end: markdown-->

It makes sense now because `T` is early bound, while the HRTB's lifetime is late-bound. `T<'a>` in a HRTB doesn't make sense.

## Take 3 – force late lifetime binding 

We still haven't solved the original problem yet. We want to have a generic function over a type that implements `ValueTrait` and a `build` function. Here's a trick – credit to my colleague CJ that leverages [GAT](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html). Another problem solved by an extra level of indirection.

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=53e10f8e164d8c7e9597371d2a15f754)

<!--kg-card-begin: markdown-->

    struct Foo<'a> {
        value: &'a str
    }
    
    trait ValueTrait {
        fn get_value(&self) -> &str;
    }
    
    impl<'a> ValueTrait for Foo<'a> {
        fn get_value(&self) -> &'a str {
            self.value
        }
    }
    
    trait Builder {
        type ValueType<'c>: ValueTrait;
        fn build<'b>(s: &'b str) -> Self::ValueType<'b>;
    }
    
    struct FooBuilder{}
    impl Builder for FooBuilder {
        type ValueType<'c> = Foo<'c>;
        
        fn build<'b>(s: &'b str) -> Foo<'b> {
            Foo {value: s}
        }
    }
    
    fn generic_function<BUILDER>()
    where BUILDER: Builder
    {
        let owned = "123".to_string();
        let built = BUILDER::build(owned.as_str());
        println!("{:?}", built.get_value());
    }
    
    fn main() {
        generic_function::<FooBuilder>();
    }

<!--kg-card-end: markdown-->

`generic_function` is generic over a type that implements the `Builder` trait, with _no lifetime associated_. The compiler will only use the concrete type `FooBuilder` to stamp out a new version of the `generic_function` while the lifetime for `build` is not bound until it's called in `BUILDER::build(owned.as_str())`, at which point, the lifetime will resolve correctly to the regional scope - the body of the function - this behavior is what we want since the very beginning, and this is late life time binding.

## Why doesn't rust make all lifetime late binding

Based on [https://rustc-dev-guide.rust-lang.org/what-does-early-late-bound-mean.html#making-more-generic-parameters-late-bound](https://rustc-dev-guide.rust-lang.org/what-does-early-late-bound-mean.html#making-more-generic-parameters-late-bound), it looks like people generally agree late-binding is generally a good thing to have. Looks like the priority of making more things support late lifetime binding is just lower priority compared to other things for rust at the moment.

Unfortunately, information about early vs. late lifetime binding is scatter. In the meantime, I hope this post clears things up a bit.

## References

- [https://rust-lang.github.io/rfcs/3216-closure-lifetime-binder.html#appendix-late-bound-regions-early-bound-regions-and-region-variables](https://rust-lang.github.io/rfcs/3216-closure-lifetime-binder.html#appendix-late-bound-regions-early-bound-regions-and-region-variables)
- [https://smallcultfollowing.com/babysteps/blog/2013/10/29/intermingled-parameter-lists/](https://smallcultfollowing.com/babysteps/blog/2013/10/29/intermingled-parameter-lists/)
- [https://doc.rust-lang.org/nomicon/hrtb.html](https://doc.rust-lang.org/nomicon/hrtb.html)
