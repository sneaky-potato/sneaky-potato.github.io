---
title: "Google Summer of Code: 0x01"
date: 2026-06-25
description: "open-source"
tags: ["tech"]
---

> It is never too late to do open source.

## Huh?

It has been roughly 2 months since I started contributing to [Lunatik](https://github.com/luainkernel/lunatik)
as part of GSoC 2026. You can find the project description [here](https://summerofcode.withgoogle.com/programs/2026/projects/IHF0HPaF).

I have been dabbling into Linux traffic controller (tc), Extensible scheduler class (sched_ext)
and a lot of eBPF code on a daily basis for this project. I knew none of it when the year began.

I am a full time employee at one of the  energy technology giants, where I work
as a full stack software engineer. In my free time, I contribute to systems software
and low level tooling pojects and this is how I got introduced to this amazing
kernel project.

## Why?

Mostly GSoC is done by engineering students during their semester breaks because summers are free
(if they are not already occupied with an internship or something similar). I did not do GSoC during
my engineering mostly because I was really lazy.

I got a job through my campus placements, and after one year I decided to put in some time in open source
and get a crack at this. I think most students do GSoC for the following reasons:
- Money, $2000 is a nice thing to have for a student.
- Credentials, GSoC on resume looks good.
- Upskilling, you learn remote collaboration, best practices and deep tech.

I already earn through my day time job. I am doing GSoC because I got impressed by this kernel project. I
wanted to learn more and get some good systems experience. I cannot have that
in my day job. I also happen to have enough free time outside work to invest in
open source.

## Okay, but what is this project?

If I had to describe this project in the simplest terms, I'd say Lunatik allows one to script the kernel.
If you know a little about kernel scripting and development, you'd know that eBPF [^1] already allows scripting
the kernel. So why make another project?

The reason is, how Lunatik allows kernel scripting: in Lua. So essentially this project allows one to
control the *low level* parts of your machine using a *high level* language: **Lua**.

Let me give an example why Lua could be useful in expressing kernel scripts.
A good way to justify Lunatik is not just to say "you can script the kernel in Lua," but to show how much accidental complexity disappears.

Consider a simple character device `/dev/passwd` that generates random
printable ASCII characters whenever it is read from. The core logic is
extremely simple:

```lua
-- ref: https://github.com/luainkernel/lunatik#lunatik
-- /lib/modules/lua/passwd.lua
--
-- implements /dev/passwd to generate passwords
-- usage: $ sudo lunatik run passwd
--        $ head -c <width> /dev/passwd

local device = require("device")
local linux  = require("linux")
local stat   = require("linux.stat")

local driver = {name = "passwd", mode = stat.IRUGO}

function driver:read() -- read(2) callback
	-- generate random ASCII printable characters
	return string.char(linux.random(32, 126))
end

-- creates a new character device
device.new(driver)
```

The entire behavior of the device is expressed in a single line: generate a
random number in the printable ASCII range and convert it to a character.

Now compare this with a traditional Linux kernel module written in C. The logic
for generating the character is still only a couple of lines, but the developer
must also deal with device registration, file operation structures, userspace
memory handling (copy_to_user), module initialization and cleanup, error
handling, and build system integration.

In other words, the complexity of the feature is not the password generation
logic itself; it is the surrounding kernel infrastructure needed to expose that
logic safely to userspace.

This is just scratching the surface of what this project can do. With Lunatik you can
interact with: raw sockets, hid, probes, netfilter hooks [^2], kernel FIFO queues,
kernel threads, xdp and with my work: **tc**, **sched** and **eBPF maps**.

> The highlighted technologies are kernel subsystems resposible for traffic shaping,
> CPU scheduling and sharing data with other subsystems.

Featured article from LWN: [Lua in the kernel?](https://lwn.net/Articles/830154/)

Lunatik has been featured in Netdev conference multiple times:
- Netdev 0x14 (2020) [Linux Network Scripting with Lua](https://netdevconf.info/0x14/session.html?talk-linux-network-scripting-with-lua)
- Netdev 0x17 (2023) [Scripting the Linux Routing Table with Lua](https://netdevconf.info/0x17/sessions/talk/scripting-the-linux-routing-table-with-lua.html)
- Netdev 0x1A (2026) [Scripting Netfilter with Lua: A Cooperative Kernel-Userspace Pipeline](https://netdevconf.info/0x1A/sessions/talk/scripting-netfilter-with-lua-a-cooperative-kernel-userspace-pipeline.html)

Lunatik has been accepted in NetBSD ecosystem as well[^3]


## Why Lua?

The project could have used any high level language for the purpose. Lua was chosen
because of following reasons:
- Lua has a small footprint, it is lightweight. The entire interpreter wraps itself in around **250KB**.
- Lua is an embeddable language, so you can freely interact with it while
working in some other systems suited language (like **C**).

## Cost?

Okay so Lunatik makes **expressing** kernel scripting much simpler. But it comes with some cost.
You see, Lunatik invokes a modified Lua interpreter in kernel context and runs the script.
Along with the interpreter comes everything that makes Lua a high-level
language: dynamic typing, memory management, and garbage collection.

This means every operation carries some overhead compared to native kernel
code. A Lua callback must pass through the interpreter, Lua objects must be
allocated and managed, and the garbage collector may occasionally run. In
workloads where every microsecond matters, this overhead is not free.

In return, however, you get dramatically shorter development cycles and much
faster iteration. For many automation, networking, and prototyping workloads,
that trade-off is worth it.

---

[^1]: eBPF - Introduction, Tutorials & Community Resources [ebpf.io](https://ebpf.io/)
[^2]: Scripting Netfilter with Lua: A Cooperative Kernel-Userspace Pipeline [netdev 0x1A](https://netdevconf.info/0x1A/sessions/talk/scripting-netfilter-with-lua-a-cooperative-kernel-userspace-pipeline.html)
[^3]: Scriptable Operating Systems with Lua [netbsd.org](http://netbsd.org/~lneto/dls14.pdf)

