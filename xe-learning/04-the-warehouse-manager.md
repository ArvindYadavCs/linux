# Chapter 4 — The Warehouse Manager at Work

> Beat A: the story.

We have warehouses (Chapter 2) and crates (Chapter 3). Nothing has moved yet.

This chapter is the machinery of movement: how a buffer gets to where it needs
to be, who gets thrown out to make room, who physically carries the bytes, and
who has to be told afterwards.

It is the busiest chapter in Act I. Everything here is TTM asking Xe questions
and Xe answering.

---

## 1. Validate: make reality match the wish

A buffer is about to be used. Somebody — a submission, a CPU mapping, a bind —
needs it to actually be *somewhere usable*. So they **validate** it.

Validation is one question asked well:

> *Is this buffer's current location compatible with its wish list? If not,
> make it so.*

Three outcomes:

* **Already fine.** The buffer's resource satisfies one of the placement
  entries. Nothing happens. This is the overwhelmingly common case, and it's
  cheap — a comparison, no allocation.
* **Needs moving.** Ask a resource manager for space in an acceptable memory
  type, then move the bytes there.
* **No space anywhere.** Start evicting other buffers, then try again.

The crucial thing about validate is that it is **idempotent and repeated**.
Every submission validates every buffer it touches, every time. The fast path
being a single compatibility check is what makes that affordable.

---

## 2. The two-pass trick

Here's a subtlety that explains a flag we met in Chapter 2 and never fully
used.

Suppose a buffer's wish list is *"VRAM0 preferred, system memory as
fallback"*, and VRAM is full.

The naive approach walks the list, fails on VRAM, succeeds on system memory,
and you end up in system memory forever. VRAM would never be used once full —
which is to say, almost immediately, and then permanently.

So TTM walks the list **twice**:

* **First pass:** consider only *desired* entries — skip anything marked
  `FALLBACK`. If space isn't free, **evict somebody** to make it free.
* **Second pass:** now consider the fallback entries. Take the consolation
  prize.

That's what `TTM_PL_FLAG_FALLBACK` was for. It doesn't mean "lower priority";
it means **"don't even look at me until you have genuinely tried to evict for
the good option."**

The story version: the clerk doesn't say "warehouse full, use the annex." The
clerk says "warehouse full — who in there hasn't been touched lately? Move
*them* to the annex, and put the new crate in the good spot." Only if there is
genuinely nobody worth moving does the new crate go to the annex.

---

## 3. Eviction: who gets thrown out

To evict, TTM needs a victim. It walks the memory type's **LRU list** —
least-recently-used first — and for each candidate asks two questions:

