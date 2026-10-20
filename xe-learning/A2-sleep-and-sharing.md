# Appendix A2 — When the Lights Go Out

> Pinning, suspend/resume, and dma-buf. The last optional branch.

---

## Beat A — The story

### Idea 1 — VRAM is volatile, so suspend means "copy everything out"

Chapter 4's premise was *"TTM can move a BO when it needs to."* Suspend is
that, for **everything at once**, with one extra rule: the destination has
to survive power loss. So everything in VRAM goes to system memory.

And the twist that makes it interesting: **the page tables were in VRAM
too** (Chapter 7 — each tile's PT tree lives in that tile's VRAM). So does
the GGTT. On resume, nothing is where its PTEs said it was, and the PTEs
themselves are gone.

### Idea 2 — Pinning creates a bootstrap problem

Most BOs are easy: TTM evicts them with the GPU's copy engine, as in
Chapter 4. But some BOs are **pinned** — they may never move during normal
operation:

```
the GuC firmware image      the CT mailbox rings (ch 12)
the kernel queues' LRCs     the page tables themselves
```

At suspend they must move anyway. And at resume you hit a chicken-and-egg:

```
restoring BOs with the GPU  needs  the copy engine
the copy engine             needs  the GuC running
the GuC running             needs  its firmware image restored   ← a pinned BO
```

The way out is to split pinned BOs into two groups — **early** (restored by
the *CPU*, before the GPU exists) and **late** (restored by the GPU, once it
does). That one split explains the whole file.

### Idea 3 — An exported buffer has two owners and no boss

`dma-buf` hands your BO to another device. Two consequences:

- That device may not be able to reach VRAM at all (no PCIe peer-to-peer),
  so exporting can **force a migration to system memory**.
- It may hold a live mapping, so you cannot move the BO without telling it
  — hence `move_notify`.

This is Chapter 14's shared-`dma_resv` problem seen from the other side:
there, someone else's wait preempted your GPU; here, your move invalidates
someone else's mapping.

---

## Beat B — The code

> Trimmed: CCS/compression, PXP, SR-IOV and display paths removed.

### The worked example

A laptop with a discrete Xe GPU. A game is running, with a 256MB texture in
VRAM. The GuC firmware and its CT rings are pinned in VRAM. **The user
closes the lid, then opens it again.**

### Named functions

| Function | Where | Job |
|---|---|---|
| `xe_bo_pin()` | `xe_bo.c:3177` | decide early vs late |
| `xe_bo_evict_all()` | `xe_bo_evict.c:160` | the whole suspend sequence |
| `xe_bo_restore_early()` | `xe_bo_evict.c:226` | CPU-copy the bootstrap BOs back |
| `xe_bo_restore_late()` | `xe_bo_evict.c:243` | GPU-copy the rest |
| `xe_dma_buf_pin()` | `xe_dma_buf.c:49` | P2P, or migrate to system |
| `xe_dma_buf_move_notify()` | `xe_dma_buf.c:318` | the importer's eviction hook |

---

### Step 1 — Pin time: which list does it join?

Long before suspend, every pinned BO picks a side:

```c
// xe_bo.c:3203
if (mem_type_is_vram(place->mem_type) || bo->flags & XE_BO_FLAG_GGTT) {
	spin_lock(&xe->pinned.lock);
	if (bo->flags & XE_BO_FLAG_PINNED_LATE_RESTORE)
		list_add_tail(&bo->pinned_link, &xe->pinned.late.kernel_bo_present);
	else
		list_add_tail(&bo->pinned_link, &xe->pinned.early.kernel_bo_present);
	spin_unlock(&xe->pinned.lock);
}
ttm_bo_pin(&bo->ttm);
```

And "early" has a mechanical consequence, decided at allocation time:

```c
// xe_bo.c:191
static bool force_contiguous(u32 bo_flags)
{
	if (bo_flags & XE_BO_FLAG_STOLEN)
		return true;
	else if (bo_flags & XE_BO_FLAG_PINNED &&
		 !(bo_flags & XE_BO_FLAG_PINNED_LATE_RESTORE))
		return true;     /* needs vmap */
	...
}
```

**An early-restore BO must be physically contiguous**, because it will be
copied by the CPU through `xe_bo_vmap()`, and only contiguous BOs can be
vmapped. "Early" and "CPU copy" and "contiguous" are three names for the
same decision.

### Step 2 — Suspend: four stages, and the order is the whole point

