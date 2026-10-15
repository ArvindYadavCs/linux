# Chapter 13 — EXEC, End to End

> Beat A: the story.

This is the chapter where everything meets. `EXEC` is how userspace says *"run
this"*, and it touches every piece of both Acts.

It is also, in Xe, **surprisingly small** — 382 lines for the whole file. And
that's the story.

> *Userspace: "run the batch buffer at GPU address `0x200000000` on queue 7,*
> *wait for fence A first, and signal fence B when done."*

Three ideas, and all three are things Xe **removed**.

---

## Idea 1: No list of buffers

In older drivers, submitting work meant handing the kernel **a list of every
buffer the work would touch**. The kernel then read and wrote each buffer's
fences to work out the ordering automatically.

Xe does not do this. **`EXEC` takes no buffer list at all.** Look at the whole
request:

```c
struct drm_xe_exec {
	__u32 exec_queue_id;   // which queue
	__u32 num_syncs;       // how many fences
	__u64 syncs;           // the fences (in and out)
	__u64 address;         // the batch buffer's GPU address
	__u16 num_batch_buffer;
};
```

**One address and some fences.** That's the entire submission.

If your work writes a buffer and the next piece of work reads it, **you** must
pass a fence between them. The kernel no longer guesses.

Why is that better? Because the old way meant walking a list of maybe thousands
of buffers on *every* submission, locking each one, reading its fences. It was
the single biggest cost in the submission path. Xe hands that responsibility to
userspace, which usually already knows the answer.

---

## Idea 2: No binding at exec time

Older drivers would let you submit work that *implicitly* mapped buffers —
"here's a buffer, make it addressable, then run this". That meant page table
work in the middle of the submission path.

Xe splits them cleanly:

```
   VM_BIND  -> make things addressable   (Chapter 8)
   EXEC     -> run work that uses them   (this chapter)
```

Two separate ioctls, and — this is the elegant part — **they use the same fence
interface**. If your `EXEC` depends on a `VM_BIND`, you pass the bind's out-fence
as the exec's in-fence. Exactly as you would between two execs.

So from userspace's point of view, a bind *is* just another piece of work to
order against. There is nothing special about it.

### Except: rebinds

One thing does still happen at exec time, and it's Act I coming back.

Remember Chapter 4: when the kernel evicts a buffer or a userptr's pages move,
it **marks the mappings stale and moves on**. Someone has to fix them.

That someone is `EXEC`. Before building the job, it re-establishes any mapping
that went stale since last time. That's unavoidable — the kernel broke them, so
the kernel must repair them — but note it's *repair*, not new work. Userspace
never asks for it.

---

## Idea 3: No ring flow control

The third thing removed, and we already built it.

The ring buffer is finite (Chapter 10). What happens when userspace submits more
work than fits?

Older drivers handled this inside the submission path: check for space, and if
there isn't any, wait for the GPU to drain some. Blocking, in an ioctl.

Xe doesn't check at all. Chapter 10's line:

```c
xe_sched_init(..., xe_lrc_ring_size() / MAX_JOB_SIZE_BYTES, ...)
```

gave `drm_sched` a credit limit equal to the ring's capacity. So **`drm_sched`
simply won't call `run_job()` when the ring is full** — the job waits in the
waiting room instead (Chapter 11).

`EXEC` creates the job and pushes it. Flow control happens somewhere else,
automatically, by a mechanism that was already there for dependencies.

---

## Why this matters

The kernel's own comment in `xe_exec.c` says it plainly after listing all
three:

> *"All of this results in a rather simple exec implementation."*

And that's the lesson of the chapter. The hot path of a GPU driver got small
because three responsibilities were moved out of it:

| Responsibility | Moved to |
|---------------|----------|
| working out ordering between jobs | **userspace**, via explicit fences |
| making buffers addressable | **`VM_BIND`**, a separate ioctl |
| waiting for ring space | **`drm_sched`**, via credit limits |

---

## The shape of it

```
   EXEC ioctl
     |
   1. look up the queue, parse the fences
     |
   2. repair anything Act I broke        <-- rebinds (Idea 2's exception)
     |
   3. lock and validate the VM's buffers
     |
   4. create the job        (Chapter 11)
     |
   5. collect dependencies:
        - the fences userspace listed
        - kernel operations still in flight
     |
   6. arm the job, hand out its "finished" fence
     |
   7. push it                            --> the waiting room (Chapter 11)
                                         --> the GuC (Chapter 12)
     |
   return 0
```

