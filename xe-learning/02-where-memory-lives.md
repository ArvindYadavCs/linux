# Chapter 2 — Where Memory Lives

> Beat A: the story.

Chapter 1 gave us the town. Now we open the warehouses.

Before any GPU work happens, one question must be answered for every single
buffer: **where do these bytes physically sit?** Everything else in Act I —
binding, page tables, eviction, faulting — is downstream of that one question.

---

## 1. Four kinds of storage

Xe knows exactly four places a buffer can live. TTM calls them **memory
types**; Xe names them `XE_PL_*`.

### `XE_PL_VRAM0` / `XE_PL_VRAM1` — the on-site warehouses

Real device memory, soldered next to the GPU. Enormous bandwidth — this is
where you want everything the GPU touches in a hot loop.

There is one per tile. VRAM0 is tile 0's warehouse, VRAM1 is tile 1's.
They are genuinely different buildings: tile 1 reaching into VRAM0 is a
cross-tile trip, not a local access.

Only discrete cards have these. On an integrated GPU there is no VRAM at all,
and the whole memory-type list collapses to system memory.

### `XE_PL_TT` — the annex warehouse

Ordinary system RAM (the same DDR your CPU uses), **prepared so the GPU can
reach it**. "Prepared" means two things have happened: the kernel has actually
allocated the pages, and those pages have been DMA-mapped so the device has
bus addresses for them.

Slower than VRAM — every access crosses PCIe — but effectively unlimited, and
it's the only kind of storage an integrated GPU has.

The name is historical: **TT = Translation Table**, from the days when
system pages had to be entered into an aperture table to be reachable.

### `XE_PL_SYSTEM` — the loading yard

This is the one that confuses everybody, so slow down here.

`XE_PL_SYSTEM` is **not** a place where the GPU can find data. It is the
*parked* state: the buffer object exists, it has an identity and a size, but
it has no device-reachable backing right now. The pages may not even be
allocated yet.

Think of it as the paperwork existing while the crate sits in the yard with no
forklift route to it. A buffer sitting in `XE_PL_SYSTEM` must be **moved** to
`XE_PL_TT` or VRAM before the GPU can use it.

This is why newly created buffers start in `XE_PL_SYSTEM`, and why swapped-out
buffers go back to it. It is a state, more than a location.

> **The one distinction to keep:** `XE_PL_SYSTEM` = system RAM the GPU
> *cannot* reach. `XE_PL_TT` = system RAM the GPU *can* reach. Same physical
> DDR, different readiness.

### `XE_PL_STOLEN` — the locked closet

A chunk of memory the firmware carved out before Linux even booted — the
BIOS "stole" it for the graphics device. The display engine's initial
framebuffer lives there, which is why you see a picture before any driver
loads.

It's small, it has awkward access rules, and it mostly matters for display and
a few hardware quirks. Mentally file it and move on.

---

## 2. Placement: a wish list, not an address

Here is the design decision that makes TTM work.

When you create a buffer you do **not** say "put this at VRAM address
0x4000000". You hand over a **wish list**: *"VRAM0 if you can; system memory
if you must."*

Two structures express this:

* A **`ttm_place`** is one option on the list. It names a memory type, and can
  add conditions: *must be physically contiguous*, *must be in the
  CPU-visible window*, *prefer the top of the range*.
* A **`ttm_placement`** is the ordered list of those options.

The list is ordered by preference. TTM walks it top to bottom and takes the
first option it can satisfy. If nothing works, it starts evicting other
buffers to make one work, and tries again.

This indirection is the whole reason a GPU can run more work than fits in
VRAM. Nobody ever holds a hard address; everybody holds a wish list, so the
driver is always free to move things.

---

## 3. The warehouse clerks

A wish list needs somebody to turn it into an actual spot. That is a
**resource manager** — one per memory type, registered with TTM at probe.

A manager has exactly one important job: *given a request and a `ttm_place`,
carve out space and return a receipt.* The receipt is a **`ttm_resource`**:
"you own this much space, starting here, in this memory type."

What's interesting is that Xe's three managers are three **completely
different algorithms**, because the three warehouses have different problems.

### The VRAM clerk — a buddy allocator

VRAM is a fixed, precious, fragmentation-prone resource where physical
contiguity genuinely matters (big pages, display scanout, CPU mapping). So it
gets the heavy machinery: a **buddy allocator**.

