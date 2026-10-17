# Chapter 15 — When the GPU Stops Answering

> Act III, part 2. TDR, engine reset, ban, GT reset, wedge.

---

## Beat A — The story

### Idea 1 — "Hung" is a guess, and a wrong guess is theft

Nothing in the hardware says *"I am stuck."* All the driver ever gets is a
timer that expired:

```c
// xe_hw_engine.c:538
hwe->eclass->sched_props.job_timeout_ms = 5 * 1000;
```

But an expired timer does **not** mean the job is hung. The timer measures
**wall clock**. The job may have spent 4.9 of those 5 seconds *queued*,
descheduled by the GuC while other contexts ran, and only 100ms actually
executing. Killing it would punish an innocent app for being unlucky in
the scheduling order.

So the first thing the timeout handler does is **refuse to believe its own
timer**. It asks the LRC how much time the job really spent *on the
hardware*, and compares that against the timeout instead.

### Idea 2 — Recovery is a ladder, climbed only as far as needed

Four rungs, each more destructive than the last:

| Rung | Action | Blast radius |
|---|---|---|
| 1 | **rearm** the timer | nothing — false alarm |
| 2 | **ban** the queue | one app's one queue |
| 3 | **reset** the GT | every queue on that GT re-submits |
| 4 | **wedge** the device | the whole device, until rebind |

The rule: *never climb higher than the evidence requires.* A shader loop in
one app's queue must not reset the GT, because that disturbs every other
process on the device. Conversely, a stuck GuC cannot be fixed by banning a
queue, so there the ladder must be climbed.

### Idea 3 — Someone must answer every outstanding fence

This is the invariant that makes everything else negotiable.

A `dma_fence` that never signals hangs whoever waits on it — forever,
across process and driver boundaries (Chapter 14's shared-BO problem, in
its worst form). So **every** path out of the handler, including the
catastrophic ones, ends with "signal everything, with an error."

Death must be announced. A silent death is worse than a loud one.

---

## Beat B — The code

### The worked example

An app submits a 3D job with an infinite shader loop on a render queue.
`job_timeout_ms` (5000) expires. Follow it.

### Named functions

| Function | Where | Job |
|---|---|---|
| `guc_exec_queue_timedout_job()` | `xe_guc_submit.c:1496` | the TDR — decides everything |
| `check_timeout()` | `xe_guc_submit.c:1369` | is it *really* hung? |
| `guc_exec_queue_stop()` / `_start()` | `:2561` / `:2825` | survive a GT reset |
| `gt_reset_worker()` | `xe_gt.c:912` | the actual reset |
| `xe_device_declare_wedged()` | `xe_device.c:1453` | the end of the line |

### The state vocabulary

Five sticky bits on `q->guc->state` describe what happened to a queue:

```c
// xe_guc_submit.c:68-75
#define EXEC_QUEUE_STATE_DESTROYED  (1 << 4)   // teardown in progress
#define EXEC_QUEUE_STATE_RESET      (1 << 6)   // GuC reset this engine
#define EXEC_QUEUE_STATE_KILLED     (1 << 7)   // userspace closed it
#define EXEC_QUEUE_STATE_WEDGED     (1 << 8)   // device is wedged
#define EXEC_QUEUE_STATE_BANNED     (1 << 9)   // WE decided it misbehaved
```

`banned` is the verdict the TDR hands down. Everything downstream — can we
resubmit? can we resume? — is a test of these bits.

---

### Step 1 — The timer fires. First, freeze this queue.

```c
// xe_guc_submit.c:1519 — the job already finished; TDR just lost the race
// with the free-job worker. Nothing is wrong.
if (test_bit(DMA_FENCE_FLAG_SIGNALED_BIT, &job->fence->flags) ||
    vf_recovery(guc))
	return DRM_GPU_SCHED_STAT_NO_HANG;

// xe_guc_submit.c:1524 — stop run_job() so no NEW job starts under us.
// Nothing may be inspected until submission is quiet.
if (xe_exec_queue_is_multi_queue(q))
	xe_guc_exec_queue_group_stop(q);
else
	xe_sched_submission_stop(sched);

// xe_guc_submit.c:1530 — only NOW is it safe to read the verdict bits
skip_timeout_check = exec_queue_reset(q) ||
	exec_queue_killed_or_banned_or_wedged(q);
```

