# Chapter 3 — The Crate

> Beat A: the story.

Chapter 2 told us where memory *can* live, and who hands out space. But we
never actually created anything. There was a receipt system with nothing
filed in it.

This chapter is about the thing itself: the **buffer object**.

---

## 1. A buffer object is not its bytes

Start with the idea that trips people up, because everything else falls out of
it.

When userspace says "give me a 4MB buffer", it does **not** get 4MB of memory.
It gets a **crate**: an object with an identity, a size, a name, and a
lifetime. Whether that crate currently has anything physical inside it — and
where that physical thing sits — is a completely separate question, answered
later, and re-answered every time the buffer is used.

So a buffer in Xe is really **three separable things**:

| Thing | Question it answers | Lives in |
|-------|--------------------|-----------|
| the **object** | who am I, how big, who owns me | GEM |
| the **resource** | where am I right now | TTM (Chapter 2) |
| the **pages** | what physical memory backs me | TTM (`ttm_tt`) |

The object is permanent for the buffer's lifetime. The resource changes every
time the buffer is evicted or migrated. The pages may not exist at all.

Hold that apart in your head and TTM stops being confusing.

---

## 2. The nesting doll

Xe's buffer type is built by stacking three layers, each one physically
embedded inside the next:

```
    struct xe_bo                  <- Xe's crate
        struct ttm_buffer_object  <- TTM's warehouse record card
            struct drm_gem_object <- DRM's shipping label
```

Each layer adds exactly what it owns and nothing else.

### The shipping label — `drm_gem_object`

The bottom layer is pure identity:

* a **size**,
* a **refcount** — the buffer dies when it hits zero,
* a **handle** — the small integer userspace uses to name it in ioctls,
* a **`dma_resv`** — the clipboard, which we'll come to,
* a set of **function pointers** the driver fills in (free, mmap, export…).

Notice what's *not* here: no address, no memory type, nothing about where the
data is. GEM genuinely does not care. It issues the tracking number.

The handle is per-open-file, by the way. Two processes can hold handle `3`
and mean different buffers; the same buffer can be handle `3` in one process
and `7` in another.

### The warehouse record card — `ttm_buffer_object`

The middle layer adds the *placement* story from Chapter 2:

* the **current resource** — the slot receipt, or NULL if unplaced,
* the **page list** — if it's currently backed by system pages,
* the **placement wish list** we built last chapter,
* a **pin count**, an **LRU priority**, and a **type**.

That last one matters more than it looks. TTM buffers come in three types:

* **device** — created by userspace, mmap-able, evictable. The normal case.
* **kernel** — created by the driver for its own use: page tables, ring
  buffers, firmware images.
* **sg** — a buffer whose pages belong to somebody else entirely (an imported
  dma-buf from another device).

A great deal of Xe's buffer code branches on that enum, and once you know it
means *"who created this and who owns the pages"*, the branches read
naturally.

### The crate — `xe_bo`

The top layer is everything specific to this hardware: which VM the buffer
belongs to, its GGTT nodes, its `XE_BO_FLAG_*` bits, cache attributes,
compression state, and the links that put it on Xe's various lists.

The pattern to internalize:

> **`xe_bo` embeds `ttm_buffer_object` embeds `drm_gem_object`.**
> Going down is a field access. Going up is `container_of`.

Xe spells the trip up as `ttm_to_xe_bo()`, and down as `&bo->ttm.base`.

---

## 3. The packing list — where pages actually come from

Chapter 2 ended on a cliffhanger. The system-memory manager allocated *no
pages* — it just checked a budget and returned a receipt with no address. So
where do the actual pages come from?

From a second structure: the **page list**, TTM's `ttm_tt`.

A `ttm_tt` is exactly what its name suggests once you stop being scared of it:
an **array of `struct page *`**, plus the matching array of **DMA addresses**
the device will use to reach them, plus a caching mode.

Two properties make it click:

