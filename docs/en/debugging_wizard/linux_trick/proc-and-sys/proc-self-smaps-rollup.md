---
topic: proc-and-sys
tags: /proc/self/smaps_rollup, smaps_rollup, smaps, Pss, Rss, Private_Dirty, Shared_Clean, Referenced, AnonHugePages
related: proc-self-status.md, proc-self-maps.md, ../memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# `/proc/self/smaps_rollup`

## 1. What it is

A **one-screen summary of the memory detail** that `/proc/<PID>/smaps` gives per mapping, **added up across all mappings** of the process. It answers questions `status` cannot: how much memory is *really mine* (`Pss`), and how much is *modified* (dirty) versus *unchanged* (clean).

| Path | Notes |
|------|-------|
| `/proc/self/smaps_rollup` | The reading process |
| `/proc/<PID>/smaps_rollup` | Another process (access-checked like `maps`) |
| `/proc/<PID>/smaps` | The same fields **per mapping** (much longer) |

**Where the data comes from:** unlike `status` (which reads ready-made counters), the kernel must **walk the process's page tables** to compute these. The first line of the file is a synthetic `[rollup]` line covering the whole address range.

## 2. Commands you actually use

```bash
cat /proc/self/smaps_rollup
grep -E '^(Rss|Pss|Anonymous|Swap)' /proc/<PID>/smaps_rollup
awk '/^Pss:/ {s += $2} END {print s " kB"}' /proc/[0-9]*/smaps_rollup 2>/dev/null   # rough total memory across processes
```

## 3. Two ideas you need first

**Clean vs dirty.** A page is **clean** if it is identical to what is on disk (it can simply be dropped and re-read). It is **dirty** if it was modified (it must be written out or swapped before reuse). Anonymous memory you wrote to is dirty; untouched program code is clean.

**Shared vs private.** A page is **shared** if more than one process maps it (libraries), **private** if only this one does.

**`Pss` (proportional set size):** each shared page is **divided among the processes sharing it**. If a 4 kB page is shared by 4 processes, each is charged 1 kB. Adding `Rss` across processes over-counts shared libraries; adding `Pss` does not.

## 4. Field reference (real output, kB)

| Field | Sample | Meaning |
|-------|--------|---------|
| `Rss` | 1632 | Resident set size (same idea as `VmRSS`) |
| `Pss` | 655 | Proportional size: private pages plus a fair share of shared ones |
| `Pss_Dirty` | 100 | The dirty part of `Pss` |
| `Pss_Anon` | 100 | The anonymous part of `Pss` |
| `Pss_File` | 555 | The file-backed part of `Pss` |
| `Pss_Shmem` | 0 | The shared-memory part of `Pss` |
| `Shared_Clean` | 1496 | Shared pages, unmodified |
| `Shared_Dirty` | 0 | Shared pages, modified |
| `Private_Clean` | 36 | Private pages, unmodified |
| `Private_Dirty` | 100 | Private pages, modified (the memory that is truly "yours") |
| `Referenced` | 1632 | Pages accessed recently (used by the kernel's page-reclaim logic) |
| `Anonymous` | 100 | Anonymous pages (no backing file) |
| `KSM` | 0 | Pages merged with identical pages by kernel same-page merging |
| `LazyFree` | 0 | Pages freed with `MADV_FREE`, reclaimable when memory is needed |
| `AnonHugePages` | 0 | Anonymous memory backed by transparent huge pages |
| `ShmemPmdMapped`, `FilePmdMapped` | 0 | Shared-memory / file pages mapped through huge-page entries |
| `Shared_Hugetlb`, `Private_Hugetlb` | 0 | Memory in explicit hugetlb pages |
| `Swap` | 0 | Memory swapped out |
| `SwapPss` | 0 | Proportional share of swapped memory |
| `Locked` | 0 | Memory locked with `mlock` |

## 5. Identities that hold (verified on the real output above)

| Identity | Check |
|----------|-------|
| `Rss = Shared_Clean + Shared_Dirty + Private_Clean + Private_Dirty` | 1496 + 0 + 36 + 100 = **1632** ✓ |
| `Pss = Pss_Anon + Pss_File + Pss_Shmem` | 100 + 555 + 0 = **655** ✓ |

Use identities like these to **validate a parser**: if they fail, the parser (or the reading) is wrong.

**Reading the sample:** `Rss` is 1632 kB, but `Pss` only 655 kB, because most resident pages are **shared library code** (`Shared_Clean` 1496 kB) that other processes also map. The memory this process alone is responsible for is small (`Private_Dirty` 100 kB).

## 6. How this relates to `status`

| Question | Use |
|----------|-----|
| How much is resident? | `VmRSS` in [status](proc-self-status.md) (cheap counters) |
| Resident split anon / file / shmem? | `RssAnon`, `RssFile`, `RssShmem` in `status` |
| What is my fair share of shared libraries? | `Pss` here |
| How much would freeing this process actually release? | Roughly `Private_Clean + Private_Dirty` here (shared pages stay while others use them) |
| Is memory being modified or just read? | Dirty vs clean here |

## 7. Gotchas

- **More expensive than `status`.** The kernel walks page tables, so polling this file at high frequency costs noticeably more. Measure before putting it in a fast loop.
- **Different processes, different instants.** Do not compare `Anonymous` here (100 kB) with `RssAnon` from a *separate* `cat /proc/self/status` (104 kB): each `cat` is a new process.
- **Newer kernels add fields** (`KSM`, `LazyFree`, `SwapPss`, ...). Parse by name.
- **`Pss` needs care in tools:** it is a fair-share estimate, not a measurement of what would be freed.

## 8. Related

- [proc-self-status](proc-self-status.md): the cheap counters (`VmRSS`, `RssAnon`, ...).
- [proc-self-maps](proc-self-maps.md): the mapping list these totals cover.
- [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md): what the three RSS buckets are.
