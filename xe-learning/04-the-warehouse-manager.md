# Chapter 4 — The Warehouse Manager at Work

> Beat A: the story.

We have warehouses (Chapter 2) and crates (Chapter 3). Nothing has moved yet.

This chapter is the machinery of movement: how a buffer gets to where it needs
to be, who gets thrown out to make room, who physically carries the bytes, and
who has to be told afterwards.

It is the busiest chapter in Act I. Everything here is TTM asking Xe questions
and Xe answering.

---

## 1. Validate: make reality match the wish

A buffer is about to be used. Somebody — a submission, a CPU mapping, a bind —
needs it to actually be *somewhere usable*. So they **validate** it.

Validation is one question asked well:

> *Is this buffer's current location compatible with its wish list? If not,
> make it so.*

Three outcomes:

* **Already fine.** The buffer's resource satisfies one of the placement
  entries. Nothing happens. This is the overwhelmingly common case, and it's
  cheap — a comparison, no allocation.
* **Needs moving.** Ask a resource manager for space in an acceptable memory
  type, then move the bytes there.
* **No space anywhere.** Start evicting other buffers, then try again.

The crucial thing about validate is that it is **idempotent and repeated**.
Every submission validates every buffer it touches, every time. The fast path
being a single compatibility check is what makes that affordable.

---

## 2. The two-pass trick

Here's a subtlety that explains a flag we met in Chapter 2 and never fully
used.

Suppose a buffer's wish list is *"VRAM0 preferred, system memory as
fallback"*, and VRAM is full.

The naive approach walks the list, fails on VRAM, succeeds on system memory,
and you end up in system memory forever. VRAM would never be used once full —
which is to say, almost immediately, and then permanently.

So TTM walks the list **twice**:

* **First pass:** consider only *desired* entries — skip anything marked
  `FALLBACK`. If space isn't free, **evict somebody** to make it free.
* **Second pass:** now consider the fallback entries. Take the consolation
  prize.

That's what `TTM_PL_FLAG_FALLBACK` was for. It doesn't mean "lower priority";
it means **"don't even look at me until you have genuinely tried to evict for
the good option."**

The story version: the clerk doesn't say "warehouse full, use the annex." The
clerk says "warehouse full — who in there hasn't been touched lately? Move
*them* to the annex, and put the new crate in the good spot." Only if there is
genuinely nobody worth moving does the new crate go to the annex.

---

## 3. Eviction: who gets thrown out

To evict, TTM needs a victim. It walks the memory type's **LRU list** —
least-recently-used first — and for each candidate asks two questions:

