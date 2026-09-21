---
topic: proc-and-sys
tags: /proc/self/fd, fd, fdinfo, file descriptor, EMFILE, pipe, socket, anon_inode, lsof, descriptor leak
related: proc-self-limits.md, proc-self-status.md, ../process-resources/ulimit.md
updated: 2026-09-21
---
# `/proc/self/fd` and `/proc/self/fdinfo`

## 1. What it is

A **directory with one entry per open file descriptor** of the process. Each entry is a **symbolic link** whose name is the descriptor number and whose target says what it refers to. It is the kernel's view of the process's descriptor table.

| Path | Notes |
|------|-------|
| `/proc/self/fd/` | The reading process |
| `/proc/<PID>/fd/` | Another process. **Access-checked**: same user or `CAP_SYS_PTRACE`, otherwise `Permission denied` |
| `/proc/<PID>/fdinfo/<N>` | Extra details about descriptor `N` (position, flags, ...) |

**Where the data comes from:** the process's descriptor table (`files_struct`) in the kernel, listed on demand by procfs. An `int fd` in your program is only an **index** into that table; the real state (file offset, flags) lives in a kernel file object.

## 2. Commands you actually use

```bash
ls -l /proc/self/fd                 # what am I holding open?
ls /proc/<PID>/fd | wc -l           # how many descriptors does that process hold?
cat /proc/<PID>/fdinfo/3            # details of descriptor 3
lsof -p <PID>                       # the friendly, slower tool built on the same data
```

## 3. Reading the listing (real output)

```
lr-x------ 1 root root 64 ... 0 -> pipe:[781]
l-wx------ 1 root root 64 ... 1 -> pipe:[782]
l-wx------ 1 root root 64 ... 2 -> pipe:[783]
lr-x------ 1 root root 64 ... 3 -> /proc/82/fd
```

- **The name** (`0`, `1`, `2`, `3`) is the descriptor number. 0, 1, 2 are stdin, stdout, stderr.
- **The link's permission bits show how it was opened:** `lr-x` = opened for **reading**, `l-wx` = opened for **writing**.
- **The target** tells you what it is:

| Target looks like | Meaning |
|-------------------|---------|
| `/path/to/file` | A regular file or directory |
| `/dev/pts/0`, `/dev/null` | A terminal, a device |
| `pipe:[781]` | A pipe (the number is its inode; two ends share it) |
| `socket:[12345]` | A network or Unix socket |
| `anon_inode:[eventpoll]` (and `[eventfd]`, `[timerfd]`, ...) | A kernel object with no file: `epoll`, `eventfd`, timers |
| `... (deleted)` | The file was deleted but is still open |

## 4. `fdinfo/<N>`

Real output for a pipe descriptor:

```
pos:    0
flags:  00
mnt_id: 16
ino:    781
```

| Field | Meaning |
|-------|---------|
| `pos` | Current file offset |
| `flags` | The `open()` flags, in **octal** (`02` = read/write, and so on) |
| `mnt_id` | The mount the file lives on |
| `ino` | The inode number |

Special descriptors (`epoll`, `eventfd`, ...) add their own extra lines.

## 5. Counting descriptors (what our tools do)

The **number of entries** = the number of open descriptors. Chapter 1's `observe_pid` counts them from *outside* the target process, so the observer's own directory handle appears in the observer's table, not in the target's.

## 6. Gotchas

- **`ls /proc/self/fd` shows its own directory handle.** Verified: `fd 3 -> /proc/82/fd` in the listing is the descriptor `ls` opened to read the directory. Count the entries **of another PID** to avoid the extra one.
- **`FDSize` in [status](proc-self-status.md) is not this count.** `FDSize` is the number of *slots allocated* in the table; the entries here are the descriptors actually in use.
- **`Permission denied` is normal** for other users' processes. Treat it as an expected outcome, not a crash.
- **The limit:** the count cannot exceed `Max open files` in [limits](proc-self-limits.md). Chapter 1's `fd_leak` failed at iteration 61 with `EMFILE` under a limit of 64 (three descriptors already in use).
- **A descriptor leak is invisible to ASan and LSan** (they track heap memory, not kernel descriptors). Only watching this directory over time, or RAII (`dw::Fd`), catches it.
- **Processes appear and vanish.** Between listing and reading a link, a descriptor can close; handle `ENOENT` gracefully.

## 7. Related

- [proc-self-limits](proc-self-limits.md): the ceiling on this count.
- [proc-self-status](proc-self-status.md): `FDSize`.
- [ulimit](../process-resources/ulimit.md): how to lower the limit to reproduce `EMFILE` safely.
- Chapter 1 of the project (`phase-0-ch1.md`): the `fd_leak` experiment and `dw::Fd`.
