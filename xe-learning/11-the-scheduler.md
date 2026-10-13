# Chapter 11 — The Waiting Room

> Beat A: the story.

Chapter 10 called `drm_sched` *"a waiting room with a door."* This chapter is
the mechanism: how a job waits, what wakes it, and what happens the instant it
walks through.

The spine:

> *One `EXEC`. The batch buffer reads a texture another job is still writing.*
> *So this job must wait.*

Three ideas.

---

## Idea 1: A job carries a list of things it's waiting for

When userspace submits work, it doesn't say "run this now". It says "run this
**after** these other things finish".

So every job owns a **set of fences** — its **dependencies**. The job cannot run
until all of them have signalled.

Where do they come from? Two places:

**1. Explicit** — userspace listed them. `EXEC` takes an array of sync objects:
*"wait for these before you start, and signal these when you're done."*

**2. Implicit** — the driver worked them out. This is the clever part, and it's
Chapter 3's clipboard paying off:

> For every buffer the job touches, the driver reads that buffer's `dma_resv`
> and adds whatever fences are already on it.

So if another job is still writing your texture, its fence is on that texture's
clipboard, and your job automatically inherits it as a dependency. **Userspace
never had to know.** The buffer itself carried the information.

That's why Act I kept insisting the `dma_resv` was "the contract". This is the
other side of it: Act I's memory management *reads* the clipboard to know when
it's safe to move a buffer; Act II's scheduling *reads the same clipboard* to
know when it's safe to run.

---

## Idea 2: Every job has two fences, and they mean different things

This is the single most confusing thing about `drm_sched`, so let's nail it.

A job has **two** fences created the moment it is initialised, plus a **third**
that appears later:

| Fence | Signals when | Who creates it |
|-------|-------------|----------------|
| **scheduled** | the job has been handed to the hardware | `drm_sched` |
| **finished** | the work is actually complete | `drm_sched` |
| **parent** | the hardware finished | the *driver*, from `run_job()` |

And the relationship:

```
   job created
       |
       |  "scheduled" and "finished" exist already, both unsignalled
       |
   dependencies all signal
       |
       v
   run_job()  ---->  driver returns a fence  ("parent")
       |
       +--> "scheduled" signals NOW           ("it's on the hardware")
       |
   ... GPU does the work ...
       |
   "parent" signals
       |
       +--> "finished" signals                ("the work is done")
```

### Why two, and not one?

Because of **when they exist**.

`finished` exists *immediately*, before the job has any dependencies resolved
and before the driver has produced anything. So userspace can be handed
`finished` as its "tell me when this is done" fence right away, synchronously,
from the ioctl.

`parent` — the real hardware fence — doesn't exist until `run_job()` runs, which
could be much later. You cannot hand userspace something that doesn't exist yet.

**So `finished` is a promise that is later wired up to `parent`.** The kernel's
own comment says exactly this:

> *"When setting up an out fence for the job, you should use this, since it's
> available immediately upon `drm_sched_job_init()`, and the fence returned by
> the driver from `run_job()` won't be created until the dependencies have
> resolved."*

And `scheduled` exists for a narrower purpose: it lets one job know that the
*previous* job on the same hardware has at least been submitted, which matters
for pipelining.

> **The short version:** `finished` is what you give to everyone else. `parent`
> is what the hardware gives you. `scheduled` is the moment in between.

---

## Idea 3: Waiting is a callback, not a loop

Nothing spins. The whole thing is event-driven, and it's three small steps.

**Step 1 — try to pop a job.** The scheduler looks at the front of the queue and
asks the job: *"what are you still waiting for?"*

**Step 2 — if there's an unsignalled dependency, register a callback on it and
give up.** The scheduler does not wait. It returns, and the worker goes idle.

**Step 3 — when that fence signals, the callback re-queues the worker.** Which
goes back to step 1, and asks again. Maybe there's another dependency; maybe
there are none left.

```
   pop job -> any unsignalled dependency?
                 |                    |
                yes                   no
                 |                    |
      register callback,          run_job()
      return (go idle)
                 |
          [fence signals]
                 |
          re-queue the worker  --> back to the top
```

