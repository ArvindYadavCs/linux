# Chapter 3 — The Crate

> Beat A: the story.

Chapter 2 told us where memory *can* live, and who hands out space. But we
never actually created anything. There was a receipt system with nothing
filed in it.

This chapter is about the thing itself: the **buffer object**.

---

## 1. A buffer object is not its bytes

Start with the idea that trips people up, because everything else falls out of
it.

When userspace says "give me a 4MB buffer", it does **not** get 4MB of memory.
It gets a **crate**: an object with an identity, a size, a name, and a
lifetime. Whether that crate currently has anything physical inside it — and
where that physical thing sits — is a completely separate question, answered
later, and re-answered every time the buffer is used.

So a buffer in Xe is really **three separable things**:

| Thing | Question it answers | Lives in |
|-------|--------------------|-----------|
| the **object** | who am I, how big, who owns me | GEM |
| the **resource** | where am I right now | TTM (Chapter 2) |
| the **pages** | what physical memory backs me | TTM (`ttm_tt`) |

The object is permanent for the buffer's lifetime. The resource changes every
time the buffer is evicted or migrated. The pages may not exist at all.

Hold that apart in your head and TTM stops being confusing.

---

## 2. The nesting doll

Xe's buffer type is built by stacking three layers, each one physically
embedded inside the next:

```
    struct xe_bo                  <- Xe's crate
        struct ttm_buffer_object  <- TTM's warehouse record card
            struct drm_gem_object <- DRM's shipping label
```

Each layer adds exactly what it owns and nothing else.

### The shipping label — `drm_gem_object`

The bottom layer is pure identity:

* a **size**,
* a **refcount** — the buffer dies when it hits zero,
* a **handle** — the small integer userspace uses to name it in ioctls,
* a **`dma_resv`** — the clipboard, which we'll come to,
* a set of **function pointers** the driver fills in (free, mmap, export…).

Notice what's *not* here: no address, no memory type, nothing about where the
data is. GEM genuinely does not care. It issues the tracking number.

The handle is per-open-file, by the way. Two processes can hold handle `3`
and mean different buffers; the same buffer can be handle `3` in one process
and `7` in another.

### The warehouse record card — `ttm_buffer_object`

The middle layer adds the *placement* story from Chapter 2:

* the **current resource** — the slot receipt, or NULL if unplaced,
* the **page list** — if it's currently backed by system pages,
* the **placement wish list** we built last chapter,
* a **pin count**, an **LRU priority**, and a **type**.

That last one matters more than it looks. TTM buffers come in three types:

* **device** — created by userspace, mmap-able, evictable. The normal case.
* **kernel** — created by the driver for its own use: page tables, ring
  buffers, firmware images.
* **sg** — a buffer whose pages belong to somebody else entirely (an imported
  dma-buf from another device).

A great deal of Xe's buffer code branches on that enum, and once you know it
means *"who created this and who owns the pages"*, the branches read
naturally.

### The crate — `xe_bo`

The top layer is everything specific to this hardware: which VM the buffer
belongs to, its GGTT nodes, its `XE_BO_FLAG_*` bits, cache attributes,
compression state, and the links that put it on Xe's various lists.

The pattern to internalize:

> **`xe_bo` embeds `ttm_buffer_object` embeds `drm_gem_object`.**
> Going down is a field access. Going up is `container_of`.

Xe spells the trip up as `ttm_to_xe_bo()`, and down as `&bo->ttm.base`.

---

## 3. The packing list — where pages actually come from

Chapter 2 ended on a cliffhanger. The system-memory manager allocated *no
pages* — it just checked a budget and returned a receipt with no address. So
where do the actual pages come from?

From a second structure: the **page list**, TTM's `ttm_tt`.

A `ttm_tt` is exactly what its name suggests once you stop being scared of it:
an **array of `struct page *`**, plus the matching array of **DMA addresses**
the device will use to reach them, plus a caching mode.

