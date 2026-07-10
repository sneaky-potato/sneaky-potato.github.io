---
title: "Google Summer of Code: 0x04"
date: 2026-07-10
description: "open-source"
tags: ["tech"]
---

> Time to do some packet scheduling in Lua

## Packet Scheduling

After XDP, Linux has another low level subsystem, called TC or 
[**T**raffic **C**ontroller](https://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html).
There is a userspace program with the [same name](https://man7.org/linux/man-pages/man8/tc.8.html)
which allows users to interact with the network scheduler.

There are some concepts associated with this whole architecture. There's qdiscs,
classid, filter.
But I won't go into details because that will bloat this post to another 10k words.

For simplicity you can consider this:
> XDP decides whether a packet should exist. TC decides how the packet should be treated.

I'd keep this simple and present you this architecture for the sake of this post.
Consider the path of the packet on its journey outside the host (from application to network card,
or as system admins would call it, **egress**):
```kroki{type=d2}
direction: right

kernel: Kernel {
    stack: "Network Stack"
    filter: "filter"
    qdisc: "qdisc"
    style: {
        stroke: blue
        font-color: purple
        stroke-dash: 3
        fill: white
    }
}

app: Application
driver: "Driver TX"

app -> kernel.stack
kernel.stack -> kernel.filter
kernel.filter -> kernel.qdisc
kernel.qdisc -> driver
```

Here in this diagram, **fiter** is a tc fiter, like HTB or [hierarchy token bucket](https://linux.die.net/man/8/tc-htb).
Likewise **qdisc** is tc queue discipline, which are queues with bandwidths.

## Classify SNI at egress

So my job was to extend Lunatik and have it interact with tc. I added a similar kfunc called `bpf_luatc_run`
which operates on `struct sk_buff` and lets you write packet scheduling in Lua.

Quite similar to the architecture presented in last post:
```kroki{type=d2}
direction: down

stack: "Network Stack"
tc: "eBPF Program\ntc.c"
kfunc: "bpf_luatc_run()"

lua: "classify.lua"
qdisc: "TC Qdisc"
driver: "Driver TX"

stack -> tc: struct sk_buff {
    style: {
        animated: true
    }
}
tc -> kfunc {
    style: {
        animated: true
    }
}
kfunc -> lua: invoke Lua {
    style: {
        animated: true
    }
}
lua -> lua: parse SNI
lua -> kfunc: skb.classid
kfunc -> tc
tc -> qdisc
qdisc -> driver
```

The whole process is documented [here](https://github.com/luainkernel/lunatik/tree/sneaky-potato/gsoc26#sniclassify)
The example is a low level packer classifier which assigns priorities based on domain names.
 
---

[^1]: The eXpress Data Path: [Fast programmable packet processing in the operating system kernel](https://dl.acm.org/doi/10.1145/3281411.3281443)
[^2]: [XDPLua](https://victornogueirario.github.io/xdplua/)