## Remember these three

1. **No buffer list.** One batch address and some fences. Ordering is
   userspace's job.
2. **No binding at exec time** — `VM_BIND` is separate, and they chain with
   fences. The only exception is repairing mappings the kernel itself broke.
3. **No ring flow control** — `drm_sched`'s credit limit already does it.

---
---

# Chapter 13 — Beat B: The Code

One function, `xe_exec_ioctl()` at **`xe_exec.c:111`**. We walk it top to
bottom in seven steps.

> Code trimmed: error gotos, PXP, parallel submission and the engine-group
> mode dance removed. `//` comments are mine.

---

## Step 1 — Look up and parse

```c
	q = xe_exec_queue_lookup(xef, args->exec_queue_id);
	//  ^^^ Chapter 1's xe_file->exec_queue.xa xarray

	if (XE_IOCTL_DBG(xe, q->flags & EXEC_QUEUE_FLAG_VM)) {
		err = -EINVAL;        // <<< you may not EXEC on a BIND queue
	}                             //     (Chapter 8's vm->q[] is off limits)

	if (XE_IOCTL_DBG(xe, q->ops->reset_status(q))) {
		err = -ECANCELED;     // this queue hung and was reset (ch 15)
	}

	vm = q->vm;                   // the queue already knows its VM (ch 10)

	for (num_syncs = 0; num_syncs < args->num_syncs; num_syncs++) {
		err = xe_sync_entry_parse(xe, xef, &syncs[num_syncs],
					  &syncs_user[num_syncs], ...);
		//  ^^^ each sync is either an IN fence (wait for it) or an
		//      OUT fence (signal it). Idea 1's entire interface.
	}
```

Notice there is **no buffer list loop here.** In an older driver this is where
you'd find hundreds of lines copying and resolving a BO list. Idea 1, as absent
code.

---

## Step 2 — Repair what Act I broke

```c
retry:
	if (!xe_vm_in_lr_mode(vm) && xe_vm_userptr_check_repin(vm)) {
		err = down_write_killable(&vm->lock);     // need the write lock
		write_locked = true;
	} else {
		err = down_read_interruptible(&vm->lock); // read lock is enough
		write_locked = false;
	}

	if (write_locked) {
		err = xe_vm_userptr_pin(vm);
		//  ^^^ re-pin userptr pages that moved since last exec.
		//      Also DISCOVERS which ones were invalidated.
		downgrade_write(&vm->lock);     // got what we needed; downgrade
	}
```

Two things worth seeing:

* **The lock is usually taken for reading.** Only when there's actual repair to
  do does it escalate to a write lock — and it immediately downgrades. Execs
  from many threads on one VM run concurrently in the common case.
* **`retry:`** — this whole function can restart, and we'll see why at step 6.

---

## Step 3 — Lock and validate

```c
	if (!xe_vm_in_lr_mode(vm)) {
		vm_exec.vm = &vm->gpuvm;
		vm_exec.flags = DRM_EXEC_INTERRUPTIBLE_WAIT;
		err = xe_validation_exec_lock(&ctx, &vm_exec, &xe->val);
		//  ^^^ CHAPTER 9's drm_exec. Locks the VM and every external
		//      buffer, with the wound-wait retry loop.
	}
```

And the callback it runs inside that loop (`xe_exec.c:95`):

```c
static int xe_exec_fn(struct drm_gpuvm_exec *vm_exec)
{
	struct xe_vm *vm = container_of(vm_exec->vm, struct xe_vm, gpuvm);

	xe_vm_set_validation_exec(vm, &vm_exec->exec);
	ret = xe_vm_validate_rebind(vm, &vm_exec->exec, 1);
	//    ^^^ TWO JOBS IN ONE CALL:
	//      "validate"  = CHAPTER 4. Make every buffer resident, migrating
	//                    and evicting as needed.
	//      "rebind"    = CHAPTER 8. Re-bind the mappings step 2 found
	//                    stale, and the ones eviction marked.
	xe_vm_set_validation_exec(vm, NULL);
	return ret;
}
```

**That one function call is Act I, invoked from Act II.** Chapter 4's validation
and Chapter 8's binding, both happening because `EXEC` needs them to.

