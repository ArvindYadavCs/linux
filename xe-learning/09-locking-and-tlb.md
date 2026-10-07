# Chapter 9 — Locking, and Making the GPU Forget

> Beat A: the story.

Two things Act I kept deferring. Both are about **correctness**, not features — and
both show up constantly in Act II, which is why they get a chapter now.

The spine:

> *Unbind the 2 MB mapping at `0x140000000`, then free the buffer.*

Three ideas.

---

## Idea 1: Locking many buffers, with no agreed order

Here is the problem in its smallest form.

A submission touches buffers A, B and C. To use them safely you must hold all
three locks at once. Meanwhile another thread touches C, B and A.

```
   thread 1 locks:  A ... then B ... then C
   thread 2 locks:  C ... then B ... then A
```

Thread 1 holds A and wants B. Thread 2 holds C and wants B. One gets B, the other
waits — and now whoever didn't get B is waiting for a lock held by someone waiting
for a lock *it* holds. **Deadlock.**

The classic fix is "always lock in a fixed order" — by address, say. That doesn't
work here, because the set of buffers isn't known in advance: it's whatever
userspace listed, and overlap resolution (Chapter 6) can add more as it goes.

### The real fix: wound-wait

So DRM uses a different kind of lock, a **ww_mutex** (wound-wait mutex). The rule
is simple and surprising:

> **When two threads collide, the younger one gives up everything and starts over.**

"Younger" means it started its locking attempt later — each attempt takes a
ticket with a timestamp.

```
   thread 1 (older, ticket 100)        thread 2 (younger, ticket 200)
   locks A                              locks C
   wants B ... gets it                  wants B ... COLLISION
                                        |
                                        +-- gives up: unlocks C
                                        +-- waits for B to be free
                                        +-- starts the whole thing over
```

Nobody deadlocks, because at any collision **somebody always backs off
completely**. And the older thread always wins, so there's no livelock either:
the oldest attempt in the system always makes progress.

### The cost: your code must be restartable

This is the part that shapes Xe's source. If locking can bail out and restart,
then **everything between "start locking" and "all locks held" must be written as
a loop body that can run several times.**

That's `drm_exec`: a helper that holds the ticket, remembers which objects you've
locked, and gives you a retry loop. It is why so much Xe code looks like this:

```
   loop {
        lock everything I need
        if (collision) restart the loop     <-- unlocks everything first
        ... do the work ...
   }
```

You have already seen this shape three times — Chapter 3's buffer creation,
Chapter 8's bind execute, Chapter 4's eviction sweep. Now you know what it's for.

---

## Idea 2: Clearing a page table entry does not unmap anything

Now the second problem, and it's a hardware fact.

Walking a 4-level page table takes up to 4 dependent memory reads (Chapter 7).
Doing that on every access would be ruinous, so the GPU **caches translations in
a TLB**.

Which means:

```
   you write 0 into the page table entry      <- the entry is gone
   the GPU still has the old translation cached   <- the mapping is NOT gone
```

The GPU happily keeps reading the memory you just freed. No fault. No error. It
just reads — or worse, writes — memory that now belongs to somebody else.

So unbinding is a **three-step** operation, and the order is not negotiable:

```
   1. write the page table entries (clear them)
   2. tell the GPU to forget its cached translations
   3. WAIT for confirmation that it has
   -------- only now is the memory safe to reuse --------
```

Step 3 is the one people forget. "Asking" isn't enough; you need to know it
happened.

### Why it's expensive

Xe doesn't write a register to do this. It sends a **message to the GuC** (Chapter
1's foreman), which performs the invalidation and sends a reply back. That's a
round trip to a microcontroller.

So Xe tries hard to invalidate **as little as possible**:

* **range invalidation** where the hardware supports it — "forget addresses
  `0x140000000` to `0x140200000`" rather than "forget everything"
* **per address space** — only this VM's translations, identified by its ASID
* **full invalidation** only as a fallback, when the range is too large or the
  hardware can't do ranges

---

## Idea 3: The invalidation is a job, with a fence

