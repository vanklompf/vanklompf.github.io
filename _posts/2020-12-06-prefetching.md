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
```
				LOCALITY
PREFETCHING	 0	 1	 2	 3
0	  		62	62	62	62
1	  		57	57	57	57
2	  		58	58	58	58
4	  		59	59	59	59
8	  		61	61	61	61
16	  		63	63	63	63
32	  		62	62	62	62
64	  		62	62	62	62
128	  		62	62	62	62
256	  		60	60	61	60
384	  		60	60	60	60
512	  		58	58	58	58
768	  		52	52	52	52
1024  		47	47	47	47
```
There is no differentce for all the values of locality, so either it requires some special conditions to work or it is just not working for this architecture

## Tracing memory performance
As we can see it is not trivial to correctly identify and solve memory bottlenecks in even simple applications. How can we know that our application is limited by memory performance and changes we are doing are beneficial? Fortunatelly CPU provides various hardware counters storing number of all memory acceses and ones that were served by cache. This can show us how effective we are using cache in CPU. Those counters can be accessed programatically so for exampe we can instrument only performance critical part of our code, but easier way to access them is by using `perf stat` tool.
Running it requires specyfing which counters we are interested in and application we want to measure:

```
perf stat -e cache-misses,cache-references,cycles,instructions,L1-dcache-loads,L1-dcache-misses,LLC-loads,LLC-load-misses ./PacketDescriptorsPrefetching
```
Here we have specified counters responsible for various levels of data cache and ones that can show instructions per cycle ratio.

Results without payload prefetching:
```
Prefetching 4 pointers ahead
Payload read processing performance long: 9 mpps
        2191257897      cache-misses              #   78.771 % of all cache refs      (49.99%)
        2781792905      cache-references                                              (50.00%)
      364710082877      cycles                                                        (50.01%)
      236285106758      instructions              #    0.65  insn per cycle           (62.51%)
       38528300611      L1-dcache-loads                                               (62.51%)
        3000142952      L1-dcache-misses          #    7.79% of all L1-dcache hits    (62.51%)
        1209407796      LLC-loads                                                     (62.50%)
         670348079      LLC-load-misses           #   55.43% of all LL-cache hits     (62.49%)
```

Results with prefetching:

```
Prefetching 0 pointers ahead
Payload read processing performance long: 4 mpps
        2156820650      cache-misses              #   78.719 % of all cache refs      (50.00%)
        2739896913      cache-references                                              (50.00%)
      176376504963      cycles                                                        (50.01%)
      239466479151      instructions              #    1.36  insn per cycle           (62.51%)
       39593265565      L1-dcache-loads                                               (62.51%)
        2957529169      L1-dcache-misses          #    7.47% of all L1-dcache hits    (62.50%)
         574832230      LLC-loads                                                     (62.49%)
          48264412      LLC-load-misses           #    8.40% of all LL-cache hits     (62.49%)
```
To some surprise general `cache-misses` and `L1-dcache-misses` stays the same. But there is clear difference for `LLC-load-misses` (LLC stands for last level cache, so L3 on this CPU) which is responsible for performance gain when prefetching. Also increased `insn per cycle` shows that CPU was able to spend more time executing code and less waiting on memory.

## Summary
I hope I have showed that prefetching can be viable technique of improving performance at least in some class of application. Having some knowledge about CPU and cache architecture as well some profiling tools can be very helpful here. Also it is worth to remember that prefetching is very architecture dependent and volatile: what works well on one CPU can work different way on other. It is always important to know range of architectures our application will work on and measure performance instead of making assumptions. Things like predictive prefetching in CPU and proper timing of prefetch calls can be tricky.
All code used to write this article can be found, as usual [on my GitHub](https://github.com/vanklompf/BlogSrc/tree/master/PacketDescriptorsProcessing/).