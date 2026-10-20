# Xe Cheat Sheet

One page to keep open while reading the code. Dense on purpose — this is
lookup, not teaching. Chapter numbers point back at the course.

---

## 1. The struct map

```
xe_device                                  the whole card        (ch 1)
 ├── xe_tile[]            memory lives here: VRAM, GGTT, migrate
 │    ├── xe_vram_region  .dpa_base = global device phys address base
 │    ├── xe_ggtt         flat, ~4GB, kernel/GuC/display/LRC      (ch 5)
 │    └── xe_migrate      the copy-engine context TTM uses        (ch 4)
 └── xe_gt[]              execution lives here (MAIN + MEDIA)
      ├── xe_hw_engine[]  render / compute / copy / media
      ├── xe_guc          firmware + CT mailbox                   (ch 12)
      └── xe_tlb_inval    per-GT invalidation                     (ch 9)

MEMORY IS PER-TILE.  EXECUTION IS PER-GT.   <- the one thing to remember
```

### Three-layer embeddings (`container_of` everywhere)

```
xe_bo         ⊃ ttm_buffer_object ⊃ drm_gem_object      (ch 3)
xe_vm         ⊃ drm_gpuvm                               (ch 6)
xe_vma        ⊃ drm_gpuva                               (ch 6)
xe_exec_queue → xe_lrc[] ⊃ xe_bo                        (ch 10)
xe_sched_job  ⊃ drm_sched_job                           (ch 11)
xe_userptr    ⊃ drm_gpusvm_pages    (shared with SVM!)  (A1)
xe_bo         ⊃ drm_pagemap_devmem  (when SVM-migrated) (A1)
```

---

## 2. Files that matter

| File | Lines | Role | Ch |
|---|---|---|---|
| `xe_bo.c` | 4092 | buffer objects, placements, `xe_bo_move` | 2–4 |
| `xe_vm.c` | 4992 | VMs, VMAs, bind ops, rebind worker | 6, 8, 14 |
| `xe_pt.c` | 2934 | page tables: stage / commit, PTE encode | 7, 8 |
| `xe_exec.c` | 382 | the `EXEC` ioctl — read this one in full | 13 |
| `xe_exec_queue.c` | 1873 | hardware contexts | 10 |
| `xe_lrc.c` | 2818 | ring + register image | 10 |
| `xe_guc_submit.c` | 3736 | GuC backend, **TDR lives here** | 12, 15 |
| `xe_guc_ct.c` | 2326 | H2G / G2H mailbox | 12 |
| `xe_migrate.c` | 2637 | GPU-driven copies and PTE writes | 4, 7 |
| `xe_ggtt.c` | 1167 | the flat global table | 5 |
| `xe_svm.c` | 2085 | SVM ranges, migration | A1 |
| `xe_pagefault.c` | 497 | recoverable fault plumbing | A1 |
| `xe_userptr.c` | 434 | malloc'd memory, no BO | 13 |
| `xe_preempt_fence.c` | 191 | LR-mode preemption — **start here for ch 14** | 14 |
| `xe_bo_evict.c` | 352 | suspend/resume lists | A2 |
| `xe_dma_buf.c` | 398 | cross-driver sharing | A2 |
| `xe_sync.c` | 446 | syncobjs and user fences | 13 |
| `xe_gpu_scheduler.c` | 139 | thin wrapper + the message queue | 11, 12 |

Smallest useful reading order: `xe_preempt_fence.c` (191) →
`xe_exec.c` (382) → `xe_bo_evict.c` (352) → `xe_userptr.c` (434).

---

## 3. uAPI → function

```c
// xe_device.c:195
XE_VM_CREATE          -> xe_vm_create_ioctl()          xe_vm.c:2080
XE_GEM_CREATE         -> xe_gem_create_ioctl()         xe_bo.c:3515
XE_VM_BIND            -> xe_vm_bind_ioctl()            xe_vm.c:3882
XE_EXEC_QUEUE_CREATE  -> xe_exec_queue_create_ioctl()  xe_exec_queue.c:1271
XE_EXEC               -> xe_exec_ioctl()               xe_exec.c:111
XE_WAIT_USER_FENCE    -> xe_wait_user_fence_ioctl()    xe_wait_user_fence.c:100
```

