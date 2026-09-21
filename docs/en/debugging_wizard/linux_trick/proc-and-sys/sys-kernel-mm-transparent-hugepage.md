---
topic: proc-and-sys
tags: /sys/kernel/mm/transparent_hugepage, THP, transparent hugepage, hugepage, enabled, defrag, madvise, AnonHugePages, page faults
related: proc-self-smaps-rollup.md, proc-self-status.md, ../memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# `/sys/kernel/mm/transparent_hugepage/`

## 1. What it is

The controls for **Transparent Huge Pages (THP)**: the kernel's ability to back memory with **2 MiB pages instead of 4 KiB pages** automatically, without the program asking. Fewer, bigger pages mean fewer page faults and less page-table overhead, at the cost of possible memory bloat and occasional latency spikes while the kernel finds contiguous memory.

| Path | Notes |
|------|-------|
| `/sys/kernel/mm/transparent_hugepage/enabled` | The main switch (**read this first**) |
| `.../defrag` | How hard the kernel works to get a huge page at fault time |
| `.../hpage_pmd_size` | The huge-page size in bytes (real value: `2097152` = 2 MiB) |
| `.../shmem_enabled` | THP for shared memory and `tmpfs` (real value: `never`) |
| `.../khugepaged/` | Settings for the background thread that merges small pages into huge ones |
| `.../hugepages-<size>kB/` | Per-size controls on newer kernels |

All are readable by anyone and writable by **root** only. `/sys` is the kernel's device and settings tree, like `/proc` but for configuration.

## 2. Reading `enabled`

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# always [madvise] never        <- the value in [brackets] is the ACTIVE one
```

| Value | Meaning |
|-------|---------|
| `always` | THP is used automatically for eligible anonymous memory |
| `madvise` | Only regions that the program marked with `madvise(MADV_HUGEPAGE)` |
| `never` | THP disabled |

## 3. `defrag` values

| Value | Behaviour when no huge page is free at fault time |
|-------|---------------------------------------------------|
| `always` | The faulting process **stalls** to reclaim or compact memory until it gets one |
| `defer` | Falls back to small pages now; wakes background threads to make huge pages available |
| `defer+madvise` | Stall for `madvise`d regions; defer for the rest |
| `madvise` | Stall only for `madvise`d regions (the real value on the test system) |
| `never` | Never stall; use small pages |

## 4. Commands you actually use

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled            # ALWAYS record this before benchmarking
cat /sys/kernel/mm/transparent_hugepage/hpage_pmd_size
grep -E 'AnonHugePages' /proc/self/smaps_rollup            # huge-page memory of this process
grep THP_enabled /proc/self/status                         # may THP be used by this process?
echo always | sudo tee /sys/kernel/mm/transparent_hugepage/enabled    # change (until reboot)
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled   # restore what you recorded
```

## 5. Why it matters to our experiments

In Chapter 2's `touch_pages`, touching 128 MB with 4 KiB pages cost **exactly 32,768 minor faults**. With `enabled = always`, one fault can install a whole 2 MiB page, so you would expect **roughly 512 times fewer faults** and RSS growing in 2 MiB steps. Same mechanism, different granularity. That is why the note says to record this setting **before comparing results**.

## 6. Gotchas

- **The bracket is the truth.** `always [madvise] never` means `madvise` is active.
- **Benchmarks change with this setting.** Fault counts, RSS growth steps, and allocation latency all depend on it. Record it beside every result, in `docs/environment.md`.
- **Not free.** THP can waste memory (a 2 MiB page for a small use) and cause latency spikes during compaction; some databases recommend turning it off. This is a trade-off to *measure*, not a recommendation.
- **Only for eligible memory.** Regions must be large and aligned enough to hold a 2 MiB page.
- **Changes need root and last until reboot** unless made persistent by other means. Restore the value you recorded after any experiment.
- **`sysfs` files are settings, not measurements.** They tell you what the kernel *may* do, not what it did. For what happened, use `AnonHugePages` in [smaps_rollup](proc-self-smaps-rollup.md).

## 7. Related

- [proc-self-smaps-rollup](proc-self-smaps-rollup.md): `AnonHugePages` shows THP actually in use.
- [proc-self-status](proc-self-status.md): `THP_enabled`, `HugetlbPages`.
- Chapter 2 of the project (`phase-0-ch2.md`): the demand-paging experiment and the hugepage exercise.
