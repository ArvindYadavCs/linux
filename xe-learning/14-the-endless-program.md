# Chapter 14 — The Program That Never Ends

> Act III, part 1. Long-running mode, preempt fences, and the rebind worker.

---

## Beat A — The story

### Idea 1 — Everything so far assumed "every job ends"

Chapters 11–13 rested on one promise: **a job finishes**. That promise is
load-bearing. It is what makes eviction possible:

```
TTM wants to move a BO
  -> wait on the BO's dma_resv fences
     -> they signal, because jobs end
        -> now it is safe to move the pages
```

Now a compute application shows up with a kernel that runs for an hour.
Or forever. The fence for that job will not signal today.

If we put that fence in `dma_resv`, the whole machine seizes: eviction
blocks forever, memory pressure cannot be relieved, `fork()` of another
process stalls behind the shrinker. One infinite kernel takes down the box.

So long-running mode (**LR mode**) makes a trade: the job's completion fence
is **never published**. Nothing in the kernel is allowed to wait on it.

But that leaves a hole. If nothing can wait on the work, how does TTM
ever evict a buffer the GPU is using?

### Idea 2 — Publish a different promise: "I got out of the way"

Instead of a fence that means *my work finished*, hang a fence that means
**my queue is off the hardware**.

That promise *can* be kept on demand, because stopping is always possible
even when finishing is not.

And here is the trick that makes the whole design click:

> **The act of waiting on the fence is what triggers the retreat.**

Nobody calls a "preempt the GPU" function. TTM just waits on `dma_resv`
like it always has. `dma_fence` calls the fence's `enable_signaling` hook
the first time someone cares — and *that hook* sends the suspend.

So the eviction path needs no new code at all. It waits on a fence; the
fence makes itself true.

### Idea 3 — Retreating is easy; coming back needs a worker

After the retreat the world is broken: the queue is suspended, the BO moved
somewhere else, its PTEs now point at the old address, maybe some userptr
pages were reclaimed in the meantime. Someone must:

1. re-pin invalidated userptrs,
2. re-validate every BO (bring back what was evicted),
3. rewrite the page tables,
4. resume the queues,
5. **hang fresh preempt fences for next time.**

That is one worker per VM: `vm->preempt.rebind_work`. The signalling fence
queues it as its last act.

So preemption is a **cycle**, not an event:

```
   arm fence -> run -> someone waits -> suspend -> signal -> evict
        ^                                                     |
        +------------- rebind worker puts it back <------------+
```

---

## Beat B — The code

### The worked example

An OpenCL kernel is running an endless loop over `buf`, a 64MB BO, on an
exec queue `q` attached to an LR VM. Another process allocates heavily;
TTM picks `buf` as its eviction victim.

Follow that one eviction all the way around the cycle.

### Named functions

| Function | Where | Job |
|---|---|---|
| `xe_vm_add_compute_exec_queue()` | `xe_vm.c:232` | hang the first preempt fence |
| `preempt_fence_enable_signaling()` | `xe_preempt_fence.c:70` | **the trap** — waiting starts the suspend |
| `__guc_exec_queue_process_msg_suspend()` | `xe_guc_submit.c:1882` | tell the GuC to deschedule |
| `preempt_fence_work_func()` | `xe_preempt_fence.c:15` | wait for the stop, signal, kick the worker |
| `preempt_rebind_work_func()` | `xe_vm.c:477` | put the world back |

---

### Step 0 — Setup: the queue joins the VM and gets its first fence

An LR queue does not just exist; it is *registered* with the VM, because the
VM must be able to find every queue it may need to preempt.

```c
// xe_vm.c:252
pfence = xe_preempt_fence_create(q, q->lr.context, ++q->lr.seqno);

// xe_vm.c:259 — the VM's roster of preemptible queues
list_add(&q->lr.link, &vm->preempt.exec_queues);
++vm->preempt.num_exec_queues;
q->lr.pfence = pfence;

// xe_vm.c:263 — publish it where eviction will trip over it.
// BOOKKEEP: "wait for me before moving memory", but it is NOT a
// dependency for new work. That distinction is the whole design.
drm_gpuvm_resv_add_fence(&vm->gpuvm, exec, pfence,
			 DMA_RESV_USAGE_BOOKKEEP, DMA_RESV_USAGE_BOOKKEEP);
```

The same fence is added to every external BO's `dma_resv` too, so an
eviction of any mapped buffer hits it:

```c
// xe_vm.c:190 (add_preempt_fences)
list_for_each_entry(q, &vm->preempt.exec_queues, lr.link)
	if (q->lr.pfence)
		dma_resv_add_fence(bo->ttm.base.resv, q->lr.pfence,
				   DMA_RESV_USAGE_BOOKKEEP);
```

### Step 1 — What EXEC deliberately does *not* do