So a job with five dependencies wakes the worker five times, each time getting
one dependency closer. **No thread is ever blocked, and no CPU is ever spent
waiting.**

That is the entire "waiting room". The door opens when the last dependency
signals, and what's on the other side is `run_job()` — the driver callback that
writes commands into the ring (Chapter 10) and tells the GuC (Chapter 12).

---

## The picture

```
   EXEC ioctl
       |
   create job -> "scheduled" + "finished" fences exist, unsignalled
       |
   collect dependencies:
       explicit (userspace's sync objects)
       implicit (every touched buffer's dma_resv)     <-- ch 3's clipboard
       |
   hand "finished" back to userspace     <-- can be done NOW
       |
   push onto the entity's queue
       |
   ======================= THE WAITING ROOM =======================
       |
   worker: any unsignalled dep? -> register callback, sleep
                                -> fence signals, re-queue, ask again
       |
   all deps signalled
       |
   ======================= THE DOOR ===============================
       |
   run_job()  ->  write commands into the ring, tell the GuC
              ->  return the hardware fence ("parent")
       |
   "scheduled" signals
       |
   ... GPU runs ...
       |
   "parent" signals -> "finished" signals -> userspace's wait returns
```

## Remember these three

1. **A job carries a set of fence dependencies** — some listed by userspace, the
   rest read automatically off every touched buffer's `dma_resv`.
2. **`finished` is the promise you hand out immediately; `parent` is the real
   hardware fence that arrives later and gets wired to it.**
3. **Waiting is a callback chain, not a loop.** Each signalling dependency
   re-queues the worker, which asks again.

---
---

# Chapter 11 — Beat B: The Code

Five functions, in call order:

| Function | File | Job |
|----------|------|-----|
| `drm_sched_job_init()` | `sched_main.c:595` | create the two fences |
| `xe_sched_job_add_deps()` | `xe_sched_job.c:356` | harvest a buffer's fences |
| `drm_sched_job_arm()` | `sched_main.c:644` | make `finished` real and orderable |
| `drm_sched_entity_pop_job()` | `sched_entity.c:533` | **the dependency loop** |
| `drm_sched_run_job_work()` | `sched_main.c:1015` | **the door** |

> Code trimmed. `//` comments are mine.

---

## Step 1 — Create the job and its fences

**File: `sched_main.c:595`**

```c
int drm_sched_job_init(struct drm_sched_job *job,
		       struct drm_sched_entity *entity,
		       u32 credits, void *owner, uint64_t drm_client_id)
{
	memset(job, 0, sizeof(*job));

	job->entity = entity;        // which in-order queue (= which exec queue)
	job->credits = credits;      // ch 10: ring capacity accounting

	job->s_fence = drm_sched_fence_alloc(entity, owner, drm_client_id);
	//             ^^^^^^^^^^^^^^^^^^^^ IDEA 2: this allocates BOTH the
	//             "scheduled" and "finished" fences, right now, before any
	//             dependency is known and before the driver has done anything.

	xa_init_flags(&job->dependencies, XA_FLAGS_ALLOC);
	//            ^^^^^^^^^^^^^^^^^^ IDEA 1: an empty set, ready to fill
	return 0;
}
```

Two lines, two ideas. The fences exist before the dependencies do — that's what
makes Idea 2 necessary.

And what `s_fence` holds (`include/drm/gpu_scheduler.h`):

```c
struct drm_sched_fence {
	struct dma_fence scheduled;    // signalled when handed to hardware
	struct dma_fence finished;     // signalled when the work completes
	struct dma_fence *parent;      // <<< the fence run_job() returned.
				       //     "We signal finished once parent
				       //      is signalled."
};
```

**Three fences, one struct.** The header's own comments are the clearest
statement of Idea 2 in the tree — worth reading in full once.

### Xe's wrapper — `xe_sched_job.c:96`