This is the idea that makes it all fit together.

The invalidation isn't a blocking call buried inside the unbind. It is **its own
job, producing its own fence** — and that fence means exactly one thing:

> *"The GPU has genuinely forgotten. The memory is now safe to reuse."*

That turns an awkward ordering requirement into an ordinary dependency. The
buffer's teardown simply waits on that fence, like anything else waits on a
fence.

And remember Chapter 5's observation, which now pays off: **the page tables
belong to the tile, but the TLBs belong to the GTs.** One tile with two GTs means
**two invalidations**, two fences:

```
   unbind 2MB at 0x140000000
         |
   write the entries (one job per tile)
         |
         +--- invalidate tile 0's primary GT TLB  -> fence
         +--- invalidate tile 0's media GT TLB    -> fence
         |
   combine with the entry-write fence into ONE fence
         |
   buffer teardown waits on it
```

That's why Chapter 8's `ops_execute()` allocated a fence array sized
*per tile, plus per TLB*. Now you know what the extra ones were.

---

## Remember these three

1. **Locking many buffers uses wound-wait: on collision the younger thread
   releases everything and retries.** That's why Xe code is full of restartable
   loops.
2. **Clearing a page table entry does not unmap anything** until the TLB is
   invalidated *and confirmed*.
3. **The invalidation is a job with a fence**, one per GT, and that fence is what
   "safe to reuse" means.

---
---

# Chapter 9 — Beat B: The Code

Five functions, two halves.

| | Function | File | Job |
|-|----------|------|-----|
| **locking** | `drm_exec_lock_obj()` | `drm_exec.c` | lock one object, or bail out |
| | `drm_exec_cleanup()` | `drm_exec.c` | the retry loop's condition |
| **TLB** | `xe_tlb_inval_job_create()` | `xe_pt.c:2742` | one job per GT |
| | `xe_tlb_inval_range()` | `xe_tlb_inval.c:340` | send the request |
| | `xe_tlb_inval_done_handler()` | `xe_tlb_inval.c:372` | the reply signals the fence |

> Code trimmed. `//` comments are mine.

---

## Part 1: Locking

### The context — `include/drm/drm_exec.h`

```c
struct drm_exec {
	u32 flags;
	struct ww_acquire_ctx ticket;       // <<< our age. Older ticket wins.

	unsigned int num_objects;           // how many we've locked so far
	struct drm_gem_object **objects;    // ...and which ones, so we can
					    // unlock them all on a restart

	struct drm_gem_object *contended;   // <<< the object we collided on
	struct drm_gem_object *prelocked;   // ...which we grab FIRST next time
};
```

