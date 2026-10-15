# Chapter 12 — Talking to the GuC

> Beat A: the story.

Three times I've written *"...and tell the GuC"* and moved on. Here is what that
one phrase actually is.

> *A job's dependencies have signalled. `run_job()` has been called.*
> *Get this work onto an engine.*

Three ideas, and they are short.

---

## Idea 1: A mailbox in memory

The GuC is a separate processor. The driver can't call its functions, so it
leaves **letters in a mailbox**.

The mailbox is a ring buffer in memory — **the same structure as Chapter 10's
command ring**. The sender writes at the tail; the receiver reads from the head;
a small descriptor beside it holds both numbers.

There are two mailboxes, because letters go both ways:

```
   H2G  "Host to GuC"      driver writes  --->  GuC reads
   G2H  "GuC to Host"      driver reads   <---  GuC writes
```

Together they're called **CT** (Command Transport).

Two facts and that's all you need:

* **They live in GGTT**, because that's the only memory the GuC can reach
  (Chapter 5).
* **Nobody waits.** You drop a letter in and carry on. Replies arrive later as
  an interrupt.

---

## Idea 2: The letter is two words long

Here's the punchline.

After Chapter 10's work — writing the actual commands into the LRC's ring and
bumping its tail — the letter to the GuC is:

```
   [ SCHED_CONTEXT ,  guc_id ]
```

**Two dwords.** An action code, and the GuC's ID number for this context.

Not the batch address. Not the job. Not the commands. Just *"context 37 has
work."*

### Why so little?

Because everything else was said **once**, earlier. The first time a queue is
used, the driver sends a bigger letter:

```
   [ REGISTER_CONTEXT, ..., hwlrca ]
```

where `hwlrca` is the LRC descriptor from the appendix — **the GGTT address of
`lrc->bo`**. One pointer. The GuC follows it to find the ring, the register
state, the page table root, everything.

So:

| When | Letter | Says |
|------|--------|------|
| once, on first use | `REGISTER_CONTEXT` | "this context lives at *this address*" |
| every submission | `SCHED_CONTEXT` | "it has work" |

**Register once. Poke cheaply, forever.** That second line is the hot path of the
whole driver.

---

## Idea 3: The GuC only writes back when something is wrong

You'd expect a "job finished" letter in the G2H mailbox. **There isn't one.**

Job completion goes a completely different way, and we already saw it in
Chapter 10: the engine's last commands write a sequence number into memory and
raise an interrupt. The interrupt handler signals the fence. No mailbox
involved.

Why? Speed. Every frame completes many jobs; routing each one through a
microcontroller's mailbox would be far too slow.

So what *does* come back on G2H? Almost entirely bad news:

* "that context hung, and I reset it"
* "a context took a page fault"
* "there was a bad memory access"
* plus a few control acknowledgements

> **Work completes through memory and an interrupt. The mailbox is for control
> and for trouble.**

---

## The picture

```
      THE DRIVER                                 |     THE GuC
                                                 |
  run_job()                                      |
    |                                            |
    1. write the commands into the LRC's ring    |
       (Chapter 10) and bump its tail            |
    |                                            |
    2. first time only:                          |
       H2G letter: REGISTER_CONTEXT -------------+--> remembers the address
                   { ..., hwlrca }               |
    |                                            |
    3. every time:                               |
       H2G letter: SCHED_CONTEXT ----------------+--> "ok, it has work"
                   { guc_id }                    |
    |                                            |     GuC now decides
    4. return the hardware fence                 |     when and where to run it
                                                 |
  ... engine runs the ring ...
  ... writes a seqno to memory, raises an interrupt ...
  ... fence signals ...                              (NOT via the mailbox)
```

## Remember these three

1. **Two mailboxes in GGTT** — H2G down, G2H up. Nobody waits.
2. **A submission is two dwords**, because the context's address was sent once
   at registration.
3. **Completion comes through memory and an interrupt, not the mailbox.** The
   mailbox carries control messages and bad news.

---
---

# Chapter 12 — Beat B: The Code

One straight line, four steps:

> *`run_job()` is called → bytes land in the H2G mailbox.*

| Step | Function | File |
|------|----------|------|
| 1 | `guc_exec_queue_run_job()` | `xe_guc_submit.c:1205` |
| 2 | `register_exec_queue()` | `xe_guc_submit.c:985` |
| 3 | `submit_exec_queue()` | `xe_guc_submit.c:1145` |
| 4 | `h2g_write()` | `xe_guc_ct.c:941` |

