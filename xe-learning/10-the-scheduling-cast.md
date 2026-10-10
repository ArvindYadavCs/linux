# Chapter 10 — The Scheduling Cast

> **Act II begins.** Beat A: the story.

Act I answered: *where is the memory, and what address does the GPU use?*

Act II answers a different question: **how does work actually get onto an
engine, and who decides when?**

The spine for this chapter and the next few:

> *Userspace has a batch buffer at GPU address `0x200000000`. Run it.*

Three ideas. This chapter is deliberately all cast and no plot — we meet the
characters now and watch them work in Chapters 11–13.

---

## Idea 1: An exec queue is a client's conveyor belt

Userspace creates exactly one kind of thing to run work: an **exec queue**.

```
   GEM_CREATE          -> make a buffer          (ch 3)
   VM_BIND             -> give it an address     (ch 8)
   EXEC_QUEUE_CREATE   -> make a conveyor belt   <-- this chapter
   EXEC                -> put work on the belt   (ch 13)
```

An exec queue is:

* **bound to one engine class** — render, or copy, or video decode. You cannot
  put a copy job on a render queue.
* **bound to one VM** — all work on this queue uses that address space.
* **in-order** — jobs on one queue execute in the order submitted. Always.
* **the unit of scheduling** — the GuC schedules *queues*, not jobs.

A client typically has several: one for render, one for copies, maybe one for
video. They run concurrently and in any order relative to each other; ordering
*between* queues is userspace's problem, expressed with fences.

> **An exec queue is what the hardware calls a *context*.** Same thing, two
> names. Xe says "exec queue" on the userspace side and "context" when talking
> to hardware or the GuC.

---

## Idea 2: A context is two things in memory — state, and a ring

Here is the hardware reality, and it is simpler than people expect.

An engine does not take "jobs". An engine is a **command streamer**: it reads a
stream of commands from memory and executes them. To run a client's work the
hardware needs two things, and both are just memory:

### The ring buffer — a circular queue of commands

A fixed-size buffer with a **head** and a **tail**:

* the **driver** writes commands at the tail and advances it
* the **hardware** reads from the head and advances it
* when they meet, the ring is empty — the engine has caught up

```
      ring buffer (circular)
    +---+---+---+---+---+---+---+---+
    |   |   | J | J | J |   |   |   |
    +---+---+---+---+---+---+---+---+
              ^           ^
             head        tail
          (hardware    (driver
           reads here)  writes here)
```

**Submitting work is literally: write commands at the tail, then tell the
hardware the tail moved.** That's it. There is no "submit" instruction.

### The LRC — the state the engine loads

A context has state: register values, the page table root, the ring's own head
and tail pointers. All of that lives in a block of memory called the **LRC**
(Logical Ring Context).

"Logical ring context" means: *the context, as a thing in memory, rather than as
live register values*. When the GuC tells an engine "run this context", it hands
over the address of an LRC. The engine **loads** that state into its registers,
starts reading the ring, and when it's time to switch away it **saves** its
registers back into the LRC.

That save/restore is the whole reason GPUs can multitask.

### They're one allocation

In Xe, the LRC and the ring live in **one buffer object**:

```
   xe_lrc->bo   (one xe_bo, in VRAM, with a GGTT address)
   +-----------------------------------------------------+
   |  per-process hardware status page (4 KB)            |
   |  the context state image (registers, PT root, ...)  |
   |  the ring buffer                                    |
   +-----------------------------------------------------+
```

And remember Chapter 5: **this buffer needs a GGTT address**, because the GuC
and the engine reach it through GGTT, not through the client's PPGTT. That's why
`xe_lrc.c` was in the list of GGTT users.

**One exec queue owns one LRC per hardware engine it submits to.**

---

## Idea 3: Two schedulers, with different jobs

This is the part that confuses people coming from other drivers, so let's be
precise. There are **two** schedulers in the path, and they are not competitors.

### `drm_sched` — the dependency gatekeeper (in the kernel)

DRM's shared scheduler front-end. Its vocabulary:

* a **job** is one unit of work
* an **entity** is an in-order queue of jobs — **one entity per exec queue** in Xe
* a job carries a set of **fence dependencies**
* when all dependencies have signalled, `drm_sched` calls the driver's
  `run_job()`

So `drm_sched`'s entire job is: **hold a job until its dependencies are met,
then hand it to the driver.** It does *not* decide which engine runs what, or
for how long. It's a waiting room with a door.

### The GuC — the actual scheduler (on the GPU)

Once `run_job()` writes commands into the ring, the **GuC** decides:

* which contexts are resident on which engines
* when to time-slice between them
* when to preempt one for a higher-priority one
* when a context has hung and must be reset