Five fields, and three of them exist purely to make restarting possible:
`objects` (what to release), `contended` (why we're restarting), `prelocked`
(what to do differently next time).

### Locking one object

```c
int drm_exec_lock_obj(struct drm_exec *exec, struct drm_gem_object *obj)
{
	ret = drm_exec_lock_contended(exec);   // first, re-take the lock we
					       // collided on last time
	if (unlikely(ret))
		return ret;

	if (exec->prelocked == obj) {          // already got it as the
		exec->prelocked = NULL;        // contended one - skip
		return 0;
	}

	ret = dma_resv_lock(obj->resv, &exec->ticket);
	//    ^^^^^^^^^^^^^ Chapter 3's clipboard lock, taken with OUR ticket

	if (unlikely(ret == -EDEADLK)) {
		// COLLISION. We are younger. Do not block, do not try again -
		// just record what we tripped on and report upward.
		drm_gem_object_get(obj);
		exec->contended = obj;
		return -EDEADLK;
	}

	if (unlikely(ret == -EALREADY) &&
	    exec->flags & DRM_EXEC_IGNORE_DUPLICATES)
		return 0;
	//  ^ "I already hold this" is fine - two VMAs can share a buffer,
	//    and Chapter 3's shared VM reservation makes duplicates common

	return drm_exec_obj_locked(exec, obj);   // remember it, for the unwind
}
```

`-EDEADLK` isn't an error here — it's **"back off and start over."**

### The retry loop

```c
#define drm_exec_until_all_locked(exec)					\
	for (bool const __maybe_unused __drm_exec_loop = false;		\
	     drm_exec_cleanup(exec);)		   // <-- loop condition	\
		if (false) {						\
drm_exec_retry: __maybe_unused;			   // <-- goto target		\
			continue;					\
		} else

#define drm_exec_retry_on_contention(exec)			\
	do {							\
		if (unlikely(drm_exec_is_contended(exec)))	\
			goto drm_exec_retry;		   // <-- jump back	\
	} while (__drm_exec_loop)
```

Ugly macro, simple idea: `drm_exec_retry_on_contention()` is a `goto` back to the
top of the loop. And the loop condition does the cleanup:

```c
bool drm_exec_cleanup(struct drm_exec *exec)
{
	if (likely(!exec->contended)) {
		ww_acquire_done(&exec->ticket);
		return false;                    // no collision -> exit the loop
	}

	if (likely(exec->contended == DRM_EXEC_DUMMY)) {
		exec->contended = NULL;
		ww_acquire_init(&exec->ticket, &reservation_ww_class);
		return true;                     // first time in -> take a ticket
	}

	drm_exec_unlock_all(exec);               // <<< RELEASE EVERYTHING
	exec->num_objects = 0;
	return true;                             // ...and go round again
}
```

**`drm_exec_unlock_all()` is Idea 1's "the younger one gives up everything."**
One function, three outcomes: enter, retry, or finish.

### How Xe uses it

From Chapter 8's bind path (`xe_vm.c:3576`):

```c
	err = vm_bind_ioctl_ops_lock_and_prep(&exec, vm, vops);  // lock + validate
	drm_exec_retry_on_contention(&exec);                     // collided? restart
	xe_validation_retry_on_oom(&ctx, &err);                  // OOM? evict, restart
```

And what it locks (`xe_vm.c:3326`):

```c
	err = drm_exec_lock_obj(exec, xe_vm_obj(vm));
	//    ^^^ ONE lock covers every buffer private to this VM, because they
	//        share its reservation object (Chapter 3). A VM with 5000
	//        buffers takes one lock, not 5000.

	list_for_each_entry(op, &vops->list, link)
		err = op_lock_and_prep(exec, vm, vops, op);
	//    ^^^ plus the external buffers, one at a time
```

That's why Chapter 3's shared-reservation decision mattered: it turns the
ww_mutex dance from "thousands of locks" into "one, plus a handful."

---

## Part 2: Making the GPU forget

### Who to ask — `xe_tlb_inval_types.h`

```c
struct xe_tlb_inval_ops {
	int (*all)(...);                                  // forget everything
	int (*ggtt)(...);                                 // forget GGTT only
	int (*ppgtt)(struct xe_tlb_inval *tlb_inval, u32 seqno,
		     u64 start, u64 end, u32 asid, ...); // forget THIS RANGE
							 // in THIS address space
	bool (*initialized)(...);
	void (*flush)(...);
	long (*timeout_delay)(...);
};

struct xe_tlb_inval {
	void *private;                    // the backend (normally the GuC)
	const struct xe_tlb_inval_ops *ops;

	int seqno;                        // <<< every request gets a number
	int seqno_recv;                   // <<< highest number confirmed done
	struct list_head pending_fences;  // fences waiting for confirmation
};
```

Look at the `ppgtt` signature — `start`, `end`, `asid`. **Idea 2's "invalidate as
little as possible", as a function prototype.**

And `seqno` / `seqno_recv` are the confirmation mechanism: requests are numbered,
replies carry the number, and a fence signals when its number has been confirmed.

### Creating the jobs — `xe_pt.c:2738`

Inside `xe_pt_update_ops_run()` (Chapter 7's phase 2):

```c
	if (pt_update_ops->needs_invalidation) {
		// ONE job for the primary GT's TLB...
		ijob = xe_tlb_inval_job_create(q, &tile->primary_gt->tlb_inval,
					       dep_scheduler, vm,
					       pt_update_ops->start,   // just the
					       pt_update_ops->last,    // range we
								       // touched
					       XE_EXEC_QUEUE_TLB_INVAL_PRIMARY_GT);
		update.ijob = ijob;

		if (tile->media_gt) {
			// ...and ANOTHER for the media GT's TLB.
			mjob = xe_tlb_inval_job_create(q,
						       &tile->media_gt->tlb_inval,
						       dep_scheduler, vm,
						       pt_update_ops->start,
						       pt_update_ops->last,
						       XE_EXEC_QUEUE_TLB_INVAL_MEDIA_GT);
		}
	}
```

**`ijob` and `mjob`. Two jobs, one tile.** This is the tile-has-GTs structure
from Chapter 1 showing up as a concrete cost: one page table, two caches to
flush.

And `needs_invalidation` is set wherever an unbind or a change-in-place happens —
`xe_pt.c:2174`, `:2282`, `:2344`. Pure binds into empty space don't need it:
there was nothing cached to forget.

### Sending the request — `xe_tlb_inval.c:340`

```c
int xe_tlb_inval_range(struct xe_tlb_inval *tlb_inval,
		       struct xe_tlb_inval_fence *fence, u64 start, u64 end,
		       u32 asid, struct drm_suballoc *prl_sa)
{
	return xe_tlb_inval_issue(tlb_inval, fence, tlb_inval->ops->ppgtt,
				  start, end, asid, prl_sa);
	//                      ^^^^^^^^^^^^^^^^^^ the backend's range op
}
```

Which for the GuC backend becomes a message — `xe_guc_tlb_inval.c:157`:

```c
	action[len++] = XE_GUC_ACTION_TLB_INVALIDATION;
	action[len++] = seqno;                            // so we can match the reply

	if (!gt_to_xe(gt)->info.has_range_tlb_inval ||
	    length > MAX_RANGE_TLB_INVALIDATION_LENGTH) {
		action[len++] = MAKE_INVAL_OP(XE_GUC_TLB_INVAL_FULL);
		//              ^^^ FALLBACK: hardware can't do ranges, or the
		//                  range is too big. Forget everything.
	} else {
		action[len++] = MAKE_INVAL_OP_FLUSH(type, need_flush);
		action[len++] = id;                       // the ASID
		action[len++] = lower_32_bits(start);     // just this range
		...
	}
```

Idea 2's three strategies, as an `if`/`else`: **range + ASID** when possible,
**full** when not.

### The confirmation — `xe_tlb_inval.c:372`

The GuC replies, and that reply is what signals the fence:

```c
void xe_tlb_inval_done_handler(struct xe_tlb_inval *tlb_inval, int seqno)
{
	spin_lock_irqsave(&tlb_inval->pending_lock, flags);

	// ... advance seqno_recv, then signal every pending fence whose
	//     seqno is now <= seqno_recv ...
}
```

**That signal is Idea 3.** Until the GuC answers, the fence is unsignalled, and
anything depending on it waits. The memory stays reserved.

There's also a timeout worker (`xe_tlb_inval_fence_timeout()`, `:69`) — if the GuC
never replies, that's a device-level failure, not something to wait on forever.

### The synchronous version, for comparison — `xe_tlb_inval.c:355`

```c
void xe_tlb_inval_vm(struct xe_tlb_inval *tlb_inval, struct xe_vm *vm)
{
	struct xe_tlb_inval_fence fence;
	u64 range = 1ull << vm->xe->info.va_bits;     // the ENTIRE address space

	xe_tlb_inval_fence_init(tlb_inval, &fence, true);
	xe_tlb_inval_range(tlb_inval, &fence, 0, range, vm->usm.asid, NULL);
	xe_tlb_inval_fence_wait(&fence);              // <<< BLOCK until confirmed
}
```

Same three steps as the asynchronous path, just with the wait inlined. Used when
a whole VM is being torn down and there's nothing left to pipeline against.

### And the "my mapping just became invalid" path — `xe_vm.c:4401`

Chapter 4's eviction called `xe_vm_invalidate_vma()`. Its last act:

```c
	ret = xe_tlb_inval_range_tilemask_submit(xe, xe_vma_vm(vma)->usm.asid,
						 xe_vma_start(vma), xe_vma_end(vma),
						 tile_mask, batch);
	//                                       ^^^^^^^^^ every tile this VMA
	//                                                 was bound on

	/* WRITE_ONCE pairs with READ_ONCE in xe_vm_has_valid_gpu_mapping() */
	WRITE_ONCE(vma->tile_invalidated, vma->tile_mask);
	//         ^^^^^^^^^^^^^^^^^^^^^ Chapter 6's third tile byte, finally used
```

`tile_mask`, `tile_present`, `tile_invalidated` — the three bytes on `xe_vma`
now all have jobs: *should be bound*, *is bound*, *has been invalidated*.

---

## Our example, end to end

```
   unbind 2MB at 0x140000000, then free the buffer
         |
   LOCK   drm_exec_until_all_locked {
              drm_exec_lock_obj(VM)        <- one lock, all private BOs
              lock the external BO
              collision? -> unlock all, retry
          }
         |
   WRITE  xe_pt_update_ops_run()           <- clear the entries (ch 7)
         |
   FORGET xe_tlb_inval_job_create(primary_gt)  -> ijob -> fence
          xe_tlb_inval_job_create(media_gt)    -> mjob -> fence
              each sends XE_GUC_ACTION_TLB_INVALIDATION with
              { seqno, asid, 0x140000000, 0x140200000 }
         |
   CONFIRM GuC replies -> xe_tlb_inval_done_handler(seqno)
                       -> fences signal
         |
   FREE   xe_vma_destroy(vma, fence)       <- deferred behind the fence (ch 8)
          buffer's memory is now genuinely reusable
```

## Try it on your machine

```bash
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_tlb_inval_fence_work_func/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_invalidate/enable
cat /sys/kernel/debug/tracing/trace_pipe

# does your hardware support range invalidation at all?
grep -rn has_range_tlb_inval drivers/gpu/drm/xe/xe_pci.c
```

## Checkpoint

1. Why can't Xe just lock buffers in address order and avoid ww_mutex entirely?
2. `drm_exec_lock_obj()` returns `-EDEADLK`. What does the caller do, and what
   does `drm_exec_cleanup()` do about the locks already held?
3. Why does one unbind on a single tile create **two** TLB invalidation jobs?
4. A pure bind into previously unmapped address space doesn't set
   `needs_invalidation`. Why is that safe?
5. What exactly does the TLB invalidation fence *mean* when it signals?

<details>
<summary>answers</summary>

1. Because the set of buffers isn't known up front. It's whatever userspace
   listed, and Chapter 6's overlap resolution can add more mappings — and
   therefore more buffers — while the plan is being built. A fixed order needs a
   fixed set.
2. The caller jumps back to the top of the `drm_exec_until_all_locked()` loop via
   `drm_exec_retry_on_contention()`. The loop condition is
   `drm_exec_cleanup()`, which calls `drm_exec_unlock_all()` — releasing every
   lock taken so far — then returns true so the body runs again. The contended
   object is remembered and acquired first on the next pass.
3. The page tables belong to the tile, but each GT in that tile has its own TLB
   caching translations from them. One tile with a primary GT and a media GT
   therefore needs two invalidations — `ijob` and `mjob` in
   `xe_pt_update_ops_run()`.
4. Because there was nothing in the TLB to forget. A TLB only caches
   translations the GPU has actually performed, and it never performed one for an
   address that was unmapped. (Unless the VM uses a scratch page, in which case
   there *was* a valid translation — which is exactly why `xe_pt.c:2344` sets
   `needs_invalidation |= xe_vm_has_scratch(vm)`.)
5. "The GuC has confirmed that the GPU no longer holds any cached translation for
   this range in this address space." That is precisely the point at which the
   underlying memory becomes safe to free or hand to another process — nothing
   earlier is.

</details>
