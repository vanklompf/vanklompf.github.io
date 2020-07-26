---
title: "Old and new function multi-versioning"
date: 2020-07-27
categories:
  - blog
tags:
  - C++
  - compilers
  
---

I have been touching multi-versioning already in two previous articles: [Part I](/blog/multi-versioning-problem/), [Part II](/blog/multi-versioning-problem-part2/) but I wanted to return to this topic once more (and probably not for the last time).
The multi-versioning feature is available in GCC for quite some time, it has appeared in GCC4.8 but went some refinement in GCC6. In this short post, I will show what has changed and how to migrate from old to a new version. New compilers support both versions so migration is optional, although it makes life much easier.

## The Old
Multi-versioning was implemented as function attribute, where for every separate specialization, function had to be defined once. So in case of targeting code to 3 CPU architectures, function would have to be defined 3 times:
```cpp
__attribute__((target("sse4.2")))
uint64_t sum(const uint32_t* val, int size) {
    uint64_t result = 0;
    for (int i=0; i<size; i++) result += val[i];
        return result;
}

__attribute__((target("avx2")))
uint64_t sum(const uint32_t* val, int size) {
    uint64_t result = 0;
    for (int i=0; i<size; i++) result += val[i];
        return result;
}

__attribute__((target("default")))
uint64_t sum(const uint32_t* val, int size) {
    uint64_t result = 0;
    for (int i=0; i<size; i++) result += val[i];
        return result;
}
```

For short functions this was acceptable (altohugh inconvinient). For longer functions and to reduce code duplication workaround was to use fully inlined implementation and wrappers for each specializations:
```cpp
inline __attribute__((always_inline))
uint64_t sumImpl(const uint32_t* val, int size) {
    uint64_t result = 0;
    for (int i=0; i<size; i++) result += val[i];
        return result;
}

__attribute__(target("sse4.2")))
uint64_t sum(const uint32_t* val, int size) {
    return sumImpl(val, size);
}

__attribute__(target("avx2")))
uint64_t sum(const uint32_t* val, int size) {
    return sumImpl(val, size);
}

__attribute__(target("default")))
uint64_t sum(const uint32_t* val, int size) {
    return sumImpl(val, size);
}
```

Note the use of attribute `always_inline`. If `sumImpl` would be called instead of being inlined, attribute `target` would not be applied to its implementation and this would be completely useless. Multi-versioning is not propagated down the call tree, but it is applied to the unrolled/inlined code.

What is worse, target attribute was also needed on function declaration, which made multi-versioning part of public API and part of function signature. In case of class member functions often also `impl` function had to be declared:
```cpp
class Summarization {
public:
    __attribute__(target("sse4.2")))
    uint64_t sum(const uint32_t* val, int size);

    __attribute__(target("avx")))
    uint64_t sum(const uint32_t* val, int size);

    __attribute__(target("avx2")))
    uint64_t sum(const uint32_t* val, int size);

private:
    uint64_t sumImpl(const uint32_t* val, int size);
};
```

Even though it makes this feature inconvenient to apply it was still useful.

## The New
GCC6 introduced much needed attribute `target_clone` which allowed to define multiple targets at once and apply this to single function body. Compiler will do the rest: clone function code and apply different optimizations to each copy.
```cpp
__attribute__(target_clone("sse4.2", "avx2", "default")))
uint64_t sum(const uint32_t* val, int size) {
    uint64_t result = 0;
    for (int i=0; i<size; i++) result += val[i];
        return result;
}
```

This greatly reduces the need for code duplication or workarounds using inlined implementations. Additionally, it is not needed (and generally not allowed) to apply the `target_clones` attribute in the function declaration, making multi-versioning internal implementation detail.

## Some details under the hood
<p align="center">
<img src="/assets/images/2020-07-27-migrating-multiversioning/under_the_hood.jpg.png">
</p>
Implementation behind dropping requirement for applying multi-versioning to declarations is quite interesting. It seems that resolver (function that selects and calls proper version of code based on runtime CPU properties) is now generated under signature of multi-versioned function itself. External code sees only single resolver function and is completely not aware of multi-versioning. As [seen in Godbolt](https://godbolt.org/z/bnv3qd) new-style multi-versioning generates `resolver` before function is even used:
<p align="center">
<img src="/assets/images/2020-07-27-migrating-multiversioning/godbold_new.png">
</p>

For old attribute `resolver` is generated only when some call is made ([link to Godbolt](https://godbolt.org/z/nEGqzT)):
<p align="center">
<img src="/assets/images/2020-07-27-migrating-multiversioning/godbold_old.png">
</p>
Function name itself is mangled, but not in a standard `C++` way, but with some GCC obscure magic. This explains why declaration has to indicate that multi-versioning is taking place so that compiler can find mangled implementation.
```cpp
jmp     _Z10sum(unsigned int const*, int)PKji
```
This probably breaks linking with code from other compilers when the multi-versioned function is exported, but I haven't checked that.

## How to?
As a summary, here are simple steps how to migrate from old to new multi-versioning
1. Reduce all copies of multi-versioned function with `target` attribute into one, with `target_clones` attribute.
2. Remove all workarounds with an inlined body of a function
3. Remove any reference to multi-versioning from function declarations