**The kernel driver does not make these decisions.** It registers contexts with
the GuC and says "this one has work". The GuC does the rest.

### The handoff, in one line

```
   userspace  ->  drm_sched  ->  run_job()  ->  ring + GuC  ->  engine
                  "deps met?"    "write cmds"   "when & where"
```

Chapter 1 called the GuC the foreman and said *"the kernel driver is the
supplier."* This is that sentence made structural: `drm_sched` is the supplier's
dispatch desk; the GuC is the foreman on the floor.

### Xe's twist: one entity, one scheduler

In most drivers many entities feed one scheduler, and the scheduler picks
between them. Xe does something unusual:

> **Each exec queue gets its own `drm_gpu_scheduler`, with exactly one entity.**

A 1:1:1 relationship — queue, scheduler, entity. There is nothing for
`drm_sched` to choose *between*, which is exactly the point: **choosing is the
GuC's job, so Xe deliberately removes that responsibility from `drm_sched` and
uses it purely as a dependency tracker.**

---

## The full cast

```
  userspace
      |
      | EXEC_QUEUE_CREATE
      v
  xe_exec_queue  ------------------- "a client's conveyor belt"
      |    |                          one engine class, one VM, in-order
      |    |
      |    +-- xe_lrc[]  ----------- "state + ring", one per engine,
      |    |                          one xe_bo, needs a GGTT address
      |    |
      |    +-- drm_sched_entity ----- "the waiting room queue"
      |    |
      |    +-- xe_gpu_scheduler ----- "the door: deps met? -> run_job()"
      |    |
      |    +-- xe_guc_exec_queue ---- "the GuC-side identity (a context ID)"
      v
  xe_sched_job  --------------------- one EXEC = one job
      |
      v
  the ring  ->  the GuC  ->  xe_hw_engine  ->  silicon
```

## Remember these three

1. **An exec queue is a client's in-order conveyor belt, bound to one engine
   class and one VM.** The hardware calls it a context.
2. **A context is state (LRC) plus a ring buffer, both just memory** — and
   submitting work means writing commands at the tail and bumping the tail
   pointer.
3. **`drm_sched` waits for dependencies; the GuC decides what runs.** Xe gives
   every queue its own scheduler precisely so `drm_sched` never has to choose.

---
---

# Chapter 10 — Beat B: The Code

No single journey this time — we're meeting structures. Five of them, in the
order they nest:

| Structure | File | Is |
|-----------|------|-----|
| `xe_exec_queue` | `xe_exec_queue_types.h:92` | the conveyor belt |
| `xe_lrc` | `xe_lrc_types.h:17` | state + ring |
| `xe_sched_job` | `xe_sched_job_types.h` | one unit of work |
| `xe_gpu_scheduler` | `xe_gpu_scheduler_types.h` | the door |
| `xe_exec_queue_ops` | `xe_exec_queue_types.h` | the backend interface |

> Code trimmed to the fields that matter. `//` comments are mine.

---

## 1. The conveyor belt — `xe_exec_queue_types.h`

```c
struct xe_exec_queue {
	struct xe_file *xef;         // which client owns it (NULL = kernel's own)
	struct xe_gt *gt;            // which GT it submits to  (ch 1: execution
				     // is per-GT)
	struct xe_vm *vm;            // <<< IDEA 1: ONE address space
	enum xe_engine_class class;  // <<< IDEA 1: ONE engine class
	u32 logical_mask;            // which engine instances it may run on

	u16 width;                   // how many batches per EXEC (parallel
				     // submission - usually 1)

	struct {
		u32 timeslice_us;        // how long the GuC lets it run
		u32 preempt_timeout_us;  // how long to wait when preempting
		u32 job_timeout_ms;      // when to declare it hung
		enum xe_exec_queue_priority priority;
	} sched_props;               // <<< all four are GuC POLICY, not
				     //     kernel policy (Idea 3)

	union {
		struct xe_execlist_exec_queue *execlist;  // bring-up only
		struct xe_guc_exec_queue *guc;            // <<< the normal path
	};

	const struct xe_exec_queue_ops *ops;        // the backend vtable
	const struct xe_ring_ops *ring_ops;         // how to write a job
	struct drm_sched_entity *entity;            // <<< IDEA 3: 1:1

	struct xe_lrc *lrc[] __counted_by(width);   // <<< IDEA 2: state + ring
};
```

Read `sched_props` carefully. **Timeslice, preempt timeout, priority — the
driver stores them but does not act on them.** They are sent to the GuC, which
enforces them. That's Idea 3 in struct form: the kernel holds the knobs, the GuC
turns them.