**It only exists when the buffer is backed by system memory.** A buffer living
in VRAM needs no page list — the VRAM resource *is* the storage, and it's
device memory, not kernel pages. This is precisely what the
`man->use_tt = true` flag meant on the system manager and *not* on the VRAM
manager.

**It can exist while being empty.** TTM distinguishes *allocated* (the array
exists) from *populated* (the array has actual pages in it). A buffer can hold
an unpopulated page list for a long time — that's a buffer that has been
swapped out, or one whose backing was deferred.

So the true picture of a system-resident buffer is:

```
   xe_bo
     ├── ttm_resource   "XE_PL_TT, 4MB"      <- the budget receipt
     └── ttm_tt         pages[1024]          <- the actual memory
                        dma_address[1024]    <- how the device reaches it
```

and of a VRAM-resident one:

```
   xe_bo
     └── ttm_resource   "XE_PL_VRAM0, buddy blocks [...]"   <- IS the storage
```

**Migration is now definable precisely:** moving a buffer from VRAM to system
memory means allocating and populating a page list, copying the bytes into it,
and swapping the resource. That's Chapter 4.

---

## 4. The pallet depot

One more layer, and it exists for a reason worth knowing.

TTM does not get pages straight from the kernel page allocator and hand them
back when done. It keeps a **pool**.

Why? Because of **cache attributes**. A page that the GPU will access
uncached, or write-combined, has had its CPU page-table attributes changed —
and on x86 that is genuinely expensive: it requires tearing down the kernel's
large-page mappings and flushing caches across every CPU.

So freeing such a page and immediately re-allocating it would burn that cost
twice for nothing. Instead TTM keeps freed pages in a pool **bucketed by
caching mode and allocation order**, and hands them straight back out to the
next requester that wants the same combination.

It's a pallet depot: pallets come back, get sorted by type, and go out again
without being rebuilt. The pool also registers with the kernel's shrinker, so
under memory pressure it gives the pages back to the system.

---

## 5. Cached, write-combined, uncached

Since caching decides which pool bucket you land in, it's worth knowing what
the three modes *mean*, because Xe picks between them on every buffer.

* **Cached (WB)** — normal memory. Fast for the CPU, but the GPU's view and
  the CPU's cache can disagree unless the hardware is coherent.
* **Write-combined (WC)** — CPU writes are batched and pushed out; CPU reads
  are *terrible* (uncached). Perfect for buffers the CPU only ever writes and
  the GPU then reads — which is most of what a graphics driver produces.
* **Uncached (UC)** — every access goes to memory. Slow, used for registers
  and a few hardware structures.

The rule of thumb that explains most of Xe's choices: **if the CPU will write
it and the GPU will read it, write-combined.** If the CPU will read it back,
you want cached, and you'd better be on coherent hardware.

There's a neat hardware fact underneath: on **discrete** cards, system memory
accessed by the GPU is always coherent with the CPU, so Xe can just use cached
everywhere and stop thinking. The caching gymnastics are mostly an
**integrated**-GPU problem.

---

## 6. The clipboard

Now the piece that ties memory management to scheduling — and the one thing
from this chapter you will meet in every later chapter.

Hanging on every crate is a **`dma_resv`**: a clipboard with two things on it.

**A lock.** Not an ordinary mutex — a `ww_mutex`, a *wound-wait* mutex, built
for the case where you must lock many objects at once and can't agree on an
order. (A submission touching 200 buffers has to lock all 200. Two threads
doing that in different orders would deadlock instantly. Wound-wait solves it
by making one thread back off and retry. That machinery is `drm_exec`, and it
gets its own chapter.)

**A list of fences, tagged by usage.** Four tags, and the distinction is the
interesting part:

* **KERNEL** — memory-management operations: the clear after allocation, the
  copy during a migration. *Everybody* must wait for these. You cannot read a
  buffer while the driver is still copying it.
* **WRITE** — somebody is writing the buffer. Readers must wait.
* **READ** — somebody is reading it. Writers must wait; other readers needn't.
* **BOOKKEEP** — "I'm using this, but manage synchronization yourself." The
  fence is recorded so the buffer can't be freed or moved out from under the
  user, but no implicit waiting is implied.

