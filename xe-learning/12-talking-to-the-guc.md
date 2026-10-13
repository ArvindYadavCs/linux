# Chapter 12 — Talking to the GuC

> Beat A: the story.

Three times now I've written *"...and tell the GuC"* and moved on. This chapter
is what that actually means.

The spine:

> *A job's dependencies have all signalled. `run_job()` has been called.*
> *Get this work onto an engine.*

Three ideas.

---

## Idea 1: The channel is a ring buffer in memory

The GuC is a separate processor on the other side of the chip. The driver can't
call its functions. It has to send it messages.

And the mechanism will look familiar, because it's **exactly the ring buffer
from Chapter 10**, used again:

* a **shared buffer in memory**, which both sides can read and write
* the **sender** writes at the tail and advances it
* the **receiver** reads from the head and advances it
* a small **descriptor** next to it holds the head and tail

There are **two** of them, because messages go both ways:

```
   H2G  (Host to GuC)          "register this context"
                               "this context has work"
      driver writes ---------------------------> GuC reads

   G2H  (GuC to Host)          "that context is now scheduled"
                               "that context hung"
      GuC writes   <--------------------------- driver reads
```

Together they're called **CT** — Command Transport. **CTB** is one of the two
buffers.

Two details worth having now:

**They live in GGTT.** Of course they do — the GuC can only reach memory through
GGTT (Chapter 5, Idea 3). This is one of the GGTT users from that chapter's
table.

**The driver notifies, it doesn't wait.** After writing a message the driver
bumps the tail in the descriptor and carries on. The GuC picks it up when it
gets to it. Replies arrive later as an interrupt, which queues a worker to drain
the G2H buffer.

So the whole conversation is asynchronous in both directions.

---

## Idea 2: "Tell the GuC" is a two-word message

Here is the punchline, and it's smaller than you'd expect.

After Chapter 10's ring writing — which put the actual commands in the LRC's
ring and bumped its tail — the message to the GuC is:

```
   [ XE_GUC_ACTION_SCHED_CONTEXT ,  guc_id ]
```

**Two dwords.** An action code, and the GuC's ID for this context. That's the
entire submission.

Not the batch address. Not the job. Not the ring contents. Just *"context 37 has
work."* The GuC already knows everything else, because the context was
**registered** earlier.

### Registration is where the real information goes

The first time a queue is used, the driver sends a bigger message:

```
   [ XE_GUC_ACTION_REGISTER_CONTEXT,
     flags, context_idx, engine_class, engine_submit_mask,
     ... ,
     hwlrca_lo, hwlrca_hi ]            <-- the LRC descriptor
```

That `hwlrca` is the 64-bit descriptor from the LRC appendix:
`flags | GGTT address of lrc->bo`. **Everything the GuC needs to know about the
context, it learns by following that address** — the ring, the register state,
the page table root, all of it.

So the division of labour is:

| When | Message | Carries |
|------|---------|---------|
| once, on first use | `REGISTER_CONTEXT` | where the context lives (its LRC) |
| every submission | `SCHED_CONTEXT` | just "this one has work" |

Register once, then poke cheaply forever. That's why a submission is two dwords.

### And a third state: enabled

There's one wrinkle. A registered context can be **scheduling-enabled** or not.
A context with no work gets disabled so the GuC doesn't waste time considering
it. So the *first* submission after an idle period sends a slightly bigger
message — *"turn scheduling on for context 37"* — and that one **does** get a
reply, because the driver needs to know when it took effect.

Three states, and you'll see all three in the code:

```
   not registered  --REGISTER_CONTEXT-->  registered, disabled
                                                |
                                      SCHED_CONTEXT_MODE_SET(ENABLE)
                                                |
                                                v
                                          registered, enabled
                                                |
                                          SCHED_CONTEXT  (x N)
```

---

## Idea 3: Queue operations travel with the jobs

Last idea, and it's the reason Chapter 10's `xe_gpu_scheduler` had a message
list bolted onto it.

Some things you do to a queue aren't jobs, but **must be ordered against the
jobs**:

* suspend it
* resume it
* change its priority or timeslice
* tear it down

Consider why ordering matters: if you tear down a queue while a job is still in
flight, you free a context the GuC is actively running. If you change priority
"at the same time" as submitting, the result is undefined.

The obvious fix would be a lock around everything. Xe does something cheaper:

> **Put those operations in the same queue as the jobs.**

The scheduler already processes jobs in order. Adding a message list that is
drained by the same machinery means a message is naturally ordered against the
jobs before and after it — no extra locking, no races.

Four opcodes, and that's the whole set:

```c
#define CLEANUP          1     /* tear this queue down */
#define SET_SCHED_PROPS  2     /* priority / timeslice / preempt timeout */
#define SUSPEND          3     /* stop running it (ch 14's preempt fences) */
#define RESUME           4     /* start again */
```

Each one becomes an H2G message to the GuC when the scheduler gets to it.

---

## The picture

```
                     THE DRIVER                   |        THE GuC
                                                  |
   run_job()                                      |
     |                                            |
     +-- write commands into the LRC's ring (ch10)|
     +-- bump lrc->ring.tail                      |
     |                                            |
     +-- is this context registered?              |
     |     no -> H2G: REGISTER_CONTEXT            |
     |              { ..., hwlrca = GGTT addr }---+--> records it
     |                                            |
     +-- is scheduling enabled?                   |
     |     no -> H2G: SCHED_CONTEXT_MODE_SET -----+--> enables it
     |              { guc_id, ENABLE }            |    ...later...
     |            <-------------------------------+--- G2H: MODE_DONE
     |                                            |
     +-- H2G: SCHED_CONTEXT { guc_id } -----------+--> "has work"
     |                                            |
     +-- return the hardware fence                |    GuC now decides
                                                  |    when & where to run it
   ... engine executes the ring ...               |
   ... LRC seqno written -> fence signals ...     |
```

## Remember these three

1. **The channel is two ring buffers in GGTT** — H2G down, G2H up — and both
   directions are asynchronous.
2. **Submission is two dwords: an action code and a context ID.** All the real
   information went across once, at registration, as the LRC's GGTT descriptor.
3. **Queue operations (suspend, resume, priority, teardown) travel as messages
   in the same queue as the jobs**, which is how they get ordered for free.

---
---

# Chapter 12 — Beat B: The Code

Five functions, following one submission down:

| Function | File | Job |
|----------|------|-----|
| `guc_exec_queue_run_job()` | `xe_guc_submit.c:1205` | the `drm_sched` callback |
| `register_exec_queue()` | `xe_guc_submit.c:985` | first-use registration |
| `submit_exec_queue()` | `xe_guc_submit.c:1145` | **the two-dword poke** |
| `h2g_write()` | `xe_guc_ct.c:941` | put a message in the ring |
| `process_g2h_msg()` | `xe_guc_ct.c:1655` | dispatch replies |

> Code trimmed: parallel submission, multi-queue, SR-IOV and replay paths
> removed. `//` comments are mine.

---

## Step 0 — The two buffers

**File: `xe_guc_ct_types.h`**

```c
struct guc_ctb {
	struct xe_bo *bo;           // the shared buffer (in GGTT - ch 5)
	struct iosys_map desc;      // CPU map of the DESCRIPTOR (head/tail/status)
	struct iosys_map cmds;      // CPU map of the MESSAGE AREA
	struct guc_ctb_info info;
};

struct guc_ctb_info {
	u32 size;        // in dwords
	u32 head;        // where the reader is
	u32 tail;        // where the writer is
	u32 space;       // how much room is left
	bool broken;     // channel is dead
};

struct xe_guc_ct {
	struct mutex lock;            // one sender at a time
	struct {
		struct guc_ctb h2g;   // <<< IDEA 1: down
		struct guc_ctb g2h;   // <<< IDEA 1: up
	} ctbs;

	u32 g2h_outstanding;          // how many replies we're still owed
	struct work_struct g2h_worker;// drains g2h after an interrupt

	enum xe_guc_ct_state state;
	u32 fence_seqno;              // numbers our requests
	struct xarray fence_lookup;   // seqno -> who is waiting for the reply
};
```

Compare `guc_ctb_info` with `xe_lrc->ring` from Chapter 10 — `size`, `head`,
`tail`. **Same data structure, different participants.** The LRC ring is
driver→engine; this one is driver→GuC.

And `fence_lookup` is how an asynchronous reply finds its waiter: each request
carries a sequence number, the reply carries it back, and the xarray maps it to
whoever asked.

---

## Step 1 — `run_job()`: the door from Chapter 11