14 ioctls total; those six are the whole memory/scheduling story.

---

## 4. The four call chains

### Bind (ch 8)
```
xe_vm_bind_ioctl
  -> vm_bind_ioctl_ops_create       parse into MAP/REMAP/UNMAP
  -> vm_bind_ioctl_ops_execute
       -> xe_pt_stage_bind          xe_pt.c:777   build a SHADOW tree
       -> xe_pt_update_ops_run                    submit PTE writes to GPU
       -> xe_pt_commit              xe_pt.c:1179  children[j] = staging[j]
```

### Exec (ch 13)
```
xe_exec_ioctl                                   xe_exec.c:111
  -> xe_validation_exec_lock(.extra.fn = xe_exec_fn)   :271  locks, and
       -> xe_vm_validate_rebind                        :106  validates INSIDE
  -> __xe_vm_userptr_needs_repin                       :324  stale? retry
  -> xe_sched_job_arm / xe_sched_job_push
       -> guc_exec_queue_run_job        xe_guc_submit.c:1209
            -> emit_job                 write the ring       (ch 10)
            -> submit_exec_queue        2-dword H2G          (ch 12)
```

### GPU page fault (A1)
```
G2H irq -> xe_pagefault_handler        xe_pagefault.c:475  queue it
        -> xe_pagefault_service                      :170  asid -> vm -> vma
             -> xe_svm_handle_pagefault  (mirror VMA)
                  -> xe_svm_garbage_collector    drain dead ranges FIRST
                  -> xe_svm_range_find_or_insert
                  -> xe_svm_alloc_vram           maybe migrate
                  -> xe_svm_range_get_pages      hmm + dma_map
                  -> xe_vm_range_rebind          ch 8's bind path, reused
             -> xe_pagefault_handle_vma  (plain BO) just rebind
```

### Suspend / resume (A2)
```
xe_pm_suspend -> xe_bo_evict_all        xe_bo_evict.c:160
                  1. user BOs      (GPU)
                  2. late pinned   (GPU)
                  3. xe_tile_migrate_wait   <- MUST finish before 4
                  4. early pinned  (CPU)    the GuC image itself

xe_pm_resume  -> xe_bo_restore_early    :226   CPU, + re-map GGTT
              -> xe_gt_resume                  GuC loads
              -> xe_bo_restore_late     :243   GPU
              (ordinary BOs: nothing — next EXEC revalidates them)
```

---

## 5. Lock order — outermost first

```
   vm->lock                     (rw_semaphore)
     └── dma_resv                (ww_mutex, via drm_exec, retry on contention)
           └── notifier_lock      (vm->svm.gpusvm.notifier_lock)
```

**The rule that causes every deadlock here:** an MMU notifier is entered
with `notifier_lock` held for **write**, so it can never take `vm->lock`.
That is why every notifier in this driver only *marks* and *lists*, and
real repair happens later in process context. (ch 13, A1)

Spinlocks, taken innermost, never nested with the above:
`vm->userptr.invalidated_lock`, `vm->svm.garbage_collector.lock`,
`xe->pinned.lock`, `guc->submission_state.lock`.

---

## 6. Flags

### `XE_BO_FLAG_*` (`xe_bo.h`)
| Flag | Meaning |
|---|---|
| `SYSTEM` / `VRAM0` / `VRAM1` / `STOLEN` | allowed placements |
| `VRAM_IF_DGFX(tile)` | VRAM on discrete, system on integrated |
| `GGTT` | also needs a GGTT mapping |
| `PINNED` | never moves during normal operation |
| `PINNED_LATE_RESTORE` | restore by **GPU**; without it, by **CPU** → must be contiguous |
| `NEEDS_CPU_ACCESS` | must be CPU-visible (vmap / BAR) |
| `CPU_ADDR_MIRROR` | SVM: no real backing, mirrors a CPU range |
| `USER` | created by an ioctl, not by the driver |