Two properties make it click:

**It only exists when the buffer is backed by system memory.** A buffer living
in VRAM needs no page list — the VRAM resource *is* the storage, and it's
device memory, not kernel pages. This is precisely what the
`man->use_tt = true` flag meant on the system manager and *not* on the VRAM
manager.

**It can exist while being empty.** TTM distinguishes *allocated* (the array
exists) from *populated* (the array has actual pages in it). A buffer can hold
an unpopulated page list for a long time — that's a buffer that has been
swapped out, or one whose backing was deferred.

So the true picture of a system-resident buffer is:

```
   xe_bo
     ├── ttm_resource   "XE_PL_TT, 4MB"      <- the budget receipt
     └── ttm_tt         pages[1024]          <- the actual memory
                        dma_address[1024]    <- how the device reaches it
```

and of a VRAM-resident one:

```
   xe_bo
     └── ttm_resource   "XE_PL_VRAM0, buddy blocks [...]"   <- IS the storage
```

**Migration is now definable precisely:** moving a buffer from VRAM to system
memory means allocating and populating a page list, copying the bytes into it,
and swapping the resource. That's Chapter 4.

---

## 4. The pallet depot

One more layer, and it exists for a reason worth knowing.

TTM does not get pages straight from the kernel page allocator and hand them
back when done. It keeps a **pool**.

Why? Because of **cache attributes**. A page that the GPU will access
uncached, or write-combined, has had its CPU page-table attributes changed —
and on x86 that is genuinely expensive: it requires tearing down the kernel's
large-page mappings and flushing caches across every CPU.

So freeing such a page and immediately re-allocating it would burn that cost
twice for nothing. Instead TTM keeps freed pages in a pool **bucketed by
caching mode and allocation order**, and hands them straight back out to the
next requester that wants the same combination.

It's a pallet depot: pallets come back, get sorted by type, and go out again
without being rebuilt. The pool also registers with the kernel's shrinker, so
under memory pressure it gives the pages back to the system.

---

## 5. Cached, write-combined, uncached

Since caching decides which pool bucket you land in, it's worth knowing what
the three modes *mean*, because Xe picks between them on every buffer.

* **Cached (WB)** — normal memory. Fast for the CPU, but the GPU's view and
  the CPU's cache can disagree unless the hardware is coherent.
* **Write-combined (WC)** — CPU writes are batched and pushed out; CPU reads
  are *terrible* (uncached). Perfect for buffers the CPU only ever writes and
  the GPU then reads — which is most of what a graphics driver produces.
* **Uncached (UC)** — every access goes to memory. Slow, used for registers
  and a few hardware structures.

The rule of thumb that explains most of Xe's choices: **if the CPU will write
it and the GPU will read it, write-combined.** If the CPU will read it back,
you want cached, and you'd better be on coherent hardware.

There's a neat hardware fact underneath: on **discrete** cards, system memory
accessed by the GPU is always coherent with the CPU, so Xe can just use cached
everywhere and stop thinking. The caching gymnastics are mostly an
**integrated**-GPU problem.

---

## 6. The clipboard

Now the piece that ties memory management to scheduling — and the one thing
from this chapter you will meet in every later chapter.

Hanging on every crate is a **`dma_resv`**: a clipboard with two things on it.

**A lock.** Not an ordinary mutex — a `ww_mutex`, a *wound-wait* mutex, built
for the case where you must lock many objects at once and can't agree on an
order. (A submission touching 200 buffers has to lock all 200. Two threads
doing that in different orders would deadlock instantly. Wound-wait solves it
by making one thread back off and retry. That machinery is `drm_exec`, and it
gets its own chapter.)

**A list of fences, tagged by usage.** Four tags, and the distinction is the
interesting part:

* **KERNEL** — memory-management operations: the clear after allocation, the
  copy during a migration. *Everybody* must wait for these. You cannot read a
  buffer while the driver is still copying it.