**File: `xe_guc_submit.c:1205`.** This is the callback `drm_sched` invoked at the
end of Chapter 11.

```c
static struct dma_fence *guc_exec_queue_run_job(struct drm_sched_job *drm_job)
{
	struct xe_sched_job *job = to_xe_sched_job(drm_job);
	struct xe_exec_queue *q = job->q;
	bool killed_or_banned_or_wedged = exec_queue_killed_or_banned_or_wedged(q);

	trace_xe_sched_job_run(job);

	if (!killed_or_banned_or_wedged && !xe_sched_job_is_error(job)) {
		if (!exec_queue_registered(q))
			register_exec_queue(q, GUC_CONTEXT_NORMAL);  // <<< step 2
			//  ^^^ FIRST USE ONLY (Idea 2)

		if (!job->restore_replay)
			q->ring_ops->emit_job(job);                  // <<< CHAPTER 10
			//  ^^^ write the command sandwich into the LRC's ring

		submit_exec_queue(q, job);                           // <<< step 3
	}

run_job_out:
	return job->fence;
	//     ^^^^^^^^^^ Chapter 11's "parent" - the hardware fence that was
	//     pre-allocated at job creation. It signals when the LRC's seqno
	//     counter reaches this job's number.
}
```

**Three lines of substance**, and they are the three ideas in order: register if
needed, write the ring (Chapter 10), poke the GuC.

Note the guard: a killed or banned queue skips all of it and the function still
returns a fence — which will signal with an error. `drm_sched` must always get a
fence back, even for work that will never run.

---

## Step 2 — Registration: the one big message

**File: `xe_guc_submit.c:985`** — gather the information:

```c
	memset(&info, 0, sizeof(info));
	info.context_idx  = q->guc->id;                      // the GuC's ID for us
	info.engine_class = xe_hwe_to_guc_class(q->hwe);     // render / copy / ...
	info.engine_submit_mask =
		xe_hwe_guc_logical_to_submit_mask(q->hwe, q->logical_mask);
	//      ^^^ which engine instances this queue may run on

	info.hwlrca_lo = lower_32_bits(xe_lrc_descriptor(lrc));
	info.hwlrca_hi = upper_32_bits(xe_lrc_descriptor(lrc));
	//   ^^^^^^^ IDEA 2. The LRC appendix's descriptor:
	//           flags | GGTT address of lrc->bo.
	//           This is the ONLY pointer the GuC gets, and everything about
	//           the context hangs off it.

	info.flags = CONTEXT_REGISTRATION_FLAG_KMD |
		FIELD_PREP(CONTEXT_REGISTRATION_FLAG_TYPE, ctx_type);
```

**`xe_guc_submit.c:935`** — send it:

```c
static void __register_exec_queue(struct xe_guc *guc,
				  struct guc_ctxt_registration_info *info)
{
	u32 action[] = {
		XE_GUC_ACTION_REGISTER_CONTEXT,
		info->flags,
		info->context_idx,
		info->engine_class,
		info->engine_submit_mask,
		info->wq_desc_lo,  info->wq_desc_hi,     // parallel submission only
		info->wq_base_lo,  info->wq_base_hi,
		info->wq_size,
		info->hwlrca_lo,   info->hwlrca_hi,      // <<< the LRC address
	};

	xe_guc_ct_send(&guc->ct, action, ARRAY_SIZE(action), 0, 0);
	//                                                   ^  ^
	//                                        no reply expected
}
```

**Twelve dwords, sent once per queue.** A plain array of `u32` — that is
genuinely what an H2G message is. And the last two arguments being `0, 0` mean
"no G2H reply expected"; registration is fire-and-forget.

---

## Step 3 — The poke

**File: `xe_guc_submit.c:1145`.** The function Idea 2 is about:

```c
static void submit_exec_queue(struct xe_exec_queue *q, struct xe_sched_job *job)
{
	struct xe_lrc *lrc = q->lrc[0];
	u32 action[3];
	int len = 0;

	xe_gt_assert(guc_to_gt(guc), exec_queue_registered(q));
	//                           ^^^ we must already be registered

	xe_lrc_set_ring_tail(lrc, lrc->ring.tail);
	//  ^^^ CHAPTER 10: publish the tail into the context image, so the
	//      engine will see the commands emit_job() just wrote.

	if (!exec_queue_enabled(q)) {
		// ---- FIRST submission after idle: turn scheduling ON ----
		action[len++] = XE_GUC_ACTION_SCHED_CONTEXT_MODE_SET;
		action[len++] = q->guc->id;
		action[len++] = GUC_CONTEXT_ENABLE;
		g2h_len = G2H_LEN_DW_SCHED_CONTEXT_MODE_SET;
		num_g2h = 1;                  // <<< this one DOES want a reply
		set_exec_queue_pending_enable(q);
		set_exec_queue_enabled(q);
	} else {
		// ---- STEADY STATE: just say "I have work" ----
		action[len++] = XE_GUC_ACTION_SCHED_CONTEXT;
		action[len++] = q->guc->id;
		//              ^^^^^^^^^^ TWO DWORDS. THAT IS THE SUBMISSION.
	}

	xe_guc_ct_send(&guc->ct, action, len, g2h_len, num_g2h);
}
```

**That `else` branch is the hot path of the entire driver.** Every frame, every
compute dispatch, every blit — millions of times — reduces to two dwords in a
ring buffer.

And the `if` is Idea 2's third state. Note `num_g2h = 1`: enabling scheduling is
the only one of the two that expects a reply, because the driver must know when
it has taken effect (for suspend/resume bookkeeping).

---

## Step 4 — Putting a message in the ring

**File: `xe_guc_ct.c:941`.** Same circular-buffer shape as Chapter 10, with one
new trick.

```c
static int h2g_write(struct xe_guc_ct *ct, const u32 *action, u32 len,
		     u32 ct_fence_value, bool want_response)
{
	struct guc_ctb *h2g = &ct->ctbs.h2g;
	u32 tail = h2g->info.tail;
	struct iosys_map map = IOSYS_MAP_INIT_OFFSET(&h2g->cmds,
						     tail * sizeof(u32));
	full_len = len + GUC_CTB_HDR_LEN;

	/* Command will wrap, zero fill (NOPs), return and check credits again */
	if (tail + full_len > h2g->info.size) {
		xe_map_memset(xe, &map, 0, 0, (h2g->info.size - tail) * sizeof(u32));
		h2g->info.tail = 0;
		desc_write(xe, h2g, tail, h2g->info.tail);
		return -EAGAIN;
	}
	// ^^^ DIFFERENT from the LRC ring! Chapter 10 SPLIT a wrapping write in
	//     two. Here a message must be contiguous, so instead we pad the end
	//     with zeros, reset the tail to 0, and return -EAGAIN so the caller
	//     retries from the start of the buffer.

	// The two-dword CT header: format, length, and our sequence number
	cmd[0] = FIELD_PREP(GUC_CTB_MSG_0_FORMAT, GUC_CTB_FORMAT_HXG) |
		 FIELD_PREP(GUC_CTB_MSG_0_NUM_DWORDS, len) |
		 FIELD_PREP(GUC_CTB_MSG_0_FENCE, ct_fence_value);
	//                                    ^^^^^^^^^^^^^^^ the number the
	//                                    reply will carry back

	if (want_response)
		cmd[1] = MAKE_ACTION(GUC_HXG_TYPE_REQUEST, action[0]);
	else
		cmd[1] = MAKE_ACTION(GUC_HXG_TYPE_FAST_REQUEST, action[0]);
	//                       ^^^^^^^^^^^^^^^^^^^^^^^^^ "don't reply to this"
	//   FAST_REQUEST is the cheap form used for SCHED_CONTEXT.

	/* Write H2G ensuring visible before descriptor update */
	xe_map_memcpy_to(xe, &map, 0, cmd, H2G_CT_HEADERS * sizeof(u32));
	xe_map_memcpy_to(xe, &map, H2G_CT_HEADERS * sizeof(u32),
			 action, len * sizeof(u32));
	xe_device_wmb(xe);
	//  ^^^ THE SAME BARRIER AS CHAPTER 10, for the same reason: the tail
	//      update is what publishes the message. If the tail became visible
	//      first, the GuC would read garbage.

	h2g->info.tail = (tail + full_len) % h2g->info.size;
	desc_write(xe, h2g, tail, h2g->info.tail);
	//  ^^^ THE NOTIFICATION. Writing the tail into the shared descriptor
	//      is how the GuC learns there's a message. No doorbell register,
	//      no interrupt from this side.
	return 0;
}
```

Three things to take from this:

