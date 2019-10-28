---
title: "The one about conference - day one"
date: 2019-10-28
categories:
  - blog
tags:
  - events
---
Here are some first thoughts after day one of [Open Source Summit in Lyon](https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/). 

<p align="center">
<img src="/assets/images/2019-10-28-OSS-conference/stage.jpg" width="800">
</p>

### Initial thoughts
I am a first time attendee at this conference but also on any conference of this size. So far it is very good experience. Advice for first time attendees on conferences: take part in networking and “meet people” events! Even if you find it hard because you have “I haven studied computer science NOT to talk with people” attitude. I was able to sign-up for First-Time Attendee Breakfast and it was a really great experience. Not because I don't know how to move around and how things work on such conferences, but because I was able to meet cool people from Sony, embedded world and what not. So definitely worth it!

Few of more memorable presentations:
### Keynote: MDS, Fallout, Zombieland & Linux - Greg Kroah-Hartman
In my opinion this was one of the best keynotes of the day. Greg briefly introduced audience into current approach Linux community is taking against class of speculative execution vulnerabilities. He mentioned that hard (and controversial) stand done in the past by Debian on disabling Hyper-Threading was right and that more of OSS community is doing the same. He also acknowledged that most of the mitigation techniques introduces performance degradation, which everyone should get used to as there are no other solutions so far. And that there are more vulnerabilities of this class to come. There is also hope in all that, which can be found at (https://make-linux-fast-again.com/) - trade security for performance!

Greg also emphasized the significance of kernel security releases, to the point where he stated that no system without relatively recent kernel is secure. This doesn't mean everyone should immediately jump to kernel 5.X, but rather that staying on some older LTS version still requires updating to bug and security fixing releases. Which means I have some work to do when back in the office!

### Keynote: Open Source, Better, Faster, Stronger - Kelly Hammond, Intel
Oh my... This talk was showing that things are not going well in Intel. It was very vague, generic, full of marketing. I know that Intel is clearly behind in terms of CPU development, but their software stacks, libraries, tools are first class. Also they can always focus on different products: Optane, networking, embedded. But they decided not to, not during this talk at least. Yes, Optane was mentioned, but in a way that I have already seen on other closed presentation by Intel: generic, vague and not convincing. If I took one thing from this talk it was that on the server storage or CPU front Intel has nothing interesting to show right now.

### Analysis of Speculative and Traditional Execution Side Channel and Protection Mechanisms - Antonio Gomez, Intel 
This was really enlightening, fun talk. Antonio showed speculative executions vulnerabilities from differnet angle then Greg KH did during his keynote. Without being too defensive, without marketing sugarcoating it was honest and to the point. In short, side-channel attacks heve long history dating back to at least 1995, but they caught a lot of attention through the media two years ago. There are no successful exploits based on that, because there is so many moving parts like: lack of control over core exploit is running on, need to coexist on the same core with attacked software for longer time, need of very precise control of timing etc. There were also some interesting propositions of new solution for Spectre-like vulnerabilities. One of those was called gang-scheduling, where the scheduler was aware of security class of every process and processes from different security domains could not run on the same core. In theory this prevents majority of speculative execution vulnerabilities, because non-secure process will never share CPU resources (execution units) with secure one. This has also positive side effects on running AVX intensive software. [AVX2 and AVX512 are lowering down clocks on CPU](https://stackoverflow.com/questions/56852812/simd-instructions-lowering-cpu-frequency), which connected with Hyper Threading can cause slowing down software not using AVX but running on the same core. Gang-scheduling can try to run AVX intensive threads on the same cores reducing avoiding performance penalty wherever possible.