A buddy allocator repeatedly halves a large block until it finds the smallest
power-of-two chunk that fits your request, and merges neighbours back together
on free. It can answer requests like *"contiguous, 2MB-aligned, and only from
the first 256MB"* — which the other clerks cannot.

An allocation is a **list of blocks**, not a single extent, unless you
explicitly demanded contiguity.

### The system-memory clerk — an accountant

For `XE_PL_TT` there is nothing to allocate an *address* for. System pages
don't need a coordinate in a warehouse — the GPU will reach them through page
tables, wherever they happen to be.

So this clerk does almost nothing: it checks "would this push us past total
system RAM?", says yes or no, and hands back a receipt with no address on it.
The actual page allocation happens later, by a different mechanism (Chapter 3).

That asymmetry is worth pausing on. **A `ttm_resource` for VRAM says where.
A `ttm_resource` for TT says only how much.**

### The stolen-memory clerk — a simple range allocator

Small fixed region, first-fit allocation out of an interval tree. Nothing
clever, because nothing clever is needed.

---

## 4. The window problem

One complication that shapes a lot of Xe code, and it's a nice physical story.

The CPU reaches device VRAM through a **PCI BAR** — a window in the CPU's
address space mapped onto the card's memory. Historically that window was
small: 256MB, onto a card with 16GB of VRAM. So **most of VRAM is invisible to
the CPU**. The GPU sees all of it; the CPU sees a slice.

