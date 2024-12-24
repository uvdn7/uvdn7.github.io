---
layout: post
title: C++ vs. Rust regarding to temporary lifetime
---

C++ views::zip | views::transform

if the transform F takes an auto& instead of const auto&, you get mysterious implementation specific error messages because it cant find template matches.

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://godbolt.org/z/P6e5fqP9n"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Compiler Explorer - C++ (x86-64 gcc 13.2)</div>
<div class="kg-bookmark-description">// requires /std:c++20 or later int main(){ std::vector&amp;lt;int&amp;gt; a{1,2,3}; std::vector&amp;lt;int&amp;gt; b{4,5,6}; auto r = std::views::zip(a, b) | std::views::transform([](const auto&amp;amp; zip){ auto&amp;amp; [a,b] = zip; return a+b; }); for (auto i: r) { std::co…</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src="https://godbolt.org/favicon.ico?v=1" alt=""><span class="kg-bookmark-publisher">Matt Godbolt</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://github.com/compiler-explorer/infra/blob/main/logo/favicon.png?raw=true" alt=""></div></a></figure>

C++'s temporary's lifetime – tl;dr it creates a temporary object which will live as long as the reference if bound to one

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://godbolt.org/z/hhes6PTTo"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Compiler Explorer - C++ (x86-64 gcc 13.2)</div>
<div class="kg-bookmark-description">// requires /std:c++20 or later struct Foo { Foo(int a) : a_(a) {} ~Foo() { std::cout&amp;lt;&amp;lt; “foo dtor (”&amp;lt;&amp;lt;a_&amp;lt;&amp;lt;”) \n”; } int a_;}; Foo foo(int a) { return Foo(a);} void bar() { const Foo&amp;amp; b = foo(3);} int main(){ Foo(1); auto f = Foo(4…</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src="https://godbolt.org/favicon.ico?v=1" alt=""><span class="kg-bookmark-publisher">Matt Godbolt</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://github.com/compiler-explorer/infra/blob/main/logo/favicon.png?raw=true" alt=""></div></a></figure>

