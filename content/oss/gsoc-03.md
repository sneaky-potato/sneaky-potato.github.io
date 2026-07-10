---
title: "Google Summer of Code: 0x03"
date: 2026-07-03
description: "open-source"
tags: ["tech"]
---

> Time to go into some specifics.

Disclaimer: this post requires fair background of Linux operating
system and eBPF.

## Raw packets

Lunatik provides support for scripting XDP[^1]. XDP stands for [e**X**press **D**ata **P**ath](https://en.wikipedia.org/wiki/Express_Data_Path).
If you don't know what this means, it is a technology inside Linux which allows
you to process network packets down to the lowest level.

At XDP, you can run a hook for processing packets even before the network stack
touches it.

XDP lets you run a small program inside the NIC driver on every incoming packet before 
it enters the normal Linux networking stack.

Normally, a packet follows a long path, the driver allocates [`struct *sk_buff`](https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/skbuff.h#L775), 
wraps the incoming packet in it, and hands it over to the networking stack. 
From there, the packet is parsed, routed, filtered, and eventually delivered to an 
application or discarded.
```kroki {type=d2}
direction: right

driver: "Driver RX"
stack: "Network Stack"
application: "Application"
driver -> stack: sk_buff
stack -> application
stack -> stack: compute {
    style: {
        animated: true
    }
}
```

With XDP, you can insert a hook right at the NIC level. The important detail is at this hook
kernel has not yet created a `struct *sk_buff` so you're looking at raw pointer. It is enveloped 
as a lightweight [`struct xdp_md`](https://elixir.bootlin.com/linux/v7.1.2/source/include/uapi/linux/bpf.h#L6559)
context describing the raw packet buffer.
```kroki {type=d2}
direction: right

driver: "Driver RX"
xdp: "XDP"
stack: "Network Stack"
drop: "Drop Packet"
tx: "Transmit"
redirect: "Redirect"

driver -> xdp
xdp -> stack: "XDP_PASS"
xdp -> drop: "XDP_DROP"
stack -> stack: compute {
    style: {
        animated: true
    }
}
xdp -> tx: "XDP_TX"
xdp -> redirect: "XDP_REDIRECT"
```

Unlike `struct sk_buff`, `struct xdp_md` exposes only a pair of pointers
(`data` and `data_end`) into the packet buffer. The program is responsible
for parsing protocol headers manually.

Hence XDP allows you to write packet logic at the earliest stage of packet processing. This opens
doors to low latency packet processing. Imagine writing a simple load balancer at this stage.
Or operate firewall rules, packet sampling, DDoS mitigation. All of this without entering the 
complicated state machine which is the network stack.

## How does Lunatik interact with XDP?

The desgin is actually pretty interesting. Lunatik uses [BPF kernel functions](https://docs.kernel.org/bpf/kfuncs.html)
to achieve this[^2]. BPF is a layer which will be required for interacting with a low level subsystem like xdp.
So basically the user needs to create two files:
1. `xdp.c`: eBPF program which calls a kfunc `bpf_luaxdp_run` defined in Lunatik.
```c
extern int bpf_luaxdp_run(char *key, ...) __ksym;

static char runtime[] = "sni";

SEC("xdp")
int filter_https(struct xdp_md *ctx)
{
	/* send only client hello packets to Lunatik */
	if (!check_pkt_is_client_hello(ctx))
        return XDP_PASS;

	int action = bpf_luaxdp_run(runtime, ...);
	return action < 0 ? XDP_PASS : action;
}
```

2. `sni.lua`: contains the logic to return xdp verdict, but in Lua
```lua
local xdp    = require("xdp")
local action = require("linux.xdp")

local blacklist = {
	"ebpf.io": true,
}

local function filter_sni(ctx)
    local packet = ctx:packet()
    local argument = ctx:argument()
	if is_not_client_hello(packet) then
		return action.PASS
	end

    local sni = parse_sni(packet, argument)

	verdict = blacklist[sni] and "DROP" or "PASS"
    ctx:action(action[verdict])
	return
end

xdp.attach(filter_sni)
```

The whole process is documented [here](https://github.com/luainkernel/lunatik/blob/master/README.md#filter) in the project's README.
The example is a low level firewall which blocks all packets from the domain `ebpf.io`

Essentially this is what happens:
```kroki{type=d2}
direction: down

driver: "Driver RX"
ebpf: "eBPF Program\nxdp.c"
kfunc: "bpf_luaxdp_run()"
lua: "sni.lua"
stack: "Network Stack"

driver -> ebpf: struct xdp_md {
    style: {
        animated: true
    }
}
ebpf -> kfunc {
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
lua -> kfunc: XDP_PASS / XDP_DROP
kfunc -> ebpf
ebpf -> stack: XDP_PASS
```

## But, why?

If you know about netfilter tables and eBPF, you might ask why to even do all of this?
It has been recently demonstrated that eBPF is turing complete [^3]. But the eBPF 
verifier has strict limitations over how you can express code in eBPF. For instance,
the sni matching example above, if implemented in eBPF will be very rigid and non
intuitive. Also to load and compile eBPF programs is another headache. With this 
architecture users can hot reload a policy written in Lua.

If you've made it till here, kudos to you! Because now we have enough context when I 
can finally start explaining my project.

My project is essentially to extend this model to other Linux subsystems, namely
TC, Sched with a binding for eBPF maps.
 
---

[^1]: The eXpress Data Path: [Fast programmable packet processing in the operating system kernel](https://dl.acm.org/doi/10.1145/3281411.3281443)
[^2]: [XDPLua](https://victornogueirario.github.io/xdplua/)
[^3]: Isovalent | [eBPF: Yes, it's Turing Complete](https://isovalent.com/blog/post/ebpf-yes-its-turing-complete/)

