# Chapter 7 — Page Tables

> Beat A: the story.

Chapter 6 gave us VMAs — mappings the driver knows about. The hardware knows
nothing about them yet. This chapter turns intent into bits the GPU reads.

It is the chapter Act I has been building toward. Five ideas.

---

## Idea 1: A tree instead of an array

GGTT was a flat array (Chapter 5): one entry per page, index it, done. That
works for 4 GB.

PPGTT covers **256 TB**. A flat array would need 64 billion entries — 512 GB of
page tables for an address space that is mostly empty.

So PPGTT is a **tree**, four levels deep, exactly like a CPU page table. Each
level has 512 entries, and each entry either:

* **points at a child table** one level down (a *page directory entry*, PDE), or
* **points at actual memory** (a *page table entry*, PTE, a leaf).

Each level covers 512× more address space than the one below:

| Level | One entry covers | Called |
|-------|-----------------|--------|
| 0 | **4 KB** | a page |
| 1 | **2 MB** | 512 pages |
| 2 | **1 GB** | |
| 3 | **512 GB** | |
| 4 | 256 TB | the root |

The win is that you only allocate the branches you use. A VM with one 4 KB
buffer mapped needs five small tables, not 512 GB of them.

Translating an address is just slicing it into 9-bit indices:

```
   GPU address 0x140000000  (5 GB)
        |
        |  >> 48 & 511  ->  0    at the root, follow entry 0
        |  >> 39 & 511  ->  0    follow entry 0
        |  >> 30 & 511  ->  5    follow entry 5
        |  >> 21 & 511  ->  0    follow entry 0
        |  >> 12 & 511  ->  0    THIS entry holds the physical address
        v
   physical page
```

Five lookups instead of one. That's the price of 256 TB.

---

## Idea 2: Huge pages — stop early and win big

Here's the trick that makes this practical.

An entry at level 1 covers 2 MB. Normally it points at a table of 512 small
entries. But if all 2 MB of that region is **contiguous physical memory**, the
entry can just point at it directly — one entry instead of 513.

That's a **huge page**. Levels 1 and 2 can both do it:

| Level | Entry covers | Instead of |
|-------|-------------|------------|
| 1 | 2 MB directly | 512 entries + a table |
| 2 | 1 GB directly | 512 tables + 262,144 entries |

The savings are enormous — less memory for page tables, far fewer TLB entries,
much faster walks.

But there are conditions. To use a 2 MB entry:

1. the **virtual** range must be 2 MB aligned and exactly 2 MB, *and*
2. the **physical** memory must be contiguous across the whole 2 MB, *and*
3. the physical address must be 2 MB aligned.

Condition 2 is why VRAM gets huge pages and scattered system memory usually
doesn't. It's also why Chapter 2's buddy allocator cared about contiguity — the
payoff arrives here.

So building page tables isn't mechanical. At every level the driver must *ask*:
"can I stop here, or must I descend?"

---

## Idea 3: The GPU writes its own page tables

Chapter 4 showed the GPU's copy engine moving buffer contents. Now the same
machinery does something stranger:

> **Xe builds a little GPU program whose job is to write page table entries,
> and submits it to the GPU.**

The command is a store-immediate: *"write these 64-bit values to this address."*
A bind becomes a batch buffer full of those, submitted like any other work.

Why not just have the CPU write them?

* **Ordering for free.** The PTE writes land in the same command stream as the
  work that uses them, so they are automatically ordered before it — no fences,
  no barriers, no waiting.
* **It's a fence.** The bind returns a `dma_fence` (Chapter 4's pattern again),
  so a submission can simply depend on "the bind finished".
* **Speed.** Mapping a 1 GB buffer with 4 KB pages is 262,144 entries. The GPU
  writes those far faster than the CPU can, and the CPU doesn't block.