(Modern systems with *resizable BAR* enabled can map all of it. Many still
can't.)

So the warehouse has a **service window** at the front. Crates the customer
needs to reach by hand must be stacked near that window. Crates only the
factory touches should be stacked at the back, deliberately, so they don't
waste window space.

That's exactly what the placement flags encode:

* a buffer the CPU must map → *constrain it to the visible range*
* a buffer the CPU will never touch → *allocate top-down*, far from the window

And the VRAM clerk keeps a second counter — not just "how much VRAM is free"
but **"how much *visible* VRAM is free"** — because you can run out of window
space while having plenty of VRAM left.

---

## 5. Who decides the wish list?

Two sources, meeting in the middle:

* **Userspace** says which regions it will accept, via a bitmask it got from
  `DEVICE_QUERY`. (Remember `mem_region_mask` being built during probe in
  Chapter 1? That's the same mask coming back around.)
* **The driver** adds the conditions it knows about: this is a page table, so
  it needs specific alignment; this is pinned, so it must be contiguous; the
  CPU will mmap this, so keep it in the visible window.

Those combine into a single `u32` of `XE_BO_FLAG_*` bits, and one function
turns that `u32` into the `ttm_placement` wish list. That translation is the
first thing that happens when any buffer is born — and it's the first thing
we'll read in Beat B.

---

## The picture

```
        ttm_device (one per xe_device)
             |
   +---------+-----------+-------------+--------------+
   |         |           |             |              |
XE_PL_    XE_PL_TT   XE_PL_VRAM0   XE_PL_VRAM1   XE_PL_STOLEN
SYSTEM    (annex)     (tile 0)      (tile 1)      (closet)
(yard)
   |         |           |             |              |
  TTM's   accountant   buddy         buddy        range alloc
 built-in              allocator     allocator
  stub


  a buffer's wish list:            what it becomes:
  ttm_placement {                  ttm_resource {
     [0] VRAM0, contiguous            mem_type = XE_PL_VRAM0
     [1] TT,    fallback              start    = <block address>
  }                                   size     = ...
                                   }
```

## The two sentences to remember

1. **Nobody owns an address; everybody owns a wish list.** That's what makes
   eviction and migration possible at all.
2. **`XE_PL_SYSTEM` is a parking state, not a place the GPU can read.**

---

## Checkpoint

1. An integrated GPU has no VRAM. What does a typical `ttm_placement` look
   like there?
2. Why does the VRAM manager need a buddy allocator when the system-memory
   manager needs essentially nothing?
3. A buffer will be mmap'd by the CPU *and* wants to live in VRAM. What
   constraint must its `ttm_place` carry, and what might it run out of?

<details>
<summary>answers</summary>

1. A single entry: `XE_PL_TT`. There is no VRAM memory type registered at all.
2. Because a VRAM resource must name a physical location in a fixed-size
   region, with contiguity and alignment constraints. A TT resource names no
   location — the pages live wherever the kernel put them, and the GPU reaches
   them through page tables.
3. It must be restricted to the CPU-visible (BAR) range. It can run out of
   *visible* VRAM even while plenty of total VRAM is free.

</details>

---

# Chapter 2 — Beat B: The Code

Paths are `drivers/gpu/drm/xe/` unless stated. Line numbers are **v7.3-rc1**.

## 1. The four memory types — `xe_bo.h:82`

```c
#define XE_PL_SYSTEM        TTM_PL_SYSTEM              /* :82  == 0 */
#define XE_PL_TT            TTM_PL_TT                  /* :83  == 1 */
#define XE_PL_VRAM0         TTM_PL_VRAM                /* :84  == 2 */
#define XE_PL_VRAM1         (XE_PL_VRAM0 + 1)          /* :85  == 3 */
#define XE_PL_STOLEN        (TTM_NUM_MEM_TYPES - 1)    /* :86  == 8 */
```

Three of these are just TTM's own numbering (`include/drm/ttm/ttm_placement.h:51`).
Two details worth noticing:

* **`XE_PL_VRAM1` is `VRAM0 + 1`.** Memory types are a flat `u32` namespace, so
  a second tile's VRAM simply takes the next slot. That's why `mem_type_is_vram()`
  at `xe_bo.c:90` is a *range* check:

  ```c
  bool mem_type_is_vram(u32 mem_type)
  {
  	return mem_type >= XE_PL_VRAM0 && mem_type != XE_PL_STOLEN;
  }
  ```

* **Stolen is parked at the very last slot** (`TTM_NUM_MEM_TYPES - 1`, i.e. 8),
  deliberately out of the way so VRAM can grow upward without colliding.

## 2. The flag vocabulary — `xe_bo.h:22`

The `u32` that drives everything:

```c
#define XE_BO_FLAG_USER                BIT(0)   /* created by userspace      */
#define XE_BO_FLAG_SYSTEM              BIT(1)   /* may live in XE_PL_TT      */
#define XE_BO_FLAG_VRAM0               BIT(2)
#define XE_BO_FLAG_VRAM1               BIT(3)
#define XE_BO_FLAG_VRAM_MASK           (XE_BO_FLAG_VRAM0 | XE_BO_FLAG_VRAM1)
#define XE_BO_FLAG_STOLEN              BIT(4)
#define XE_BO_FLAG_GGTT                BIT(5)
#define XE_BO_FLAG_PINNED              BIT(7)
#define XE_BO_FLAG_FORCE_WC            BIT(10)
#define XE_BO_FLAG_PAGETABLE           BIT(12)
#define XE_BO_FLAG_NEEDS_CPU_ACCESS    BIT(13)
#define XE_BO_FLAG_NEEDS_64K           BIT(15)
#define XE_BO_FLAG_NEEDS_2M            BIT(16)
#define XE_BO_FLAG_CPU_ADDR_MIRROR     BIT(24)
```

Notice the shape: bits 1–4 are **which warehouses are acceptable**; the rest are
**conditions**. That is the whole "wish list" idea, expressed as one integer.

And a convenience that keeps tile code generic (`xe_bo.h:30`):

```c
#define XE_BO_FLAG_VRAM(vram)          (XE_BO_FLAG_VRAM0 << ((vram)->id))
#define XE_BO_FLAG_VRAM_IF_DGFX(tile)  (IS_DGFX(...) ? XE_BO_FLAG_VRAM(...) \
                                                    : XE_BO_FLAG_SYSTEM)
```

`XE_BO_FLAG_VRAM_IF_DGFX(tile)` is everywhere in Xe. It means *"VRAM on a
discrete card, system memory on an integrated one"* — one expression that
writes both drivers.

## 3. The wish-list builder — `xe_bo.c:288`

The whole translation, in one small function:

```c
static int __xe_bo_placement_for_flags(struct xe_device *xe, struct xe_bo *bo,
				       u32 bo_flags, enum ttm_bo_type type)
{
	u32 c = 0;

	try_add_vram(xe, bo, bo_flags, type, &c);     /* :293 */
	try_add_system(xe, bo, bo_flags, &c);         /* :294 */
	try_add_stolen(xe, bo, bo_flags, &c);         /* :295 */

	if (!c)
		return -EINVAL;

	bo->placement = (struct ttm_placement) {
		.num_placement = c,
		.placement = bo->placements,
	};
	return 0;
}
```

**The call order *is* the preference order.** VRAM first, system second, stolen
last. That single ordering is Xe's memory policy in one line: always prefer
device memory, fall back to host memory.

The array it fills lives in the BO itself (`xe_bo_types.h`):

```c
struct ttm_place placements[XE_BO_MAX_PLACEMENTS];
struct ttm_placement placement;
```

### `try_add_system()` — `xe_bo.c:176`

```c
if (bo_flags & XE_BO_FLAG_SYSTEM) {
	bo->placements[*c] = (struct ttm_place) {
		.mem_type = XE_PL_TT,
		.flags = (bo_flags & XE_BO_FLAG_VRAM_MASK) ?
			 TTM_PL_FLAG_FALLBACK : 0,
	};
	*c += 1;
}
```

Two things in five lines:

1. `XE_BO_FLAG_SYSTEM` maps to **`XE_PL_TT`**, not `XE_PL_SYSTEM`. The story's
   distinction, made concrete: when you say "system memory is acceptable" you
   mean *GPU-reachable* system memory. `XE_PL_SYSTEM` is never something you
   ask for — TTM puts you there.
2. **`TTM_PL_FLAG_FALLBACK`** is only set if VRAM was *also* requested. It tells
   TTM "this entry is a consolation prize — don't pick it while the desired one
   is still achievable, even if that means evicting somebody." Without it, TTM
   would happily take the easy option and never use VRAM.

### `force_contiguous()` — `xe_bo.c:191`

```c
static bool force_contiguous(u32 bo_flags)
{
	if (bo_flags & XE_BO_FLAG_STOLEN)
		return true; /* users expect this */
	else if (bo_flags & XE_BO_FLAG_PINNED &&
		 !(bo_flags & XE_BO_FLAG_PINNED_LATE_RESTORE))
		return true; /* needs vmap */
	else if (bo_flags & XE_BO_FLAG_CPU_ADDR_MIRROR)
		return true;

	return bo_flags & XE_BO_FLAG_NEEDS_CPU_ACCESS &&
	       bo_flags & XE_BO_FLAG_PINNED;
}
```

Every branch is the same underlying reason: **something wants one flat CPU
pointer to the whole buffer.** `xe_bo_vmap()` can only produce that over a
contiguous range. Pinned buffers get vmap'd during suspend/resume; stolen
memory is mapped as one window; CPU-address-mirrored (SVM) buffers must line
up 1:1 with a CPU range.

Contiguity is expensive in a buddy allocator, so Xe asks for it only when a
CPU pointer genuinely requires it.

### `add_vram()` — `xe_bo.c:230`, and the window

This is the payoff of the story's section 4:

```c
static void add_vram(struct xe_device *xe, struct xe_bo *bo,
		     struct ttm_place *places, u32 bo_flags, u32 mem_type, u32 *c)
{
	struct ttm_place place = { .mem_type = mem_type };
	struct ttm_resource_manager *mgr = ttm_manager_type(&xe->ttm, mem_type);
	struct xe_ttm_vram_mgr *vram_mgr = to_xe_ttm_vram_mgr(mgr);
	struct xe_vram_region *vram = container_of(vram_mgr, struct xe_vram_region, ttm);
	u64 io_size = vram->io_size;

	if (force_contiguous(bo_flags))
		place.flags |= TTM_PL_FLAG_CONTIGUOUS;

	if (io_size < vram->usable_size) {            /* the BAR is smaller than VRAM */
		if (bo_flags & XE_BO_FLAG_NEEDS_CPU_ACCESS) {
			place.fpfn = 0;                       /* pin into the window  */
			place.lpfn = io_size >> PAGE_SHIFT;
		} else {
			place.flags |= TTM_PL_FLAG_TOPDOWN;   /* stay out of the way  */
		}
	}
	places[*c] = place;
	*c += 1;
}
```

Read the `if` out loud: *"only bother with window logic if the window is
actually smaller than the warehouse."* With resizable BAR working, `io_size ==
usable_size` and this whole block is skipped — no constraints at all.

When the window is small:
* CPU needs it → **`fpfn=0, lpfn=window`**: a hard address range restriction.
* CPU doesn't → **`TTM_PL_FLAG_TOPDOWN`**: allocate from the far end, keeping
  the scarce window free for buffers that need it.

That's the "stack it at the back of the warehouse" instruction, in two lines.

### One flag bit per tile — `xe_bo.c:86` and `:208`

```c
#define for_each_set_bo_vram_flag(bit__, bo_flags__) \
	for (unsigned int __bit_tmp = BIT(0); __bit_tmp <= XE_BO_FLAG_VRAM_MASK; __bit_tmp <<= 1) \
		for_each_if(((bit__) = __bit_tmp) & (bo_flags__) & XE_BO_FLAG_VRAM_MASK)
```

`try_add_vram()` loops this, so a BO flagged `VRAM0 | VRAM1` gets **two**
placement entries — "either tile's warehouse will do". `vram_bo_flag_to_tile_id()`
at `:208` turns the bit back into a tile index by bit arithmetic.

## 4. The three clerks, side by side

### The VRAM clerk — `xe_ttm_vram_mgr.c`

```c
static const struct ttm_resource_manager_func xe_ttm_vram_mgr_func = {  /* :272 */
	.alloc      = xe_ttm_vram_mgr_new,        /* :52  */
	.free       = xe_ttm_vram_mgr_del,        /* :175 */
	.intersects = xe_ttm_vram_mgr_intersects, /* :212 */
	.compatible = xe_ttm_vram_mgr_compatible, /* :242 */
	.debug      = xe_ttm_vram_mgr_debug,
};
```

State — `xe_ttm_vram_mgr_types.h`:

```c
struct xe_ttm_vram_mgr {
	struct ttm_resource_manager manager;
	struct gpu_buddy mm;        /* the buddy allocator          */
	u64 visible_size;           /* how big the BAR window is    */
	u64 visible_avail;          /* how much of it is still free */
	u64 default_page_size;
	struct mutex lock;
	u32 mem_type;
};
```

> **Rename alert:** in v7.3 the buddy allocator moved out of DRM into the core
> kernel. It is `struct gpu_buddy` in `include/linux/gpu_buddy.h` now. Older Xe
> material calls it `drm_buddy` — same allocator.

`visible_avail` is the second counter the story promised. Two independent
budgets, and either can run out.

Inside `xe_ttm_vram_mgr_new()` (`:52`), the placement flags become buddy flags:

```c
if (place->flags & TTM_PL_FLAG_TOPDOWN)
	vres->flags |= GPU_BUDDY_TOPDOWN_ALLOCATION;
if (place->flags & TTM_PL_FLAG_CONTIGUOUS)
	vres->flags |= GPU_BUDDY_CONTIGUOUS_ALLOCATION;
if (place->fpfn || lpfn != man->size >> PAGE_SHIFT)
	vres->flags |= GPU_BUDDY_RANGE_ALLOCATION;

err = gpu_buddy_alloc_blocks(mm, (u64)place->fpfn << PAGE_SHIFT,
			     (u64)lpfn << PAGE_SHIFT, size,
			     min_page_size, &vres->blocks, vres->flags);
```

One wish-list entry → one buddy call. That is the entire handoff.

Then window accounting (`:130`) walks the returned blocks and adds up how much
of the allocation landed below `visible_size`, subtracting it from
`visible_avail`.

And the punchline at `:145`:

```c
if (!(vres->base.placement & TTM_PL_FLAG_CONTIGUOUS) &&
    xe_is_vram_mgr_blocks_contiguous(mm, &vres->blocks))
	vres->base.placement |= TTM_PL_FLAG_CONTIGUOUS;   /* got lucky */

if (vres->base.placement & TTM_PL_FLAG_CONTIGUOUS)
	vres->base.start = gpu_buddy_block_offset(block) >> PAGE_SHIFT;
else
	vres->base.start = XE_BO_INVALID_OFFSET;
```

**`start` is only meaningful for a contiguous allocation.** Otherwise the
resource has no single address and `start` is poisoned with
`XE_BO_INVALID_OFFSET` — the real location is the block *list*. Xe even
opportunistically upgrades an allocation to "contiguous" if the buddy happened
to hand back adjacent blocks, so an io-mapping fast path can still be used.

### The system clerk — `xe_ttm_sys_mgr.c`

```c
int xe_ttm_sys_mgr_init(struct xe_device *xe)      /* :103 */
{
	struct ttm_resource_manager *man = &xe->mem.sys_mgr;
	struct sysinfo si;

	si_meminfo(&si);
	gtt_size = (u64)si.totalram * si.mem_unit;    /* budget == all of RAM */

	man->use_tt = true;
	man->func = &xe_ttm_sys_mgr_func;
	ttm_resource_manager_init(man, &xe->ttm, gtt_size >> PAGE_SHIFT);
	ttm_set_driver_manager(&xe->ttm, XE_PL_TT, man);
	ttm_resource_manager_set_used(man, true);
	...
}
```

And `alloc` (`:28`) is the accountant the story described:

```c
ttm_resource_init(tbo, place, &node->base.base);

if (!(place->flags & TTM_PL_FLAG_TEMPORARY) &&
    ttm_resource_manager_usage(man) > (man->size << PAGE_SHIFT)) {
	r = -ENOSPC;
	goto err_fini;
}

node->base.mm_nodes[0].start = 0;
node->base.mm_nodes[0].size  = PFN_UP(node->base.base.size);
node->base.base.start = XE_BO_INVALID_OFFSET;      /* :51 — no address */
```

Compare with the VRAM clerk: **no allocator, no lock, no address.** Just a
budget check and `XE_BO_INVALID_OFFSET`. Exactly the asymmetry from Beat A —
*"a TT resource says only how much."*

`man->use_tt = true` is the important flag: it tells TTM "resources in this
type are backed by a `ttm_tt` page array", which is where the real pages come
from. That's Chapter 3.

### The stolen clerk — `xe_ttm_stolen_mgr.c:235`

```c
ret = ttm_range_man_init_nocheck(&xe->ttm, XE_PL_STOLEN, false,
				 stolen_size >> PAGE_SHIFT);
```

One line. It uses TTM's **stock** range manager (`ttm/ttm_range_manager.c`,
a `drm_mm` interval allocator) — Xe writes no allocation code for stolen at
all. Note the detection ladder just above it at `:211`: SR-IOV VF gets zero,
discrete probes the LMEM BAR, integrated gen12.70+ has its own path, older
parts read the classic stolen-memory registers.

### And `XE_PL_SYSTEM`?

Xe never registers a manager for it. TTM does, itself:

```
xe_device.c:526    ttm_device_init(&xe->ttm, &xe_ttm_funcs, ...)
  -> ttm_device.c:230    ttm_sys_man_init(bdev)
       -> ttm_sys_manager.c:35   a stub manager: alloc always succeeds,
                                 size 0, no address, use_tt = true
```

`XE_PL_SYSTEM` is the state every BO starts in and can always be pushed back
to, which is exactly why its manager can never fail. The parking yard is
always big enough.

## 5. Where the wish list gets used

```
xe_gem_create_ioctl()            userspace names regions + flags
  -> xe_bo_create_user()
     -> __xe_bo_create_locked()          xe_bo.c:2522
        -> xe_bo_init_locked()           xe_bo.c:2321
           -> __xe_bo_placement_for_flags()   xe_bo.c:288   <-- built here
              -> ttm_bo_init_reserved()       TTM takes over
                 -> ttm_bo_validate() -> man->func->alloc()  <-- clerk runs
```

The wish list is built once at creation and stored in the BO. Every later
`xe_bo_validate()` re-runs TTM against that same stored `placement`.

## Try it yourself

```bash
# the entire policy, in one function
sed -n '176,320p' drivers/gpu/drm/xe/xe_bo.c

# three clerks, three sizes
wc -l drivers/gpu/drm/xe/xe_ttm_{vram,sys,stolen}_mgr.c

# who asks for VRAM-or-system generically
git grep -c 'XE_BO_FLAG_VRAM_IF_DGFX' drivers/gpu/drm/xe | sort -t: -k2 -rn | head -5
```

On hardware: `/sys/kernel/debug/dri/0/vram0_mm` and `gtt_mm` print each
manager's state, including `visible_avail`.

## Checkpoint

1. A BO is created with `XE_BO_FLAG_VRAM0 | XE_BO_FLAG_SYSTEM`. How many
   entries does its `ttm_placement` have, in what order, and which one carries
   `TTM_PL_FLAG_FALLBACK`?
2. Why is `ttm_resource.start` sometimes `XE_BO_INVALID_OFFSET` for a VRAM
   resource?
3. A system with resizable BAR enabled — which lines of `add_vram()` stop
   running, and why is that good?

<details>
<summary>answers</summary>

1. Two. `[0] = XE_PL_VRAM0`, `[1] = XE_PL_TT`, because `try_add_vram()` runs
   before `try_add_system()`. The `XE_PL_TT` entry carries `FALLBACK`, because
   VRAM was also requested.
2. Because the allocation is a *list* of buddy blocks with no single base
   address. `start` is only filled in when the resource is contiguous.
3. `io_size == usable_size`, so the whole `if (io_size < vram->usable_size)`
   block is skipped: no `fpfn/lpfn` restriction and no `TOPDOWN`. Good because
   every placement constraint is a constraint the buddy allocator might fail to
   satisfy — with a full-size BAR, allocation is strictly freer.

</details>
