---
title: "Google Summer of Code 0x06: Maps"
date: 2026-09-17
description: "open-source"
tags: ["tech"]
---

> Why to leave eBPF maps in all of this?

## Motivation

At this point we can use Lunatik to script 3 solid subsystems: xdp, tc, sched_ext.
But the issue is the Lua scripts are not stateful.

Since these scripts run in softirq / hardirq path (per packet or per task) we need a
way to maintain state in between the invokations to make Lua do much more.
An example use case could be: a **runtime-aware** scheduler.

Something like:
```lua
local stats = map

local function workload(ctx)
    local task = ctx:task()
    local pid = task:pid()
    local runtime = task:runtime()

    local last_runtime = stats:lookup(pid) or runtime
    local delta = runtime - last_runtime

    stats:update(pid, runtime)

    if delta > 5 * MS then
        ctx:dsq(BATCH)
    else
        ctx:dsq(DEFAULT)
    end
end

sched.attach(workload)
```

The current runtime alone isn't enough. To know how much a task has run since
the previous invocation, we need to remember its previous runtime. Since
`workload()` is invoked repeatedly for different tasks, this state also needs to
be associated with the task: here, using its PID as the map key.

The important realization is the state should not be restricted to the local
script state but a global state.

## Enter eBPF maps

Maps provide a way for eBPF programs to communicate with each other (kernel space)
and with user space [^1].

This is perfect, since we are already building a bridge between Lunatik
and eBPF. The bridge will be incomplete without a support for Lua scripts to interact
with eBPF maps.

## Design

The kernel owns the map object and other program use it. Likewise Lua can be a consumer,
it need not own the map object itself. So we can design the separation like this:

- The kernel creates / frees / deletes the map.
- Lua can interact with the map, as in update, add, delete some key from the map.

This requires Lunatik to be able to use an existing map object somehow.

## Challenges

So, I quickly started looking for the right API to interact with these maps.
eBPF maps are created via `bpf(BPF_MAP_CREATE, ...)` and accessed via
`bpf_map_lookup_elem`, `bpf_map_update_elem`, and `bpf_map_delete_elem` helpers.
We want to access these via kernel's internal map API without going through syscall
interface.

The following APIs are exposed by kernel to use:
- All userspace calls (through libbpf) go through [`SYSCALL_DEFINE3()`](https://elixir.bootlin.com/linux/v6.19.2/source/kernel/bpf/syscall.c#L6272)
- [`bpf_map_get(u32 ufd)`](https://elixir.bootlin.com/linux/v6.19.2/source/include/linux/bpf.h#L2487)
- [`bpf_map_get_with_uref(u32 ufd)`](https://elixir.bootlin.com/linux/v6.19.2/source/include/linux/bpf.h#L2488)

Map resolution normally happens as follows:
- from userspace we call `open(path)` to get an fd on the pinned bpffs file.
- now we share this fd to kernel side
- we can call `bpf_map_get_with_uref(u32 ufd)` from kernel side to resolve the
  fd to pointer to `struct bpf_map`

This will need the file descriptor which needs to be created via
a process context. But Lunatik does not run in process context.

So we need some other way to recover the map object.

## Pinned maps

Fortunately, we can pin eBPF maps to a filesystem path like so:
```sh
sudo bpftool map create /sys/fs/bpf/runtime_stats \
    type hash \
    key 4 \
    value 16 \
    entries 4096 \
    name runtime_stats
```

This will create a hash type eBPF map, having key size 4, value size 16, limited to
4096 entries, named `runtime_stats` and pinned to the filesystem path `/sys/fs/bpf/runtime_stats`

Now, we just need a way to recover the map object from the pinned filesystem somehow.

## Maps API

We need to bypass the standard way and use other kernel helpers to
resolve map pointer directly from bpffs path. We can use `kern_path` to resolve 
the dentry without triggering the open handler, then read `inode->i_private` 
directly where bpffs stores the pointer to `struct bpf_map`. 

I verified this approach via a kernel module POC [^2].
```c
static int luabpf_map_open(lua_State *L)
{
    const char *path = luaL_checkstring(L, 1);
    struct path kpath;
    struct bpf_map *map;
    struct bpf_map **udata;
    int err;

    err = kern_path(path, LOOKUP_FOLLOW, &kpath);
    if (err)
        return luaL_error(L, "kern_path failed: %d", err);

    map = d_inode(kpath.dentry)->i_private;
    path_put(&kpath);

    if (!map || IS_ERR(map))
        return luaL_error(L, "not a valid bpf map path");

    bpf_map_inc(map); // increase reference counter

    udata = lua_newuserdata(L, sizeof(struct bpf_map *));
    *udata = map;
    luaL_setmetatable(L, "bpf.map");
    return 1;
}
```

- `lookup(key)` on the map pointer will return the value stored in the map for the provided key. We can reuse `luadata` for the actual types.
- `update(key, data)` on the map pointer will update the map for the provided key with the given data.
- `delete(key)` on the map pointer will delete the key from the map.
- Once we have the pointer to `struct bpf_map`, we could call kernel ops helpers defined [here](https://elixir.bootlin.com/linux/v6.19.2/source/include/linux/bpf.h#L106) to do lookup, update, delete.

#### Cleanup
- `close()` will cleanup the map from Lua, and decrease the reference counter via [`bpf_map_put(struct bpf_map *map)`](https://elixir.bootlin.com/linux/v6.19.2/source/include/linux/bpf.h#L2521)
---

[^1]: Maps - [eBPF docs](https://docs.ebpf.io/linux/concepts/maps/)
[^2]: GitHub - [map_kmod.c](https://github.com/sneaky-potato/minimal-scheduler/blob/master/map_kmod.c)

