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

---

# Chapter 1 — Beat B: The Code

Everything below is `drivers/gpu/drm/xe/` unless stated otherwise.
Line numbers are against **v7.3-rc1**.

## Where each character lives

| Character | Type | Declared in |
|-----------|------|-------------|
| the company | `struct xe_device` | `xe_device_types.h:107` |
| a building | `struct xe_tile` | `xe_tile_types.h:37` |
| a shop floor | `struct xe_gt` | `xe_gt_types.h:118` |
| a machine | `struct xe_hw_engine` | `xe_hw_engine_types.h:108` |
| the foreman | `struct xe_guc` | `xe_guc_types.h`, reached via `gt->uc.guc` |
| an open file | `struct xe_file` | `xe_device_types.h:620` |

Xe's convention: **`xe_foo_types.h` holds the struct, `xe_foo.h` holds the API,
`xe_foo.c` holds the implementation.** Learn that and you can find anything.

## 1. `struct xe_device` — `xe_device_types.h:107`

```c
struct xe_device {
	struct drm_device drm;                  /* :109  DRM base — MUST be first */
	struct intel_display *display;          /* :113  display, if built in     */
	struct xe_devcoredump devcoredump;      /* :117                           */
	struct intel_device_info {              /* :120  what this chip can do    */
		u8 tile_count;                  /* :148                           */
		u8 max_gt_per_tile;             /* :150                           */
		u8 gt_count;                    /* :154                           */
		u8 vm_max_level;                /* :156  page table depth         */
		u8 va_bits;                     /* :158  VA width, e.g. 48 or 57  */
		u8 has_usm:1;                   /* :221  unified shared memory    */
		u8 is_dgfx:1;                   /* :225  discrete? i.e. real VRAM */
		...
	} info;
	...
	struct xe_tile tiles[XE_MAX_TILES_PER_DEVICE];  /* :379  the buildings    */
	...
};
```

Two things to notice:

* `struct drm_device drm` is the **first member**. That is the classic
  container-of trick: given a `drm_device *`, `to_xe_device()` gets you back
  the `xe_device`.
* `info` is a big pile of one-bit feature flags filled in from the PCI ID at
  probe. Most `if (xe->info.has_foo)` checks you'll meet later come from here.
  `is_dgfx` is the one to remember for memory: discrete cards have real VRAM,
  integrated ones do not.

Also note there is **no `struct xe_vram` array on the device** — VRAM hangs
off tiles. That is fact #1 made concrete.

## 2. `struct xe_tile` — `xe_tile_types.h:37`

The whole struct is small enough to read in one go. The part that matters:

```c
struct xe_tile {
	struct xe_device *xe;
	u8 id;

	struct xe_gt *primary_gt;      /* render + compute */
	struct xe_gt *media_gt;        /* video            */

	struct xe_mmio mmio;

	struct {
		struct xe_vram_region *vram;        /* the warehouse        */
		struct xe_ggtt *ggtt;               /* the global addr book */
		struct xe_sa_manager *kernel_bb_pool;
		struct xe_sa_manager *reclaim_pool;
	} mem;

	struct xe_migrate *migrate;    /* the forklift crew */
	...
};
```

Read `tile->mem` out loud: *VRAM, GGTT, batch-buffer pool*. Those three are
the tile's memory identity. `tile->migrate` is the copy engine context used to
blit between them and — importantly, later — to write page tables.

## 3. `struct xe_gt` — `xe_gt_types.h:118`

The execution identity:

```c
struct xe_gt {
	struct {
		enum xe_gt_type type;     /* :125  MAIN or MEDIA              */
		u64 engine_mask;          /* :135  which engines exist here   */
		u8 id;                    /* :144                             */
	} info;

	struct xe_mmio mmio;                  /* :181                       */
	struct { struct xe_force_wake fw; } pm;/* :189 wake the hw up       */
	struct xe_tlb_inval tlb_inval;        /* :217  TLB invalidation     */
	struct { ... } usm;                   /* :228  page fault state     */
	struct xe_uc uc;                      /* :254  THE FIRMWARE         */
	struct xe_hw_fence_irq fence_irq[XE_ENGINE_CLASS_MAX]; /* :268      */
	struct xe_hw_engine hw_engines[XE_NUM_HW_ENGINES];     /* :274      */
	struct { ... } fuse_topo;             /* :294  how many EUs survive */
};
```

Compare this list with `xe_tile`'s and the split is obvious: **the tile has
memory words (vram, ggtt, migrate); the GT has execution words (engines,
firmware, forcewake, TLBs, fences).**

`tlb_inval` sitting on the GT is your first hint that memory and execution are
not fully separable — a page table change made for a tile must be invalidated
on every GT that could have cached it. That's Chapter 9.

## 4. The foreman is nested three deep

```c
struct xe_uc {              /* xe_uc_types.h */
	struct xe_guc guc;      /* the scheduler firmware      */
	struct xe_huc huc;      /* media decryption firmware   */
	struct xe_gsc gsc;      /* security controller         */
	struct xe_wopcm wopcm;  /* the carve-out they run from */
};
```