**Can I lock it?** Eviction needs the victim's `dma_resv`. If another thread
holds it, skip (or wound it, if we're in a ww-mutex transaction).

**Is evicting it valuable?** This is a driver callback, and it's a veto. TTM
asks: *"would moving this buffer out actually help me satisfy the placement I
want?"* A buffer sitting outside the address range we need doesn't help, so
evicting it would be pointless work.

Xe adds a second, subtler veto — and it's worth understanding because it
prevents a genuine disaster:

> **Never evict a buffer belonging to a VM that is currently validating.**

Picture it. A submission is validating 500 buffers belonging to VM A. Buffer
#400 doesn't fit, so TTM goes looking for a victim — and picks buffer #7,
which this very submission already validated and still needs. The submission
would evict its own working set, one buffer at a time, forever making no
progress. Xe's veto breaks that loop.

Once a victim is chosen, its own placement is consulted through another driver
callback: *"where should this evicted buffer go?"* Xe's answer is a small
table:

* evicted from **VRAM or stolen** → go to **TT** (GPU-reachable system memory)
* evicted from **TT** → go to **SYSTEM** (the parking yard; this is swapout)

And two special answers:

* a buffer userspace marked **"don't need"** → don't move it, **throw the
  contents away**
* the device has been **unplugged** → same, purge everything; there is no
  hardware left to preserve data for

That last pair is the interesting design point: **a placement list with zero
entries means "discard the backing store entirely."** Eviction and destruction
are the same mechanism with a different wish list.

---

## 4. Who carries the crate

Now the physical move. And the answer is the thing people find surprising:

> **The GPU copies its own memory.**

Not `memcpy`. Not DMA engines in the chipset. Xe builds a small batch buffer of
copy commands, submits it to the **blitter engine** (the copy engine from
Chapter 1), and the GPU does the work.

Why? Because it's dramatically faster — the GPU has the bandwidth to VRAM, and
the CPU reaching into VRAM through the BAR window would be crawling. Also the
CPU may not even be able to *see* the VRAM in question (Chapter 2's window
problem).

This has a huge consequence:

> **A migration returns a fence, not a finished copy.**

The move callback submits the copy and returns immediately. The bytes are still
in flight. TTM attaches the copy's fence to the buffer's `dma_resv` under
`DMA_RESV_USAGE_KERNEL` — the clipboard from Chapter 3 — and lets everyone
carry on. Anybody who later touches the buffer waits on that fence.

That's Chapter 3's *"return early, record a fence"* pattern doing real work.
Eviction under memory pressure doesn't stall the world; it queues copies.

### Clear instead of copy

One optimization worth its own name. Before copying, Xe asks: *does the source
actually contain anything meaningful?*

Sometimes it doesn't — a brand-new buffer, or one whose pages were never
populated. There's nothing to preserve. In that case the GPU doesn't copy, it
just **clears** the destination — which it must do anyway, so another process's
data can't leak through recycled memory.

So the move callback really has three modes: *nothing to do* (just swap the
resource pointer), *clear the destination*, and *copy source to destination*.

---

## 5. You can't get there from here

Now a constraint that falls straight out of Chapter 2, and it's the most
satisfying "aha" in Act I.

Remember: `XE_PL_SYSTEM` is system memory the GPU **cannot reach**.
`XE_PL_TT` is system memory the GPU **can** reach.

And we just established that **the GPU does the copying.**

Therefore: the GPU cannot copy a buffer from VRAM to `XE_PL_SYSTEM`. It cannot
address the destination. Nor from `XE_PL_SYSTEM` to VRAM — it can't address the
source.

So those moves are **impossible in one step**. The driver's answer is to tell
TTM: *"I can't do this directly — bounce through `XE_PL_TT` first."* TTM
performs the move in two hops:

```
   VRAM  --[GPU copies]-->  XE_PL_TT  --[just relabel]-->  XE_PL_SYSTEM
```

The second hop moves no data at all. The pages are the same pages; they simply
stop being DMA-mapped for the device. It's a change of *status*, not of
location — exactly what Chapter 2 said the SYSTEM/TT distinction was.

The mechanism is a special return code meaning "multi-hop", plus a temporary
intermediate placement. When you see it in code, read it as: **"the copy engine
can't see one end of this move."**

---

## 6. Telling everyone the map is stale

The last piece, and the one that reaches out of memory management into the rest
of the driver.

A buffer's bytes just moved. Anybody holding a **GPU address** for it now holds
a lie. Page tables say "virtual address X maps to VRAM page Y"; the data is no
longer at VRAM page Y.

So *before* the move happens, the driver is given a chance to react. This is
the notify callback, and Xe uses it to do three things:

1. **Tear down CPU mappings.** Any kernel vmap of the buffer is now invalid.
2. **Walk every VM that has this buffer mapped** and mark those mappings as
   needing a rebind. A buffer can be mapped into many VMs at many addresses;
   all of them must be told.
3. **Wait, or arrange to wait, for GPU work using those mappings to finish.**
   You cannot pull the rug out from under a job that is mid-execution.

Point 2 is where two worlds meet. Every buffer keeps a list of the mappings
that reference it — the list whose emptiness Chapter 3's destructor asserted.
Migration walks that list and invalidates each one.

And there's a fork in the road here that previews a lot of later material:

* A VM in **normal mode** must be told to re-bind its mappings before its next
  submission. The driver marks them and the work happens later.
* A VM in **fault mode** doesn't need telling. Its mappings are established on
  demand by page faults, so an invalidated mapping simply faults again next
  time it's touched.

Same event, two completely different recovery strategies. Chapters 10 and 18.

---

## 7. The LRU, and why it's moved in bulk

Eviction picks victims by LRU order, so something must maintain that order.
Every resource sits on its memory type's LRU list, and using a buffer moves it
to the tail.

But consider a submission touching 5000 buffers in one VM. Moving 5000 entries
to the tail of a list, one at a time, under a lock, on every submission, is
absurd.

So TTM supports **bulk moves**: a group of resources that are guaranteed to
stay adjacent on the LRU and can be moved to the tail as a unit, in constant
time. Xe gives every VM one of these, and every user buffer created against
that VM joins it.

This is the same trick as the shared reservation object from Chapter 3, applied
to a different data structure: **make the VM the unit of accounting, not the
buffer.** It's why Xe's submission path is cheap.

There is also a small **priority** dimension — a handful of priority levels, so
some buffers are considered for eviction before others regardless of age.

---

## 8. Shrinking and purging

Two more pressure valves, both driven from outside the GPU.

**The shrinker.** Xe registers with the kernel's memory shrinker, so when the
*system* is short on memory — not the GPU — the kernel can ask Xe to give
pages back. Xe responds by swapping buffer contents out to backup storage and
freeing the pages, or by purging buffers whose contents nobody wants.

**Purgeable buffers.** Userspace can mark a buffer *"I don't need the contents
any more, but keep the object around"* — a cache it can regenerate, typically.
Under pressure, Xe simply throws the contents away rather than paying to
preserve them. This is what the `MADVISE` ioctl from Chapter 1's table is for.

The interaction with eviction is neat: a *don't-need* buffer being evicted
doesn't get copied anywhere. It gets purged. The cheapest possible migration is
the one where you discard the cargo.

---

## 9. The whole conversation

Step back and look at the shape of this chapter. TTM is a state machine that
does not know what a GPU is. Everything hardware-specific arrives through a
table of callbacks the driver fills in. Roughly:

| TTM asks | Xe answers |
|----------|-----------|
| create/populate/free a page list | pool allocation, caching, CCS pages (ch 3) |
| where should this evicted buffer go? | VRAM→TT, TT→SYSTEM, don't-need→purge |
| is evicting this one worthwhile? | not if its VM is mid-validation |
| move these bytes | submit a blit; here's a fence |
| the buffer is about to move | invalidate mappings, trigger rebind |
| how does the CPU map this memory? | BAR offsets for VRAM, pages for TT |
| the buffer is being destroyed | unmap from GGTT, drop VM reference |

**That table *is* the driver's memory management.** Everything else in Act I is
either building the inputs to it (Chapters 2–3) or dealing with its
consequences (Chapters 5–11).

---

## The picture

```
  somebody needs a buffer usable
            |
            v
    +-----------------+   compatible?  ---> yes ---> done (fast path)
    |  validate       |
    +-----------------+
            | no
            v
   pass 1: desired placements only, evict if needed
            |
            +--- no luck ---> pass 2: fallback placements accepted
            |
            v
   +---------------------+      +-------------------------------+
   | resource manager    |      | eviction: walk LRU, ask the   |
   | allocates space     |<-----| driver "worth it?", move the  |
   +---------------------+      | victim out (recursively)      |
            |                   +-------------------------------+
            v
   +--------------------------------------------+
   | notify: invalidate mappings, mark rebind   |
   +--------------------------------------------+
            |
            v
   +--------------------------------------------+
   | move: submit a GPU blit, get a fence       |
   |   VRAM <-> SYSTEM ? bounce through TT      |
   +--------------------------------------------+
            |
            v
   fence lands on the dma_resv; everyone else waits on it
```

## Three sentences to remember

1. **Validate means "make reality match the wish list"** — and the common case
   is that it already does.
2. **The GPU copies its own memory**, so migration is asynchronous and returns
   a fence — and VRAM↔SYSTEM needs a TT hop because the copy engine can't
   address `XE_PL_SYSTEM`.
3. **Moving a buffer invalidates every mapping of it**, and that is the hinge
   between memory management and everything else.

---

## Checkpoint

1. A buffer's wish list is "VRAM0 desired, TT fallback". VRAM0 is full but
   contains several idle buffers. What happens, and in what order?
2. Why can't the driver move a buffer directly from VRAM to `XE_PL_SYSTEM`?
3. Why does Xe refuse to evict buffers belonging to a VM that is currently
   validating?
4. Eviction under memory pressure doesn't block the caller until the bytes have
   moved. What makes that safe?

<details>
<summary>answers</summary>

1. Pass one considers only VRAM0. It's full, so TTM walks VRAM0's LRU, picks
   the least recently used evictable buffer, asks the driver where it should go
   (TT), moves it, and retries the allocation — repeating until there's room.
   The fallback TT entry is never reached. The new buffer lands in VRAM0.
2. Because the GPU's copy engine performs the move, and `XE_PL_SYSTEM` pages
   are by definition not DMA-mapped for the device — the GPU cannot address the
   destination. The move goes VRAM→TT (real copy) then TT→SYSTEM (unmap only).
3. Otherwise a submission validating a large working set could evict buffers it
   has already validated and still needs, livelocking against itself.
4. The copy is a GPU job whose fence is attached to the buffer's `dma_resv`
   under `DMA_RESV_USAGE_KERNEL`. Any later access — CPU or GPU — waits on the
   fences already on the clipboard, so nobody can observe the buffer mid-copy.

</details>
