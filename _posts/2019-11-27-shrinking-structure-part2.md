---
title: "The one with shrinking structure (Part II)"
published: false
date: 2019-11-29
categories:
  - blog
tags:
  - C++
  - performance
  - cache
  - Packet Processing
  
---

In (the last post)[] we have been looking on to how size of data, specifically packet descriptors, impacts processing performance. As it turns out that reducing data size is really important in gaining speed. In todays post I will try to present, working on the same packet processing example, even few additional, more aggresive and intrusive methods to reduce data size.


**Know your data**
Standard types like uint8_t, uint16_t are perfectly fine for most use cases. But if we know the exact range of possible values and want our data to be really compact we can have more granularity by using bitfields and share bytes/words between few separate fields.
What can we assume here:
 * There is only so much physical ports possible on NIC. In this case 8 sounds as reasonable number (Napatech NICs from example can have up to 4 physical ports, 8 with some special wiring)
 * Limiting ethernet frame size to 8k (13 bits) seems like good enough assumption
 * 64bit nanosecond timestamp is ~584 years, more than enough for some uses and definitely we could shrink it to some smaller size, like 48 bits
Of course those assumptions are not true in general case, but **there are always some assumptions that can be made for a given application.** Sometimes it makes application less future-proof. In case of limiting frame size, jumbo frames won't work. And sometimes it makes application more complex. If we reduce timestamp size so much that wrapping around is possible we need to introduce some kind of logic supporting that. Therefore this should be always conscious decision, based on both measurements and common sense.

This is how structure with those new assumptions looks like:
```cpp
struct PacketDescriptorWithAssumptions {
	uint64_t timestamp : 48;
	uint64_t size : 13;
	uint64_t physicalPort : 3;
	uint64_t* payload;
	uint64_t payloadHash;
	uint8_t isValid:1;
	uint8_t flag2:1;
	uint8_t flag3:1;
}
```
Unfortunatelly neither c++ standard nor gcc extensions have 48 bit types for x86 architecture, but bitfields works just fine here. Of course accessing and setting bitfield values will be slightly slower than using basic integer types. Size of  new structure is **25 bytes**, so thre were some minor savings. Processing performance is now **000 Mpps** RX and **000 Mpps** TX.

**Tagged pointers**
So how much more we can compress those descriptors? Is there any unused space left in their memory representation? It turns out it is, just not that obvious and under some conditions.
For a moment let's think actually how much information basic data types keep.
| Type | Bits of information |
|-------|--------|
| uint8_t | 8 |
| uint8_t* | 64 |
| uint16_t* | 63 |
| uint32_t* | 62 |
| uint64_t* | 61 |
Let's explain that. `uint8_t` is simple, this is just integer type, every bit is meaningful. For `uint8_t` pointer there is also nothing ususual. To be able to address every possible byte in memory, so with 1 byte resolution, every value of 64bit pointer is needed. Interesting things start to happen with `uint16_t*`. To address those, only even memory addresses are needed, this is 2 bytes resolution and one last bit in pointer actually doesn't hold any valuable piece of information. This is hovever with important assumption that values are aligned! It will not work for values which are, for example due to structure packing, unaligned. It is the same for 32 and 64bit integers, but just pointer resolution is 4 and 8 bytes (still, assuming aligned pointers). So in `uint64_t` aligned pointer there are 3 bits of information that can be used to store for example `physicalPort`. Memory representation oif this will look like that:
```
ddddddddd|ddddddddd|ddddddddd|ddddddddd|ddddddddd|ddddddddd|ddddddddd|dddddTTT
```
Where `d` is pointer data and `T` is "tag", part of pointer we can use for other purposes.
Here is `PacketDescriptor` implementation using tagged pointer. Check (source code)[] for accessor extracting tag and pointer body:
```cpp
struct PacketDescriptorTagged
{
	uint64_t timestamp : 48;
	uint64_t size : 13;
	uint64_t isValid:1;
	uint64_t flag2:1;
	uint64_t flag3:1;
	struct
	{
	    uintptr_t payload : (sizeof(intptr_t)*CHAR_BIT-3);
	    uintptr_t physicalPort : 3;
	} tagged;
	uint64_t payloadHash;
};
```
`uintptr_t` is unsigned integer type capable of holding a pointer. Putting it here allows using bitfields and makes casting to real pointer easier. Size of  new structure is **24 bytes**, so savings were really miniscule. Processing performance is now **000 Mpps** RX and **000 Mpps** TX.

**"Stamped" pointers**
Another trick that we can use relies on architecture of x86 processors, so it is absolutely not portable. In theory on 64bit machines there is 10^19 addressable bytes, which is 16 EiB (Exabytes). And only this amount of memory requires full range of pointers. In reality current 64bit systems uses no more than 48bits to address virtual memory (even less to address physical memory). You can check that by running:
```
cat /proc/cpuinfo | grep address
address sizes   : 39 bits physical, 48 bits virtual
```
Using only 48 bits out of 64bit pointer gives us whole 2 bytes of data to use! Amazing! But not so fast. Although those upper bytes in pointer are not used to store  information, there are certain requirements they must fulfil. All 16 upper bits needs to have the same value as bit 47. This is called canonical address and if this requirement is not met, the processor will raise an exception. Handling canonical pointer will definitely cause some overhead in processing. Fortunatelly most operating systems assign only lower half of address space, where bit 47 is zero. This is another assumption, but lets go with that. Name "stamped" was taken from (implementation of this pattern in `folly` library)[https://github.com/facebook/folly/blob/master/folly/experimental/StampedPtr.h]

This is implementation using "stamped" pointer:
```cpp
struct PacketDescriptorStamped
{
	struct
	{
		uintptr_t size : 13;
		uintptr_t isValid:1;
		uintptr_t flag2:1;
		uintptr_t flag3:1;
	    uintptr_t payload : ((sizeof(intptr_t)-2)*CHAR_BIT-3);
	    uintptr_t physicalPort : 3;
	} tagged;
	uint64_t payloadHash;
	uint64_t timestamp : 48;
};
```
