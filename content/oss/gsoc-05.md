---
title: "Google Summer of Code 0x04: Sched Ext"
date: 2026-08-14
description: "open-source"
tags: ["tech"]
---

> Next natural step is task scheduling in Lua

## Scheduler Class

After XDP and TC, Linux has another low level subsystem, called Sched Ext or
[**Ext**ensible **Sched**uler Class](https://docs.kernel.org/scheduler/sched-ext.html).

There is a concept of **Scheduler class** in the kernel. The struct is defined in the 
source code at [linux/kernel/sched.h](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/sched.h#L1702-L1760).

There are multiple scheduler **classes** in Linux, for example:
- realtime scheduler class: [linux/kernel/sched/rt.c](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/rt.c#L2357-L2391)
- completely fair scheduler class: [linux/kernel/sched/fair.c](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/fair.c#L10759-L10802)

Linux kernel 6.12 introduced [`sched_ext`](https://sched-ext.com/) as a new scheduling class that allows
pluggable CPU schedulers via eBPF.

Enables implementing and dynamically loading thread schedulers. No need for
recompiling the kernel and rebooting.

[`sched-ext/scx`](https://github.com/sched-ext/scx/) project is a collection of sched_ext schedulers and tools.
Schedulers in scx range from simple demonstrative policies to
production-oriented ones tailored for specific use cases:

- scx_simple : basic FIFO or least-run-time policy
- scx_nest : places tasks on high-frequency cores
- scx_lavd : is optimized for gaming workloads
- scx_rusty : partitions CPUs by last-level cache to improve locality
- scx_bpfland : threads that block frequently (i.e. perform many voluntary
  context switches per second) are assumed to be interactive, and thus
  prioritized

---

[^1]: Whirl Offload | [Understanding tc “direct action” mode for BPF](https://qmonnet.github.io/whirl-offload/2020/04/11/tc-bpf-direct-action/)

