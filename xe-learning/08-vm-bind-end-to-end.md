# Chapter 8 — VM_BIND End to End

> Beat A: the story.

This chapter introduces almost nothing new. That's the point.

Chapters 2–7 built the pieces: placement, buffers, migration, addresses,
mappings, page tables. This chapter is the **single function that uses all of
them**, in order. If you can follow it, Act I is done.

We'll use one request as the spine:

> Userspace calls `VM_BIND`: *"map buffer B, 2 MB, at GPU address
> 0x140000000, and signal this fence when done."*

That is exactly the mapping Chapter 7 just built page tables for. Now we see
who called it.

Three ideas.

---

## Idea 1: `VM_BIND` is a batch, and it does not wait

Two surprises in the uAPI, both deliberate.

**It's a batch.** One `VM_BIND` call carries a *list* of operations — map this,
unmap that, map a third thing. Userspace can rearrange a whole address space in
one syscall. The list either all succeeds or all fails.

**It returns immediately.** The ioctl does not wait for the page tables to be
written. It submits the work and returns, handing back a **fence** that signals
when the binding is actually live.

So a typical sequence from userspace is:

```
   VM_BIND(map B at 0x140000000)   -> returns at once, gives fence F
   EXEC(work that reads 0x140000000, wait for F first)
```

Userspace never blocks. The dependency is expressed as a fence and the GPU sorts
out the ordering. This is Chapter 3's *"return early, record a fence"* pattern
one more time, now at the top of the stack.

And it goes both ways: `VM_BIND` also accepts **input** fences — "don't start
this bind until X has signalled." So binds can be chained behind other work
without the CPU being involved at all.

---

## Idea 2: A bind is a GPU job, on its own queue

This is the handshake between the two halves of the course.

Chapter 7 said the GPU writes its own page table entries. That means a bind is
**a submission** — the same kind of thing Act II is about. It needs somewhere to
be submitted *to*.

So **every VM gets its own private bind queue, per tile, created when the VM is
created.** Not shared with the client's rendering work. A dedicated channel whose
only job is executing page table updates for this address space.

Why a dedicated queue?

* **Ordering.** Binds for one VM must happen in order relative to each other. One
  queue gives that for free.
* **Isolation.** A hung render queue must not block address-space updates.
* **It's the natural unit.** Chapter 6's lesson — *make the VM the unit, not the
  buffer* — applied once more.

Userspace *can* supply its own bind queue instead (to control ordering between
binds and its own work), but the default is the VM's private one.

Keep this picture:

```
   one VM
     |
     +-- bind queue, tile 0   ---> writes tile 0's page tables
     +-- bind queue, tile 1   ---> writes tile 1's page tables
```

Two tiles means **two separate jobs** for one bind — and therefore two fences,
combined into one before being handed back.

---

## Idea 3: It's one transaction, in five stages

The whole ioctl is arranged so that **nothing is visible until the end**, and
failure at any stage leaves the VM exactly as it was.

Five stages:

| | Stage | What it does | Built in |
|-|-------|-------------|----------|
| 1 | **Check** | validate arguments, look up the VM, the buffers, the queue | — |
| 2 | **Plan** | work out the map/unmap/remap operations, allocate the VMAs | ch 6 |
| 3 | **Lock & validate** | lock every buffer involved, make sure each is resident | ch 3, 4 |
| 4 | **Run** | stage page table entries, submit the GPU job, get a fence | ch 7 |
| 5 | **Finish** | commit, destroy replaced mappings, signal the output fences | ch 6, 7 |

