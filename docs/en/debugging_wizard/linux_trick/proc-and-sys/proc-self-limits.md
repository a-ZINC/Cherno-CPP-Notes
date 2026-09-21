---
topic: proc-and-sys
tags: /proc/self/limits, limits, rlimit, RLIMIT_NOFILE, RLIMIT_CORE, RLIMIT_STACK, RLIMIT_AS, ulimit, prlimit
related: ../process-resources/ulimit.md, proc-self-status.md, proc-self-fd.md
updated: 2026-09-21
---
# `/proc/self/limits`

## 1. What it is

The resource limits a process is **actually running with right now**: one row per limit, with its **soft** and **hard** value. This is the file that answers "why does my `ulimit` change not work?" because it shows what the kernel really applies to *this* process, not what your shell was asked to do.

| Path | Notes |
|------|-------|
| `/proc/self/limits` | The reading process |
| `/proc/<PID>/limits` | Another process (readable without special privilege) |

**Where the data comes from:** the process's `rlimit` array in the kernel (set by `setrlimit`, inherited from the parent) → formatted by procfs on each read. `ulimit` (see [ulimit](../process-resources/ulimit.md)) and `prlimit` are the tools that change these values.

## 2. Commands you actually use

```bash
cat /proc/self/limits
grep -E 'core|open files|stack' /proc/<PID>/limits    # check a running process
prlimit --pid <PID>                                   # same, plus edit with --nofile=... etc.
```

## 3. Columns

| Column | Meaning |
|--------|---------|
| `Limit` | Name of the limit (text, contains spaces) |
| `Soft Limit` | The value the kernel **enforces** |
| `Hard Limit` | The ceiling the soft limit may be raised to |
| `Units` | `bytes`, `seconds`, `files`, ... (blank for nice and real-time priority) |

`unlimited` means no limit. When parsing, split by **column position or by two-or-more spaces**, since limit names contain single spaces.

## 4. Row reference

*Values are from a real run. They differ between systems.*

| Row (`Limit`) | Kernel name | `ulimit` flag | Sample soft / hard | What it limits |
|---------------|-------------|---------------|--------------------|----------------|
| `Max cpu time` | `RLIMIT_CPU` | `-t` | unlimited / unlimited | CPU seconds; the process gets `SIGXCPU`, then `SIGKILL` |
| `Max file size` | `RLIMIT_FSIZE` | `-f` | unlimited / unlimited | Largest file the process may create |
| `Max data size` | `RLIMIT_DATA` | `-d` | unlimited / unlimited | Data segment: heap and private writable mappings |
| `Max stack size` | `RLIMIT_STACK` | `-s` | 8388608 / unlimited | Main-thread stack growth (8 MiB by default) |
| `Max core file size` | `RLIMIT_CORE` | `-c` | **0** / unlimited | Core dump size; **0 disables core dumps** |
| `Max resident set` | `RLIMIT_RSS` | `-m` | unlimited / unlimited | **Not enforced on modern Linux**; do not rely on it |
| `Max processes` | `RLIMIT_NPROC` | `-u` | 15931 / 15931 | Processes and threads for the **user** |
| `Max open files` | `RLIMIT_NOFILE` | `-n` | 20000 / 20000 | Open file descriptors; exceeding gives `EMFILE` |
| `Max locked memory` | `RLIMIT_MEMLOCK` | `-l` | 8388608 / 8388608 | Memory the process may `mlock` |
| `Max address space` | `RLIMIT_AS` | `-v` | unlimited / unlimited | Total **virtual** size (VSZ), not RSS |
| `Max file locks` | `RLIMIT_LOCKS` | `-x` | unlimited / unlimited | Legacy limit on file locks |
| `Max pending signals` | `RLIMIT_SIGPENDING` | `-i` | 15931 / 15931 | Queued signals (matches `SigQ` in `status`) |
| `Max msgqueue size` | `RLIMIT_MSGQUEUE` | `-q` | 819200 / 819200 | Bytes in POSIX message queues |
| `Max nice priority` | `RLIMIT_NICE` | `-e` | 0 / 0 | How far an unprivileged process may **raise its priority** (lower its nice value): down to `20 − limit`. `0` means it may not raise its priority at all |
| `Max realtime priority` | `RLIMIT_RTPRIO` | `-r` | 0 / 0 | Highest real-time scheduling priority allowed |
| `Max realtime timeout` | `RLIMIT_RTTIME` | `-R` | unlimited / unlimited | CPU microseconds a real-time task may run without blocking |

## 5. Cross-checks you can do

- **`Max pending signals` = the `/` limit in `SigQ` of [status](proc-self-status.md).** Verified: both were 15931.
- **`Max open files` versus reality:** compare with the number of entries in [`/proc/self/fd`](proc-self-fd.md). The kernel returns `EMFILE` when they meet.
- **`Max core file size` = 0** is why crashes leave no core file by default (see [core-dumps](../crash-debugging/core-dumps.md)).

## 6. How Debugging Wizard will use it

Phase 2's process observer will read this file to report each process's headroom (for example open files used versus `Max open files`). It must treat the text `unlimited` as a special value (not a number) and treat permission or missing-file errors as normal.

## 7. Gotchas

- **Inherited at start.** A process keeps the limits it started with; changing `ulimit` in a shell affects only programs launched *afterwards*. This file shows the truth.
- **Soft above hard is rejected.** Verified: raising a soft limit above the hard limit returns an error.
- **`Max address space` breaks sanitizers.** An ASan build under `ulimit -v 1000000` fails to start (`ReserveShadowMemoryRange failed`), since it reserves about 20 TiB of address space. Limit memory with cgroups instead when you need a real cap.
- **Units differ per row.** Bytes, seconds, counts, microseconds. Read the `Units` column before comparing.
- **`Max processes` is per user**, shared by all the user's processes and threads.

## 8. Related

- [ulimit](../process-resources/ulimit.md): the command that changes these values.
- [proc-self-status](proc-self-status.md): `SigQ`, `FDSize`.
- [proc-self-fd](proc-self-fd.md): the count that `Max open files` caps.