`skip_timeout_check` means *"we already know this queue is doomed, don't
bother asking whether the timer was fair."* A queue the GuC already reset,
or that userspace already closed, doesn't deserve a careful hearing.

Before destroying any evidence, take the crash dump:

```c
// xe_guc_submit.c:1545 — capture register state while it still exists
if (!xe_device_is_in_reset(xe) && !exec_queue_killed(q) &&
    !xe->devcoredump.captured && !xe_guc_capture_get_matching_and_lock(q)) {
	CLASS(xe_force_wake, fw_ref)(gt_to_fw(q->gt), XE_FORCEWAKE_ALL);
	xe_engine_snapshot_capture_for_queue(q);
}
```

### Step 2 — Don't believe the timer

```c
// xe_guc_submit.c:1559
if (!skip_timeout_check && !check_timeout(q, job))
	goto rearm;        // <-- rung 1: false alarm, give it more time
```

And the function that earns that:

```c
// xe_guc_submit.c:1369
static bool check_timeout(struct xe_exec_queue *q, struct xe_sched_job *job)
{
	u32 timeout_ms = q->sched_props.job_timeout_ms;

	// Never even started? Then this is not the app's fault — the GuC
	// failed to schedule it. Caller will escalate to a GT reset.
	if (!xe_sched_job_started(job))
		return true;

	// The LRC's own running-time counter, maintained by the hardware.
	ctx_timestamp = lower_32_bits(xe_lrc_timestamp(q->lrc[0]));

	// Identical to last time we looked -> the context is not advancing
	// at all. Genuinely dead, but give it `threshold` chances first.
	if (ctx_timestamp == job->sample_timestamp)
		return xe_sched_invalidate_job(job, 0);

	job->sample_timestamp = ctx_timestamp;
	ctx_job_timestamp = xe_lrc_ctx_job_timestamp(q->lrc[0]);

	// THE KEY LINE: time spent RUNNING, not time since submission.
	diff = ctx_timestamp - ctx_job_timestamp;

	// +5% slack, because GuC scheduling latency is not the app's fault.
	running_time_ms = ADJUST_FIVE_PERCENT(xe_gt_clock_interval_to_ms(gt, diff));

	return running_time_ms >= timeout_ms;
}
```

`ctx_timestamp - ctx_job_timestamp` is Idea 1 in one subtraction. A job
starved by other contexts has a small difference and survives; a job
burning the GPU has a large one and dies.

### Step 3 — The verdict

Our shader loop really has been running 5s of GPU time. Sentence passed:

```c
// xe_guc_submit.c:1569 — rung 2: ban THIS QUEUE. Not the GT, not the device.
set_exec_queue_banned(q);
```

But is a ban enough? Two cases escalate to rung 3:

```c
// xe_guc_submit.c:1486
static bool timeout_needs_gt_reset(struct xe_exec_queue *q,
				   struct xe_sched_job *job,
				   bool skip_timeout_check)
{
	// A KERNEL queue (migration, page-table writes) timing out means the
	// driver's own machinery is broken. Always escalate.
	if (q->flags & EXEC_QUEUE_FLAG_KERNEL)
		return true;

	// A job that never started is a GuC/hardware problem, not an app
	// problem. Banning the app would be blaming the victim.
	return !skip_timeout_check && !xe_sched_job_started(job);
}
```

```c
// xe_guc_submit.c:1655
if (timeout_needs_gt_reset(q, job, skip_timeout_check)) {
	// Has this job already survived 2 GT resets? Then resetting again
	// is not going to help; stop trying.
	if (!xe_sched_invalidate_job(job, 2)) {
		clear_exec_queue_banned(q);   // un-blame the app
		xe_gt_reset_async(q->gt);     // rung 3
		goto rearm;
	}
	if (q->flags & EXEC_QUEUE_FLAG_KERNEL) {
		xe_gt_WARN(q->gt, true, "Kernel-submitted job timed out\n");
		wedge_device = true;          // rung 4
	}
}
```

### Step 4 — Kick it off the hardware

A banned queue must stop *executing*, not just stop being submitted to. The
GuC still thinks the context is runnable, so tell it otherwise:

