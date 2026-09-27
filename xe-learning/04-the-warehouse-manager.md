# Chapter 4 — Moving Things Around

> Beat A: the story. (Beat B, the code, follows.)

We have warehouses (Chapter 2) and crates (Chapter 3). Nothing has moved yet.
This chapter is about movement.

Only four ideas. Take them one at a time.

---

## Idea 1: "Validate" means *put it where it needs to be*

Somebody is about to use a buffer. Before they can, the buffer must actually be
somewhere usable. The act of making that true is called **validate**.

Validate asks one question:

> *Is this buffer already in a place its wish list allows?*

* **Yes** → do nothing. Done. (This is what happens almost every time.)
* **No** → find space somewhere on the wish list, and move the bytes there.

That's it. Validate is called constantly — every submission validates every
buffer it touches — so the "yes, already fine" path is deliberately just one
comparison.

---

## Idea 2: To make room, kick out the oldest crate

What if there is no space?

The clerk does **not** immediately give up and use the fallback warehouse.
Instead it looks inside the *good* warehouse and asks: *who in here hasn't been
used lately?* That crate gets moved out, and the new one takes its place.

This is **eviction**. Victims are chosen oldest-first (an LRU list).

Only if there is genuinely nobody worth evicting does the new crate settle for
the fallback warehouse.

So placement really happens in two rounds:

| Round | What it will accept |
|-------|--------------------|
| 1 | only the **preferred** spot — evicting somebody if needed |
| 2 | the **fallback** spot |

Without round 1, VRAM would stop being used the first time it filled up — and
then forever. That's why the order matters.

**One special rule from Xe:** never evict a buffer belonging to a VM that is
*currently* validating. Otherwise a submission validating 500 buffers could
evict buffers it already validated and still needs — forever making no
progress.

---

## Idea 3: The GPU copies its own memory

Now the actual move. Who carries the bytes?

**The GPU does.** Not `memcpy`. Xe builds a tiny program of copy commands and
submits it to the **blitter** (the copy engine from Chapter 1).

Why? The GPU has huge bandwidth to VRAM. The CPU reaching into VRAM through
that small BAR window would crawl — and might not even be able to see the
memory at all.

This has one big consequence:

> **A move returns a fence, not finished bytes.**

The copy is submitted and the function returns immediately. The bytes are still
in flight. The copy's fence goes onto the buffer's clipboard (the `dma_resv`
from Chapter 3), and anybody who touches the buffer later waits on it.

So eviction under memory pressure doesn't freeze the machine. It queues copies.

### And a nice consequence of that

Remember from Chapter 2:

* `XE_PL_SYSTEM` = system RAM the GPU **cannot** reach
* `XE_PL_TT` = system RAM the GPU **can** reach

If the GPU does the copying, then the GPU **cannot** copy VRAM → `XE_PL_SYSTEM`.
It cannot address the destination!

So that move happens in two steps:

```
  VRAM  --[GPU copies]-->  XE_PL_TT  --[just relabel, 0 bytes]-->  XE_PL_SYSTEM
```

The second step moves no data. Same pages — they just stop being DMA-mapped for
the device. Exactly what Chapter 2 said the SYSTEM/TT difference was.

---

## Idea 4: Moving a buffer breaks every map of it

A buffer's bytes just moved. Anybody holding a **GPU address** for it now holds
a wrong address. Page tables still say "virtual address X lives at VRAM page
Y", but the data left.

So *before* the move, the driver walks every VM that has this buffer mapped and
deals with it. Two different strategies:

* **Normal VM** → just *mark* the mappings as stale. They get rebuilt before
  that VM's next submission.
* **Fault-mode VM** → wipe the page table entries right now. They'll be rebuilt
  automatically by page faults when next touched.

This is the hinge between memory management and the rest of the driver, and we
come back to it in Chapters 10 and 18.

---

## Two more things, briefly

**Bulk LRU.** A submission may touch 5000 buffers. Moving 5000 LRU entries one
by one would be absurd, so buffers belonging to one VM are grouped and moved as
a unit. Same trick as the shared lock in Chapter 3: *make the VM the unit, not
the buffer.*

**Purging.** Userspace can say "I don't need these contents any more, just keep
the object". Under pressure, Xe throws the contents away instead of paying to
copy them. The cheapest move is the one where you discard the cargo.

---

## The whole chapter in one picture