And the `union`: `guc` or `execlist`. Xe keeps a non-GuC path for early silicon,
but it's vestigial. We follow `guc`.

### The uAPI

```c
struct drm_xe_exec_queue_create {
	__u64 extensions;      // priority, timeslice, ... set via extensions
	__u16 width;           // batches per EXEC
	__u16 num_placements;  // how many engine instances to choose from
	__u32 vm_id;           // <<< the VM this queue is bound to
	__u32 flags;
	__u32 exec_queue_id;   // OUT - the handle
	__u64 instances;       // which engines, from DEVICE_QUERY
};
```

`vm_id` being mandatory is Idea 1: **a queue without an address space is
meaningless.**

---

## 2. State + ring — `xe_lrc_types.h:17`

```c
struct xe_lrc {
	struct xe_bo *bo;        // <<< ONE buffer object holding:
				 //      - the per-process HW status page
				 //      - the context state image
				 //      - the ring buffer
				 //    In VRAM, with a GGTT address (ch 5).

	struct xe_bo *seqno_bo;  // separate, ALWAYS SYSTEM MEMORY -
				 // "GPU write, CPU read" (see below)

	struct xe_gt *gt;        // execution is per-GT

	struct {
		u32 size;        // ring size in bytes
		u32 tail;        // <<< IDEA 2: where the driver writes next
		u32 old_tail;    // shadow, so we can tell if it moved
	} ring;

	u64 desc;                // the LRC descriptor the GuC is given
	struct xe_hw_fence_ctx fence_ctx;   // where completion fences come from
};
```

Two details worth stopping on.

**`seqno_bo` is always system memory.** Its comment says why: *"Always in system
memory as this a CPU read, GPU write path object."* The GPU writes completion
sequence numbers; the CPU polls them. Chapter 3's caching rule inverted — when
the **CPU reads**, you want cached system memory, not VRAM behind a BAR window.

**There is no `head` field.** The driver doesn't track it — the *hardware* owns
head, and writes it into the status page. The driver only ever writes `tail`.
Idea 2's asymmetry, visible as a missing struct member.

### What the LRC buffer contains — `xe_lrc.c:47`

```c
#define LRC_PPHWSP_SIZE               SZ_4K   // per-process HW status page
#define LRC_INDIRECT_CTX_BO_SIZE      SZ_4K
#define LRC_INDIRECT_RING_STATE_SIZE  SZ_4K
```

Status page first, then the register image, then the ring. One allocation, three
regions — which is why `xe_lrc` has a `size` separate from `ring.size`.

### Writing into the ring — `xe_lrc.c:1826`

The function that *is* Idea 2:

```c
void xe_lrc_write_ring(struct xe_lrc *lrc, const void *data, size_t size)
{
	ring = __xe_lrc_ring_map(lrc);          // CPU mapping of the ring region

	rhs = lrc->ring.size - lrc->ring.tail;  // bytes left before we wrap
	if (size > rhs) {
		// The write straddles the end of the buffer: split it in two.
		__xe_lrc_write_ring(lrc, ring, data, rhs);          // to the end
		__xe_lrc_write_ring(lrc, ring, data + rhs, size - rhs); // from 0
	} else {
		__xe_lrc_write_ring(lrc, ring, data, size);
	}

	/*
	 * The ring and the LRC context image are both WC, so the ring tail
	 * update which publishes these writes can become visible to the device
	 * first. Ensure the ring contents are visible before returning.
	 */
	xe_device_wmb(xe);
	// ^^^ THE critical barrier. The tail update is what tells hardware
	//     "new commands". If the tail becomes visible before the commands
	//     do, the engine executes garbage.
}
```

The split write is the circular buffer, in four lines. And that comment is the
single most important barrier in the submission path — **Chapter 3's
write-combined caching, with real consequences.**

---

## 3. One unit of work — `xe_sched_job_types.h`

```c
struct xe_sched_job {
	struct drm_sched_job drm;   // <<< the generic job (deps + fences)
	struct xe_exec_queue *q;    // which conveyor belt
	struct kref refcount;

	struct dma_fence *fence;    // signals when the GPU finishes this job
	u32 lrc_seqno;              // the sequence number written into the ring

	struct {
		bool used;
		u64 addr;           // userspace wants a value written here
		u64 value;          // ...this value...
	} user_fence;               // ...when the job completes

	bool ring_ops_flush_tlb;    // prepend a TLB flush (ch 9!)
	struct xe_job_ptrs ptrs[];  // the batch buffer address(es)
};
```

