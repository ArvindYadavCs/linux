# Chapter 16 — The Life of One Frame

> The capstone. Every layer of this course, in the order one frame touches them.

---

## Beat A — The story

### Idea 1 — Draw the line between "once" and "every frame"

Almost everything this course explained in detail is **built once, at
startup, and never rebuilt**: the VM, the page-table tree, the LRC, the
ring, the GuC context registration.

The per-frame path deliberately touches almost none of it. At 60fps you get
16.6ms per frame, and you cannot spend it allocating page tables.

```
  COLD PATH (once per app)          HOT PATH (60 times per second)
  ─────────────────────────         ──────────────────────────────
  open device                       reuse BOs
  create VM  -> page tables         write commands into a BO
  create BOs -> VRAM/system          XE_EXEC ioctl
  VM_BIND    -> write PTEs           wait for the fence
  create exec queue -> LRC, ring
  first submit -> GuC registers it
```

Knowing which side of that line a thing sits on **is** the architecture. If
you only remember one diagram from this course, remember that one.

### Idea 2 — The whole driver is one fence graph

Count the different mechanisms this course covered for "wait until X":

| What is waited on | Chapter |
|---|---|
| a buffer finished moving | 4 |
| PTEs finished being written | 8 |
| a TLB invalidation completed | 9 |
| a previous job on this queue finished | 11, 13 |
| a long-running queue got off the hardware | 14 |
| a hung job was declared dead | 15 |

They are all **the same primitive**: `dma_fence`. One struct, one
`->signal()`, one wait API. A frame is just a walk through that graph, and
that is why the layers compose at all.

### Idea 3 — Each layer answers exactly one question

| Layer | The one question it answers |
|---|---|
| **TTM** | Where do the bytes physically live right now? |
| **drm_gpuvm + xe_pt** | What address does the GPU use to reach them? |
| **drm_gpu_scheduler** | In what order does work run? |
| **GuC** | Which context is on the hardware at this instant? |
| **dma_fence** | Who is allowed to proceed? |

No layer answers two questions. That discipline is why fault mode
(Appendix A1) can replace gpuvm's answer without touching any of the other
four, and why Chapter 14's preempt fence could change the *meaning* of a
fence without changing the fence API.

---

## Beat B — The frame

### The worked example

A game. One VM, one render exec queue, a 4MB vertex buffer in VRAM, a
command buffer, and a user fence it polls to know the frame is done. We
follow frame #1000 — so all the cold-path work is long since finished.

### Everything userspace can ask for

The whole driver has **14 entry points** (`xe_device.c:195`). Our frame uses
exactly two of them:

```c
// xe_device.c:195 — the complete uAPI, trimmed to what this course covered
DRM_IOCTL_DEF_DRV(XE_GEM_CREATE,        xe_gem_create_ioctl,        ...)  // ch 3
DRM_IOCTL_DEF_DRV(XE_VM_CREATE,         xe_vm_create_ioctl,         ...)  // ch 6
DRM_IOCTL_DEF_DRV(XE_VM_BIND,           xe_vm_bind_ioctl,           ...)  // ch 8
DRM_IOCTL_DEF_DRV(XE_EXEC_QUEUE_CREATE, xe_exec_queue_create_ioctl, ...)  // ch 10
DRM_IOCTL_DEF_DRV(XE_EXEC,              xe_exec_ioctl,              ...)  // ch 13  <-- frame
DRM_IOCTL_DEF_DRV(XE_WAIT_USER_FENCE,   xe_wait_user_fence_ioctl,   ...)  // ch 13  <-- frame
```

That is the whole contract. Everything in fifteen chapters hangs off those
six calls.

---

## Phase A — The cold path (done once, frames 1..999 ago)

One line each. The chapter number is where to go if a line is unclear.

```c
// 1. ch 6 — xe_vm.c:2080. Allocate a VM: a drm_gpuvm interval tree plus a
//    page-table tree per tile, each root PT living in that tile's VRAM.
xe_vm_create_ioctl()

// 2. ch 3 — xe_bo.c:3515. Allocate the vertex buffer: a xe_bo wrapping a
//    ttm_buffer_object wrapping a drm_gem_object. TTM picks VRAM0.
xe_gem_create_ioctl()

// 3. ch 8 — xe_vm.c:3882. Map it at GPU VA 0x140000000. Creates a xe_vma,
//    then stages, submits and commits PTEs — written BY THE GPU using
//    MI_STORE_DATA_IMM on a kernel migration queue (ch 7).
xe_vm_bind_ioctl()

// 4. ch 10 — xe_exec_queue.c:1271. Allocate the hardware context: an LRC
//    BO holding [ ring | PPHWSP | register image ], and a
//    drm_gpu_scheduler + entity, 1:1:1 with the queue.
xe_exec_queue_create_ioctl()
```

And one step that happens lazily, on the **first** submission only:

```c
// ch 12 — xe_guc_submit.c:1233. The GuC learns this context exists.
// Note the guard: this costs an H2G round trip, so it happens once,
// not per frame.
if (!exec_queue_registered(q))
	register_exec_queue(q, GUC_CONTEXT_NORMAL);
```

**After Phase A nothing in the memory-management stack needs to run again**
unless something moves. That is the whole point of the cold/hot split.

---

## Phase B — The frame (16.6ms, 1000 times so far)

### Step 1 — Userspace writes commands. The kernel is not involved.

The app writes draw commands into a command-buffer BO it already has
mapped. No ioctl, no fence, no kernel code at all. This is why the GPU VA
from Phase A had to be *stable* — userspace baked `0x140000000` into those
commands.

### Step 2 — `XE_EXEC`

```c
// ch 13 — xe_exec.c:111
xe_exec_ioctl()
```

Inside, in order (all Chapter 13):

```c
// 1. ch 9 — xe_exec.c:271. Lock the VM and every BO it maps, with
//    ww_mutex wound-wait backoff and retry one level down.
err = xe_validation_exec_lock(&ctx, &vm_exec, &xe->val);
```

Note what was handed to it back at `xe_exec.c:121`:

```c
struct drm_gpuvm_exec vm_exec = {.extra.fn = xe_exec_fn};
```

`xe_exec_fn` is a **callback run inside the locking loop**, and the comment
above it says why:

```c
// xe_exec.c:95
/*
 * Add validation and rebinding to the drm_exec locking loop, since both can
 * trigger eviction which may require sleeping dma_resv locks.
 */
static int xe_exec_fn(struct drm_gpuvm_exec *vm_exec)
{
	// 2. ch 4 + ch 8 — is every BO still resident where its PTEs say?
	//    TTM brings back anything evicted; ch 8 rewrites those PTEs.
	//    This can itself need more locks, hence being inside the loop.
	return xe_vm_validate_rebind(vm, &vm_exec->exec, 1);
}
```

That nesting is Chapter 9's whole point in one place: validation may evict,
eviction needs sleeping locks, so validation has to be able to join the
backoff — it cannot run after the locks are settled.

Then, back in the ioctl:

```c
// 3. ch 13 — xe_exec.c:324. Is any userptr stale? If so, drop everything
//    and retry the whole ioctl. A plain BO-backed frame: no-op.
err = __xe_vm_userptr_needs_repin(vm);

// 4. ch 11 — build the job and publish its fence
xe_sched_job_arm(job);
drm_gpuvm_resv_add_fence(&vm->gpuvm, exec, &job->drm.s_fence->finished,
			 DMA_RESV_USAGE_BOOKKEEP, DMA_RESV_USAGE_BOOKKEEP);

// 5. ch 11 — hand it to the scheduler and return to userspace
xe_sched_job_push(job);
```

`XE_EXEC` returns here. The GPU has not been touched yet.

### Step 3 — The scheduler picks it up

```c
// ch 11/12 — xe_guc_submit.c:1209, on the scheduler's worker thread
static struct dma_fence *guc_exec_queue_run_job(struct drm_sched_job *drm_job)
{
	if (!exec_queue_registered(q))
		register_exec_queue(q, GUC_CONTEXT_NORMAL);   // once only (Phase A)

	q->ring_ops->emit_job(job);     // ch 10: write the ring (BB_START, fence write)
	submit_exec_queue(q, job);      // ch 12: tell the GuC

	return job->fence;              // this is what everyone will wait on
}
```

### Step 4 — Two writes reach the hardware

```c
// ch 10 — xe_guc_submit.c:1161. Publish the new ring tail in the LRC, so
// the hardware knows how far there is work to run.
xe_lrc_set_ring_tail(lrc, lrc->ring.tail);

// ch 12 — then a two-dword H2G: "context N has work."
// This is the entire submission protocol.
action[len++] = XE_GUC_ACTION_SCHED_CONTEXT;
action[len++] = q->guc->id;
xe_guc_ct_send(&guc->ct, action, len, g2h_len, num_g2h);
```

Everything in Chapters 10 and 12 exists so that these two writes are all
that a frame costs.

### Step 5 — The GPU runs, and the fence comes home

The GPU executes the ring, reaches the `MI_STORE_DWORD` the ring ops
appended, and writes the seqno into the LRC. That triggers an interrupt,
`job->fence` signals, and the graph unwinds:

```
  job->fence signals
     ├─► ch 13: the user fence worker writes the value into the app's memory
     ├─► ch 4:  TTM may now evict those BOs (the BOOKKEEP fence is signalled)
     └─► ch 11: credits return to the scheduler; the next frame can submit
```

The last handoff is a nice one — the kernel writes *directly into the
application's address space*:

```c
// ch 13 — xe_sync.c:76
static void user_fence_worker(struct work_struct *w)
{
	WRITE_ONCE(ufence->signalled, 1);
	if (mmget_not_zero(ufence->mm)) {
		kthread_use_mm(ufence->mm);              // borrow the app's mm
		copy_to_user(ufence->addr, &ufence->value, sizeof(ufence->value));
		kthread_unuse_mm(ufence->mm);
		mmput(ufence->mm);
	}
}
```

