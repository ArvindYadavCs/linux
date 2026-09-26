# Xe Deep Dive — Memory Management & Scheduling

A story-driven walkthrough of the Intel `xe` DRM driver, the DRM core pieces it
stands on (GEM, TTM, `drm_gpuvm`, `drm_gpusvm`, `drm_sched`), and the two flows
that matter most: **how memory gets mapped** and **how work gets scheduled**.

Tree this is written against: Linux **v7.3-rc1**, `drivers/gpu/drm/xe`.

## How this course works

Every chapter has two beats:

* **Beat A — The Story.** The idea in plain language. Characters, their job,
  who they talk to. No code.
* **Beat B — The Code.** The same idea, now in the actual source: structs,
  functions, file:line. Only after Beat A is understood.

We never move to the next chapter until both beats of the current one land.

## The map

### Act 0 — The World
| # | Chapter | Main source |
|---|---------|-------------|
| 1 | The Cast: device, tile, GT, engine, GuC | `xe_device_types.h`, `xe_tile_types.h`, `xe_gt_types.h` |

### Act I — Memory Management
| # | Chapter | Main source |
|---|---------|-------------|
| 2 | Where memory lives: placements & resource managers | `xe_bo.h`, `xe_ttm_vram_mgr.c`, `xe_ttm_sys_mgr.c` |
| 3 | The crate: `xe_bo` on top of GEM + TTM | `xe_bo_types.h`, `xe_bo.c` |
| 4 | TTM the warehouse manager: validate, move, evict | `xe_bo.c` (`xe_ttm_funcs`), `ttm/ttm_bo.c` |
| 5 | Two address spaces: GGTT vs PPGTT | `xe_ggtt.c` |
| 6 | The VM and the VMA (`drm_gpuvm`) | `xe_vm.c`, `xe_vm_types.h` |
| 7 | Page tables: staging, commit, the GPU writing its own PTEs | `xe_pt.c`, `xe_migrate.c` |
| 8 | VM_BIND end to end | `xe_vm.c:xe_vm_bind_ioctl` |
| 9 | Coherency: `dma_resv`, `drm_exec`, TLB invalidation | `xe_tlb_inval.c`, `xe_exec.c` |
| 10 | userptr, SVM and page faults | `xe_userptr.c`, `xe_svm.c`, `xe_pagefault.c` |
| 11 | Pinning, eviction, suspend/resume, dma-buf | `xe_bo_evict.c`, `xe_dma_buf.c` |

### Act II — Scheduling
| # | Chapter | Main source |
|---|---------|-------------|
| 12 | The Cast again: exec queue, LRC, ring, GuC | `xe_exec_queue_types.h`, `xe_lrc_types.h` |
| 13 | The LRC and the ring: how a batch reaches hardware | `xe_lrc.c`, `xe_ring_ops.c` |
| 14 | `drm_gpu_scheduler`: entities, jobs, fences | `drm/scheduler/sched_main.c`, `xe_sched_job.c` |
| 15 | Xe's 1:1 entity:scheduler model and messages | `xe_gpu_scheduler.c` |
| 16 | GuC submission: CT, registration, G2H | `xe_guc_submit.c`, `xe_guc_ct.c` |
| 17 | EXEC ioctl end to end | `xe_exec.c` |
| 18 | Where MM meets scheduling: LR mode, preempt fences, rebind worker | `xe_preempt_fence.c`, `xe_vm.c` |
| 19 | When things go wrong: TDR, reset, ban, wedge | `xe_guc_submit.c`, `xe_gt.c` |
| 20 | The full picture: the life of one frame | everything |

## Chapters written so far

* [01 — The Cast](01-the-cast.md) — story + code
* [02 — Where Memory Lives](02-where-memory-lives.md) — story + code
* [03 — The Crate](03-the-crate.md) — story + code
* [04 — The Warehouse Manager at Work](04-the-warehouse-manager.md) — story + code