* **WRITE** — somebody is writing the buffer. Readers must wait.
* **READ** — somebody is reading it. Writers must wait; other readers needn't.
* **BOOKKEEP** — "I'm using this, but manage synchronization yourself." The
  fence is recorded so the buffer can't be freed or moved out from under the
  user, but no implicit waiting is implied.

That's the whole contract between the two halves of this course:

> **Memory management may not move or free a buffer until the fences on its
> clipboard have signalled. Scheduling adds a fence to the clipboard for every
> job that touches the buffer.**

Neither side needs to know anything else about the other.

---

## 7. How a crate is born

Put it together. When userspace asks for a buffer:

1. **Validate and translate.** The ioctl checks the arguments and turns
   userspace's requested regions and flags into the `XE_BO_FLAG_*` word.
2. **Allocate the crate.** `xe_bo` is allocated and zeroed; the GEM object
   inside it is initialized with the size and the driver's function pointers.
3. **Build the wish list.** Chapter 2's `__xe_bo_placement_for_flags()` runs.
4. **Hand it to TTM.** TTM assigns the object a reservation object, locks it,
   and tries to satisfy the wish list — which runs a resource manager, which
   may evict somebody, and which may need pages from the pool.
5. **Wait for the clear.** New buffers must not leak another process's data,
   so the GPU zeroes them. That's an asynchronous job, and its fence goes on
   the clipboard under KERNEL.
6. **Issue the tracking number.** A handle is created in this file's handle
   table and returned to userspace.

Step 5 is worth dwelling on. The ioctl returns a handle to a buffer that is
*still being zeroed by the GPU*. That's not a bug — it's the design. The fence
on the clipboard guarantees that anything which later touches the buffer will
wait. Userspace gets its handle immediately and the work overlaps.

You'll see that pattern relentlessly in Xe: **return early, record a fence,
let the dependency system sort it out.**

---

## The picture

```
              struct xe_bo
  +--------------------------------------------------+
  |  flags, vm, tile, ggtt_node[], cpu_caching, ...   |   Xe layer
  |                                                   |
  |  struct ttm_buffer_object                         |
  |  +---------------------------------------------+  |
  |  |  type (device/kernel/sg), pin_count         |  |   TTM layer
  |  |  placement  ------> the wish list (ch 2)    |  |
  |  |  resource   ------> WHERE it is now         |  |
  |  |  ttm        ------> ttm_tt: the pages       |  |
  |  |                                             |  |
  |  |  struct drm_gem_object                      |  |
  |  |  +---------------------------------------+  |  |
  |  |  |  size, refcount, funcs                |  |  |   GEM layer
  |  |  |  resv ------> dma_resv: lock + fences |  |  |
  |  |  +---------------------------------------+  |  |
  |  +---------------------------------------------+  |
  +--------------------------------------------------+
```

## Three sentences to remember

1. **An object, a location, and some pages are three different things** —
   `drm_gem_object`, `ttm_resource`, `ttm_tt`.
2. **The page list exists only for system-backed buffers.** VRAM needs none;
   its resource *is* the storage.
3. **The `dma_resv` is the contract**: MM may not move what scheduling has
   fenced.

---

## Checkpoint

1. A buffer is created and immediately migrated VRAM → system memory. Which of
   the three pieces (object / resource / pages) changes, and which survives?
2. Why does TTM pool freed pages instead of returning them to the kernel?
3. `xe_gem_create_ioctl()` returns while the GPU is still zeroing the buffer.
   What stops a later job from reading uninitialized data?

<details>
<summary>answers</summary>

1. The object survives unchanged — same handle, same refcount, same clipboard.
   The resource is replaced (VRAM buddy blocks freed, a TT receipt issued). The
   page list is newly created and populated; before the move there wasn't one.
2. Because pages handed to the GPU often have non-default cache attributes, and
   changing those attributes on x86 is expensive. The pool recycles pages
   bucketed by caching mode and order so the cost isn't paid twice.
