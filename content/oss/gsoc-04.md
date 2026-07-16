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
classid, filter [^1].
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
    stack -> filter -> qdisc
}

app: Application {
    near: top-left
}
driver: "Driver TX" {
    near: bottom-right
}

app -> kernel.stack
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
        skb->tc_classid = *priority;
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
    ["netflix%.com"] = TC_H_MAKE(1, 10),
    ["zoom%.com"] = TC_H_MAKE(1, 20),
}

local function filter_sni(ctx)
    local skb = ctx:skb
    local sni = parse_sni(skb:data())

    for pattern, classid in pairs(policy) do
        if sni:match(pattern) then
            skb.classid = classid
            break
        end
    end
    ctx:action(action.ACT_OK)
end

tc.attach(filter_sni)
```

3. This example uses `docker0` as the inteface, you can use your own interface.
Create the following script and run it with sudo.
It will create the actual tc classes and qdiscs attached to the interface:
```sh
#!/bin/bash
TC="tc"
IF="docker0"

# clean up [any] existing root qdisc
$TC qdisc del dev $IF root 2>/dev/null
# create root HTB qdisc defaulting to class 1:10
$TC qdisc add dev $IF root handle 1: htb default 10
# create parent and leaf class
$TC class add dev $IF parent 1: classid 1:1 htb rate 100mbit
$TC class add dev $IF parent 1:1 classid 1:10 htb rate 30mbit prio 1
$TC class add dev $IF parent 1:1 classid 1:20 htb rate 70mbit prio 2

$TC qdisc add dev $IF clsact
```

4. Attach the eBPF classifier on egress:
```
sudo tc filter add dev docker0 egress bpf da obj tc.o sec classifier
```

The whole process is documented [here](https://github.com/luainkernel/lunatik/tree/sneaky-potato/gsoc26#sniclassify)
The example is a low level packet classifier which assigns priorities based on domain names.

For this example, packets from `netflix.com` get a lower bandwidth (30%) and those from
`zoom.com` get a higher bandwidth (70%).

Quite similar to the architecture presented in last post:
```kroki{type=d2}
direction: down

stack: "Network Stack"
tc: "eBPF Program\ntc.o"
kfunc: "bpf_luatc_run()"

lua: "sni.lua"
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
lua -> kfunc: skb->tc_classid = HIGH
kfunc -> tc
tc -> qdisc
qdisc -> driver
```

## Verify

Watch the qdiscs in realtime.
```sh
watch -n 1 tc -s class show dev docker0
```

Generate some traffic usng curl.
```sh
docker run --rm -it alpine sh -c "apk add --no-cache curl && curl https://www.netflix.com > /dev/null"
```

---

[^1]: Whirl Offload | [Understanding tc “direct action” mode for BPF](https://qmonnet.github.io/whirl-offload/2020/04/11/tc-bpf-direct-action/)