> Code trimmed hard — error paths, parallel submission, SR-IOV and multi-queue
> removed. `//` comments are mine.

---

## Step 1 — Three things happen

**`xe_guc_submit.c:1205`.** This is the `run_job()` callback from Chapter 11.

```c
static struct dma_fence *guc_exec_queue_run_job(struct drm_sched_job *drm_job)
{
	struct xe_sched_job *job = to_xe_sched_job(drm_job);
	struct xe_exec_queue *q = job->q;

	if (!exec_queue_registered(q))
		register_exec_queue(q, GUC_CONTEXT_NORMAL);   // STEP 2 (first use)

	q->ring_ops->emit_job(job);                           // CHAPTER 10
						              // (write the ring)

	submit_exec_queue(q, job);                            // STEP 3

	return job->fence;      // Chapter 11's "parent" fence
}
```

**Three lines of substance.** Register if needed, write the commands, poke the
GuC. That's the whole function, and it's the whole chapter.

---

## Step 2 — Registration (once per queue)

**`xe_guc_submit.c:985`** gathers the facts:

```c
	info.context_idx  = q->guc->id;                   // our ID number
	info.engine_class = xe_hwe_to_guc_class(q->hwe);  // render / copy / video

	info.hwlrca_lo = lower_32_bits(xe_lrc_descriptor(lrc));
	info.hwlrca_hi = upper_32_bits(xe_lrc_descriptor(lrc));
	//   ^^^^^^^ IDEA 2: the GGTT address of lrc->bo.
	//           The only pointer the GuC ever gets.
```

**`xe_guc_submit.c:935`** sends them:

```c
	u32 action[] = {
		XE_GUC_ACTION_REGISTER_CONTEXT,
		info->flags,
		info->context_idx,
		info->engine_class,
		info->engine_submit_mask,
		...
		info->hwlrca_lo,                 // <<< the address
		info->hwlrca_hi,
	};

	xe_guc_ct_send(&guc->ct, action, ARRAY_SIZE(action), 0, 0);
```

**A letter is just a `u32[]`.** That's genuinely all an H2G message is — an array
of dwords, first one being the action code.

---

## Step 3 — The poke (every submission)

**`xe_guc_submit.c:1145`.** The function Idea 2 is about:

```c
static void submit_exec_queue(struct xe_exec_queue *q, struct xe_sched_job *job)
{
	struct xe_lrc *lrc = q->lrc[0];
	u32 action[3];
	int len = 0;

	xe_lrc_set_ring_tail(lrc, lrc->ring.tail);
	//  ^^^ Chapter 10: publish the tail, so the engine will see the
	//      commands emit_job() just wrote.

	action[len++] = XE_GUC_ACTION_SCHED_CONTEXT;
	action[len++] = q->guc->id;
	//              ^^^^^^^^^^ TWO DWORDS. THAT IS THE SUBMISSION.

	xe_guc_ct_send(&guc->ct, action, len, g2h_len, num_g2h);
}
```

That's it. Every frame, every blit, every compute dispatch — **two dwords**.

> One wrinkle I'm skipping: if the context has been idle, the GuC turns
> scheduling *off* for it, and the first submission after that sends a 3-dword
> "turn scheduling back on" letter instead (and that one does get a reply). The
> real function has an `if` for it. The steady-state path is the two dwords
> above.

---

## Step 4 — How the letter physically lands

**`xe_guc_ct.c:941`.** Same shape as Chapter 10's ring write:

```c
static int h2g_write(struct xe_guc_ct *ct, const u32 *action, u32 len, ...)
{
	struct guc_ctb *h2g = &ct->ctbs.h2g;
	u32 tail = h2g->info.tail;
	struct iosys_map map = IOSYS_MAP_INIT_OFFSET(&h2g->cmds,
						     tail * sizeof(u32));

	// 1. copy the letter in at the tail
	xe_map_memcpy_to(xe, &map, 0, cmd, H2G_CT_HEADERS * sizeof(u32));
	xe_map_memcpy_to(xe, &map, H2G_CT_HEADERS * sizeof(u32),
			 action, len * sizeof(u32));

	// 2. make sure it's visible BEFORE we announce it
	xe_device_wmb(xe);
	//  ^^^ the same barrier as Chapter 10, for the same reason

	// 3. advance our tail
	h2g->info.tail = (tail + full_len) % h2g->info.size;

	// 4. announce it, by writing the tail into the shared descriptor
	desc_write(xe, h2g, tail, h2g->info.tail);
	//  ^^^ THIS is how the GuC learns there's a letter. No doorbell
	//      register, no interrupt - it polls the descriptor.

	return 0;
}
```