3. The clear is an asynchronous GPU job whose fence is added to the buffer's
   `dma_resv` under `DMA_RESV_USAGE_KERNEL`. Every later user of the buffer
   waits on the fences already on the clipboard.

</details>

---

# Chapter 3 — Beat B: The Code

Paths are `drivers/gpu/drm/xe/` unless stated. Line numbers are **v7.3-rc1**.

## 1. The nesting doll, annotated

`xe_bo_types.h`, with each field tagged by which story it belongs to:

```c
struct xe_bo {
	struct ttm_buffer_object ttm;      /* the two layers below, embedded   */

	/* --- identity / ownership --- */
	struct xe_vm *vm;                  /* NULL for "external" objects      */
	struct xe_tile *tile;              /* kernel BOs only                  */
	u32 flags;                         /* the XE_BO_FLAG_* word            */
	struct dma_buf *dma_buf;           /* if imported                      */

	/* --- placement (chapter 2) --- */
	struct ttm_place placements[XE_BO_MAX_PLACEMENTS];
	struct ttm_placement placement;

	/* --- addressing (chapter 5) --- */
	struct xe_ggtt_node *ggtt_node[XE_MAX_TILES_PER_DEVICE];

	/* --- CPU mapping --- */
	struct iosys_map vmap;
	struct ttm_bo_kmap_obj kmap;
	u16 cpu_caching;

	/* --- list membership --- */
	struct list_head pinned_link;
	struct list_head vram_userfault_link;
	struct llist_node freed;           /* deferred free list               */
};
```

And one level down (`include/drm/ttm/ttm_bo.h`):

```c
struct ttm_buffer_object {
	struct drm_gem_object base;        /* <- the bottom layer              */
	struct ttm_device *bdev;
	enum ttm_bo_type type;             /* device / kernel / sg             */
	struct kref kref;
	struct ttm_resource *resource;     /* WHERE  (may be NULL)             */
	struct ttm_tt *ttm;                /* PAGES  (may be NULL)             */
	struct ttm_lru_bulk_move *bulk_move;
	unsigned pin_count;
	struct sg_table *sg;
};
```

`resource` and `ttm` both being nullable pointers is the "object is not its
bytes" idea, spelled in C.

The conversions (`xe_bo.h`):

```c
ttm_to_xe_bo(tbo)    /* container_of upward from TTM  */
gem_to_xe_bo(obj)    /* container_of upward from GEM  */
&bo->ttm.base        /* downward: xe_bo -> gem object */
xe_bo_device(bo)     /* -> struct xe_device *         */
```

## 2. The GEM vtable — `xe_bo.c:2256`

```c
static const struct drm_gem_object_funcs xe_gem_object_funcs = {
	.free   = xe_gem_object_free,      /* :1882 */
	.close  = xe_gem_object_close,
	.mmap   = xe_gem_object_mmap,
	.export = xe_gem_prime_export,     /* dma-buf export */
	.vm_ops = &xe_gem_vm_ops,          /* CPU page faults */
};
```

Five entries. That's the entire surface GEM needs from a driver: how to free
it, what happens when a handle closes, how to mmap it, how to export it, and
what to do on a CPU page fault into its mapping.

## 3. The uAPI — `include/uapi/drm/xe_drm.h`

```c
struct drm_xe_gem_create {
	__u64 extensions;
	__u64 size;
	__u32 placement;                 /* the memory-region bitmask */
#define DRM_XE_GEM_CREATE_FLAG_DEFER_BACKING        (1 << 0)
#define DRM_XE_GEM_CREATE_FLAG_SCANOUT              (1 << 1)
#define DRM_XE_GEM_CREATE_FLAG_NEEDS_VISIBLE_VRAM   (1 << 2)
#define DRM_XE_GEM_CREATE_FLAG_NO_COMPRESSION       (1 << 3)
	__u32 flags;
	__u32 vm_id;
	__u32 handle;                    /* OUT */
#define DRM_XE_GEM_CPU_CACHING_WB  1
#define DRM_XE_GEM_CPU_CACHING_WC  2
	__u16 cpu_caching;
	__u16 pad[3];
	__u64 reserved[2];
};
```