* **`xe_device_wmb()` again.** Exactly the barrier from Chapter 10's
  `xe_lrc_write_ring()`, with the same comment and the same reason. Two ring
  buffers, one discipline.
* **Wrapping is handled differently.** The LRC ring splits a write; CT pads and
  retries. Because a CT *message* must be contiguous to be parsed, while a ring
  of GPU *commands* doesn't care where it wraps.
* **`FAST_REQUEST`** is the no-reply form. Submissions use it; that's part of why
  they're cheap.

### Who holds the lock

`xe_guc_ct_send()` at `:1274` wraps this in `ct->lock`, and there is flow control
above it: `__guc_ct_send_locked()` at `:1053` checks there's room in H2G **and**
that there's reserved space in G2H for however many replies this request will
produce. If not, it drains G2H first. A conversation can't be started that
there's no room to finish.

---

## Step 5 — Replies

**File: `xe_guc_ct.c:1655`.** When the GuC writes into G2H it raises an
interrupt; the handler queues `g2h_worker`, which drains the buffer and calls:

```c
static int process_g2h_msg(struct xe_guc_ct *ct, u32 *msg, u32 len)
{
	action = FIELD_GET(GUC_HXG_EVENT_MSG_0_ACTION, hxg[0]);

	switch (action) {
	case XE_GUC_ACTION_SCHED_CONTEXT_MODE_DONE:
		ret = xe_guc_sched_done_handler(guc, payload, adj_len);
		//    ^^^ "scheduling is now enabled/disabled on that context"
		//        - the reply to Idea 2's third state
		break;
	case XE_GUC_ACTION_DEREGISTER_CONTEXT_DONE:
		ret = xe_guc_deregister_done_handler(guc, payload, adj_len);
		//    ^^^ "that context is fully gone, safe to free its LRC"
		break;
	case XE_GUC_ACTION_CONTEXT_RESET_NOTIFICATION:
		ret = xe_guc_exec_queue_reset_handler(guc, payload, adj_len);
		//    ^^^ "that context hung and I reset it"  (chapter 15)
		break;
	case XE_GUC_ACTION_ENGINE_FAILURE_NOTIFICATION:
		ret = xe_guc_exec_queue_reset_failure_handler(...);
		break;
	case XE_GUC_ACTION_NOTIFY_MEMORY_CAT_ERROR:
		ret = xe_guc_exec_queue_memory_cat_error_handler(...);
		//    ^^^ a bad memory access - a page table problem (ch 7, 9)
		break;
	case XE_GUC_ACTION_REPORT_PAGE_FAULT_REQ_DESC:
		ret = xe_guc_pagefault_handler(guc, payload, adj_len);
		//    ^^^ fault-mode VM took a page fault (ch 6's personality)
		break;
	case XE_GUC_ACTION_STATE_CAPTURE_NOTIFICATION:
		ret = xe_guc_error_capture_handler(guc, payload, adj_len);
		break;
	...
	}
}
```

**Read that switch as "everything the GuC can tell you."** It is a short list,
and almost all of it is bad news — hangs, resets, faults, memory errors. The GuC
is quiet when things go well.