```c
struct xe_sched_job *xe_sched_job_create(struct xe_exec_queue *q, u64 *batch_addr)
{
	job = job_alloc(...);
	job->q = q;
	xe_exec_queue_get(job->q);        // a job holds the queue alive

	err = drm_sched_job_init(&job->drm, q->entity, 1, NULL, ...);
	//                                  ^^^^^^^^^ ch 10's 1:1 entity

	for (i = 0; i < q->width; ++i) {
		struct dma_fence *fence = xe_lrc_alloc_seqno_fence();
		job->ptrs[i].lrc_fence = fence;
		//   ^^^ Xe pre-allocates the HARDWARE fence here, before
		//       run_job() needs it. It signals when the LRC's seqno
		//       counter reaches this job's number (ch 10's ring sandwich).
	}

	for (i = 0; i < width; ++i)
		job->ptrs[i].batch_addr = batch_addr[i];
	//   ^^^ the only thing userspace actually gave us: an address
}
```

Note `xe_lrc_alloc_seqno_fence()`. Xe pre-allocates the hardware fence at job
creation so that `run_job()` — which runs in a context where allocation failure
would be awkward — never has to allocate. The fence is armed later and becomes
`parent`.

---

## Step 2 — Harvest the dependencies

**File: `xe_sched_job.c:356`** — the whole function:

```c
int xe_sched_job_add_deps(struct xe_sched_job *job, struct dma_resv *resv,
			  enum dma_resv_usage usage)
{
	return drm_sched_job_add_resv_dependencies(&job->drm, resv, usage);
	//     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
	//  "Walk this buffer's clipboard and add every fence matching `usage`
	//   to my dependency set."
}
```