Four flags and a caching mode. Compare with i915's `GEM_CREATE_EXT` zoo — the
uAPI restraint is deliberate.

## 4. `xe_gem_create_ioctl()` — `xe_bo.c:3515`

### Argument validation

```c
if (XE_IOCTL_DBG(xe, (args->placement & ~xe->info.mem_region_mask) ||
		 !args->placement))
	return -EINVAL;
```

`mem_region_mask` — built during probe back in Chapter 1, reported to
userspace by `DEVICE_QUERY`, and here used to reject regions this device
doesn't have. The loop closes.

`XE_IOCTL_DBG` is worth knowing: it evaluates the condition, and *if debug is
enabled* also prints which check rejected the ioctl. When userspace gets a
mysterious `-EINVAL` from Xe, `drm.debug` plus this macro tells you the exact
line.

### Flags translation — the bit trick

```c
bo_flags |= args->placement << (ffs(XE_BO_FLAG_SYSTEM) - 1);
```

One line converts the uAPI region mask into driver flags. It works because
the two bit layouts were **deliberately designed one shift apart**:

```
  uAPI placement bit 0 (system) --> XE_BO_FLAG_SYSTEM = BIT(1)
  uAPI placement bit 1 (vram0)  --> XE_BO_FLAG_VRAM0  = BIT(2)
  uAPI placement bit 2 (vram1)  --> XE_BO_FLAG_VRAM1  = BIT(3)
```

Remember `mem_region_mask |= BIT(vram->id) << 1` in `xe_tile_init_noalloc()`?
Same alignment, from the other end.

### The caching rules

```c
if (XE_IOCTL_DBG(xe, bo_flags & XE_BO_FLAG_VRAM_MASK &&
		 args->cpu_caching != DRM_XE_GEM_CPU_CACHING_WC))
	return -EINVAL;

if (XE_IOCTL_DBG(xe, bo_flags & XE_BO_FLAG_FORCE_WC &&
		 args->cpu_caching == DRM_XE_GEM_CPU_CACHING_WB))
	return -EINVAL;
```

Beat A's rule of thumb, enforced as policy:
* **VRAM must be WC.** Device memory reached over the BAR cannot be sanely
  cached by the CPU.
* **Scanout must not be WB.** `DRM_XE_GEM_CREATE_FLAG_SCANOUT` sets
  `XE_BO_FLAG_FORCE_WC` at `:3557`, because the display engine reads the
  framebuffer without snooping CPU caches.

### `xe_validation_guard()` — the retry loop

```c
err = 0;
xe_validation_guard(&ctx, &xe->val, &exec,
		    (struct xe_val_flags) {.interruptible = true}, err) {
	if (vm) {
		err = xe_vm_drm_exec_lock(vm, &exec);
		drm_exec_retry_on_contention(&exec);
		if (err)
			break;
	}
	bo = xe_bo_create_user(xe, vm, args->size, args->cpu_caching,
			       bo_flags, &exec);
	drm_exec_retry_on_contention(&exec);
	if (IS_ERR(bo)) {
		err = PTR_ERR(bo);
		xe_validation_retry_on_oom(&ctx, &err);
		break;
	}
}
```

This is Xe-specific scaffolding you won't find in older DRM drivers, and it
looks strange until you see what it is: **a loop body that can be restarted.**

Creating a buffer can fail two *recoverable* ways:

1. **Lock contention.** The ww_mutex from Beat A wounded us — another thread
   locking the same set of objects won the race. `drm_exec_retry_on_contention()`
   unwinds every lock taken so far and jumps back to the top.
