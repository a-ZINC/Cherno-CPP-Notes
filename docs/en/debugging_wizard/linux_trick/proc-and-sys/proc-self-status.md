---
topic: proc-and-sys
tags: /proc/self/status, status, VmSize, VmRSS, VmHWM, VmPeak, RssAnon, Threads, State, Tgid, Uid, capabilities, signals, SigCgt, context switches, Cpus_allowed
related: proc-self-maps.md, proc-self-smaps-rollup.md, proc-self-limits.md, ../memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# `/proc/self/status`

## 1. What it is

A **text file the kernel generates every time you read it**: one `Key:<tab>value` line per fact about a process. Nothing is stored on disk.

| Path | Whose status |
|------|--------------|
| `/proc/self/status` | The process doing the reading (`self`) |
| `/proc/<PID>/status` | Process `<PID>`. **Readable by everyone** (unlike `maps` and `fd`) |
| `/proc/<PID>/task/<TID>/status` | One **thread** of that process |

**Where the data comes from:** MMU page-table activity → memory counters in the process's `mm_struct` and fields in its `task_struct` → formatted as text by the procfs handler → `read()` → your program.

**Cost:** about 4 µs of kernel time per read in our Chapter 2 measurement, mostly formatting these ~60 lines.

## 2. Commands you actually use

```bash
cat /proc/self/status                                        # everything
grep -E 'VmSize|VmRSS|VmHWM' /proc/self/status               # just memory
grep -E 'Threads|ctxt' /proc/<PID>/status                    # threads and scheduling
ls /proc/<PID>/task                                          # one entry per thread ID
```

## 3. Field reference

*Values below come from a real run on a recent kernel. Different kernels show different fields, so always find a field by its **name**, never by line number.*

### 3.1 Identity and state

| Field | Meaning | Notes |
|-------|---------|-------|
| `Name` | Command name (`comm`), max 15 characters | `cat` for the reader itself |
| `Umask` | File-creation mask | `0022` is typical |
| `State` | What the task is doing right now | See the state table below |
| `Tgid` | **Thread-group ID: what user space calls the PID** | Same for all threads |
| `Ngid` | NUMA group ID | `0` if not used |
| `Pid` | **Thread ID (TID)** of this task | Equals `Tgid` for the main thread |
| `PPid` | Parent process ID | |
| `TracerPid` | PID of a tracer (`gdb`, `strace`), or `0` | Non-zero means someone is `ptrace`-attached |
| `Uid`, `Gid` | Four IDs: **real, effective, saved, filesystem** | `1000 1000 1000 1000` for a normal user |
| `FDSize` | Number of descriptor **slots allocated** in the table | **Not** the number of open files (count `/proc/self/fd` for that) |
| `Groups` | Supplementary group IDs | Empty for root here |
| `NStgid`, `NSpid`, `NSpgid`, `NSsid` | The same IDs as seen inside nested PID namespaces | Matter in containers |
| `Kthread` | `1` if this is a kernel thread | Newer kernels |

**`State` letters:**

| Letter | Meaning |
|--------|---------|
| `R` | Running, or runnable (on a run queue). The reader always sees itself as `R` |
| `S` | Sleeping, interruptible (waiting for an event; the usual idle state) |
| `D` | **Uninterruptible** sleep, usually waiting for disk or another device I/O; cannot be killed until it returns |
| `T` / `t` | Stopped by a signal / stopped by a tracer |
| `Z` | Zombie: exited, but the parent has not yet collected its exit status |
| `X` | Dead (rarely visible) |

**Verified with threads:** a process with 3 threads showed `Pid` 90, 91, 92 (one per thread) and `Tgid` 90 for all three.

### 3.2 Memory (per **process**, shared by all threads)

| Field | Meaning | Notes |
|-------|---------|-------|
| `VmPeak` | Highest `VmSize` ever reached | Virtual-size peak |
| `VmSize` | **Total virtual address space** mapped ("VSZ") | The sum of all VMAs, touched or not |
| `VmLck` | Memory locked with `mlock` (cannot be swapped out) | |
| `VmPin` | Pages **pinned** long-term (cannot be moved) | Typically device DMA |
| `VmHWM` | **Peak** resident set size ("high water mark") | Approximate; see gotchas |
| `VmRSS` | **Resident set size**: pages currently in RAM | `= RssAnon + RssFile + RssShmem` |
| `RssAnon` | Resident **anonymous** pages (heap, stack, anonymous `mmap`) | Where leaks show up |
| `RssFile` | Resident **file-backed** pages (program, libraries, mapped files) | |
| `RssShmem` | Resident **shared-memory** pages (`tmpfs`, `shm_open`, `memfd`) | See [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md) |
| `VmData` | Private writable mappings excluding the stack (heap, `.data`/`.bss`, private anonymous) | |
| `VmStk` | Stack size | |
| `VmExe` | Executable code of the main program | |
| `VmLib` | Executable code of shared libraries | |
| `VmPTE` | Memory the kernel uses for this process's **page tables** | Kernel memory, not counted in RSS |
| `VmSwap` | Anonymous memory currently swapped out | Not in RSS |
| `HugetlbPages` | Memory in explicit hugetlb huge pages | |

**Identity check from a real run:** `VmRSS` 1636 kB = `RssAnon` 104 + `RssFile` 1532 + `RssShmem` 0. ✓

### 3.3 Threads and signals

