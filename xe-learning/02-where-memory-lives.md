# Chapter 2 — Where Memory Lives

> A buffer has to be *somewhere*. These are the somewheres.

---

## Three places that matter

```
  XE_PL_VRAM0   the memory chips on tile 0.   FAST for the GPU.
  XE_PL_TT      ordinary system RAM, set up so the GPU can reach it.
  XE_PL_SYSTEM  ordinary system RAM, NOT reachable by the GPU.
```

(There's a fourth, `XE_PL_STOLEN` — a small chunk the BIOS reserved. Used
for a few display things. Ignore it.)

```c
// xe_bo.h:82
#define XE_PL_SYSTEM		TTM_PL_SYSTEM
#define XE_PL_TT		TTM_PL_TT
#define XE_PL_VRAM0		TTM_PL_VRAM
#define XE_PL_VRAM1		(XE_PL_VRAM0 + 1)
#define XE_PL_STOLEN		(TTM_NUM_MEM_TYPES - 1)
```

## TT vs SYSTEM — the confusing pair

They are **the same RAM**. The difference is whether the GPU can get to it.

| | pages allocated? | GPU can reach it? |
|---|---|---|
| `XE_PL_SYSTEM` | maybe not | **no** |
| `XE_PL_TT` | yes | **yes** — DMA-mapped |

Moving a buffer from `SYSTEM` to `TT` doesn't copy anything. It allocates
the pages (if needed) and DMA-maps them. Think of `SYSTEM` as *"parked"*
and `TT` as *"plugged in."*

So there are really only two interesting questions about a buffer:

```
  is it in VRAM, or in system RAM?        <- the performance question
  is it plugged in for the GPU?           <- the TT vs SYSTEM question
```

---

## You don't pick a place. You give a wish list.

This surprises people. When you create a buffer you do **not** say "put it
in VRAM." You say which places are *acceptable*, in order:

```
  "VRAM0 if you can.  Otherwise system RAM.  Never stolen."
```

TTM then picks. And it may **change its mind later** — if VRAM fills up,
your buffer gets moved out to system RAM without asking (that's Chapter 4).

The wish list is `bo->placement`, built once at create time:

```c
// xe_bo.c:288
static int __xe_bo_placement_for_flags(struct xe_device *xe, struct xe_bo *bo,
				       u32 bo_flags, enum ttm_bo_type type)
{
	u32 c = 0;

	try_add_vram(xe, bo, bo_flags, type, &c);     // VRAM entries first
	try_add_system(xe, bo, bo_flags, &c);         // then system RAM
	try_add_stolen(xe, bo, bo_flags, &c);         // then stolen

	if (!c)
		return -EINVAL;                       // no place allowed = bug

	bo->placement = (struct ttm_placement) {
		.num_placement = c,
		.placement = bo->placements,
	};
	return 0;
}
```

Three `try_add_*` calls, each one looking at a flag and appending an entry
if it's set. The **order of the calls is the order of preference.**

## First choice vs fallback

Order alone isn't enough, so entries carry a flag. The cleanest example in
the driver is this fixed pair:

```c
// xe_bo.c:66
static const struct ttm_place tt_placement_flags[] = {
	{ .mem_type = XE_PL_TT,     .flags = TTM_PL_FLAG_DESIRED  },   // try me
	{ .mem_type = XE_PL_SYSTEM, .flags = TTM_PL_FLAG_FALLBACK }    // ...or me
};
```

TTM walks the list **twice**: once trying only `DESIRED` entries, then
again allowing `FALLBACK` ones. So a `FALLBACK` entry means *"only if
everything else failed."*

You can see the same idea built dynamically:

```c
// xe_bo.c:182 (try_add_system)
bo->placements[*c] = (struct ttm_place) {
	.mem_type = XE_PL_TT,
	// If VRAM was also allowed, system RAM is only a FALLBACK.
	// If VRAM wasn't allowed, system RAM is the first choice (flags = 0).
	.flags = (bo_flags & XE_BO_FLAG_VRAM_MASK) ? TTM_PL_FLAG_FALLBACK : 0,
};
```

One ternary, and it means: *"am I the plan, or the backup plan?"*

---

## One gotcha worth knowing

The CPU cannot see all of VRAM. It reaches VRAM through a PCI window
(a "BAR"), and on big cards that window is **smaller than the VRAM**.

So if the CPU will need to touch a buffer, it must land in the visible part:

```c
// xe_bo.c:249 (add_vram)
if (io_size < vram->usable_size) {            // window smaller than VRAM
	if (bo_flags & XE_BO_FLAG_NEEDS_CPU_ACCESS) {
		place.fpfn = 0;
		place.lpfn = io_size >> PAGE_SHIFT;   // force it into the window
	} else {
		place.flags |= TTM_PL_FLAG_TOPDOWN;   // keep the window free
	}
}
```

Two branches, one idea: **buffers the CPU needs go at the bottom (inside
the window); everything else is pushed to the top, to leave the window
free.** That's why `XE_BO_FLAG_NEEDS_CPU_ACCESS` exists.

---

## The example

Your app asks for a 4MB texture on a discrete card:

```
flags = XE_BO_FLAG_VRAM0 | XE_BO_FLAG_SYSTEM
```

The wish list comes out as:

```
  [0]  VRAM0,  flags 0            <- first choice, pushed TOPDOWN
  [1]  TT,     flags FALLBACK     <- only if VRAM is full
```

What happens next:

| Situation | Where it lands |
|---|---|
| VRAM has room | VRAM0 |
| VRAM is full | system RAM (`TT`) |
| VRAM frees up later | stays in system RAM until something moves it (ch 4) |

---

## What to carry forward

1. **`TT` and `SYSTEM` are the same RAM** — `TT` just means the GPU can
   reach it. Not a copy, just a mapping.
2. **A buffer's placement is a wish list, not an address.** Order gives
   preference, `FALLBACK` marks the backup, and TTM is free to move the
   buffer later without telling your app.

Next: Chapter 3 — what a buffer object actually *is*, as a C struct.
