---
topic: proc-and-sys
tags: /proc/self/maps, maps, VMA, mapping, address space, heap, stack, vdso, ASLR, permissions, anonymous, file-backed
related: proc-self-status.md, proc-self-smaps-rollup.md, ../memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# `/proc/self/maps`

## 1. What it is

The list of a process's **virtual memory areas (VMAs)**: every address range the process is *allowed* to use, one line each. A VMA is a **promise** ("this range is valid, with these permissions, backed by this file or by nothing"). It does **not** say how much of it is actually in RAM (that is the page tables and `VmRSS`).

| Path | Notes |
|------|-------|
| `/proc/self/maps` | The reading process |
| `/proc/<PID>/maps` | Another process. **Access-checked** (needs permission to inspect that process: same user, or `CAP_SYS_PTRACE`), unlike `status` |

**Where the data comes from:** the process's list of VMAs in `mm_struct` → formatted as text by procfs on each `read()`.

## 2. Commands you actually use

```bash
cat /proc/self/maps                       # the reader's own map
cat /proc/<PID>/maps                      # another process (permission needed)
grep -E '\[heap\]|\[stack\]' /proc/<PID>/maps
pmap -x <PID>                             # the same information, summarized, with RSS per line
```

## 3. Reading one line

```
7f1fa5028000-7f1fa51b0000 r-xp 00028000 fe:00 191494   /usr/lib/x86_64-linux-gnu/libc.so.6
```

| Field | Example | Meaning |
|-------|---------|---------|
| Address range | `7f1fa5028000-7f1fa51b0000` | Start and end in hex, no `0x`. The end is **exclusive**. Size = end − start (here 1,568 kB) |
| Permissions | `r-xp` | Four flags, see below |
| Offset | `00028000` | Where in the backing file this mapping starts (hex bytes). `00000000` for anonymous |
| Device | `fe:00` | `major:minor` of the device holding the file. `00:00` means no device |
| Inode | `191494` | The file's inode. **`0` means no file: an anonymous mapping** |
| Path | `.../libc.so.6` | The file, a special name, or **empty** (anonymous) |

**Permissions:**

| Position | Letters | Meaning |
|----------|---------|---------|
| 1 | `r` / `-` | readable |
| 2 | `w` / `-` | writable |
| 3 | `x` / `-` | executable |
| 4 | `p` / `s` | **p**rivate (copy-on-write) / **s**hared with other processes |

Normally no region is both `w` and `x` ("W^X"). The **MMU** enforces these flags; a violation raises a fault and the kernel sends `SIGSEGV`.

## 4. Special names and how to classify a line

| Path column | What it is |
|-------------|------------|
| `/usr/bin/prog`, `/usr/lib/.../libc.so.6` | **File-backed**: the program or a library segment, or a file you `mmap`'d |
| *(empty)* | **Anonymous** memory: `malloc` of large blocks (`mmap`), thread stacks, the zero-filled `.bss` part of a program or library |
| `[heap]` | The `brk` heap (small `malloc`s) |
| `[stack]` | The main thread's stack |
| `[vdso]`, `[vvar]`, `[vvar_vclock]` | Kernel-provided pages that make some syscalls (like reading the time) fast. `[vvar_vclock]` appears on newer kernels |
| `[vsyscall]` | A legacy fixed page at a high address |
| `... (deleted)` | The mapped file was deleted while still mapped |

One executable appears as **several lines**: the kernel maps each ELF segment separately with its own permissions (read-only headers, `r-xp` code, `r--p` constants, `rw-p` data).

## 5. Worked example: what each line of a small program shows

| Line pattern | Meaning |
|--------------|---------|
| `r-xp` on the program | Its code |
| `rw-p` on the program | Its writable data |
| `rw-p ... [heap]` | Small `malloc`s |
| `r-xp` on `libc.so.6` | The C library's code (about 1.5 MB, larger than a tiny program's own code) |
| `rw-p` with no path | Anonymous private memory |
| `rw-p ... [stack]` | The stack (about 132 kB here) |

## 6. How Debugging Wizard uses it

`dw::read_maps` (Chapter 2) parses each line into a `MapEntry` (start, end, perms, offset, inode, path). `dw::find_map(maps, addr)` answers "which mapping contains this address?", which is how `heap_vs_mmap` showed that small `malloc`s land in `[heap]` and large ones in anonymous mappings.

Parsing rule: everything after the inode is the path, **including spaces**. An unexpected line makes `read_maps` return `false` (fail loudly, never guess).

## 7. Gotchas

- **A line is a VMA, not an allocation.** The kernel **merges** neighbouring anonymous VMAs with identical permissions. A 200 KB `malloc` showed up in a 224 kB region.
- **ASLR randomizes addresses** on every `exec`. Never compare raw addresses between runs; compare sizes and roles.
- **Sum of ranges ≈ `VmSize`.** Verified three ways (maps sum, `VmSize`, `pmap`): they agree apart from a constant **4 kB**, the `[vsyscall]` page (a strong hypothesis: it appears in `maps` but not in `VmSize`).
- **Anonymous lines end with trailing spaces** in the text; a parser must not assume a path exists.
- **Not a consistent snapshot for big processes.** Reading in several chunks while memory changes can mix old and new state.
- **Permission denied** on other users' processes is normal, not a failure of your tool.
- **Address size is not usage.** A huge line (for example the shadow memory an ASan build reserves, which pushes `VmSize` to about 20 TiB) may have almost nothing resident. For RSS per mapping use `smaps` or `pmap -x`.

## 8. Related

- [proc-self-status](proc-self-status.md): `VmSize` is the total of these lines.
- [proc-self-smaps-rollup](proc-self-smaps-rollup.md): RSS and `Pss` totals across all these mappings.
- Chapter 2 of the project (`phase-0-ch2.md`): demand paging, and `maps_summary`.
