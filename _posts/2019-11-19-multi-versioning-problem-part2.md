---
title: "The one with multi-versioning (Part II)"
date: 2019-11-19
toc: true
categories:
  - blog
tags:
  - C++
  - performance
  - compilers
  - bitset
---
In [the last post](/blog/multi-versioning-problem/) we have focused on `popcount` operation and function multi-versioning. In this post I show shortcomings of multi-versioning usied with `popcnt` builtin. Of course those limitations are valid for other builtins as well.

### Start simple
Let's imagine we want to count bits in huge vector of integers. There is a builtin for that, isn't it? So this should be simple:
```cpp
inline int popcount64_builtin(uint64_t data) {
    return __builtin_popcountll(data);
}

int runPopcount64_builtin(const uint8_t* bitfield, int64_t size, int repeat) {
    int res = 0;
    const uint64_t* data = (const uint64_t*)bitfield;

    for (int r=0; r<repeat; r++)
    for (int i=0; i<size/8; i++) {
        res += popcount64_builtin(data[i]);
    }

    return res;
}
```
Compile with `-O3` optimization level aaaand:
```
runPopcount64_builtin: 2829 MB/s
```

<p align="center">
<img src="https://memecrunch.com/meme/BNKZO/wait-a-sec/image.jpg" width="300">
</p>
Something is wrong. In previous post it was above 12000 MB/s. Quick check on Godbolt and we see:
```cpp
call __popcountdi2
```
in our assembly output. Which means a function call per every `popcnt` use.

### Assume worst
It turns out that code was compiled without either a `-march` or `-m` flag so the compiler assumed the worst: we can be running this code on really old hardware. Before using the `popcnt` hardware instruction a runtime platform check is necessary. Such a runtime check happens inside __popcountdi2. Performing a platform check is probably not that expensive, but losing inline, however, really is. So let's add `-mpopcnt` to compiler flags to indicate that we do have CPU support for this instruction.
```
runPopcount64_builtin: 15634 MB/s
```
Ok, much better now. Quick check in Godbolt:
```cpp
popcnt  rcx, QWORD PTR [rdi+rax*8]
```
and we can see that there is no function call and `popcnt` instruction is used directly.

