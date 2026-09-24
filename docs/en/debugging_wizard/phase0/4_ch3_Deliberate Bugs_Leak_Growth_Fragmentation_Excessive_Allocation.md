# Debugging Wizard — Phase 0, Chapter 3
# Deliberate Bugs: Leak, Growth, Fragmentation, Excessive Allocation

> **How to use this note** (same as Chapters 1-2)
> 1. Read the short concept, then answer 🧠 THINK / 🔬 PREDICT **on paper first**.
> 2. Only then open the **✅ Solution**.
> 3. Run the experiment and record **your** numbers.
> 4. Ask me follow-up questions any time — they go in the **Clarification Log** at the end.
>
> **About "reference data" boxes.** Measured on a different machine (sandbox VM, kernel 6.18, GCC 13, glibc's default allocator, transparent hugepages `madvise`). They show the *shape* of the result, not your exact numbers.
>
> **All code is in this note.** Every new file is listed in full in **Appendix A**.
>
> **Honesty note.** All four new programs were compiled with the strict warning flags (zero warnings), run plain and under ASan/UBSan, and the numbers quoted below came from real runs. CMake itself was not run in my sandbox — your first `cmake --preset debug` after adding this chapter is its real test. This chapter adds **no new code to `src/`**: it *uses* Chapter 2's `dw_core` to diagnose bugs, rather than extending the library. Unlike Chapters 1-2, every experiment here is **self-contained** (one process, inline snapshots) — none of them need Chapter 1's `observe_pid` or a backgrounded process.

---

## 0. Chapter Overview

### 0.1 What You Will Learn

- **Classify** a memory defect into one of four categories — leak, growth, fragmentation, excessive allocation — by its *signature* in `VmSize`, `RssAnon`, and minor faults, not just by whether a sanitizer flags it.
- **Explain** why an unbounded cache is invisible to LeakSanitizer, and why that makes it more dangerous in production, not less.
- **Observe** heap fragmentation directly: freed memory that stays resident, and a later allocation that still needs fresh address space.
- **Measure** the per-allocation overhead of many small allocations versus one large one, and connect it to Chapter 2's malloc/mmap threshold.
- **Distinguish** fact (a number `dw_core` read from the kernel) from interpretation (which bug you believe it is) — the project's core epistemic rule, now applied to real defects.

### 0.2 Prerequisites

Chapters 1 and 2 complete. `dw_core` builds. No new tools needed.

### 0.3 Why This Matters to Debugging Wizard

Chapter 1 showed that a leak "appears to work." This chapter shows something sharper: **three of these four bug categories produce no sanitizer report at all**, because nothing is technically wrong from ASan/LSan's point of view — the memory is reachable, or it was freed correctly. Debugging Wizard's eventual job (Phase 13, the diagnosis engine) is to recognize these patterns from *symptoms in the numbers*, the same way you are about to.

### 0.4 Step Map

| Step | Focus |
|------|-------|
| 3.1 | Four categories, and why sanitizers can't be the whole story |
| 3.2 | The leak, revisited through `RssAnon` |
| 3.3 | Growth: an unbounded cache (invisible to LSan) |
| 3.4 | Fragmentation: freed memory that doesn't come back |
| 3.5 | Excessive allocation: the cost of many small pieces |
| 3.6 | A bug-signature table |
| 3.7 | Explain-it-back, knowledge check, completion |

### 0.5 Project Conventions (unchanged)

Code lives in the note. Predict before you run. Change one variable at a time. Shell for short glue only; measurement code is C++. Record the machine in `docs/environment.md`.

---

## Step 3.1 — Four Categories, and Why Sanitizers Can't Be the Whole Story

Your master document's memory lab (§8) names several defects. This chapter groups them into four categories with genuinely different signatures:

| Category | What it is | Example |
|----------|-----------|---------|
| **Leak** | Memory whose *pointer is lost* — nothing in the program can reach it anymore | Chapter 1's `leak_bounded`: overwrites the only pointer each iteration |
| **Growth** | Memory that is *never released by design* — still reachable, held on purpose, but with no bound | A cache, log, or registry that never evicts |
| **Fragmentation** | Memory that *was freed*, but in pieces too scattered to satisfy a later request | Alternating allocation sizes leave a checkerboard of holes |
| **Excessive allocation** | Correctly freed, but each allocation carries *fixed overhead*, so many small ones cost more than one big one | Header/bookkeeping bytes per chunk, multiplied by chunk count |

### 🧠 THINK

LeakSanitizer's rule is: at exit, anything **still reachable** through a global, a stack variable, or a live container is *not* a leak, however large. Which of the four categories above will LSan catch, and which will it wave through as "no leaks found"?

<details>
<summary>✅ Solution</summary>

LSan catches only the **leak** row: memory nothing can reach. **Growth** is reachable by definition (held in a live container), so LSan says nothing. **Fragmentation** involves memory that was properly `free()`'d — not leaked at all, just wastefully scattered. **Excessive allocation** is also properly freed — the "waste" is overhead per live allocation, not a lost pointer. Three of four categories produce a clean sanitizer report. This is the chapter's central warning: **"ASan and LSan are clean" is not "this program is memory-healthy."**

</details>

---

## Step 3.2 — The Leak, Revisited Through `RssAnon`

Chapter 1's `leak_bounded` grew `VmSize` by about 1 MB/s and `VmRSS` by about 4 kB/s, watched from a **second terminal** using `observe_pid` on a backgrounded process. This chapter's other three experiments are **self-contained**: one process, printing its own `dw::MemSnapshot` inline, no backgrounding, no second tool. For consistency — and because it makes the leak directly comparable to Step 3.3's growth experiment, which uses the identical technique — `leak_revisited.cpp` reproduces the same defect (allocate 1 MB, touch one byte, never free) instrumented this way.

📄 **Full file:** Appendix A.1 (`leak_revisited.cpp`).

```bash
./build/debug/lab/leak_revisited 20
```

### 🔬 PREDICT

1. Chapter 1 touched one byte per 1 MB block and found `VmRSS` growing about 250x slower than `VmSize`. Will `RssAnon` alone (rather than `VmRSS`, which also includes file-backed memory) show the same ratio here?
2. Compare the *shape* of this table to Step 3.3's `growth_bounded` table before you've run either: will you be able to tell them apart from the numbers alone?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data** (`leak_revisited 20`, kB):
> | Round | VmSize | RssAnon | minor faults (this round) |
> |-------|--------|---------|----------------------------|
> | 0 | 6,368 | 200 | — |
> | 1 | 7,396 | 212 | 5 |
> | 5 | 11,508 | 228 | 1 |
> | 10 | 16,648 | 248 | 1 |
> | 20 | 26,928 | 288 | 1 |

1. **Yes.** `VmSize` grew by exactly 1,028 kB/round (the 1 MiB request plus glibc's header, rounded to a page — same as Chapter 1). `RssAnon` grew by only **4 kB/round**, a ratio of about 257:1, matching Chapter 1's ~250:1 finding almost exactly. One extra fault appears in round 1 (process warm-up); every round after that costs exactly **1** minor fault, for exactly the one page touched.
2. **No — and that's the point.** Run `growth_bounded` and `leak_revisited` side by side and look only at the `RssAnon` column: both climb steadily, round after round, with no visible ceiling. The table alone cannot tell you which is which. Confirm this yourself by comparing the two outputs.

</details>

### 🧠 THINK

If a leak instead held **file-backed** memory (for example, `mmap`-ing the same large file repeatedly without `munmap`), would it show up in `RssAnon` or `RssFile`? Would `RssAnon` growth alone be enough to *conclude* "this is a leak"?

<details>
<summary>✅ Solution</summary>

A file-backed leak grows `RssFile`, not `RssAnon`. And no — `RssAnon` growth alone is a **fact**, not a conclusion, confirmed directly above: `leak_revisited` and `growth_bounded` produce visually indistinguishable `RssAnon` columns, yet only one of them is a leak by LSan's definition (Step 3.3). The field tells you *what kind* of memory; it cannot by itself tell you *why* it's growing.

</details>

---

## Step 3.3 — Growth: An Unbounded Cache

📄 **Full file:** Appendix A.2 (`growth_bounded.cpp`).

A cache that never evicts — a request log, a memoization table, a session registry with no expiry. Every entry stays **reachable** through the `cache` vector, so nothing is ever "lost." Bounded so it terminates.

```bash
./build/debug/lab/growth_bounded 40
```

### 🔬 PREDICT

1. Will `RssAnon` grow roughly linearly with the number of rounds, the way Chapter 1's leak grew linearly with time?
2. At exit, will LeakSanitizer report anything?
3. If you `strace -c` this program, would you expect it to look different from Chapter 1's leak in terms of *what kind* of memory operations dominate?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data** (`growth_bounded 40`, kB):
> | Round | VmSize | RssAnon | entries | minor faults (this interval) |
> |-------|--------|---------|---------|-------------------------------|
> | 0 | 6,372 | 200 | 0 | — |
> | 5 | 7,432 | 992 | 10,000 | 105 |
> | 10 | 8,340 | 1,776 | 20,000 | 40 |
> | 20 | 10,288 | 3,336 | 40,000 | 39 |
> | 40 | 14,184 | 6,464 | 80,000 | 39 |

1. **Yes, roughly linear** — about 160 kB of `RssAnon` per round of 2,000 entries (about 80 bytes/entry: the string content plus `std::string`'s and `std::vector`'s bookkeeping). Indistinguishable in *shape* from Chapter 1's leak.
2. > **📊 Reference data (ASan/LSan build, 20 rounds):** exit code **0**, no leak report. The program printed `final: 40000 entries still reachable through \`cache\`` — LSan agrees they're reachable and says nothing.

   Verified: this is **not a leak** by LSan's definition, and LSan correctly says so.
3. Structurally similar (repeated small allocations), but the *content* differs: a leak often reuses the same code path allocating the same-shaped garbage, while a growing cache's allocations are usually tied to distinguishable inputs (this program's strings each encode their own round and index). In a real system this is often the only external clue: the *keys* going into the cache tell you it's demand-driven growth, not a bug in a fixed code path.

</details>

### 🤔 EXPLAIN

Why is an unbounded cache arguably **more dangerous** in production than a classic leak, given that sanitizers catch the leak and not the cache?

<details>
<summary>✅ Solution</summary>

A classic leak is a **fixed bug**: it exists at one line of code, sanitizers find it in CI, and it's fixed once. An unbounded cache is a **design decision that looks correct** until traffic grows: it passes every test, ships, and only becomes a problem hours or days into production under real load — exactly the long-running scenario Debugging Wizard exists for. It also has no sanitizer to catch it, ever, at any stage. Diagnosing it needs exactly what Chapters 1-2 built: watching `RssAnon` over time and asking "why does this keep growing," not waiting for a tool to flag a line of code.

</details>

---

## Step 3.4 — Fragmentation: Freed Memory That Doesn't Come Back

📄 **Full file:** Appendix A.3 (`fragmentation.cpp`).

Allocate many same-size blocks, free **every other one** (a checkerboard of holes), then ask for one big block equal in size to everything just freed.

```bash
./build/debug/lab/fragmentation                    # 20,000 x 2,048 bytes (~40 MB, below mmap threshold)
./build/debug/lab/fragmentation 20000 64            # smaller chunks
./build/debug/lab/fragmentation 2000 200000         # chunks ABOVE the 128 KiB mmap threshold (Ch2, Step 2.6)
```

### 🔬 PREDICT

For the **default** run (2,048-byte blocks, heap-allocated per Chapter 2's mmap-threshold rule):

1. After freeing every other block, does `RssAnon` drop?
2. The big allocation requests *exactly* the number of bytes just freed. Does `VmSize` stay flat (reusing the holes) or jump (fresh memory)?
3. Repeat your prediction for the **large-block** run (200,000 bytes each, above the mmap threshold). Same answers, or different?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data — default (20,000 × 2,048 bytes, kB):**
> | Step | VmSize | RssAnon |
> |------|--------|---------|
> | 0 baseline | 6,372 | 204 |
> | 1 all allocated+touched | 46,792 | 40,676 |
> | 2 freed every other block | **46,792** | **40,676** |
> | 3 one big alloc (= all freed bytes, 20,000 kB) | **66,796** | 40,680 |
> | 4 everything freed | 6,608 | 492 |

1. **No — `RssAnon` did not move at all** between steps 1 and 2 (40,676 kB both times). glibc's allocator keeps freed heap chunks in its own free lists for reuse; it does not hand pages back to the kernel just because you called `free()`.
2. **`VmSize` jumped by about 20,004 kB — almost exactly the request size — a fresh allocation, not reuse of the holes.** Even though the *total* freed space equaled the request, no single **contiguous** run of holes was that large (they alternate with still-live blocks), so the allocator had to go elsewhere. This is fragmentation made visible: *enough freed memory existed, but not in one usable piece.*

> **📊 Reference data — large blocks (2,000 × 200,000 bytes, above the mmap threshold, kB):**
> | Step | VmSize | RssAnon |
> |------|--------|---------|
> | 0 baseline | 6,372 | 200 |
> | 1 all allocated+touched | 398,372 | 8,220 |
> | 2 freed every other block | **202,372** | **4,220** |
> | 3 one big alloc (= all freed bytes, ≈190,725 kB) | 397,688 | 4,224 |
> | 4 everything freed | 6,372 | 220 |

3. **Different, and instructively so.** This time `VmSize` **dropped by half at step 2** — because each block above the mmap threshold is its own individual `mmap`, so `free()` calls `munmap()` and the address space really is returned immediately (matching Chapter 2, Step 2.6). But step 3 *still* jumps back up by almost the full request size: 1,000 separate freed mappings, however much total space they cover, are **1,000 separate address ranges**, and no allocator can weld them into one contiguous block on request. This is a different flavor of the same lesson: **fragmentation is about contiguity, not total free bytes**, whether the fragments are heap chunks or separate VMAs.

</details>

### 👀 OBSERVE

Run the **small-block** case (`fragmentation 20000 64`, well below the mmap threshold) and fill in:

```
step 2 (freed every other) vs step 1: RssAnon changed by ______ kB
step 3 (big alloc) vs step 2:         VmSize changed by ______ kB
```

<details>
<summary>✅ Solution</summary>

> **📊 Reference data:** step 2 vs step 1, `RssAnon` changed by **+4 kB** (essentially flat, same as the default case). Step 3 vs step 2, `VmSize` changed by **+628 kB**, almost exactly the requested 625 kB — **also a fresh request, not reuse**, even at small size. The relative *size* of the ask compared to the individual hole sizes matters more than absolute scale: many tiny holes still can't satisfy one request that needs to be contiguous within them.

</details>

### ⚠️ A caution before you try this under ASan

**PREDICT first:** if you run `fragmentation` under the ASan build, do you expect the same pattern?

<details>
<summary>✅ Solution</summary>

**No — and this matters.** ASan **replaces malloc entirely** with its own allocator (redzones, quarantine — Chapter 1, Step 1.6-1.10). Verified: under ASan, `RssAnon` **rose at every single step**, including "freed every other block" and "everything freed" — it never came back down within the run, because ASan's quarantine holds freed memory rather than returning it for reuse, exactly like Chapter 1's benchmark discovery. **A sanitizer build is the wrong tool for studying an allocator's fragmentation behavior**, because you'd be studying ASan's allocator, not glibc's. Use the plain build for this experiment; save ASan for correctness bugs.

</details>

---

## Step 3.5 — Excessive Allocation: The Cost of Many Small Pieces

📄 **Full file:** Appendix A.4 (`excess_alloc.cpp`).

Same total payload, delivered two ways: **one** big allocation, or **many** small ones. Both are touched page-by-page and freed correctly — no leak, no fragmentation (nothing alternates). The only variable is allocation *count*.

```bash
./build/debug/lab/excess_alloc 8192 32     # 8 MB payload, 32-byte chunks
./build/debug/lab/excess_alloc 8192 16     # same payload, 16-byte chunks
```

### 🔬 PREDICT

1. Will the many-small version use **more** resident memory than the one-big version, for the *same* payload? Roughly what overhead, as a percentage?
2. Does halving the chunk size (32 → 16 bytes) roughly double the overhead percentage, or worse?
3. Which costs more wall-clock time, and by how much?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data (8,192 kB payload; three repeats of the 32-byte case shown for spread):**
> | Chunk size | Version | RssAnon Δ (kB) | Overhead | Wall time (ms) |
> |-----------|---------|----------------|----------|----------------|
> | 32 B | one big | 8,192-8,196 | — | 2.1-3.3 |
> | 32 B | many small (262,144 chunks) | 14,332 | **+75.0%** | 6.9-9.1 |
> | 16 B | one big | 8,196 | — | 2.3 |
> | 16 B | many small (524,288 chunks) | 20,476 | **+149.9%** | 13.8 |

1. **Yes — about 75% more resident memory** for 32-byte chunks, for *identical payload bytes*. Every heap chunk carries a size-header (glibc's minimum usable chunk is 24-32 bytes on 64-bit systems, regardless of what you asked for), so a 32-byte request already pays close to 100% overhead per chunk in the worst case, averaging out to the ~75% observed here.
2. **Worse than double: roughly 2x the overhead percentage (75% → 150%), not a fixed extra amount.** Once the request size drops toward or below glibc's minimum chunk payload, the **fixed** per-chunk overhead becomes a *larger fraction of a smaller request*: overhead is roughly constant in bytes-per-chunk, so shrinking the payload-per-chunk directly inflates the percentage.
3. **Many-small is consistently slower — roughly 3x here (2-3 ms vs 7-9 ms at 32 bytes; the gap widens further at 16 bytes, 2.3 ms vs 13.8 ms)**, because each small allocation is a separate function call with its own bookkeeping, on top of touching more distinct pages relative to payload.

</details>

### 🧠 THINK

`excess_alloc` under ASan behaves fine (no leak, exits 0), but its RSS numbers look different from the plain build. Why would ASan change the *overhead percentage* itself, not just the absolute numbers?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data (ASan build, 2,048 kB payload, 32-byte chunks):** one-big RssAnon +2,252 kB, many-small +4,108 kB — overhead **90.6%**, versus 75.0% in the plain build at the larger size.

ASan adds its **own** fixed overhead per allocation (redzones before and after every block, metadata to track it), on top of glibc's. That fixed cost is being measured by the exact same mechanism as this experiment: many small allocations pay a per-allocation tax more times than one big allocation does. ASan doesn't change *what* we're measuring, it adds another layer of exactly the same phenomenon — which is a good sign that the experiment is measuring something real, not an artifact of one allocator's internals.

</details>

---

## Step 3.6 — A Bug-Signature Table

The point of Chapters 1-3 together: recognize a category from its **numbers**, before you ever reach for a sanitizer.

| Category | `VmSize` | `RssAnon` | Minor faults | LSan at exit | Fixed by |
|----------|----------|-----------|--------------|---------------|----------|
| **Leak** | Grows, unbounded | Grows, unbounded | Grows with time | **Reports it** | Find and fix the lost pointer |
| **Growth** | Grows, unbounded | Grows, unbounded | Grows with time | **Silent** — reachable | Add a bound: eviction, TTL, a cap |
| **Fragmentation** | Flat after `free()`, then **jumps** on a large request despite "enough" freed bytes | Flat after `free()` | Low after `free()`, spikes on the next large request | **Silent** — everything freed correctly | Pool allocator, avoid mixed alloc sizes, or `malloc_trim` |
| **Excessive allocation** | Grows faster than payload size would suggest | Grows faster than payload size would suggest | Grows with chunk count, not payload size | **Silent** — everything freed correctly | Batch allocations, reserve capacity up front |

**Leak vs growth look identical in the numbers alone** — the only distinguishing fact is whether a sanitizer's exit-time reachability scan flags it, which means the difference is **only visible if you can afford to let the program exit and check**. For a program that never exits, that distinguishing test doesn't exist — which is exactly the situation Debugging Wizard itself will be in, and exactly why later phases (10-13) build event and correlation tooling instead of relying on this one exit-time check.

### 🧠 THINK

Given the table, could `dw_core`'s `read_mem_snapshot` **alone**, sampled once, ever tell you which of these four categories you're looking at?

<details>
<summary>✅ Solution</summary>

**No — a single snapshot is a fact, not a diagnosis** (your master document's §28, Fact vs Interpretation). Every category shares the same instantaneous signature: elevated `RssAnon`. Distinguishing them requires the **trend** (does it grow without bound, or plateau?), an **event** (a large allocation request and whether it triggers a jump), or an **external check** (LSan's exit-time reachability scan — something a snapshot alone cannot do). This is precisely why the project's reasoning chain runs *observation → correlation → hypothesis → verification*, not straight from one measurement to a conclusion.

</details>

---

## Step 3.7 — Explain It Back, Knowledge Check, Completion

### 🎯 Explain-it-back (no notes)

> Growth and a classic leak both show `RssAnon` climbing steadily, and neither one crashes or slows down obviously in the short term. Explain how you would tell them apart *without* waiting for the program to exit, and why that's hard.

<details>
<summary>✅ Model answer</summary>

Without an exit-time scan, the two are genuinely difficult to tell apart from `RssAnon` alone, because both show the same "grows without bound" signature. The practical distinguishing evidence is contextual, not numeric: does growth correlate with a plausible driver (more cache keys, more sessions, more traffic) that would justify more *live, useful* memory, or does it grow independent of any such driver? Correlating the growth rate with an external signal — request rate, distinct keys seen, active connections — is the kind of cross-layer evidence Debugging Wizard's later phases (12: correlation, 13: diagnosis) are built to gather. A single memory reader, however well built, cannot resolve this alone; it can only supply the fact that something is growing.

</details>

### Knowledge Check

**Level 1: Recall.** Why doesn't LeakSanitizer report anything for `growth_bounded`?

<details><summary>✅ Solution</summary>Because every entry is still reachable through the live <code>cache</code> vector at exit. LSan's definition of a leak is unreachable memory, not merely memory that keeps growing.</details>

**Level 2: Understanding.** In the fragmentation experiment, `RssAnon` didn't drop after freeing half the blocks. Where did that memory actually go?

<details><summary>✅ Solution</summary>Nowhere — it's still resident RAM, mapped into the process, sitting in glibc's internal free lists waiting to be reused by a future allocation of a similar size. <code>free()</code> returns a chunk to the allocator, not to the kernel; the pages stay mapped and resident unless the allocator specifically decides to trim them.</details>

**Level 3: Mechanism.** Why did the large-block fragmentation run show `VmSize` actually dropping at step 2, when the small-block run didn't?

<details><summary>✅ Solution</summary>Blocks above glibc's mmap threshold (128 KiB by default, Chapter 2 Step 2.6) are each their own individual <code>mmap</code> region, so <code>free()</code> on one of them calls <code>munmap()</code> immediately, genuinely returning that address range to the kernel. Small blocks live inside the shared heap (<code>brk</code>) arena, which glibc does not shrink on every individual <code>free()</code>.</details>

**Level 4: Experiment.** How would you check whether a real program's `RssAnon` growth is "growth" (Step 3.3) rather than "excessive allocation" (Step 3.5)?

<details><summary>✅ Solution</summary>Compare the growth rate to the *payload* it's supposedly holding: track how many live, useful items exist (cache entries, sessions, whatever the domain object is) alongside <code>RssAnon</code>. If RSS grows roughly proportional to a reasonable per-item size, it's likely legitimate growth (Step 3.3's pattern). If RSS grows much faster than the payload would suggest — many small allocations paying a large per-chunk fixed cost — that points to excessive allocation (Step 3.5's pattern), fixable by batching without changing what's logically stored.</details>

**Level 5: Debugging.** A colleague says "we switched to ASan and our fragmentation problem disappeared, we're safe now." What's wrong with that conclusion?

<details><summary>✅ Solution</summary>ASan replaces the allocator entirely, so what "disappeared" is glibc's fragmentation behavior specifically — it was never being tested under ASan, since ASan's own quarantine-based allocator has completely different (and, per Step 3.4's reference data, actually worse in this respect — memory never returns) characteristics. The production binary uses glibc's allocator, not ASan's; nothing about production fragmentation was actually verified.</details>

**Level 6: Systems reasoning.** All four bug categories can produce a program that "passes every test" and "runs fine" for a while. What property do they all share that makes them specifically dangerous for a *long-running* tool like Debugging Wizard, as opposed to a short batch job?

<details><summary>✅ Solution</summary>
All four are properties that only become visible or costly **over time or under sustained load** — a short-lived process exits before a leak or unbounded cache accumulates to a noticeable size, before fragmentation is exercised by enough alternating-size requests, before excessive-allocation overhead multiplies across enough operations to matter. A tool meant to run for hours or days (your master document's §5, Continuous Operation) turns every one of these from "theoretically present" into "eventually fatal," which is exactly why Phase 0's self-observation habits and Phase 1's continuous-operation testing exist: correctness at t=0 says nothing about health at t=6 hours.
</details>

### Exercises

1. **Distinguish leak from growth experimentally.** Modify `growth_bounded` so that instead of appending to `cache`, it does what Chapter 1's leak did — allocate and immediately drop the pointer each round. Confirm LSan *does* now report it, and that the `RssAnon` growth curve looks the same either way.
2. **Trim the fragmentation.** After freeing every other block in `fragmentation`, call `malloc_trim(0)` before the big allocation. Predict whether `VmSize` still jumps, then test.
3. **Reserve to fix excessive allocation.** In `excess_alloc`'s many-small path, replace individually-touched small `malloc`s with one `std::vector<char>` sized to the full payload up front. Measure the new overhead.
4. **A fifth category?** Your master document also lists "double allocation" and "forgotten free" as distinct entries (§8). Which of this chapter's four categories does each actually reduce to, and why did we not need a fifth program?
5. **Validate with an independent tool, if available.** If `valgrind --tool=massif` is installed on your machine, run it against `fragmentation` and compare its picture of heap usage over time with `dw_core`'s.

---

## Chapter Completion Criteria

I can:

- [ ] Name the four bug categories and explain what distinguishes each in the kernel's own numbers
- [ ] Explain why growth is invisible to LeakSanitizer, and why that makes it more dangerous, not less
- [ ] Explain the fragmentation result: `RssAnon` flat after `free()`, `VmSize` still jumping on a later large request
- [ ] Explain why large (mmap-threshold) blocks behave differently under fragmentation than small (heap) blocks
- [ ] Explain why ASan is the wrong tool for studying glibc's fragmentation or overhead behavior
- [ ] Quantify the overhead of many small allocations versus one big one, and explain why it worsens as chunk size shrinks
- [ ] Explain why a single memory snapshot cannot, by itself, distinguish these four categories
- [ ] Answer all six knowledge-check levels without opening the solutions

---

## What This Unlocks Next

**Chapter 4 (error handling and logging).** Every experiment in this chapter assumed allocations succeed and `/proc` reads work. Chapter 4 builds the discipline for when they don't — and gives Debugging Wizard a way to *report* what it found here (a growing cache, a fragmented heap) rather than just printing numbers to a terminal.

---

## Clarification Log

> Ask me anything about any step. Each question and answer can be appended here so this file becomes your complete study record.

| # | Step | Your question | Answer summary |
|---|------|---------------|----------------|
| 1 | | | |
| 2 | | | |
| 3 | | | |

---

## Appendix A — Complete Code, File by File

Everything new or changed in this chapter is on this page; no download is needed. Each listing is the exact file that was compiled and tested while preparing this note (the CMake file excepted, see the honesty note at the top). This chapter adds no files to `src/`, `include/`, or `tests/` — see the honesty note for why.

Create or replace each file at the path in its heading (all under your existing `lab/` folder).

### A.1 `lab/leak_revisited.cpp`
**New.** Step 3.2: the same defect as Chapter 1's `leak_bounded`, but self-observing (inline `dw::MemSnapshot`), matching this chapter's other experiments.
```cpp
// The Chapter 1 leak, revisited: same defect (allocate 1 MB, touch one byte,
// never free), but SELF-OBSERVING -- no background process, no external
// observer needed. Every round prints its own dw::MemSnapshot, the same
// style as growth_bounded.cpp, so leak and growth can be compared side by
// side using identical instrumentation.
// Usage: leak_revisited [rounds=20]
#include <cstdio>
#include <cstdlib>

#include "dw/proc_self.hpp"

namespace {

void print_header() {
    std::printf("%-6s %-12s %-12s %-10s\n", "round", "VmSize(kB)", "RssAnon(kB)", "minflt(d)");
}

void print_row(int round, const dw::MemSnapshot& s, long minflt_delta) {
    std::printf("%-6d %-12ld %-12ld %-10ld\n", round, s.vm_size_kb, s.rss_anon_kb, minflt_delta);
}

}  // namespace

int main(int argc, char** argv) {
    const int rounds = (argc > 1) ? std::atoi(argv[1]) : 20;

    dw::MemSnapshot prev;
    dw::read_mem_snapshot(prev);
    print_header();
    print_row(0, prev, 0);

    for (int r = 1; r <= rounds; ++r) {
        char* p = static_cast<char*>(std::malloc(1024 * 1024));  // 1 MB
        if (p == nullptr) {
            std::fprintf(stderr, "allocation failed at round %d\n", r);
            return 1;
        }
        p[0] = 'x';  // touch one byte -> one 4 KB page becomes resident
        // deliberately never freed

        dw::MemSnapshot now;
        dw::read_mem_snapshot(now);
        print_row(r, now, now.minor_faults - prev.minor_faults);
        prev = now;
    }

    std::printf("\nno pointer to any of the %d blocks survives past this point: this IS a leak\n",
                rounds);
    return 0;
}
```

### A.2 `lab/growth_bounded.cpp`

**New.** Step 3.3: an unbounded cache — reachable, so invisible to LeakSanitizer.

```cpp
// GROWTH, not a leak: an unbounded cache that keeps every entry forever.
// Every pointer stays REACHABLE (held in `cache`), so this is invisible to
// LeakSanitizer -- the defect is a missing bound, not a lost pointer.
// Bounded so it terminates (usage: growth_bounded [rounds=40]).
#include <cstdio>
#include <cstdlib>
#include <string>
#include <vector>

#include "dw/proc_self.hpp"

namespace {

void print_header() {
    std::printf("%-6s %-12s %-12s %-10s %-10s\n", "round", "VmSize(kB)", "RssAnon(kB)",
                "entries", "minflt(d)");
}

void print_row(int round, const dw::MemSnapshot& s, long entries, long minflt_delta) {
    std::printf("%-6d %-12ld %-12ld %-10ld %-10ld\n", round, s.vm_size_kb, s.rss_anon_kb, entries,
                minflt_delta);
}

}  // namespace

int main(int argc, char** argv) {
    const int rounds = (argc > 1) ? std::atoi(argv[1]) : 40;

    // A "cache" that never evicts: a common real-world growth bug (a request
    // log, a memoization table, a connection registry that never prunes).
    std::vector<std::string> cache;

    dw::MemSnapshot prev;
    dw::read_mem_snapshot(prev);
    print_header();
    print_row(0, prev, 0, 0);

    for (int r = 1; r <= rounds; ++r) {
        for (int i = 0; i < 2000; ++i) {
            cache.push_back("cached-entry-" + std::to_string(r) + "-" + std::to_string(i));
        }
        dw::MemSnapshot now;
        dw::read_mem_snapshot(now);
        if (r % 5 == 0 || r == rounds) {
            print_row(r, now, static_cast<long>(cache.size()), now.minor_faults - prev.minor_faults);
        }
        prev = now;
    }

    std::printf("\nfinal: %zu entries still reachable through `cache`\n", cache.size());
    return 0;
}
```
### A.3 `lab/fragmentation.cpp`

**New.** Step 3.4: alternating allocation pattern, watched through `dw_core`.

```cpp
// FRAGMENTATION: free every other block, leaving a checkerboard of holes,
// then ask for one big contiguous block. Watch whether the allocator reuses
// the holes or asks the kernel for fresh memory anyway.
// Usage: fragmentation [blocks=20000] [block_bytes=2048]
#include <cstdio>
#include <cstdlib>
#include <vector>

#include "dw/proc_self.hpp"

namespace {

dw::MemSnapshot snap() {
    dw::MemSnapshot s;
    dw::read_mem_snapshot(s);
    return s;
}

void row(const char* label, const dw::MemSnapshot& s, const dw::MemSnapshot& base) {
    std::printf("%-32s %10ld %10ld %12ld\n", label, s.vm_size_kb, s.rss_anon_kb,
                s.minor_faults - base.minor_faults);
}

}  // namespace

int main(int argc, char** argv) {
    const long blocks = (argc > 1) ? std::atol(argv[1]) : 20000;
    const size_t block_bytes = (argc > 2) ? static_cast<size_t>(std::atol(argv[2])) : 2048;

    std::printf("%-32s %10s %10s %12s\n", "step", "VmSize kB", "RssAnon kB", "minflt(d)");
    const dw::MemSnapshot base = snap();
    row("0 baseline", base, base);

    std::vector<char*> ptrs;
    ptrs.reserve(static_cast<size_t>(blocks));
    for (long i = 0; i < blocks; ++i) {
        char* p = static_cast<char*>(std::malloc(block_bytes));
        if (p == nullptr) {
            std::fprintf(stderr, "allocation failed at block %ld\n", i);
            return 1;
        }
        p[0] = 1;  // touch it so the page is actually resident
        ptrs.push_back(p);
    }
    row("1 all blocks allocated+touched", snap(), base);

    // Free every other block: leaves a checkerboard of used/free chunks,
    // so no run of freed chunks is long enough to satisfy a big request.
    for (size_t i = 1; i < ptrs.size(); i += 2) {
        std::free(ptrs[i]);
        ptrs[i] = nullptr;
    }
    row("2 freed every other block", snap(), base);

    // Now ask for one big block, as large as ALL the freed space combined.
    const size_t freed_bytes = (ptrs.size() / 2) * block_bytes;
    void* big = std::malloc(freed_bytes);
    if (big == nullptr) {
        std::fprintf(stderr, "big allocation failed\n");
        return 1;
    }
    static_cast<char*>(big)[0] = 1;
    row("3 one big alloc = all freed bytes", snap(), base);

    std::free(big);
    for (char* p : ptrs) {
        std::free(p);
    }
    row("4 everything freed", snap(), base);

    std::printf("\nrequested for the big block: %zu bytes (%.0f kB)\n", freed_bytes,
                static_cast<double>(freed_bytes) / 1024.0);
    return 0;
}
```
### A.4 `lab/excess_alloc.cpp`

**New.** Step 3.5: one big allocation vs many small ones, same payload.

```cpp
// EXCESSIVE ALLOCATION: the same total payload as either ONE allocation or
// MANY small ones. Measures the overhead the many-small-allocations version
// pays: extra RSS, extra minor faults, extra wall time.
// Usage: excess_alloc [total_kb=8192] [small_bytes=32]
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <vector>

#include "dw/proc_self.hpp"

namespace {

dw::MemSnapshot snap() {
    dw::MemSnapshot s;
    dw::read_mem_snapshot(s);
    return s;
}

struct Result {
    double wall_s = 0;
    long rss_anon_delta_kb = 0;
    long minflt_delta = 0;
};

Result one_big_allocation(size_t total_bytes) {
    const dw::MemSnapshot before = snap();
    const auto t0 = std::chrono::steady_clock::now();

    char* p = static_cast<char*>(std::malloc(total_bytes));
    for (size_t off = 0; off < total_bytes; off += 4096) {
        p[off] = 1;  // touch every page once, same total work as the other version
    }

    const auto t1 = std::chrono::steady_clock::now();
    const dw::MemSnapshot after = snap();
    std::free(p);

    return {std::chrono::duration<double>(t1 - t0).count(), after.rss_anon_kb - before.rss_anon_kb,
            after.minor_faults - before.minor_faults};
}

Result many_small_allocations(size_t total_bytes, size_t chunk_bytes) {
    const size_t count = total_bytes / chunk_bytes;
    std::vector<char*> ptrs;
    ptrs.reserve(count);

    const dw::MemSnapshot before = snap();
    const auto t0 = std::chrono::steady_clock::now();

    for (size_t i = 0; i < count; ++i) {
        char* p = static_cast<char*>(std::malloc(chunk_bytes));
        p[0] = 1;  // touch the (one) page this chunk lives on
        ptrs.push_back(p);
    }

    const auto t1 = std::chrono::steady_clock::now();
    const dw::MemSnapshot after = snap();
    for (char* p : ptrs) {
        std::free(p);
    }

    return {std::chrono::duration<double>(t1 - t0).count(), after.rss_anon_kb - before.rss_anon_kb,
            after.minor_faults - before.minor_faults};
}

}  // namespace

int main(int argc, char** argv) {
    const size_t total_kb = (argc > 1) ? static_cast<size_t>(std::atol(argv[1])) : 8192;
    const size_t small_bytes = (argc > 2) ? static_cast<size_t>(std::atol(argv[2])) : 32;
    const size_t total_bytes = total_kb * 1024;

    const Result big = one_big_allocation(total_bytes);
    const Result many = many_small_allocations(total_bytes, small_bytes);

    std::printf("payload requested        : %zu kB\n", total_kb);
    std::printf("small-chunk size          : %zu bytes  (%zu chunks)\n\n", small_bytes,
                total_bytes / small_bytes);

    std::printf("%-24s %12s %14s %12s\n", "version", "wall(ms)", "RssAnon d(kB)", "minflt(d)");
    std::printf("%-24s %12.2f %14ld %12ld\n", "one big allocation", big.wall_s * 1000,
                big.rss_anon_delta_kb, big.minflt_delta);
    std::printf("%-24s %12.2f %14ld %12ld\n", "many small allocations", many.wall_s * 1000,
                many.rss_anon_delta_kb, many.minflt_delta);

    const double overhead_kb = static_cast<double>(many.rss_anon_delta_kb - big.rss_anon_delta_kb);
    std::printf("\nRSS overhead of many-small vs one-big : %.0f kB (%.1f%% of payload)\n", overhead_kb,
                overhead_kb / static_cast<double>(total_kb) * 100.0);
    return 0;
}
```
### A.5 `lab/CMakeLists.txt`

```cmake
# Lab = throwaway experiments and deliberately broken programs.
# Code graduates to src/ only after it has earned its place.
function(dw_lab name)
  add_executable(${name} ${name}.cpp)
  target_link_libraries(${name} PRIVATE dw_options)
endfunction()

dw_lab(hello)
dw_lab(leak_bounded)
dw_lab(bugs)
dw_lab(fd_leak)
dw_lab(fd_raii)
dw_lab(bench)
dw_lab(observe_pid)
dw_lab(run_measure)

# Programs that use the real tool's library (dw_core).
function(dw_lab_core name)
  add_executable(${name} ${name}.cpp)
  target_link_libraries(${name} PRIVATE dw_core)
endfunction()

dw_lab_core(maps_summary)
dw_lab_core(touch_pages)
dw_lab_core(heap_vs_mmap)
dw_lab_core(leak_revisited)
dw_lab_core(growth_bounded)
dw_lab_core(fragmentation)
dw_lab_core(excess_alloc)
```

### A.6 Build and run

```bash
cmake --build --preset debug -j
./build/debug/lab/leak_revisited 20
./build/debug/lab/growth_bounded 40
./build/debug/lab/fragmentation
./build/debug/lab/fragmentation 2000 200000
./build/debug/lab/excess_alloc 8192 32
```

**Expected:** all four build with no warnings; `leak_revisited` and `growth_bounded` both print a steadily growing `RssAnon` column (visually indistinguishable from each other — that's the point of Step 3.2/3.3); `fragmentation` prints five rows where `RssAnon` stays flat between rows 1 and 2, then `VmSize` jumps at row 3; `excess_alloc` prints two version rows with the many-small one showing higher `RssAnon` delta and wall time.

If the build errors, copy the full message and ask me. Then: `git add -A && git commit -m "Phase 0 chapter 3: bug-category experiments"`.
