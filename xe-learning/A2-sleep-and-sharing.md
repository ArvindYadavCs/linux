# Appendix A2 — When the Lights Go Out

> Suspend, resume, and sharing a buffer with another driver.

---

## Beat A — The story

### Idea 1 — VRAM forgets everything

When the laptop sleeps, VRAM loses power. Every byte in it is gone.

So suspend means: **copy all of VRAM into system memory**, which does
survive. Resume means: copy it back.

That includes things you might not think of. The page tables live in VRAM
(Chapter 7). The GGTT lives there too (Chapter 5). So on resume, not only
is the data gone — **the maps that said where the data was are gone.**

### Idea 2 — Some of that memory *is* the machinery doing the copying

The copy is done by the GPU's copy engine, which needs:

```
  the copy engine  needs  the GuC running
  the GuC          needs  its firmware image  ← which lives in VRAM
```

So you cannot simply copy everything out. You must **use the machinery
first, and move the machinery last.** And on the way back up you have the
same problem in reverse: the GuC's firmware has to be in VRAM before the
GPU can copy anything else.

That is why pinned memory is split in two:

| Group | Moved by | When on resume |
|---|---|---|
| **early** | the **CPU** | before the GPU exists |
| **late** | the **GPU** | after the GuC is running |

### Idea 3 — And most memory needs no code at all

Here is the punchline. Ordinary buffers — the app's textures, vertex data,
all of it — get **no special suspend handling whatsoever**.

They were already evictable (Chapter 4), and an evicted buffer is already
re-validated and re-bound on its next use (Chapter 13). Suspend just evicts
them, and the normal path does the rest.

**Only the exceptions needed new code.** That's the lesson of this
appendix.

---

## Beat B — The code

> Trimmed: exported dma-bufs are left out of the suspend code here and
> covered in their own section below.

### Two characters

| | our texture | the GuC firmware |
|---|---|---|
| Size | 256MB | small |
| Pinned? | **no** | **yes**, "early" |
| Who moves it | the GPU | the **CPU** |
| Restored on resume? | **never** | first thing |

The user closes the lid. Follow both.

### Named functions

| Function | Where |
|---|---|
| `xe_bo_evict_all()` | `xe_bo_evict.c:160` |
| `xe_bo_restore_early()` | `xe_bo_evict.c:226` |
| `xe_bo_restore_late()` | `xe_bo_evict.c:243` |

### The bookkeeping, in one picture

Pinned BOs live on lists. Two pairs, and a BO moves right at suspend and
left at resume. That is the whole of it:

```
   xe->pinned.early.kernel_bo_present  ◄──►  xe->pinned.early.evicted
   xe->pinned.late.kernel_bo_present   ◄──►  xe->pinned.late.evicted

                    suspend ──►        ◄── resume
```

---

### Step 1 — Going to sleep

```c
// xe_bo_evict.c:160
int xe_bo_evict_all(struct xe_device *xe)
{
	// (1) Everything ordinary — our 256MB texture. GPU copies it out.
	xe_bo_evict_all_user(xe);

	// (2) Pinned BOs the GPU can still move. GPU copies them, and they
	//     move from the "present" list to the "evicted" list.
	xe_bo_apply_to_pinned(xe, &xe->pinned.late.kernel_bo_present,
			      &xe->pinned.late.evicted, xe_bo_evict_pinned);

	// (3) WAIT. Stages 1 and 2 used the GPU. They must FINISH before
	//     we take the GPU apart.
	for_each_tile(tile, xe, id)
		xe_tile_migrate_wait(tile);

	// (4) Only now the machinery itself — the GuC firmware. The CPU
	//     copies this one, because after this the GPU can't copy at all.
	return xe_bo_apply_to_pinned(xe, &xe->pinned.early.kernel_bo_present,
				     &xe->pinned.early.evicted, xe_bo_evict_pinned);
}
```

Step (3) is the one to remember. The source comment says it outright:

```c
/*
 * Wait for all user BO to be evicted as those evictions depend on the
 * memory moved below.
 */
```

*The memory moved below* is the GuC image. Stages 1 and 2 cannot still be
running when stage 4 moves it.

### Step 2 — Waking up, part one: the CPU does the bootstrap

```c
// xe_pm.c:258
/*
 * This only restores pinned memory which is the memory required for the
 * GT(s) to resume.
 */
err = xe_bo_restore_early(xe);
```

```c
// xe_bo_evict.c:226 — the exact mirror of step 1 stage (4)
int xe_bo_restore_early(struct xe_device *xe)
{
	return xe_bo_apply_to_pinned(xe, &xe->pinned.early.evicted,
				     &xe->pinned.early.kernel_bo_present,
				     xe_bo_restore_and_map_ggtt);
}
```

