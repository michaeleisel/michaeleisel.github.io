---
layout: post
title:  "Faster Dev Tools with Custom Allocators"
date:   2021-07-13 21:45:46 -0400
categories: jekyll update
---

## Introduction

When trying to speed up applications that are bottlenecks in their workflow, developers may try a few things. For example, they may look through the man page for flags to speed it up (like `-j8` to parallelize `make` tasks). Or, they may improvise some caching system to eliminate unnecessary work. But there's another trick one can try: alternate allocators.

In a [previous post](https://eisel.me/allocator), I looked at what allocators are and how the default one can be replaced at runtime. In this post, we'll look at the practical applications of alternate allocators for speeding up dev tools. For many applications, work done by the allocator takes up a substantial portion of the overall time. There are alternate allocators out there, such as jemalloc, that tend to be faster than Apple's. For many applications, they can deliver an approximate ~5% reduction in time taken for the whole application, and in some cases even more. When it comes to dev tools that are being used all the time, these sort of performance gains can add up.

## Example: Speeding Up Compilers

Objective-C and Swift compilation can both be sped up 

## Example: llvm-profdata

llvm-profdata

## Estimating the Potential Performance Gain

One easy way to know if a tool could be sped up through a different allocator, aside from just timing it with/without a different one, is to profile that tool with the Time Profiler. Although it may surprise some, ordinary mac executables can be profiled in Instruments, even if they weren't built by the developer themself. If a lot of time seems to be taken up by `malloc` and `free`, then there's more potential gain by using a different allocator. Although the performance impact of allocation is a [complicated beast](), this is one good litmus test.

## Which Alternate Allocator To Use?

jemalloc is a great choice because of its speed and its level of support on macOS (probably the highest level for any alternate allocator out there). The only other one I'm aware of with solid macOS support, both for the allocator itself and for seamlessly inserting it as the default, is mimalloc. In my testing, jemalloc and mimalloc seemed to have pretty similar performance, so I'd just stick with jemalloc. I also tested a simple bump allocator that never frees any memory and just allocates forever but it generally seemed slower.

## Conclusion

Apple's default allocator is rock-solid and has some nice logging tools with it, but unfortunately it seems slower than other battle-tested allocators (please reach out if you know of exceptions). By replacing it in key places, one can reduce bottlenecks.

- Allocators can be even faster

- Plug for my products?
- Show that total memory usage was also factored in
