# Chapter 6 — The VM and the VMA

> Beat A: the story.

Chapter 5 gave the kernel its address book (GGTT). Now userspace gets one.

Five ideas.

---

## Idea 1: A VM is a private address space; a VMA is one mapping in it

Two words, and the whole chapter hangs off them.

A **VM** is one client's private GPU address space. 256 TB of nothing, at
first — no buffers visible anywhere.

A **VMA** is one entry in it. It says exactly one thing:

> *"GPU addresses `0x10000`–`0x1FFFF` mean: buffer *X*, starting at offset
> `0x2000` inside it."*

So a VMA is a little three-part record:

```
   [ address, length ]  ->  [ which buffer, what offset inside it ]
```

Userspace creates and destroys these with the **`VM_BIND`** ioctl. That's the
whole userspace-facing model:

* `GEM_CREATE` → make a buffer (Chapter 3)
* `VM_BIND` → give it an address (this chapter)
* `EXEC` → run work that uses that address (Act II)

Note what a VMA is *not*: it is not a page table. A VMA is the *intent* —
bookkeeping in driver memory. Turning it into real page table entries the
hardware reads is Chapter 7. Keeping those two apart matters, because they can
disagree: a VMA can exist while its page tables are stale (remember Chapter 4's
"mark the mappings and move on").

---

## Idea 2: The bookkeeping goes both ways

You need to answer two different questions fast, so there are two data
structures over the same set of VMAs.

**Question A: "GPU address 0x18000 just faulted — which mapping covers it?"**
Answered by the VM's **interval tree**, sorted by address. Log-time lookup.

**Question B: "I'm about to move this buffer — who maps it?"**
Answered by a **per-buffer list** of the VMAs that point at it.

```
    the VM                                  a buffer
    +-------------------+                   +--------------------+
    | interval tree     |                   |  list of mappings  |
    |   VMA @ 0x00000   |<----------------->|    -> VMA @ 0x00000|
    |   VMA @ 0x10000   |                   |    -> VMA @ 0x40000|
    |   VMA @ 0x40000   |<----------------->|                    |
    +-------------------+                   +--------------------+
      "who covers 0x18000?"                  "who maps me?"
```

That per-buffer list is not new to you. It is exactly the list:

* Chapter 3's destructor asserted was **empty** before freeing a buffer,
* Chapter 4's migration **walked** to invalidate every mapping.

Now you know where it comes from.

One subtlety: a buffer can be mapped into several VMs, and several times into
the *same* VM. So between a buffer and a VM there's a little middleman object
holding "this buffer, as seen by this VM" — it owns the list of that VM's
mappings of that buffer, and it is what the evicted-list and extobj bookkeeping
hang off.

---

## Idea 3: The hard part is overlap

Here's the real work of this chapter, and it is *all* of the difficulty.

Userspace is allowed to just say **"map buffer B at 0x10000, length 0x10000"**
without telling you what's already there. If something already covers part of
that range, the driver must sort it out.

There are only a handful of cases, but they must all be right.

Suppose `0x00000`–`0x2FFFF` is already mapped to buffer A, and userspace now
maps buffer B over `0x10000`–`0x1FFFF`:

```
  before:   |<--------------- A --------------->|
            0x00000                        0x2FFFF

  request:            |<---- B ---->|
                      0x10000   0x1FFFF

  after:    |<- A ->| |<---- B ---->| |<- A ->|
            0x00000    0x10000         0x20000
             head          new          tail
```

The old mapping has to be **split into two**, with the new one between them.
And the tail piece must point at the *correct offset* inside A — it is no longer
the start of A.

This algebra is shared DRM code (`drm_gpuvm`), and it hands the driver back a
**list of operations** in one of three flavours:

| Operation | Meaning |
|-----------|---------|
| **MAP** | create a new mapping |
| **UNMAP** | destroy an existing mapping entirely |
| **REMAP** | destroy one mapping, and recreate up to two pieces of it (`prev` and `next`) |

Our example produces exactly two operations: a **REMAP** (drop old A-mapping,
recreate head and tail) then a **MAP** (the new B mapping).

The crucial design point:

> **Nothing is modified while the operation list is built.** The list is a
> *plan*. The driver inspects it, allocates everything it will need, and only
> then commits.

That's what makes `VM_BIND` able to fail cleanly. If allocation fails halfway
through planning, nothing has changed yet.

---

## Idea 4: Four kinds of mapping

Not every mapping points at a buffer. Four flavours exist, and they explain a
lot of branching in the code.

**1. Buffer-backed.** The normal case. Points at a GEM object.

**2. userptr.** Points at a range of the *calling process's own memory* —
plain `malloc`'d pages, never a GEM object. The driver pins nothing; instead it
registers for notifications so that if the process unmaps or the kernel moves
those pages, the mapping is invalidated. (Chapter 10.)

**3. NULL / sparse.** Points at **nothing at all**. Reads return zero, writes
are discarded. Sounds useless; it's essential — sparse resources in Vulkan and
D3D are exactly "a huge address range, most of which is unbacked." Cheap,
because no memory is involved.

**4. CPU-address-mirror.** A range where GPU addresses are deliberately made
*identical* to CPU addresses, so the same pointer works on both sides. This is
shared virtual memory, and the mappings inside it are built on demand by page
faults rather than by `VM_BIND`. (Chapter 10.)

---

## Idea 5: A VM has a personality

Chosen at creation time, and it changes how everything else behaves.

**Normal mode.** Every buffer a submission touches must be resident and mapped
*before* the work runs. The kernel validates the whole working set up front.
Simple, predictable, and the GPU never faults.

**LR mode** (Long Running). For compute work that may run for minutes and has
no natural end point. The kernel can't wait for such work to finish before
moving memory, so it needs the ability to **preempt** it instead. That's what
preempt fences are for (Chapter 18).

**Fault mode.** The GPU is allowed to touch unmapped addresses and take a
**page fault**, which the driver services by binding memory on demand — exactly
like CPU demand paging. Requires LR mode.

And one option worth knowing: **scratch page**. Instead of faulting on an
unmapped access, point every unmapped address at a single harmless page. The
same trick GGTT used in Chapter 5, applied to PPGTT — it turns a crash into a
garbage read, which is much kinder while bringing up new userspace.

---

## The picture

```
 userspace: "map buffer B at 0x10000, len 0x10000"
                        |
                        v
        +-----------------------------------+
        |  drm_gpuvm works out the algebra  |   <-- nothing changed yet
        |  -> [ REMAP(old A), MAP(new B) ]  |
        +-----------------------------------+
                        |
                        v
        +-----------------------------------+
        |  Xe allocates the VMAs it needs   |   <-- can still fail cleanly
        +-----------------------------------+
                        |
                        v
        +-----------------------------------+
        |  commit: update the interval tree |   <-- now it's real
        |  and each buffer's mapping list   |
        +-----------------------------------+
                        |
                        v
             page tables  (Chapter 7)
```

## Remember these three

1. **A VMA is `[address, length] -> [buffer, offset]`.** It is intent, not page
   tables.