**Can I lock it?** Eviction needs the victim's `dma_resv`. If another thread
holds it, skip (or wound it, if we're in a ww-mutex transaction).

**Is evicting it valuable?** This is a driver callback, and it's a veto. TTM
asks: *"would moving this buffer out actually help me satisfy the placement I
want?"* A buffer sitting outside the address range we need doesn't help, so
evicting it would be pointless work.

Xe adds a second, subtler veto — and it's worth understanding because it
prevents a genuine disaster:

> **Never evict a buffer belonging to a VM that is currently validating.**

Picture it. A submission is validating 500 buffers belonging to VM A. Buffer
#400 doesn't fit, so TTM goes looking for a victim — and picks buffer #7,
which this very submission already validated and still needs. The submission
would evict its own working set, one buffer at a time, forever making no
progress. Xe's veto breaks that loop.

Once a victim is chosen, its own placement is consulted through another driver
callback: *"where should this evicted buffer go?"* Xe's answer is a small
table:

* evicted from **VRAM or stolen** → go to **TT** (GPU-reachable system memory)
* evicted from **TT** → go to **SYSTEM** (the parking yard; this is swapout)

And two special answers:

* a buffer userspace marked **"don't need"** → don't move it, **throw the
  contents away**
* the device has been **unplugged** → same, purge everything; there is no
  hardware left to preserve data for

That last pair is the interesting design point: **a placement list with zero
entries means "discard the backing store entirely."** Eviction and destruction
are the same mechanism with a different wish list.

---

## 4. Who carries the crate

Now the physical move. And the answer is the thing people find surprising:

> **The GPU copies its own memory.**

Not `memcpy`. Not DMA engines in the chipset. Xe builds a small batch buffer of
copy commands, submits it to the **blitter engine** (the copy engine from
Chapter 1), and the GPU does the work.

Why? Because it's dramatically faster — the GPU has the bandwidth to VRAM, and
the CPU reaching into VRAM through the BAR window would be crawling. Also the
CPU may not even be able to *see* the VRAM in question (Chapter 2's window
problem).

This has a huge consequence:

> **A migration returns a fence, not a finished copy.**

The move callback submits the copy and returns immediately. The bytes are still
in flight. TTM attaches the copy's fence to the buffer's `dma_resv` under
`DMA_RESV_USAGE_KERNEL` — the clipboard from Chapter 3 — and lets everyone
carry on. Anybody who later touches the buffer waits on that fence.

That's Chapter 3's *"return early, record a fence"* pattern doing real work.
Eviction under memory pressure doesn't stall the world; it queues copies.

### Clear instead of copy

One optimization worth its own name. Before copying, Xe asks: *does the source
actually contain anything meaningful?*

Sometimes it doesn't — a brand-new buffer, or one whose pages were never
populated. There's nothing to preserve. In that case the GPU doesn't copy, it
just **clears** the destination — which it must do anyway, so another process's
data can't leak through recycled memory.

So the move callback really has three modes: *nothing to do* (just swap the
resource pointer), *clear the destination*, and *copy source to destination*.

---

## 5. You can't get there from here

Now a constraint that falls straight out of Chapter 2, and it's the most
satisfying "aha" in Act I.

Remember: `XE_PL_SYSTEM` is system memory the GPU **cannot reach**.
`XE_PL_TT` is system memory the GPU **can** reach.

And we just established that **the GPU does the copying.**

Therefore: the GPU cannot copy a buffer from VRAM to `XE_PL_SYSTEM`. It cannot
address the destination. Nor from `XE_PL_SYSTEM` to VRAM — it can't address the
source.

So those moves are **impossible in one step**. The driver's answer is to tell
TTM: *"I can't do this directly — bounce through `XE_PL_TT` first."* TTM
performs the move in two hops:

```
   VRAM  --[GPU copies]-->  XE_PL_TT  --[just relabel]-->  XE_PL_SYSTEM
```

The second hop moves no data at all. The pages are the same pages; they simply
stop being DMA-mapped for the device. It's a change of *status*, not of
location — exactly what Chapter 2 said the SYSTEM/TT distinction was.

The mechanism is a special return code meaning "multi-hop", plus a temporary
intermediate placement. When you see it in code, read it as: **"the copy engine
can't see one end of this move."**

---

## 6. Telling everyone the map is stale

The last piece, and the one that reaches out of memory management into the rest
of the driver.

A buffer's bytes just moved. Anybody holding a **GPU address** for it now holds
a lie. Page tables say "virtual address X maps to VRAM page Y"; the data is no
longer at VRAM page Y.

So *before* the move happens, the driver is given a chance to react. This is
the notify callback, and Xe uses it to do three things:

1. **Tear down CPU mappings.** Any kernel vmap of the buffer is now invalid.
2. **Walk every VM that has this buffer mapped** and mark those mappings as
   needing a rebind. A buffer can be mapped into many VMs at many addresses;
   all of them must be told.
3. **Wait, or arrange to wait, for GPU work using those mappings to finish.**
   You cannot pull the rug out from under a job that is mid-execution.

Point 2 is where two worlds meet. Every buffer keeps a list of the mappings
that reference it — the list whose emptiness Chapter 3's destructor asserted.
Migration walks that list and invalidates each one.

And there's a fork in the road here that previews a lot of later material:

* A VM in **normal mode** must be told to re-bind its mappings before its next
  submission. The driver marks them and the work happens later.
* A VM in **fault mode** doesn't need telling. Its mappings are established on
  demand by page faults, so an invalidated mapping simply faults again next
  time it's touched.

Same event, two completely different recovery strategies. Chapters 10 and 18.

---

## 7. The LRU, and why it's moved in bulk

Eviction picks victims by LRU order, so something must maintain that order.
Every resource sits on its memory type's LRU list, and using a buffer moves it
to the tail.

But consider a submission touching 5000 buffers in one VM. Moving 5000 entries
to the tail of a list, one at a time, under a lock, on every submission, is
absurd.

So TTM supports **bulk moves**: a group of resources that are guaranteed to
stay adjacent on the LRU and can be moved to the tail as a unit, in constant
time. Xe gives every VM one of these, and every user buffer created against
that VM joins it.

This is the same trick as the shared reservation object from Chapter 3, applied
to a different data structure: **make the VM the unit of accounting, not the
buffer.** It's why Xe's submission path is cheap.

There is also a small **priority** dimension — a handful of priority levels, so
some buffers are considered for eviction before others regardless of age.

---

## 8. Shrinking and purging

Two more pressure valves, both driven from outside the GPU.

**The shrinker.** Xe registers with the kernel's memory shrinker, so when the
*system* is short on memory — not the GPU — the kernel can ask Xe to give
pages back. Xe responds by swapping buffer contents out to backup storage and
freeing the pages, or by purging buffers whose contents nobody wants.

**Purgeable buffers.** Userspace can mark a buffer *"I don't need the contents
any more, but keep the object around"* — a cache it can regenerate, typically.
Under pressure, Xe simply throws the contents away rather than paying to
preserve them. This is what the `MADVISE` ioctl from Chapter 1's table is for.

The interaction with eviction is neat: a *don't-need* buffer being evicted
doesn't get copied anywhere. It gets purged. The cheapest possible migration is
the one where you discard the cargo.

---

## 9. The whole conversation

Step back and look at the shape of this chapter. TTM is a state machine that
does not know what a GPU is. Everything hardware-specific arrives through a
table of callbacks the driver fills in. Roughly:

| TTM asks | Xe answers |
|----------|-----------|
| create/populate/free a page list | pool allocation, caching, CCS pages (ch 3) |
| where should this evicted buffer go? | VRAM→TT, TT→SYSTEM, don't-need→purge |
| is evicting this one worthwhile? | not if its VM is mid-validation |
| move these bytes | submit a blit; here's a fence |
| the buffer is about to move | invalidate mappings, trigger rebind |
| how does the CPU map this memory? | BAR offsets for VRAM, pages for TT |
| the buffer is being destroyed | unmap from GGTT, drop VM reference |

**That table *is* the driver's memory management.** Everything else in Act I is
either building the inputs to it (Chapters 2–3) or dealing with its
consequences (Chapters 5–11).

---

## The picture

```
  somebody needs a buffer usable
            |
            v
    +-----------------+   compatible?  ---> yes ---> done (fast path)
    |  validate       |
    +-----------------+
            | no
            v
   pass 1: desired placements only, evict if needed
            |
            +--- no luck ---> pass 2: fallback placements accepted
            |
            v
   +---------------------+      +-------------------------------+
   | resource manager    |      | eviction: walk LRU, ask the   |
   | allocates space     |<-----| driver "worth it?", move the  |
   +---------------------+      | victim out (recursively)      |
            |                   +-------------------------------+
            v
   +--------------------------------------------+
   | notify: invalidate mappings, mark rebind   |
   +--------------------------------------------+
            |
            v
   +--------------------------------------------+
   | move: submit a GPU blit, get a fence       |
   |   VRAM <-> SYSTEM ? bounce through TT      |
   +--------------------------------------------+
            |
            v
   fence lands on the dma_resv; everyone else waits on it
```

## Three sentences to remember

1. **Validate means "make reality match the wish list"** — and the common case
   is that it already does.
2. **The GPU copies its own memory**, so migration is asynchronous and returns
   a fence — and VRAM↔SYSTEM needs a TT hop because the copy engine can't
   address `XE_PL_SYSTEM`.
3. **Moving a buffer invalidates every mapping of it**, and that is the hinge
   between memory management and everything else.

---

## Checkpoint

1. A buffer's wish list is "VRAM0 desired, TT fallback". VRAM0 is full but
   contains several idle buffers. What happens, and in what order?
2. Why can't the driver move a buffer directly from VRAM to `XE_PL_SYSTEM`?
3. Why does Xe refuse to evict buffers belonging to a VM that is currently
   validating?
4. Eviction under memory pressure doesn't block the caller until the bytes have
   moved. What makes that safe?

<details>
<summary>answers</summary>

1. Pass one considers only VRAM0. It's full, so TTM walks VRAM0's LRU, picks
   the least recently used evictable buffer, asks the driver where it should go
   (TT), moves it, and retries the allocation — repeating until there's room.
   The fallback TT entry is never reached. The new buffer lands in VRAM0.
2. Because the GPU's copy engine performs the move, and `XE_PL_SYSTEM` pages
   are by definition not DMA-mapped for the device — the GPU cannot address the
   destination. The move goes VRAM→TT (real copy) then TT→SYSTEM (unmap only).
3. Otherwise a submission validating a large working set could evict buffers it
   has already validated and still needs, livelocking against itself.
4. The copy is a GPU job whose fence is attached to the buffer's `dma_resv`
   under `DMA_RESV_USAGE_KERNEL`. Any later access — CPU or GPU — waits on the
   fences already on the clipboard, so nobody can observe the buffer mid-copy.

</details>

---

# Chapter 4 — Beat B: The Code

Paths are `drivers/gpu/drm/xe/` unless stated. Line numbers are **v7.3-rc1**.
Note two functions in this chapter share line 962 in different files — watch
the filename.

## 1. The conversation, as a table — `xe_bo.c:1828`

```c
const struct ttm_device_funcs xe_ttm_funcs = {
	.ttm_tt_create     = xe_ttm_tt_create,        /* ch 3 */
	.ttm_tt_populate   = xe_ttm_tt_populate,      /* ch 3 */
	.ttm_tt_unpopulate = xe_ttm_tt_unpopulate,    /* ch 3 */
	.ttm_tt_destroy    = xe_ttm_tt_destroy,       /* ch 3 */
	.evict_flags       = xe_evict_flags,          /* :315  "where should it go?" */
	.move              = xe_bo_move,              /* :962  "move these bytes"   */
	.io_mem_reserve    = xe_ttm_io_mem_reserve,   /* "how does the CPU map it?" */
	.io_mem_pfn        = xe_ttm_io_mem_pfn,
	.access_memory     = xe_ttm_access_memory,    /* ptrace / coredump reads    */
	.release_notify    = xe_ttm_bo_release_notify,
	.eviction_valuable = xe_bo_eviction_valuable, /* :1241 the veto             */
	.delete_mem_notify = xe_ttm_bo_delete_mem_notify,
	.swap_notify       = xe_ttm_bo_swap_notify,
};
```

Thirteen entries. **This is the entire interface between TTM and Xe's memory
management.** Beat A's table, verbatim.

## 2. The canned placements — `xe_bo.c:52`

Three pre-built wish lists Xe hands back from `evict_flags`:

```c
static const struct ttm_place sys_placement_flags = {
	.mem_type = XE_PL_SYSTEM, .flags = 0,
};
static struct ttm_placement sys_placement = {
	.num_placement = 1, .placement = &sys_placement_flags,
};

static struct ttm_placement purge_placement;            /* :64  <-- !!! */

static const struct ttm_place tt_placement_flags[] = {
	{ .mem_type = XE_PL_TT,     .flags = TTM_PL_FLAG_DESIRED  },
	{ .mem_type = XE_PL_SYSTEM, .flags = TTM_PL_FLAG_FALLBACK },
};
static struct ttm_placement tt_placement = {
	.num_placement = 2, .placement = tt_placement_flags,
};
```

Look at line 64. **`purge_placement` is a bare, zero-initialized
`ttm_placement`** — `num_placement == 0`. That is the whole implementation of
"throw the contents away." Beat A's claim that *a zero-entry wish list means
discard the backing store* is literally one uninitialized static variable.

And `tt_placement` shows the two-pass flags in their natural habitat: try TT
(desired, evict for it if necessary), settle for SYSTEM (fallback) only after
that fails.

## 3. `xe_bo_validate()` — `xe_bo.c:3302`

```c
int xe_bo_validate(struct xe_bo *bo, struct xe_vm *vm, bool allow_res_evict,
		   struct drm_exec *exec)
{
	struct ttm_operation_ctx ctx = {
		.interruptible = true,
		.no_wait_gpu = false,
		.gfp_retry_mayfail = true,
	};

	if (xe_bo_is_pinned(bo))
		return 0;                       /* pinned: already where it must be */

	if (vm) {
		lockdep_assert_held(&vm->lock);
		xe_vm_assert_held(vm);
		ctx.allow_res_evict = allow_res_evict;
		ctx.resv = xe_vm_resv(vm);      /* ch 3: the shared lock */
	}

	xe_vm_set_validating(vm, allow_res_evict);     /* <-- arms the veto */
	trace_xe_bo_validate(bo);
	ret = ttm_bo_validate(&bo->ttm, &bo->placement, &ctx);
	xe_vm_clear_validating(vm, allow_res_evict);

	return ret;
}
```

Small function, four things worth naming:

* **`xe_bo_is_pinned()` → return 0 immediately.** A pinned buffer cannot move,
  so validation is a no-op. Cheapest possible fast path.
* **`ctx.resv = xe_vm_resv(vm)`** — Chapter 3's shared reservation object again.
  It tells TTM "I hold this lock; don't evict things that share it."
* **`gfp_retry_mayfail = true`** — allocate without triggering the OOM killer.
  GPU memory pressure should fail gracefully and let the caller retry (that's
  `xe_validation_retry_on_oom` from Chapter 3), not shoot processes.
* **`xe_vm_set_validating()` / `clear_validating()`** — this arms and disarms
  the self-eviction veto. Remember the pair; §6 is where it fires.

## 4. TTM's loop — `ttm/ttm_bo.c:1074`

```c
int ttm_bo_validate(struct ttm_buffer_object *bo,
		    struct ttm_placement *placement,
		    struct ttm_operation_ctx *ctx)
{
	dma_resv_assert_held(bo->base.resv);

	if (!placement->num_placement)
		return ttm_bo_pipeline_gutting(bo);      /* purge_placement lands here */

	force_space = false;
	do {
		if (bo->resource &&
		    ttm_resource_compatible(bo->resource, placement, force_space))
			return 0;                            /* THE FAST PATH */

		if (bo->pin_count)
			return -EINVAL;

		ret = ttm_bo_alloc_resource(bo, placement, ctx, force_space, &res);
		force_space = !force_space;              /* <-- the two-pass flip */
		if (ret == -ENOSPC)
			continue;
		if (ret)
			return ret;

bounce:
		ret = ttm_bo_handle_move_mem(bo, res, false, ctx, &hop);
		if (ret == -EMULTIHOP) {
			ret = ttm_bo_bounce_temp_buffer(bo, ctx, &hop);
			if (!ret)
				goto bounce;                 /* <-- the second hop */
		}
		...
	} while (ret && force_space);
}
```

Every one of Beat A's claims is visible here:

* **`!placement->num_placement` → gutting.** The purge path, first thing in the
  function.
* **`ttm_resource_compatible()` → return 0.** The fast path — one comparison,
  no allocation. This is what runs for almost every buffer on almost every
  submission.
* **`force_space = !force_space`** is the two-pass mechanism. First iteration
  `false`, second `true`, and the `while (ret && force_space)` ends it.
* **`goto bounce`** is the multi-hop. `ttm_bo_bounce_temp_buffer()` at `:334`
  moves the BO to the temporary stop, then we re-enter the move for the final
  leg.

## 5. The line that makes two-pass work — `ttm/ttm_bo.c:962`

Inside `ttm_bo_alloc_resource()`, walking the placement array:

```c
for (i = 0; i < placement->num_placement; ++i) {
	const struct ttm_place *place = &placement->placement[i];

	man = ttm_manager_type(bdev, place->mem_type);
	if (!man || !ttm_resource_manager_used(man))
		continue;

	if (place->flags & (force_space ? TTM_PL_FLAG_DESIRED :
			    TTM_PL_FLAG_FALLBACK))
		continue;                                  /* <-- THE line */

	ret = ttm_bo_alloc_at_place(bo, place, force_space, res, &alloc_state);

	if (ret == -ENOSPC) {
		continue;                                  /* try next placement  */
	} else if (ret == -EBUSY) {
		ret = ttm_bo_evict_alloc(bdev, man, place, bo, ctx,
					 ticket, res, &alloc_state);   /* EVICT */
		...
	}
	return 0;
}
return -ENOSPC;
```

Read the ternary carefully — it's the crux of the whole chapter:

| pass | `force_space` | skips entries flagged | so it considers |
|------|---------------|----------------------|-----------------|
| 1 | `false` | `FALLBACK` | desired placements only |
| 2 | `true` | `DESIRED` | fallback placements only |

**Pass 1 refuses to look at the consolation prize. Pass 2 refuses to look at
the good option** (it already failed). That inverted skip is how "evict before
you settle" is implemented in one line.

Also note `man || ttm_resource_manager_used(man)` → `continue`. On an
integrated GPU there is no VRAM manager registered, so a VRAM placement entry
is silently skipped rather than failing. Chapter 2's `IS_DGFX` asymmetry,
handled for free.

### The eviction walk — `ttm/ttm_bo.c:720`

```c
static int ttm_bo_evict_alloc(...)
{
	state->in_evict = true;

	evict_walk.walk.arg.trylock_only = true;
	lret = ttm_lru_walk_for_evict(&evict_walk.walk, bdev, man, 1);
	...
	if (lret || !ticket)
		goto out;

	evict_walk.walk.arg.trylock_only = false;
retry:
	do {
		evict_walk.walk.arg.ticket = ticket;
		evict_walk.evicted = 0;
		lret = ttm_lru_walk_for_evict(&evict_walk.walk, bdev, man, 1);
	} while (!lret && evict_walk.evicted);
	...
}
```

**Two sweeps, and the first is `trylock_only`.** Sweep one only considers
victims whose lock is free right now — cheap, no risk of blocking. Only if that
finds nothing does sweep two arm the ww-mutex `ticket` and start contending
for locks properly.

That is Beat A's "can I lock it?" question, done twice with different
aggression. The `do { } while (evicted)` loop keeps evicting until one
allocation attempt succeeds or nothing is left to evict.

## 6. Xe's two answers

### "Where should this evicted buffer go?" — `xe_bo.c:315`

```c
static void xe_evict_flags(struct ttm_buffer_object *tbo,
			   struct ttm_placement *placement)
{
	bool device_unplugged = drm_dev_is_unplugged(&xe->drm);

	if (!xe_bo_is_xe_bo(tbo)) {                       /* not ours */
		if (tbo->type == ttm_bo_type_sg) {
			placement->num_placement = 0;             /* can't move it */
			return;
		}
		*placement = device_unplugged ? purge_placement : sys_placement;
		return;
	}

	bo = ttm_to_xe_bo(tbo);
	if (bo->flags & XE_BO_FLAG_CPU_ADDR_MIRROR) {
		*placement = sys_placement;                   /* SVM: ch 10 */
		return;
	}

	if (device_unplugged && !tbo->base.dma_buf) {
		*placement = purge_placement;                 /* nothing to save */
		return;
	}

	if (xe_bo_madv_is_dontneed(bo)) {
		*placement = sys_placement;   /* NOT purge_placement -- see below */
		return;
	}

	switch (tbo->resource->mem_type) {
	case XE_PL_VRAM0:
	case XE_PL_VRAM1:
	case XE_PL_STOLEN:
		*placement = tt_placement;    /* VRAM/stolen -> GPU-reachable RAM */
		break;
	case XE_PL_TT:
	default:
		*placement = sys_placement;   /* TT -> parking yard (swapout)     */
		break;
	}
}
```

Beat A's answer table, one-for-one. Two subtleties the code volunteers:

* **`num_placement = 0` for foreign sg BOs.** A scatter-gather buffer from
  another device has pages Xe doesn't own — it cannot be moved *or* purged, so
  the zero-entry list here means "refuse", not "discard". Same value, opposite
  meaning, distinguished by context.
* **The `dontneed` case deliberately does *not* use `purge_placement`.** The
  comment explains it: purging via TTM's gutting path would skip
  `xe_bo_move()`, and Xe wants its *own* purge procedure to run there. So it
  asks for `sys_placement` and purges at the top of the move callback instead.
  That's the `evict && dontneed` branch at `:985`.

### "Is evicting this worthwhile?" — `xe_bo.c:1241`

```c
static bool
xe_bo_eviction_valuable(struct ttm_buffer_object *bo, const struct ttm_place *place)
{
	struct drm_gpuvm_bo *vm_bo;

	if (!ttm_bo_eviction_valuable(bo, place))
		return false;                      /* TTM's own range check first */

	if (!xe_bo_is_xe_bo(bo))
		return true;

	drm_gem_for_each_gpuvm_bo(vm_bo, &bo->base) {
		if (xe_vm_is_validating(gpuvm_to_vm(vm_bo->vm)))
			return false;                  /* <-- the self-eviction veto */
	}

	return true;
}
```

**Beat A's livelock disaster, prevented in five lines.** `xe_vm_is_validating()`
reads the flag `xe_bo_validate()` set two sections ago. The loop walks every VM
this buffer is mapped into — because a buffer shared between VMs must not be
evicted if *any* of them is mid-validation.

`drm_gem_for_each_gpuvm_bo` iterates the per-buffer list of VM associations.
That list is Chapter 6's subject; here it's used purely as "who would be
hurt if I moved this?"

## 7. `xe_bo_move()` — `xe_bo.c:962`

~240 lines. It is a **decision ladder**, and the only way to read it is
top-down, because every rung assumes the ones above it did not fire.

### Rung 0: the state variables — `:1012`

```c
bool handle_system_ccs = (!IS_DGFX(xe) && xe_bo_needs_ccs_pages(bo) &&
			  ttm && ttm_tt_is_populated(ttm)) ? true : false;

tt_has_data = ttm && (ttm_tt_is_populated(ttm) || ttm_tt_is_swapped(ttm));

move_lacks_source = !old_mem || (handle_system_ccs ? (!bo->ccs_cleared) :
				 (!mem_type_is_vram(old_mem_type) && !tt_has_data));

needs_clear = (ttm && ttm->page_flags & TTM_TT_FLAG_ZERO_ALLOC) ||
	(!ttm && ttm_bo->type == ttm_bo_type_device);
```

Two booleans carry the whole function:

* **`move_lacks_source`** — "there is nothing meaningful to copy *from*." True
  for a fresh BO, or one whose page list was never populated.
* **`needs_clear`** — "the destination must be zeroed." True when TTM asked for
  zeroed pages, or when a *user-visible* BO has no page list (so we can't know
  what's in the destination and must not leak it).

`move_lacks_source && needs_clear` → **clear**. `!move_lacks_source` →
**copy**. `move_lacks_source && !needs_clear` → **nothing**. Beat A's three
modes, as two bits.

### The ladder

```c
:985   if (evict && xe_bo_madv_is_dontneed(bo))       -> purge, free dst, done
:995   if ((!old_mem && ttm) && !handle_system_ccs)   -> creation path: map sg,
                                                         ttm_bo_move_null()
:1004  if (ttm_bo->type == ttm_bo_type_sg)            -> dma-buf path
:1023  if (new_mem->mem_type == XE_PL_TT)             -> xe_tt_map_sg() first
:1029  if (move_lacks_source && !needs_clear)         -> ttm_bo_move_null()
:1031  if (CPU_ADDR_MIRROR && new == SYSTEM)          -> xe_svm_bo_evict()
:1045  if (old == SYSTEM && new == TT)                -> ttm_bo_move_null()
:1053  if (old == TT && new == TT)                    -> ttm_bo_move_null()
:1060  if (!move_lacks_source && !pinned)             -> xe_bo_move_notify()
:1066  if (old == TT && new == SYSTEM)                -> wait BOOKKEEP, then null
:1082  if (SYSTEM <-> VRAM with real data)            -> -EMULTIHOP
:1095  pick a migrate context
:1132  clear or copy -> fence
:1151  ttm_bo_move_accel_cleanup(fence)
:1183  out: wait KERNEL if landing in SYSTEM, unmap sg
```

**`ttm_bo_move_null()` appears five times.** Every one is a "move" that copies
zero bytes — it just swaps the resource pointer. Count them and you see how
often migration is pure bookkeeping:

* creating a BO into TT (`:995`)
* nothing worth copying (`:1029`)
* SYSTEM → TT (`:1045`) — the pages don't move; they become DMA-mapped
* TT → TT (`:1053`) — a failed multi-hop landing back where it was
* TT → SYSTEM (`:1066`) — the pages don't move; they stop being DMA-mapped

The two SYSTEM↔TT entries are Beat A's "change of status, not location",
appearing exactly where predicted.

### The multi-hop — `:1082`

```c
if (!move_lacks_source &&
    ((old_mem_type == XE_PL_SYSTEM && resource_is_vram(new_mem)) ||
     (mem_type_is_vram(old_mem_type) && new_mem->mem_type == XE_PL_SYSTEM))) {
	hop->fpfn = 0;
	hop->lpfn = 0;
	hop->mem_type = XE_PL_TT;
	hop->flags = TTM_PL_FLAG_TEMPORARY;
	ret = -EMULTIHOP;
	goto out;
}
```

The "you can't get there from here" rule, in nine lines. Note the guard:
**`!move_lacks_source`**. If there's nothing to copy, no copy engine is
involved, so no hop is needed — a data-less SYSTEM↔VRAM transition is fine in
one step. The hop exists *only* because the blitter can't address
`XE_PL_SYSTEM`.

`TTM_PL_FLAG_TEMPORARY` marks the intermediate resource so TTM knows it's a way
station. (And the `old == TT && new == TT` rung at `:1053` exists to handle a
multi-hop that failed partway and left the BO sitting at the way station.)

### Who does the copying — `:1095`

```c
if (bo->tile)
	migrate = bo->tile->migrate;                       /* kernel BO: its tile */
else if (resource_is_vram(new_mem))
	migrate = mem_type_to_migrate(xe, new_mem->mem_type);   /* dst tile */
else if (mem_type_is_vram(old_mem_type))
	migrate = mem_type_to_migrate(xe, old_mem_type);        /* src tile */
else
	migrate = xe->tiles[0].migrate;                         /* neither: tile 0 */
```

**Chapter 1's `tile->migrate` finally used.** The priority order is
*destination tile, else source tile, else tile 0* — copy from the side that owns
the VRAM, because that tile's blitter has local bandwidth to it.

And the copy itself, `:1132`:

```c
if (move_lacks_source) {
	u32 flags = 0;
	if (mem_type_is_vram(new_mem->mem_type))
		flags |= XE_MIGRATE_CLEAR_FLAG_FULL;
	else if (handle_system_ccs)
		flags |= XE_MIGRATE_CLEAR_FLAG_CCS_DATA;

	fence = xe_migrate_clear(migrate, bo, new_mem, flags);
} else {
	fence = xe_migrate_copy(migrate, bo, bo, old_mem, new_mem,
				handle_system_ccs);
}
```

Both return **`struct dma_fence *`**. That return type *is* Beat A's headline:
migration is asynchronous. `xe_migrate_clear()` is at `xe_migrate.c:1599`,
`xe_migrate_copy()` at `:1095` — Chapter 7 reads them, because the same
machinery writes page tables.

Then `ttm_bo_move_accel_cleanup(ttm_bo, fence, evict, true, new_mem)` at
`:1151` is TTM's "accelerated" (fenced) completion: it parks the old resource
until the fence signals, installs the new one, and adds the fence to the
`dma_resv` under KERNEL. If that fails, the fallback at `:1154` just
`dma_fence_wait()`s synchronously — correctness over pipelining.

### The exit — `:1183`

```c
out:
	if ((!ttm_bo->resource || ttm_bo->resource->mem_type == XE_PL_SYSTEM) &&
	    ttm_bo->ttm) {
		long timeout = dma_resv_wait_timeout(ttm_bo->base.resv,
						     DMA_RESV_USAGE_KERNEL, false,
						     MAX_SCHEDULE_TIMEOUT);
		...
		xe_tt_unmap_sg(xe, ttm_bo->ttm);
	}
```

Landing in `XE_PL_SYSTEM` is the one case that **must** block. The pages are
about to stop being DMA-mapped, so every outstanding GPU access to them has to
be finished first — you cannot unmap memory a copy engine is still reading.
Async everywhere else; synchronous here.

## 8. Telling the VMs — `xe_bo.c:805` and `:669`

```c
static int xe_bo_move_notify(struct xe_bo *bo, const struct ttm_operation_ctx *ctx)
{
	if (xe_bo_is_pinned(bo))
		return -EINVAL;                 /* pinned buffers do not move */

	xe_bo_vunmap(bo);                       /* 1. kill CPU mappings   */
	ret = xe_bo_trigger_rebind(xe, bo, ctx);/* 2. tell every VM       */
	if (ret)
		return ret;

	if (ttm_bo->base.dma_buf && !ttm_bo->base.import_attach)
		dma_buf_invalidate_mappings(ttm_bo->base.dma_buf);   /* 3. importers */

	if (mem_type_is_vram(old_mem_type)) {
		/* drop it off the VRAM CPU-fault list */
		list_del_init(&bo->vram_userfault_link);
	}
	return 0;
}
```

Beat A's three jobs, plus a fourth: other *devices* that imported this buffer
get told too.

`xe_bo_trigger_rebind()` at `:669` is where the fault-mode fork lives:

```c
drm_gem_for_each_gpuvm_bo(vm_bo, obj) {
	struct xe_vm *vm = gpuvm_to_vm(vm_bo->vm);

	if (!xe_vm_in_fault_mode(vm)) {
		drm_gpuvm_bo_evict(vm_bo, true);       /* mark: needs rebind later */
		if (!xe_device_is_l2_flush_optimized(xe))
			continue;                          /* <-- done for this VM   */
	}

	if (!idle) {
		timeout = dma_resv_wait_timeout(bo->ttm.base.resv,
						DMA_RESV_USAGE_BOOKKEEP,
						ctx->interruptible,
						MAX_SCHEDULE_TIMEOUT);
		...
		idle = true;
	}

	drm_gpuvm_bo_for_each_va(gpuva, vm_bo) {
		struct xe_vma *vma = gpuva_to_vma(gpuva);
		ret = xe_vm_invalidate_vma(vma);        /* zap the PTEs now */
	}
}
```

Read the `continue`. **A normal-mode VM is merely *marked* and we move on** —
`drm_gpuvm_bo_evict()` puts its mappings on an evicted list, and the rebind
happens before that VM's next submission (Chapter 18's rebind worker). No
waiting, no PTE writes, right here in the eviction path.

A **fault-mode** VM falls through: wait for outstanding GPU work
(`BOOKKEEP`), then walk every mapping and invalidate its page-table entries
immediately. Its mappings will be re-established by page faults on demand
(Chapter 10).

Same event, two strategies — exactly as Beat A promised, and the `continue` is
the branch point.

## 9. The rest of the callbacks

**`swap_notify`** — TTM is about to swap this buffer out to backup storage:

```c
if (xe_tt->purgeable)
	xe_ttm_bo_purge(ttm_bo, &ctx);
```

If userspace said *don't need*, don't waste I/O writing it out. Discard it.
Beat A's "cheapest migration discards the cargo."

**`delete_mem_notify`** — the resource is going away; detach VF CCS state, and
for imported dma-bufs `dma_buf_unmap_attachment()` and drop the sg table.

**`io_mem_reserve`** — "how does the CPU reach this?" For SYSTEM and TT,
nothing to do (pages). For VRAM, check visibility first, then compute BAR
offsets:

```c
if (!xe_ttm_resource_visible(xe, mem))
	return -EINVAL;                       /* outside the window: refuse */
mem->bus.offset  = mem->start << PAGE_SHIFT;
mem->bus.offset += vram->io_start;
mem->bus.is_iomem = true;
```

Chapter 2's window problem, enforced at CPU-mapping time. A buffer allocated
top-down (deliberately outside the window) **cannot** be CPU-mapped, and this
is where that fails.

**`release_notify`** — a subtle one, and a preview:

```c
dma_resv_for_each_fence(&cursor, &ttm_bo->base._resv, DMA_RESV_USAGE_BOOKKEEP, fence) {
	if (xe_fence_is_xe_preempt(fence) && !dma_fence_is_signaled(fence)) {
		if (!replacement)
			replacement = dma_fence_get_stub();
		dma_resv_replace_fences(&ttm_bo->base._resv, fence->context,
					replacement, DMA_RESV_USAGE_BOOKKEEP);
	}
}
```

A buffer being destroyed may still carry unsignalled **preempt fences**. Those
only signal when the VM's exec queues are preempted — and if the VM is going
away too, that may never happen, so TTM's teardown would wait forever.
Replacing them with an already-signalled stub breaks the cycle. Chapter 18
explains what a preempt fence is; file this as "destruction must not wait on a
promise nobody will keep."

## 10. Eviction on demand — `xe_bo.c:3901`

```c
int xe_bo_evict(struct xe_bo *bo, struct drm_exec *exec)
{
	struct ttm_placement placement;

	xe_evict_flags(&bo->ttm, &placement);              /* ask ourselves! */
	ret = ttm_bo_validate(&bo->ttm, &placement, &ctx);
	if (ret)
		return ret;

	dma_resv_wait_timeout(bo->ttm.base.resv, DMA_RESV_USAGE_KERNEL,
			      false, MAX_SCHEDULE_TIMEOUT);
	return 0;
}
```

Elegant: to evict a buffer deliberately, Xe calls **its own `evict_flags`
callback** to produce the target placement, then validates against it. The
same code path TTM uses under pressure, driven manually.

Note the trailing wait: unlike TTM's internal eviction, this one blocks until
the copy has actually completed. Its callers (suspend, device removal) need the
data to be *there*, not merely promised. `xe_bo_evict_all()` in
`xe_bo_evict.c:160` is the suspend-time sweep that uses it.

## Try it yourself

```bash
# the 13 questions, then the hardest function in Act I
sed -n '1828,1842p' drivers/gpu/drm/xe/xe_bo.c
sed -n '962,1200p'  drivers/gpu/drm/xe/xe_bo.c

# every zero-byte "move"
grep -n 'ttm_bo_move_null' drivers/gpu/drm/xe/xe_bo.c

# the two-pass line, in context
sed -n '962,1030p' drivers/gpu/drm/ttm/ttm_bo.c
```

With tracing on, the migration decisions are visible directly:

```bash
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_bo_move/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_evict/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

`trace_xe_bo_move` prints source type, destination type, and
`move_lacks_source` — the three things that pick the rung.

## Checkpoint

1. In `ttm_bo_alloc_resource()`, what exactly does
   `place->flags & (force_space ? TTM_PL_FLAG_DESIRED : TTM_PL_FLAG_FALLBACK)`
   accomplish, and what would break if the ternary were inverted?
2. `purge_placement` is declared with no initializer. Why is that sufficient,
   and why does the `dontneed` case in `xe_evict_flags()` refuse to use it?
3. `ttm_bo_move_null()` is called five times in `xe_bo_move()`. What do the
   SYSTEM→TT and TT→SYSTEM cases have in common, and why is one of them
   nonetheless preceded by a blocking wait?
4. Why does `ttm_bo_evict_alloc()` sweep the LRU twice, with `trylock_only`
   true the first time?
5. A normal-mode VM and a fault-mode VM both have a buffer that is being
   evicted. Trace what happens to each one's page tables.

<details>
<summary>answers</summary>

1. It inverts which placement entries are skipped per pass: pass 1 (`force_space
   == false`) skips `FALLBACK` entries, pass 2 skips `DESIRED` ones. Inverting
   it would make TTM accept the fallback placement before ever trying to evict
   for the desired one — VRAM would stop being used the moment it first filled.
2. `num_placement` is 0 in a zero-initialized `ttm_placement`, and
   `ttm_bo_validate()` treats a zero-entry list as "gut the backing store". The
   `dontneed` case avoids it because gutting happens inside TTM and would skip
   `xe_bo_move()` entirely, where Xe's own purge procedure lives; so it asks for
   `sys_placement` and purges at the top of the move callback instead.
3. Both move zero bytes — the pages are identical; only their DMA-mapped status
   changes. TT→SYSTEM must wait first because the pages are about to be
   unmapped from the device, and any in-flight GPU access to them has to finish
   before that is safe.
4. The first sweep is cheap and non-blocking: it only considers victims whose
   reservation lock is immediately free. Only if that finds no usable victim
   does it arm the ww-mutex ticket and start genuinely contending for locks,
   which can wound other threads and force them to retry.
5. Normal mode: `drm_gpuvm_bo_evict(vm_bo, true)` marks the VM's mappings as
   needing rebind and the eviction path moves on — no PTE writes now; the
   rebind worker fixes them before the VM's next submission. Fault mode: the
   code waits for outstanding GPU work on the buffer, then calls
   `xe_vm_invalidate_vma()` on every mapping, zapping the PTEs immediately; the
   mappings come back later via page faults.

</details>