So the path to the scheduler is `gt->uc.guc`, and there is one **per GT** — a
media GT runs its own GuC. Helper: `exec_queue_to_guc(q)` in
`xe_guc_submit.c` walks `q->gt->uc.guc`.

## 5. `struct xe_hw_engine` — `xe_hw_engine_types.h:108`

```c
struct xe_hw_engine {
	struct xe_gt *gt;
	const char *name;                 /* "rcs0", "bcs0", "vcs0" ... */
	enum xe_engine_class class;
	u16 instance;
	u16 logical_instance;
	u32 mmio_base;
	struct xe_bo *hwsp;               /* HW status page             */
	struct xe_execlist_port *exl_port;/* only for the execlist path */
	struct xe_hw_fence_irq *fence_irq;
	struct xe_hw_engine_group *hw_engine_group;
};
```

`exl_port` is worth a note: Xe **does** carry a non-GuC execlist backend
(`xe_execlist.c`), but it exists only for early silicon bring-up. The GuC path
is the real one. Both plug into the same `drm_sched_backend_ops` shape — see
`xe_guc_submit.c:1976` vs `xe_execlist.c:321`. Same socket, two plugs.

## 6. Navigating the hierarchy

`xe_device.h:122` and `:130`:

```c
#define for_each_tile(tile__, xe__, id__) \
	for ((id__) = 0; (id__) < (xe__)->info.tile_count; (id__)++) \
		for_each_if((tile__) = &(xe__)->tiles[(id__)])

#define for_each_gt(gt__, xe__, id__) \
	for ((id__) = 0; (id__) < (xe__)->info.tile_count * (xe__)->info.max_gt_per_tile; (id__)++) \
		for_each_if((gt__) = xe_device_get_gt((xe__), (id__)))
```

Note the GT loop's bound: `tile_count * max_gt_per_tile`. GT IDs are a flat
index over a 2-D space, and `for_each_if` skips the holes where a tile has no
media GT. Going back up: `tile_to_xe()`, `gt_to_tile()`, `gt_to_xe()`.

Whenever you read Xe code and see `for_each_tile`, think *"doing something to
memory"*. `for_each_gt` — *"doing something to execution"*. It is a
surprisingly reliable tell.

## 7. The town being built — `xe_device_probe()` at `xe_device.c:945`

The probe order is not arbitrary; it is a dependency chain. Trimmed:

```c
int xe_device_probe(struct xe_device *xe)
{
	xe_pat_init_early(xe);              /* cache attribute tables         */
	xe_sriov_init(xe);
	xe_set_dma_info(xe);
	xe_mmio_probe_tiles(xe);            /* now tiles exist for real       */

	for_each_gt(gt, xe, id)
		xe_gt_init_early(gt);       /* :969  GT structs, no hw yet    */

	for_each_tile(tile, xe, id)
		xe_ggtt_init_early(tile->mem.ggtt);   /* :984                 */

	probe_has_flat_ccs(xe);
	xe_vram_probe(xe);                  /* :1001 how big is VRAM?         */

	for_each_tile(tile, xe, id)
		xe_tile_init_noalloc(tile); /* :1005 -> xe_ttm_vram_mgr_init  */

	xe_ttm_sys_mgr_init(xe);            /* :1014 system memory manager    */
	xe_ttm_stolen_mgr_init(xe);         /* :1019 stolen memory manager    */
	...
	xe_display_init_early(xe);
	for_each_tile(tile, xe, id)
		xe_tile_init(tile);         /* kernel_bb_pool, memirq         */
	xe_irq_install(xe);
	for_each_gt(gt, xe, id)
		xe_gt_init(gt);             /* GuC load, engines, submission  */
	xe_pagefault_init(xe);
	...
}
```

Read it as the town's construction schedule:

1. Survey the land (`mmio_probe_tiles`, `vram_probe`).
2. Put up the address books (`ggtt_init_early`) — **before** the warehouses,
   because GGTT lives at the bottom of VRAM.
3. Open the warehouses (`vram_mgr`, `sys_mgr`, `stolen_mgr`) — after this
   point `xe_bo_create()` works.
4. Only then hire the foreman and start the machines (`xe_gt_init`, which
   loads and starts the GuC).

The comment at `xe_device.c:1011` is explicit about the ordering constraint:
allocations are only allowed once the managers are up.

`xe_tile_init_noalloc()` at `xe_tile.c:182` is the hinge:

```c
if (IS_DGFX(xe) && !ttm_resource_manager_used(&tile->mem.vram->ttm.manager)) {
	err = xe_ttm_vram_mgr_init(xe, tile->mem.vram);
	xe->info.mem_region_mask |= BIT(tile->mem.vram->id) << 1;
}
```

`IS_DGFX` again — integrated parts never get a VRAM manager. And
`mem_region_mask` is what `DRM_IOCTL_XE_DEVICE_QUERY` later reports to
userspace as "here are the memory regions you may ask for".

