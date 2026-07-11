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

So my job was to extend Lunatik and have it interact with tc. 

I added a similar kfunc called `bpf_luatc_run`
which operates on `struct sk_buff` and lets you write packet scheduling policies in Lua.
We still need bpf layer and much like xdp, we need 2 files:
1. `tc.c`: eBPF program which calls a kfunc `bpf_luatc_run` defined in Lunatik.
```c
extern int bpf_luatc_run(char *key, ...) __ksym;

static char runtime[] = "sni";

SEC("classifier")
int classify(struct __sk_buff *skb)
{
    __u32 key = skb->hash;
    /* flow_cache is an eBPF map which stores
     * priority for a specific traffic flow
     */
    __u32 *priority = bpf_map_lookup_elem(&flow_cache, &key);
    if (priority) {
        skb->priority = *priority;
        return TC_ACT_OK;
    }

	/* send only client hello packets to Lunatik */
	if (!check_pkt_is_client_hello(skb))
        return TC_ACT_OK;

	int action = bpf_luatc_run(runtime, ...);
	return action < 0 ? TC_ACT_OK : action;
}
```

2. `sni.lua`: contains the logic to set packet priority, but in Lua
```lua
local tc     = require("tc")
local action = require("linux.tc")

local policy = {
    ["netflix%.com"] = TC_H_MAKE(1, 20),
    ["zoom%.com"] = TC_H_MAKE(1, 10),
}

local function filter_sni(ctx)
    local skb = ctx:skb
    local sni = parse_sni(skb:data())

    for pattern, classid in pairs(policy) do
        if sni:match(pattern) then
            skb.priority = classid
            break
        end
    end
    ctx:action(action.ACT_OK)
end

xdp.attach(filter_sni)
```

3. Create the actual tc classes and qdiscs:
```sh
sudo tc qdisc del dev docker0 root 2>/dev/null
sudo tc qdisc add dev docker0 root handle 1: htb default 30
sudo tc class add dev docker0 parent 1: classid 1:10 htb rate 20mbit
sudo tc class add dev docker0 parent 1: classid 1:20 htb rate 10mbit
```

4. Attach the eBPF classifier on egress:
```
sudo tc filter add dev docker0 parent 1: bpf da obj tc.o sec classifier
```

The whole process is documented [here](https://github.com/luainkernel/lunatik/tree/sneaky-potato/gsoc26#sniclassify)
The example is a low level packet classifier which assigns priorities based on domain names.

For this example, packets from `netflix.com` get a lower bandwidth and those from
`zoom.com` get a higher bandwidth.

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
lua -> kfunc: skb.classid = HIGH
kfunc -> tc
tc -> qdisc
qdisc -> driver
```

---

[^1]: The eXpress Data Path: [Fast programmable packet processing in the operating system kernel](https://dl.acm.org/doi/10.1145/3281411.3281443)
[^2]: [XDPLua](https://victornogueirario.github.io/xdplua/)

