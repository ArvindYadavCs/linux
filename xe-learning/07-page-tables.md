# Chapter 7 — Page Tables

> Beat A: the story.

Chapter 6 gave us VMAs — mappings the *driver* knows about. The hardware still
knows nothing. This chapter writes the bits the GPU actually reads.

Three ideas. We will use **one example all the way through**:

> Map 2 MB at GPU address **0x140000000** (that's 5 GB), backed by VRAM.

---

## Idea 1: A tree, not an array

GGTT in Chapter 5 was a flat array. Entry *N* covers page *N*. Simple, and fine
for 4 GB.

PPGTT covers **256 TB**. A flat array would need 64 billion entries — about
512 GB of page tables, for an address space that is nearly all empty.

So PPGTT is a **tree**, 4 levels deep. Each level has 512 entries, and each
level covers 512× more address space than the one below it:

| Level | One entry covers |
|-------|-----------------|
| 0 | 4 KB |
| 1 | 2 MB |
| 2 | 1 GB |
| 3 | 512 GB |
| 4 | the whole space (the root) |

**You only allocate the branches you actually use.** That's the entire point.

To translate an address, slice it into 9-bit chunks — each chunk is an index
into one level:

```
   0x140000000  (5 GB)

   level 4:  (0x140000000 >> 48) & 511  =  0
   level 3:  (0x140000000 >> 39) & 511  =  0
   level 2:  (0x140000000 >> 30) & 511  =  5     <- 5 GB / 1 GB per entry
   level 1:  (0x140000000 >> 21) & 511  =  0
   level 0:  (0x140000000 >> 12) & 511  =  0
```

So our address lives at: root entry 0 → entry 0 → **entry 5** → entry 0 →
entry 0.

---

## Idea 2: Huge pages — stop early

Normally you walk all the way to level 0, and each level-0 entry maps one 4 KB
page. For our 2 MB mapping that would be **512 entries**.

But look at the table again: **a level-1 entry already covers exactly 2 MB.**

So if the 2 MB of physical memory behind our mapping happens to be one
contiguous block, we don't need 512 small entries at all. We can set a single
flag on the level-1 entry that says *"don't treat me as a pointer to a child
table — treat me as the memory itself."*

**One entry instead of 513.** That's a **huge page**.

Levels 1 and 2 can both do it (2 MB and 1 GB). Three things must be true:

1. the mapping's **virtual** range exactly fills one entry (2 MB, 2 MB-aligned)
2. the **physical** memory is one contiguous run across the whole 2 MB
3. that physical address is itself 2 MB-aligned

Condition 2 is the interesting one. Contiguous VRAM → yes. Scattered system RAM
pages → almost never. *This is where Chapter 2's buddy allocator finally pays
off.*

Our example satisfies all three, so the entire 2 MB mapping will be **one 64-bit
write**.

---

## Idea 3: The GPU writes the entries — carefully

Two things happen here, and they fit together.

### The GPU does the writing

Chapter 4 showed the GPU's copy engine moving buffer contents. Here it does
something odder: **Xe builds a tiny GPU program whose job is to write page table
entries**, and submits it like any other work.

The command is a *store-immediate*: "write this 64-bit value to this address."

Why not just let the CPU store it?

* **Ordering comes free.** The entry writes sit in the same command stream as the
  work that uses them, so they are automatically ordered before it.
* **It returns a fence.** A submission can simply wait on "the bind finished" —
  Chapter 4's pattern again.
* **Reach.** Page tables live in VRAM, which the CPU may not even be able to see.

### But nothing is written until success is certain

A bind can fail halfway — out of memory for a new table, the buffer got evicted.
If you were editing the **live** tree when that happened, you'd leave a
half-updated tree that the GPU is actively walking. Disaster.

So Xe never edits the live tree. Three phases:

```
   PHASE 1  STAGE     work out the new shape in a parallel "staging" copy.
                      Allocate tables. Compute entry values. Collect them
                      into a list. The live tree is untouched.

   PHASE 2  SUBMIT    hand the list to the GPU as store commands. Get a fence.

   PHASE 3  COMMIT    only now, point the live tree at the staged tables.
```

Fail in phase 1 → throw the staging away, nothing changed.

This is the same shape as Chapter 6's plan-then-commit, one level down. Chapter
6 planned *which mappings* exist; Chapter 7 stages *which entries* exist.

---

## One footnote: unbinding needs more

Binding is "write entries, done." **Unbinding is not just the reverse.**

Zeroing an entry isn't enough, because the GPU caches translations in its
**TLB**. A cleared entry plus a stale TLB entry means the GPU keeps reading
memory you just freed.

So unbind is: write the entries → **invalidate the TLB** → wait for that to
finish → *now* the memory is safe to reuse. That invalidation is slow and has to
be asked of the GuC. It gets Chapter 9.

---

## Remember these three

1. **4-level tree; each level covers 512x more than the one below.**
2. **Huge pages mean stopping at level 1 (2 MB) or 2 (1 GB)** instead of
   descending — if the physical memory is contiguous.
3. **Stage, submit, commit.** The live tree is never edited in place.

---
---

# Chapter 7 — Beat B: The Code

Only **three functions** matter in this chapter. Everything else is plumbing:

| Function | File | Job |
|----------|------|-----|
| `xe_pt_stage_bind_entry()` | `xe_pt.c:559` | at each level: stop, or descend? |
| `write_pgtable()` | `xe_migrate.c:1742` | turn the result into GPU commands |
| `xe_pt_commit()` | `xe_pt.c:1179` | swap the staged tree in |

We follow our one example through all three.

> **All code below is trimmed.** I've removed error handling, SVM, 64K-compact
> layout, purged buffers and atomic-access bits so the shape is visible.
> `//` comments are mine.

---

## The picture, before we start

Our VM already has a root. Nothing else:

```
   BEFORE

   level 4  [ 0 ][ 1 ]...[511]          <- the root table (an xe_bo)
              |
              +-- entry 0: empty
```

What we want:

```
   AFTER

   level 4  [ 0 ] -------> level 3  [ 0 ] -------> level 2  [ 5 ] ---+
                                                                     |
                                      level 1  [ 0 ] <---------------+
                                                 |
                                                 +-- value 0x...893
                                                     ("this IS 2MB of VRAM")
```

Four tables, and **one meaningful entry** at the bottom. Note there is no
level-0 table at all — the huge page is why.

---

## Step 1 — A page table is just a buffer

**File: `xe_pt_types.h`**

```c
struct xe_pt {
	struct xe_ptw base;      // has TWO pointer arrays: children[] and
				 // staging[]. That is Idea 3's shadow tree,
				 // built into the node type.

	struct xe_bo *bo;        // <<< the real 4KB of memory holding this
				 // table's 512 entries.
				 // A PAGE TABLE IS A BUFFER OBJECT (ch 3).

	unsigned int level;      // 0 = leaf table, 4 = root
};
```

That `bo` is worth a pause. Page tables are allocated, placed and validated by
everything in Chapters 2–4 — they are not special memory.

And the shifts from Idea 1, verbatim — **`xe_pt.c:46`**:

```c
static const u64 xe_normal_pt_shifts[] = {12, 21, 30, 39, 48};
//                                        ^^  ^^  ^^  ^^  ^^
//                                        L0  L1  L2  L3  L4
// index at level L  =  (addr >> shifts[L]) & 511
```

---

## Step 2 — The one decision, made at every level

**File: `xe_pt.c:559`**, `xe_pt_stage_bind_entry()`.

The tree is walked from the root down, and this function is called once for each
entry on the way. It answers one question and returns one of two actions.

```c
	// ================= QUESTION: can I stop here? =================
	if (level == 0 || xe_pt_huge_leaf_allowed(addr, next, level, xe_walk)) {

		// YES -> build the entry that points at real memory.
		pte = vm->pt_ops->pte_encode_vma(xe_res_dma(curs),  // physical addr
						 xe_walk->vma,
						 pat_index,
						 level);            // level sets the
								    // huge-page bit
		pte |= xe_walk->default_vram_pte;     // adds the "device memory" bit

		xe_pt_insert_entry(xe_walk, xe_parent, offset, NULL, pte);
		//                                           ^^^^  ^^^
		//        no child table            the value we computed
		//   NOTHING is written to hardware - this just records the work.

		*action = ACTION_CONTINUE;      // <<< DO NOT DESCEND
		return 0;
	}

	// ================= NO -> descend. Need a child table? =================
	if (!*child) {
		xe_child = xe_pt_create(xe_walk->vm, xe_walk->tile, level - 1, ...);
		//         ^^^ allocates a 4KB xe_bo for the new table

		*child = &xe_child->base;      // <<< goes into the STAGING array,
					       //     not children[]

		// and an entry at THIS level pointing down at it
		pte = vm->pt_ops->pde_encode_bo(xe_child->bo, 0);
		xe_pt_insert_entry(xe_walk, xe_parent, offset, xe_child, pte);
	}

	*action = ACTION_SUBTREE;              // <<< DESCEND one level
	return 0;
```

Two actions, and they are Ideas 1 and 2:

* `ACTION_SUBTREE` — "this entry points at a child table; go deeper"
* `ACTION_CONTINUE` — "this entry *is* the answer; stop"

### For our example, this runs four times

| Call | level | 2 MB aligned & contiguous? | action |
|------|-------|---------------------------|--------|
| 1 | 4 | no — one entry covers 256 TB | allocate L3, **descend** |
| 2 | 3 | no — covers 512 GB | allocate L2, **descend** |
| 3 | 2 | no — covers 1 GB | allocate L1, **descend** |
| 4 | **1** | **yes — covers exactly 2 MB** | **one huge entry, stop** |

The walk never reaches level 0. That's the huge page.

### The three conditions, in code

**File: `xe_pt.c:441`** — Idea 2's list, one `if` each:

```c
	if (level > MAX_HUGEPTE_LEVEL)
		return false;             // only levels 1 and 2 can do this

	// CONDITION 1: does the virtual range exactly fill one entry?
	if (!xe_pt_covers(addr, next, level, &xe_walk->base))
		return false;

	// CONDITION 2: is the physical memory contiguous for the whole 2MB?
	if (next - xe_walk->va_curs_start > xe_walk->curs->size)
		return false;
	//   curs->size = bytes left in the current contiguous run.
	//   Scattered system RAM: this is 4096, so always fails.
	//   Contiguous VRAM:      this is large, so it passes.

	// CONDITION 3: is the physical address itself 2MB aligned?
	dma = addr - xe_walk->va_curs_start + xe_res_dma(xe_walk->curs);
	return IS_ALIGNED(dma, next - addr);
```

---

## Step 3 — What value gets written

**File: `xe_vm.c:1463`.** An entry is just *physical address, OR'd with some
flag bits*:

```c
static u64 xelp_pte_encode_bo(struct xe_bo *bo, u64 bo_offset,
			      u16 pat_index, u32 pt_level)
{
	pte  = xe_bo_addr(bo, bo_offset, XE_PAGE_SIZE);   // the physical address
	pte |= XE_PAGE_PRESENT | XE_PAGE_RW;              // bits 0 and 1
	pte |= pte_encode_pat_index(pat_index, pt_level); // cache attributes
	pte |= pte_encode_ps(pt_level);                   // the HUGE PAGE bit

	if (xe_bo_is_vram(bo))
		pte |= XE_PPGTT_PTE_DM;                   // bit 11: device memory
	return pte;
}
```

The huge-page bit is where level matters — **`xe_vm.c:1414`**:

```c
static u64 pte_encode_ps(u32 pt_level)
{
	if (pt_level == 1) return XE_PDE_PS_2M;    // bit 7: "this level-1 entry
						   //  IS 2MB of memory"
	if (pt_level == 2) return XE_PDPE_PS_1G;   // bit 7: same, for 1GB
	return 0;                                  // level 0: nothing needed
}
```

### Our value, bit by bit

VRAM at device address `0x640000000`, cache index 2 (which sets one PAT bit, at
bit 4):

```
   physical address                 0x0000000640000000
   XE_PAGE_PRESENT     bit 0      | 0x0000000000000001
   XE_PAGE_RW          bit 1      | 0x0000000000000002
   cache attribute     bit 4      | 0x0000000000000010
   XE_PDE_PS_2M        bit 7      | 0x0000000000000080   <- "I am a huge page"
   XE_PPGTT_PTE_DM     bit 11     | 0x0000000000000800   <- "device memory"
                                    ------------------
                                    0x0000000640000893
```

That single number maps 2 MB.

(The cache attribute bits are genuinely scattered across bits 3, 4, 7, 61 and
62, and bit 2 of the index moves between bit 7 and bit 12 depending on level —
because at levels 1 and 2 bit 7 is already the huge-page flag. That's
`pte_encode_pat_index()`. You don't need to memorize it; just know *why* it takes
`pt_level`.)

### Versus GGTT

| | GGTT (ch 5) | PPGTT (here) |
|--|-------------|--------------|
| "device memory" | bit 1 | bit 11 |
| read/write bit | none, always RW | bit 1 |
| huge page bit | none | bit 7 |
| our example | `0x0030000112345001` | `0x0000000640000893` |

---

## Step 4 — The work list, and the GPU command

Phase 1 is done. What it produced — **`xe_pt_types.h`**:

```c
struct xe_vm_pgtable_update {
	struct xe_bo *pt_bo;             // which table to write into
	u32 ofs;                         // first entry index
	u32 qwords;                      // how many entries
	struct xe_pt_entry *pt_entries;  // the values
};
```

For us, exactly one of these:

```
   { pt_bo = <the level-1 table>,  ofs = 0,  qwords = 1,
     pt_entries[0].pte = 0x0000000640000893 }
```

**Phase 2** turns it into GPU commands — **`xe_migrate.c:1742`**:

```c
	do {
		u64 addr = ppgtt_ofs + ofs * 8;        // 8 bytes per entry

		chunk = min(size, MAX_PTE_PER_SDI);    // one command writes at
						       // most 510 entries

		// ---- the command: "store immediate data" ----
		bb->cs[bb->len++] = MI_STORE_DATA_IMM | MI_SDI_NUM_QW(chunk);
		bb->cs[bb->len++] = lower_32_bits(addr);   // where to write
		bb->cs[bb->len++] = upper_32_bits(addr);

		// ---- the payload: the entry values, inlined into the command ----
		ops->populate(pt_update, tile, NULL, bb->cs + bb->len,
			      ofs, chunk, update);

		bb->len += chunk * 2;                  // 2 dwords per 64-bit value
		ofs += chunk;  size -= chunk;
	} while (size);
```

So our whole batch buffer is six dwords:

```
   MI_STORE_DATA_IMM | MI_SDI_NUM_QW(1)
   lower_32( <address of the level-1 table> + 0 )
   upper_32( <address of the level-1 table> + 0 )
   0x40000893                      <- low half of our entry
   0x00000006                      <- high half
```

Submit that, get a fence back. **Idea 3's first half, done.**

One nice detail in the function that fills the payload —
**`xe_pt.c:1095`**:

```c
	if (map)
		xe_map_wr(..., u64, ptes[idx].pte);   // CPU writes the table directly
	else
		ptr[i] = ptes[idx].pte;               // value goes into the GPU
						      // command buffer
```

**That `if` is the CPU-path / GPU-path fork.** Same function, two destinations.

---

## Step 5 — The commit

**File: `xe_pt.c:1179`.** The fence signalled; the entries are in memory. Make
the staged tree live:

```c
	for (i = 0; i < num_entries; i++) {
		pt_dir = as_xe_pt_dir(entries[i].pt);

		for (j = 0; j < entries[i].qwords; j++) {
			int j_ = j + entries[i].ofs;

			pt_dir->children[j_] = pt_dir->staging[j_];
			// ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
			// THIS IS THE COMMIT. One pointer assignment.
			// Everything else in the chapter exists so that we
			// only reach this line when success is certain.

			xe_pt_destroy(oldpte, ..., deferred);
			// free what the entry used to point at - deferred,
			// because the GPU may still be walking it
		}
	}
```

And on failure, `xe_pt_abort_bind()` at `:1206` does the opposite: restore
`staging` from `children`, free the tables we speculatively allocated. **Nothing
observable ever happened.**

### Who drives the three phases

Chapter 8's `VM_BIND` calls these in order:

```c
xe_pt_update_ops_prepare()   // xe_pt.c:2468   PHASE 1 - stage
xe_pt_update_ops_run()       // xe_pt.c:2704   PHASE 2 - submit, return a fence
xe_pt_update_ops_fini()      // xe_pt.c:2882   PHASE 3 - commit
xe_pt_update_ops_abort()     // xe_pt.c:2908   or: throw it away
```

`_run()` returns `struct dma_fence *` — the same signature shape as
`xe_migrate_copy()` in Chapter 4, because it is the same machinery.

---

## The whole chapter on one page

```
  VMA: 2MB at 0x140000000, VRAM at 0x640000000
                   |
  PHASE 1 ---------+--- xe_pt_stage_bind_entry(), called 4 times
                   |      L4: descend   L3: descend   L2: descend
                   |      L1: all 3 conditions pass -> ONE huge entry
                   |
                   |    work list: { L1 table, entry 0, 1 value, 0x...893 }
                   |
  PHASE 2 ---------+--- write_pgtable()
                   |      MI_STORE_DATA_IMM <L1 table+0> 0x...893
                   |      submit -> fence
                   |
  PHASE 3 ---------+--- xe_pt_commit()
                          children[0] = staging[0]
```

## Try it on your machine

```bash
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vm_pgtable_update/enable
cat /sys/kernel/debug/tracing/trace_pipe
# prints each staged entry: level, offset, value
```

## Checkpoint

1. Our 2 MB mapping needed **one** entry. How many would it have needed without
   the huge page, and what else would have had to be allocated?
2. Which of the three huge-page conditions usually fails for a buffer in system
   memory, and why?
3. `pte_encode_ps()` takes the level. What does it do with it?
4. Which single line of code is the commit?
5. Why isn't zeroing an entry enough to unmap a page?

<details>
<summary>answers</summary>

1. 512 entries at level 0 — plus a whole new level-0 table to hold them, plus a
   normal (non-huge) entry at level 1 pointing down at that table.
2. Condition 2 — contiguity. System memory pages are scattered, so
   `curs->size` is only 4096 bytes and the check fails immediately. VRAM
   allocated as one buddy block passes it.
3. At level 1 it sets bit 7 (`XE_PDE_PS_2M`), at level 2 it sets bit 7
   (`XE_PDPE_PS_1G`) — the flag that means "this entry is the memory, not a
   pointer to a child table". At level 0 it returns 0, because a level-0 entry
   always maps memory.
4. `pt_dir->children[j_] = pt_dir->staging[j_]` in `xe_pt_commit()`.
5. The GPU caches translations in its TLB. A zeroed entry plus a cached
   translation still resolves, so the GPU would keep reading memory that has
   been freed. The TLB must be invalidated and the invalidation confirmed
   complete.

</details>
