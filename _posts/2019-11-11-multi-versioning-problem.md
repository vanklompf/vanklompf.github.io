---
title: "The one with multi-versioning (Part I)"
date: 2019-11-11
toc: true
categories:
  - blog
tags:
  - C++
  - performance
  - compilers
  - bitset
---
## Introduction
In [previous](/blog/compiler-flags-magic/) post I have described benefits of inlining and surprising impact of some compilation flags using `Find First Set` as an example. Today and in the upcoming second part of this post, I will explore function multi-versioning intricacies using [Population Count](https://en.wikichip.org/wiki/population_count) case. 

So actually what `popcount` is? It is quite simple bitwise operation counting set bits (ones) in given word/byte/vector. So for example for:
```
8456dec => ‭10000100001000‬bin
```
`popcount` returns 3. 

Operations looks simple but it has surprisingly wide set of use cases:
   * [Chess playing algorithms](https://www.chessprogramming.org/Population_Count)
   * [Neural netowrks](https://sushscience.wordpress.com/2017/10/01/understanding-binary-neural-networks/)
   * [Information theory and cryptography](https://en.wikipedia.org/wiki/Hamming_distance)
   
## Implementation
The most basic and naive implementation can look like that:
```cpp
int popcount(unsigned x) {
    int c;
    for (c = 0; x != 0; x >>= 1)
        if (x & 1)
            c++;
    return c;
}
```
Its performance will be terrible, but probably for simple solutions it will be enough. Also it is worth to know, because this is a question often appearing on recruitment talks!

There are other, better implementations using different optimization approaches: lookup tables or [counting in a tree pattern](https://en.wikipedia.org/wiki/Hamming_weight#Efficient_implementation). Modern CPUs have also dedicated instructions for calculating `popcount`. The easiest way to access that is, similarly like for `fsl`, to use compiler builtins. Here are the signatures of builtins provided by GCC: 
```cpp
int __builtin_popcountl (unsigned long)
int __builtin_popcountll (unsigned long long)
```
As usual one is for 32 and other 64bit integers. 

## A bit of CPU magic
CPUs with SSE4 extension provides dedicated instruction theoretically improving performance of `popcount` called [`POPCNT`](https://www.felixcloutier.com/x86/popcnt). Most recent processors additionally have vectorized version: [`VPOPCNTQ`](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=VPOPCNTQ&expand=4368), which can count bits in entire 512bits (64 bytes) vector at once! It is available using compiler intrinsic:
```cpp
__m512i _mm512_popcnt_epi64(__m512i __A)
```
According to gcc flags definition, only Intel CPUs starting from `icelake` will support this instruction. So it is probably not yet available outside of simulators or pre-production samples. As soon as I will lay my hands on this hardware, I will definitely check it!

There are also other implementations, for example mixing different available vector operations (SSE, AVX) with other optimizations. I started even implementing few examples, but then I found that, as usual, someone on the internet has done it better. There is a great library full of different super-optimized [implementations of `popcount`](https://github.com/WojciechMula/sse-popcount), based on research and resulting in some [papers published](https://arxiv.org/abs/1611.07612) done by Wojciech Mula. Below few results from different implementations:
```
Naive popcount32: 225 MB/s
HW popcount32: 6106 MB/s
HW popcount64: 13355 MB/s
Avx2 assisted popcount: 13355 MB/s
Vectorized HW popcount: N/A

```
Results are not suprising, naive implementation is terribly slow, 64bit version is significantly faster than 32bit one. And AVX512 vectorized `popcount` is not available on my machine. Source code for benchmark can be found on my [GitHub account](https://github.com/vanklompf/BlogSrc/tree/master/BitSet_Popcnt).


## Architecture specific code
Before going further and joining `popcount` with function multi-versioning, we need to know what all this multi-versioning actually is and why we need that.

Let's imagine our application is running on a few generations of servers using different architectures of x86 processors, some with and some without hardware support of `popcount`, AVX, SSE4 etc. All of those servers need to run the same binary - which in most cases is the only sane solution as compiling differently for different architectures is overhead nightmare! How can we do that in single binary than? Let's start with figuring out how customizing for specific architecture works.

In general case tuning application for specific architecture is done by using `march` flag. We can provide values like `skylake`, `sandybridge`, `core2` etc. In case of supporting multiple architectures `march` has to be set to **lowest supported CPU type**. This is because the compiler can and usually will emit assembler instructions for configured CPU architecture. If binary will be run on older CPU, it will just not work, crashing immediately when executing not supported instructions. Lets check this on an example of `count_if` function (function taken from [here](http://0x80.pl/notesen/2019-02-02-autovectorization-gcc-clang.html)):
```cpp
size_t count_if_epi8(const std::vector<int8_t>& v) {
    return std::count_if(v.begin(), v.end(), [](int32_t x){return x == 42 || x == -1;});
}
```

When compiling for pure `x86-64` architecture, resulting assembler code will be:
```cpp
size_t count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&):
        mov     rsi, QWORD PTR [rdi+8]
        mov     rdx, QWORD PTR [rdi]
        xor     eax, eax
        cmp     rsi, rdx
        jne     .L5
        jmp     .L6
(...)
```

but when enabling `-march=skylake` it is something completely different:
```cpp
size_t count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&):
		(...)
        vmovdqa ymm4, YMMWORD PTR .LC0[rip] // <-- AXV2
        add     rcx, rdx
        vpxor   xmm5, xmm5, xmm5
        vpcmpeqd        ymm3, ymm3, ymm3 // <-- AXV2
		(...)
```
Notice `vmovdqa` and `vpcmpeqd` instructions, which are AVX specific. Those will crash if run on CPU without AVX extension.

## Function multi-versioning
Fortunately there is a way to have both: ability to run on older platforms and taking advantage of new CPU extensions. To achieve that we need to use GCC feature called [function multi-versioning](https://gcc.gnu.org/onlinedocs/gcc-9.1.0/gcc/Function-Multiversioning.html). Basically what it does is allowing to define multiple versions of function, targeted to different architectures, with some boilerplate code that will recognize architecture at runtime and select proper function. Lets try that with previous example:
```cpp
__attribute__ ((target ("avx2")))
size_t count_if_epi8(const std::vector<int8_t>& v) {
    return std::count_if(v.begin(), v.end(), [](int32_t x){return x == 42 || x == -1;});
}
```

```cpp
__attribute__ ((target ("default")))
size_t count_if_epi8(const std::vector<int8_t>& v) {
    return std::count_if(v.begin(), v.end(), [](int32_t x){return x == 42 || x == -1;});
}
```
Target architecture is defined by `__attribute__ target`, allowed values are similar to those used in `march` switch.  In [output from Godbolt](https://godbolt.org/z/4V6TeP) there are indeed two functions:
```cpp
count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&).avx2
count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&):
```
also some code selecting version based on detected CPU is present:
```cpp
        call    __cpu_indicator_init
        mov     eax, OFFSET FLAT:count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&)
        mov     edx, OFFSET FLAT:count_if_epi8(std::vector<signed char, std::allocator<signed char> > const&).avx2
        test    BYTE PTR __cpu_model[rip+13], 4
```

In gcc 6 and later, there is simpler, better, more compact way to create multi-versioned function using `__attribute__ target_clones`:
```cpp
 __attribute__((target_clones("avx2","default")))
size_t count_if_epi8(const std::vector<int8_t>& v) {
    return std::count_if(v.begin(), v.end(), [](int32_t x){return x == 42 || x == -1;});
}
```

**So in the end we have what we wanted**: possibility to run most optimized code across a range of different architectures in a single binary. But as usual...
## It's not perfect
Multi-versioning has also some limitations. Due to the fact that there need to be condition to check CPU architecture, followed by function call, multi-versioned functions are never inlined. As seen in [previous post](/blog/compiler-flags-magic/) this can have quite significant cost, negating improvements from architecture specific optimizations. In next post I will try to dig deeper, using `popcount` example, on to how much this is problematic in practice and what are ways to fix that.