**Four steps: copy, barrier, advance, announce.** Exactly the ring discipline
from Chapter 10, with the descriptor playing the part the ring tail played
there.

And the mailbox struct, for completeness (`xe_guc_ct_types.h`):

```c
struct guc_ctb_info {
	u32 size;        // in dwords
	u32 head;        // where the reader is
	u32 tail;        // where the writer is
};

struct xe_guc_ct {
	struct {
		struct guc_ctb h2g;    // down
		struct guc_ctb g2h;    // up
	} ctbs;
	struct work_struct g2h_worker;   // drains g2h after an interrupt
};
```

`size`/`head`/`tail` — **the same three fields as `xe_lrc->ring`.** One idea,
two users.

---

## What comes back

**`xe_guc_ct.c:1655`.** The GuC writes into G2H, raises an interrupt, and a
worker calls this:

```c
	switch (action) {
	case XE_GUC_ACTION_CONTEXT_RESET_NOTIFICATION:   // "that context hung"
	case XE_GUC_ACTION_REPORT_PAGE_FAULT_REQ_DESC:   // "it page faulted"
	case XE_GUC_ACTION_NOTIFY_MEMORY_CAT_ERROR:      // "bad memory access"
	case XE_GUC_ACTION_SCHED_CONTEXT_MODE_DONE:      // "scheduling enabled"
	case XE_GUC_ACTION_DEREGISTER_CONTEXT_DONE:      // "context fully gone"
	...
	}
```

**Idea 3, as a switch statement.** Hangs, faults, memory errors, and a few
acknowledgements. **No "job complete" case** — that arrives as an interrupt from
the engine instead.

---

## One last thing: suspend and resume

Chapter 10 mentioned that `xe_gpu_scheduler` has a message list. Here's why.

Suspending a queue, changing its priority or tearing it down must not happen
*while* a job is being submitted. So instead of locking, Xe puts those
operations **in the same in-order queue as the jobs**:

```c
#define CLEANUP          1     /* tear this queue down */
#define SET_SCHED_PROPS  2     /* priority / timeslice */
#define SUSPEND          3
#define RESUME           4
```

The scheduler drains jobs and messages from one list, so a message is
automatically ordered against the jobs around it. Each one becomes its own H2G
letter when the scheduler reaches it.

That's what Chapter 10's `xe_exec_queue_ops` entries actually do: post a
message.

---

## The whole flow

```
  run_job()                              xe_guc_submit.c:1205
     |
     1. emit_job()           -> commands into the LRC ring    (ch 10)
     |
     2. register_exec_queue() (first use only)                 :985
     |     letter: REGISTER_CONTEXT { ..., hwlrca = GGTT addr }
     |
     3. submit_exec_queue()                                    :1145
     |     set_ring_tail()
     |     letter: SCHED_CONTEXT { guc_id }       <-- two dwords
     |       |
     |       +-- h2g_write()                      xe_guc_ct.c:941
     |             copy -> barrier -> advance -> announce
     |
     4. return the fence

  ... GuC picks it up, runs it on an engine ...
  ... engine writes a seqno, raises an interrupt, fence signals ...

  if something breaks: GuC writes G2H -> interrupt -> worker -> switch
```

## Try it

```bash
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_guc_ctb_h2g/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

Run anything graphical and watch the flood of 2-dword `SCHED_CONTEXT` letters.
That's Idea 2, live.

## Checkpoint

1. A submission sends only an action code and an ID number. How does the GuC find
   the batch buffer?
2. What tells the GuC a new letter is waiting?
3. Why is there no "job complete" message in the G2H mailbox?
4. What are the four steps of `h2g_write()`, and which one is the announcement?

<details>
<summary>answers</summary>

1. It doesn't look for it. The batch address was written into the LRC's ring by
   `emit_job()` as a command. The GuC only needs to know *which context* has
   work; the LRC — whose GGTT address it got at registration — contains the ring,
   and the ring contains the command.
2. The driver writing its new tail value into the shared CT descriptor. There's
   no doorbell register and no interrupt in that direction — the GuC polls it.
3. Speed. Completion happens constantly, and routing each one through a
   microcontroller's mailbox would be far too slow. Instead the engine writes a
   sequence number into memory and raises an interrupt, and the interrupt handler
   signals the fence.
4. Copy the letter in at the tail; write barrier; advance the local tail; write
   the tail into the shared descriptor. The fourth one is the announcement — and
   the barrier before it exists so the GuC can never see the announcement before
   the letter.

</details>
