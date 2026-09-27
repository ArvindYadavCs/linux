# Chapter 5 — Two Address Spaces

> Beat A: the story.

Four chapters and we have never once given a buffer a **GPU address**.
Everything so far has been about *physical* places — which warehouse, which
page. Now we start addressing.

Four ideas.

---

## Idea 1: Physical places need name tags

The GPU does not issue physical addresses. Like a CPU, it issues **virtual
addresses**, and a page table translates them.

So on top of everything in Chapters 2–4, there is a second, independent
question for every buffer:

> *What address does the GPU use to reach this?*

These two questions are genuinely separate:

| Question | Answered by | Chapters |
|----------|-------------|----------|
| where do the bytes physically live? | placement, resource managers | 2–4 |
| what address does the GPU see? | page tables | 5–8 |

And they change independently. A buffer can be migrated (new physical location,
same GPU address — the page table just gets rewritten). A buffer can be
unmapped and remapped elsewhere (same physical location, new GPU address).

---

## Idea 2: There are two address books, for two different customers

Here is the part that confuses people. Intel GPUs have **two completely
separate translation mechanisms**, and Xe uses both.

### GGTT — the building's public directory

* **One per tile.**
* **Global** — there is only one, shared by everybody in that tile.
* **Flat** — a single array of entries, no tree. Entry *N* covers page *N*.
  Index the array, get the physical address. Done.
* **Small** — about 4 GB of address space.
* **Kernel only.** Userspace never sees a GGTT address.

Think of it as the receptionist's directory nailed to the wall in the lobby.
One copy, everybody reads it, and it is a plain list.

### PPGTT — a customer's private address book

* **One per VM**, and a VM belongs to one client.
* **Private** — client A's address 0x1000 and client B's address 0x1000 are
  unrelated. This is what isolates GPU processes from each other.
* **Multi-level tree** — up to 4 levels, exactly like a CPU page table.
* **Huge** — 256 TB of address space on most platforms (48 address bits).
* **This is what userspace uses** for all its buffers.

> **The one-line summary:** GGTT is small, flat, global and for the kernel.
> PPGTT is huge, tree-shaped, per-client and for userspace.

Chapter 5 is GGTT. Chapters 6–8 are PPGTT.

---

## Idea 3: What actually lives in GGTT? Surprisingly little — but all critical

If PPGTT does all the real work, why keep GGTT at all?

Because some hardware **cannot** use PPGTT. Three customers:

**1. The GuC.** Chapter 1's foreman. It's a small microcontroller, and it can
only fetch things through GGTT. So every structure the driver shares with the
GuC — the firmware image itself, the CT mailbox, the log buffer, the ADS
configuration blob — must have a GGTT address.

**2. The display engine.** It scans out a framebuffer continuously, 60 times a
second, and it is not part of any client's VM. It reads through GGTT.

**3. Hardware contexts and rings.** When the GuC tells an engine "run this
context," it hands over a **GGTT address** for that context's state. So every
exec queue's LRC and ring buffer needs a GGTT slot, even though the work inside
it uses PPGTT addresses.

That third one is the important link: **GGTT is how work gets *started*; PPGTT
is what the work *uses* once running.**

There's also a bootstrapping reason. At driver load there are no VMs and no page
tables yet, but the driver already needs to talk to firmware. GGTT is a flat
array in hardware-provided memory — it works from the first instruction.

---

## Idea 4: The flat table lives in a hardware window

GGTT's data structure is not something the driver allocates. It's a region the
hardware provides, called **GSM**, that the driver maps through MMIO.

The consequence is delightfully simple: **writing a page table entry is a single
64-bit store to an iomem address.**

```
   GPU address 0x201000
        |  divide by page size (4 KB) -> index 0x201
        v
   gsm[0x201] = physical address | flags      <- one writeq(), done
```

No tree walk. No allocation. No PTE-writing GPU jobs (those come in Chapter 7,
for PPGTT). Just array indexing.

Two practical details fall out:

**Someone has to hand out slots.** The address space is 4 GB and callers just
say "give me 2 MB somewhere." Xe uses the kernel's generic range allocator
(`drm_mm`) to track which parts of the 4 GB are in use.

**The ends are off-limits.** The GuC's hardware redirects accesses below a
certain offset (the WOPCM region) and above a fixed ceiling to somewhere other
than the GGTT. Rather than checking every object for "will the GuC touch this?",
Xe simply **excludes those ranges from the allocator entirely**. Costs about
20–25 MB of wasted address space, and in exchange every allocation is
automatically GuC-safe. A nice trade.

