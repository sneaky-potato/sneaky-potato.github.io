---
title: "Google Summer of Code 2026: Lunatik"
date: 2026-09-27
description: "open-source"
tags: ["tech"]
---

Hey all

I just concluded Google Summer of Code with LabLua organization, where I
contributed to Lunatik. My project can be found [here](https://summerofcode.withgoogle.com/programs/2026/projects/IHF0HPaF).
The final work product link is [here](https://gist.github.com/sneaky-potato/9aa9128776457f57c1b60a6f6d2e60a4).

I added 3 bindings to Lunatik as part of this contribution:
- luatc: binding for Linux traffic control
- luasched: binding for Linux extensible scheduler class
- luabpf: binding for eBPF maps

I also have written a 7 part blog post on my journey with LabLua and GSoC, please do give it a read:
- [Google Summer of Code 0x01: Pilot](/oss/gsoc-01)
- [Google Summer of Code 0x02: Proposal](/oss/gsoc-02)
- [Google Summer of Code 0x03: XDP](/oss/gsoc-03)
- [Google Summer of Code 0x04: TC](/oss/gsoc-04)
- [Google Summer of Code 0x05: sched_ext](/oss/gsoc-05)
- [Google Summer of Code 0x06: Maps](/oss/gsoc-06)
- [Google Summer of Code 0x07: Conclusion](/oss/gsoc-07)

It was a large project and covering everything up in a single blog post was not possible. I've kept the parts short and to the point.

I also recorded a demonstration video covering the bindings I added to Lunatik:
[Lunatik eBPF Abstraction Layer](https://youtu.be/i2ViZGbANwQ)

The next step is to go deep with these binding and find a solid use-case to showcase, I already have some ideas, but I will keep experimenting.

Thanks

sneaky-potato

