---
title: "The one with shrinking structure (Part 2)"
published: false
date: 2019-11-4
categories:
  - blog
tags:
  - C++
  - performance
  - cache
  - Packet Processing
  
---

**Know your data**
Standard types like uint8_t, uint16_t are perfectly fine for most use cases. But if we know the exact range of possible values and want our data to be really compact we can have more granularity by using bitfields.
What can we assume here:
There is so much physical ports possible on NIC. In this case 8 sounds as reasonable number (Napatech NICs from example can have up to 4 physical ports, 8 with some special wiring)
Limiting ethernet frame size to 8k (13 bits) seems like good enough assumption
64bit nanosecond timestamp is ~584 years, more than enough for some uses and definitely we could shrink it to some smaller size
Of course those assumptions are not true in general case, but there are always SOME assumptions that can be made for a given application. Sometimes it makes application less future-proof if for example full jumbo frames needs to be processed or there is new NIC with 10 ports. This should be always conscious decision.

So how structure with those new assumptions could look like?