```c
// xe_guc_submit.c:1573
if (!xe_device_is_in_reset(xe) && !wedged &&
    (exec_queue_enabled(primary) || exec_queue_pending_disable(primary))) {

	// Drain any in-flight enable/disable first — we must not race.
	ret = wait_event_timeout(guc->ct.wq,
				 (!exec_queue_pending_enable(primary) &&
				  !exec_queue_pending_disable(primary)) ||
				 xe_guc_read_stopped(guc) ||
				 vf_recovery(guc), HZ * 5);
	if (!ret || xe_guc_read_stopped(guc))
		goto trigger_reset;          // GuC not answering -> escalate

	disable_scheduling(primary, skip_timeout_check);   // H2G

	// Then wait for the GuC to confirm. We may NOT signal any fence
	// before the context is off the hardware, or the app could free
	// memory the GPU is still reading.
	smp_rmb();
	ret = wait_event_timeout(guc->ct.wq,
				 !xe_uc_fw_is_running(&guc->fw) ||
				 !exec_queue_pending_disable(primary) ||
				 xe_guc_read_stopped(guc), HZ * 5);
	if (!ret || xe_guc_read_stopped(guc)) {
trigger_reset:
		// The GuC itself is unresponsive. Climb the ladder.
		xe_devcoredump(primary, job, "Schedule disable failed to respond, ...");
		xe_gt_reset_async(primary->gt);
		xe_sched_tdr_queue_imm(sched);
		goto rearm;
	}
}
```

Note the ordering comment in the source: **signal nothing until scheduling
is confirmed disabled.** Fence signalling tells userspace "the GPU is done
with your buffer." Saying that while the GPU is still running is a
use-after-free waiting to happen.

### Step 5 — Answer every fence (Idea 3, in two lines)

```c
// xe_guc_submit.c:1666
xe_sched_job_set_error(job, err);                  // err == -ETIME: the culprit
drm_sched_for_each_pending_job(tmp_job, &sched->base, NULL)
	xe_sched_job_set_error(to_xe_sched_job(tmp_job), -ECANCELED);
```

Two different errors, deliberately. `-ETIME` marks the job that hung;
`-ECANCELED` marks the innocent jobs queued behind it. Userspace can tell
"you broke it" from "you were standing next to the thing that broke."

Then let submission run again and wake anyone sleeping on a user fence:

```c
// xe_guc_submit.c:1674
xe_sched_submission_start(sched);
xe_guc_exec_queue_trigger_cleanup(q);
```

```c
// xe_guc_submit.c:550 — tiny, but it is what stops xe_wait_user_fence()
// from hanging forever on a queue that just died.
static void xe_guc_exec_queue_trigger_cleanup(struct xe_exec_queue *q)
{
	wake_up_all(&xe->ufence_wq);
	xe_sched_tdr_queue_imm(&q->guc->sched);
}
```

Finally:

```c
// xe_guc_submit.c:1685
return DRM_GPU_SCHED_STAT_NO_HANG;
```

**Every** path returns `NO_HANG`, even the one that just banned a queue.
That is not a lie — it tells `drm_sched` "I have handled this myself, put
the job back on the pending list so `free_job` runs." Xe does its own
recovery and does not want the scheduler's generic version on top.

---

### Step 6 — Rung 3: the GT reset

`xe_gt_reset_async()` queues a worker (never synchronous — the TDR must not
block on it):

```c
// xe_gt.c:982
void xe_gt_reset_async(struct xe_gt *gt)
{
	if (xe_device_is_in_reset(xe))
		return;                        // already resetting

	if (!xe_fault_gt_reset() && xe_uc_reset_prepare(&gt->uc))
		return;                        // one already in flight

	xe_gt_info(gt, "reset queued from %ps\n", __builtin_return_address(0));
	queue_work(gt->ordered_wq, &gt->reset.worker);
}
```

```c
// xe_gt.c:912 — the sequence, with everything incidental removed
static void gt_reset_worker(struct work_struct *w)
{
	if (xe_device_wedged(gt_to_xe(gt)))       goto err_pm_put;  // pointless now
	if (!xe_device_uc_enabled(gt_to_xe(gt)))  goto err_pm_put;  // needs GuC

	xe_gt_sanitize(gt);                  // forget cached hardware state
	fw_ref = xe_force_wake_get(gt_to_fw(gt), XE_FORCEWAKE_ALL);

	xe_guc_rc_disable(&gt->uc.guc);      // stop RC6 racing the reset
	xe_uc_stop_prepare(&gt->uc);
	xe_pagefault_reset(gt_to_xe(gt), gt);
	xe_uc_stop(&gt->uc);                 // -> xe_guc_submit_stop(): see below
	xe_tlb_inval_reset(&gt->tlb_inval);  // abandon in-flight invalidations

	err = do_gt_reset(gt);               // the actual hardware reset
	if (err) goto err_out;

	err = do_gt_restart(gt);             // reload GuC, re-register queues
	if (err) goto err_out;

	xe_gt_info(gt, "reset done\n");
	return;

err_out:
	XE_WARN_ON(xe_uc_start(&gt->uc));
err_fail:
	xe_gt_err(gt, "reset failed (%pe)\n", ERR_PTR(err));
	xe_device_declare_wedged(gt_to_xe(gt));   // rung 4
}
```

