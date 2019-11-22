---
title: "The one with shrinking structure (Part II)"
published: false
date: 2019-11-27
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
	uint16_t timestampNs_high;
	uint32_t timestampNs_low;
	uint64_t* payload;
	uint64_t payloadHash;
	uint16_t size : 13;
	uint16_t physicalPort : 3;
	uint8_t isValid:1;
	uint8_t flag2:1;
	uint8_t flag3:1;
}
```
Unfortunatelly neither c++ standard nor gcc extensions have 48 bit types for x86 architecture, so there is need for some special accessors for timestamp. This is one of the tradeoffs mentioned above, more speed but at cost of compilcations.

**Tagged pointers**
So how much more we can compress those descriptors? Is there any unused space left in their memory representation? It turns out it is, just not that obvious and under some conditions.
For a moment lets think actually how much information basic data types keep.
| Type | Bits of information |
|-------|--------|
| uint8_t | 8 |
| uint8_t* | 64 |
| uint16_t* | 63 |
| uint32_t* | 62 |
| uint64_t* | 61 |
Let's explain that. `uint8_t` is simple, this is just integer type, every bit is meaningful. For `uint8_t` pointer there is also nothing ususual. To be able to address every possible byte in memory, so with 1 byte resolution, every value of 64bit pointer is needed. Interesting things start to happen with `uint16_t*`. To address those, only even memory addresses are needed, this is 2 bytes resolution and one last bit in pointer actually doesn't hold any valuable piece of information. This is hovever with important assumption that values are aligned! It will not work for values which are, for example due to structure packing, unaligned. It is the same for 32 and 64bit integers, but just pointer resolution is 4 and 8 bytes (still, assuming aligned pointers). So in `uint64_t` aligned pointer there are 3 bits of information that can be used to store something else. 