```c
// xe_bo_evict.c:160
int xe_bo_evict_all(struct xe_device *xe)
{
	// (1) Ordinary non-pinned user BOs — our 256MB texture. GPU copy.
	ret = xe_bo_evict_all_user(xe);

	// (2) Pinned EXTERNAL BOs (exported dma-bufs). GPU copy.
	ret = xe_bo_apply_to_pinned(xe, &xe->pinned.late.external,
				    &xe->pinned.late.external, xe_bo_evict_pinned);

	// (3) Pinned LATE kernel BOs. GPU copy. Note the list move:
	//     kernel_bo_present -> evicted, so resume knows what to bring back.
	if (!ret)
		ret = xe_bo_apply_to_pinned(xe, &xe->pinned.late.kernel_bo_present,
					    &xe->pinned.late.evicted, xe_bo_evict_pinned);

	/*
	 * Wait for all user BO to be evicted as those evictions depend on the
	 * memory moved below.
	 */
	for_each_tile(tile, xe, id)
		xe_tile_migrate_wait(tile);

	if (ret)
		return ret;

	// (4) Only NOW the early BOs — the machinery stages 1-3 were using.
	//     CPU copy, because the GPU is about to stop being usable.
	return xe_bo_apply_to_pinned(xe, &xe->pinned.early.kernel_bo_present,
				     &xe->pinned.early.evicted,
				     xe_bo_evict_pinned);
}
```

Read that source comment again — it is the key sentence in the file:

> *"Wait for all user BO to be evicted as those evictions depend on the
> memory moved below."*

Stages 1–3 are GPU copies. A GPU copy needs the migrate context, the GuC,
and the CT rings — which are **exactly** the early BOs that stage 4 moves.
So: use the machinery, **wait for it to finish**, and only then dismantle
it. Reverse any two of those and the GPU is copying with half its
infrastructure already in system memory.

Then the rest of suspend:

```c
// xe_pm.c:195 — eviction comes BEFORE the GTs are suspended
err = xe_bo_evict_all(xe);

for_each_gt(gt, xe, id)
	err = xe_gt_suspend(gt);

xe_irq_suspend(xe);
```

### Step 3 — Resume, early: hand-copy the bootstrap

```c
// xe_pm.c:258
/*
 * This only restores pinned memory which is the memory required for the
 * GT(s) to resume.
 */
err = xe_bo_restore_early(xe);
```

```c
// xe_bo_evict.c:226 — the exact mirror of stage 4
int xe_bo_restore_early(struct xe_device *xe)
{
	return xe_bo_apply_to_pinned(xe, &xe->pinned.early.evicted,
				     &xe->pinned.early.kernel_bo_present,
				     xe_bo_restore_and_map_ggtt);
}
```

And note what restoring has to include — the GGTT is gone too:

```c
// xe_bo_evict.c:192
static int xe_bo_restore_and_map_ggtt(struct xe_bo *bo)
{
	ret = xe_bo_restore_pinned(bo);      // copy the bytes back to VRAM
	if (ret)
		return ret;

	// The GGTT's own PTEs did not survive the power cycle (ch 5), so
	// re-insert this BO's GGTT mapping by hand.
	if (bo->flags & XE_BO_FLAG_GGTT) {
		for_each_tile(tile, xe_bo_device(bo), id) {
			if (tile != bo->tile && !(bo->flags & XE_BO_FLAG_GGTTx(tile)))
				continue;
			xe_ggtt_map_bo_unlocked(tile->mem.ggtt, bo);
		}
	}
	return 0;
}
```

Only after this can the GPU come up:

```c
// xe_pm.c:266
xe_irq_resume(xe);
for_each_gt(gt, xe, id)
	err = xe_gt_resume(gt);          // GuC loads — its image is back now
```

### Step 4 — Resume, late: the GPU does the bulk

```c
// xe_pm.c:282
err = xe_bo_restore_late(xe);
```

```c
// xe_bo_evict.c:243
int xe_bo_restore_late(struct xe_device *xe)
{
	// Pinned late kernel BOs, by GPU copy — the copy engine works again.
	ret = xe_bo_apply_to_pinned(xe, &xe->pinned.late.evicted,
				    &xe->pinned.late.kernel_bo_present,
				    xe_bo_restore_and_map_ggtt);

	for_each_tile(tile, xe, id)
		xe_tile_migrate_wait(tile);

	if (!IS_DGFX(xe))
		return 0;

	/* Pinned user memory in VRAM should be validated on resume */
	ret = xe_bo_apply_to_pinned(xe, &xe->pinned.late.external,
				    &xe->pinned.late.external,
				    xe_bo_restore_pinned);
	for_each_tile(tile, xe, id)
		xe_tile_migrate_wait(tile);

	return ret;
}
```

**And our 256MB texture?** It is never mentioned. It is not pinned, so
nobody restores it. It sits in system memory until the game's next
`EXEC`, where Chapter 13's step 2 validates it and Chapter 8 rewrites its
PTEs — the ordinary eviction-recovery path, which already existed.

That is the nicest thing in this appendix: **suspend/resume only had to
add code for the exceptions.** Everything that could already be evicted and
restored on demand needed nothing new.

---

### Step 5 — Exporting: can the other device even reach VRAM?