2. **Out of memory.** No space, and the eviction we tried wasn't enough.
   `xe_validation_retry_on_oom()` escalates to *exhaustive* eviction — taking a
   device-wide lock so one allocator can evict everything without racing
   others — and retries.

`xe_validation_guard` is `scoped_guard()` + `drm_exec_until_all_locked()`
(`xe_validation.h:188`), so the retry is a real `goto`-based loop and cleanup
happens automatically on scope exit.

**Read the braces as "try this; it may run several times."** Once that clicks,
this pattern appears all over Xe and stops being noise.

## 5. Down the create path

```
xe_gem_create_ioctl()                       :3515
  └─ xe_bo_create_user()                    :2808   adds XE_BO_FLAG_USER
     └─ __xe_bo_create_locked()             :2522   VM linkage, GGTT insert
        └─ xe_bo_init_locked()              :2321   the real work
           └─ ttm_bo_init_reserved()                TTM takes over
```

### `__xe_bo_create_locked()` — `:2522`

```c
bo = xe_bo_init_locked(xe, bo, tile,
		       vm ? xe_vm_resv(vm) : NULL,          /* shared resv!   */
		       vm && !xe_vm_in_fault_mode(vm) &&
		       flags & XE_BO_FLAG_USER ?
		       &vm->lru_bulk_move : NULL,            /* shared LRU move */
		       size, cpu_caching, type, flags, NULL, exec);
```

Two decisions in that call, both important later:

* **`xe_vm_resv(vm)`** — a BO created against a VM **shares the VM's
  reservation object** instead of having its own. That means locking the VM
  locks all of its buffers at once. It's the single biggest reason Xe's
  submission path is cheap: a VM with 5000 private buffers is one lock, not
  5000.
* **`&vm->lru_bulk_move`** — the BO joins the VM's bulk-LRU group, so TTM
  moves the whole VM's buffers up the LRU list in one operation rather than
  one at a time.

Then, if `XE_BO_FLAG_GGTT` is set, it inserts the BO into each requested
tile's GGTT right here at `:2560`. That's Chapter 5.

### `xe_bo_init_locked()` — `:2321`

The alignment logic first:

```c
if (flags & (XE_BO_FLAG_VRAM_MASK | XE_BO_FLAG_STOLEN) &&
    !(flags & XE_BO_FLAG_IGNORE_MIN_PAGE_SIZE) &&
    ((xe->info.vram_flags & XE_VRAM_FLAGS_NEED64K) ||
     (flags & (XE_BO_FLAG_NEEDS_64K | XE_BO_FLAG_NEEDS_2M | XE_BO_FLAG_NEEDS_1G)))) {
	if (flags & XE_BO_FLAG_NEEDS_1G)      align = SZ_1G;
	else if (flags & XE_BO_FLAG_NEEDS_2M) align = SZ_2M;
	else                                  align = SZ_64K;
	...
}
```

Some hardware **requires** 64K pages in VRAM (`XE_VRAM_FLAGS_NEED64K`); large
pages are an optimization elsewhere. Either way the alignment lands in
`tbo->page_alignment`, which Chapter 2's buddy allocator read as
`min_page_size`. Two chapters, one variable.

Then the wiring:

```c
bo->ttm.base.funcs = &xe_gem_object_funcs;
bo->ttm.priority = XE_BO_PRIORITY_NORMAL;
drm_gem_private_object_init(&xe->drm, &bo->ttm.base, size);   /* GEM layer up */

if (resv) {
	ctx.allow_res_evict = !(flags & XE_BO_FLAG_NO_RESV_EVICT);
	ctx.resv = resv;
}

if (!(flags & XE_BO_FLAG_FIXED_PLACEMENT))
	err = __xe_bo_placement_for_flags(xe, bo, bo->flags, type);   /* ch 2 */

err = ttm_bo_init_reserved(&xe->ttm, &bo->ttm, type,
			   placement, alignment, &ctx, NULL, resv,
			   xe_ttm_bo_destroy);
```