* **Reach.** Page tables live in VRAM, which the CPU may not even be able to see
  (Chapter 2's window problem).

There is also a CPU path, used when no GPU submission is possible — early boot,
or a VM whose page tables are in system memory. But the GPU path is the normal
one, and it's why `xe_migrate` shows up in a chapter about page tables.

---

## Idea 4: Build a shadow tree, then swap it in

This is the most important design idea in the chapter, and it's how Xe makes
binding safe.

A bind can fail halfway: out of memory for a new table, the buffer got evicted,
a userptr got invalidated. If you were editing the *live* page tables when that
happened, you'd be left with a half-updated tree the GPU is actively walking.
Disaster.

So Xe never edits the live tree. It works in three phases:

**Phase 1 — Stage.** Walk the tree and build the *new* shape in a parallel
"staging" set of pointers. Allocate any new tables needed. Compute every PTE
value. **The live tree is untouched**, and every entry that needs writing is
collected into a list.

**Phase 2 — Submit.** Hand that list to the GPU as a batch of store commands.
Get a fence back.

**Phase 3 — Commit.** *Only now*, flip the live pointers to the staged ones and
free the tables that got replaced.

If anything fails during phase 1, throw the staging away. Nothing changed.

Notice this is the same shape as Chapter 6's plan-then-commit, one level down.
Chapter 6 planned *which mappings* should exist; Chapter 7 stages *which page
table entries* should exist. Both refuse to touch live state until success is
certain.

---

## Idea 5: Unbinding is not the reverse of binding

A quick but important asymmetry.

To bind, you write PTEs and then the GPU can use them. Fine.

To **unbind**, writing zeroes to the PTEs is not enough — because the GPU caches
translations in its **TLB**. A cleared PTE with a stale TLB entry means the GPU
happily keeps reading memory you just freed.

So unbinding is: *write the PTEs* → *invalidate the TLB* → *wait for that to
complete* → *only then* is the memory safe to reuse.

That TLB invalidation is per-GT (Chapter 1: memory is per-tile, execution is
per-GT), it has to be requested from the GuC, and it is genuinely slow. It gets
its own chapter — Chapter 9.

---

## The picture

```
   a VMA:  [0x140000000, +2MB)  ->  buffer in VRAM
                        |
   PHASE 1: STAGE (nothing live is touched)
                        |
        walk down:  L4[0] -> L3[0] -> L2[5] -> L1[0]
                        |
        at L1: 2MB aligned? physically contiguous? YES
                -> stop here, emit ONE huge PTE
                        |
        collected: [ { table, offset 0, 1 entry, value 0x...893 } ]
                        |
   PHASE 2: SUBMIT
        build a batch buffer:
             MI_STORE_DATA_IMM  <addr of L1 table + 0>  0x...893
        submit to the GPU  ->  get a fence
                        |
   PHASE 3: COMMIT (fence signalled)
        live pointers  <-  staged pointers
        free any replaced tables
```

## Remember these three

1. **PPGTT is a 4-level tree; each level covers 512x more than the one below.**
   Huge pages mean stopping at level 1 (2 MB) or 2 (1 GB) instead of descending.
2. **The GPU writes its own page table entries**, as a normal submission, which
   makes ordering free and returns a fence.
3. **Stage, submit, commit.** The live tree is never edited in place.

---
---

# Chapter 7 — Beat B: The Code

One journey, with real numbers:

> *Bind a VMA at GPU address `0x140000000` (5 GB), 2 MB long, backed by
> contiguous VRAM at device address `0x640000000`.*

Eight steps. `//` comments are mine.
Files: `drivers/gpu/drm/xe/`. v7.3-rc1.

---

## Step 1 — The structures

**File: `xe_pt_types.h`** — one node of the tree:

```c
struct xe_pt {
	struct xe_ptw base;      // the generic walker node: has children[] and
				 // staging[] pointer arrays (Idea 4!)
	struct xe_bo *bo;        // <<< the actual 4KB page of memory holding
				 // this table's 512 entries. A page table IS
				 // a buffer object (Chapter 3).
	unsigned int level;      // 0 = leaf page table, 4 = root
	unsigned int num_live;   // how many entries are in use
	bool rebind;
	bool is_compact;         // 64K-page compact layout variant
};
```

Two things worth saying out loud:

* **A page table is an `xe_bo`.** It gets allocated, placed, and validated by
  everything in Chapters 2–4. That's what `XE_BO_FLAG_PAGETABLE` was for, and
  why Chapter 3's caching code had a special case for page tables.
* **`base` has both `children` and `staging` arrays.** Idea 4's shadow tree is
  built right into the node type.

The address slicing from Idea 1 — `xe_pt.c:46`:

```c
static const u64 xe_normal_pt_shifts[]  = {12, 21, 30, 39, 48};
static const u64 xe_compact_pt_shifts[] = {16, 21, 30, 39, 48};
//                                         ^^ 64K leaves instead of 4K

#define XE_PT_HIGHEST_LEVEL (ARRAY_SIZE(xe_normal_pt_shifts) - 1)   // = 4
```

Index at level L is `(addr >> shifts[L]) & 511`. For our address:

```
   0x140000000 >> 48 & 511 = 0      level 4 (root)
   0x140000000 >> 39 & 511 = 0      level 3
   0x140000000 >> 30 & 511 = 5      level 2   <- 5 GB / 1 GB per entry
   0x140000000 >> 21 & 511 = 0      level 1   <- we will STOP here
   0x140000000 >> 12 & 511 = 0      level 0   (not needed)
```

And the collected work list:

```c
struct xe_vm_pgtable_update {
	struct xe_bo *pt_bo;             // which page-table BO to write into
	u32 ofs;                         // first entry to write (in qwords)
	u32 qwords;                      // how many entries
	struct xe_pt *pt;
	struct xe_pt_entry *pt_entries;  // the values: { child pt, u64 pte }
	u32 flags;
};
```

**This struct is the whole handoff between phases.** Phase 1 fills a list of
these; phase 2 turns them into GPU stores.

---

## Step 2 — What a PTE actually looks like

**File: `xe_vm.c:1463`.** Compare with Chapter 5's two-bit GGTT entry.

```c
static u64 xelp_pte_encode_bo(struct xe_bo *bo, u64 bo_offset,
			      u16 pat_index, u32 pt_level)
{
	u64 pte;

	pte = xe_bo_addr(bo, bo_offset, XE_PAGE_SIZE);   // the physical address
							// (ch 4's helper)
	pte |= XE_PAGE_PRESENT | XE_PAGE_RW;             // bit 0, bit 1
	pte |= pte_encode_pat_index(pat_index, pt_level);// cache attributes
	pte |= pte_encode_ps(pt_level);                  // "I am a huge page"

	if (xe_bo_is_vram(bo) || xe_bo_is_stolen_devmem(bo))
		pte |= XE_PPGTT_PTE_DM;                  // bit 11: device memory
							 // (GGTT used bit 1!)
	return pte;
}
```

The two helpers, `xe_vm.c:1388` and `:1414`:

```c
static u64 pte_encode_pat_index(u16 pat_index, u32 pt_level)
{
	u64 pte = 0;
	// The cache index is up to 5 bits, SCATTERED across the entry:
	if (pat_index & BIT(0)) pte |= XE_PPGTT_PTE_PAT0;      // bit 3
	if (pat_index & BIT(1)) pte |= XE_PPGTT_PTE_PAT1;      // bit 4
	if (pat_index & BIT(2)) {
		if (pt_level)   pte |= XE_PPGTT_PDE_PDPE_PAT2; // bit 12 (non-leaf)
		else            pte |= XE_PPGTT_PTE_PAT2;      // bit 7  (leaf)
	}
	if (pat_index & BIT(3)) pte |= XELPG_PPGTT_PTE_PAT3;   // bit 62
	if (pat_index & BIT(4)) pte |= XE2_PPGTT_PTE_PAT4;     // bit 61
	return pte;
}

static u64 pte_encode_ps(u32 pt_level)
{
	if (pt_level == 1) return XE_PDE_PS_2M;     // bit 7 -> "this level-1
						    // entry IS the memory"
	if (pt_level == 2) return XE_PDPE_PS_1G;    // bit 7 -> same, 1 GB
	return 0;
}
```

Look at bit 2 of the PAT index: it lands at **bit 7 for a leaf but bit 12 for a
non-leaf** — because bit 7 is already taken by the huge-page flag at those
levels. That collision is why the function needs `pt_level` at all.

### Our actual values

Discrete Xe2, so `xe->pat.idx[XE_CACHE_WB] = 2` (`xe_pat.c:645`), binary `00010`
→ only `BIT(1)` set → `XE_PPGTT_PTE_PAT1` = bit 4.

**A normal 4 KB leaf PTE** for a VRAM page at `0x640000000`:

```
   physical address           0x0000000640000000
   XE_PAGE_PRESENT  bit 0   | 0x0000000000000001
   XE_PAGE_RW       bit 1   | 0x0000000000000002
   PAT1             bit 4   | 0x0000000000000010
   XE_PPGTT_PTE_DM  bit 11  | 0x0000000000000800
                              ------------------
                              0x0000000640000813
```

**Our 2 MB huge PTE** at level 1 — same, plus the page-size bit:

```
   XE_PDE_PS_2M     bit 7   | 0x0000000000000080
                              ------------------
                              0x0000000640000893   <<< what we will write
```

**A PDE** (non-leaf, pointing at a child table in VRAM at `0x500000000`).
Note `xe_vm.c:1425` picks a *different* cache index for non-leaf nodes —
uncached, because only the hardware walker reads them and only 2 bits are
available:

```c
static u64 xelp_pde_encode_bo(struct xe_bo *bo, u64 bo_offset)
{
	u64 pde = xe_bo_addr(bo, bo_offset, XE_PAGE_SIZE);
	pde |= XE_PAGE_PRESENT | XE_PAGE_RW;
	pde |= pde_encode_pat_index(pde_pat_index(bo));   // only PAT0/PAT1
	return pde;
}
```

With `XE_CACHE_NONE = 3` → both PAT bits → `0x000000050000001B`.

| | GGTT (ch 5) | PPGTT (here) |
|--|-------------|--------------|
| "device memory" bit | bit 1 | **bit 11** |
| read/write bit | none — always RW | **bit 1** |
| PAT bits | 2 (bits 52–53) | up to **5** (bits 3, 4, 7/12, 61, 62) |
| huge page bit | none | **bit 7** |
| our example | `0x0030000112345001` | `0x0000000640000893` |

---

## Step 3 — Phase 1 begins: the staging walk

**File: `xe_pt.c:777`**, `xe_pt_stage_bind()`. Set up the walk:

```c
	struct xe_pt_stage_bind_walk xe_walk = {
		.base = {
			.ops = &xe_pt_stage_bind_ops,      // the callback below
			.shifts = xe_normal_pt_shifts,     // {12,21,30,39,48}
			.max_level = XE_PT_HIGHEST_LEVEL,  // 4
			.staging = true,                   // <<< IDEA 4:
							   // walk the staging
							   // pointers, not live
		},
		.vm = vm,
		.tile = tile,
		.curs = &curs,                  // cursor over PHYSICAL memory
		.va_curs_start = xe_vma_start(vma),
		.vma = vma,
		.wupd.entries = entries,        // the output work list
	};
	struct xe_pt *pt = vm->pt_root[tile->id];    // start at this TILE's root
```

Then point the cursor at the physical memory (`:867`) — the same
`xe_res_cursor` Chapter 5 used:

```c
	if (!xe_vma_is_null(vma) && !range && !is_purged) {
		if (xe_vma_is_userptr(vma))
			xe_res_first_dma(...userptr.pages.dma_addr...);  // userptr
		else if (xe_bo_is_vram(bo) || xe_bo_is_stolen(bo))
			xe_res_first(bo->ttm.resource, ...);             // VRAM
		else
			xe_res_first_sg(xe_bo_sg(bo), ...);              // system
	} else if (!range) {
		curs.size = xe_vma_size(vma);   // NULL VMA: no physical memory
	}
```

Chapter 6's four kinds of mapping (Idea 4 there), as four cursor sources.
And `dma_offset` picks up Chapter 5's region base again:

```c
	xe_walk.default_vram_pte |= XE_PPGTT_PTE_DM;
	xe_walk.dma_offset = (bo && !is_purged) ?
			     vram_region_gpu_offset(bo->ttm.resource) : 0;
```

Then walk:

```c
walk_pt:
	ret = xe_pt_walk_range(&pt->base, pt->level,           // from the root
			       xe_vma_start(vma), xe_vma_end(vma),
			       &xe_walk.base);

	*num_entries = xe_walk.wupd.num_used_entries;   // how much work we made
```

---

## Step 4 — The decision at every level

**File: `xe_pt.c:559`**, `xe_pt_stage_bind_entry()`. Called once per entry, at
every level, on the way down. **This is the heart of the chapter.**

```c
	// ---- QUESTION 1: can I stop here? (Idea 2) ----
	if (level == 0 || xe_pt_huge_leaf_allowed(addr, next, level, xe_walk)) {

		// YES. Build a leaf entry and do not descend.
		bool is_vram = xe_res_is_vram(curs);

		pte = vm->pt_ops->pte_encode_vma(xe_res_dma(curs) +      // physical
						 xe_walk->dma_offset,    // + region base
						 xe_walk->vma,
						 pat_index, level);      // level -> PS bit
		pte |= is_vram ? xe_walk->default_vram_pte      // adds the DM bit
			       : xe_walk->default_system_pte;

		// Add the 64K hint if 16 consecutive 4K pages line up
		if (level == 0 && !xe_parent->is_compact) {
			if (xe_pt_is_pte_ps64K(addr, next, xe_walk))
				pte |= XE_PTE_PS64;
		}

		// Record the work. Nothing is written to hardware yet!
		ret = xe_pt_insert_entry(xe_walk, xe_parent, offset, NULL, pte);

		xe_res_next(curs, next - addr);     // advance the physical cursor
		*action = ACTION_CONTINUE;          // <<< DO NOT DESCEND
		return ret;
	}

	// ---- QUESTION 2: no, so descend. Do I need a child table? ----
	covers = xe_pt_covers(addr, next, level, &xe_walk->base);
	if (covers || !*child) {
		// Either there is no child table yet, or the new mapping fully
		// replaces the existing one - so allocate a fresh table.
		xe_child = xe_pt_create(xe_walk->vm, xe_walk->tile, level - 1,
					xe_vm_validation_exec(vm));
		//        ^^^ allocates a 4KB xe_bo. Chapters 2-4 run here.

		if (!covers)
			xe_pt_populate_empty(xe_walk->tile, xe_walk->vm, xe_child);
			// partial overlap: pre-fill with scratch/empty entries so
			// the untouched parts stay valid

		*child = &xe_child->base;    // <<< into the STAGING array

		// Compact 64K layout, if the whole 2MB region can use 64K pages
		if (... level == 1 && covers && xe_pt_scan_64K(addr, next, xe_walk)) {
			walk->shifts = xe_compact_pt_shifts;   // leaf shift 12 -> 16
			flags |= XE_PDE_64K;
			xe_child->is_compact = true;
		}

		// A PDE pointing at the child we just made
		pte = vm->pt_ops->pde_encode_bo(xe_child->bo, 0) | flags;
		ret = xe_pt_insert_entry(xe_walk, xe_parent, offset, xe_child, pte);
	}

	*action = ACTION_SUBTREE;            // <<< DESCEND one level
	return ret;
```

Two return paths, and they are Idea 1 vs Idea 2: `ACTION_CONTINUE` means *"this
entry is the answer"*; `ACTION_SUBTREE` means *"go deeper"*.

### The huge-page test, in full — `xe_pt.c:441`

```c
static bool xe_pt_hugepte_possible(u64 addr, u64 next, unsigned int level,
				   struct xe_pt_stage_bind_walk *xe_walk)
{
	if (level > MAX_HUGEPTE_LEVEL)
		return false;                    // only levels 1 and 2 can

	// CONDITION 1: does the VIRTUAL range exactly fill one entry?
	if (!xe_pt_covers(addr, next, level, &xe_walk->base))
		return false;

	if (xe_vma_is_null(xe_walk->vma) || ...)
		return true;                     // no physical memory to check

	// CONDITION 2: is the PHYSICAL memory contiguous across the whole thing?
	if (next - xe_walk->va_curs_start > xe_walk->curs->size)
		return false;
	//  ^ curs->size is how many bytes the current contiguous run has left.
	//    Too short -> the region is fragmented -> no huge page.

	// CONDITION 3: is the physical address itself aligned to the huge size?
	size = next - addr;
	dma  = addr - xe_walk->va_curs_start + xe_res_dma(xe_walk->curs);
	return IS_ALIGNED(dma, size);
}
```

Idea 2's three conditions, one per comment. **Condition 2 is where Chapter 2's
buddy allocator pays off**: a contiguous VRAM allocation gives a long
`curs->size`, so huge pages are available. Scattered system pages give
`curs->size == 4096` every time, so they never are.

### For our example

Our VMA is 2 MB, 2 MB-aligned virtually, and backed by contiguous VRAM at
`0x640000000` (which is 2 MB aligned). So:

* level 4, 3, 2 → not covered by one entry → `ACTION_SUBTREE`, allocate/descend
* **level 1 → all three conditions pass → one huge PTE, stop**

Result: the work list holds **one entry**:

```
   entries[0] = { pt_bo   = <the level-1 table's BO>,
                  ofs     = 0,                       // index 0 at level 1
                  qwords  = 1,                       // ONE entry
                  pt_entries[0].pte = 0x0000000640000893 }
```

One 64-bit write maps 2 MB. Without huge pages that would have been 512 writes
plus a new page table.

---

## Step 5 — Phase 2: hand the list to the GPU

**File: `xe_migrate.c:1742`**, `write_pgtable()`. This turns one work-list entry
into GPU commands.

```c
static void write_pgtable(struct xe_tile *tile, struct xe_bb *bb, u64 ppgtt_ofs,
			  const struct xe_vm_pgtable_update_op *pt_op,
			  const struct xe_vm_pgtable_update *update,
			  struct xe_migrate_pt_update *pt_update)
{
	u32 ofs = update->ofs, size = update->qwords;

	if (!ppgtt_ofs)
		ppgtt_ofs = xe_migrate_vram_ofs(tile_to_xe(tile),
						xe_bo_addr(update->pt_bo, 0,
							   XE_PAGE_SIZE), false);
	//  ^ the GPU needs an ADDRESS for the page table it is about to write.
	//    So the page table is itself mapped into the migrate VM. Yes, really.

	do {
		u64 addr = ppgtt_ofs + ofs * 8;     // 8 bytes per entry

		chunk = min(size, MAX_PTE_PER_SDI); // one command writes at most
						    // 0x1FE = 510 entries

		if (!(bb->len & 1))
			bb->cs[bb->len++] = MI_NOOP;   // keep the payload aligned

		// ---- THE COMMAND ----
		bb->cs[bb->len++] = MI_STORE_DATA_IMM | MI_SDI_NUM_QW(chunk);
		bb->cs[bb->len++] = lower_32_bits(addr);   // where to write
		bb->cs[bb->len++] = upper_32_bits(addr);

		// ---- THE PAYLOAD: the PTE values, written INTO the command ----
		if (pt_op->bind)
			ops->populate(pt_update, tile, NULL, bb->cs + bb->len,
				      ofs, chunk, update);
		else
			ops->clear(...);

		bb->len += chunk * 2;              // 2 dwords per 64-bit entry
		ofs  += chunk;
		size -= chunk;
	} while (size);
}
```

`MI_STORE_DATA_IMM` is Idea 3 made concrete — **"store this immediate data to
this address."** The PTE values are literally inlined into the command stream.

And `populate` is `xe_vm_populate_pgtable()` at `xe_pt.c:1095`:

```c
	for (i = 0; i < num_qwords; i++) {
		u32 idx = qword_ofs - update->ofs + i;

		if (map)
			xe_map_wr(tile_to_xe(tile), map, (qword_ofs + i) *
				  sizeof(u64), u64, ptes[idx].pte);   // CPU path
		else
			ptr[i] = ptes[idx].pte;                       // GPU path:
								      // into the
								      // batch buffer
	}
```

One function, two destinations. `map != NULL` → the CPU writes the page table
directly. `map == NULL` → the values go into the GPU command buffer. **That
`if` is the CPU-path / GPU-path fork from Idea 3.**

For our example the whole batch buffer is:

```
   MI_NOOP
   MI_STORE_DATA_IMM | MI_SDI_NUM_QW(1)
   lower_32(<addr of L1 table> + 0)
   upper_32(<addr of L1 table> + 0)
   0x40000893          // lower 32 bits of our PTE
   0x00000006          // upper 32 bits
```

Six dwords to map 2 MB.

---

## Step 6 — Phase 3: commit

**File: `xe_pt.c:1179`.** The fence has signalled; the entries are in memory.
Now make the staging tree the live tree:

```c
static void xe_pt_commit(struct xe_vma *vma,
			 struct xe_vm_pgtable_update *entries,
			 u32 num_entries, struct llist_head *deferred)
{
	for (i = 0; i < num_entries; i++) {
		struct xe_pt *pt = entries[i].pt;

		if (!pt->level)
			continue;             // leaf tables have no children

		pt_dir = as_xe_pt_dir(pt);
		for (j = 0; j < entries[i].qwords; j++) {
			struct xe_pt *oldpte = entries[i].pt_entries[j].pt;
			int j_ = j + entries[i].ofs;

			pt_dir->children[j_] = pt_dir->staging[j_];
			// ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ IDEA 4.
			//   One pointer assignment per entry. THAT is the
			//   "swap the shadow tree in" moment.

			xe_pt_destroy(oldpte, ..., deferred);
			// free whatever this entry used to point at - deferred,
			// because the GPU may still be walking it
		}
	}
}
```

**`children[j_] = staging[j_]`** is the entire commit. Everything before it was
preparation; everything about failure handling exists so that we only reach this
line when success is certain.

And the failure path, `xe_pt_abort_bind()` at `:1206`, does the opposite —
restores `staging` from `children` and frees the tables that were speculatively
allocated. Nothing observable ever happened.

---

## Step 7 — Who drives the three phases

**File: `xe_pt.c`**, three entry points that Chapter 8's `VM_BIND` will call in
order:

```c
int  xe_pt_update_ops_prepare(struct xe_tile *tile, struct xe_vma_ops *vops);  // :2468
struct dma_fence *
     xe_pt_update_ops_run(struct xe_tile *tile, struct xe_vma_ops *vops);      // :2704
void xe_pt_update_ops_fini(struct xe_tile *tile, struct xe_vma_ops *vops);     // :2882
void xe_pt_update_ops_abort(struct xe_tile *tile, struct xe_vma_ops *vops);    // :2908
```

| Function | Phase | Does |
|----------|-------|------|
| `..._prepare` | 1 | calls `xe_pt_stage_bind()` for every op, builds the work lists |
| `..._run` | 2 | submits the batch, returns a fence |
| `..._fini` | 3 | commits and frees on success |
| `..._abort` | — | throws the staging away on failure |

Note `_run` returns `struct dma_fence *`. Same signature shape as
`xe_migrate_copy()` in Chapter 4 — because it *is* the same machinery.

And one flag from `xe_pt_types.h` worth pointing at:

```c
struct xe_vm_pgtable_update_ops {
	...
	bool needs_invalidation;   // <<< IDEA 5. Set for unbinds. Chapter 9
				   //     acts on it.
};
```

---

## The journey, end to end

```
 Step 1  struct xe_pt                xe_pt_types.h     a tree node; base has
                                                       children[] + staging[]
         xe_normal_pt_shifts         xe_pt.c:46        {12,21,30,39,48}
 Step 2  xelp_pte_encode_bo()        xe_vm.c:1463      what a PTE looks like
         pte_encode_ps()             xe_vm.c:1414      the huge-page bit
 Step 3  xe_pt_stage_bind()          xe_pt.c:777       PHASE 1 begins
 Step 4  xe_pt_stage_bind_entry()    xe_pt.c:559       stop or descend?
         xe_pt_hugepte_possible()    xe_pt.c:441       Idea 2's 3 conditions
 Step 5  write_pgtable()             xe_migrate.c:1742 PHASE 2: MI_STORE_DATA_IMM
         xe_vm_populate_pgtable()    xe_pt.c:1095      CPU path vs GPU path
 Step 6  xe_pt_commit()              xe_pt.c:1179      PHASE 3: children = staging
 Step 7  xe_pt_update_ops_*()        xe_pt.c:2468+     the three phases, driven
```

## Try it on your machine

```bash
# every page-table entry Xe stages, with level and value
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vm_pgtable_update/enable
cat /sys/kernel/debug/tracing/trace_pipe

# and with CONFIG_DRM_XE_DEBUG_VM=y, xe_vm_dbg_print_entries()
# (xe_pt.c:1295) prints every staged entry before submission
```

## Checkpoint

1. Our 2 MB mapping produced **one** page table write. How many would it have
   needed with 4 KB pages, and what exactly made the difference?
2. `pte_encode_pat_index()` takes `pt_level`. Why does it need to know the level?
3. The GPU writes page tables — but page tables live in memory the GPU addresses
   through page tables. How is that not circular?
4. What single line of code *is* the commit, and why is everything else in the
   chapter arranged around it?
5. Why is clearing a PTE not enough to unbind a page?

<details>
<summary>answers</summary>

1. 512 entries, plus allocating a whole new level-0 table to hold them (and a
   PDE pointing at it). The difference was that all three of
   `xe_pt_hugepte_possible()`'s conditions held: the virtual range exactly filled
   one level-1 entry, the physical VRAM was contiguous across all 2 MB, and the
   physical address was 2 MB aligned.
2. Because bit 2 of the cache index lands in different places: bit 7 for a leaf
   (`XE_PPGTT_PTE_PAT2`) but bit 12 for a non-leaf (`XE_PPGTT_PDE_PDPE_PAT2`).
   At levels 1 and 2 bit 7 is already the huge-page flag, so the PAT bit has to
   move.
3. The page tables are themselves mapped into a special VM belonging to
   `xe_migrate` — see `xe_migrate_vram_ofs()` in `write_pgtable()`. That VM's own
   page tables are set up once, by the CPU, at driver load. The bootstrap is
   broken by hand; everything after it uses the GPU.
4. `pt_dir->children[j_] = pt_dir->staging[j_]` in `xe_pt_commit()`. Because the
   live tree must never be half-updated while the GPU walks it, every earlier
   phase exists to guarantee that this assignment cannot fail and is only reached
   once the new entries are genuinely in memory.
5. The GPU caches translations in its TLB. A zeroed PTE with a live TLB entry
   still resolves, so the GPU would keep reading memory that has been freed.
   The TLB must be invalidated and that invalidation confirmed complete —
   Chapter 9.

</details>
