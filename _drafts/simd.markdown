---
layout: post
title: SIMD
---

questions:

- what are intrinsics
- SSE2 vs. AVX vs. AVX2
-  SSE has 128 bits (SSE2, SSE3, 4 ... are just adding more instructions) XMM registers
-  AVX2 uses YMM registers with 256 bits
-  AVX512 uses ZMM registers with 512 bits
<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://stackoverflow.com/questions/31490853/are-different-mmx-sse-and-avx-versions-complementary-or-supersets-of-each-other"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Are different mmx, sse and avx versions complementary or supersets of each other?</div>
<div class="kg-bookmark-description">I’m thinking I should familiarize myself with x86 SIMD extensions. But before I even began I ran into trouble. I can’t find a good overview on which of them are still relevant. The x86 architectur…</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src="https://cdn.sstatic.net/Sites/stackoverflow/Img/apple-touch-icon.png?v=c78bd457575a" alt=""><span class="kg-bookmark-author">Stack Overflow</span><span class="kg-bookmark-publisher">snoukkis</span>
</div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://cdn.sstatic.net/Sites/stackoverflow/Img/apple-touch-icon@2.png?v=73d79a89bded" alt=""></div></a></figure>
- how do I know if my CPU supports AVX-512
-  my AMD CPU only supports avx2

references

- [https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/](https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/) 