| Field | Meaning | Notes |
|-------|---------|-------|
| `Threads` | Number of threads in the process | Per **process** |
| `SigQ` | `queued/limit` real-time signals | The limit equals **Max pending signals** in `/proc/self/limits` (15931 in both, verified) |
| `SigPnd` | Signals pending for **this thread** (hex mask) | |
| `ShdPnd` | Signals pending for the **whole process** (hex mask) | |
| `SigBlk` | Signals **blocked** (hex mask) | |
| `SigIgn` | Signals **ignored** (hex mask) | |
| `SigCgt` | Signals **caught** by a handler (hex mask) | |

**Decoding a mask:** signal *N* is bit *N−1*. Example: `0000000000004002` = bits 1 and 14 = signals **2 (SIGINT)** and **15 (SIGTERM)**. Handy: `SigCgt` tells you which signals a program handles (a set bit for signal 11 means it handles `SIGSEGV`, as ASan does).

### 3.4 Capabilities and security

| Field | Meaning |
|-------|---------|
| `CapInh`, `CapPrm`, `CapEff`, `CapBnd`, `CapAmb` | Capability sets (hex): inheritable, permitted, **effective** (what it can do now), bounding, ambient. A normal user typically has `CapEff` of all zeros |
| `NoNewPrivs` | `1` = the process can never gain privileges through `exec` |
| `Seccomp` | `0` disabled, `1` strict, `2` filter mode |
| `Seccomp_filters` | Number of seccomp filters attached |
| `Speculation_Store_Bypass`, `SpeculationIndirectBranch` | Status of CPU speculative-execution mitigations for this task |
| `CoreDumping` | `1` while the process is writing a core dump |
| `THP_enabled` | `1` if transparent huge pages may be used for this process |
| `untag_mask` | Address-tagging mask (all ones = no tagging) |

### 3.5 CPU and NUMA placement

| Field | Meaning |
|-------|---------|
| `Cpus_allowed` / `Cpus_allowed_list` | CPUs this task may run on: hex mask / readable list (`0-7`). Set by `taskset` or `sched_setaffinity` |
| `Mems_allowed` / `Mems_allowed_list` | NUMA memory nodes it may allocate from |

### 3.6 Scheduling

| Field | Meaning |
|-------|---------|
| `voluntary_ctxt_switches` | Times the task **gave up the CPU itself** (blocked on I/O, a lock, `sleep`) |
| `nonvoluntary_ctxt_switches` | Times the scheduler **took the CPU away** while the task was still runnable (preemption) |

**Verified:** in a test with two busy-spinning threads and one sleeping main thread, the spinners were preempted about **10 times more** (78-79 non-voluntary switches versus 8). A high non-voluntary count suggests CPU contention; a high voluntary count suggests frequent waiting.

## 4. Which fields are per-thread and which per-process?

| Per **process** (same in every thread's file) | Per **thread** (differ between `task/<TID>/status` files) |
|-----------------------------------------------|------------------------------------------------------------|
| `Tgid`, `Threads`, all `Vm*` and `Rss*` fields, `HugetlbPages` | `Pid` (the TID), `Name`, `State`, `SigPnd`, `SigBlk`, `Cpus_allowed*`, both `*_ctxt_switches` |

`/proc/<PID>/status` shows the **main thread's** per-thread values. To inspect another thread, read `/proc/<PID>/task/<TID>/status`. Verified: `VmRSS` was identical across three threads (10.5 MB, tiny drift between reads), while `Pid` and context switches differed.

## 5. How Debugging Wizard uses it

`dw::read_mem_snapshot` (Chapter 2) reads this file and fills:

| `MemSnapshot` field | Source line |
|---------------------|-------------|
| `vm_size_kb` | `VmSize` |
| `vm_rss_kb` | `VmRSS` |
| `vm_hwm_kb` | `VmHWM` |
| `rss_anon_kb`, `rss_file_kb`, `rss_shmem_kb` | `RssAnon`, `RssFile`, `RssShmem` |
| `threads` | `Threads` |

Missing fields stay `-1` (never `0`), and lines are matched by **key name**.

## 6. Gotchas

- **Kernel threads and zombies have no `Vm*` lines.** Verified: `/proc/2/status` (`kthreadd`) has `Name:` and `Threads:` but no `VmSize` or `VmRSS`. A parser must not assume they exist.
- **The format grows.** New kernels add fields (`Kthread`, `untag_mask`, `Seccomp_filters`, `THP_enabled` appeared in our test kernel). Parse by name; ignore unknown keys.
- **`VmHWM` is approximate.** In Chapter 2 it read 76-148 kB *lower* after a memory release than a moment before, in 30 of 30 runs. Treat it as accurate to about 0.1%, not to the kilobyte.
- **Not atomic.** Counters are read at slightly different instants, so `VmRSS` may differ from the sum of the three `Rss*` fields by a few kB.
- **`VmSize` is a reservation, not usage.** An ASan build shows about 20 TiB.
- **The reader is always `R`.** `cat /proc/self/status` shows `State: R (running)` because `cat` is running while it reads.
- **Single `read()` can truncate.** The file is about 1.4 KB, but `VmRSS` sits past byte 250; a small fixed buffer silently cuts it off. Always read until EOF.

## 7. Related

- [proc-self-maps](proc-self-maps.md): the per-mapping detail behind `VmSize`.
- [proc-self-smaps-rollup](proc-self-smaps-rollup.md): a finer split of RSS (`Pss`, clean vs dirty).
- [proc-self-limits](proc-self-limits.md): the limits that produce `SigQ`'s ceiling and more.
- [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md): deep-dive on the three RSS buckets.