Look at the callback's name — restoring is **two** jobs, not one:

```c
// xe_bo_evict.c:192
static int xe_bo_restore_and_map_ggtt(struct xe_bo *bo)
{
	ret = xe_bo_restore_pinned(bo);        // 1. copy the bytes back to VRAM
	if (ret)
		return ret;

	// 2. The GGTT's PTEs did not survive the power cut either, so
	//    re-insert this BO's GGTT mapping by hand.
	if (bo->flags & XE_BO_FLAG_GGTT)
		for_each_tile(tile, xe_bo_device(bo), id)
			xe_ggtt_map_bo_unlocked(tile->mem.ggtt, bo);

	return 0;
}
```

That second half is Idea 1's twist in code: the data came back, but the
*address* it lives at had to be re-established too.

Now, and only now, the GPU can start:

```c
// xe_pm.c:266
xe_irq_resume(xe);
for_each_gt(gt, xe, id)
	err = xe_gt_resume(gt);        // the GuC loads — its image is back
```

### Step 3 — Waking up, part two: the GPU does the rest

```c
// xe_bo_evict.c:243
int xe_bo_restore_late(struct xe_device *xe)
{
	// Same shape, but these go back via the copy engine, which works again.
	xe_bo_apply_to_pinned(xe, &xe->pinned.late.evicted,
			      &xe->pinned.late.kernel_bo_present,
			      xe_bo_restore_and_map_ggtt);

	for_each_tile(tile, xe, id)
		xe_tile_migrate_wait(tile);
	...
}
```

### Step 4 — And the texture?

**Nothing restores it.**

Search `xe_bo_evict.c` for it and you'll find nothing — it isn't pinned, so
it isn't on any list. It stays in system memory after resume.

Then the game submits its next frame, and Chapter 13's ordinary `EXEC` path
runs:

```
  EXEC -> xe_vm_validate_rebind()   "is everything resident?"
            -> TTM moves it back to VRAM        (ch 4)
            -> PTEs rewritten at the new place  (ch 8)
```

The same three lines that handle an eviction under memory pressure. The
game never knows a suspend happened.

So the entire suspend/resume feature is: **a list for the exceptions, and a
rule about what order to move them in.** Everything else was already there.

---

## Sharing a buffer with another driver

`dma-buf` hands your BO to a different device. One thing can surprise you:

```c
// xe_dma_buf.c:49
static int xe_dma_buf_pin(struct dma_buf_attachment *attach)
{
	bool allow_vram = true;

	// If ANY importer can't do PCIe peer-to-peer, VRAM is off the table
	// for ALL of them — there is one buffer, and it has one location.
	list_for_each_entry(attach, &dmabuf->attachments, node) {
		if (!attach->peer2peer) {
			allow_vram = false;
			break;
		}
	}

	if (!allow_vram)
		xe_bo_migrate(bo, XE_PL_TT, NULL, exec);   // forced to system RAM

	return xe_bo_pin_external(bo, !allow_vram, exec);
}
```

**One importer that can't reach VRAM drags the buffer out of VRAM for
everybody.** There is no per-importer placement; the buffer is in one place.

(`xe_bo_pin_external()` adds it to a third list,
`xe->pinned.late.external`, which suspend walks alongside the other two —
that's the stage I left out of Step 1.)

And the other direction. When Xe *imports* someone else's buffer, that
someone may move it at any time, and tells us:

```c
// xe_dma_buf.c:318
static void xe_dma_buf_move_notify(struct dma_buf_attachment *attach)
{
	struct xe_bo *bo = gem_to_xe_bo(attach->importer_priv);

	XE_WARN_ON(xe_bo_evict(bo, exec));   // just drop our placement
}
```

Which is the same move as every notifier in this course: **mark it
unusable, repair on next use.** The next `EXEC` validates and re-binds it.
No new code path.

---

## What to carry forward

1. **Use the machinery, then move the machinery.** Suspend copies with the
   GPU, waits for those copies to finish, and only then hand-copies the
   GPU's own firmware out. Resume is the same thing backwards.
2. **"Early" means the CPU moves it.** One flag decides which list a pinned
   BO joins and which resume phase touches it. Everything else follows.
3. **Only exceptions needed code.** The 256MB texture gets zero special
   handling, because evict-and-revalidate already existed. When a new
   feature only has to describe its exceptions, the layers under it were
   designed right.

---

That completes the course: **16 chapters + 2 appendices**. See `README.md`
for the full map.