### Plane multi-versioning
But what if we don’t know the exact CPU type of the platform where our code will run? Or even worse, we know that it may run on an old CPU? We can allow code to run slower on old CPUs, yet can't allow to crash due to executing unsupported instructions. But hey, apparently there is this thing called [function multi-versioning](/blog/multi-versioning-problem/#function-multi-versioning). Let's try it out here:
```cpp
__attribute__((target_clones("popcnt","default")))
inline int popcount64_builtin_multiarch(uint64_t data) {
    return __builtin_popcountll(data);
}
```
The results are still not that great:
```
runPopcount64_builtin_multiarch: 3562 MB/s
```
When checking [output in Godbolt](https://godbolt.org/z/7caZ8f) there is this:
```cpp
popcount64_builtin_multiarch(unsigned long) [clone .resolver]:
        sub     rsp, 8
        call    __cpu_indicator_init
        test    BYTE PTR __cpu_model[rip+12], 4
        mov     eax, OFFSET FLAT:popcount64_builtin_multiarch(unsigned long)
        mov     edx, OFFSET FLAT:popcount64_builtin_multiarch(unsigned long) [clone .popcnt.0]
```
This is a function resolver - boilerplate code that selects a proper version of function to run based on the concrete CPU type known at runtime. See that `__cpu_model`, right? It is as smart as it can be, but still requires function call, which effectively defeats the whole idea of using hardware accelerated `popcnt`. 

### Multi-versioning of outer loop
But maybe we can be smarter than that? Let’s try not to multi-version the callee inside the loop, but rather the entire outer loop. This should allow us to run the best possible popcount implementation without the runtime overhead of dispatching function on every loop iteration. The dispatch will happen once, before entering the loop. The code looks like this:
```cpp
inline int popcount64_builtin_multiarch_loop(uint64_t data) {
	return __builtin_popcountll(data);
}

__attribute__((target_clones("popcnt","default")))
int runPopcount64_builtin_multiarch_loop(const uint8_t* bitfield, int64_t size, int repeat) {
    int res = 0;
    const uint64_t* data = (const uint64_t*)bitfield;

    for (int r=0; r<repeat; r++)
    for (int i=0; i<size/8; i++) {
        res += popcount64_builtin_multiarch_loop(data[i]);
    }

    return res;
}
```
And the results are bad again:
```
runPopcount64_builtin_multiarch_loop: 2826 MB/s
```
Another look on [Godbolt output](https://godbolt.org/z/Z4q89f) reveals that something weird is happening there. There are two versions of the function, but no resolver code and most likely the slower version ends up being called all the time. Maybe it’s a compiler bug or perhaps it just doesn’t work like that.

### Intrinsics
<p align="center">
<img src="https://media.giphy.com/media/1MclE9Kaw3996/giphy.gif" width="300">
</p>

Ok, this approach is not working. So maybe let's try something more radical. Except `builtins`, which are all nice and civilised, because doing whole platform dispatching "automagically", there are also `intrinsics`. Those are more like C++ wrappers for assembly instructions. No boilerplate code and compiler assistance, but C++ function semantic, so no need to write assembler snippets. Our function looks like that:
```cpp
inline int popcount64_intrinsic(uint64_t data) {
    return _mm_popcnt_u64(data);
}
```
Of course we would have to provide platform detecting code ourselves but this isn't really difficult to do.
```cpp
if (__builtin_cpu_supports("popcnt"))
    do_something();
```
And of course we will be smart this time round and select proper implementation outside of the loop, to run it only once and get this sweet inlining. Unfortunately:
```
error: inlining failed in call to always_inline ‘long long int _mm_popcnt_u64(long long unsigned int)’: target specific option mismatch
```
Compiler doesn't let us to be that smart. No matter if we know what we are doing and control the execution of this code based on runtime platform, it will not emit a `popcnt` instruction here.

### Assembler
Ok, it is getting weird now. We can try to be even more hardcore and use inline assembler:
```cpp
inline int popcount64_asm(uint64_t data) {
        uint64_t ret;
        __asm__ volatile("popcntq %0, %1"
                     : "=a" (ret)
                     : "r" (data));
        return ret;
}
```
The compiler will not object to emit `popcnt` instruction if we really, really insist and write it directly as inline assembly code. And the result is:
```
runPopcount64_asm: 14354 MB/s
```
<p align="center">
<img src="https://pics.me.me/thumb_hmisee-ger-hm-i-see-holding-cup-of-coffee-52748457.png" width="300">
</p>
And this is all great, but there is something just not right if we need to resolve to using assembly here. It is not the end yet!

### Multiple files, multiple flags
What about another idea? Move functions using hardware `popcnt` to another cpp file and compile it with proper flags, select proper function using `__builtin_cpu_supports` and link everything into one binary. Can it be done? Apparently yes. In `CMake` it looks like that:
```
set_source_files_properties(popcount_with_hw_support.cpp PROPERTIES COMPILE_FLAGS -mpopcnt)
```
And it works! I tested it with both the builtin and the intrinsic and the results are good (I have no idea why not the same, the assembly output for both is identical...).
```
runPopcount64_builtin_mpopcnt: 12221 MB/s
runPopcount64_intrinsic_mpopcnt: 15264 MB/s
```
This is good, but not perfect. It requires moving entire fragments of code (not only small, internal functions) into separate files. If this is just part of a bigger implementation with a lot of inlining, the amount of code needed to extract can be quite high. Also there might appear need to create multiple copies of functions for number of possible CPU architectures (`SSSE3`, `SSE4`, `AVX` etc.). But finally...

### GCC9 to the rescue
<p align="center">
<img src="http://www.clker.com/cliparts/x/i/K/f/n/q/super-hero-red-cape.svg" width="200">
</p>
Remember that idea with [doing multi-versioning for entire loop](#multi-versioning-of-outer-loop)? So far I was using `gcc7`, but apparently it works fine in `gcc9`! Finally there is resolver for function with loop, where proper version is selected and run. Check this [Godbolt output](https://godbolt.org/z/EjpAzx). And results:
```
runPopcount64_builtin_multiarch_loop: 14825 MB/s
```

### Summary
In the end something that supposed to be simple: multi-versioning `popcnt` functionality, turned out to be quite hard to do right. Benchmarking was invaluable tool to know real performance and Godbolt to know why performance is so bad. There are three ways to do `popcount` multi-versioning effectively:
  * move version for different CPUs to separate files and compile those files with proper `-m` and `-march` flags
  * assembler inline
  * **multi-versioning of outer loop** and `gcc9` (I believe this one is the best)

The final working code presented in this post is, as usual [on my GitHub](https://github.com/vanklompf/BlogSrc/tree/master/MultiVersionInline).