```c
// xe_dma_buf.c:49
static int xe_dma_buf_pin(struct dma_buf_attachment *attach)
{
	bool allow_vram = true;

	// If ANY importer lacks peer-to-peer, VRAM is off the table for ALL
	// of them — there is one buffer and one placement.
	list_for_each_entry(attach, &dmabuf->attachments, node) {
		if (!attach->peer2peer) {
			allow_vram = false;
			break;
		}
	}

	if (!allow_vram) {
		ret = xe_bo_migrate(bo, XE_PL_TT, NULL, exec);   // forced to system
		if (ret)
			return ret;
	}

	return xe_bo_pin_external(bo, !allow_vram, exec);
}
```

One importer without P2P support **drags the buffer out of VRAM for
everyone**. And then `xe_bo_pin_external()` puts it on
`xe->pinned.late.external` (`xe_bo.c:3150`) — which is exactly the list
stage 2 of suspend walks. The three subjects of this appendix meet in one
list.

The mapping itself branches on where the BO actually landed:

```c
// xe_dma_buf.c:119
switch (bo->ttm.resource->mem_type) {
case XE_PL_TT:
	// ordinary pages -> scatter-gather -> dma_map_sgtable
	sgt = drm_prime_pages_to_sg(obj->dev, bo->ttm.ttm->pages, ...);
	dma_map_sgtable(attach->dev, sgt, dir, DMA_ATTR_SKIP_CPU_SYNC);
	break;
case XE_PL_VRAM0:
case XE_PL_VRAM1:
	// P2P: build an sgt of DEVICE addresses, no struct pages involved
	xe_ttm_vram_mgr_alloc_sgt(xe_bo_device(bo), bo->ttm.resource, 0,
				  bo->ttm.base.size, attach->dev, dir, &sgt);
	break;
}
```

### Step 6 — Importing: somebody else's buffer just moved

When Xe *imports* a dma-buf, the exporter may move it at any time, and
tells us via the attach ops:

```c
// xe_dma_buf.c:327
static const struct dma_buf_attach_ops xe_dma_buf_attach_ops = {
	.allow_peer2peer     = true,
	.invalidate_mappings = xe_dma_buf_move_notify
};
```

```c
// xe_dma_buf.c:318 — "your mapping is stale"
static void xe_dma_buf_move_notify(struct dma_buf_attachment *attach)
{
	struct xe_bo *bo = gem_to_xe_bo(attach->importer_priv);

	XE_WARN_ON(xe_bo_evict(bo, exec));   // drop our placement entirely
}
```

Same shape as every notifier in this course: **mark it unusable now, repair
on next use.** The next `EXEC` validates the BO, re-maps the dma-buf, and
rewrites the PTEs. Chapter 13's `retry:` label covers this case too,
without knowing it exists.

### Step 7 — Runtime PM uses the same functions

Nothing new is needed for autosuspend; the same two phases, different
trigger:

```c
// xe_pm.c:631 (xe_pm_runtime_suspend)       |  // xe_pm.c:693 / :722
err = xe_bo_evict_all(xe);                   |  err = xe_bo_restore_early(xe);
                                             |  ...
                                             |  err = xe_bo_restore_late(xe);
```

And there is a lighter variant for the case where VRAM keeps power but the
device idles — only the user BOs go out, the pinned machinery stays:

```c
// xe_pm.c:448
err = xe_bo_evict_all_user(xe);      // stage 1 only
```

---

## The shared-`dma_resv` loop closes

Chapter 14's critique asked what happens when a BO is shared. Now both
halves are visible:

| Direction | Mechanism | Effect |
|---|---|---|
| someone else waits on **our** BO | their `dma_resv_wait` enables our preempt fence | **our** GPU queues get suspended (ch 14) |
| we move **their** BO | — | not our call; we must wait for `move_notify` |
| **they** move their BO | `invalidate_mappings` → `xe_dma_buf_move_notify()` | our mapping dies; next `EXEC` repairs it |
| one importer lacks P2P | `xe_dma_buf_pin()` | the buffer leaves VRAM for everyone |

Four different cross-driver interactions, and not one of them has a
coordinator. Each side only ever does two things: **publish a fence** so
others can wait, and **accept a notification** so it can repair itself.
That is the entire protocol for memory shared between drivers.

---

## What to carry forward

1. **Order follows dependency, not importance.** Suspend evicts with the
   GPU first and dismantles the GPU's own infrastructure last, with an
   explicit `xe_tile_migrate_wait()` between. Resume is the exact mirror:
   CPU-copy the bootstrap, start the GuC, then GPU-copy the rest.
2. **"Early" means contiguous means CPU-copyable.** One flag
   (`XE_BO_FLAG_PINNED_LATE_RESTORE`) decides a BO's allocation shape, its
   suspend list, and which resume phase touches it. Trace a flag like that
   through a driver and you learn its architecture.
3. **Exceptions are the only thing that needed new code.** The 256MB
   texture gets no special handling at all — eviction and
   validate-then-rebind already covered it. A feature that only has to
   handle its exceptions is a sign the layers underneath were right.

---

That completes the course: **16 chapters + 2 appendices**. See
`README.md` for the full map.
