---
title: "Google Summer of Code 0x04: sched_ext"
date: 2026-08-14
description: "open-source"
tags: ["tech"]
---

> Task scheduling in a dyanmic, garbage collected, interpreted language like
> Lua

## Scheduler Class

After adding XDP and TC bindings to Lunatik, the next subsystem is Linux's
`sched_ext` or [**Ext**ensible
**Sched**uler](https://docs.kernel.org/scheduler/sched-ext.html) Framework.
Unlike XDP and TC, however, scheduling is extremely latency-sensitive, which
makes running Lua code directly from the scheduling path an interesting
problem.

There is a concept of **Scheduler class** in the kernel. The struct is defined in the 
source code at [linux/kernel/sched.h](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/sched.h#L1702-L1760).

There are multiple scheduler classes in Linux, for example:
- realtime scheduler class: [linux/kernel/sched/rt.c](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/rt.c#L2357-L2391)
- completely fair scheduler class: [linux/kernel/sched/fair.c](https://github.com/torvalds/linux/blob/v5.5/kernel/sched/fair.c#L10759-L10802)

Linux kernel 6.12 introduced [sched_ext](https://sched-ext.com/) as a framework
[^1] that allows pluggable custom CPU schedulers via eBPF. This enables
implementing and dynamically loading thread schedulers. No need for recompiling
the kernel and rebooting.

[sched-ext/scx](https://github.com/sched-ext/scx/) project is a collection of
`sched_ext` schedulers and tools. Schedulers in scx range from simple
demonstrative policies to production-oriented ones tailored for specific use
cases:

- scx_simple : basic FIFO or least-run-time policy
- scx_nest : places tasks on high-frequency cores
- scx_lavd : is optimized for gaming workloads
- scx_rusty : partitions CPUs by last-level cache to improve locality
- scx_bpfland : threads that block frequently (i.e. perform many voluntary
  context switches per second) are assumed to be interactive, and thus
  prioritized

## BPF Scheduler

As I already mentioned, with `sched_ext` we can load schedulers during runtime
using eBPF. The important point is *eBPF* here. Because this means `sched_ext/scx` 
schedulers are only extensible through eBPF. Much like XDP and TC. So we can go
with a similar approach and add support for this via `luasched` a Lunatik
binding for scheduling.

## Motivation

Suppose we want:

- Nginx workers to get low latency scheduling
- Firefox background processes to receive larger time slices
- Everything else to use the default policy

Traditionally this logic would be hardcoded inside the scheduler like so:

```c
if (is_nginx(task))
    ...
else if (is_firefox(task))
    ...
```
There are two pain points:

- Every change is in eBPF, and the eBPF code is a pain to write.
- Every policy change requires recompilation.

With `luasched`, the scheduler asks Lua how a task should be treated in *Lua*
which is relatively simple to write.

```lua
local sched = require("sched")

local function schedule(ctx)
    local task = ctx:task()
    if task:comm():match("^nginx") then
        ctx:dsq(REALTIME)
        ctx:slice(20000000)
    else
        ctx:dsq(DEFAULT)
        ctx:slice(10000000)
    end
end

sched.attach(schedule)
```

## Caching results

Lua is a *dynamic*, *garbage collected*, *interpreted* language. These are
some words which typically don't go together with a latency sensitive work
like task scheduling. So we'd not want to invoke the Lua runtime each time
a task gets enqueued for scheduling. We can design an architecture where we
leverage the flexibility of Lua only when it is actually required.

The following diagram summarizes the architecture described above:
```kroki{type=d2}
vars: {
  d2-config: {
    layout-engine: elk
  }
}

enqueue: "Task enqueue"

scheduler: "sched_ext eBPF scheduler" {
  cache: "eBPF HASH MAP\nPID → task_class" {
    shape: cylinder
  }
  lookup: "Lookup PID\nin task_classes"
  insert: "scx_bpf_dsq_insert()"
}

kfunc: "bpf_luasched_run()" {
  near: top-right
}

lua: "Lua runtime" {
  policy: "workload.lua\n\nScheduling policy"
  near: center-right
}

dsq: "Dispatch queues" {
  realtime: "REALTIME\nDSQ 0"
  batch: "BATCH\nDSQ 1"
  default: "DEFAULT\nDSQ 2"
}

cpu: "CPU"

enqueue -> scheduler.lookup {
    style: {
        animated: true
    }
}

scheduler.lookup -> scheduler.cache: "lookup PID"
scheduler.lookup -> scheduler.insert: "hit\nDSQ+slice_ns"
scheduler.lookup -> kfunc: "miss" {
    style: {
        animated: true
    }
}

kfunc -> lua.policy: "invoke Lua" {
    style: {
        animated: true
    }
}
lua.policy -> scheduler.insert: "DSQ+slice_ns" {
    style: {
        animated: true
    }
}

scheduler.insert -> dsq.realtime
scheduler.insert -> dsq.batch
scheduler.insert -> dsq.default

dsq.realtime -> cpu
dsq.batch -> cpu
dsq.default -> cpu
```

---

[^1]: Phoronix | [Sched_ext Merged For Linux 6.12 - Scheduling Policies As BPF Programs](https://www.phoronix.com/news/Linux-6.12-Lands-sched-ext)

