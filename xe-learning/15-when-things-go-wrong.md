# Chapter 15 — When the GPU Stops Answering

> Act III, part 2. What happens when a job hangs.

---

## Beat A — The story

### Idea 1 — The driver never *knows* a job is hung. It guesses.

No signal comes from the hardware saying *"I am stuck."* All the driver gets
is a timer that ran out — 5 seconds, by default.

But a timer running out proves nothing. The job might have spent 4.9 of
those 5 seconds **waiting its turn**, descheduled by the GuC while other
apps ran, and only 100ms actually executing.

So the driver's first move is to **distrust its own timer** and go find out
how much time the job really spent on the hardware.

### Idea 2 — Four rungs. Climb as few as possible.

```
  rung 1   rearm the timer     -> nothing happened, false alarm
  rung 2   ban the queue       -> this one app's queue dies
  rung 3   reset the GT        -> every queue on that GT is disturbed
  rung 4   wedge the device    -> device is dead until someone rebinds it
```

A shader loop in one app must only reach **rung 2**. Resetting the GT for it
would disturb every other process on the machine. Each rung up costs more
innocent bystanders, so you only climb when the rung below cannot fix it.

### Idea 3 — Whatever happens, every fence must be answered.

A `dma_fence` that never signals hangs whoever is waiting on it — forever,
possibly in another process or another driver.

So every single path out of this code ends the same way: **signal
everything, with an error.** Even the catastrophic one. Death gets
announced; it is never silent.

---

## Beat B — The code

> Code below is trimmed hard: multi-queue, SR-IOV and crash-dump paths
> removed so the shape is visible. Line numbers point at the real thing.

### The worked example

An app submits a 3D job with an infinite shader loop. Its 5-second timer
expires. We follow that one job to the end.

### Named functions

| Function | Where | Job |
|---|---|---|
| `guc_exec_queue_timedout_job()` | `xe_guc_submit.c:1496` | the whole decision |
| `check_timeout()` | `xe_guc_submit.c:1369` | is it *really* hung? |
| `gt_reset_worker()` | `xe_gt.c:912` | rung 3 |
| `xe_device_declare_wedged()` | `xe_device.c:1453` | rung 4 |

---

### Step 1 — Is it really hung?

```c
// xe_guc_submit.c:1524
// Freeze submission first. Nothing may be inspected while new jobs
// could still start underneath us.
xe_sched_submission_stop(sched);

// xe_guc_submit.c:1559
// Now ask the real question. If the answer is no -> rung 1, just
// give the job more time.
if (!check_timeout(q, job))
	goto rearm;
```

And the function that answers it:

```c
// xe_guc_submit.c:1369
static bool check_timeout(struct xe_exec_queue *q, struct xe_sched_job *job)
{
	// The LRC keeps two hardware counters (Chapter 10):
	//   ctx_timestamp     = total time this context has ever run
	//   ctx_job_timestamp = that same counter when THIS job started
	ctx_timestamp     = xe_lrc_timestamp(q->lrc[0]);
	ctx_job_timestamp = xe_lrc_ctx_job_timestamp(q->lrc[0]);

	// THE WHOLE POINT: subtracting them gives time spent RUNNING,
	// not time since submission. A job starved by other apps has a
	// tiny difference and survives.
	diff = ctx_timestamp - ctx_job_timestamp;

	// +5% slack, because GuC scheduling delay is not the app's fault.
	running_time_ms = ADJUST_FIVE_PERCENT(clock_to_ms(gt, diff));

	return running_time_ms >= q->sched_props.job_timeout_ms;
}
```

Our shader loop really did burn 5 seconds of GPU. Guilty.

### Step 2 — Ban the queue and take it off the hardware

```c
// xe_guc_submit.c:1569
// Rung 2. Note what is NOT reset here: not the GT, not the device.
// Just this one queue.
set_exec_queue_banned(q);

// xe_guc_submit.c:1596
// Tell the GuC to stop running this context.
disable_scheduling(q);

// xe_guc_submit.c:1608
// Then WAIT for the GuC to confirm it stopped. This wait is not
// optional: signalling a fence means "GPU is done with your buffer",
// and the app may free that buffer. Saying it early is a
// use-after-free.
ret = wait_event_timeout(guc->ct.wq,
			 !exec_queue_pending_disable(q), HZ * 5);

// xe_guc_submit.c:1615
// GuC never answered? Then the GuC itself is broken, and banning a
// queue cannot fix that. Climb to rung 3.
if (!ret) {
	xe_gt_reset_async(q->gt);
	goto rearm;
}
```

### Step 3 — Answer every fence (Idea 3, in two lines)

```c
// xe_guc_submit.c:1666
xe_sched_job_set_error(job, -ETIME);        // the job that hung

drm_sched_for_each_pending_job(tmp_job, &sched->base, NULL)
	xe_sched_job_set_error(tmp_job, -ECANCELED);   // the ones queued behind it
```

Two different errors on purpose. `-ETIME` says *"you broke it."*
`-ECANCELED` says *"you were standing next to the thing that broke."*
Userspace can tell the difference.

Then wake anyone sleeping on a user fence, so they see the error instead of
waiting forever:

```c
// xe_guc_submit.c:550
static void xe_guc_exec_queue_trigger_cleanup(struct xe_exec_queue *q)
{
	wake_up_all(&xe->ufence_wq);          // unblocks xe_wait_user_fence()
	xe_sched_tdr_queue_imm(&q->guc->sched);
}
```

**For our example, this is the end.** One queue banned, its app gets
`-ETIME`, every other process on the GPU kept running and noticed nothing.
That is rung 2 doing its job.

The remaining steps only happen when a ban is not enough.

---

### Step 4 — Rung 3: reset the GT

Two things escalate here:

```c
// xe_guc_submit.c:1486
if (q->flags & EXEC_QUEUE_FLAG_KERNEL)
	return true;              // the DRIVER's own queue hung -> we are broken

return !xe_sched_job_started(job);  // job never even started -> GuC's fault,
                                    // not the app's. Don't blame the app.
```

The reset runs in a worker, never inline:

```c
// xe_gt.c:912 — stripped to the shape
static void gt_reset_worker(struct work_struct *w)
{
	xe_gt_sanitize(gt);            // forget cached hardware state
	xe_uc_stop(&gt->uc);           // every queue on this GT: see below
	xe_tlb_inval_reset(&gt->tlb_inval);

	err = do_gt_reset(gt);         // the actual hardware reset
	if (!err)
		err = do_gt_restart(gt);   // reload GuC, re-register queues
	if (err)
		xe_device_declare_wedged(gt_to_xe(gt));   // rung 4
}
```

**What this costs the innocent.** The GuC is about to be reloaded and will
come back knowing nothing — no contexts, no scheduling state. So every
queue on the GT is stopped, and:

```c
// xe_guc_submit.c:2590 — the collateral damage, stated plainly
// Any USER queue with a started-but-unfinished job gets banned, whether
// or not it caused the hang. We cannot know, and we cannot safely re-run
// a half-executed job. (Kernel/VM queues are exempt — they are ours.)
if (!(q->flags & (EXEC_QUEUE_FLAG_KERNEL | EXEC_QUEUE_FLAG_VM))) {
	job = xe_sched_first_pending_job(sched);
	if (job && xe_sched_job_started(job) && !xe_sched_job_completed(job))
		set_exec_queue_banned(q);
}
```

That paragraph is why Idea 2 exists. A GT reset is never free.

**And the queues that survive simply re-run their work:**

```c
// xe_guc_submit.c:2825
if (!exec_queue_killed_or_banned_or_wedged(q)) {       // survivors only
	job = xe_sched_first_pending_job(sched);

	// Rewind the ring tail to where this job began, so resubmission
	// rewrites the same bytes in the same place.
	q->lrc[0]->ring.tail = job->ptrs[0].head;
	xe_lrc_set_ring_tail(q->lrc[0], xe_lrc_ring_head(q->lrc[0]));

	xe_sched_resubmit_jobs(sched);   // emit them all again
}
```

This is Chapter 10 paying off. The ring is just memory with a head and a
tail — rewind the tail, replay. **An innocent app never learns that a GT
reset happened.**

### Step 5 — Rung 4: wedge

Only reached when recovery itself failed: the reset didn't complete, or the
firmware won't load.

```c
// xe_device.c:1453
void xe_device_declare_wedged(struct xe_device *xe)
{
	if (!atomic_xchg(&xe->wedged.flag, 1)) {       // once only
		xe->needs_flr_on_fini = true;
		drm_err(&xe->drm, "CRITICAL: ... declared device as wedged.\n"
				  "IOCTLs and executions are blocked.\n" ...);
	}

	// Tell userspace, via the standard DRM uevent, that this device is
	// gone — and how a human could bring it back.
	xe_device_set_wedged_method(xe, DRM_WEDGE_RECOVERY_REBIND |
					DRM_WEDGE_RECOVERY_BUS_RESET);
	drm_dev_wedged_event(&xe->drm, xe->wedged.method, NULL);
}
```

From here every ioctl is rejected (`xe_device.c:227`). The only exits are an
unbind/rebind or a bus reset.

Even this — the terminal state — is **announced**. Idea 3 holds all the way
to the bottom.

---

## One more thing worth knowing

The timer is not the only way a hang is found. The GuC can reset an engine
by itself and tell us, or report a bad memory access (a "CAT error"). Both
land in the same place:

```c
// xe_guc_submit.c:3232
// Don't duplicate the logic — just set the timer to zero and let the
// handler above run.
xe_sched_tdr_queue_imm(&q->guc->sched);
```

**One recovery path, several doorbells.** That is also why
`guc_exec_queue_timedout_job()` checks so much state at its top: it
genuinely doesn't know which door it came in through.

---

## What to carry forward

1. **A timer is evidence, not a verdict.** `ctx_timestamp -
   ctx_job_timestamp` turns wall-clock into GPU-run-time, and that one
   subtraction is the difference between a driver that kills innocent apps
   under load and one that doesn't.
2. **Climb as few rungs as possible.** Rung 2 costs one app. Rung 3 bans
   every user queue with work in flight, guilty or not. Rung 4 costs the
   machine.
3. **Nothing is ever left waiting.** `-ETIME` for the culprit, `-ECANCELED`
   for its neighbours, a wakeup for user-fence sleepers, and a uevent even
   when the device is beyond saving.

Next: Chapter 16 — the life of one frame, through every layer of this course.
