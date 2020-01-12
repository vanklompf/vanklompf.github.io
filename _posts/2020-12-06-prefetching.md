---
title: "Prefetching in data processing"
published: false
date: 2020-01-06
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
### Prefetching descriptors
Let's get back to our example implementation of processing packet descriptors from previous article. Using `PacketDescriptorPacked`, which had best performance in most cases we will try improve it further with prefetching. Prefetching of arbitrary memory location can be done using: `__builtin_prefetch` which details can be found [here](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html). The only obligatory argument is pointer to memory location that needs to be loaded into cache. Of course not only exact address will be loaded, but entire 64bytes cache line. Pointer can be invalid or null, which is useful because for example some bound checking can be ignored saving CPU cycles. There are two additional arguments to `__builtin_prefetch` but thoise are not important for this example. 
So let's try and add prefetching into benchmark loop. How much data to prefetch? In theory this depends on many factors like cache sizes, memory latency, time to process single line of cache worth of data. This is very difficult to evaluate properly, so best option is to just try multiple options. This chart shows performance for different number lines of cache loaded in advance. 
[chart]
There was no visible performance improvement no matter how much data were prefetched. Reason for that is something that was already mentioned before: CPU is smart enough to do prefetching for us. Access pattern in our benchmark is completely linear which is extremely easy to predict. Data are being prefetched transparently and no amount of explicit prefetching can improve that.

### Prefetching payload
Another example which is also practical and applicable to working with network packets is iterating over packet descriptors but processing also payload pointed by one of descriptor structure field. This has dramatic impact on memory access pattern: descriptors are placed back to back, one after another, but payload can be in completely different memory location, not adjacent one to another, which is not possible to predict easily.
[image]
Benchmark simulating this kind of processing was implemented in `ProcessDescriptorsPayloadRead` function. It is completely trivial, just summing up first word from packet payloads. We are not prefetching descriptors here as it was proven pointless in previous example, but just payload for different number of packets ahead. This way we can expect at least some improvements in performance, because CPU has no way knowing our data structures and guessing which data will be needed next, but our program knows that and can issue prefetch command for those data.
[chart]
Suprisingly there was just marginal performance improvement. There is also reason for that. Time spent in processing data is so miniscule compared to time needed to load (or prefetch) data that no memory bandwidth became limiting factor of performance here. No prefetching amount can improve that significantly, because memory bus is busy almost all the time anyway. For prefetching to make sense and improve anything it needs to have chance running "in background", so when CPU is proccessing current batch of data, future data alre alredy being prefetched. Loading data from RAM to cache can work in paralell with CPU doing other work.

### Prefetching payload with heavy processing
Modified example simulates more time consuming processing of data. So at least there are periods of time where CPU is busy processing data already available in cache and more data can be loaded during that period, making difference for prefetching case.
[chart]
Here performance improvement is much more significant going up to xx%, compared with no-prefetching case. As for quite simple and safe change in code, not affectiong business logic at all this is quite impresive result. This shows prefetching as powerful tool for memory-bound applications. There are some prerequisites, like certain processing to memory access ratio and keeping prefetching logic as minimal as possible, so that our improvements are not diminished by it, but overall this makes good and relatively simple optimising technique.

### Bonus: optional arguments
`__builtin_prefetch` has two additional arguments: `rw` and `locality`. 
`rw` indicates if prefetched memory is going to be used for read or write. This sounds simple in theory, but is more complicated to make it work in practice. One issue is that only most modern CPUs supports, according to (gcc manual)[https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html] starting since `broadwell` architecture at least for Intel. So sometimes to emit `PREFETCHW` instructionand and actually make any difference between read and write prefetching architectural dispatching is needed. Second more complex issue is that actually it is hard to find case where write prefetching makes any difference. On first glimpse, it seems that when loading data to cache it make no difference if we are going later to read or write, but there are some cases in multi-core and especially multi-socket machines, where it can make quite lot of difference. It is mostly about invalidating cache on all other CPUs, so that when writing to cache memory, there will be no need for cache sync-up between CPUS. But this is way out of scope for this post and requires more studying on my side.
As for `locality` it is supposed to indicate degree of `temporal locality`. A value of zero means that the data has no temporal locality, higher values means more temporal locality. This probably has something to do with levels of cache where data needs to be loaded, but let's just try it out and see what difference does it make in our examples.

