---
title: "The one with compiler flag magic"
date: 2019-11-01
toc: true
categories:
  - blog
tags:
  - C++
  - performance
  - compilers
  - bitset
---

### Introduction
At some point in the project I'm working on, I have noticed that variable-size bit set implementation is minor, but still performance bottleneck. For those who are not familiar with the topic, bit set is data structure similar in some way to array, but instead holding and giving access to individual bytes or words it allows accessing individual bits. Underlying structure is usually an array of integers, but provided API allows poking individual bits. For more details check [Wikipedia](https://en.wikipedia.org/wiki/Bit_array) and [C++ Reference](https://en.cppreference.com/w/cpp/utility/bitset) It has a lot of applications in cryptography, neural netowrks etc.

After short investigation I have found two operations that are slower than necessary: Bit Count and Find First Set. In today's post I will focus on the latter. Find First Set or `ffs` is operation which for given word returns position of least significant bit set to 1. So for example for:
```
56dec => 00111000bin
```
Find First Set returns 4 (remember least significant, so we are counting from the end).
### Solution
In implementation I was profiling `ffs` used was [one from not standard extension of glibc](http://man7.org/linux/man-pages/man3/ffs.3.html). In fact there is entire family of functions working on different data sizes, most useful are `ffsl` for 32bit integers and `ffsll` for 64bit ones with signatures looking like that:
```cpp
int ffsl(long int i);
int ffsll(long long int i);
```
I did not know what was internal implementation of those, but there was one thing that stands out: this is library function so function call is required! For small operations like `ffs` call overhead is significant. For example, if you want to apply `ffsll` to 64kb buffer, there will be over 8000 function calls! And there is an alternative! GCC (and probably other popular compilers) provides family of highly optimized builtins:
```cpp
int __builtin_ffsl (long)
int __builtin_ffsll (long long)
```
Advantage of builtins is fact that call is replaced with implementation (often in assembler), just like for inline functions. So it is time for…
### Results
To prove that change will actually improve anything, I have prepared simple benchmark calculating `ffsl` for entire few megabytes buffer. Results were more than surprising:
```
runGlibcFFsl64: 8106 MB/s
runBuiltinFFsl64: 8191 MB/s
```
What? Almost no difference! How is that even possible? Function call should be real overhead! 
<p align="center">
<img src="/assets/images/mindblown.gif" width="220">
</p>

To check what is happening under the hood I have used [Godbolt](https://godbolt.org/).
For C++ code:
```cpp
int builtinFfsll(uint64_t x) {
    return __builtin_ffsll(x);
}
 
int glibcFfsll(uint64_t x) {
    return ffsll(x);
}
```

this was assembler output:
```cpp
builtinFfsll(unsigned long):
bsfq %rdi, %rax
movq $-1, %rdx
cmove %rdx, %rax
incq %rax
ret
glibcFfsll(unsigned long):
jmp ffsll
```
So there is no doubt that there is call there (`jmp` instruction)! 

### Disbelief
At this point I have suspected some kind of compiler magic optimizing-out and not doing my calculations. But no matter how much changes I have introduced, results were still the same - there is no performance difference between GlibC and builtin implementation. I also started playing with different compiler flags, `march`, language standards, different compiler versions, different optimization flags… Nothing worked. Finally I have applied the same compiler flags as in other project I seen that working and it finally started to make sense! 
```
runGlibcFFsl64: 3865 MB/s
runBuiltinFFsl64: 8191 MB/s
```
## Enlightenment
After some time digging up which flag was to blame, it turned out it was `-std=gnu++11` that made GlibC implementation so fast! And it was so difficult to spot because I’m using CMake which, when setting C++ standard level to 11, sets this flag silently!
<p align="center">
<img src="/assets/images/magic.gif" width="220">
</p>

What `-std=gnu++11` really does? List of [changes introduced](https://gcc.gnu.org/onlinedocs/gcc/C-Extensions.html) by this flag is long and I wasn’t able to find explicit description what it does to `fsll`, but [Godbolt](https://godbolt.org/) provides some clues:
```cpp
builtinFfsll(unsigned long):
bsfq %rdi, %rax
movq $-1, %rdx
cmove %rdx, %rax
incq %rax
ret
glibcFfsll(unsigned long):
bsfq %rdi, %rax
movq $-1, %rdx
cmove %rdx, %rax
incq %rax
ret
```
Now both functions builtin and GlibC are identical. It seems that with GNU extension enabled compiler recognized call to `ffsl` and replaced it with builtin implementation!

### Results I was expecting
I have implemented some CMake magic to apply `-std=gnu++11` to some files and not apply it to others to have measure the difference, and those are final results:
```
runGlibcFFsl64: 3865 MB/s
runGlibcFFsl64_gnu: 8106 MB/s
runBuiltinFFsl64: 8191 MB/s
runGlibcFFsl32: 1667 MB/s
runBuiltinFFsl32: 4106 MB/s
```
As expected, buitin is much faster than regular library function. But library function with GNU extensions enabled is also fast. So function call is a major overhead, reducing performance by more than half. Additionally I have done measurements for 32bit types, with performance further reduced by half. It is not surprising, because when operating on smaller types more operations is needed to process the same volume of data. It's a pity that there is no SSE or AVX extension that could work on 128 or 256bit values. But probably that woudn't be useful for `ffs`. In the next part I will show how bit counting operations can be improved by using SSE/AVX operations and runtime dispatching for different architectures.

And here is [code for benchmark](https://github.com/vanklompf/BlogSrc/tree/master/BitSet) used to write this article.