`ctx.resv` tells TTM *"I already hold this lock — don't try to take it, and
don't evict objects sharing it."* That's how shared-resv VMs stay deadlock-free
during eviction.

And note the placement override just above:

```c
placement = (type == ttm_bo_type_sg ||
	     bo->flags & XE_BO_FLAG_DEFER_BACKING) ? &sys_placement : &bo->placement;
```

`DEFER_BACKING` (the uAPI flag) and imported dma-bufs get a bare
`XE_PL_SYSTEM` placement — created in the parking yard, backed later. Beat A's
"a crate may be empty," as one ternary.

### The KERNEL fence wait — `:2457`

The end of the function, and the payoff of story §7:

```c
if (type == ttm_bo_type_kernel) {
	long timeout = dma_resv_wait_timeout(bo->ttm.base.resv,
					     DMA_RESV_USAGE_KERNEL,
					     ctx.interruptible,
					     MAX_SCHEDULE_TIMEOUT);
	...
}
```

Read the asymmetry carefully. **Kernel buffers block here; user buffers do
not.** The comment above it explains why: userspace objects already go through
a dependency system that will wait on the clear fence before anything reads
them, so blocking would be pure latency. Internal driver callers mostly write
to their buffer with the CPU immediately after creating it and were never
written to expect an in-flight async clear — so Xe pays the wait for them.

## 6. The page list — `xe_ttm_tt_create()` at `:471`

Xe subclasses `ttm_tt` (`:374`):

```c
struct xe_ttm_tt {
	struct ttm_tt ttm;
	struct sg_table sgt;
	struct sg_table *sg;
	bool purgeable;
};
```

### The CCS extra pages

```c
extra_pages = 0;
if (xe_bo_needs_ccs_pages(bo))
	extra_pages = DIV_ROUND_UP(xe_device_ccs_bytes(xe, xe_bo_size(bo)), PAGE_SIZE);
```

Flat CCS is hardware memory compression: alongside a compressed surface the
GPU keeps **compression metadata** in a separate region of VRAM. When such a
buffer is evicted to system memory, that metadata has to go somewhere — so the
page list is allocated *larger than the buffer*, with the tail holding the CCS
bytes. `xe_bo_needs_ccs_pages()` at `:3929` is the eligibility ladder
(Xe2 discrete handles it in hardware; needs flat CCS; device-type only; not
if compression was disabled).

### The caching decision tree

```c
enum ttm_caching caching = ttm_cached;

if (!IS_DGFX(xe)) {                       /* integrated only */
	switch (bo->cpu_caching) {
	case DRM_XE_GEM_CPU_CACHING_WC: caching = ttm_write_combined; break;
	default:                        caching = ttm_cached;         break;
	}

	if ((!bo->cpu_caching && bo->flags & XE_BO_FLAG_FORCE_WC) ||
	    (!xe->info.has_cached_pt && bo->flags & XE_BO_FLAG_PAGETABLE))
		caching = ttm_write_combined;
}

if (bo->flags & XE_BO_FLAG_NEEDS_UC)
	caching = ttm_uncached;
```

The whole `if (!IS_DGFX(xe))` guard is Beat A's hardware fact in code — its
comment says it outright: *"DGFX system memory is always WB / ttm_cached …
GPU system memory accesses are always coherent with the CPU."* On discrete,
there is no decision to make.

The `has_cached_pt` clause is a nice concrete example: on some generations the
GPU's own **page-table walker** isn't coherent with CPU caches, so page-table
buffers must be write-combined or the GPU would read stale PTEs. Chapter 7
will build those page tables; this is where their memory gets its cache mode.

### `xe_ttm_tt_populate()` — `:550`

```c
if ((tt->page_flags & TTM_TT_FLAG_EXTERNAL) &&
    !(tt->page_flags & TTM_TT_FLAG_EXTERNAL_MAPPABLE))
	return 0;                                  /* dma-buf: not ours   */

if (ttm_tt_is_backed_up(tt) && !xe_tt->purgeable)
	err = ttm_tt_restore(ttm_dev, tt, ctx);    /* swapped out: read back */
else
	err = ttm_pool_alloc(&ttm_dev->pool, tt, ctx);   /* the pallet depot */
```

