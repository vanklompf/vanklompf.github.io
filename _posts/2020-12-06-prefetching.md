---
title: "Prefetching in data processing"
published: false
date: 2020-12-06
categories:
  - blog
tags:
  - C++
  - performance
  - cache
  - Packet Processing
  
---

In [the last post](/blog/shrinking-structure-part2) we have been looking on various, sometimes intrusive and complicated methods of compacting data structures for providing better memory and cache usage. Today we will continue with other method of improving performance for data intensive applications: prefetching.

## Data flow in computer system
Before starting with showing examples, let's think about how and when actually data moves from memory to processor. Imagine very simplistic example: calculating sum of array elements.
```cpp
uint64* data[64000];
uint64_t sum = 0;
for(int i=0;i<64000;i++) {
    sum += data[i];
}
```

Here is the simplest system consisting only from CPU and memory:
[IMAGE]

On such system memory will be accessed, every time we reference `data[i]`.  In modern CPU that would be disaster as access to RAM is terribly slow, in range of ~100ns (where one CPU cycle is ~0.4ns). Fortunatelly most of modern CPU including small embedded ones are more compilcated than that.
To improve performance, back in '80s mainstream computer systems were enchanced witch cache, in most cases many levels of it:
[IMAGE]
Here when reading value from memory, not only this particular value is loaded but also nearby 64 bytes of data. This chunk of data is called cache line. So in our example where data are accessed in 64bit (8 byte) chunks, every 8th iteration CPU will have to wait on memory access.

[IMAGE]
This design was enhanced even further witch introduction of `speculative prefetching`. This ideas is based on fact that data in most cases are accessed in predictable manner and can be read ahead of time to cache. Simplest case of predictability is continous linear memory access, exactly as in our example. Any relativaly modern CPU will notice that and read few next cache lines, even before CPU will attempt accessing those, so data will be in cache when needed. This way there will be almost no memory stalls during program execution. This process of reading data from RAM to cache, ahead of time is called prefetching. Predicting which cache lines to read next is not limited to continous access, it works well also for stride access where processing every Nth element in memory. Please note that image is not accurate when it comes to "prefetching unit" location, which is usually part of CPU internals or memory controler. Also there is much more into that, like multiple cache levels, cache policies, cache synchronisation protocols but describint those in details would definitely be out of scope for this article.

## Practical examples
Let's get back to our example implementation of processing packet descriptors from previous article. Using `PacketDescriptorPacked`, which had best performance in most cases we will try improve this performance further with prefetching. Prefetching of arbitrary memory location can be done using: `__builtin_prefetch` which details can be found [here](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html). The only obligatory argument is pointer to memory location that needs to be loaded into cache. Of course not only exact address will be loaded, but entire 64bytes cache line. Pointer can be invalid or null, which is important because for example some bound checking can be ignored saving CPU cycles. There are two additional arguments to `__builtin_prefetch` but thoise are not important for this example. 
 