The comment above it explains why it's a callback rather than straight-line
code: *"both can trigger eviction which may require sleeping dma_resv locks"* —
so it has to run **inside** the `drm_exec` retry loop, where a lock collision
can restart everything.

---

## Step 4 — Create the job

```c
	job = xe_sched_job_create(q, xe_exec_queue_is_parallel(q) ?
				  addresses : &args->address);
	//                                  ^^^^^^^^^^^^^^ Idea 1: the ONE
	//    piece of information userspace gave us about the work itself.
```

Chapter 11's job creation: allocates the `scheduled` and `finished` fences and
pre-allocates the hardware fence.

---

## Step 5 — Collect dependencies

Two sources, exactly as Chapter 11 described.

**Implicit — kernel operations still in flight:**

```c
	if (!xe_vm_in_lr_mode(vm)) {
		struct drm_gem_object *obj;

		drm_exec_for_each_locked_object(exec, obj) {
			err = xe_sched_job_add_deps(job, obj->resv,
						    DMA_RESV_USAGE_KERNEL);
			//                          ^^^^^^^^^^^^^^^^^^^^^^
			//  KERNEL only! Chapter 3's four usage tags.
			//  We wait for the driver's own operations - the
			//  rebinds from step 3, evictions, clears - but NOT
			//  for other users' reads and writes.
			//  THAT is Idea 1: userspace orders itself.
		}
	}
```

**Read that `DMA_RESV_USAGE_KERNEL` carefully.** It is Idea 1 in one argument. An
older driver would pass `READ` or `WRITE` here and inherit every other client's
work. Xe asks only for *kernel* fences — the memory-management operations it
started itself.

**Explicit — what userspace listed:**

```c
	for (i = 0; i < num_syncs && !err; i++)
		err = xe_sync_entry_add_deps(&syncs[i], job);
	//      ^^^ the IN fences. "Wait for fence A."
```

---

## Step 6 — The point of no return

```c
	if (!xe_vm_in_lr_mode(vm)) {
		err = xe_svm_notifier_lock_interruptible(vm);
		err = __xe_vm_userptr_needs_repin(vm);
		if (err)
			goto err_repin;     // <<< goes to `retry:` via -EAGAIN
		//  ^^^ LAST CHECK: did any userptr get invalidated while we
		//      were doing all of the above? If so, everything we just
		//      validated may be stale. Drop the locks and start over.
	}

	/*
	 * Point of no return, if we error after this point just set an error on
	 * the job and let the DRM scheduler / backend clean up the job.
	 */
	xe_sched_job_arm(job);
	//  ^^^ CHAPTER 11's arm. Before this line, failure means "free the job".
	//      After it, the fences are real and visible - failure means
	//      "submit it anyway, marked as failed".
```

**That comment is the hinge of the whole function.** Everything before it is
undoable; nothing after it is. It's Chapter 11's `init`/`arm` split, with the
reason spelled out.

The check just above it is also the answer to step 2's `retry:` label. Userptr
pages can be invalidated by the CPU at any moment, including while we hold no
relevant lock. So the last thing we do before committing is ask "is any of this
stale?" — and if so, throw it all away and begin again.

---

## Step 7 — Publish and push

```c
	if (!xe_vm_in_lr_mode(vm))
		drm_gpuvm_resv_add_fence(&vm->gpuvm, exec,
					 &job->drm.s_fence->finished,
					 DMA_RESV_USAGE_BOOKKEEP,
					 DMA_RESV_USAGE_BOOKKEEP);
	//  ^^^ Put this job's "finished" fence on the VM's and every locked
	//      buffer's clipboard, as BOOKKEEP. Act I will now wait for this
	//      job before moving any of those buffers (Chapter 4).
	//      THE CONTRACT, signed.

	for (i = 0; i < num_syncs; i++) {
		xe_sync_entry_signal(&syncs[i], &job->drm.s_fence->finished);
		//  ^^^ hook up the OUT fences. "Signal fence B when done."
		//      Note it's the FINISHED fence - Chapter 11's promise,
		//      handed to userspace right now, before anything runs.
	}

	xe_sched_job_push(job);
	//  ^^^ into the waiting room (Chapter 11), which eventually calls
	//      run_job() -> the ring and the GuC (Chapter 12).

	spin_lock(&xe->ttm.lru_lock);
	ttm_lru_bulk_move_tail(&vm->lru_bulk_move);
	spin_unlock(&xe->ttm.lru_lock);
	//  ^^^ CHAPTER 4's bulk LRU: mark this whole VM's buffers as
	//      recently used, in one operation. This is where that
	//      optimisation earns its keep - once per exec, not once
	//      per buffer.
```