Three cases, one `if`-ladder: *somebody else's pages*, *our pages that were
swapped out*, *fresh pages from the pool*. `ttm_pool_alloc()` is the Beat A
pool — `ttm/ttm_pool.c`, 1564 lines of order-and-caching-bucketed recycling.

`unpopulate()` at `:580` is the mirror: unmap the scatter-gather table, then
`ttm_pool_free()` — back to the depot, not to the kernel.

## 7. How a crate dies

```
last handle closed / last reference dropped
  └─ drm_gem_object_put()  -> refcount hits 0
     └─ xe_gem_object_free()              :1882
        └─ ttm_bo_fini()                          TTM teardown, may defer
           └─ xe_ttm_bo_destroy()         :1844   the driver's last word
```

`xe_ttm_bo_destroy()` reads like a checklist of everything this buffer joined:

```c
drm_gem_object_release(&bo->ttm.base);
xe_assert(xe, list_empty(&ttm_bo->base.gpuva.list));   /* no mappings left! */

for_each_tile(tile, xe, id)
	if (bo->ggtt_node[id])
		xe_ggtt_remove_bo(tile->mem.ggtt, bo);

if (bo->vm && xe_bo_is_user(bo))
	xe_vm_put(bo->vm);
...
kfree(bo);
```

That `xe_assert` on `gpuva.list` is the guardrail for the next chapters: a
buffer must not still be mapped into any VM when it dies. Chapter 6 is about
that list.

Note also `ttm_bo_fini()` may **defer** the free (`delayed_delete` in
`ttm_buffer_object`) if fences are still outstanding — you cannot free memory
the GPU is mid-way through writing. The clipboard again.

## Try it yourself

```bash
# the three layers, one after another
sed -n '/^struct xe_bo {/,/^};/p'              drivers/gpu/drm/xe/xe_bo_types.h
sed -n '/^struct ttm_buffer_object {/,/^};/p'  include/drm/ttm/ttm_bo.h
sed -n '/^struct ttm_tt {/,/^};/p'             include/drm/ttm/ttm_tt.h

# how pervasive the retry pattern is
git grep -c 'xe_validation_guard\|drm_exec_retry_on_contention' drivers/gpu/drm/xe | sort -t: -k2 -rn | head
```

## Checkpoint

1. Why does a BO created against a VM use `xe_vm_resv(vm)` rather than its own
   reservation object? What does that buy at submission time?
2. `xe_bo_init_locked()` blocks on `DMA_RESV_USAGE_KERNEL` fences for kernel
   BOs but not user BOs. Why is that not a correctness bug for user BOs?
3. A buffer with flat-CCS compression is evicted from VRAM to system memory.
   Why is its `ttm_tt` bigger than the buffer?
4. What are the two recoverable failures `xe_validation_guard()` retries, and
   how do they differ?

<details>
<summary>answers</summary>

1. All the VM's private buffers then share one lock. Submitting work that
   touches thousands of buffers takes one reservation lock instead of
   thousands, and `ctx.resv` tells TTM not to try re-taking it during eviction.
2. Because user BOs are only ever reached through the submission path, which
   already waits on the fences in each buffer's `dma_resv` before the job runs.
   Kernel BOs are typically written by the CPU immediately after creation, with
   no such dependency step.
3. The hardware keeps compression metadata separate from the surface. Evicting
   to system memory must preserve it, so `extra_pages` worth of CCS data is
   appended to the page list.
4. Lock contention (ww_mutex wound — unwind all locks and retry the whole
   transaction) and OOM (retry under device-wide exhaustive eviction). The
   first is a race against another thread; the second is genuine memory
   pressure.

</details>
