# Chapter 1 — The Cast

> Beat A: the story. (Beat B, the code, is appended once Beat A lands.)

## The factory town

Think of the GPU as a **factory town** that does work for customers
(applications) who never set foot inside it. They send in orders and raw
material; finished goods come back out. Everything in the `xe` driver is
either a building in that town, a worker, a warehouse, a ledger, or a
messenger.

Let's meet them in the order they were built.

---

## The hardware side

### 1. `xe_device` — the company

One PCI device. One `xe_device`. This is the whole company: it owns the PCI
resources, the DRM device that userspace opens as `/dev/dri/renderD128`, the
TTM device (the warehouse system), and an array of tiles.

Everything else hangs off this.

### 2. `xe_tile` — a factory building

A modern Intel GPU can be built from more than one physical **tile** — a
self-contained slab of silicon. Think of each tile as a separate factory
building on the same campus.

What makes a tile a building rather than just a room is that **it has its own
memory and its own address book**:

* its own **VRAM** — the on-site warehouse,
* its own **GGTT** — the building's single global address book,
* its own **migrate** context — its own forklift crew for moving crates,
* its own MMIO window.

This is the single most important structural fact in Xe's memory management:
*memory is per-tile*. When you hear "VRAM0" and "VRAM1" later, that is tile 0's
warehouse and tile 1's warehouse. They are different places.

### 3. `xe_gt` — the shop floor

Inside a building, the **GT** (Graphics Technology) is the shop floor: the
execution hardware, its caches, its TLBs, its reset domain, its firmware.

A tile can have up to two GTs:

* the **primary GT** — render and compute,
* the **media GT** — video decode and encode (on newer parts this is a
  genuinely separate GT with its own reset domain and its own GuC).

So the hierarchy is: device → tiles → GTs. Memory belongs to the *tile*.
Execution belongs to the *GT*. Keep those two apart in your head and half of
Xe's structure explains itself.

### 4. `xe_hw_engine` — the machines

On the shop floor stand the actual machines. Each is a hardware engine with its
own command streamer:

| Class | Nickname | Does |
|-------|----------|------|
| `RENDER` | RCS | 3D rendering |
| `COMPUTE` | CCS | compute kernels |
| `COPY` | BCS | blitter — bulk memory copies |
| `VIDEO_DECODE` | VCS | video decode |
| `VIDEO_ENHANCE` | VECS | video post-processing |
| `OTHER` | | e.g. GSC, the security controller |

An engine consumes a **ring buffer** of commands. That's its conveyor belt. It
does not know about processes, files or fairness. It executes what's on the
belt.

### 5. The GuC — the foreman

The **GuC** (Graphics microcontroller) is a small CPU inside the GT running
Intel firmware. In Xe it is not optional and not a helper — it is *the
scheduler*. The host driver does not write to engine registers to run work.
It hands contexts to the GuC and the GuC decides which context occupies which
engine, when to time-slice, and when to preempt.

This is the defining difference between `xe` and the older `i915`. In `i915`
the kernel driver was the dispatcher. In `xe` the kernel driver is the
*supplier*: it prepares work and gives it to the foreman.

The driver talks to the foreman over a shared-memory mailbox called **CT**
(CTB — command transport buffer). Messages going down are **H2G** (host to
GuC), messages coming back are **G2H**.

---

## The software side

Now the paperwork. These are not hardware — they are the kernel's models of it.

### 6. DRM and GEM — the front desk

**DRM** is the Linux graphics subsystem: the `/dev/dri/*` character devices,
the ioctl dispatch table, file handles, and generic services.

**GEM** (Graphics Execution Manager) is DRM's object system. Its core type,
`drm_gem_object`, gives every buffer a lifetime (refcount), a **handle** that
userspace can name it by, and a `dma_resv` — a reservation object that tracks
who is using the buffer and which fences must signal before it is safe to move
or free.

GEM is the front desk: it issues receipt numbers for crates. It does not know
where the crates physically are.

### 7. TTM — the warehouse manager

**TTM** (Translation Table Manager) is the part that *does* know where crates
are, and moves them.

TTM's worldview:

* memory comes in **memory types** (system RAM, GTT-mapped system RAM, VRAM,
  stolen memory),
* each type has a **resource manager** that allocates space inside it,
* every buffer has a **placement** — the list of memory types it is allowed to
  live in, in preference order,
* buffers can be **evicted** — kicked out of VRAM to make room, with their
  contents copied,
* an **LRU list** decides who gets evicted first.

TTM is the warehouse manager who says: "warehouse A is full, crate 9 hasn't
been touched in a while, move it to the off-site warehouse, here's the
paperwork, here's the forklift."

`xe_bo` is Xe's crate: a `ttm_buffer_object` (which itself embeds a
`drm_gem_object`) plus Xe-specific fields.

### 8. `drm_gpuvm` — the customer's private map

A crate sitting in a warehouse is useless until the GPU can *address* it.
Each customer gets a private virtual address space — a **VM** — and a ledger
recording which piece of which crate is visible at which address.

`drm_gpuvm` is DRM's shared implementation of that ledger: an interval tree of
**`drm_gpuva`** entries (virtual address range → buffer + offset), plus the
logic to work out what a new mapping does to existing ones (split this one,
shrink that one, delete the third).

Xe wraps it: `xe_vm` embeds a `drm_gpuvm`, and `xe_vma` embeds a `drm_gpuva`.

### 9. `drm_sched` — the dispatcher

**`drm_gpu_scheduler`** is DRM's shared scheduling front end. Its vocabulary:

* a **job** is one unit of work,
* an **entity** is a queue of jobs that must run in order,
* jobs have **dependencies** expressed as fences,
* when all dependencies signal, the scheduler calls the driver's `run_job()`.

In Xe, one `drm_sched_entity` maps to exactly one hardware context — the
**exec queue**. `drm_sched`'s job is only to hold jobs until their
dependencies are met and then hand them to the GuC. The real scheduling
decisions are the foreman's.

### 10. `dma_fence` — the receipt everybody trusts

A **`dma_fence`** is a one-shot "this is done" signal that can be waited on by
anyone in the kernel, and shared across drivers. It is the universal currency
of the graphics stack: "the copy finished", "the bind finished", "the frame
rendered".

Fences are what let memory management and scheduling talk to each other
without knowing anything about each other. TTM does not know what a GPU job
is; it only knows "I must not move this crate until these fences signal".

---

## The one diagram

```
                       xe_device  (the PCI device / the company)
                            |
        +-------------------+-------------------+
        |                                       |
    xe_tile 0                               xe_tile 1
    |  VRAM0  GGTT0  migrate                |  VRAM1  GGTT1  migrate
    |                                       |
    +-- xe_gt (primary: RCS/CCS/BCS) + GuC  +-- xe_gt (primary) + GuC
    +-- xe_gt (media:   VCS/VECS)    + GuC  +-- xe_gt (media)   + GuC


   userspace process
        |
        |  drm file  ->  xe_file
        |
        +-- xe_bo      crates            (GEM + TTM)
        +-- xe_vm      private map       (drm_gpuvm)  -> xe_vma -> xe_bo
        +-- xe_exec_queue  order queue   (drm_sched_entity) -> GuC -> engine
```

## The two sentences to remember

1. **Memory is per-tile; execution is per-GT.**
2. **The kernel driver prepares work and owns memory; the GuC decides what
   runs.**

Everything in the next nineteen chapters is detail hung on those two hooks.
