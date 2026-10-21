# Chapter 1 — The Cast

> Who is who in the Xe driver. Read this once; everything later hangs off it.

---

## There are only four hardware things

```
  xe_device        the graphics card you plugged in
    └── xe_tile    a slab of silicon on that card.  Has its own MEMORY.
         └── xe_gt a block of engines on that slab.  Does the WORK.
              └── xe_hw_engine  one engine: render, or copy, or video
```

That's it. Four levels. Each one is a C struct with the same name.

A cheap laptop GPU has **1 tile, 1 GT, a few engines**. A big datacenter
card has **2 tiles, 2 GTs each, lots of engines**. Same four structs
either way.

---

## The one fact that matters

> **Memory belongs to the tile. Work belongs to the GT.**

Everything confusing about Xe becomes clear once you hold those apart.

A **tile** owns:

| | |
|---|---|
| VRAM | the memory chips on that slab |
| GGTT | one address table for that slab |
| a copy engine context | for moving data in and out |

A **GT** owns:

| | |
|---|---|
| engines | the things that actually execute |
| TLBs | its own address caches |
| a GuC | its own firmware scheduler |
| a reset domain | it can be reset without touching the other GT |

So on a 2-tile card there are **two separate pools of VRAM**. Not one big
one. "VRAM0" and "VRAM1" later in the course mean tile 0's memory and tile
1's memory, and they are different places.

### Why you care

If a buffer lives in tile 0's VRAM and you run a job on tile 1's GT, the
data has to **cross between tiles** to get there. Slow. The driver spends a
lot of effort avoiding that, and you'll see why in Chapter 4.

---

## Two GTs per tile?

Yes, on newer parts:

```
  xe_tile
    ├── primary_gt    render + compute
    └── media_gt      video decode + encode
```

They share the tile's memory, but they are **separately resettable** and
each runs its own GuC firmware. When Chapter 15 talks about "resetting a
GT", it means one of these — not the whole card.

---

## What your application creates

Your app never touches tiles or GTs. It creates three things:

| What the app makes | What it is | Belongs to |
|---|---|---|
| **BO** (buffer object) | a chunk of memory | a **tile** (ch 3) |
| **VM** | an address space for the GPU | the device, but with a page table **per tile** (ch 6) |
| **exec queue** | a slot to submit work into | a **GT** (ch 10) |

Look at the right-hand column. That's the memory/execution split again,
showing up in the API. Three objects, and the chapters that cover them.

---

## The code

Only three things to see. First, a device holds its tiles in a plain array:

```c
// xe_device_types.h:379
struct xe_tile tiles[XE_MAX_TILES_PER_DEVICE];
```

Second, a tile points down at its GTs and holds its own memory:

```c
// xe_tile_types.h:47
struct xe_gt *primary_gt;        // render + compute
// xe_tile_types.h:54
struct xe_gt *media_gt;          // video

// xe_tile_types.h:82 — inside the tile's `mem` struct
struct xe_vram_region *vram;     // THIS tile's memory
// xe_tile_types.h:85
struct xe_ggtt *ggtt;            // THIS tile's address table
```

Third, a GT points back **up** at its tile:

```c
// xe_gt_types.h:120
/** @tile: Backpointer to GT's tile */
struct xe_tile *tile;
```

That backpointer is how code reads. When you see this in a function:

```c
struct xe_tile *tile = gt_to_tile(gt);
```

it is asking *"I have a GT; which memory pool do I belong to?"* — walking
from the execution side to the memory side. You will see that line
hundreds of times.

---

## What to carry forward

1. **Four structs, nested:** device → tile → GT → engine.
2. **Memory is per-tile, work is per-GT.** When something in this driver
   looks over-complicated, it is usually because it has to keep those two
   straight.

Next: Chapter 2 — where memory can actually live, and who decides.
