---
topic: crash-debugging
tags: core dump, core, core_pattern, SIGSEGV, SIGABRT, gdb, backtrace, coredumpctl, apport, post-mortem
related: process-resources/ulimit.md, proc-and-sys/proc-sys-kernel-core-pattern.md, proc-and-sys/proc-self-limits.md, memory/rss-anon-file-shmem.md
updated: 2026-09-21
---
# Core Dumps and Post-Mortem Debugging with GDB

## 1. What it is

A **core dump** is a snapshot of a process's memory and CPU registers, written to disk at the moment a fatal signal kills it. You open it later in GDB and see *where* the program died and *what state it was in*, without re-running it.

**It contains:** registers (instruction pointer `RIP`, stack pointer `RSP`, frame pointer `RBP`, and the rest), the process's **anonymous memory** (stack, heap, data), the list of mapped libraries, every thread's state, and the signal that killed it. By default the *contents* of file-backed mappings (the library code itself) are **not** written, since GDB can reload them from the files on disk. This is controlled by `/proc/<PID>/coredump_filter`, and it is one reason the executable and libraries must match the ones that crashed.

**Signals that produce a core dump:**

| Signal | Name | Typical cause |
|--------|------|---------------|
| `SIGSEGV` | Segmentation fault | Null pointer, invalid address, bad permissions |
| `SIGABRT` | Abort | `abort()`, failed `assert()`, uncaught C++ exception |
| `SIGFPE` | Arithmetic error | Integer division by zero |
| `SIGBUS` | Bus error | Unaligned or impossible memory access |
| `SIGILL` | Illegal instruction | Corrupt or unsupported instruction |

## 2. Try it: a program that really crashes

```cpp
// crash.cpp: write through a null pointer -> SIGSEGV
#include <cstdio>

int main() {
    std::printf("about to crash\n");
    std::fflush(stdout);
    volatile int* p = nullptr;
    *p = 42;
    return 0;
}
```

```bash
g++ -g -O0 crash.cpp -o crash     # -g keeps symbols, -O0 keeps the code readable
ulimit -c unlimited               # allow a core file in THIS shell
./crash                           # prints: Segmentation fault (core dumped)
echo $?                           # 139 = 128 + signal 11
ls -l core*                       # the core file (in the current directory)
gdb ./crash core                  # open the corpse
```

**Verified:** exit status 139, and a core file of about 448 KiB with permissions `-rw-------`.

> **Not every bug crashes.** In our Chapter 1 bug zoo, the plain `bugs overflow` and `bugs uaf` exit normally with status 0: memory bugs often *appear to work*. Only a fault the kernel can catch (a null pointer, a wild address) produces a core dump. AddressSanitizer catches the others.

## 3. Where does the core file go? (`core_pattern`)

Full reference: [proc-sys-kernel-core-pattern](../proc-and-sys/proc-sys-kernel-core-pattern.md).

```bash
cat /proc/sys/kernel/core_pattern
```

| Value starts with | Meaning |
|-------------------|---------|
| a plain name, for example `core` | The kernel writes a file **in the crashing process's current directory** |
| `\|` (a pipe), for example `\|/usr/share/apport/apport ...` | The kernel **pipes** the dump to a handler program (Apport on Ubuntu, `systemd-coredump` elsewhere). No local file appears. |

Common pattern tokens: `%e` executable name, `%p` process ID, `%t` timestamp, `%s` signal number. Example: `core.%e.%p`.

**Before changing it, save the old value** so you can restore it:

```bash
cat /proc/sys/kernel/core_pattern                     # write this down
sudo sysctl -w kernel.core_pattern=core.%e.%p         # local files
sudo sysctl -w kernel.core_pattern='<the old value>'  # restore later
```

## 4. Analyzing the core in GDB

```bash
gdb ./crash core              # executable FIRST, core second
```

| Command | What it does |
|---------|--------------|
| `bt` | **Backtrace**: the chain of calls that led to the crash (start here) |
| `bt full` | Backtrace **with local variables** of every frame |
| `frame N` (or `up` / `down`) | Move to stack frame N to inspect it |
| `info locals` | Local variables of the current frame |
| `info args` | Arguments of the current function |
| `print p` or `p *p` | Show a variable or what a pointer points to |
| `list` | Source lines around the current position (needs `-g`) |
| `info registers` | CPU registers at the moment of death |
| `thread apply all bt` | Backtrace of **every** thread (essential for multithreaded crashes) |

Reading order that works: `bt` → `frame N` for the first frame in *your* code → `info locals` → `print` the suspicious pointer.

## 5. If there is no core file

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Segmentation fault` with no `(core dumped)` | `ulimit -c` is `0` (the default) | `ulimit -c unlimited` in **that** shell |
| `(core dumped)` but no file | `core_pattern` pipes to Apport or `systemd-coredump` | `coredumpctl gdb <name>` (systemd), or set a plain `core_pattern` |
| File appears somewhere unexpected | Written to the process's **current directory** at crash time | Use an absolute pattern such as `/tmp/core.%e.%p` |
| GDB shows `??` instead of function names | Binary built without `-g`, or a different binary than the one that crashed | Rebuild with `-g`; use the **exact** binary that crashed |

**GDB stuck on `--Type <RET> for more--`:** press `c` (finish output) or `q` (quit the listing), and turn the pager off with `set pagination off`.

## 6. Making it permanent

```bash
# Local core files for every boot (a drop-in file is tidier than editing sysctl.conf)
echo "kernel.core_pattern = core.%e.%p" | sudo tee /etc/sysctl.d/60-core.conf
sudo sysctl --system
```

If it reverts after a reboot, a crash-handler service (Apport) is setting it again; then use `coredumpctl` instead, or disable that service.

GDB settings (`~/.gdbinit`):

```bash
echo "set pagination off"        >> ~/.gdbinit
echo "set debuginfod enabled on" >> ~/.gdbinit    # fetch debug symbols for system libraries
```

## 7. Gotchas

- **`ulimit -c` is per shell.** A limit set in one terminal does nothing in another. For a systemd service use `LimitCORE=` (see [ulimit](../process-resources/ulimit.md)).
- **AddressSanitizer builds do not leave a core file.** Verified: an ASan build handles the `SIGSEGV` itself, prints its own report, and exits with status 1. Use the plain `debug` build for core dumps.
- **Cores can hold secrets** (passwords, keys, tokens in memory) and can be as large as the process's memory. Do not share them casually.
- **Debuggers and sanitizers do not mix well.** LeakSanitizer does not work under `ptrace`, so under GDB run sanitizer builds with `ASAN_OPTIONS=detect_leaks=0`.

## 8. Related

- [ulimit](../process-resources/ulimit.md): the `-c` switch that permits the dump.
- [rss-anon-file-shmem](../memory/rss-anon-file-shmem.md): a core's size roughly follows the process's **anonymous** memory (`RssAnon`), which is why a leaking or memory-hungry program produces a large core.