That's the whole contract between the two halves of this course:

> **Memory management may not move or free a buffer until the fences on its
> clipboard have signalled. Scheduling adds a fence to the clipboard for every
> job that touches the buffer.**

Neither side needs to know anything else about the other.

---

## 7. How a crate is born

Put it together. When userspace asks for a buffer:

1. **Validate and translate.** The ioctl checks the arguments and turns
   userspace's requested regions and flags into the `XE_BO_FLAG_*` word.
2. **Allocate the crate.** `xe_bo` is allocated and zeroed; the GEM object
   inside it is initialized with the size and the driver's function pointers.
3. **Build the wish list.** Chapter 2's `__xe_bo_placement_for_flags()` runs.
4. **Hand it to TTM.** TTM assigns the object a reservation object, locks it,
   and tries to satisfy the wish list — which runs a resource manager, which
   may evict somebody, and which may need pages from the pool.
5. **Wait for the clear.** New buffers must not leak another process's data,
   so the GPU zeroes them. That's an asynchronous job, and its fence goes on
   the clipboard under KERNEL.
6. **Issue the tracking number.** A handle is created in this file's handle
   table and returned to userspace.

Step 5 is worth dwelling on. The ioctl returns a handle to a buffer that is
*still being zeroed by the GPU*. That's not a bug — it's the design. The fence
on the clipboard guarantees that anything which later touches the buffer will
wait. Userspace gets its handle immediately and the work overlaps.

You'll see that pattern relentlessly in Xe: **return early, record a fence,
let the dependency system sort it out.**

---

## The picture

```
              struct xe_bo
  +--------------------------------------------------+
  |  flags, vm, tile, ggtt_node[], cpu_caching, ...   |   Xe layer
  |                                                   |
  |  struct ttm_buffer_object                         |
  |  +---------------------------------------------+  |
  |  |  type (device/kernel/sg), pin_count         |  |   TTM layer
  |  |  placement  ------> the wish list (ch 2)    |  |
  |  |  resource   ------> WHERE it is now         |  |
  |  |  ttm        ------> ttm_tt: the pages       |  |
  |  |                                             |  |
  |  |  struct drm_gem_object                      |  |
  |  |  +---------------------------------------+  |  |
  |  |  |  size, refcount, funcs                |  |  |   GEM layer
  |  |  |  resv ------> dma_resv: lock + fences |  |  |
  |  |  +---------------------------------------+  |  |
  |  +---------------------------------------------+  |
  +--------------------------------------------------+
```

## Three sentences to remember

1. **An object, a location, and some pages are three different things** —
   `drm_gem_object`, `ttm_resource`, `ttm_tt`.
2. **The page list exists only for system-backed buffers.** VRAM needs none;
   its resource *is* the storage.
3. **The `dma_resv` is the contract**: MM may not move what scheduling has
   fenced.

---

## Checkpoint

1. A buffer is created and immediately migrated VRAM → system memory. Which of
   the three pieces (object / resource / pages) changes, and which survives?
2. Why does TTM pool freed pages instead of returning them to the kernel?
3. `xe_gem_create_ioctl()` returns while the GPU is still zeroing the buffer.
   What stops a later job from reading uninitialized data?

<details>
<summary>answers</summary>

1. The object survives unchanged — same handle, same refcount, same clipboard.
   The resource is replaced (VRAM buddy blocks freed, a TT receipt issued). The
   page list is newly created and populated; before the move there wasn't one.
2. Because pages handed to the GPU often have non-default cache attributes, and
   changing those attributes on x86 is expensive. The pool recycles pages
   bucketed by caching mode and order so the cost isn't paid twice.
3. The clear is an asynchronous GPU job whose fence is added to the buffer's
   `dma_resv` under `DMA_RESV_USAGE_KERNEL`. Every later user of the buffer
   waits on the fences already on the clipboard.

</details>