### `XE_VM_FLAG_*` (`xe_vm_types.h:257`)
```
BIT(0) 64K            BIT(1) LR_MODE        BIT(2) MIGRATION
BIT(3) SCRATCH_PAGE   BIT(4) FAULT_MODE     BIT(5) BANNED
BIT(8) GSC            BIT(9) NO_VM_OVERCOMMIT
```
```c
xe_vm_in_lr_mode(vm)            = LR_MODE
xe_vm_in_fault_mode(vm)         = FAULT_MODE
xe_vm_in_preempt_fence_mode(vm) = LR_MODE && !FAULT_MODE    // ch 14
```

### Exec queue state (`xe_guc_submit.c:65`)
```
BIT(0) REGISTERED   BIT(1) ENABLED     BIT(2) PENDING_ENABLE
BIT(3) PENDING_DISABLE  BIT(4) DESTROYED   BIT(5) SUSPENDED
BIT(6) RESET        BIT(7) KILLED      BIT(8) WEDGED
BIT(9) BANNED       BIT(10) PENDING_RESUME
```
Sticky verdicts that survive a GT reset: `WEDGED BANNED KILLED DESTROYED
SUSPENDED` (`guc_exec_queue_stop`, `xe_guc_submit.c:2578`).

---

## 7. PTE bits (`regs/xe_gtt_defs.h`)

```
address field:  XE_PAGE_ADDR_MASK(shift) = GENMASK_ULL(51, shift)
```

### GGTT
```
bit  0   XE_PAGE_PRESENT
bit  1   XE_GGTT_PTE_DM         device memory (VRAM)
bits 2-11 GGTT_PTE_VFID         SR-IOV VF id
bit 52   XELPG_GGTT_PTE_PAT0
bit 53   XELPG_GGTT_PTE_PAT1
```

### PPGTT
```
bit  0   XE_PAGE_PRESENT
bit  1   XE_PAGE_RW
bit  3   XE_PPGTT_PTE_PAT0
bit  4   XE_PPGTT_PTE_PAT1
bit  6   XE_PDE_64K             (in a PDE)
bit  7   XE_PPGTT_PTE_PAT2  /  XE_PDE_PS_2M  /  XE_PDPE_PS_1G   <- context!
bit  8   XE_PTE_PS64
bit  9   XE_PTE_NULL            sparse mapping, reads 0
bit 10   XE_USM_PPGTT_PTE_AE    atomic enable
bit 11   XE_PPGTT_PTE_DM        device memory  /  XE_PDE_IPS_64K in a PDE
bit 12   XE_PPGTT_PDE_PDPE_PAT2 (PDE/PDPE only)
bit 61   XE2_PPGTT_PTE_PAT4
bit 62   XELPG_PPGTT_PTE_PAT3
```

**Bit 7 is overloaded** — PAT2 in a leaf PTE, "this is a 2M/1G leaf" in a
PDE/PDPE. Always know which level you are looking at.

Level shifts: `{12, 21, 30, 39, 48}` → 4K, 2M, 1G, 512G, 256T.

Worked examples from the course:
```
0x0030000112345001   GGTT, system memory @ 0x112345000, PAT0+PAT1
0x0000000640000893   PPGTT, 2MB VRAM leaf @ 0x640000000, DM set
0x0000000112345003   PPGTT, 4K system @ 0x112345000, RW, DM CLEAR
```

---

## 8. Memory types (`xe_bo.h:82`)

```c
XE_PL_SYSTEM  = TTM_PL_SYSTEM            swappable, not GPU-mapped
XE_PL_TT      = TTM_PL_TT                system pages, GPU-mapped
XE_PL_VRAM0   = TTM_PL_VRAM              tile 0 device memory
XE_PL_VRAM1   = XE_PL_VRAM0 + 1          tile 1
XE_PL_STOLEN  = TTM_NUM_MEM_TYPES - 1    BIOS-reserved
```

