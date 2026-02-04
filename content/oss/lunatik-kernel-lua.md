---
title: "Lunatik, lua inside kernel boundary"
date: 2026-01-31
description: "lua, open-source, design-patterns"
tags: ["life"]
---

{{< lead >}}
Controlling low level components with a high level language
{{< /lead >}}

## What?
Lunatik is a framework for scripting the Linux kernel using Lua. It allows developers to write kernel-resident logic in a high-level language, while still interacting directly with kernel facilities such as networking, devices, timing, and system state.

At a high level, Lunatik consists of:
- A modified Lua interpreter that runs inside the Linux kernel
- A kernel-space C API to manage Lua runtimes and execute scripts
- Lua-based device driver layer
- User-space CLI tool (lunatik) to load, run, and manage kernel Lua environments
- Lua bindings that expose kernel facilities to Lua scripts

This architecture allows Lua to be used as a control and orchestration language inside the kernel, enabling rapid prototyping, instrumentation, and kernel-adjacent logic without writing everything in C.

## Why Script the Kernel?

Traditionally, Linux kernel development is done entirely in C, which offers maximum control and performance, but at the cost of:
- Long development and iteration cycles
- Higher risk of memory safety bugs
- Lower-level abstractions for expressing logic

Lunatik explores a different tradeoff: keeping performance-critical  components in C, while enabling higher-level logic to be expressed in Lua.
This makes Lunatik particularly useful for:
- Prototyping kernel logic
- Writing dynamic device behavior
- Experimentation and research

## Notable contributions
I've been an active contributor to Lunatik, focusing on performance, API correctness, and low-level networking support. Some of my notable contributions include:

### Improving Method Dispatch Performance
Lunatik previously used index-based (__index) lookups for method dispatch, which introduced extra allocations and overhead in hot paths.
I redesigned this mechanism to eagerly wrap class methods at initialization, eliminating repeated index lookups during runtime. This:
- Reduced dynamic allocations in hot paths
- Simplified the method call path
- Improved overall dispatch performance

This change significantly reduced per-call overhead in performance-sensitive code paths. PR: [#410](https://github.com/luainkernel/lunatik/pull/410)

### Enforcing Correct Raw Socket Binding Semantics

To prevent subtle API misuse, I enforced explicit pairing of (protocol, ifindex) when binding raw sockets.
This change:
- Prevents ambiguous or incorrect socket binding
- Makes API usage more explicit and self-documenting
- Improves correctness for low-level packet handling

PR: [#378](https://github.com/luainkernel/lunatik/pull/378)

### Others
- Added LLDP example using AF\_PACKET to demonstrate low-level packet I/O [#341](https://github.com/luainkernel/lunatik/pull/341)
- Refactored and standardized RAW packet socket patterns for safer reuse [#360](https://github.com/luainkernel/lunatik/pull/360), [#364](https://github.com/luainkernel/lunatik/pull/364)