Notice there is **no "job complete" message.** Completion doesn't come through
the CT at all: the engine writes a sequence number into the LRC's `seqno_bo`
(Chapter 10's ring sandwich) and raises a user interrupt, and *that* signals the
hardware fence. Routing every completion through a microcontroller mailbox would
be far too slow.

> **Jobs complete through memory and an interrupt. The CT is for control and for
> trouble.**

---

## Step 6 — The message mechanism

**File: `xe_guc_submit.c:1938`** — Idea 3's whole vocabulary:

```c
#define CLEANUP          1     /* Non-zero values to catch uninitialized msg */
#define SET_SCHED_PROPS  2
#define SUSPEND          3
#define RESUME           4
```

Added to the queue like this (`:2095`):

```c
static void guc_exec_queue_add_msg(struct xe_exec_queue *q,
				   struct xe_sched_msg *msg, u32 opcode)
{
	msg->opcode = opcode & OPCODE_MASK;
	msg->private_data = q;

	if (opcode & MSG_HEAD)
		xe_sched_add_msg_head(&q->guc->sched, msg);   // jump the queue
	else
		xe_sched_add_msg(&q->guc->sched, msg);        // <<< ordered
							      // behind the jobs
}
```

And each one, when the scheduler reaches it, becomes an H2G message. For example
`SUSPEND` sends `SCHED_CONTEXT_MODE_SET` with `GUC_CONTEXT_DISABLE` and waits
for the `MODE_DONE` reply from step 5.

**That `MSG_HEAD` option is worth noting**: teardown can jump ahead of pending
jobs, because if the queue is being destroyed there's no point running them.
Everything else goes to the back, which is Idea 3's ordering guarantee.

These are what Chapter 10's `xe_exec_queue_ops` entries actually do —
`set_priority`, `suspend`, `resume`, `fini` are all "post a message".

---

## Our submission, end to end

```
  Chapter 11 ends: all dependencies signalled, worker calls run_job()
        |
  guc_exec_queue_run_job()              xe_guc_submit.c:1205
        |
        +-- not registered? -> register_exec_queue()    :985
        |     H2G[12]: REGISTER_CONTEXT { ..., hwlrca = GGTT addr of LRC }
        |
        +-- q->ring_ops->emit_job(job)                  CHAPTER 10
        |     writes the command sandwich into the LRC ring
        |
        +-- submit_exec_queue()                         :1145
        |     xe_lrc_set_ring_tail()  -> publish the tail
        |     not enabled? H2G[3]: SCHED_CONTEXT_MODE_SET { id, ENABLE }
        |     else         H2G[2]: SCHED_CONTEXT         { id }
        |         |
        |         +-- xe_guc_ct_send()                  xe_guc_ct.c:1274
        |               h2g_write()                     :941
        |                 memcpy into the H2G ring
        |                 xe_device_wmb()
        |                 desc_write(tail)   <-- the GuC now knows
        |
        +-- return job->fence                           (Chapter 11's "parent")

  ... GuC schedules the context onto an engine ...
  ... engine reads the ring, runs the batch, writes seqno ...
  ... user interrupt -> hardware fence signals -> "finished" signals ...

  meanwhile, if anything goes wrong:
     GuC writes G2H -> interrupt -> g2h_worker -> process_g2h_msg()  :1655
```

## Try it on your machine

```bash
# every message in both directions
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_guc_ctb_h2g/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_guc_ctb_g2h/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_exec_queue_submit/enable
cat /sys/kernel/debug/tracing/trace_pipe

# the CT state, and the GuC's own log
cat /sys/kernel/debug/dri/0/gt0/guc_ct
cat /sys/kernel/debug/dri/0/gt0/guc_log
```

Run something graphical with `xe_guc_ctb_h2g` enabled and watch the flood of
2-dword `SCHED_CONTEXT` messages. That's Idea 2, live.

## Checkpoint

1. A submission sends only an action code and a context ID. How does the GuC know
   where the batch buffer is?
2. `h2g_write()` pads the buffer with zeros and returns `-EAGAIN` when a message
   would wrap, but `xe_lrc_write_ring()` splits the write in two. Why the
   difference?
3. What tells the GuC that a new message is waiting?
4. There is no "job complete" message in `process_g2h_msg()`. How does the driver
   learn a job finished?
5. Why does Xe put suspend/resume/priority changes in the scheduler's queue
   instead of just calling them directly?

<details>
<summary>answers</summary>

1. It doesn't, and it doesn't need to. The batch address was written into the
   LRC's ring by `emit_job()` as an `MI_BATCH_BUFFER_START` command. The GuC only
   needs to know *which context* has work; the context's LRC — whose GGTT address
   it was given at registration — contains the ring, and the ring contains the
   command.
2. Because a CT message has a header declaring its length and must be contiguous
   to be parsed. A ring of GPU commands has no such constraint — the command
   streamer simply wraps. So CT pads to the end and retries from offset 0, while
   the LRC ring splits.
3. The driver writing the new tail value into the shared CT descriptor. There is
   no doorbell register and no interrupt in the H2G direction — the GuC polls the
   descriptor.
4. Through memory and an interrupt, not the CT. The engine's last commands write
   a sequence number into the LRC's `seqno_bo` and raise a user interrupt; the
   interrupt handler signals the hardware fence whose seqno has been reached.
   Routing completions through the mailbox would be far too slow.
5. Ordering. Those operations must not interleave with job submission — tearing
   down a queue with jobs in flight, or changing priority mid-submit, is
   undefined. Putting them in the same in-order queue as the jobs makes them
   ordered against the jobs for free, with no additional locking.

</details>
