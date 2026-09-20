# Chapter 2 — Where Memory Lives

> Beat A: the story.

Chapter 1 gave us the town. Now we open the warehouses.

Before any GPU work happens, one question must be answered for every single
buffer: **where do these bytes physically sit?** Everything else in Act I —
binding, page tables, eviction, faulting — is downstream of that one question.

---

## 1. Four kinds of storage

Xe knows exactly four places a buffer can live. TTM calls them **memory
types**; Xe names them `XE_PL_*`.

### `XE_PL_VRAM0` / `XE_PL_VRAM1` — the on-site warehouses

Real device memory, soldered next to the GPU. Enormous bandwidth — this is
where you want everything the GPU touches in a hot loop.

There is one per tile. VRAM0 is tile 0's warehouse, VRAM1 is tile 1's.
They are genuinely different buildings: tile 1 reaching into VRAM0 is a
cross-tile trip, not a local access.

Only discrete cards have these. On an integrated GPU there is no VRAM at all,
and the whole memory-type list collapses to system memory.

### `XE_PL_TT` — the annex warehouse

Ordinary system RAM (the same DDR your CPU uses), **prepared so the GPU can
reach it**. "Prepared" means two things have happened: the kernel has actually
allocated the pages, and those pages have been DMA-mapped so the device has
bus addresses for them.

Slower than VRAM — every access crosses PCIe — but effectively unlimited, and
it's the only kind of storage an integrated GPU has.

The name is historical: **TT = Translation Table**, from the days when
system pages had to be entered into an aperture table to be reachable.

### `XE_PL_SYSTEM` — the loading yard

This is the one that confuses everybody, so slow down here.

`XE_PL_SYSTEM` is **not** a place where the GPU can find data. It is the
*parked* state: the buffer object exists, it has an identity and a size, but
it has no device-reachable backing right now. The pages may not even be
allocated yet.

Think of it as the paperwork existing while the crate sits in the yard with no
forklift route to it. A buffer sitting in `XE_PL_SYSTEM` must be **moved** to
`XE_PL_TT` or VRAM before the GPU can use it.

This is why newly created buffers start in `XE_PL_SYSTEM`, and why swapped-out
buffers go back to it. It is a state, more than a location.

> **The one distinction to keep:** `XE_PL_SYSTEM` = system RAM the GPU
> *cannot* reach. `XE_PL_TT` = system RAM the GPU *can* reach. Same physical
> DDR, different readiness.

### `XE_PL_STOLEN` — the locked closet

A chunk of memory the firmware carved out before Linux even booted — the
BIOS "stole" it for the graphics device. The display engine's initial
framebuffer lives there, which is why you see a picture before any driver
loads.

It's small, it has awkward access rules, and it mostly matters for display and
a few hardware quirks. Mentally file it and move on.

---

## 2. Placement: a wish list, not an address

Here is the design decision that makes TTM work.

When you create a buffer you do **not** say "put this at VRAM address
0x4000000". You hand over a **wish list**: *"VRAM0 if you can; system memory
if you must."*

Two structures express this:

* A **`ttm_place`** is one option on the list. It names a memory type, and can
  add conditions: *must be physically contiguous*, *must be in the
  CPU-visible window*, *prefer the top of the range*.
* A **`ttm_placement`** is the ordered list of those options.

The list is ordered by preference. TTM walks it top to bottom and takes the
first option it can satisfy. If nothing works, it starts evicting other
buffers to make one work, and tries again.

This indirection is the whole reason a GPU can run more work than fits in
VRAM. Nobody ever holds a hard address; everybody holds a wish list, so the
driver is always free to move things.

---

## 3. The warehouse clerks

A wish list needs somebody to turn it into an actual spot. That is a
**resource manager** — one per memory type, registered with TTM at probe.

A manager has exactly one important job: *given a request and a `ttm_place`,
carve out space and return a receipt.* The receipt is a **`ttm_resource`**:
"you own this much space, starting here, in this memory type."

What's interesting is that Xe's three managers are three **completely
different algorithms**, because the three warehouses have different problems.

### The VRAM clerk — a buddy allocator

VRAM is a fixed, precious, fragmentation-prone resource where physical
contiguity genuinely matters (big pages, display scanout, CPU mapping). So it
gets the heavy machinery: a **buddy allocator**.