**And there's a scratch page.** Unused entries don't point at nothing — they
point at one harmless scratch page. So a stray GPU access to an unmapped GGTT
address reads garbage instead of faulting the machine.

---

## The picture

```
   ONE PER TILE                              ONE PER CLIENT
   +----------------------------+            +---------------------------+
   |  GGTT  (flat, ~4 GB)       |            |  PPGTT (tree, 256 TB)     |
   |                            |            |                           |
   |  gsm[0] -> scratch page    |            |     level 4 root          |
   |  gsm[1] -> scratch page    |            |      /        \           |
   |  gsm[2] -> GuC firmware    |            |   lvl 3      lvl 3        |
   |  gsm[3] -> CT mailbox      |            |    /            \         |
   |  gsm[4] -> an LRC          |            |  lvl 2 ...    lvl 2 ...   |
   |  gsm[5] -> a framebuffer   |            |                           |
   |  ...                       |            |  userspace buffers        |
   +----------------------------+            +---------------------------+
      users: GuC, display,                      users: userspace, via
             LRCs/rings                                VM_BIND
      written by: a single writeq()            written by: a GPU job (ch 7)
```

## Remember these three

1. **Physical location and GPU address are separate questions.** Chapters 2–4
   answered the first; 5–8 answer the second.
2. **GGTT: one per tile, flat, ~4 GB, kernel only** — for the GuC, the display
   engine, and hardware contexts. **PPGTT: one per client, tree, 256 TB** — for
   userspace.
3. **GGTT starts the work; PPGTT is what the work uses.**

---
---

# Chapter 5 — Beat B: The Code

One journey again:

> *The driver creates a buffer for the GuC. Give it a GGTT address.*

Six steps. `//` comments are mine.
All files are `drivers/gpu/drm/xe/`. Line numbers are v7.3-rc1.

---

## Step 1 — The two structures

**File: `xe_ggtt.c:111`** — the whole GGTT, per tile:

```c
struct xe_ggtt {
	struct xe_tile *tile;        // which tile owns this (ch 1: one per tile)

	u64 start;                   // first usable address (just above WOPCM)
	u64 size;                    // how much is usable

	struct xe_bo *scratch;       // the harmless page every unused entry
				     // points at

	struct mutex lock;           // one lock for the whole table

	u64 __iomem *gsm;            // <<< THE TABLE ITSELF.
				     // An iomem pointer into hardware-provided
				     // memory. gsm[i] IS page table entry i.
				     // Writing an entry = one store here.

	const struct xe_ggtt_pt_ops *pt_ops;  // per-platform PTE encoding

	struct drm_mm mm;            // <<< THE SLOT ALLOCATOR.
				     // Generic kernel range allocator, tracking
				     // which parts of the address space are used

	struct workqueue_struct *wq; // for deferred node removal
};
```

Two fields carry the whole design: **`gsm` is the table** (a flat array), and
**`mm` is who hands out slots in it**.

**File: `xe_ggtt.c:79`** — one allocation inside the table:

```c
struct xe_ggtt_node {
	struct xe_ggtt *ggtt;            // back pointer
	struct drm_mm_node base;         // the reserved range: start + size.
					 // This is what drm_mm hands back.
	struct work_struct delayed_removal_work;
	bool invalidate_on_remove;
};
```

And remember Chapter 3's `xe_bo`:

```c
struct xe_ggtt_node *ggtt_node[XE_MAX_TILES_PER_DEVICE];
```

**One slot per tile.** A buffer the GuC on both tiles must see needs a GGTT
address in *both* tiles' tables — different addresses, same buffer. That array
is why.

---

## Step 2 — Setting the table up

**File: `xe_ggtt.c:391`**, `xe_ggtt_init_early()`. This ran during probe in
Chapter 1, *before* the VRAM manager.

```c
	if (!IS_SRIOV_VF(xe)) {
		if (GRAPHICS_VERx100(xe) >= 1250)
			gsm_size = SZ_8M;          // 8 MB of table...
		else
			gsm_size = probe_gsm_size(pdev);   // older: ask PCI config

		ggtt_start = wopcm;               // <<< start ABOVE the WOPCM region.
						  // Everything below is GuC-unsafe,
						  // so we never allocate there.

		ggtt_size = (gsm_size / 8) * (u64)XE_PAGE_SIZE - ggtt_start;
		//           ^^^^^^^^^^^^   ^^^^^^^^^^^^
		// 8 bytes per entry, so 8 MB of table = 1M entries.
		// 1M entries x 4 KB per page = 4 GB of address space.
		// (Then subtract the part we're skipping at the bottom.)
	}

	ggtt->gsm = ggtt->tile->mmio.regs + SZ_8M;
	//          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
	// The table is just an offset inside this tile's MMIO window.
	// No allocation - the hardware provides it.

	if (ggtt_size + ggtt_start > GUC_GGTT_TOP)
		ggtt_size = GUC_GGTT_TOP - ggtt_start;
	//  ^^^ and clamp the TOP too. GUC_GGTT_TOP is 0xFEE00000 -
	//      above that, GuC accesses get redirected elsewhere.
```

