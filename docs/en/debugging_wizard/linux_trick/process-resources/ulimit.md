---
topic: process-resources
tags: ulimit, rlimit, limits, nofile, EMFILE, nproc, stack, core, prlimit, setrlimit
related: proc-and-sys/proc-self-limits.md, crash-debugging/core-dumps.md, memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# `ulimit`: Per-Process Resource Limits

## 1. What it is

`ulimit` is a **shell built-in** that reads and sets resource limits (the kernel calls them *rlimits*) for the current shell and **every process started from it afterwards**. The shell only asks; the **kernel enforces** (through the `setrlimit` system call).

## 2. Soft vs hard limits

| Limit | Flag | Meaning | Who can change it |
|-------|------|---------|-------------------|
| **Soft** | `-S` | The ceiling the kernel actually enforces now | Any user, up to the hard limit |
| **Hard** | `-H` | The maximum the soft limit may be raised to | Non-root can only **lower** it (one-way); root can raise it |

Rules: soft ≤ hard always. Without `-S` or `-H`, `ulimit` sets **both** and shows the soft value.

## 3. Flags you will use

| Flag | Resource | When you need it |
|------|----------|------------------|
| `-a` | Show all limits | First thing to run |
| `-c` | Core dump file size | `ulimit -c unlimited` to allow crash dumps |
| `-n` | Open file descriptors | Servers; also to trigger `EMFILE` on purpose |
| `-u` | Max processes/threads | Stops fork bombs; counted **per user**, not per shell |
| `-s` | Stack size (kB) | Deep recursion overflowing the stack |
| `-v` | Virtual memory (address space) | Cap runaway allocation. It limits **VSZ, not RSS** (see gotchas) |
| `-t` | CPU time (seconds) | Kill tasks that run too long |

## 4. Commands you actually use

```bash
ulimit -a                      # everything
ulimit -n                      # soft limit for open files
ulimit -Hn                     # hard limit for open files
ulimit -c unlimited            # allow core dumps in THIS shell

( ulimit -n 64; ./fd_leak )    # subshell trick: the limit applies only inside the parentheses
```

The **subshell trick** is the safest way to experiment: when the parentheses end, your real shell is untouched. (This is how Chapter 1 forced `fd_leak` to fail quickly.)

## 5. Where the truth lives

```bash
grep -E 'core|open files' /proc/self/limits       # soft and hard, per process
prlimit --pid <PID> --nofile                      # look at (or change) ANOTHER running process
```

`/proc/<PID>/limits` shows what a process is *really* running with. Use it when a setting "does not work". Every row is explained in [proc-self-limits](../proc-and-sys/proc-self-limits.md).

## 6. Making it permanent

| Situation | Where to set it |
|-----------|-----------------|
| Login sessions (terminal, ssh) | `/etc/security/limits.conf` or `/etc/security/limits.d/*.conf` (applied by PAM) |
| A systemd service | `LimitNOFILE=`, `LimitCORE=`, ... in the unit file (limits.conf does **not** apply) |
| One command | The subshell trick above |

## 7. Gotchas

- **Per shell and inherited.** `ulimit -c unlimited` in one terminal does not affect another. Children inherit limits from their parent.
- **Old processes keep old limits.** A process started before you changed the limit is unchanged. Check `/proc/<PID>/limits`.
- **Soft above hard fails.** Verified: with hard = 20000, `ulimit -S -n 30000` returns an error.
- **`-v` limits address space (VSZ), not resident memory.** Verified: an AddressSanitizer build under `ulimit -v 1000000` fails to start with `ReserveShadowMemoryRange failed ... Perhaps you're using ulimit -v`, because ASan reserves about 20 TiB of address space it barely uses. Never use `-v` to cap "memory use" on sanitizer builds; see [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md).
- **Exhausting `-n`** gives `EMFILE` ("Too many open files"). In Chapter 1, `fd_leak` failed at iteration 61 with a limit of 64 (descriptors 0, 1, 2 already used). Even diagnostic tools can fail once descriptors run out.
- **`-u` counts a user's total processes and threads**, so a low value can break unrelated programs of the same user.

## 8. Related

- [core-dumps](../crash-debugging/core-dumps.md): `ulimit -c` is the switch that allows a core file.
- [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md): what "memory" means when limiting it.
