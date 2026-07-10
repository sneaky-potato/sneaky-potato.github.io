---
title: "Google Summer of Code: 0x02"
date: 2026-06-26
description: "open-source"
tags: ["tech"]
---

> We start with some raw code contribution.

Disclaimer: the below process is not a roadmap for GSoC. It is just a document of my own
experience which I want to share with anyone interested.

## Contributor phase

I already knew about LabLua organization because one of my batchmates had participated in
GSoC in 2024 [^1]. Naturally curious I tried understanding the codebase. I was able to build
the project and run the example scripts, and I got interested. I checked open issues and
saw one related to [LLDP example](https://github.com/luainkernel/lunatik/issues/126).

I was able to get my first PR merged in the codebase on [Dec 29, 2025](https://github.com/luainkernel/lunatik/pull/341),
7 days after I had created the PR. The review process was rigorous, strict concerns over
code hygene, commit history and documentation. I was impressed by the amount of things I
learnt apart from technicalities of LLDP.

## Member phase

After 6 successful PRs merged, I was invited to join the GitHub organization on
17 Janurary, 2026.
I did not stop there, I contributed more, and it came naturally, as the issues I was
working on, kind of helped me see other issues with the codebase. One thing led to the
other and I was touching core Lunatik object and class code. I made some improvements,
specifically with latency.

2 PRs I had lot of fun while writing in this period are the following:
- wrap class methods eagerly instead of via __index [#410](https://github.com/luainkernel/lunatik/pull/410)
- add support for non-shared objects in shared class [#434](https://github.com/luainkernel/lunatik/pull/434)

## Proposal phase

Google announced LabLua was going to participate, as I was fairly acive on the project
Matrix, I kept regularly checking project ideas page.
The proposal which I think I could do was [Lunatik Bindng for sched_ext](https://github.com/labluapucrio/gsoc#lunatik-binding-for-sched-ext).

As a next step I asked the maintainers about this on the very day the mentoring
organizatons were announced (specifically February 19, 2026). The maintainer pushed me
to take on something more challenging, they said:

> I'm afraid, due to your current level of knowledge on Lunatik, sched-ext could be a bit easy to you..
> we expect that collaborators have adequate challenging level..

And I at once, started to reason what could be a nice proposal. I merged two
ideas mentioned on the page and came up with my own view. I created multiple
gists to share my ideas, and the biggest feedback I got was:

> I think it's important that you dig into the foundations of this work..
> you have walked a good way on the "how's".. but you need the "why's"
> (but I see you are already looking for it as from your TC doc)

And there I was, trying to find a solid **why** to my proposal. Because when you only know a 
hammer.. everything is a nail.. it shouldn't be the case here.. Lua is pretty much a glue language
and I needed to treat it like that.

## Proposal

My final proposal which I submitted after multiple review iterations [is here](https://github.com/sneaky-potato/gsoc/blob/master/lablua-apply.pdf).

I proposed to add Lunatik Bindings for TC, Sched, and eBPF maps. The project promises to
make Lunatik work with eBPF and not against it. If I have to summarize my project, I'd say:

> eBPF defines the structure, fast path.
> Lua provides the dynamic policy, slow path.

## GSoC results

On April 30, 2026 the accepted projects were announced and at 11:54 PM IST, I recieved a mail from GSoC and I was selected [^2].

---

[^1]: Project archive details | Google Summer of Code [2024 Program LabLua: Lunatik binding for Netfilter](https://summerofcode.withgoogle.com/archive/2024/projects/BIJAPZjf)
[^2]: Program Project | Google Summer of Code [2026 Program LabLua: Lunatik eBPF Abstraction Layer: TC, Scheduler binding and eBPF Maps](https://summerofcode.withgoogle.com/programs/2026/projects/IHF0HPaF)