**That one line is Idea 1's implicit dependencies.** Pass a buffer's `dma_resv`
and a usage class (Chapter 3's READ / WRITE / KERNEL / BOOKKEEP) and the fences
come across automatically.

Our example — the job reads a texture another job is writing:

```
   texture's dma_resv:
       [ WRITE  : fence of the job still writing it ]      <- picked up
       [ KERNEL : fence of the clear done at creation ]    <- picked up

   our job's dependencies after xe_sched_job_add_deps(resv, USAGE_READ):
       { writer's fence, clear fence }
```

The usage class is what makes this precise: we're *reading*, so we must wait for
**writers** (and for kernel operations), but not for other **readers**.
Chapter 3's four tags, finally doing their job.

Explicit dependencies arrive through the same door, one fence at a time, via
`drm_sched_job_add_dependency()` at `sched_main.c:671`.

---

## Step 3 — Arm it

**File: `sched_main.c:644`**

```c
void drm_sched_job_arm(struct drm_sched_job *job)
{
	drm_sched_entity_select_rq(entity);
	sched = entity->rq->sched;

	job->sched = sched;
	job->s_priority = entity->priority;

	drm_sched_fence_init(job->s_fence, job->entity);
	//                   ^^^^^^^^^^^^ gives both fences their context and
	//                   sequence number - making them properly ordered
	//                   fences that others can wait on.
}
```

Why is this separate from `init()`? Because between the two you are allowed to
**fail and bail out**. Up to `arm()`, the job's fences aren't published; nothing
outside can be waiting on them, so you can free everything. After `arm()` the
fences have identities and may already be visible to other code.

**`init` then `arm` is the same "prepare, then commit" shape as Chapters 6, 7
and 8.** Third time in this course.

---

## Step 4 — The dependency loop

**File: `sched_entity.c:533`.** This is the heart of the chapter.

```c
struct drm_sched_job *drm_sched_entity_pop_job(struct drm_sched_entity *entity)
{
	sched_job = drm_sched_entity_queue_peek(entity);   // front of the queue
	if (!sched_job)
		return NULL;                              // nothing to do

	// ===================== IDEA 3 =====================
	while ((entity->dependency =
			drm_sched_job_dependency(sched_job, entity))) {
		//      ^^^ "give me the next dependency that hasn't signalled"
		//          (returns NULL when there are none left)

		if (drm_sched_entity_add_dependency_cb(entity, sched_job))
			return NULL;
		//  ^^^ Register a callback on that fence and RETURN NULL.
		//      "Not ready. Wake me when this one signals."
		//
		//      If the fence turns out to be already signalled, this
		//      returns false and the loop just asks for the next one.
	}
	// ==================================================
	// Falling out of the loop means: ZERO unsignalled dependencies left.

	if (entity->guilty && atomic_read(entity->guilty))
		dma_fence_set_error(&sched_job->s_fence->finished, -ECANCELED);
	//  ^^^ this queue was blamed for a hang (ch 15) - fail the job instead
	//      of running it

	rcu_assign_pointer(entity->last_scheduled,
			   dma_fence_get(&sched_job->s_fence->finished));
	//                                      ^^^^^^^^ remember this job's
	//   finished fence, so the NEXT job can be ordered behind it

	spsc_queue_pop(&entity->job_queue);
	return sched_job;                                 // ready to run
}
```

Read the `while` loop twice. It has **two exits**:

* `return NULL` — a dependency isn't ready. A callback is now armed, and the
  worker will be re-queued when it fires.
* fall through — nothing left to wait for. Hand the job back.

**There is no wait, no sleep, no poll.** The loop either finds something to wait
on (and leaves) or finds nothing (and proceeds). That's Idea 3 in nine lines.

---

## Step 5 — The door

**File: `sched_main.c:1015`.** The worker that runs when something might be
ready:

```c
static void drm_sched_run_job_work(struct work_struct *w)
{
	entity = drm_sched_select_entity(sched);
	if (!entity)
		return;        // nothing ready, or not enough ring credits (ch 10)

	sched_job = drm_sched_entity_pop_job(entity);      // <<< step 4
	if (!sched_job) {
		complete_all(&entity->entity_idle);
		drm_sched_run_job_queue(sched);
		return;        // still waiting on a dependency - a callback is armed
	}

	s_fence = sched_job->s_fence;

	atomic_add(sched_job->credits, &sched->credit_count);   // ch 10's
							        // ring accounting
	drm_sched_job_begin(sched_job);     // put it on the pending list, so the
					    // timeout detector can see it (ch 15)

	// ===================== THE DOOR =====================
	fence = sched->ops->run_job(sched_job);
	//      ^^^^^^^^^^^^^^^^^^^ THE DRIVER CALLBACK.
	//      For Xe this is guc_exec_queue_run_job(): write commands into the
	//      ring, bump the tail, tell the GuC. (Chapter 12.)
	//      It returns the HARDWARE fence - Idea 2's "parent".
	// ====================================================

	drm_sched_fence_scheduled(s_fence, fence);
	//                        ^^^^^^^^^^^^^^ IDEA 2, THE WIRE-UP:
	//   signals "scheduled" right now, AND records `fence` as the parent.

	if (!IS_ERR_OR_NULL(fence)) {
		r = dma_fence_add_callback(fence, &sched_job->cb,
					   drm_sched_job_done_cb);
		//  ^^^ when the HARDWARE fence signals, drm_sched_job_done()
		//      runs - and that is what signals "finished".
		dma_fence_put(fence);
	} else {
		drm_sched_job_done(sched_job, ...);   // run_job failed: finish
						     // the job with an error
	}

	drm_sched_run_job_queue(sched);     // try again - maybe the next job is
					    // ready too
}
```

Every part of Idea 2 is in this one function:

| Line | Does |
|------|------|
| `run_job(sched_job)` | the driver produces `parent` |
| `drm_sched_fence_scheduled(s_fence, fence)` | signals **scheduled**, stores **parent** |
| `dma_fence_add_callback(fence, ...)` | arranges for **finished** to signal when **parent** does |

And `drm_sched_run_job_queue(sched)` at the end is the loop: having run one job,
immediately ask whether another is ready.

### Where Xe pushes the job in — `xe_sched_job.c:293`

```c
void xe_sched_job_push(struct xe_sched_job *job)
{
	xe_sched_job_get(job);                   // hold a reference across the push
	trace_xe_sched_job_exec(job);
	drm_sched_entity_push_job(&job->drm);    // <<< into the waiting room
	xe_sched_job_put(job);
}
```

Four lines. `drm_sched_entity_push_job()` adds the job to the entity's queue and
kicks the worker — which lands us in step 5.

---

## Our example, end to end

```
  EXEC: run batch at 0x200000000; it READS a texture someone is writing

  xe_sched_job_create()               xe_sched_job.c:96
    -> drm_sched_job_init()           sched_main.c:595
         "scheduled" + "finished" fences now exist, unsignalled
    -> xe_lrc_alloc_seqno_fence()     the future "parent", pre-allocated

  xe_sched_job_add_deps(texture->resv, USAGE_READ)    xe_sched_job.c:356
         deps = { writer's fence, creation-clear fence }

  drm_sched_job_arm()                 sched_main.c:644
         fences get identities; "finished" handed to userspace

  xe_sched_job_push()                 xe_sched_job.c:293
    -> drm_sched_entity_push_job()    into the queue, kick the worker

  --- worker wakes ---
  drm_sched_entity_pop_job()          sched_entity.c:533
         writer's fence unsignalled -> register callback, return NULL
  --- worker idles ---

  --- writer's fence signals -> callback re-queues the worker ---
  drm_sched_entity_pop_job()
         clear fence already signalled -> loop continues -> no deps left
         -> return the job

  drm_sched_run_job_work()            sched_main.c:1015
    -> run_job()  = guc_exec_queue_run_job()      <<< CHAPTER 12
         writes the ring, bumps the tail, tells the GuC
         returns the hardware fence
    -> "scheduled" signals
    -> callback armed on the hardware fence

  --- GPU executes; LRC seqno reaches this job's number ---
    -> hardware fence signals
    -> drm_sched_job_done() -> "finished" signals
    -> userspace's wait returns
```

## Try it on your machine

```bash
# the scheduler's own tracepoints: queued, run, done
echo 1 > /sys/kernel/debug/tracing/events/gpu_scheduler/enable
# plus Xe's
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_create/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_exec/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_run/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

The gap between `xe_sched_job_exec` and `xe_sched_job_run` for a given job *is*
the time it spent in the waiting room.

## Checkpoint

1. A job's `finished` fence exists before the job has any dependencies resolved
   and before the driver has produced anything. Why is that necessary?
2. Userspace never told the kernel "wait for the job writing my texture". How
   does that dependency get into the job's dependency set?
3. `drm_sched_entity_pop_job()` has a `while` loop over dependencies, but never
   waits. What are its two exits?
4. What three things does `drm_sched_run_job_work()` do with the fence that
   `run_job()` returns?
5. Why are `drm_sched_job_init()` and `drm_sched_job_arm()` separate functions?

<details>
<summary>answers</summary>

1. Because userspace needs an out-fence *synchronously*, from the ioctl. The real
   hardware fence doesn't exist until `run_job()` runs, which may be much later —
   so `finished` is created immediately as a promise, and later wired to the
   hardware fence (`parent`). The header comment on `drm_sched_fence.finished`
   says precisely this.
2. Implicitly, from the buffer. `xe_sched_job_add_deps()` walks the texture's
   `dma_resv` and copies every fence matching the requested usage class into the
   job's dependency set. Because the job is *reading*, it picks up WRITE and
   KERNEL fences but not other readers' READ fences.
3. `return NULL` when a dependency hasn't signalled — having first registered a
   callback that will re-queue the worker — and falling out of the loop when
   `drm_sched_job_dependency()` returns NULL, meaning nothing is left to wait for.
4. Passes it to `drm_sched_fence_scheduled()`, which signals **scheduled** and
   stores the fence as **parent**; adds a callback so that when it signals,
   `drm_sched_job_done()` signals **finished**; and drops its own reference with
   `dma_fence_put()`.
5. So that failure between them is cheap. Before `arm()` the job's fences have no
   identity and nothing outside can be waiting on them, so the job can simply be
   freed. After `arm()` they are real, ordered fences that may already be visible
   to other code. It's the same prepare-then-commit discipline as Chapters 6–8.

</details>