#### What a reset does to every queue on the GT

The GuC is about to be reloaded. It will come back knowing **nothing** —
no registered contexts, no scheduling state. So every queue must forget
what the GuC used to know, while remembering the verdicts:

```c
// xe_guc_submit.c:2561
static void guc_exec_queue_stop(struct xe_guc *guc, struct xe_exec_queue *q)
{
	xe_sched_submission_stop(sched);

	// A suspend was in flight and will never be acked. Signal it, or
	// Chapter 14's preempt fence hangs forever.
	if (q->guc->suspend_pending) {
		set_exec_queue_suspended(q);
		suspend_fence_signal(q);
	}

	// KEEP the sticky verdicts; DROP everything the GuC owned
	// (REGISTERED, ENABLED, PENDING_ENABLE/DISABLE, RESET).
	atomic_and(EXEC_QUEUE_STATE_WEDGED | EXEC_QUEUE_STATE_BANNED |
		   EXEC_QUEUE_STATE_KILLED | EXEC_QUEUE_STATE_DESTROYED |
		   EXEC_QUEUE_STATE_SUSPENDED,
		   &q->guc->state);
	q->guc->resume_time = 0;

	// Ban any USER queue that had a job in flight — we cannot know
	// whether it was the cause, and we cannot re-run a partially
	// executed job safely. Kernel and VM queues are exempt: they are
	// ours, and the driver cannot function without them.
	if (!(q->flags & (EXEC_QUEUE_FLAG_KERNEL | EXEC_QUEUE_FLAG_VM))) {
		struct xe_sched_job *job = xe_sched_first_pending_job(sched);

		if (job && ((xe_sched_job_started(job) && !xe_sched_job_completed(job)) ||
			    xe_sched_invalidate_job(job, 2))) {
			set_exec_queue_banned(q);
			xe_guc_exec_queue_trigger_cleanup(q);
		}
	}
}
```

That last block is the collateral damage of rung 3, stated plainly: *any*
user queue with a started-but-unfinished job gets banned, whether or not it
caused the hang. This is why the ladder matters — a GT reset is never free.

#### And coming back: rewind the ring, re-run the jobs

```c
// xe_guc_submit.c:2825
static void guc_exec_queue_start(struct xe_exec_queue *q)
{
	if (!exec_queue_killed_or_banned_or_wedged(q)) {   // survivors only
		struct xe_sched_job *job = xe_sched_first_pending_job(sched);

		if (job) {
			for (i = 0; i < q->width; ++i) {
				// Rewind the SOFTWARE ring tail to where this job
				// began, so resubmission rewrites the same bytes
				// in the same place...
				q->lrc[i]->ring.tail = job->ptrs[i].head;
				// ...and set the LRC's tail to its head, so the
				// freshly loaded GuC only ever sees the tail
				// advancing as jobs are written out again.
				xe_lrc_set_ring_tail(q->lrc[i],
						     xe_lrc_ring_head(q->lrc[i]));
			}
		}
		xe_sched_resubmit_jobs(sched);     // re-emit every pending job
	}

	xe_sched_submission_start(sched);
	xe_sched_submission_resume_tdr(sched);  // timers run again
}
```

This is why Chapter 10's LRC layout mattered. The ring is just memory with
a head and a tail; rewinding the tail and replaying is enough to make an
unfinished job run again on fresh hardware. Innocent apps **never notice a
GT reset happened**.

---

### Step 7 — Rung 4: wedge

Two ways to get here. One is policy:

```c
// xe_guc_submit.c:1352
static bool guc_submit_hint_wedged(struct xe_guc *guc)
{
	// Only in debug mode 2. Normally this returns false immediately.
	if (xe->wedged.mode != XE_WEDGED_MODE_UPON_ANY_HANG_NO_RESET)
		return false;
	if (xe_device_wedged(xe))
		return true;
	xe_device_declare_wedged(xe);
	return true;
}
```