The app's poll loop sees its value appear, and frame #1000 is done.

---

## The map

Every chapter, and what it contributed to that frame:

| Ch | Subject | Its role in the frame |
|---|---|---|
| 1 | device / tile / GT | which tile's VRAM, which GT's engines |
| 2 | memory types | `XE_PL_VRAM0` was chosen for the vertex buffer |
| 3 | the BO | the three-layer struct the buffer actually is |
| 4 | TTM moves | step 2.3 — bring back anything evicted since last frame |
| 5 | GGTT vs PPGTT | the LRC lives in GGTT; the vertex buffer in PPGTT |
| 6 | VM and VMA | the interval tree that owns VA `0x140000000` |
| 7 | page tables | the PTE values the GPU walks to reach the buffer |
| 8 | VM_BIND | wrote those PTEs, once, in Phase A |
| 9 | locking + TLB | step 2.1's `drm_exec` retry loop |
| 10 | exec queue + LRC | the ring the commands were written into |
| 11 | drm_gpu_scheduler | ordering, credits, the three fences |
| 12 | the GuC | the two-dword doorbell in step 4 |
| 13 | EXEC | steps 2 and 5 |
| 14 | LR mode | **not used** — this is a dma-fence VM |
| 15 | TDR / reset | **not used** — the frame finished in time |

Note the last two. For a healthy frame, Chapters 14 and 15 contribute
*nothing*. They are the machinery for when the assumptions in Chapters 1–13
stop holding.

## Where each step can break

| Step | What can go wrong | Who handles it |
|---|---|---|
| 2.1 lock | contention | ww_mutex backs off and retries (ch 9) |
| 2.2 userptr | app moved its pages | repair, then retry the ioctl (ch 13) |
| 2.3 validate | buffer was evicted | TTM moves it back, PTEs rewritten (ch 4, 8) |
| 2.3 validate | no memory anywhere | eviction, or a preempt fence if some VM is long-running (ch 14) |
| 4 doorbell | GuC not answering | 5s timeout, then GT reset (ch 15) |
| 5 GPU runs | shader never finishes | TDR: check real runtime, ban the queue (ch 15) |
| 5 GPU runs | bad address | CAT error G2H, routed into the TDR (ch 15) |

Every failure mode has exactly one owner. No two layers try to fix the same
thing — which is the Idea-3 discipline showing up again, in the error paths.

---

## The one-page mental model

```
  userspace                 draws into a BO it already has mapped
      │
      │ XE_EXEC
      ▼
  xe_exec.c         lock everything (ch 9) -> validate (ch 4) -> build job
      │
      ▼
  drm_gpu_scheduler ordering, credits, dependencies (ch 11)
      │
      ▼
  xe_lrc            write the ring; publish the tail (ch 10)
      │
      ▼
  xe_guc_ct         2 dwords: "context N has work" (ch 12)
      │
      ▼
  GuC firmware      picks a context, loads its LRC onto an engine
      │
      ▼
  GPU               walks PPGTT (ch 7) to reach memory TTM placed (ch 2-4)
      │                at the VA that VM_BIND mapped (ch 6, 8)
      ▼
  seqno write ──► interrupt ──► job->fence signals
      │
      ├─► user fence written into the app's memory  (ch 13)
      ├─► BOs become evictable again                (ch 4)
      └─► credits freed; next frame may submit      (ch 11)
```

Read top to bottom, that is the entire driver on a good day. Chapters 14
and 15 are what happens when it isn't one.

---

## What to carry forward

1. **The cold/hot split is the design.** Page tables, LRCs and GuC
   registration are built once. A frame costs a lock dance, a validate, a
   ring write and two dwords. Every chapter about memory management was
   really about making the hot path short.
2. **One primitive, everywhere.** `dma_fence` is how moving memory, writing
   PTEs, invalidating TLBs, finishing jobs, preempting queues and killing
   hung work all talk to each other. Learn it once, and the layers stop
   looking like separate subsystems.
3. **One question per layer.** TTM: where. gpuvm: what address. drm_sched:
   what order. GuC: what now. dma_fence: who may proceed. When a new
   feature arrives — fault mode, SVM, LR mode — it changes exactly one of
   those answers, and that is why it fits.

---

That is the course. Remaining optional material:

* **Appendix A1** — userptr, SVM and page faults (`xe_svm.c`,
  `xe_pagefault.c`): the mechanism that replaces gpuvm's answer with
  "fault it in on demand". The three userptr Q&As from this course fold in
  here.
* **Appendix A2** — pinning, suspend/resume, dma-buf (`xe_bo_evict.c`,
  `xe_dma_buf.c`): what happens to all this memory when the machine
  sleeps, and how buffers cross driver boundaries.