```
   somebody needs a buffer usable
              |
              v
     already in an allowed place?  --- yes ---> DONE
              | no
              v
     round 1: try the preferred spot
              |
        no room?  ---> evict the oldest crate there, then retry
              |
        still nothing?  ---> round 2: accept the fallback spot
              |
              v
     tell every VM "your map is stale"
              |
              v
     submit a GPU copy  --->  get a fence  --->  return immediately
              |
              v
     fence goes on the clipboard; everyone else waits on it
```

## Remember these three

1. **Validate = make reality match the wish list.** Usually it already does.
2. **The GPU copies its own memory**, so moves are asynchronous and return a
   fence — and VRAM↔SYSTEM needs a stop at TT along the way.
3. **Moving a buffer invalidates every mapping of it.**

---
---

# Chapter 4 — Beat B: The Code

We follow **one single journey**, start to finish:

> *A buffer needs to be in VRAM. VRAM is full. Something has to move.*

Eight steps, in order. Each code block is commented line by line.
(Comments marked `//` are mine, explaining the code. Comments with `/* */` are
the kernel's own.)

Files: `drivers/gpu/drm/xe/` for Xe, `drivers/gpu/drm/ttm/` for TTM.
Line numbers are v7.3-rc1.

---

## Step 1 — Somebody asks: `xe_bo_validate()`

**File: `xe_bo.c:3302`**

This is the front door. Everybody who needs a buffer usable calls this.

```c
int xe_bo_validate(struct xe_bo *bo, struct xe_vm *vm, bool allow_res_evict,
		   struct drm_exec *exec)
{
	struct ttm_operation_ctx ctx = {
		.interruptible = true,      // a stuck wait can be Ctrl-C'd
		.no_wait_gpu = false,       // we're allowed to wait for the GPU
		.gfp_retry_mayfail = true,  // if RAM runs out, FAIL - don't invoke
					    // the OOM killer to shoot processes
	};
	int ret;

	if (xe_bo_is_pinned(bo))
		return 0;                   // pinned = cannot move = nothing to do.
					    // Fastest possible exit.

	if (vm) {
		ctx.resv = xe_vm_resv(vm);  // "I already hold this VM's lock.
					    //  Don't try to take it again, and
					    //  don't evict buffers sharing it."
					    // (Chapter 3's shared lock)
	}

	xe_vm_set_validating(vm, allow_res_evict);   // raise a flag: "this VM is
						     //  validating right now"
						     // Step 4 reads this flag.

	ret = ttm_bo_validate(&bo->ttm, &bo->placement, &ctx);
	//                               ^^^^^^^^^^^^^^
	//    The wish list we built in Chapter 2. Hand it to TTM and let TTM
	//    do the work. Everything from here to Step 5 happens inside TTM.

	xe_vm_clear_validating(vm, allow_res_evict); // lower the flag

	return ret;
}
```

**Takeaway:** Xe's own part is tiny. It sets up options, raises a flag, and
hands the wish list to TTM.

---

## Step 2 — TTM checks the easy case

**File: `ttm/ttm_bo.c:1074`** — we are now inside TTM.

```c
int ttm_bo_validate(struct ttm_buffer_object *bo,
		    struct ttm_placement *placement,     // the wish list
		    struct ttm_operation_ctx *ctx)
{
	if (!placement->num_placement)               // an EMPTY wish list means
		return ttm_bo_pipeline_gutting(bo);  // "throw the contents away"
						     // (used for purging)

	force_space = false;                         // start with ROUND 1
	do {
		if (bo->resource &&                          // do we have a spot, and
		    ttm_resource_compatible(bo->resource,    // is it one the wish
					    placement,       // list allows?
					    force_space))
			return 0;                            // YES -> done, no work.
							     // <<< THE COMMON CASE
```

That `return 0` is the whole point of Idea 1. One comparison, no allocation.

Our buffer isn't in VRAM though, so we keep going:

```c
		if (bo->pin_count)
			return -EINVAL;              // pinned buffers must never move

		// Ask for space. This is where rounds 1 and 2 happen (Step 3).
		ret = ttm_bo_alloc_resource(bo, placement, ctx, force_space, &res);

		force_space = !force_space;          // flip false->true, so the NEXT
						     // loop iteration is ROUND 2

		if (ret == -ENOSPC)
			continue;                    // round 1 found nothing ->
						     // loop again, now as round 2
		if (ret)
			return ret;                  // a real error

bounce:
		// We have space. Now actually move the bytes (Steps 5-7).
		ret = ttm_bo_handle_move_mem(bo, res, false, ctx, &hop);

		if (ret == -EMULTIHOP) {             // the driver said "I can't do
						     //  this in one step"
			ret = ttm_bo_bounce_temp_buffer(bo, ctx, &hop);
			if (!ret)
				goto bounce;         // move to the halfway stop,
						     // then try the final leg again
		}
		...
	} while (ret && force_space);                 // loop runs at most twice
}
```

**Takeaway:** one `do/while` loop that runs at most twice — round 1, then round
2. `force_space` is just the round number as a bool. The `goto bounce` is the
two-step move from Idea 3.

---

## Step 3 — Finding space: the two rounds

**File: `ttm/ttm_bo.c:962`**, inside `ttm_bo_alloc_resource()`.

This walks the wish list. One line does all the round-1-vs-round-2 magic:

```c
	// Walk the wish list entries in order (VRAM first, then TT, ...)
	for (i = 0; i < placement->num_placement; ++i) {
		const struct ttm_place *place = &placement->placement[i];

		man = ttm_manager_type(bdev, place->mem_type);   // the clerk for this
								 // memory type
		if (!man || !ttm_resource_manager_used(man))
			continue;      // no such warehouse on this machine.
				       // (On an integrated GPU there is no VRAM
				       //  manager, so VRAM entries skip silently.)

		// ==================== THE KEY LINE ====================
		if (place->flags & (force_space ? TTM_PL_FLAG_DESIRED
						: TTM_PL_FLAG_FALLBACK))
			continue;
		// Round 1 (force_space == false): skip entries marked FALLBACK.
		//    -> only the preferred spot is considered.
		// Round 2 (force_space == true):  skip entries marked DESIRED.
		//    -> only the fallback spot is considered.
		// ======================================================

		ret = ttm_bo_alloc_at_place(bo, place, force_space, res, &alloc_state);

		if (ret == -ENOSPC) {
			continue;              // this warehouse is full -> try the
					       // next wish-list entry
		} else if (ret == -EBUSY) {
			// Full, but there ARE crates in there we could move out.
			// Go evict somebody. (Step 4)
			ret = ttm_bo_evict_alloc(bdev, man, place, bo, ctx,
						 ticket, res, &alloc_state);
			...
		}
		return 0;                      // got space!
	}

	return -ENOSPC;                        // nothing on the wish list worked
```

To be completely concrete, for a wish list of
`[VRAM0 (desired), TT (fallback)]`:

| | round 1 | round 2 |
|--|---------|---------|
| VRAM0 entry | considered — evict for it | skipped |
| TT entry | skipped | considered |

**Takeaway:** the ternary inverts which entries get skipped. That's all
"evict before you settle" is.

---

## Step 4 — Choosing a victim

**File: `ttm/ttm_bo.c:720`**, `ttm_bo_evict_alloc()`.

TTM walks the LRU list (oldest first) looking for someone to move out. It does
this **twice**, and the reason is nice:

```c
	// SWEEP 1: only consider victims whose lock happens to be free right now.
	evict_walk.walk.arg.trylock_only = true;
	lret = ttm_lru_walk_for_evict(&evict_walk.walk, bdev, man, 1);
	//   Cheap and never blocks. If we find an easy victim, great.

	if (lret || !ticket)
		goto out;                    // found one (or we're not allowed to
					     // contend for locks) -> stop here

	// SWEEP 2: nothing easy. Now fight for locks properly.
	evict_walk.walk.arg.trylock_only = false;
retry:
	do {
		evict_walk.walk.arg.ticket = ticket;   // ww-mutex ticket: lets us
						       // "wound" other threads and
						       // make them retry
		evict_walk.evicted = 0;
		lret = ttm_lru_walk_for_evict(&evict_walk.walk, bdev, man, 1);
	} while (!lret && evict_walk.evicted);         // keep evicting until an
						       // allocation succeeds, or
						       // nothing is left to evict
```

For each candidate, TTM asks the driver *"is evicting this one worth it?"*.
Back in Xe:

**File: `xe_bo.c:1241`**

```c
static bool
xe_bo_eviction_valuable(struct ttm_buffer_object *bo, const struct ttm_place *place)
{
	struct drm_gpuvm_bo *vm_bo;

	if (!ttm_bo_eviction_valuable(bo, place))
		return false;
	// TTM's own check first: would moving this buffer actually free space in
	// the address range we need? If not, moving it is pointless work.

	if (!xe_bo_is_xe_bo(bo))
		return true;                 // not our buffer, no opinion

	// Xe's extra rule. Walk every VM this buffer is mapped into:
	drm_gem_for_each_gpuvm_bo(vm_bo, &bo->base) {
		if (xe_vm_is_validating(gpuvm_to_vm(vm_bo->vm)))
			return false;
		// ^^^ THE VETO. This VM is validating RIGHT NOW (the flag raised
		//     back in Step 1). If we evicted this buffer, that validation
		//     would be destroying its own working set and would never
		//     finish. So: hands off.
	}

	return true;                         // safe to evict
}
```

**Takeaway:** two sweeps (easy victims, then contested ones), and one veto that
prevents a submission from eating itself.

---

## Step 5 — Now move the bytes: `xe_bo_move()`

**File: `xe_bo.c:962`** — back in Xe.

This function is long (~240 lines), but it is only a **list of special cases,
checked in order**, ending in one real copy. Don't read it as 240 lines; read it
as a checklist.

First it computes two booleans that decide everything (`:1012`):

```c
	// Is there anything worth copying FROM?
	move_lacks_source = !old_mem ||             // brand-new buffer, no old spot
			    (!mem_type_is_vram(old_mem_type) && !tt_has_data);
			    // ^ or: not in VRAM, and its pages were never filled

	// Must the destination be zeroed?
	needs_clear = (ttm && ttm->page_flags & TTM_TT_FLAG_ZERO_ALLOC) ||
		      (!ttm && ttm_bo->type == ttm_bo_type_device);
		      // ^ a user-visible buffer with no page list: we don't know
		      //   what's in the destination, and must not leak it
```

Those two bits pick one of three behaviours:

| `move_lacks_source` | `needs_clear` | what happens |
|---|---|---|
| true | false | **nothing** — just swap the resource pointer |
| true | true | **clear** the destination |
| false | — | **copy** source → destination |

Then the checklist. Here it is with only what matters:

```c
:985   if (evict && buffer is marked "don't need")
	       -> throw the contents away, done.          // purge

:995   if (this is a brand-new buffer)
	       -> ttm_bo_move_null()                      // 0 bytes copied

:1029  if (move_lacks_source && !needs_clear)
	       -> ttm_bo_move_null()                      // 0 bytes copied

:1045  if (old == SYSTEM && new == TT)
	       -> ttm_bo_move_null()                      // 0 bytes copied!
	       // Same pages. They just became DMA-mapped for the device.

:1060  if (there IS data here)
	       -> xe_bo_move_notify()                     // Step 6: tell the VMs

:1066  if (old == TT && new == SYSTEM)
	       -> wait for GPU work, then ttm_bo_move_null()   // 0 bytes copied!
	       // Same pages. They just stopped being DMA-mapped.

:1082  if (SYSTEM <-> VRAM and there is real data)
	       -> return -EMULTIHOP                       // "can't do it directly"

:1132  -> submit a real GPU copy or clear                 // Step 7
```

Count the `ttm_bo_move_null()` calls: **five of them.** Five "moves" that copy
zero bytes. Two of those are `SYSTEM ↔ TT` — exactly Idea 3's *"change of
status, not location"*.

### The two-step move, in code (`:1082`)

```c
	if (!move_lacks_source &&                       // there IS data to copy, AND
	    ((old_mem_type == XE_PL_SYSTEM && resource_is_vram(new_mem)) ||
	     (mem_type_is_vram(old_mem_type) && new_mem->mem_type == XE_PL_SYSTEM))) {
	    //  ^ we're going SYSTEM -> VRAM, or VRAM -> SYSTEM

		hop->mem_type = XE_PL_TT;               // "stop at TT on the way"
		hop->flags = TTM_PL_FLAG_TEMPORARY;     // "TT is just a way station"
		ret = -EMULTIHOP;                       // tell TTM: do it in 2 steps
		goto out;
	}
```

Notice the guard `!move_lacks_source`. If there's no data to copy, no copy
engine is involved, so no stopover is needed. **The stopover exists purely
because the blitter cannot address `XE_PL_SYSTEM`.**

---

## Step 6 — Tell the VMs their maps are stale

**File: `xe_bo.c:805`**, `xe_bo_move_notify()`.

```c
	if (xe_bo_is_pinned(bo))
		return -EINVAL;                  // pinned buffers never move

	xe_bo_vunmap(bo);                        // 1. kill any kernel CPU mapping
	ret = xe_bo_trigger_rebind(xe, bo, ctx); // 2. tell every VM  <-- the big one
	if (ret)
		return ret;

	if (ttm_bo->base.dma_buf && !ttm_bo->base.import_attach)
		dma_buf_invalidate_mappings(ttm_bo->base.dma_buf);
					         // 3. tell OTHER DEVICES that
					         //    imported this buffer
```

And `xe_bo_trigger_rebind()` at `:669` is where Idea 4's two strategies split:

```c
	// Walk every VM that has this buffer mapped:
	drm_gem_for_each_gpuvm_bo(vm_bo, obj) {
		struct xe_vm *vm = gpuvm_to_vm(vm_bo->vm);

		if (!xe_vm_in_fault_mode(vm)) {
			// ---- NORMAL VM ----
			drm_gpuvm_bo_evict(vm_bo, true);
			// Just MARK the mappings as stale. Somebody else rebuilds
			// them before this VM's next submission. No waiting, no
			// page-table writes here.
			if (!xe_device_is_l2_flush_optimized(xe))
				continue;        // <-- done with this VM, next!
		}

		// ---- FAULT-MODE VM (falls through to here) ----
		if (!idle) {
			// Wait for GPU work that is using these mappings.
			// You cannot pull the rug out from a running job.
			timeout = dma_resv_wait_timeout(bo->ttm.base.resv,
							DMA_RESV_USAGE_BOOKKEEP, ...);
			idle = true;             // only wait once, not per VM
		}

		// Wipe the page table entries NOW. They will be rebuilt later,
		// automatically, by page faults. (Chapter 10)
		drm_gpuvm_bo_for_each_va(gpuva, vm_bo) {
			struct xe_vma *vma = gpuva_to_vma(gpuva);
			ret = xe_vm_invalidate_vma(vma);
		}
	}
```

**Takeaway:** that `continue` is the fork. Normal VMs are marked and skipped;
fault-mode VMs get their page tables wiped immediately.

---

## Step 7 — The actual copy

**File: `xe_bo.c:1095`.** First, *which* copy engine?

```c
	if (bo->tile)
		migrate = bo->tile->migrate;          // a kernel buffer: use its tile
	else if (resource_is_vram(new_mem))
		migrate = mem_type_to_migrate(xe, new_mem->mem_type);
						      // going INTO VRAM: use the
						      // destination tile's engine
	else if (mem_type_is_vram(old_mem_type))
		migrate = mem_type_to_migrate(xe, old_mem_type);
						      // coming OUT of VRAM: use the
						      // source tile's engine
	else
		migrate = xe->tiles[0].migrate;       // neither side is VRAM: tile 0
```

This is Chapter 1's `tile->migrate` finally being used. The rule is *use the
tile that owns the VRAM*, because its blitter has local bandwidth to it.

Then the copy itself (`:1132`):

```c
	if (move_lacks_source) {
		// Nothing to copy. Just zero the destination.
		fence = xe_migrate_clear(migrate, bo, new_mem, flags);
	} else {
		// Real copy: old location -> new location.
		fence = xe_migrate_copy(migrate, bo, bo, old_mem, new_mem,
					handle_system_ccs);
	}

	// Look at the type of `fence`: struct dma_fence *.
	// Nothing has been copied yet! We submitted a GPU job and got a promise.

	ret = ttm_bo_move_accel_cleanup(ttm_bo, fence, evict, true, new_mem);
	// TTM: "park the old resource until this fence signals, install the new
	//       one, and put the fence on the buffer's clipboard under KERNEL."
	// Everyone who touches this buffer later will wait on that fence.
```

**Takeaway:** the return type `struct dma_fence *` *is* Idea 3. The move
function returns a promise, not finished bytes.

---

## Step 8 — The one place that must block

**File: `xe_bo.c:1183`**, the end of `xe_bo_move()`.

```c
out:
	if ((!ttm_bo->resource ||
	     ttm_bo->resource->mem_type == XE_PL_SYSTEM) && ttm_bo->ttm) {
		// We are landing in XE_PL_SYSTEM. These pages are about to stop
		// being reachable by the device.
		dma_resv_wait_timeout(ttm_bo->base.resv,
				      DMA_RESV_USAGE_KERNEL, false,
				      MAX_SCHEDULE_TIMEOUT);
		// So we MUST wait: you cannot un-map memory that a copy engine is
		// still reading from.

		xe_tt_unmap_sg(xe, ttm_bo->ttm);   // now safe to unmap
	}
```

Everything else in this chapter is asynchronous. **This is the exception**, and
the reason is physical: unmapping memory out from under a running copy would
corrupt it.

---

## The journey, end to end

```
 Step 1  xe_bo_validate()            xe_bo.c:3302    set up, raise flag, call TTM
 Step 2  ttm_bo_validate()           ttm_bo.c:1074   already fine? else loop twice
 Step 3  ttm_bo_alloc_resource()     ttm_bo.c:962    round 1 / round 2 skip line
 Step 4  ttm_bo_evict_alloc()        ttm_bo.c:720    2 LRU sweeps
         xe_bo_eviction_valuable()   xe_bo.c:1241    the veto
 Step 5  xe_bo_move()                xe_bo.c:962     checklist of special cases
 Step 6  xe_bo_move_notify()         xe_bo.c:805     tell the VMs
         xe_bo_trigger_rebind()      xe_bo.c:669     normal vs fault-mode fork
 Step 7  xe_migrate_copy/clear()     xe_bo.c:1132    submit GPU job, get fence
 Step 8  out:                        xe_bo.c:1183    block only if landing in SYSTEM
```

## The other callbacks, in one table

TTM asks Xe thirteen questions in total (`xe_bo.c:1828`). We used four of them
above. Here are the rest, one line each:

| Callback | What it does |
|----------|-------------|
| `evict_flags` `:315` | "where should this evicted buffer go?" VRAM→TT, TT→SYSTEM, don't-need→purge |
| `io_mem_reserve` | "how does the CPU map this?" VRAM → BAR offsets; refuses if outside the window |
| `swap_notify` | about to swap out — if marked don't-need, discard instead |
| `delete_mem_notify` | resource going away — detach imported dma-buf pages |
| `release_notify` | buffer dying — replace unsignalled preempt fences with stubs so teardown can't hang |
| `ttm_tt_*` (4 of them) | the page list — all covered in Chapter 3 |

One detail from `evict_flags` worth seeing, because it's a lovely trick
(`xe_bo.c:64`):

```c
static struct ttm_placement purge_placement;   // no initializer at all!
```

A zero-initialized `ttm_placement` has `num_placement == 0`. And Step 2's very
first check was `if (!placement->num_placement) return ttm_bo_pipeline_gutting(bo);`
— throw the contents away. **"Discard this buffer" is implemented as an
uninitialized variable.**

## Try it on your machine

```bash
# watch real migrations happen
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_bo_move/enable
cat /sys/kernel/debug/tracing/trace_pipe
# prints: source type, destination type, move_lacks_source
#         - the three things that pick a rung in Step 5

# count the zero-byte moves for yourself
grep -n 'ttm_bo_move_null' drivers/gpu/drm/xe/xe_bo.c
```

## Checkpoint

1. In Step 3, what does round 1 refuse to look at, and what does round 2 refuse
   to look at?
2. Step 5 calls `ttm_bo_move_null()` for `SYSTEM → TT`. Why is copying zero
   bytes correct there?
3. Why can't the GPU copy a buffer straight from VRAM to `XE_PL_SYSTEM`?
4. Why does `xe_bo_eviction_valuable()` return false for a VM that is currently
   validating?
5. Everything in Step 7 is asynchronous, but Step 8 blocks. Why?

<details>
<summary>answers</summary>

1. Round 1 refuses to look at `FALLBACK` entries (so it only tries the preferred
   spot, evicting if needed). Round 2 refuses to look at `DESIRED` entries
   (because those already failed).
2. The pages don't move. `SYSTEM` and `TT` are the *same* system RAM; the only
   difference is whether those pages are DMA-mapped for the device. So the move
   is a status change, not a data copy.
3. Because the GPU's copy engine does the copying, and `XE_PL_SYSTEM` pages are
   by definition not reachable by the device. It goes VRAM → TT (real copy),
   then TT → SYSTEM (zero bytes).
4. Because that validation is in the middle of collecting a working set. Evicting
   one of its buffers would make it destroy its own progress, and it would never
   finish.
5. Step 8 only runs when the buffer is landing in `XE_PL_SYSTEM`, which means its
   pages are about to be unmapped from the device. You cannot unmap memory a
   copy engine may still be reading.

</details>