So the usable range is `[WOPCM, GUC_GGTT_TOP)`, and Idea 4's trade is these two
clamps. Everything the allocator hands out afterwards is automatically
GuC-reachable.

Then the platform's PTE encoder is chosen, and `drm_mm` is initialized over that
range.

---

## Step 3 — Writing one entry

**File: `xe_ggtt.c:238`.** This is the entire mechanism:

```c
static void xe_ggtt_set_pte(struct xe_ggtt *ggtt, u64 addr, u64 pte)
{
	xe_tile_assert(ggtt->tile, !(addr & XE_PTE_MASK));       // must be page-aligned
	xe_tile_assert(ggtt->tile, addr < ggtt->start + ggtt->size);  // in range

	writeq(pte, &ggtt->gsm[addr >> XE_PTE_SHIFT]);
	//                     ^^^^^^^^^^^^^^^^^^^^
	// GPU address >> 12  ==  divide by 4 KB  ==  the array index.
	// One 64-bit store to iomem. That's a GGTT page table update.
}
```

Compare this with what Chapter 7 will need for PPGTT (walking a 4-level tree and
submitting a GPU job to write the entries) and you see why GGTT survives: it is
*trivially* cheap.

What goes *in* an entry — `xe_ggtt.c:145`:

```c
static u64 xelp_ggtt_pte_flags(struct xe_bo *bo, u16 pat_index)
{
	u64 pte = XE_PAGE_PRESENT;             // bit 0: "this entry is valid"

	if (xe_bo_is_vram(bo) || xe_bo_is_stolen_devmem(bo))
		pte |= XE_GGTT_PTE_DM;         // bit 1: "Device Memory" -
					       // the address below is VRAM,
					       // not system RAM
	return pte;
}
```

Two bits. A GGTT entry is just *physical address | present | is-it-VRAM*
(newer platforms add two cache-attribute bits). Compare that with a PPGTT entry
in Chapter 7 — this is a *much* simpler table.

---

## Step 4 — Getting a slot: `xe_ggtt_insert_bo()`

**File: `xe_ggtt.c:787`**, `__xe_ggtt_insert_bo_at()`. This is the main event.

```c
static int __xe_ggtt_insert_bo_at(struct xe_ggtt *ggtt, struct xe_bo *bo,
				  u64 start, u64 end, struct drm_exec *exec)
{
	u64 alignment = bo->min_align > 0 ? bo->min_align : XE_PAGE_SIZE;
	u8 tile_id = ggtt->tile->id;

	if (xe_bo_is_vram(bo) && ggtt->flags & XE_GGTT_FLAGS_64K)
		alignment = SZ_64K;        // some hardware needs 64K alignment
					   // for VRAM (ch 3's NEEDS_64K again)

	if (XE_WARN_ON(bo->ggtt_node[tile_id])) {
		return 0;                  // already has a slot in THIS tile's
					   // table - nothing to do
	}

	err = xe_bo_validate(bo, NULL, false, exec);
	if (err)
		return err;
	// ^^^ CHAPTER 4! Before we can write a page table entry we must know
	//     the buffer's physical location. So validate it first.
	//     Address mapping depends on placement - the order is not optional.

	bo->ggtt_node[tile_id] = ggtt_node_init(ggtt);   // allocate the node struct

	mutex_lock(&ggtt->lock);

	// Callers pass absolute GPU addresses, but drm_mm tracks offsets from 0.
	// Translate:
	if (start >= ggtt->start)
		start -= ggtt->start;
	else
		start = 0;
	if (end >= ggtt->start)
		end -= ggtt->start;
	else
		end = 0;

	// Ask the range allocator for a free hole of the right size.
	err = drm_mm_insert_node_in_range(&ggtt->mm,
					  &bo->ggtt_node[tile_id]->base,
					  xe_bo_size(bo),      // how much
					  alignment,           // aligned how
					  0,
					  start, end,          // within this window
					  0);
	if (err) {
		ggtt_node_fini(bo->ggtt_node[tile_id]);   // no room in 4 GB
		bo->ggtt_node[tile_id] = NULL;
	} else {
		// Got an address range. Now fill in the entries.
		u16 cache_mode = bo->flags & XE_BO_FLAG_NEEDS_UC ? XE_CACHE_NONE
								 : XE_CACHE_WB;
		u16 pat_index = xe_cache_pat_idx(tile_to_xe(ggtt->tile), cache_mode);
		u64 pte = ggtt->pt_ops->pte_encode_flags(bo, pat_index);  // the flags
		xe_ggtt_map_bo(ggtt, bo->ggtt_node[tile_id], bo, pte);    // Step 5
	}
	mutex_unlock(&ggtt->lock);

	if (!err && bo->flags & XE_BO_FLAG_GGTT_INVALIDATE)
		xe_ggtt_invalidate(ggtt);       // Step 6
	...
}
```