2. **Two structures, two questions:** an interval tree per VM ("who covers this
   address?"), a list per buffer ("who maps me?").
3. **Overlap is resolved by building a plan first** — MAP / UNMAP / REMAP — and
   committing only once everything is allocated.

---
---

# Chapter 6 — Beat B: The Code

One journey, with real numbers:

> *Buffer A is already mapped over `0x00000`–`0x2FFFF`.*
> *Userspace now calls `VM_BIND` to map buffer B over `0x10000`–`0x1FFFF`.*

Seven steps. `//` comments are mine.
Files: `drivers/gpu/drm/xe/` and `drivers/gpu/drm/drm_gpuvm.c`. v7.3-rc1.

---

## Step 1 — The two structures

Same embedding trick as Chapter 3. **`xe_vm` wraps `drm_gpuvm`; `xe_vma` wraps
`drm_gpuva`.**

**File: `include/drm/drm_gpuvm.h`** — one mapping, the generic part:

```c
struct drm_gpuva {
	struct drm_gpuvm *vm;            // which VM owns this mapping
	struct drm_gpuvm_bo *vm_bo;      // the "this buffer, as seen by this VM"
					 // middleman from Idea 2
	enum drm_gpuva_flags flags;

	struct {
		u64 addr;                // <<< GPU address
		u64 range;               // <<< length
	} va;

	struct {
		u64 offset;              // <<< offset inside the buffer
		struct drm_gem_object *obj;   // <<< which buffer
		struct list_head entry;  // link in THAT BUFFER's list of mappings
					 // (Idea 2, question B)
	} gem;

	struct {
		struct rb_node node;     // node in the VM's INTERVAL TREE
		struct list_head entry;  // ...and in a flat ordered list
		u64 __subtree_last;      // interval-tree augmentation
	} rb;
};
```

Read the three sub-structs as Idea 1's three parts: `va` is *where*, `gem` is
*what*, `rb` is *how we find it fast*.

**File: `xe_vm_types.h:112`** — Xe's additions:

```c
struct xe_vma {
	struct drm_gpuva gpuva;          // <<< the generic mapping, embedded

	union {
		struct list_head rebind;   // "my page tables are stale" list
		struct list_head destroy;
	} combined_links;

	u8 tile_invalidated;             // which tiles' TLBs need flushing
	u8 tile_mask;                    // which tiles should have this bound
	u8 tile_present;                 // which tiles DO have it bound now
	u8 tile_staged;

	struct xe_vma_mem_attr attr;     // PAT index, preferred location, ...
};
```

Those three `tile_*` bytes are the honest admission that **a VMA is intent, not
page tables** (Idea 1). `tile_mask` is what *should* be bound; `tile_present` is
what *is*. When they differ, work is pending.

And the VM side — **`xe_vm_types.h:209`**, trimmed:

```c
struct xe_vm {
	struct drm_gpuvm gpuvm;          // <<< the interval tree lives in here

	struct xe_device *xe;
	struct xe_exec_queue *q[XE_MAX_TILES_PER_DEVICE];  // queues used to
							   // run bind jobs
	struct ttm_lru_bulk_move lru_bulk_move;  // ch 4's bulk LRU group
	u64 size;                                // 1 << va_bits

	struct xe_pt *pt_root[XE_MAX_TILES_PER_DEVICE];  // ch 7: one page
							 // table tree per tile
	unsigned long flags;             // LR_MODE / FAULT_MODE / SCRATCH_PAGE
	struct rw_semaphore lock;        // the VM's big lock
	struct list_head rebind_list;    // VMAs whose page tables are stale
	...
};
```

---

## Step 2 — Creating the VM

**File: `xe_vm.c:1621`**, `xe_vm_create()`. Only the lines that matter here:

```c
	vm->size = 1ull << xe->info.va_bits;   // 48 bits -> 256 TB of address
					       // space. Idea 1's "nothing, at first"

	vm->flags = flags;                     // the personality (Idea 5)

	INIT_LIST_HEAD(&vm->rebind_list);      // stale-page-table list
	ttm_lru_bulk_move_init(&vm->lru_bulk_move);   // ch 4's bulk LRU

	for_each_tile(tile, xe, id)
		xe_range_fence_tree_init(&vm->rftree[id]);
```

and further down, the page table roots (`:1735`):

```c
	vm->pt_root[id] = xe_pt_create(vm, tile, xe->info.vm_max_level, exec);
	//                                       ^^^^^^^^^^^^^^^^^^^^
	//  One tree per TILE (memory is per-tile - ch 1), 4 levels deep.
```

The personality flags come from userspace via `xe_vm_create_ioctl()` at `:2080`,
which maps `DRM_XE_VM_CREATE_FLAG_*` to `XE_VM_FLAG_*`.

---

## Step 3 — The ioctl arrives

**File: `xe_vm.c:2412`**, `vm_bind_ioctl_ops_create()`. This is where the
**plan** gets built. Our call is a MAP:

```c
	switch (operation) {
	case DRM_XE_VM_BIND_OP_MAP:
		...
		fallthrough;
	case DRM_XE_VM_BIND_OP_MAP_USERPTR: {
		struct drm_gpuvm_map_req map_req = {
			.map.va.addr   = range_start,          // 0x10000
			.map.va.range  = range_end - range_start,  // 0x10000
			.map.gem.obj   = obj,                  // buffer B
			.map.gem.offset = bo_offset_or_userptr,// 0
		};

		// Hand the request to shared DRM code. It returns a LIST OF
		// OPERATIONS describing what must happen - and changes NOTHING.
		ops = drm_gpuvm_sm_map_ops_create(&vm->gpuvm, &map_req);
		break;
	}
	case DRM_XE_VM_BIND_OP_UNMAP:
		ops = drm_gpuvm_sm_unmap_ops_create(&vm->gpuvm, addr, range);
		break;
	case DRM_XE_VM_BIND_OP_PREFETCH:
		ops = drm_gpuvm_prefetch_ops_create(&vm->gpuvm, addr, range);
		break;
	case DRM_XE_VM_BIND_OP_UNMAP_ALL:
		// "unmap this buffer from everywhere in this VM" - walks the
		// per-buffer list from Idea 2
		vm_bo = drm_gpuvm_bo_obtain_locked(&vm->gpuvm, obj);
		ops = drm_gpuvm_bo_unmap_ops_create(vm_bo);
		break;
	}
```

Those five cases are the entire `VM_BIND` uAPI (`include/uapi/drm/xe_drm.h`):

```c
#define DRM_XE_VM_BIND_OP_MAP          0x0
#define DRM_XE_VM_BIND_OP_UNMAP        0x1
#define DRM_XE_VM_BIND_OP_MAP_USERPTR  0x2
#define DRM_XE_VM_BIND_OP_UNMAP_ALL    0x3
#define DRM_XE_VM_BIND_OP_PREFETCH     0x4
```

Then per-operation flags get stamped onto each MAP op:

```c
	drm_gpuva_for_each_op(__op, ops) {
		struct xe_vma_op *op = gpuva_op_to_vma_op(__op);

		if (__op->op == DRM_GPUVA_OP_MAP) {
			if (flags & DRM_XE_VM_BIND_FLAG_READONLY)
				op->map.vma_flags |= XE_VMA_READ_ONLY;
			if (flags & DRM_XE_VM_BIND_FLAG_NULL)
				op->map.vma_flags |= DRM_GPUVA_SPARSE;
				//                   ^^^^^^^^^^^^^^^^ Idea 4's
				//                   "points at nothing" mapping
			if (flags & DRM_XE_VM_BIND_FLAG_CPU_ADDR_MIRROR)
				op->map.vma_flags |= XE_VMA_SYSTEM_ALLOCATOR;
				//                   ^^^^^^^^^^^^^^^^^^^^^^^ SVM
			op->map.pat_index = pat_index;   // ch 5's cache index
			...
		}
	}
```

Idea 4's four flavours are literally these flag bits.

---

## Step 4 — What `drm_gpuvm` decides, with our numbers

**File: `drm_gpuvm.c`**, inside `__drm_gpuvm_sm_map()`. It walks every existing
mapping that overlaps the requested range. For us that's one: buffer A over
`[0x00000, 0x30000)`.

Our request sits strictly *inside* it, so this branch fires (`:2475`):

```c
	// `va`    = the existing mapping  (addr = 0x00000, range = 0x30000)
	// `addr`  = 0x00000,  `end` = 0x30000     (the old mapping)
	// `req_addr` = 0x10000, `req_end` = 0x20000   (what we want)
	// `ls_range` = req_addr - addr = 0x10000      ("left side" length)

	// p (built just above): the HEAD piece we keep
	//   p = { .va.addr = 0x00000, .va.range = 0x10000,
	//         .gem.obj = A, .gem.offset = 0 }

	if (end > req_end) {                       // 0x30000 > 0x20000  -> yes
		struct drm_gpuva_op_map n = {      // the TAIL piece we keep
			.va.addr = req_end,               // 0x20000
			.va.range = end - req_end,        // 0x10000
			.gem.obj = obj,                   // still buffer A
			.gem.offset = offset + ls_range + req_range,
			//            0     + 0x10000   + 0x10000 = 0x20000
			//            ^^^ THE IMPORTANT BIT. The tail must point
			//                0x20000 into A, not at A's start.
		};

		ret = op_remap_cb(ops, priv, &p, &n, &u);   // emit the REMAP
		...
		break;
	}
	...
	return op_map_cb(ops, priv, op_map);        // emit the MAP - always LAST
```

So the list `ops` comes back holding **exactly two operations, in this order**:

```
  ops[0]  DRM_GPUVA_OP_REMAP
            .unmap = the old VMA  [0x00000, 0x30000) -> A + 0
            .prev  = map          [0x00000, 0x10000) -> A + 0
            .next  = map          [0x20000, 0x30000) -> A + 0x20000
  ops[1]  DRM_GPUVA_OP_MAP
            [0x10000, 0x20000) -> B + 0
```

Compare that with Idea 3's diagram — it is the same picture, as a data
structure. And the operation types (`drm_gpuvm.h`):

```c
enum drm_gpuva_op_type {
	DRM_GPUVA_OP_MAP,        // create
	DRM_GPUVA_OP_REMAP,      // destroy one, recreate up to 2 pieces
	DRM_GPUVA_OP_UNMAP,      // destroy
	DRM_GPUVA_OP_PREFETCH,   // "migrate this range somewhere" - no change
	DRM_GPUVA_OP_DRIVER,     // driver-private op (Xe uses this for SVM)
};

struct drm_gpuva_op_remap {
	struct drm_gpuva_op_map *prev;      // NULL if nothing to keep in front
	struct drm_gpuva_op_map *next;      // NULL if nothing to keep behind
	struct drm_gpuva_op_unmap *unmap;   // the victim
};
```

`prev` and `next` are each optional, which covers the simpler overlap shapes:
overlap the front only → `next` alone; the back only → `prev` alone; exactly
cover it → a plain UNMAP.

**Nothing has been modified yet.** `ops` is pure description.

---

## Step 5 — Turning the plan into objects

**File: `xe_vm.c:2779`**, `vm_bind_ioctl_ops_parse()`. Walk the plan, allocate a
real `xe_vma` for every piece that needs one.

For our `ops[1]` (the MAP):

```c
	case DRM_GPUVA_OP_MAP:
	{
		struct xe_vma_mem_attr default_attr = {
			.default_pat_index = op->map.pat_index,
			.pat_index = op->map.pat_index,
			.purgeable_state = XE_MADV_PURGEABLE_WILLNEED,
			...
		};

		flags |= op->map.vma_flags & XE_VMA_CREATE_MASK;

		vma = new_vma(vm, &op->base.map, &default_attr, flags);
		if (IS_ERR(vma))
			return PTR_ERR(vma);      // fail here and NOTHING is
						  // committed yet - Idea 3

		op->map.vma = vma;                // remember it for the commit

		// Count how many page-table update operations this will need.
		// (Chapter 7 consumes this count.)
		if (((op->map.immediate || !xe_vm_in_fault_mode(vm)) && ...))
			xe_vma_ops_incr_pt_update_ops(vops, op->tile_mask, 1);
		break;
	}
```

And for our `ops[0]` (the REMAP) — two new VMAs, one per surviving piece:

```c
	case DRM_GPUVA_OP_REMAP:
	{
		struct xe_vma *old = gpuva_to_vma(op->base.remap.unmap->va);
		//              ^^^ the old A-mapping [0x00000, 0x30000)

		op->remap.start = xe_vma_start(old);    // 0x00000  - remember the
		op->remap.range = xe_vma_size(old);     // 0x30000    original extent

		flags |= op->base.remap.unmap->va->flags & XE_VMA_CREATE_MASK;

		if (op->base.remap.prev) {
			vma = new_vma(vm, op->base.remap.prev, &old->attr, flags);
			//                                     ^^^^^^^^^^ inherit
			//   the OLD mapping's attributes - PAT index, preferred
			//   location, etc. A split must not silently change them.
			op->remap.prev = vma;             // head piece

			// OPTIMIZATION: if the head's page tables are already
			// correct and aligned, we don't need to rewrite them.
			op->remap.skip_prev = skip ||
				(!xe_vma_is_userptr(old) &&
				 IS_ALIGNED(xe_vma_end(vma),
					    xe_vma_max_pte_size(old)));
			if (op->remap.skip_prev) {
				// shrink the range we'll actually re-bind
				op->remap.range -= xe_vma_end(vma) - xe_vma_start(old);
				op->remap.start  = xe_vma_end(vma);
			} else {
				num_remap_ops++;
			}
		}

		if (op->base.remap.next) {
			vma = new_vma(vm, op->base.remap.next, &old->attr, flags);
			op->remap.next = vma;             // tail piece
			... same skip logic ...
		}
		break;
	}
```

Two things worth pausing on:

* **`&old->attr` is passed for the split pieces.** Splitting a mapping must
  preserve its properties — a split is not a new mapping with fresh defaults.
* **`skip_prev` / `skip_next`.** In our example the head is `[0x00000,
  0x10000)`, and its page table entries are *unchanged* by this operation. If
  the boundary happens to be aligned to the page size already in use, Xe marks
  the piece as "already bound" and doesn't rewrite its PTEs at all. **The head
  VMA object is new; its page tables are recycled.** That optimisation is
  exactly why Idea 1 insisted VMA ≠ page table.

---

## Step 6 — Commit

**File: `xe_vm.c:2700`**, inside `vm_bind_ioctl_ops_execute()`'s commit phase:

```c
	case DRM_GPUVA_OP_MAP:
		err |= xe_vm_insert_vma(vm, op->map.vma);       // add to the tree
		break;

	case DRM_GPUVA_OP_REMAP:
		// 1. remove the old mapping from the tree
		prep_vma_destroy(vm, gpuva_to_vma(op->base.remap.unmap->va), ...);

		// 2. insert the surviving pieces
		if (op->remap.prev)
			err |= xe_vm_insert_vma(vm, op->remap.prev);
		if (op->remap.next)
			err |= xe_vm_insert_vma(vm, op->remap.next);
		break;
```

And the insert itself — `xe_vm.c:1328`:

```c
static int xe_vm_insert_vma(struct xe_vm *vm, struct xe_vma *vma)
{
	lockdep_assert_held(&vm->lock);

	mutex_lock(&vm->snap_mutex);
	err = drm_gpuva_insert(&vm->gpuvm, &vma->gpuva);   // into the interval tree
	mutex_unlock(&vm->snap_mutex);
	XE_WARN_ON(err);	/* Shouldn't be possible */

	return err;
}
```

Note the `XE_WARN_ON(err)` with that comment. Insertion *cannot* fail at this
point, because the plan from Step 4 already proved the range is free. That
assertion is the payoff of building a plan first.

After the commit the tree holds three mappings where it held one:

```
   0x00000 - 0x0FFFF  ->  A + 0x00000    (new xe_vma, page tables untouched)
   0x10000 - 0x1FFFF  ->  B + 0x00000    (new xe_vma, page tables to write)
   0x20000 - 0x2FFFF  ->  A + 0x20000    (new xe_vma, page tables untouched)
```

---

## Step 7 — Linking to the buffer

The other half of Idea 2. **File: `xe_vm.c:1082`**, in `xe_vma_create()`:

```c
	// Which kind of mapping is this? (Idea 4)
	if (!bo && !is_null && !is_cpu_addr_mirror) {
		// userptr: allocate the LARGER struct, which has room for the
		// mmu-notifier state
		struct xe_userptr_vma *uvma = kzalloc_obj(*uvma);
		vma = &uvma->vma;
	} else {
		vma = kzalloc_obj(*vma);
		if (bo)
			vma->gpuva.gem.obj = &bo->ttm.base;
	}

	// Fill in Idea 1's three-part record:
	vma->gpuva.vm        = &vm->gpuvm;
	vma->gpuva.va.addr   = start;              // where
	vma->gpuva.va.range  = end - start + 1;    // how long
	vma->gpuva.flags     = flags;

	for_each_tile(tile, vm->xe, id)
		vma->tile_mask |= 0x1 << id;       // should be bound on all tiles

	if (bo) {
		// Get (or create) the "buffer as seen by this VM" middleman.
		vm_bo = drm_gpuvm_bo_obtain_locked(vma->gpuva.vm, &bo->ttm.base);

		drm_gpuvm_bo_extobj_add(vm_bo);    // if the BO is external to this
						   // VM, track it for locking
		drm_gem_object_get(&bo->ttm.base); // a mapping holds a reference -
						   // the buffer can't vanish
		vma->gpuva.gem.offset = bo_offset_or_userptr;  // offset inside it

		drm_gpuva_link(&vma->gpuva, vm_bo);
		// ^^^ THE OTHER DIRECTION. Puts this VMA on the buffer's list of
		//     mappings. This is the list Chapter 3's destructor asserted
		//     was empty, and Chapter 4's migration walked.

		drm_gpuvm_bo_put(vm_bo);
	} else /* userptr or null */ {
		if (!is_null && !is_cpu_addr_mirror) {
			// userptr: register an mmu-interval notifier on the
			// process's pages (Chapter 10)
			err = xe_userptr_setup(uvma, xe_vma_userptr(vma), size);
		}
	}
```

`drm_gem_object_get()` there is the answer to "what if userspace frees a buffer
that's still mapped?" — it can't. A mapping holds a reference, so the buffer
survives until the mapping is torn down.

---

## The journey, end to end

```
 Step 1  struct drm_gpuva            drm_gpuvm.h      where + what + tree node
         struct xe_vma               xe_vm_types.h:112 + tile_mask/tile_present
         struct xe_vm                xe_vm_types.h:209 + pt_root, rebind_list
 Step 2  xe_vm_create()              xe_vm.c:1621     size, flags, page table roots
 Step 3  vm_bind_ioctl_ops_create()  xe_vm.c:2412     5 uAPI ops -> build a PLAN
 Step 4  __drm_gpuvm_sm_map()        drm_gpuvm.c:2475 the overlap algebra
 Step 5  vm_bind_ioctl_ops_parse()   xe_vm.c:2779     plan -> xe_vma objects
 Step 6  xe_vm_insert_vma()          xe_vm.c:1328     commit into the tree
 Step 7  xe_vma_create()             xe_vm.c:1082     link into the buffer's list
```

## Try it on your machine

```bash
# every VM_BIND operation, with addresses
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_bind/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_unbind/enable
cat /sys/kernel/debug/tracing/trace_pipe

# the vm_dbg() at xe_vm.c:2429 prints op/addr/range for each request
echo 0x2 > /sys/module/drm/parameters/debug     # DRM_UT_DRIVER
```

## Checkpoint

1. A VMA is created for the *head* piece of a split, but its page table entries
   are not rewritten. How is that possible, and which field records it?
2. In our example, why is the tail piece's `gem.offset` `0x20000` and not `0`?
3. Why can `xe_vm_insert_vma()` treat failure as "shouldn't be possible"?
4. Userspace maps a buffer, then closes its GEM handle. Why doesn't the buffer
   disappear from under the mapping?
5. Which two questions do the interval tree and the per-buffer list answer, and
   where did you meet the per-buffer list in earlier chapters?

<details>
<summary>answers</summary>

1. Because a VMA is *intent* and the page tables are separate state. The head
   piece covers the same addresses, the same buffer and the same offsets as
   before, so its existing PTEs are still correct. `op->remap.skip_prev` (and
   `skip_next`) record that, and `tile_present` on the VMA tracks what is
   actually bound.
2. Because the tail covers GPU addresses `0x20000`–`0x2FFFF`, which were
   originally `0x20000` bytes into buffer A. `drm_gpuvm` computes it as
   `offset + ls_range + req_range = 0 + 0x10000 + 0x10000`. Getting this wrong
   would silently corrupt data.
3. Because the operation list built in Step 4 already determined exactly which
   ranges are free — every overlap was resolved into an UNMAP or REMAP. By
   commit time the address range is guaranteed vacant.
4. `xe_vma_create()` takes a reference with `drm_gem_object_get()`. Closing the
   handle drops userspace's reference, but the mapping's reference keeps the
   buffer alive until the mapping is destroyed.
5. Tree: *"which mapping covers GPU address X?"* — used by page fault handling.
   List: *"which mappings reference this buffer?"* — Chapter 3's destructor
   asserted it was empty before freeing, and Chapter 4's `xe_bo_trigger_rebind()`
   walked it to invalidate every mapping before a migration.

</details>
