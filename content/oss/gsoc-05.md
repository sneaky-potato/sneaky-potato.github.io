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

I designed the workflow like this:

1. A `luasched` scheduling decision is defined as the pair `(dsq, slice)`.
Both the values are related to a task which needs scheduling.

Here `dsq` is the sched_ext queue to which the task needs dispatching.
And `slice` is the time slice in nanoseconds which this task should get.

Our binding needs to return these values to the eBPF program which will eventually
call the scx dispatch function[^2] (which was later refactored as insert function[^3]).

`scx_bpf_dsq_insert(struct task_struct *p, u64 dsq, u64 slice_ns, u64 enq_flags);`

2. I added a kfunc similar to TC case for sched as well `bpf_luasched_run` which operates
on `struct task_struct`. It lets you write scheduling policy in Lua. However bpf layer will still
be needed.

`scheduler.c`: We will write a basic sched_ext eBPF program, start by defining the important stuff.

```c
#define DSQ_REALTIME 0
#define DSQ_BATCH    1
#define DSQ_DEFAULT  2

struct task_class {
	u64 dsq;
	u64 slice_ns;
};

static char runtime[] = "workload";

extern int bpf_luasched_run(const char *key, size_t key__sz, 
    struct task_struct *task, struct task_class *cls) __ksym;

/* create 3 dispatch queue for this example */
s32 BPF_STRUCT_OPS_SLEEPABLE(luasched_init)
{
    scx_bpf_create_dsq(DSQ_REALTIME, -1);
	scx_bpf_create_dsq(DSQ_BATCH, -1);
	scx_bpf_create_dsq(DSQ_DEFAULT, -1);
	return 0;
}
```

Next, define the dispatch policy, ours will be straight forward: schedule tasks from the queues
in this order: `DSQ_REALTIME` -> `DSQ_BATCH` -> `DSQ_DEFAULT`

```c
void BPF_STRUCT_OPS(luasched_dispatch, s32 cpu, struct task_struct *prev)
{
	if (!scx_bpf_dsq_move_to_local(DSQ_REALTIME)) {
		if (!scx_bpf_dsq_move_to_local(DSQ_BATCH)) {
			scx_bpf_dsq_move_to_local(DSQ_DEFAULT);
		}
	}
}
```

Now, the main enqueue logic which will use again use an eBPF map to cache the scheduling
decision based on task PID.

```c
void BPF_STRUCT_OPS(luasched_enqueue, struct task_struct *p, u64 enq_flags)
{
	pid_t pid = p->pid;
    /* check map for task PID */
	struct task_class *cls = bpf_map_lookup_elem(&task_classes, &pid);
	if (cls) {
		scx_bpf_dsq_insert(p, cls->dsq, cls->slice_ns, enq_flags);
		return;
	}

	struct task_class lua_cls = { 
        .dsq = DSQ_DEFAULT, 
        .slice_ns = SCX_SLICE_DFL
    };

    /* invoke Lua to get the scheduling decision */
	int ret = bpf_luasched_run(runtime, sizeof(runtime), p, &lua_cls);

	if (ret) {
		lua_cls.dsq = DSQ_DEFAULT;
		lua_cls.slice_ns = SCX_SLICE_DFL;
	}

    /* update the map once we know the scheduling decision from Lua */
	bpf_map_update_elem(&task_classes, &pid, &lua_cls, BPF_ANY);
	scx_bpf_dsq_insert(p, lua_cls.dsq, lua_cls.slice_ns, enq_flags);
}
```

Next, we need a cleanup task to clear the cache when task is stopped.

```c
void BPF_STRUCT_OPS(luasched_exit_task, struct task_struct *p, struct scx_exit_task_args *args)
{
	pid_t pid = p->pid;
	bpf_map_delete_elem(&task_classes, &pid);
}
```

Finally wire everything up:
```c
SEC(".struct_ops")
struct sched_ext_ops luasched_ops = {
	.init       = (void *)luasched_init,
	.dispatch   = (void *)luasched_dispatch,
	.enqueue    = (void *)luasched_enqueue,
	.exit_task  = (void *)luasched_exit_task,
	.name       = "luasched",
};
```

3. Now the second step: `workload.lua`
```lua
local sched = require("sched")
local scx   = require("linux.scx")

local REALTIME = 0
local BATCH = 1
local DEFAULT = 2

local policy = {
	{ pattern = "^nginx", dsq = REALTIME, slice = 1000000 }, -- 1ms
	{ pattern = "^firefox", dsq = BATCH, slice = 10000000 }, -- 10ms
}

local function log(command, dsq, slice)
	print(string.format("workload: [%s]: %d %d", command, dsq, slice))
end

local function workload(ctx)
	local task = ctx:task()
	for _, rule in ipairs(policy) do
		if task:comm():match(rule.pattern) then
			ctx:dsq(rule.dsq)
			ctx:slice(rule.slice)
			log(task:comm(), rule.dsq, rule.slice)
			return
		end
	end
	ctx:dsq(DEFAULT)
	ctx:slice(scx.SLICE_DFL)
end

sched.attach(workload)
```

4. Now we are ready to spin up our own scheduler:
```sh
sudo bpftool struct_ops register scheduler.o /sys/fs/bpf/luasched
```

The whole process is documented [here](https://github.com/luainkernel/lunatik/tree/sneaky-potato/gsoc26#workload-scheduler)
Example is a low level scheduler which assigns dispatch queues and time slices
based on task command names.

For this example, tasks matching command name `nginx` get a high priority (`DSQ_REALTIME`) 
and those matching `firefox` get a lower priority (`DSQ_BATCH`).


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
[^2]: Linux source code v6.12 - Bootlin Elixir | [tools/sched_ext/include/scx/common.bpf.h](https://elixir.bootlin.com/linux/v6.12/source/tools/sched_ext/include/scx/common.bpf.h#L39)
[^3]: Linux source code v7.2.2 - Bootlin Elixir | [tools/sched_ext/include/scx/compat.bpf.h](https://elixir.bootlin.com/linux/v7.2.2/source/tools/sched_ext/include/scx/compat.bpf.h#L356)

