---
title: "Coming soon - High performance integers merging"
date: 2020-10-03
categories:
  - Blog
tags:
  - performance
  - simd
  - research
link: https://www.researchgate.net/publication/328875271_A_Fast_and_Simple_Approach_to_Merge_and_Merge_Sort_Using_Wide_Vector_Instructions
---

There is nothing simpler than merging two sorted vectors of integers, right? Well, it gets difficult when required performance is in hundreds of millions integers per second,
I will try my best at implementing merging algorithm from this research paper for AVX512 flavor from Xeon Scalable CPUs. Or at least some subset of it I will find practical. Coming soon!

<p align="center">
<img src="/assets/images/2020-10-03-fast-merging/merge-arrows.jpg" width="800">
</p>