A buddy allocator repeatedly halves a large block until it finds the smallest
power-of-two chunk that fits your request, and merges neighbours back together
on free. It can answer requests like *"contiguous, 2MB-aligned, and only from
the first 256MB"* — which the other clerks cannot.

An allocation is a **list of blocks**, not a single extent, unless you
explicitly demanded contiguity.

### The system-memory clerk — an accountant

For `XE_PL_TT` there is nothing to allocate an *address* for. System pages
don't need a coordinate in a warehouse — the GPU will reach them through page
tables, wherever they happen to be.

So this clerk does almost nothing: it checks "would this push us past total
system RAM?", says yes or no, and hands back a receipt with no address on it.
The actual page allocation happens later, by a different mechanism (Chapter 3).

That asymmetry is worth pausing on. **A `ttm_resource` for VRAM says where.
A `ttm_resource` for TT says only how much.**

### The stolen-memory clerk — a simple range allocator

Small fixed region, first-fit allocation out of an interval tree. Nothing
clever, because nothing clever is needed.

---

## 4. The window problem

One complication that shapes a lot of Xe code, and it's a nice physical story.

The CPU reaches device VRAM through a **PCI BAR** — a window in the CPU's
address space mapped onto the card's memory. Historically that window was
small: 256MB, onto a card with 16GB of VRAM. So **most of VRAM is invisible to
the CPU**. The GPU sees all of it; the CPU sees a slice.

(Modern systems with *resizable BAR* enabled can map all of it. Many still
can't.)

So the warehouse has a **service window** at the front. Crates the customer
needs to reach by hand must be stacked near that window. Crates only the
factory touches should be stacked at the back, deliberately, so they don't
waste window space.

That's exactly what the placement flags encode:

* a buffer the CPU must map → *constrain it to the visible range*
* a buffer the CPU will never touch → *allocate top-down*, far from the window

And the VRAM clerk keeps a second counter — not just "how much VRAM is free"
but **"how much *visible* VRAM is free"** — because you can run out of window
space while having plenty of VRAM left.

---

## 5. Who decides the wish list?

Two sources, meeting in the middle:

* **Userspace** says which regions it will accept, via a bitmask it got from
  `DEVICE_QUERY`. (Remember `mem_region_mask` being built during probe in
  Chapter 1? That's the same mask coming back around.)
* **The driver** adds the conditions it knows about: this is a page table, so
  it needs specific alignment; this is pinned, so it must be contiguous; the
  CPU will mmap this, so keep it in the visible window.

Those combine into a single `u32` of `XE_BO_FLAG_*` bits, and one function
turns that `u32` into the `ttm_placement` wish list. That translation is the
first thing that happens when any buffer is born — and it's the first thing
we'll read in Beat B.

---

## The picture

```
        ttm_device (one per xe_device)
             |
   +---------+-----------+-------------+--------------+
   |         |           |             |              |
XE_PL_    XE_PL_TT   XE_PL_VRAM0   XE_PL_VRAM1   XE_PL_STOLEN
SYSTEM    (annex)     (tile 0)      (tile 1)      (closet)
(yard)
   |         |           |             |              |
  TTM's   accountant   buddy         buddy        range alloc
 built-in              allocator     allocator
  stub


  a buffer's wish list:            what it becomes:
  ttm_placement {                  ttm_resource {
     [0] VRAM0, contiguous            mem_type = XE_PL_VRAM0
     [1] TT,    fallback              start    = <block address>
  }                                   size     = ...
                                   }
```

## The two sentences to remember

1. **Nobody owns an address; everybody owns a wish list.** That's what makes
   eviction and migration possible at all.
2. **`XE_PL_SYSTEM` is a parking state, not a place the GPU can read.**

---

## Checkpoint

1. An integrated GPU has no VRAM. What does a typical `ttm_placement` look
   like there?
2. Why does the VRAM manager need a buddy allocator when the system-memory
   manager needs essentially nothing?
3. A buffer will be mmap'd by the CPU *and* wants to live in VRAM. What
   constraint must its `ttm_place` carry, and what might it run out of?

<details>
<summary>answers</summary>

1. A single entry: `XE_PL_TT`. There is no VRAM memory type registered at all.
2. Because a VRAM resource must name a physical location in a fixed-size
   region, with contiguity and alignment constraints. A TT resource names no
   location — the pages live wherever the kernel put them, and the GPU reaches
   them through page tables.
3. It must be restricted to the CPU-visible (BAR) range. It can run out of
   *visible* VRAM even while plenty of total VRAM is free.

</details>