Three things to take from this:

* **`xe_bo_validate()` comes first.** You cannot write a page table entry until
  you know what physical address to put in it. Chapters 2–4 are a prerequisite
  for Chapter 5, in code as well as in the course.
* **`drm_mm_insert_node_in_range()`** is the slot allocator doing its one job.
  `xe_ggtt_insert_bo()` at `:884` is just this with the window `[0, U64_MAX]` —
  "anywhere will do."
* **One mutex for the whole table.** GGTT updates are rare and kernel-only, so a
  single lock is fine. PPGTT could never do this.

---

## Step 5 — Filling the entries: `xe_ggtt_map_bo()`

**File: `xe_ggtt.c:681`.** We have an address range and the buffer's physical
pages. Walk both in lockstep.

```c
static void xe_ggtt_map_bo(struct xe_ggtt *ggtt, struct xe_ggtt_node *node,
			   struct xe_bo *bo, u64 pte)
{
	struct xe_res_cursor cur;

	start = xe_ggtt_node_addr(node);        // the GPU address we were given
	end = start + xe_bo_size(bo);

	if (!xe_bo_is_vram(bo) && !xe_bo_is_stolen(bo)) {
		// ---- SYSTEM MEMORY ----
		// The physical addresses come from the page list's DMA addresses
		// (Chapter 3's ttm_tt). Walk them one 4 KB page at a time.
		for (xe_res_first_sg(xe_bo_sg(bo), 0, xe_bo_size(bo), &cur);
		     cur.remaining; xe_res_next(&cur, XE_PAGE_SIZE))
			ggtt->pt_ops->ggtt_set_pte(ggtt,
						   end - cur.remaining, // GPU addr
						   pte | xe_res_dma(&cur));
						   //     ^^^^^^^^^^^^^^^ the DMA
						   // address of this page

	} else {
		// ---- VRAM (or stolen) ----
		pte |= vram_region_gpu_offset(bo->ttm.resource);
		// ^ VRAM addresses are relative to the region; add its base

		for (xe_res_first(bo->ttm.resource, 0, xe_bo_size(bo), &cur);
		     cur.remaining; xe_res_next(&cur, XE_PAGE_SIZE))
			ggtt->pt_ops->ggtt_set_pte(ggtt,
						   end - cur.remaining,
						   pte + cur.start);
						   //    ^^^^^^^^^ offset within
						   // the VRAM resource
	}
}
```

Two branches, and they are **exactly Chapter 2's asymmetry**:

| | where physical addresses come from | Chapter 2 said |
|--|-----------------------------------|----------------|
| system memory | the `ttm_tt` page list's DMA addresses | *"a TT resource says only how much"* |
| VRAM | the `ttm_resource`'s own offsets | *"a VRAM resource says where"* |

`xe_res_cursor` is the helper that hides the difference — it iterates either a
scatter-gather list or a list of buddy blocks with the same three calls
(`xe_res_first*`, `xe_res_next`, `xe_res_dma`).

---

## Step 6 — Making the hardware notice

**File: `xe_ggtt.c:577`.**

```c
static void xe_ggtt_invalidate(struct xe_ggtt *ggtt)
{
	// A read of any register flushes posted writes out of the PCI write
	// buffer, so our writeq()s have actually landed before we invalidate.
	xe_mmio_read32(xe_root_tile_mmio(xe), VF_CAP_REG);

	/* Each GT in a tile has its own TLB to cache GGTT lookups */
	ggtt_invalidate_gt_tlb(ggtt->tile->primary_gt);
	ggtt_invalidate_gt_tlb(ggtt->tile->media_gt);
}
```