## 8. The front desk — `xe_device.c:195`

```c
static const struct drm_ioctl_desc xe_ioctls[] = {
	DRM_IOCTL_DEF_DRV(XE_DEVICE_QUERY,       xe_query_ioctl,            DRM_RENDER_ALLOW),  /* :195 */
	DRM_IOCTL_DEF_DRV(XE_GEM_CREATE,         xe_gem_create_ioctl,       DRM_RENDER_ALLOW),  /* :196 */
	DRM_IOCTL_DEF_DRV(XE_GEM_MMAP_OFFSET,    xe_gem_mmap_offset_ioctl,  DRM_RENDER_ALLOW),  /* :197 */
	DRM_IOCTL_DEF_DRV(XE_VM_CREATE,          xe_vm_create_ioctl,        DRM_RENDER_ALLOW),  /* :199 */
	DRM_IOCTL_DEF_DRV(XE_VM_DESTROY,         xe_vm_destroy_ioctl,       DRM_RENDER_ALLOW),  /* :200 */
	DRM_IOCTL_DEF_DRV(XE_VM_BIND,            xe_vm_bind_ioctl,          DRM_RENDER_ALLOW),  /* :201 */
	DRM_IOCTL_DEF_DRV(XE_EXEC,               xe_exec_ioctl,             DRM_RENDER_ALLOW),  /* :202 */
	DRM_IOCTL_DEF_DRV(XE_EXEC_QUEUE_CREATE,  xe_exec_queue_create_ioctl,  ...),             /* :203 */
	DRM_IOCTL_DEF_DRV(XE_EXEC_QUEUE_DESTROY, xe_exec_queue_destroy_ioctl, ...),             /* :205 */
	DRM_IOCTL_DEF_DRV(XE_WAIT_USER_FENCE,    xe_wait_user_fence_ioctl,  ...),               /* :209 */
	DRM_IOCTL_DEF_DRV(XE_MADVISE,            xe_vm_madvise_ioctl,       ...),               /* :212 */
	...
};
```

This is the **entire userspace API**, seventeen entries. Xe deliberately has a
tiny uAPI compared to i915. Group them:

* **memory**: `GEM_CREATE`, `GEM_MMAP_OFFSET`, `VM_CREATE`, `VM_DESTROY`,
  `VM_BIND`, `MADVISE`, `VM_QUERY_MEM_RANGE_ATTRS`, `VM_GET_PROPERTY`
* **execution**: `EXEC_QUEUE_CREATE`, `EXEC_QUEUE_DESTROY`,
  `EXEC_QUEUE_SET/GET_PROPERTY`, `EXEC`, `WAIT_USER_FENCE`
* **introspection**: `DEVICE_QUERY`, `OBSERVATION`

Act I of this course is the first group. Act II is the second.

(There is a second, two-entry table at `xe_device.c:422`,
`xe_ioctls_admin_only[]`, used when the device is running as an SR-IOV PF.
Ignore it for now.)

## 9. The customer's folder — `struct xe_file` at `xe_device_types.h:620`

```c
struct xe_file {
	struct xe_device *xe;
	struct drm_file *drm;

	struct { struct xarray xa; struct mutex lock; } vm;          /* :627 */
	struct { struct xarray xa; struct mutex lock;
	         atomic_t pending_removal; } exec_queue;             /* :639 */

	u64 run_ticks[XE_ENGINE_CLASS_MAX];                          /* :657 */
};
```

One per `open()` of the render node. Two xarrays: **VMs** and **exec queues** —
exactly the two things a client owns. When userspace passes a VM id or an
exec-queue id into an ioctl, that id is an index into one of these xarrays.

This struct is the bridge between the two halves of the course. Act I fills the
first xarray. Act II fills the second.

## Try it yourself

```bash
# the whole hierarchy, in one screen
sed -n '37,90p'   drivers/gpu/drm/xe/xe_tile_types.h
sed -n '118,130p' drivers/gpu/drm/xe/xe_gt_types.h

# who touches tiles vs who touches GTs
git grep -c 'for_each_tile' drivers/gpu/drm/xe | sort -t: -k2 -rn | head
git grep -c 'for_each_gt'   drivers/gpu/drm/xe | sort -t: -k2 -rn | head
```

On a running machine: `/sys/kernel/debug/dri/0/` has `gt0/`, `gt1/`, `tile0/`
subdirectories — the same hierarchy, exposed.

## Checkpoint

You should now be able to answer, without looking:

1. Given a `struct xe_gt *gt`, how do you reach the GuC? the VRAM? the device?
2. Why does `xe_ggtt_init_early()` run before `xe_ttm_vram_mgr_init()`?
3. Why is there one GuC per GT rather than one per device?

Answers: (1) `gt->uc.guc`; `gt_to_tile(gt)->mem.vram`; `gt_to_xe(gt)`.
(2) GGTT is carved out of the bottom of VRAM, so its range must be reserved
before the buddy allocator is told what's free. (3) Because a GT is the reset
and scheduling domain — resetting the media GT must not disturb render.
