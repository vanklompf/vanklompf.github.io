---
title: "The one about conference - day two"
date: 2019-10-29
categories:
  - blog
tags:
  - events
---
Few updates from second day of [Open Source Summit in Lyon](https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/). 
<p align="center">
<img src="/assets/images/2019-10-28-OSS-conference/linus.jpg" width="500">
</p>


## Linus Torvalds in conversation with Dirk Hohndel
The man himself, Linus Torvalds said he hates doing public speeches, so formula of this keynote was rather panel or interview. Linus described what is his current role in kernel development and how his workflow looks. Surprisingly or not **he mostly reads emails**, a lot of them. And **his main role is to say no to people**. 
<p align="center">
<figure width="500">
  <img src="/assets/images/2019-10-28-OSS-conference/knight.jpg"/>
  <figcaption>Knights Who Say "Ni!"</figcaption>
</figure>
</p>
As much as negative that sounds this is something really crucial in a project of this size. There has to be some gatekeeper with deep understanding of problems and ability to keep code both high quality and readability. Discussion also concerned change of Linux from hobby, fun project into something like OSS enterprise.
<p align="center">
<figure width="500">
  <img src="/assets/images/2019-10-28-OSS-conference/enterprise.jpg"/>
  <figcaption>Knights Who Say "Ni!"</figcaption>
</figure>
</p>
Complexity and learning curve one need to climb to be able commit kernel changes grows every year. On the other hand Linus said that it was always easier to start contributing to edges of system like some less critical device drivers, utilities, tools etc. With growth of Linux edges also grows and there is more and more opportunities to contribute there.

And to my suprise **besides Linux and Git, there is a third major project started by Linus few years back!** It is called [Subsurface](https://en.wikipedia.org/wiki/Subsurface_(software)) and it is software for logging and planning scuba dives. It looks like he is a man of many talents!


## NVMe/TCP for your Data Center - Orit Wasserman, Lightbits Labs
It wasn't actually talk that was so compelling, but product it was about. Topic was NVMe/TCP, protocol for accessing remote NVMe devices using standard Linux TCP stack. Solution is developed by Israeli startup as OSS. It is competing with iSCSI and RDMA protocols, but is easier to set-up, does not require dedicated networking hardware and still **provides high performance in range of 10GB/s**, all on standard TCP stack! Which is impressive. It is ready to use and integrated to mainline 5.X kernels. In future it should get TCP offloading from vendors like Solarflare or Mellanox, which will increase performance and decrease CPU usage even further.


## Data Protection on NV-DIMM - Dr. Hannes Reinecke
Yet another talk about solid storage devices and there is still at least one more I'm interested in. This time it was about NV-DIMM/Optane/XPoint/"lost track of latest marketing names". Presentation was quite complex and deep, but also hugely entertaining, thanks to personal traits of dr. Reinecke. It was like mixing rocket science with Monty Python. But let's focus on the important stuff. In general Optane in form of NV-DIMMs are rather new creatures, documentation about its capabilities is scarce and use cases are not yet well recognized. So far I was aware that two modes of operation are available for NV-DIMM: 
  * [memory mode](https://thessdguy.com/intels-optane-two-confusing-modes-part-2-memory-mode/) where device works as DRAM, with "real" DRAM being cache for Optane solid state memory. There is no control over what and when gets written from DRAM to Optane. This is both transparent and uncontrollable.
  * [application mode](https://thessdguy.com/intels-optane-two-confusing-modes-part-3-app-direct-mode/) with **direct access to solid state memory as block device**, which was something new for me. This mode also provides persistence which opens quite new use cases!
  
Talk was focused on app mode. It turns out that Optane can use something like [XiP](https://en.wikipedia.org/wiki/Execute_in_place) known from embedded systems, where executables can be run directly from storage device, without the need of (expensive) copying to DRAM. **This feature is called DAX** and it goes further than that, allowing access to data on Optane without going through page cache, so it is not limited only to binary execution. At the same time NVME provides parallelism  where data are sharded across all available DIMM modules for better performance. This sharding is limited to single CPU domain, and cannot be done in hardware across muilti-socket systems. Additionally there is no way to have redundancy between modules using memory controller. This can be addressed with solutions like [MD RAID](https://www.thomas-krenn.com/en/wiki/Linux_Software_RAID), but than DAX functionality is lost. And **this** was what this talk was all about: how to keep DAX and also have control over redundancy/parallelism using Optane. It is easy for RAID0, as data are written only once, with direct mapping between page cache and device, more difficult for RAID1 where data are supposed to be written n-times. There are solutions for that, like delayed write for second volume, but it is not trivial due to problems like data inconsistency and long time needed to achieve eventual consistencys.

**For details, conclusions and better understanding I highly recommend watching this talk if/when it gets available online. Don't believe me, I was barely catching up!**


