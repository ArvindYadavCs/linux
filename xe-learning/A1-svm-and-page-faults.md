# Appendix A1 — Memory That Isn't There Yet

> SVM, recoverable page faults, and migration. The optional branch off Act I.

---

## Beat A — The story

### Idea 1 — Invert the contract: bind nothing, fault on demand

Every chapter so far followed the same contract: **bind first, then run.**
`VM_BIND` writes PTEs; `EXEC` runs; if a page isn't mapped, that's a bug.

Fault mode flips it. Map **nothing**. Let the GPU hit an unmapped address,
take a fault, and bind the page inside the fault handler. One VMA can cover
a terabyte of address space while PTEs exist only for the pages actually
touched.

The one hardware feature that makes this possible: the GPU can take a
**recoverable** page fault — stop, wait for the kernel, and resume — rather
than dying like Chapter 15's CAT error.

### Idea 2 — The GPU's addresses become the CPU's addresses

A fault-mode VM maps a special kind of VMA:

```
CPU_ADDR_MIRROR:  "for this whole range, GPU VA == CPU VA. Forever."
```

There is no BO. There is no offset table. A `malloc()` pointer is simply a
valid GPU pointer, and stays valid as the allocator reuses it. Linked
lists, trees and pointer-chasing data structures work on the GPU without
being flattened or relocated.

The fault handler's only job is to answer: *where does the CPU keep these
pages right now?*

### Idea 3 — Pages can move into VRAM, and the CPU gets faulted out