Small function, but note what it proves: **the table belongs to the tile, and
the caches belong to the GTs.** Chapter 1's split ("memory is per-tile,
execution is per-GT") shows up here as *one table, two TLBs to invalidate*.

Also note this is *not* done on every insert — only when
`XE_BO_FLAG_GGTT_INVALIDATE` is set. At driver load, before any engine is
running, there is nothing cached to invalidate, so most setup skips it.

---

## The journey, end to end

```
 Step 1  struct xe_ggtt              xe_ggtt.c:111   gsm = the table, mm = the allocator
         struct xe_ggtt_node         xe_ggtt.c:79    one reserved range
 Step 2  xe_ggtt_init_early()        xe_ggtt.c:391   map GSM, clamp both ends
 Step 3  xe_ggtt_set_pte()           xe_ggtt.c:238   one writeq() = one PTE
         xelp_ggtt_pte_flags()       xe_ggtt.c:145   present + is-it-VRAM
 Step 4  __xe_ggtt_insert_bo_at()    xe_ggtt.c:787   validate, then drm_mm
 Step 5  xe_ggtt_map_bo()            xe_ggtt.c:681   walk pages, write entries
 Step 6  xe_ggtt_invalidate()        xe_ggtt.c:577   flush both GTs' TLBs
```

## Who actually asks for GGTT slots

Grep the tree and Idea 3's three customers appear exactly as promised:

```bash
git grep -l 'XE_BO_FLAG_GGTT' drivers/gpu/drm/xe | grep -v 'xe_bo\|xe_ggtt'
```

| File | What it needs a GGTT address for |
|------|---------------------------------|
| `xe_guc.c`, `xe_guc_ct.c`, `xe_guc_ads.c`, `xe_guc_log.c` | everything shared with the foreman |
| `xe_lrc.c` | **every exec queue's context and ring** |
| `xe_hw_engine.c` | the hardware status page |
| `xe_huc.c`, `xe_gsc.c` | other firmware images |
| `xe_migrate.c` | Chapter 4's copy engine batch buffers |

`xe_lrc.c` is the one to notice. **Every single exec queue needs a GGTT slot** —
which is the concrete form of "GGTT starts the work; PPGTT is what the work
uses". Chapter 13 comes back to it.

## And a first look at the other side

For contrast, PPGTT's root — `xe_vm.c:1735`, inside `xe_vm_create()`:

```c
	vm->pt_root[id] = xe_pt_create(vm, tile, xe->info.vm_max_level, exec);
	//  ^^^^^^^^^^^                          ^^^^^^^^^^^^^^^^^^^^
	//  one tree per tile                    4 on every current platform
```

and its size — `xe_vm.c:1643`:

```c
	vm->size = 1ull << xe->info.va_bits;   // 48 bits -> 256 TB
					       // (57 on Xe3: 128 PB)
```

Set those beside GGTT's ~4 GB flat array and the division of labour is obvious.
Chapters 6–8 build this side.

## Checkpoint

1. Why does `__xe_ggtt_insert_bo_at()` call `xe_bo_validate()` before allocating
   an address?
2. `xe_bo` has an *array* of `ggtt_node`. Why not a single pointer?
3. 8 MB of GSM gives roughly 4 GB of address space. Work out why.
4. What are the two branches in `xe_ggtt_map_bo()`, and which earlier chapter
   predicted them?
5. `xe_ggtt_invalidate()` invalidates two TLBs for one table. Why two?

<details>
<summary>answers</summary>

1. A page table entry contains a *physical* address. Until the buffer has been
   validated you don't know where it physically is — and it might not be backed
   at all. Placement must be settled before addressing.
2. GGTT is per tile. A buffer that must be reachable by hardware on two tiles
   needs an entry in each tile's table, at two independent addresses. One slot
   per tile, hence the array.
3. Each entry is 8 bytes, so 8 MB holds 1M entries. Each entry maps one 4 KB
   page, so 1M × 4 KB = 4 GB. (Xe then clamps off the bottom, below WOPCM, and
   the top, above `GUC_GGTT_TOP`.)
4. System memory takes physical addresses from the `ttm_tt` page list's DMA
   addresses; VRAM takes them from the `ttm_resource` itself plus the region's
   base offset. Chapter 2 predicted it: a TT resource says only *how much*, a
   VRAM resource says *where*.
5. The table belongs to the tile, but each GT in that tile has its own TLB
   caching lookups from it — the primary GT and the media GT. Chapter 1's
   "memory is per-tile, execution is per-GT" showing through.

</details>