Same embedding pattern as all of Act I: **`xe_sched_job` wraps
`drm_sched_job`.** The generic part holds dependencies and fences; Xe's part
holds the batch address and the hardware specifics.

And the generic part's two key fields (`include/drm/gpu_scheduler.h`):

```c
struct drm_sched_job {
	struct drm_sched_fence *s_fence;    // the "scheduled" + "finished" fences
	struct drm_sched_entity *entity;
	struct xarray dependencies;         // <<< IDEA 3: what we're waiting for
};
```

`dependencies` is the whole reason `drm_sched` exists.

---

## 4. The door — `xe_gpu_scheduler_types.h`

```c
struct xe_gpu_scheduler {
	struct drm_gpu_scheduler base;        // the generic scheduler
	const struct xe_sched_backend_ops *ops;

	struct list_head msgs;                // <<< Xe's addition
	spinlock_t msg_lock;
	struct work_struct work_process_msg;
};

#define xe_sched_entity  drm_sched_entity     // just an alias
```

Xe adds **messages** to `drm_sched`. Why? Because some operations on a queue
(suspend it, resume it, change its priority, tear it down) must be **ordered
with respect to its jobs** — you can't change a queue's priority halfway through
submitting. Putting them in the same queue as the jobs gives that ordering free.
Chapter 12 uses them.

### The wiring — `xe_guc_submit.c:1986`

Where the 1:1:1 relationship is actually created:

```c
static int guc_exec_queue_init(struct xe_exec_queue *q)
{
	timeout = (q->vm && xe_vm_in_lr_mode(q->vm)) ? MAX_SCHEDULE_TIMEOUT :
		  msecs_to_jiffies(q->sched_props.job_timeout_ms);
	//        ^^^ LR-mode queues (ch 6) NEVER time out - they're supposed to
	//            run forever. That's what makes them "long running".

	err = alloc_guc_id(guc, q);         // <<< the GuC-side context ID

	err = xe_sched_init(&ge->sched, &drm_sched_ops, &xe_sched_ops,
			    submit_wq,
			    xe_lrc_ring_size() / MAX_JOB_SIZE_BYTES,   // <<< !!
			    64, timeout, ...);
	//                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
	//  The scheduler's queue depth = how many jobs FIT IN THE RING.
	//  drm_sched will not hand us more jobs than the ring can hold.
	//  Flow control, derived from Idea 2's buffer size.
}
```

That credit limit is my favourite line in the file. **`drm_sched`'s back-pressure
is literally the ring's capacity** — the waiting room is sized by the door.

---

## 5. The backend interface — `xe_exec_queue_types.h`

Everything the rest of the driver can ask of a submission backend:

```c
struct xe_exec_queue_ops {
	int  (*init)(struct xe_exec_queue *q);
	void (*kill)(struct xe_exec_queue *q);
	void (*fini)(struct xe_exec_queue *q);

	int  (*set_priority)(struct xe_exec_queue *q, enum ... priority);
	int  (*set_timeslice)(struct xe_exec_queue *q, u32 timeslice_us);
	int  (*set_preempt_timeout)(struct xe_exec_queue *q, u32 us);
	// ^^^ all three become MESSAGES to the GuC (Idea 3)

	int  (*suspend)(struct xe_exec_queue *q);
	int  (*suspend_wait)(struct xe_exec_queue *q);
	void (*resume)(struct xe_exec_queue *q);
	// ^^^ preemption, used by ch 18's preempt fences

	bool (*reset_status)(struct xe_exec_queue *q);
};
```

Notice what is **not** here: no "run this job". That's `drm_sched`'s
`run_job()`, a different vtable. This one is lifecycle and policy.

And the two implementations, same shape:

```
   xe_guc_submit.c:1976      static const struct drm_sched_backend_ops drm_sched_ops
   xe_execlist.c:321         static const struct drm_sched_backend_ops drm_sched_ops
```

---

## Bonus: what a job looks like in the ring

Not strictly cast, but it makes Idea 2 concrete. `xe_ring_ops.c`,
`__emit_job_gen12_simple()` — the driver builds an array of command dwords and
writes it:

```c
	u32 dw[MAX_JOB_SIZE_DW], i = 0;

	*head = lrc->ring.tail;                 // remember where this job starts

	i = emit_copy_timestamp(gt_to_xe(gt), lrc, dw, i);   // for stats

	if (job->ring_ops_flush_tlb) {
		dw[i++] = preparser_disable(true);
		i = emit_flush_imm_ggtt(xe_lrc_start_seqno_ggtt_addr(lrc),
					seqno, MI_INVALIDATE_TLB, dw, i);
		//                             ^^^^^^^^^^^^^^^^ CHAPTER 9,
		//   as a single command inline in the ring. Much cheaper than
		//   a GuC round trip - available because it's ordered naturally.
		dw[i++] = preparser_disable(false);
	} else {
		i = emit_store_imm_ggtt(xe_lrc_start_seqno_ggtt_addr(lrc),
					seqno, dw, i);        // "job started"
	}

	i = emit_bb_start(batch_addr, ppgtt_flag, dw, i);
	//  ^^^^^^^^^^^^^ MI_BATCH_BUFFER_START - "go execute USERSPACE's
	//                commands at this PPGTT address", then come back

	dw[i++] = MI_ARB_ON_OFF | MI_ARB_DISABLE;   // don't preempt during
						    // fence signalling

	if (job->user_fence.used)
		i = emit_store_imm_ppgtt_posted(job->user_fence.addr,
						job->user_fence.value, dw, i);

	i = emit_flush_imm_ggtt(xe_lrc_seqno_ggtt_addr(lrc), seqno, 0, dw, i);
	//  ^^^ "job finished" - this write is what makes the completion fence
	//      signal

	i = emit_user_interrupt(dw, i);             // poke the CPU

	xe_lrc_write_ring(lrc, dw, i * sizeof(*dw));   // <<< into the ring
```

Read it as a sandwich: **bookkeeping, "I started", go run the user's batch,
"I finished", wake the CPU.** The user's work is one command in the middle. And
`MAX_JOB_SIZE_DW` is **74** — the entire wrapper is at most 74 dwords.

---

## The cast, with line numbers

```
 xe_exec_queue          xe_exec_queue_types.h:92    conveyor belt
   ->  xe_lrc[]         xe_lrc_types.h:17           state + ring (one xe_bo)
   ->  drm_sched_entity include/drm/gpu_scheduler.h  in-order job queue
   ->  xe_gpu_scheduler xe_gpu_scheduler_types.h     the door + messages
   ->  xe_guc_exec_queue xe_guc_exec_queue_types.h   GuC context id + state

 xe_sched_job           xe_sched_job_types.h         one EXEC
 xe_exec_queue_ops      xe_exec_queue_types.h        lifecycle + policy vtable
 xe_ring_ops            xe_ring_ops_types.h          emit_job: job -> commands

 wiring:  guc_exec_queue_init()      xe_guc_submit.c:1986
 ring:    xe_lrc_write_ring()        xe_lrc.c:1826
 commands:__emit_job_gen12_simple()  xe_ring_ops.c
```

## Try it on your machine

```bash
# every exec queue your session creates, and every job on it
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_exec_queue_create/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_sched_job_run/enable
cat /sys/kernel/debug/tracing/trace_pipe

# how big is the ring, and therefore the scheduler's credit limit?
grep -rn 'xe_lrc_ring_size' drivers/gpu/drm/xe/xe_lrc.c
```

## Checkpoint

1. There is no `head` field in `struct xe_lrc`. Why not?
2. `xe_lrc->bo` is in VRAM, but `xe_lrc->seqno_bo` is always system memory. Why
   the difference?
3. `xe_sched_init()` passes `xe_lrc_ring_size() / MAX_JOB_SIZE_BYTES` as the
   scheduler's credit limit. What is that doing, and why that number?
4. `xe_exec_queue` stores `timeslice_us` and `priority`, but the kernel never
   acts on them. Who does?
5. What is the single most important barrier in `xe_lrc_write_ring()`, and what
   breaks without it?

<details>
<summary>answers</summary>

1. Because the driver never writes it. The hardware owns the head pointer —
   it advances as the engine consumes commands and publishes it in the status
   page. The driver only writes the tail. The missing field is the read/write
   split made visible.
2. The LRC image and ring are GPU-read, so VRAM is right. `seqno_bo` is the
   opposite direction: the GPU *writes* completion numbers and the **CPU reads**
   them. CPU reads from VRAM through the BAR are very slow, so Chapter 3's rule
   says use cached system memory. The struct comment says exactly this.
3. It's flow control. `drm_sched` will not hand the driver more jobs than are
   credited, and the credit count is how many maximum-size jobs fit in the ring.
   So the scheduler can never be asked to write a job the ring has no room for.
4. The GuC. The driver stores those properties and sends them to the GuC (via
   scheduler messages); the GuC enforces time-slicing, priority and preemption
   on the actual engines. That's Idea 3.
5. `xe_device_wmb(xe)` at the end. The ring and the context image are
   write-combined, so writes can become visible out of order. The tail update is
   what tells the hardware "there are new commands". If the tail became visible
   before the commands themselves, the engine would start executing whatever was
   previously in that part of the ring.

</details>