```c
// xe_device_types.h:58
enum xe_wedged_mode {
	XE_WEDGED_MODE_NEVER = 0,                    // never wedge
	XE_WEDGED_MODE_UPON_CRITICAL_ERROR = 1,      // DEFAULT: only if recovery fails
	XE_WEDGED_MODE_UPON_ANY_HANG_NO_RESET = 2,   // debug: wedge on ANY hang,
};                                                   // resets disabled, state preserved
```

Mode 2 exists purely so a developer can keep the broken hardware state for
inspection instead of having it reset away.

The other way is the honest one: **recovery itself failed** — a GT reset
that didn't complete, or firmware that won't load.

```c
// xe_device.c:1453
void xe_device_declare_wedged(struct xe_device *xe)
{
	if (xe->wedged.mode == XE_WEDGED_MODE_NEVER)
		return;

	if (!atomic_xchg(&xe->wedged.flag, 1)) {   // once only
		xe->needs_flr_on_fini = true;      // force a function reset at unload
		drm_err(&xe->drm,
			"CRITICAL: Xe has declared device %s as wedged.\n"
			"IOCTLs and executions are blocked.\n" ...);
	}

	for_each_gt(gt, xe, id)
		xe_gt_declare_wedged(gt);

	if (xe_device_wedged(xe)) {
		if (xe->wedged.mode == XE_WEDGED_MODE_UPON_ANY_HANG_NO_RESET)
			xe_device_set_wedged_method(xe, DRM_WEDGE_RECOVERY_NONE);
		else if (!xe->wedged.method)
			xe_device_set_wedged_method(xe, DRM_WEDGE_RECOVERY_REBIND |
						    DRM_WEDGE_RECOVERY_BUS_RESET);

		// Tell userspace, through the standard DRM uevent, that this
		// device is gone and how a human could bring it back.
		drm_dev_wedged_event(&xe->drm, xe->wedged.method, NULL);
	}
}
```

From here every ioctl is rejected (`xe_device.c:227`, `:245`), and the only
exits are an unbind/rebind or a bus reset. Note that even this terminal
state is **announced**, not silent — Idea 3 holds all the way down.

---

## The other doors into this code

The TDR is not the only way a hang is discovered:

| Trigger | Entry point | What it does |
|---|---|---|
| job timeout (this chapter) | `guc_exec_queue_timedout_job()` `:1496` | the full ladder |
| **GuC reset an engine itself** | `xe_guc_exec_queue_reset_handler()` `:3205` | sets `RESET`, then kicks the TDR immediately |
| **memory CAT error** (bad GPU access) | `xe_guc_exec_queue_memory_cat_error_handler()` `:3266` | same — route into the TDR |
| userspace closed the queue | `guc_exec_queue_kill()` `:2087` | sets `KILLED`, signals the suspend fence, kicks cleanup |

They all funnel into the same place:

```c
// xe_guc_submit.c:3232 (reset handler) — don't duplicate the logic,
// just shorten the timer to zero and let the TDR do its job.
xe_guc_exec_queue_reset_trigger_cleanup(q);
    └─► xe_sched_tdr_queue_imm(&q->guc->sched);
```

**One recovery path, many doorbells.** The GuC telling you it reset an
engine and your own timer expiring both end in
`guc_exec_queue_timedout_job()`, which is why that function has to check so
many state bits up front — it genuinely does not know which door it came
through.

---

## What to carry forward

1. **A timer is evidence, not a verdict.** `check_timeout()` converting
   wall-clock into GPU-execution time (`ctx_timestamp - ctx_job_timestamp`,
   +5%) is the difference between a driver that kills innocent apps under
   load and one that doesn't.
2. **Escalate as little as possible.** Ban a queue before resetting a GT;
   reset a GT before wedging a device. `guc_exec_queue_stop()` shows the
   cost of skipping a rung: a GT reset bans every user queue with work in
   flight, guilty or not.
3. **Every fence gets answered, on every path.** `-ETIME` for the culprit,
   `-ECANCELED` for its neighbours, `suspend_fence_signal()` for a preempt
   fence caught mid-flight, and a `drm_dev_wedged_event()` even when the
   device is unrecoverable. Nothing is ever left waiting.

Next: Chapter 16 — the life of one frame, through every layer of this course.