Four calls, and each one is a different chapter paying off:

| Call | Chapter |
|------|---------|
| `drm_gpuvm_resv_add_fence(... BOOKKEEP)` | 3 and 4 — the contract |
| `xe_sync_entry_signal(... finished)` | 11 — the promise |
| `xe_sched_job_push()` | 11 → 12 — the waiting room, then the GuC |
| `ttm_lru_bulk_move_tail()` | 4 — the bulk LRU |

Then the function returns. **The GPU may not have started yet.**

---

## The whole flow, with chapters

```
 xe_exec_ioctl()                           xe_exec.c:111
   |
 1 xe_exec_queue_lookup()                  the queue, from the xarray    [ch 1]
   xe_sync_entry_parse()                   in/out fences                 [ch 13]
   |
 2 retry:  down_read(&vm->lock)
   xe_vm_userptr_pin()                     repin moved pages           [ch 10*]
   |
 3 xe_validation_exec_lock()               lock everything             [ch 9]
     -> xe_exec_fn()
          -> xe_vm_validate_rebind()       validate + rebind         [ch 4, 8]
   |
 4 xe_sched_job_create()                   two fences + a hw fence     [ch 11]
   |
 5 xe_sched_job_add_deps(..., KERNEL)      kernel ops only            [ch 3]
   xe_sync_entry_add_deps()                userspace's in-fences
   |
 6 __xe_vm_userptr_needs_repin()           stale? -> goto retry
   xe_sched_job_arm()                      POINT OF NO RETURN         [ch 11]
   |
 7 drm_gpuvm_resv_add_fence(BOOKKEEP)      sign the contract        [ch 3, 4]
   xe_sync_entry_signal(finished)          hand out the promise       [ch 11]
   xe_sched_job_push()                     the waiting room       [ch 11, 12]
   ttm_lru_bulk_move_tail()                bulk LRU                   [ch 4]
   |
 return 0
```

Every bracket is a chapter you've already read. **That's the whole of Act I and
Act II, in one 382-line file.**

(* Chapter 10 in the *appendix* — userptr. The mechanism is there even though we
haven't covered it; it's the only forward reference in the function.)

---

## Try it

```bash
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_exec_queue_submit/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_exec/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_run/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

`job_exec` is step 7's push. `job_run` is `run_job()`. The gap between them is
the dependency wait.

## Checkpoint

1. `EXEC` passes no list of buffers. So how does the kernel know which buffers to
   lock and validate in step 3?
2. Step 5 asks for `DMA_RESV_USAGE_KERNEL` fences only. What does Xe deliberately
   *not* wait for, and whose job is that instead?
3. What is "the point of no return", and what changes on either side of it?
4. Why can this whole function jump back to `retry:` right before committing?
5. Userspace gets its out-fence before the GPU has done anything. Which fence is
   it, and why is that safe?

<details>
<summary>answers</summary>

1. From the VM. A queue is bound to one VM (Chapter 10), and the VM already
   knows all its mappings (Chapter 6's interval tree). `xe_validation_exec_lock()`
   locks the VM — which, thanks to Chapter 3's shared reservation object, covers
   every private buffer in one lock — plus each external buffer individually. No
   list needed, because the VM *is* the list.
2. It does not wait for other clients' reads and writes (`DMA_RESV_USAGE_READ` /
   `WRITE`). It waits only for the driver's own memory-management operations:
   rebinds, evictions, clears. Ordering between pieces of userspace work is
   userspace's job, via explicit in/out fences. That's Idea 1.
3. `xe_sched_job_arm()`. Before it, the job's fences have no identity and nothing
   can be waiting on them, so any error just frees the job. After it the fences
   are real and may already be visible to others, so an error must instead mark
   the job failed and let it flow through the scheduler normally.
4. Because userptr pages can be invalidated by the CPU at any time, including
   while the exec is validating. The last check before arming asks whether
   anything went stale; if so, everything validated may be wrong, so the locks
   are dropped and the whole function restarts.
5. The job's **finished** fence (Chapter 11), which exists from the moment the
   job was created. It's safe because it's a promise: `drm_sched` wires it to the
   real hardware fence once `run_job()` produces one, so it signals exactly when
   the work completes.

</details>