This is the part userptr (Chapter 13's Q&A) deliberately does **not** do.

When the GPU faults on a range it is about to work on heavily, SVM can
**migrate** those pages into VRAM. They keep the same CPU virtual address —
but they become *device-private*. A CPU access to them now takes a fault of
its own, and **that** handler migrates them back to system memory.

Two fault handlers, pointing at each other. Which immediately raises the
obvious danger, and the fix for it, both covered below.

---

## Beat B — The code

> Trimmed hard: stats, multi-pagemap, atomics and SR-IOV removed.

### The worked example

```c
float *buf = malloc(2 * 1024 * 1024);   // ordinary malloc
buf[0] = 1.0f;                          // (1) CPU writes
gpu_kernel<<<...>>>(buf);               // (2) GPU reads  -> fault
printf("%f\n", buf[0]);                 // (3) CPU reads back -> fault
```

Three touches of the same pointer. No `VM_BIND` between them. We trace all
three.

### Named functions

| Function | Where | Job |
|---|---|---|
| `xe_pagefault_handler()` | `xe_pagefault.c:475` | IRQ: park the fault on a queue |
| `xe_pagefault_service()` | `xe_pagefault.c:170` | find the VM and VMA, route it |
| `__xe_svm_handle_pagefault()` | `xe_svm.c:1233` | the real work |
| `xe_svm_alloc_vram()` | `xe_svm.c:1648` | migrate into VRAM |
| `drm_pagemap_migrate_to_ram()` | `drm_pagemap.c:1348` | the CPU faulting it back |

---

### Step 0 — Setup: a bind that writes no PTEs

The app creates its VM with `XE_VM_CREATE_FLAG_FAULT_MODE`, then binds the
mirror range once:

```c
// xe_vm.c:1093 — a VMA with no BO and no pages. It is a DECLARATION
// ("this range mirrors the CPU"), not a mapping.
bool is_cpu_addr_mirror = (flags & XE_VMA_SYSTEM_ALLOCATOR);

// xe_vm.c:1102 — note what is NOT required: no bo, and that is legal
// only for the three address-less kinds.
if (!bo && !is_null && !is_cpu_addr_mirror)
	return ERR_PTR(-EINVAL);
```

No page tables are touched. The GPU's PPGTT still has nothing for this
range.

### Step 1 — The CPU writes. Nothing happens.

`buf[0] = 1.0f` faults in one ordinary anonymous page via the normal Linux
mm. Xe is not involved, not notified, and does not care.

### Step 2 — The GPU reads, and faults

The GPU walks the PPGTT, finds nothing, and raises a recoverable fault. The
GuC forwards it as a G2H, landing in interrupt context:

```c
// xe_pagefault.c:475 — IRQ context. Do the absolute minimum:
// copy the fault onto a per-ASID queue and wake a worker.
spin_lock_irqsave(&pf_queue->lock, flags);
memcpy(pf_queue->data + pf_queue->head, pf, sizeof(*pf));
pf_queue->head = (pf_queue->head + xe_pagefault_entry_size()) % pf_queue->size;
queue_work(xe->usm.pf_wq, &pf_queue->worker);
spin_unlock_irqrestore(&pf_queue->lock, flags);
```

Note the queue is indexed by **ASID** (`pf->consumer.asid % COUNT`), so
different VMs' faults land on different queues and don't serialise behind
each other.

The worker drains it, with a deliberate fairness cap:

```c
// xe_pagefault.c:298
#define USM_QUEUE_MAX_RUNTIME_MS      20
threshold = jiffies + msecs_to_jiffies(USM_QUEUE_MAX_RUNTIME_MS);
```

Then the routing. **The ASID is how a hardware fault finds a software VM:**

```c
// xe_pagefault.c:183
vm = xe_pagefault_asid_to_vm(xe, pf->consumer.asid);   // ASID -> xe_vm

down_write(&vm->lock);
vma = xe_vm_find_vma_by_addr(vm, pf->consumer.page_addr);   // ch 6's tree

// Permission check before anything else: a write fault on a read-only
// VMA is the app's bug, not a page to go fetch.
if (xe_vma_read_only(vma) &&
    pf->consumer.access_type != XE_PAGEFAULT_ACCESS_TYPE_READ)
	return -EPERM;

// xe_pagefault.c:211 — the fork. Our mirror VMA takes the first branch.
if (xe_vma_is_cpu_addr_mirror(vma))
	err = xe_svm_handle_pagefault(vm, vma, gt, pf->consumer.page_addr, atomic);
else
	err = xe_pagefault_handle_vma(gt, vma, atomic);   // plain BO, just rebind
```

### Step 3 — The handler: four moves

```c
// xe_svm.c:1233 — stripped to its spine
static int __xe_svm_handle_pagefault(...)
{
	int migrate_try_count = ctx.devmem_only ? 3 : 1;

retry:
	// (a) Process pending UNMAPs first, so our view of ranges is current.
	err = xe_svm_garbage_collector(vm);

	// (b) Find or create the SVM range covering this address. Ranges are
	//     chunks (not single pages) so one fault binds many pages.
	range = xe_svm_range_find_or_insert(vm, fault_addr, vma, &ctx);

	// Already bound by a fault we raced with? Done, nothing to do.
	if (xe_svm_range_is_valid(range, tile, ctx.devmem_only, dpagemap))
		goto out;

	// (c) Worth moving into VRAM? (Idea 3 — see Step 4.)
	if (--migrate_try_count >= 0 &&
	    xe_svm_range_needs_migrate_to_vram(range, vma, dpagemap)) {
		err = xe_svm_alloc_vram(range, &ctx, dpagemap);
		ctx.timeslice_ms <<= 1;      /* double the timeslice on retry */
		if (err)
			goto retry;          // VRAM full: just use system memory
	}

get_pages:
	// (d) Where are the pages NOW? hmm_range_fault + dma_map, exactly as
	//     userptr does (ch 13). This is the shared drm_gpusvm helper.
	err = xe_svm_range_get_pages(vm, range, &ctx);
	if (err == -EFAULT || err == -EPERM || err == -EOPNOTSUPP) {
		ctx.timeslice_ms <<= 1;
		goto retry;                  // CPU mappings changed under us
	}

	// (e) Write the PTEs — ch 7 and ch 8, unchanged. Same stage/commit
	//     machinery that VM_BIND uses.
	xe_validation_guard(&vctx, &vm->xe->val, &exec, ...) {
		err = xe_vm_drm_exec_lock(vm, &exec);
		drm_exec_retry_on_contention(&exec);
		fence = xe_vm_range_rebind(vm, vma, range, BIT(tile->id));
	}

	// The PTE writes are GPU work. Wait, then tell the GuC to resume
	// the faulting context.
	dma_fence_wait(fence, false);
out:
	return 0;
}
```

#### Why step (a) is first, and why it is inline

The MMU notifier that handles a `munmap` of a mirrored range runs with the
notifier lock held for write: it cannot sleep and cannot submit GPU work.
So it only does what is possible there:

```c
// xe_svm.c:133
drm_gpusvm_range_set_unmapped(&range->base, &range->pages, 1, mmu_range);
spin_lock(&vm->svm.garbage_collector.lock);
list_add_tail(&range->garbage_collector_link,
	      &vm->svm.garbage_collector.range_list);
spin_unlock(&vm->svm.garbage_collector.lock);
queue_work(xe->usm.pf_wq, &vm->svm.garbage_collector.work);
```

The real teardown submits GPU work and waits, so it needs process context:

```c
// xe_svm.c:304
fence = xe_vm_range_unbind(vm, range);        // GPU work: tear down PTEs
dma_fence_put(fence);
drm_gpusvm_unmap_pages(...);                  // dma_unmap
drm_gpusvm_range_remove(&vm->svm.gpusvm, &range->base);   // out of the tree
```

**So the range tree legitimately contains zombies** — ranges marked
`unmapped`, still in the tree, pages already returned to the kernel.

The handler cannot wait for the worker to clear them, because the worker
needs the very lock the handler is holding:

```c
// xe_svm.c:437               |  // xe_pagefault.c:190
down_write(&vm->lock);        |  down_write(&vm->lock);   // the handler, already
xe_svm_garbage_collector(vm); |                           // holding it
up_write(&vm->lock);          |
```

So it calls the same function inline. One function, two callers — the
worker takes the lock, the handler already has it
(`lockdep_assert_held_write(&vm->lock)` satisfies both).

Three things break without it:

1. **A leak.** `find_or_insert()` could return a zombie, and the handler
   would DMA-map and bind pages the app no longer owns.
2. **An infinite fault loop.** The notifier only *zapped* the PTEs; the
   range still records a binding, so `xe_svm_range_is_valid()` says "already
   bound", the handler returns 0 having bound nothing, and the GPU faults on
   the same address forever.
3. **A dangling `xe_vma *`.** Collecting a range can *restructure* the VMA
   layout and says so explicitly:

```c
// xe_svm.c:368 (xe_svm_range_set_default_attr)
err = xe_vm_alloc_cpu_addr_mirror_vma(vm, start, end - start);   // NEW VMA
/*
 * On call from xe_svm_handle_pagefault original VMA might be changed
 * signal this to lookup for VMA again.
 */
return -EAGAIN;
```

That `-EAGAIN` travels out to the outer wrapper, which re-resolves the VMA
from scratch (`xe_svm.c:1418`). Draining *first* means that discovery
happens before anything is committed; draining later would leave the
handler mid-bind on a freed VMA.

Note also that step (a) sits **inside** the `retry:` label, so it runs
again on every retry — a retry means time passed, and more ranges may have
been queued meanwhile.

Look at step (e): **the fault handler reuses Chapter 8's bind path
verbatim.** Fault mode didn't add a new way to write page tables. It only
changed *who decides when* — which is Chapter 16's Idea 3 in action: one
layer's answer replaced, the rest untouched.

### Step 4 — Migration into VRAM

The decision first:

```c
// xe_svm.c:1156
bool xe_svm_range_needs_migrate_to_vram(...)
{
	if (!flags.migrate_devmem || !dpagemap)
		return false;                  // app or platform said no

	if (xe_svm_range_has_pagemap(range, dpagemap))
		return false;                  // already there

	if (range_size < SZ_64K && !supports_4K_migration(vm->xe))
		return false;                  // too small to be worth it

	return true;
}
```

Then the move. The copy is done **by the GPU's own copy engine** — the same
`xe_migrate` context Chapter 4 used for evictions:

```c
// xe_svm.c:748 — two directions, one implementation
static int xe_svm_copy_to_devmem(...)
{
	return xe_svm_copy(pages, pagemap_addr, npages, XE_SVM_COPY_TO_VRAM, ...);
}
static int xe_svm_copy_to_ram(...)
{
	return xe_svm_copy(pages, pagemap_addr, npages, XE_SVM_COPY_TO_SRAM, ...);
}
```

After the copy, the pages are **device-private**: a `struct page` backed by
VRAM, registered with a `dev_pagemap`. Still the same CPU virtual address —
but no longer CPU-accessible.

### Step 5 — The CPU reads it back, and faults

`printf("%f\n", buf[0])` now touches a device-private page. Linux mm sees
the pagemap and calls its owner:

```c
// drm_pagemap.c:1374 — the ops that make this work
static const struct dev_pagemap_ops drm_pagemap_pagemap_ops = {
	.folio_free   = drm_pagemap_folio_free,
	.migrate_to_ram = drm_pagemap_migrate_to_ram,   // <-- the CPU's way back
	.folio_split  = drm_pagemap_folio_split,
};
```

```c
// drm_pagemap.c:1348 — a normal CPU page fault, handled by the GPU driver
static vm_fault_t drm_pagemap_migrate_to_ram(struct vm_fault *vmf)
{
	struct drm_pagemap_zdd *zdd = drm_pagemap_page_zone_device_data(vmf->page);

	err = __drm_pagemap_migrate_to_ram(vmf->vma, vmf->page, vmf->address,
					   zdd->devmem_allocation->size);
	return err ? VM_FAULT_SIGBUS : 0;
}
```

That copies the data back to system memory (GPU copy engine again, via
`xe_svm_copy_to_ram`), restores ordinary pages at the same virtual address,
and the CPU's load completes. **The app never knew.**

### Step 6 — The ping-pong guard

Two fault handlers that migrate in opposite directions is a livelock
waiting to happen: GPU faults → migrate to VRAM → CPU faults → migrate to
RAM → GPU faults → …

The brake is a **timeslice** that doubles on every retry:

```c
// xe_svm.c:1244 — initial budget
.timeslice_ms = need_vram && devmem_possible ? vm->xe->atomic_svm_timeslice_ms : 0,

// and on each retry, in three places in the handler:
ctx.timeslice_ms <<= 1;    /* Double timeslice if we have to retry */
```

```c
// xe_svm.c:1682 — the budget is handed down to the migration itself
err = drm_pagemap_populate_mm(dpagemap, start, end, mm, ctx->timeslice_ms);
```

The timeslice says *"leave these pages in VRAM for at least this long before
letting the CPU pull them back."* Each round of contention doubles it, so a
genuinely shared buffer converges on staying in one place instead of
thrashing. Same idea as Chapter 14's `min_run_period_ms` — a minimum
residency to guarantee forward progress.

### Step 7 — What TTM does with an SVM buffer

The VRAM holding migrated pages is still a `xe_bo`, so TTM can still try to
evict it. Two hooks make that safe:

```c
// xe_bo.c:334 — an SVM BO has exactly one eviction destination: system.
// No VRAM-to-VRAM shuffling, because the pages' identity is tied to the
// CPU address space.
if (bo->flags & XE_BO_FLAG_CPU_ADDR_MIRROR) {
	*placement = sys_placement;
	return;
}
```

```c
// xe_bo.c:1031 — and the move itself is not a TTM copy. It is a full
// SVM migration back to RAM, because the device-private pages must be
// replaced with real ones.
if (!move_lacks_source && (bo->flags & XE_BO_FLAG_CPU_ADDR_MIRROR) &&
    new_mem->mem_type == XE_PL_SYSTEM) {
	ret = xe_svm_bo_evict(bo);            // -> drm_pagemap_evict_to_ram()
	if (!ret)
		ttm_bo_move_null(ttm_bo, new_mem);
}
```

So memory pressure and a CPU access reach the same destination through
different doors — which is exactly the shape Chapter 15's TDR had.

---

## userptr vs SVM, side by side

The three Q&As from Chapter 13 and this appendix, in one table:

| | **userptr** | **SVM / fault mode** |
|---|---|---|
| Who binds? | `VM_BIND`, up front | the **fault handler**, on demand |
| GPU VA | chosen by userspace | **equals** the CPU VA |
| Backed by | a `xe_vma` with a userptr | a `CPU_ADDR_MIRROR` VMA + SVM ranges |
| Pages live in | **system RAM, always** | system RAM **or VRAM** |
| How the GPU reaches them | DMA address in the PTE, over PCIe | PTE to system *or* to VRAM (`DM` bit set) |
| Does the CPU lose access? | **never** | **yes**, while migrated (device-private) |
| Who repairs staleness? | the next `EXEC` (`retry:`) | the fault handler |
| Shared machinery | `drm_gpusvm_get_pages()`, `hmm_range_fault`, `dma_map_page` | the same |

The last row is why `struct xe_userptr` embeds a `struct
drm_gpusvm_pages`. userptr is, in effect, **SVM with migration and faulting
switched off**.

---

## What to carry forward

1. **Fault mode replaces one answer, not the stack.** The handler still
   calls Chapter 8's `xe_vm_range_rebind()` to write Chapter 7's PTEs under
   Chapter 9's locks. Only "who decides when to bind" changed. That is what
   a well-layered driver buys you.
2. **The ASID is the bridge.** A hardware fault carries a number; that
   number finds a `xe_vm`; that VM's interval tree finds a VMA; the VMA's
   kind picks the handler. Four lookups from silicon to software policy.
3. **Two handlers facing each other need a brake.** GPU-faults-in and
   CPU-faults-back would livelock without the doubling timeslice —
   the same minimum-residency trick as Chapter 14's preempt fences. Any
   system that can move a resource both ways needs one.

---

One appendix left: **A2** — pinning, suspend/resume and dma-buf. What
happens to all this memory when the machine sleeps, and how buffers cross
driver boundaries (including the shared-`dma_resv` problem from the
Chapter 14 critique).
