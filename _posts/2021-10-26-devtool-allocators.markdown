---
layout: post
title:  "Faster Mac Dev Tools with Custom Allocators"
date:   2021-10-26 21:45:46 -0400
categories: jekyll update
---

## Introduction

When trying to speed up a dev tool, like the swift compiler or the linker, developers may try a few things. For example, they may look through the man page for flags to speed them up (like `-j8` to parallelize `make` tasks). Or, they may improvise some caching system on top of the tool to eliminate unnecessary work. But there's another trick one can try: alternate allocators.

In a [previous post](https://eisel.me/allocator), I looked at what allocators are and how the default one for iOS (and really macOS) can be replaced at runtime. In this post, we'll look at the practical applications of alternate allocators for speeding up dev tools. For many dev tools, this trick can bring at least a modest speedup.

## How to do it

Fortunately, using a different allocator for a tool doesn't require it to be rebuilt. Instead, for allocators that are designed to swap themselves in, one can just add the environment variable `DYLD_INSERT_LIBRARIES=/path/to/allocator.dylib`. One example is jemalloc, which can be built by downloading the [jemalloc repo](https://github.com/jemalloc/jemalloc), running `cd jemalloc && ./autogen.sh && make`, and using the resulting `lib/libjemalloc.dylib`.

__Note: for tools that are hardened to strip `DYLD_` environment variables or prevent libraries from being sideloaded, this technique won't work.__

## Example: Compilers

Objective-C, C++, and Swift compilation can be sped up by swapping out the allocator. The impact for Objective-C in debug builds can be smaller, but for Swift and C++, especially for builds with optimizations, it can reduce time taken by 10% or more. We can change the allocator for swift, for example, by creating a small shell script:
```
export DYLD_INSERT_LIBRARIES=/path/to/libalternate_malloc.dylib
# Just calling /usr/bin/swiftc doesn't seem to work (maybe /usr/bin/swiftc is hardened?), so instead call the actual swiftc binary in the toolchain
exec `xcode-select -p`/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc "$@"
```

Then, set the Xcode build variable `SWIFT_EXEC = /path/to/my_swiftc.sh`, and it will have the swift compiler use this alternate allocator. Similarly, clang can be set with `CC = /path/to/my_clang.sh`.

## Example: llvm-profdata

llvm-profdata is a tool that takes the code coverage results from different test runs and merges them together to get the overall code coverage for a test suite. At one company, this step was adding 40 seconds to CI time, but with jemalloc it was cut in half to 20 seconds.

## Estimating the Potential Performance Gain

One easy way to know if a tool could be sped up through a different allocator, aside from just timing it with/without a different one, is to profile that tool with the Time Profiler. Although it may surprise some, ordinary mac executables can be profiled in Instruments, even if they weren't built by the developer themself. The executable can either be selected in Time Profiler and have its flags specified, or else a trace of all processes can be run while the executable is running. If a lot of time seems to be taken up by `malloc` and `free`, then there's more potential gain by using a different allocator. Although the performance impact of allocation is a complicated beast, this is a good litmus test to decide if it's worth trying.

## Which Alternate Allocator To Use?

jemalloc is a great choice because of its speed and its level of support on macOS (probably the highest level for any alternate allocator out there). The only other one I'm aware of with solid macOS support, both for the allocator itself and for seamlessly inserting it as the default, is mimalloc. In my testing, jemalloc and mimalloc seemed to have pretty similar performance, so I'd just stick with jemalloc. I also tested a simple bump allocator that never frees any memory and just allocates forever, but it generally seemed slower.

## Conclusion

Apple's default allocator is rock-solid and has some nice logging tools with it, but unfortunately it seems slower than other battle-tested allocators (please reach out if you know of exceptions). By replacing it in key places, one can reduce bottlenecks.
