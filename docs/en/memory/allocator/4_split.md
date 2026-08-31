# Stage 4 — Splitting

## 1. What problem does the previous allocator have?

Confirmed on real hardware last time: Stage 3 reuses a whole free block **even when it's far bigger than the request**, wasting the difference as pure internal fragmentation. A `malloc(20)` that happens to land a 200-byte free block keeps all 200 bytes locked to that one small object — the other 180 bytes are unusable by anyone until that specific block is freed again, even though they were sitting right there, structurally free, the whole time.

[code link](https://github.com/a-ZINC/xAlloc/tree/split)
```
metadata test passed
freelist test passed
first block: 24, brk count:1, split count: 1
split block: 16, split count1
Successfully freed memSplit
Successfully freed memConsume
====Monotonic sbrk alloc====
   Time:            174845.303864ms
   Throughput:      1526.2234335305134op/sec
   Before RSS:      18768kb
   After RSS:       347984kb
   Peak RSS:        347984kb
   RSS Growth:      321mb
   [EXTRA]:
        Heap growth:          321mb
        freelist Count:       64468
        list avg size:        433byte
        total request:        440mb
        request/heap ratio:   1.3707165109034267
        sbrk count:           113567
        split count:           84048
====Standard malloc====
   Time:            316.527993ms
   Throughput:      843062.875642724op/sec
   Before RSS:      348148kb
   After RSS:       655208kb
   Peak RSS:        655208kb
   RSS Growth:      299mb
```

## 2. What new mechanism fixes it?

**Splitting**: when a free-list match is found that's larger than needed, carve it into two pieces — exactly the requested size (returned to the caller), and a new, smaller free block made from the leftover, immediately pushed back onto the free list for future reuse. Only split when the leftover is big enough to be a valid block on its own (`>= MIN_BLOCK_SIZE`) — otherwise a useless, unusable sliver gets created instead of real savings.Let me fix a bug in the instrumentation before running this — the "Heap size" calculation I wrote doesn't make dimensional sense (mixing KB and pointer arithmetic incorrectly).Whoa — stop and look at this before running it yourself. **Throughput just collapsed: 28,414 ops/sec, down from Stage 3's 5.6M — a roughly 200x regression**, and the runtime went from 47ms to **9.4 full seconds**. This is not noise, and I'm not going to smooth it over. Let's dig into exactly why, because it's real and it's the entire reason splitting can't safely ship alone.

## What the numbers themselves reveal

```
Free list @ end:    32,492 blocks, avg size 26.1 bytes
Splits performed:   125,563
sbrk() calls made:  40,108
```

**The free list grew to 32,492 entries, averaging just 26 bytes each.** First-fit search is `O(n)` in the number of free blocks — and splitting's whole mechanism is *creating more, smaller free blocks over time* (the file's own warning comment, playing out far more violently than intended). Every one of 125,563 splits added a new tiny leftover block to the list. By the end of the run, a huge fraction of `malloc` calls have to **walk tens of thousands of mostly-too-small entries** before finding (or failing to find) a match — and since most of those entries are ~26 bytes, they can't satisfy anything but the smallest requests, so a request for, say, 500 bytes might scan through *most of the list* uselessly before hitting a real candidate or falling through to `sbrk`.

## The other half of the story — it did work, on memory

```
RSS growth: 30,108 KB   (vs. Stage 3's 40,288 KB — a real ~25% further reduction)
Requested/HeapGrown ratio: 1.39x
```

So splitting **delivered exactly the memory win it promised** — meaningfully less real growth than Stage 3, on top of Stage 3's win over Stage 1/2. It just did so by creating a search-cost time bomb that got worse as the run progressed. **This is a completely honest, textbook example of the Fragmentation Primer's Tier 3.2 tension** — except here the tradeoff isn't internal-vs-external fragmentation, it's **memory efficiency vs. search-time complexity**, and it's just as real.

## Why this sets up Stage 5 perfectly, not by coincidence

Coalescing (merging adjacent free blocks back into one) doesn't just fix the "trapped between neighbors" problem you'd expect from its name — it directly **shrinks the free list back down**, since two small free neighbors become *one* larger free entry instead of two small ones. If the throughput collapse we just measured is really caused by free-list length, Stage 5 should partially reverse it as a side effect, *before* Stage 7's size classes solve the search-cost problem properly. That's a concrete, falsifiable prediction — worth testing directly once Stage 5 exists.Run this on your real machine — I want to see whether the 200x throughput collapse reproduces at that scale too, or whether it's partly a sandbox artifact like some of our earlier surprises. Given how severe this is, I'd genuinely expect it to show up on any hardware, but let's verify before treating it as confirmed. Say **"Stage 5"** when you're ready to fix it.