Chapter 13's exec path is riddled with `if (!xe_vm_in_lr_mode(vm))`. Each
one is the same decision: **do not let anything wait on this job.**

```c
// xe_exec.c:303 — do not even depend on KERNEL ops; the preempt fence
// already orders us against eviction.
if (!xe_vm_in_lr_mode(vm)) {
	drm_exec_for_each_locked_object(exec, obj)
		xe_sched_job_add_deps(job, obj->resv, DMA_RESV_USAGE_KERNEL);
}

xe_sched_job_arm(job);

// xe_exec.c:334 — THE important one. In LR mode the finished fence is
// never published, so no shrinker, no eviction, no other driver can
// ever block on a kernel that does not return.
if (!xe_vm_in_lr_mode(vm))
	drm_gpuvm_resv_add_fence(&vm->gpuvm, exec, &job->drm.s_fence->finished,
				 DMA_RESV_USAGE_BOOKKEEP, DMA_RESV_USAGE_BOOKKEEP);

// xe_exec.c:344 — no last_fence either: "wait for the previous job"
// has no meaning when jobs do not end.
if (!xe_vm_in_lr_mode(vm))
	xe_exec_queue_last_fence_set(q, vm, &job->drm.s_fence->finished);

xe_sched_job_push(job);
xe_vm_reactivate_rebind(vm);   // there is work now -> the worker matters again
```

The userptr check from Chapter 13 is skipped too (`xe_exec.c:319`) — in LR
mode the rebind worker owns userptr repair, not exec.

### Step 2 — Pressure: the wait springs the trap

TTM selects `buf`, and does the ordinary thing: wait on its `dma_resv`.
`dma_fence` lazily calls `enable_signaling` on each unsignalled fence:

```c
// xe_preempt_fence.c:70 — called by dma_fence the FIRST time anyone waits.
static bool preempt_fence_enable_signaling(struct dma_fence *fence)
{
	struct xe_preempt_fence *pfence = container_of(fence, ..., base);
	struct xe_exec_queue *q = pfence->q;

	pfence->error = q->ops->suspend(q);   // <-- send SUSPEND to the GuC

	// Remember we really did suspend. A suspend() that FAILED (banned,
	// wedged queue) must never be paired with a resume() later.
	if (!pfence->error)
		WRITE_ONCE(q->lr.suspended, true);

	// suspend() only *asks*. Waiting for it to take effect can sleep,
	// and we are in fence-signalling context -> hand off to a worker.
	queue_work(q->vm->xe->preempt_fence_wq, &pfence->preempt_work);
	return true;
}
```

Read that again: **the eviction path contains no preemption logic.** It
waits on a fence. The fence makes itself come true.

### Step 3 — The suspend travels as a scheduler message