Each stage can fail. Each has an undo. Stage 3 can even fail *recoverably* —
lock contention or out-of-memory — in which case the whole thing simply retries
(Chapter 3's `xe_validation_guard`).

Notice the ordering is forced, not arbitrary:

* you can't **lock** buffers until you know which ones the plan touches → 2 before 3
* you can't write a page table **entry** until you know the buffer's physical
  address → 3 before 4
* you can't **commit** until the GPU has actually written the entries → 4 before 5

That chain is Act I in one sentence.

---

## The picture

```
   userspace: VM_BIND { map B at 0x140000000, 2MB; signal fence F }
         |
    (1) CHECK      args valid? which VM? which buffer? which queue?
         |
    (2) PLAN       -> [ MAP op ]        and allocate an xe_vma     (ch 6)
         |
    (3) LOCK       lock the VM + buffer B
        VALIDATE   is B resident? if not, migrate it in            (ch 4)
         |
    (4) RUN        stage page table entries                        (ch 7)
                   submit MI_STORE_DATA_IMM job to the bind queue
                   -> fence
         |
    (5) FINISH     commit the staged tree, insert the VMA,
                   signal fence F
         |
    return 0  (the GPU may still be writing the entries)
```

## Remember these three

1. **`VM_BIND` is a batch and it's asynchronous** — it returns a fence, not a
   finished binding.
2. **A bind is a GPU job on the VM's own bind queue**, one per tile.
3. **Five stages, one transaction**, and their order is forced by the
   dependencies between Chapters 4, 6 and 7.

---
---

# Chapter 8 — Beat B: The Code

The function is `xe_vm_bind_ioctl()` at **`xe_vm.c:3882`**. Here are the only
functions you need to follow, in the order they're called:

| Stage | Function | Line |
|-------|----------|------|
| 1 | `vm_bind_ioctl_check_args()` | `xe_vm.c:3620` |
| 2 | `vm_bind_ioctl_ops_create()` + `..._parse()` | `:2412`, `:2779` |
| 3 | `vm_bind_ioctl_ops_lock_and_prep()` | `:3320` |
| 4 | `ops_execute()` | `:3406` |
| 5 | `vm_bind_ioctl_ops_fini()` | `:3536` |

Stages 2–5 all live inside `vm_bind_ioctl_ops_execute()` at `:3564`.

> Code is trimmed: error gotos, debug injection and SVM paths removed.
> `//` comments are mine.

---

## The request, as userspace sends it

**File: `include/uapi/drm/xe_drm.h`**

```c
struct drm_xe_vm_bind {
	__u32 vm_id;              // which VM (an index into xe_file's xarray, ch 1)
	__u32 exec_queue_id;      // 0 = "use the VM's own bind queue"  (Idea 2)
	__u32 num_binds;          // <<< IDEA 1: this is a BATCH

	union {
		struct drm_xe_vm_bind_op bind;   // num_binds == 1: inline
		__u64 vector_of_binds;           // num_binds > 1: a user pointer
	};

	__u32 num_syncs;          // how many fences
	__u64 syncs;              // the fence array (in AND out)
};
```

That union is a nice touch: the common case (one operation) needs no second
copy from userspace.

Our call fills in: `num_binds = 1`, one op with
`op = DRM_XE_VM_BIND_OP_MAP, addr = 0x140000000, range = 0x200000, obj = B`,
and one sync of type syncobj to signal.

---

## Stage 1 — Check and look up

**`xe_vm.c:3882`**, the top of the ioctl:

```c
	vm = xe_vm_lookup(xef, args->vm_id);
	//    ^^^ Chapter 1's xe_file->vm.xa xarray. A VM id is just an index.
	if (XE_IOCTL_DBG(xe, !vm))
		return -EINVAL;

	err = vm_bind_ioctl_check_args(xe, vm, args, &bind_ops);
	//    ^^^ validates every op: alignment, ranges inside vm->size,
	//        no unknown flags, no nonsense combinations.
	//        Also copies the array in from userspace if num_binds > 1.

	if (args->exec_queue_id) {
		q = xe_exec_queue_lookup(xef, args->exec_queue_id);

		if (XE_IOCTL_DBG(xe, !(q->flags & EXEC_QUEUE_FLAG_VM))) {
			err = -EINVAL;      // <<< you may only bind with a queue
					    //     created FOR binding
		}
	}
	// q stays NULL if userspace didn't supply one -> we'll use vm->q[] later
```

Then (further down) the buffers get looked up and referenced, and the sync
objects are parsed. `bos[i]` holds one buffer per operation.

Nothing has changed yet. Everything so far is lookups and validation.

### Where the bind queue came from

A quick detour back to **`xe_vm.c:1793`**, inside `xe_vm_create()`:

```c
	/* Kernel migration VM shouldn't have a circular loop.. */
	if (!(flags & XE_VM_FLAG_MIGRATION)) {
		for_each_tile(tile, xe, id) {
			u32 create_flags = EXEC_QUEUE_FLAG_VM;

			if (!vm->pt_root[id])
				continue;

			q = xe_exec_queue_create_bind(xe, tile, vm, create_flags, 0);
			vm->q[id] = q;        // <<< IDEA 2: one bind queue PER TILE,
					      //     created with the VM
		}
	}
```

That's Idea 2, four lines. And note the comment: the *migration* VM — the one
whose page tables `xe_migrate` uses to reach other page tables (Chapter 7's
bootstrap) — deliberately gets **no** bind queue, or you'd have a loop.

---

## Stage 2 — Plan

**`xe_vm.c:4029`**, the main loop. This is Chapter 6, called once per operation:

```c
	xe_vma_ops_init(&vops, vm, q, syncs, num_syncs);
	//              ^^^^^ the container that carries everything through
	//                    stages 2-5: the op list, the queue, the fences

	if (args->num_binds > 1)
		vops.flags |= XE_VMA_OPS_ARRAY_OF_BINDS;

	for (i = 0; i < args->num_binds; ++i) {
		// CHAPTER 6: work out what this request does to existing mappings.
		// Returns a list of MAP / UNMAP / REMAP operations. Changes nothing.
		ops[i] = vm_bind_ioctl_ops_create(vm, &vops, bos[i], obj_offset,
						  addr, range, op, flags,
						  prefetch_region, pat_index);

		// CHAPTER 6: turn that plan into real xe_vma objects, and count
		// how many page-table update operations stage 4 will need.
		err = vm_bind_ioctl_ops_parse(vm, ops[i], &vops);
		if (err)
			goto unwind_ops;
	}

	err = xe_vma_ops_alloc(&vops, args->num_binds > 1);
	//    ^^^ allocate the per-tile page-table update arrays, now that we
	//        know how many entries we need

	fence = vm_bind_ioctl_ops_execute(vm, &vops);     // stages 3-5
```

For our single MAP, `vops.list` ends up holding exactly one operation, carrying
one freshly allocated `xe_vma` for `[0x140000000, +2MB) -> B + 0`.

**Still nothing committed.** The VMA exists but is not in the VM's interval tree.

---

## Stage 3 — Lock and validate

**`xe_vm.c:3564`**, `vm_bind_ioctl_ops_execute()`. The outer wrapper first:

```c
	xe_validation_guard(&ctx, &vm->xe->val, &exec,
			    ((struct xe_val_flags) {
				    .interruptible = true,
				    .exec_ignore_duplicates = true,
			    }), err) {
		// ^^^ Chapter 3's retry loop. Everything in these braces may run
		//     SEVERAL TIMES - on lock contention, or after an OOM that
		//     exhaustive eviction might fix.

		err = vm_bind_ioctl_ops_lock_and_prep(&exec, vm, vops);  // STAGE 3
		drm_exec_retry_on_contention(&exec);    // locks lost -> start over
		xe_validation_retry_on_oom(&ctx, &err); // no memory -> evict harder
		if (err)
			return ERR_PTR(err);

		xe_vm_set_validation_exec(vm, &exec);
		fence = ops_execute(vm, vops);                           // STAGE 4
		xe_vm_set_validation_exec(vm, NULL);

		vm_bind_ioctl_ops_fini(vm, vops, fence);                 // STAGE 5
	}
```

Stages 3, 4 and 5 are those three lines. Everything else is the retry machinery.

And stage 3 itself — **`:3320`**:

```c
static int vm_bind_ioctl_ops_lock_and_prep(struct drm_exec *exec,
					   struct xe_vm *vm,
					   struct xe_vma_ops *vops)
{
	err = drm_exec_lock_obj(exec, xe_vm_obj(vm));
	//    ^^^ lock the VM itself. Because of Chapter 3's SHARED reservation
	//        object, this one lock covers every buffer private to this VM.

	list_for_each_entry(op, &vops->list, link) {
		err = op_lock_and_prep(exec, vm, vops, op);
		//    ^^^ and lock + validate any EXTERNAL buffers, per operation
		if (err)
			return err;
	}
	return 0;
}
```

For a MAP, `op_lock_and_prep()` does one thing that matters:

```c
	case DRM_GPUVA_OP_MAP:
		err = vma_lock_and_validate(exec, op->map.vma, { ... });
		//                                ^^^^^^^^^^^^
		//  CHAPTER 4: xe_bo_validate(). Make sure buffer B is resident
		//  in a place its wish list allows - migrating it, and evicting
		//  somebody else, if necessary.
		//
		//  THIS is why stage 3 must come before stage 4: a page table
		//  entry needs a physical address, and until we validate, the
		//  buffer may not have one.
```

One subtlety in that function worth seeing:

```c
	// We only allow evicting a BO within the VM if it is not part of an
	// array of binds, as an array of binds can evict another BO within the
	// bind.
	res_evict = !(vops->flags & XE_VMA_OPS_ARRAY_OF_BINDS);
```

A batch of binds could otherwise evict a buffer that a *later* operation in the
same batch needs. Same family of problem as Chapter 4's self-eviction veto, at a
different scale.

---

## Stage 4 — Run

**`xe_vm.c:3406`**, `ops_execute()`. This is Chapter 7, driven per tile.

```c
	number_tiles = vm_ops_setup_tile_args(vm, vops);   // pick a queue per tile
	if (number_tiles == 0)
		return ERR_PTR(-ENODATA);                  // nothing to do

	// We will produce one fence per tile, plus TLB-invalidation fences,
	// so allocate an array to collect them.
	fences = kmalloc_objs(*fences, n_fence);
	cf = dma_fence_array_alloc(n_fence);

	// ---- CHAPTER 7, PHASE 1: stage the page table entries ----
	for_each_tile(tile, vm->xe, id) {
		if (!vops->pt_update_ops[id].num_ops)
			continue;
		err = xe_pt_update_ops_prepare(tile, vops);
	}

	// ---- CHAPTER 7, PHASE 2: submit the GPU job, per tile ----
	for_each_tile(tile, vm->xe, id) {
		fence = xe_pt_update_ops_run(tile, vops);   // -> MI_STORE_DATA_IMM
		...
collect_fences:
		fences[current_fence++] = fence ?: dma_fence_get_stub();
		// plus a TLB invalidation fence per GT (Chapter 9)
	}

	// Combine every tile's fence into ONE fence to hand back.
	dma_fence_array_init(cf, n_fence, fences, ...);
	return cf ? &cf->base : fence;
```

Two tiles → two jobs → two fences → **one `dma_fence_array`**. That's Idea 2's
"two separate jobs for one bind" made concrete.

And the queue selection — **`:3380`**:

```c
static int vm_ops_setup_tile_args(struct xe_vm *vm, struct xe_vma_ops *vops)
{
	struct xe_exec_queue *q = vops->q;        // what userspace supplied (or NULL)

	for_each_tile(tile, vm->xe, id) {
		if (vops->pt_update_ops[id].num_ops)
			++number_tiles;

		if (q) {
			vops->pt_update_ops[id].q = q;          // userspace's queue
			if (vm->pt_root[id] && !list_empty(&q->multi_gt_list))
				q = list_next_entry(q, multi_gt_list);
				// ^ a multi-tile user bind queue is a LIST of
				//   queues; take the next one for the next tile
		} else {
			vops->pt_update_ops[id].q = vm->q[id];  // <<< the VM's own
							        //     bind queue
		}
	}
	return number_tiles;
}
```

The `else` branch is Idea 2's default, in one line.

---

## Stage 5 — Finish

**`xe_vm.c:3536`**, `vm_bind_ioctl_ops_fini()`:

```c
	ufence = find_ufence_get(vops->syncs, vops->num_syncs);

	list_for_each_entry(op, &vops->list, link) {
		if (ufence)
			op_add_ufence(vm, op, ufence);

		// Destroy the mappings this bind replaced - but PASS THE FENCE.
		// The old VMA is only actually freed once the GPU has finished
		// writing the new page table entries.
		if (op->base.op == DRM_GPUVA_OP_UNMAP)
			xe_vma_destroy(gpuva_to_vma(op->base.unmap.va), fence);
		else if (op->base.op == DRM_GPUVA_OP_REMAP)
			xe_vma_destroy(gpuva_to_vma(op->base.remap.unmap->va),
				       fence);
	}

	if (fence) {
		for (i = 0; i < vops->num_syncs; i++)
			xe_sync_entry_signal(vops->syncs + i, fence);
		//  ^^^ IDEA 1. Hook userspace's output fences up to the bind's
		//      fence. When the GPU finishes the entry writes, F signals.
	}
```

`xe_vma_destroy(vma, fence)` is the careful bit: an unbind's old mapping cannot
be freed immediately, because the GPU job that clears its page table entries
hasn't run yet. The fence defers it.

(The actual interval-tree insert/remove — Chapter 6's `xe_vm_insert_vma()` —
happens just before this, in the commit phase of `ops_execute`.)

Then back in the ioctl:

```c
	fence = vm_bind_ioctl_ops_execute(vm, &vops);
	if (IS_ERR(fence))
		err = PTR_ERR(fence);
	else
		dma_fence_put(fence);      // we don't keep it - the sync objects do

	return err;                        // <<< returns NOW. The GPU may still
					   //     be writing page table entries.
```

**Idea 1's whole point, in the last line of the function.**

---

## The undo path

If any stage fails, `xe_vm.c:4084`:

```c
unwind_ops:
	if (err && err != -ENODATA)
		vm_bind_ioctl_ops_unwind(vm, ops, args->num_binds);
	//  ^^^ walks every operation BACKWARDS and reverses it: free the VMAs
	//      stage 2 allocated, restore any mapping it was going to replace.

	xe_vma_ops_fini(&vops);
	for (i = args->num_binds - 1; i >= 0; --i)
		if (ops[i])
			drm_gpuva_ops_free(&vm->gpuvm, ops[i]);
free_syncs:
	if (err == -ENODATA)
		err = vm_bind_ioctl_signal_fences(vm, q, syncs, num_syncs);
	//  ^^^ "nothing to do" still has to signal the output fences, or
	//      userspace waits forever on a fence nobody will ever signal
	while (num_syncs--)
		xe_sync_entry_cleanup(&syncs[num_syncs]);
```

That `-ENODATA` case is a good detail: a bind that turns out to be a no-op must
*still* signal its fences. Silence would hang the application.

---

## The whole journey

```
 xe_vm_bind_ioctl()                        xe_vm.c:3882
   |
 (1) xe_vm_lookup()                        the VM, from xe_file's xarray  [ch 1]
     vm_bind_ioctl_check_args()            :3620
     xe_exec_queue_lookup()                or fall back to vm->q[]
   |
 (2) for each bind:
       vm_bind_ioctl_ops_create()          :2412   plan the ops          [ch 6]
       vm_bind_ioctl_ops_parse()           :2779   allocate the VMAs     [ch 6]
     xe_vma_ops_alloc()
   |
 vm_bind_ioctl_ops_execute()               :3564   <- retry loop         [ch 3]
   |
 (3)   vm_bind_ioctl_ops_lock_and_prep()   :3320
         drm_exec_lock_obj(VM)                     one lock, all BOs     [ch 3]
         op_lock_and_prep() -> xe_bo_validate()    make B resident       [ch 4]
   |
 (4)   ops_execute()                       :3406
         vm_ops_setup_tile_args()                  pick the bind queue
         xe_pt_update_ops_prepare()                stage entries         [ch 7]
         xe_pt_update_ops_run()                    submit, get a fence   [ch 7]
         dma_fence_array_init()                    combine per-tile fences
   |
 (5)   vm_bind_ioctl_ops_fini()            :3536
         xe_vma_destroy(old, fence)                deferred free
         xe_sync_entry_signal()                    signal userspace's fence
   |
 return 0
```

Count the bracketed chapter tags. **Every single one has already been covered.**

## Try it on your machine

```bash
# the ops a single VM_BIND expands into
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vm_ops_execute/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_bind/enable
echo 1 > /sys/kernel/debug/tracing/events/xe/xe_vma_unbind/enable
cat /sys/kernel/debug/tracing/trace_pipe

# and the bind queues a VM owns
ls /sys/kernel/debug/dri/0/
```

## Checkpoint

1. Why must stage 3 (lock & validate) come before stage 4 (write page tables)?
2. `VM_BIND` returns 0 while the GPU may still be writing entries. How does
   userspace know when the mapping is actually usable?
3. A bind on a two-tile device returns one fence. Where did it come from?
4. Why does `xe_vma_destroy()` take a fence instead of just freeing the mapping?
5. A `VM_BIND` that turns out to have nothing to do returns `-ENODATA`
   internally. Why does the code still signal the output fences?

<details>
<summary>answers</summary>

1. A page table entry contains a *physical* address. Until the buffer is
   validated it may not be resident at all, so there is no address to write.
   Validation is also what migrates it in, which can evict other buffers.
2. Through the output fence. `vm_bind_ioctl_ops_fini()` hooks userspace's sync
   objects up to the bind job's fence, so `EXEC` can simply declare a dependency
   on it and the GPU enforces the ordering.
3. Each tile got its own job on its own bind queue, producing its own fence.
   `ops_execute()` collects them (plus TLB-invalidation fences) into a
   `dma_fence_array`, which behaves as a single fence that signals when all of
   them have.
4. Because the GPU job that *clears* the old mapping's page table entries hasn't
   run yet. Freeing the VMA immediately would destroy state the pending job still
   needs; the fence defers the free until the job completes.
5. Because userspace is waiting on those fences. A bind with no work still has to
   signal them, or the application blocks forever on a fence nobody will ever
   signal.

</details>