Resource managers: gpu_buddy (VRAM), accountant (TT), `drm_mm` range
(stolen).

---

## 9. Fence vocabulary

| Fence | Means | Published where | Ch |
|---|---|---|---|
| `job->drm.s_fence->scheduled` | dependencies met | internal | 11 |
| `job->drm.s_fence->finished` | **GPU finished the work** | `dma_resv` BOOKKEEP — *except in LR mode* | 11, 13 |
| `job->fence` (the "parent") | backend-level completion | returned by `run_job` | 11 |
| **preempt fence** | **"my queue is off the hardware"** | `dma_resv` BOOKKEEP | 14 |
| bind fence | PTE writes landed | `dma_resv` KERNEL | 8 |
| user fence | value copied into the app's memory | `xe_sync.c:76` | 13 |

`dma_resv` usage classes, strongest first:
`KERNEL` (moves, PTE writes) > `WRITE` > `READ` > `BOOKKEEP` (jobs,
preempt fences).

---

## 10. Symptom → where to look

| You see | Look at |
|---|---|
| `"Timedout job: seqno=..."` | `guc_exec_queue_timedout_job` `xe_guc_submit.c:1496` (ch 15) |
| `"Engine reset: engine_class=..."` | GuC reset an engine itself, `:3205` |
| `"Kernel-submitted job timed out"` | karma budget spent → **wedge** incoming, `:1658` |
| `"reset started"` / `"reset done"` | `gt_reset_worker` `xe_gt.c:912` |
| `"CRITICAL: ... declared device as wedged"` | `xe_device_declare_wedged` `xe_device.c:1453` |
| `"VM worker error:"` | the LR rebind worker died, `xe_vm.c:594` (ch 14) |
| `"PREEMPT FENCE RETRY guc_id="` | VF-migration retry, `xe_preempt_fence.c:28` |
| `"Schedule disable failed to respond"` | GuC not answering → GT reset, `:1619` |
| ioctls all return `-ECANCELED` | device is wedged, `xe_device.c:227` |
| `-EFAULT` from an `EXEC` | userptr pages gone (app freed them), `xe_userptr.c:320` |
| repeated `-EAGAIN` in a trace | seqno/retry loop; userptr or SVM churn (ch 13, A1) |

Useful tracepoints: `xe_vma_*`, `xe_sched_job_*`, `xe_exec_queue_*`,
`xe_vm_rebind_worker_*`, `xe_guc_ct_*`.

---

## 11. Chapter index

| Act | # | Title |
|---|---|---|
| I | 1 | The Cast — device / tile / GT |
| | 2 | Where Memory Lives — TTM placements |
| | 3 | The Crate — the BO's three layers |
| | 4 | Moving Things Around — eviction |
| | 5 | Two Address Spaces — GGTT vs PPGTT |
| | 6 | The VM and the VMA |
| | 7 | Page Tables — PTE encoding, huge pages |
| | 8 | VM_BIND End to End |
| | 9 | Locking, and Making the GPU Forget |
| II | 10 | The Scheduling Cast — exec queue, LRC |
| | 11 | The Waiting Room — drm_gpu_scheduler |
| | 12 | Talking to the GuC |
| | 13 | EXEC, End to End |
| III | 14 | The Program That Never Ends — LR mode |
| | 15 | When Things Go Wrong — TDR, reset, wedge |
| | 16 | The Life of One Frame — capstone |
| App | A1 | Memory That Isn't There Yet — SVM, faults |
| | A2 | When the Lights Go Out — suspend, dma-buf |

---

## The five sentences

1. **Memory is per-tile; execution is per-GT.**
2. **Everything is a `dma_fence`** — moves, PTE writes, TLB flushes, job
   completion, preemption, death.
3. **Each layer answers one question:** TTM *where*, gpuvm *what address*,
   drm_sched *what order*, GuC *what now*, dma_fence *who may proceed*.
4. **Notifiers only mark; process context repairs** — because a notifier
   can never take `vm->lock`.
5. **The cold path is built once; a frame costs a lock dance, a validate, a
   ring write and two dwords.**