`suspend()` does not touch registers. It queues a message onto the queue's
own `drm_gpu_scheduler`, so it serialises against that queue's job
submission for free (Chapter 12's mechanism, reused):

```c
// xe_guc_submit.c:2238
static bool __guc_exec_queue_suspend(struct xe_exec_queue *q)
{
	struct xe_sched_msg *msg = ge->static_msgs + STATIC_MSG_SUSPEND;

	xe_sched_msg_lock(sched);
	first = (++ge->suspend_count == 1);   // refcounted: many waiters, one suspend
	if (first) {
		guc_exec_queue_try_add_msg(q, msg, SUSPEND);
		ge->suspend_pending = true;       // the "not stopped yet" flag
	}
	xe_sched_msg_unlock(sched);
	return first;
}
```

When the scheduler thread picks the message up:

```c
// xe_guc_submit.c:1882
static void __guc_exec_queue_process_msg_suspend(struct xe_sched_msg *msg)
{
	if (... && !exec_queue_suspended(q) && exec_queue_enabled(q)) {
		// Do not suspend mid-resume; wait for any pending enable first.
		wait_event(guc->ct.wq, ... !exec_queue_pending_disable(q));

		// ANTI-THRASH. Under steady memory pressure we would otherwise
		// suspend the queue the instant it resumes, and the app would
		// make zero forward progress. Guarantee it a minimum slice.
		s64 since_resume_ms = ktime_ms_delta(ktime_get(), q->guc->resume_time);
		s64 wait_ms = q->vm->preempt.min_run_period_ms - since_resume_ms;
		if (wait_ms > 0 && q->guc->resume_time)
			xe_sleep_relaxed_ms(wait_ms);

		set_exec_queue_suspended(q);
		disable_scheduling(q, false);     // H2G: SCHED_CONTEXT disable
	}
}
```

`min_run_period_ms` is the fairness valve. Without it, preemption under
pressure becomes a livelock: evict, resume, evict, resume, no work done.

### Step 4 — The GuC acks; the fence signals

The GuC stops running the context and sends a G2H. That clears the flag and
wakes the waiter:

```c
// xe_guc_submit.c:1849
static void __suspend_fence_signal(struct xe_exec_queue *q)
{
	WRITE_ONCE(q->guc->suspend_pending, false);
	wake_up(&q->guc->suspend_wait);
}
```

Meanwhile the worker queued back in Step 2 has been sitting in
`suspend_wait`:

```c
// xe_preempt_fence.c:15
static void preempt_fence_work_func(struct work_struct *w)
{
	bool cookie = dma_fence_begin_signalling();   // lockdep: signalling critical section

	if (pfence->error) {
		dma_fence_set_error(&pfence->base, pfence->error);
	} else if (!q->ops->reset_status(q)) {
		int err = q->ops->suspend_wait(q);        // sleeps until actually off the HW

		if (err == -EAGAIN) {                     // SR-IOV VF migration: try again
			queue_work(q->vm->xe->preempt_fence_wq, &pfence->preempt_work);
			dma_fence_end_signalling(cookie);
			return;
		}
		if (err)
			dma_fence_set_error(&pfence->base, err);
	} else {
		dma_fence_set_error(&pfence->base, -ENOENT);  // queue got reset; nothing to stop
	}

	dma_fence_signal(&pfence->base);   // <-- TTM's wait returns. Eviction proceeds.

	xe_vm_queue_rebind_worker(q->vm);  // and the cycle must be closed
	xe_exec_queue_put(q);
	dma_fence_end_signalling(cookie);
}
```

Note that the fence signals **even on error**. A preempt fence that never
signals would be exactly the deadlock LR mode exists to avoid, so failures
are reported through `dma_fence_set_error()` and the VM is killed later —
never by hanging.

`buf` is now free to move. TTM evicts it to system memory. Its PTEs are
stale, the queue is suspended, and `vm->preempt.rebind_work` is queued.

### Step 5 — The rebind worker puts the world back

One function, one retry loop. This is the heart of LR mode.

```c
// xe_vm.c:477
static void preempt_rebind_work_func(struct work_struct *w)
{
	struct xe_vm *vm = container_of(w, struct xe_vm, preempt.rebind_work);

	down_write(&vm->lock);
	if (xe_vm_is_closed_or_banned(vm)) { up_write(&vm->lock); return; }

retry:
	// (a) system suspend in progress? park ourselves on a list and bail;
	//     xe_vm_resume_rebind_worker() re-queues us on PM resume.
	if (!try_wait_for_completion(&vm->xe->pm_block) && vm_suspend_rebind_worker(vm)) {
		up_write(&vm->lock);
		xe_pm_might_block_on_suspend();
		return;
	}

	// (b) Chapter's-worth of Ch 13 recap: fix any userptrs the MMU
	//     notifier invalidated while we were stopped.
	if (xe_vm_userptr_check_repin(vm)) {
		err = xe_vm_userptr_pin(vm);
		if (err) goto out_unlock_outer;
	}

	// (c) lock the VM and every BO it maps (ww_mutex retry loop, Ch 9)
	drm_exec_until_all_locked(&exec) {
		bool done = false;
		err = xe_preempt_work_begin(&exec, vm, &done);
		drm_exec_retry_on_contention(&exec);
		xe_validation_retry_on_oom(&ctx, &err);
		if (err || done) { ...; goto out_unlock_outer; }
	}

	// (d) one spare fence per queue, allocated BEFORE the point of no return
	err = alloc_preempt_fences(vm, &preempt_fences, &fence_count);
	if (err) goto out_unlock;

	// (e) rewrite the page tables for everything on vm->rebind_list —
	//     evicted BOs at their new addresses, re-pinned userptrs at
	//     their new DMA addresses.
	err = xe_vm_rebind(vm, true);
	if (err) goto out_unlock;

	// (f) the PTE writes are themselves GPU work. Wait for them.
	wait = dma_resv_wait_timeout(xe_vm_resv(vm), DMA_RESV_USAGE_KERNEL,
				     false, MAX_SCHEDULE_TIMEOUT);
	if (wait <= 0) { err = -ETIME; goto out_unlock; }

	// (g) LAST CHANCE. Under the notifier lock: did a userptr go stale
	//     while we were doing all of the above? (Ch 13's seqno check.)
	xe_svm_notifier_lock(vm);
	if (retry_required(tries, vm)) {
		xe_svm_notifier_unlock(vm);
		err = -EAGAIN;              // start over from `retry:`
		goto out_unlock;
	}

	// everything we just validated is hot; move it to the LRU tail
	spin_lock(&vm->xe->ttm.lru_lock);
	ttm_lru_bulk_move_tail(&vm->lru_bulk_move);
	spin_unlock(&vm->xe->ttm.lru_lock);

	/* Point of no return. */
	arm_preempt_fences(vm, &preempt_fences);           // fresh fences for next time
	resume_and_reinstall_preempt_fences(vm, &exec);    // and let the GPU run again
	xe_svm_notifier_unlock(vm);
	...
}
```

The ordering in (d)–(g) is the careful part. Everything that can fail —
allocation, locking, binding, waiting — happens **before** the point of no
return. After `arm_preempt_fences()` there is no error path, because a
half-armed VM has queues that can never be preempted again.

Inside the step that closes the loop:

```c
// xe_vm.c:157 — new fence per queue, new seqno. The old one has signalled
// and is useless; a signalled fence protects nothing.
list_for_each_entry(q, &vm->preempt.exec_queues, lr.link) {
	fence = xe_preempt_fence_arm(to_preempt_fence_from_link(list->next),
				     q, q->lr.context, ++q->lr.seqno);
	dma_fence_put(q->lr.pfence);
	q->lr.pfence = fence;
}
```

```c
// xe_vm.c:200
list_for_each_entry(q, &vm->preempt.exec_queues, lr.link) {
	// Only resume what we actually suspended, and skip queues that got
	// reset/banned meanwhile — teardown owns those now.
	if (READ_ONCE(q->lr.suspended) && !q->ops->reset_status(q)) {
		WRITE_ONCE(q->lr.suspended, false);
		q->ops->resume(q);      // RESUME msg -> SCHED_CONTEXT enable
	}
	// publish the NEW fence, so the next eviction trips the trap again
	drm_gpuvm_resv_add_fence(&vm->gpuvm, exec, q->lr.pfence,
				 DMA_RESV_USAGE_BOOKKEEP, DMA_RESV_USAGE_BOOKKEEP);
}
```

And resume is the mirror image of suspend — refcounted, message-driven:

```c
// xe_guc_submit.c:2264
last = (--ge->suspend_count == 0);
if (last)
	guc_exec_queue_try_add_msg(q, msg, RESUME);
```

```c
// xe_guc_submit.c:1917
clear_exec_queue_suspended(q);
if (!exec_queue_enabled(q)) {
	q->guc->resume_time = RESUME_PENDING;   // starts the min_run_period clock
	enable_scheduling(q);
}
```

The kernel resumes **exactly where it stopped** — the LRC still holds its
register image and ring tail (Chapter 10). Preemption never lost any work.

### Step 6 — The three ways out of the worker

```c
out_unlock_outer:
	if (err == -EAGAIN) {
		goto retry;             // (1) something moved under us; go again
	}
	if (err) {
		drm_warn(&vm->xe->drm, "VM worker error: %d\n", err);
		xe_vm_kill(vm, true);   // (2) unrecoverable -> kill the VM
	}
	up_write(&vm->lock);
```

And the quiet third exit, decided up front:

```c
// xe_vm.c:413 (xe_preempt_work_begin)
if (xe_vm_is_idle(vm)) {
	vm->preempt.rebind_deactivated = true;   // (3) nothing running; stop spinning
	*done = true;
	return 0;
}
if (!preempt_fences_waiting(vm)) {
	*done = true;                            // nobody asked to preempt; no work
	return 0;
}
```

`rebind_deactivated` is undone by the next submission —
`xe_vm_reactivate_rebind()` at the end of exec (`xe_exec.c:347`). An idle
LR VM costs nothing.

---

## The three VM modes, side by side

```c
// xe_vm.h:218
xe_vm_in_fault_mode(vm)          // FAULT_MODE: bind on demand from page faults
xe_vm_in_lr_mode(vm)             // LR_MODE:    job fences never published
xe_vm_in_preempt_fence_mode(vm)  // LR && !FAULT: this chapter
```

| | dma-fence VM (Ch 13) | **preempt-fence VM (this ch.)** | fault-mode VM (A1) |
|---|---|---|---|
| Jobs end? | yes | no | no |
| Job fence in `dma_resv`? | yes, BOOKKEEP | **never** | never |
| How does eviction proceed? | wait for the job | **suspend the queue** | unmap + re-fault |
| Who repairs userptrs? | exec's `retry:` | **the rebind worker** | the fault handler |
| Who rewrites PTEs? | exec / bind ioctl | **the rebind worker** | the fault handler |

---

## What to carry forward

1. **A fence does not have to promise completion.** It can promise
   *vacancy*. That one substitution is what lets an endless kernel coexist
   with a memory manager that must be able to move things.
2. **`enable_signaling` is an action hook, not a notification.** Waiting on
   a preempt fence *causes* the preemption. The eviction path needed zero
   new code.
3. **Every retreat needs a return path**, and the return path is where all
   the difficulty lives: re-pin, re-validate, re-bind, verify nothing moved
   again, and only then re-arm and resume — in that order, with every
   failure handled before the point of no return.

Next: Chapter 15 — when the GPU stops answering. TDR, reset, ban, wedge.
