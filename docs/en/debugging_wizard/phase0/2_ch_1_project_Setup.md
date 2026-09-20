# Debugging Wizard — Phase 0, Chapter 1
# Project Setup, Tooling, and Safety (VS Code, CMake, Sanitizers, RAII)

> **How to use this note**
> 1. Read the short concept at the start of each step.
> 2. Answer the 🧠 THINK / 🔬 PREDICT questions **on paper first**.
> 3. Only then open the **✅ Solution** block.
> 4. Run the experiment and record **your** numbers. If reality disagrees with your prediction, that is the most valuable moment in the chapter.
> 5. Ask me follow-up questions any time. Questions and answers go in the **Clarification Log** at the end.
>
> **About "reference data" boxes.** Some steps include results I measured while testing this repo. They came from a *different machine* (a sandbox with GCC 13, not your Acer with GCC 15). They show what to expect in shape, not in exact numbers. Your numbers are the ones that matter.
>
> **Read this note in VS Code** with the *Markdown Preview Mermaid Support* extension (already in the repo's recommended extensions) so the diagrams render. Collapsible solution blocks work in the built-in preview.

---

## 0. Chapter Overview

### 0.1 What You Will Learn

By the end of this chapter you can:

- **Explain** the path from source file to ELF file to running process.
- **Configure** VS Code with clangd, CMake Tools, and gdb, and use them to complete a build → diagnose → debug cycle.
- **Build** one repo reproducibly with CMake presets (Debug, RelWithDebInfo, sanitizers).
- **Predict and measure** VSZ vs RSS growth for a leaking program.
- **Identify** a leak, heap overflow, use-after-free, and signed overflow with ASan, LSan, UBSan, and Valgrind.
- **Implement** an RAII file-descriptor wrapper (`dw::Fd`) and **test** it.
- **Measure** the runtime and memory cost of sanitizers, changing one variable at a time.

### 0.2 Prerequisites

Ubuntu (native), GCC, CMake, and:

```bash
sudo apt install build-essential cmake gdb strace valgrind time git \
                 clangd clang-format clang-tidy
```

If `clangd` is not found, run `apt search clangd`; on some releases the package is versioned (for example `clangd-20`).

### 0.3 Why This Matters to Debugging Wizard

Debugging Wizard is a **long-running** program that will diagnose other programs. A leak of 4 KB per sample at 10 samples per second costs about 140 MB per hour. A tool that becomes the problem it diagnoses is worthless. This chapter builds the workshop: a reproducible build, an editor that finds mistakes early, and safety nets that catch what remains.

### 0.4 Step Map

| Step | Focus |
|------|-------|
| 1.1 | Source → ELF → process |
| 1.2 | Your machine and the repository |
| 1.3 | CMake and presets |
| 1.4 | VS Code as a professional C++ environment |
| 1.5 | First program: observe it with `strace` and `/proc/self` |
| 1.6 | The leak experiment: VSZ vs RSS |
| 1.7 | Sanitizers: how they work |
| 1.8 | Break it: the bug zoo |
| 1.9 | RAII: the forgotten `close()` and `dw::Fd` |
| 1.10 | Measure the cost of sanitizers |
| 1.11 | Explain-it-back, knowledge check, completion |

---

## Step 1.1 — From Source to Running Process

```mermaid
flowchart TD
    A[main.cpp source] --> B[Compiler: machine code plus symbols]
    B --> C[Linker: ELF executable on disk]
    C --> D[Shell: fork then execve]
    D --> E[Kernel loads ELF: creates address space]
    E --> F[Dynamic loader ld.so maps libc, libstdc++]
    F --> G[CPU executes main]
    G --> H[exit_group syscall: kernel frees address space]
```

**How to read this diagram:**

| Element | Meaning |
|---------|---------|
| Compiler → Linker | User-space tools that turn text into instructions and stitch pieces together. No kernel involvement yet. |
| ELF on disk | A file with **segments** (code, data) and headers describing how to map them. It is not a process. |
| `fork` + `execve` | The **shell initiates** this. `execve` asks the kernel to replace the process image with the new program. |
| Kernel loads ELF | Creates **virtual memory mappings** (`mm_struct`, VMAs). Mostly nothing is copied into RAM yet. Pages load lazily on first access (page faults). |
| `ld.so` | For dynamically linked binaries, the kernel also maps the loader named in the ELF's `PT_INTERP`, which then maps shared libraries. |
| CPU executes | The MMU translates virtual addresses through page tables the kernel built. |
| `exit_group` | The process ends and the kernel tears down its address space and closes leftover file descriptors. |

**Where blocking and latency can appear:** disk reads on first page fault (cold cache), library loading, and page faults on every first touch of memory.

**How to observe each stage:** `readelf -h`, `strace`, `/proc/<pid>/maps`, `/proc/<pid>/status`.

### 🧠 THINK

1. The compiler accepted your program. What did it verify, and what did it *not* verify?
2. When `execve` finishes, is the whole executable already in RAM? Why or why not?
3. If your program never calls `free()`, who reclaims its memory when it exits? How does that change for a program that runs six hours?

<details>
<summary>✅ Solution</summary>

1. It verified **syntax, types, and declarations**. It did not verify *runtime behavior*: pointer validity at the moment of use, whether every allocation is freed, whether an index is in range, or whether arithmetic overflows. Those are properties of a specific execution, not of the source text.
2. No. The kernel creates **mappings**, not copies. Pages are faulted in from the page cache or disk on demand. That is why a large binary starts quickly and why RSS is smaller than the size of the binary plus libraries.
3. At exit the **kernel** reclaims all of the process's memory regardless of leaks, so leaks are harmless in a 10 ms program. A long-running tool never exits, so only the program itself can return memory. That is why leaks matter so much for Debugging Wizard.

</details>

---

## Step 1.2 — Your Machine and the Repository

### Your environment (recorded from your terminal)

| Item | Value |
|------|-------|
| Machine | Acer Aspire A715-75G |
| OS | Ubuntu (native) |
| Kernel | `7.0.0-31-generic` |
| Compiler | g++ (Ubuntu 15.2.0-16ubuntu1) 15.2.0 |
| CMake | 4.2.3 |
| Architecture | x86_64 |

A note from your terminal: GCC takes long options with **two** dashes. `g++ -version` failed with "unrecognized command-line option", while `g++ --version` worked.

### 🧪 EXPERIMENT: complete the hardware record

Benchmarks are only interpretable next to the machine that produced them. Run these and paste the results into `docs/environment.md`:

```bash
lscpu | grep -E 'Model name|^CPU\(s\)|Thread|L1d|L2|L3'
free -h
lsblk -d -o NAME,ROTA,SIZE,MODEL
cat /proc/sys/kernel/perf_event_paranoid
```

### 🧠 THINK

1. Why record the kernel version and CPU model alongside every measurement?
2. `lsblk` shows a `ROTA` column (1 = spinning disk, 0 = SSD). Why will this matter in Phase 5?
3. `perf_event_paranoid` controls who may use hardware counters. Which future phase depends on it?

<details>
<summary>✅ Solution</summary>

1. Results depend on the CPU (cache sizes, core count, frequency scaling), the kernel (which `/proc` fields exist, scheduler behavior), and the compiler version. A number without its environment cannot be compared or reproduced.
2. Disk latency and throughput differ by orders of magnitude between spinning disks, SATA SSDs, and NVMe. Our disk metrics and workloads must be interpreted against the device.
3. **Phase 7** (`perf_event_open` and hardware counters). Higher values restrict unprivileged use, and we will learn to read and adjust that setting safely when we get there.

</details>

### The repository (one repo, not chapter folders)

```
debugging-wizard/
├── CMakeLists.txt          # top-level build description
├── CMakePresets.json       # named configurations: debug, asan, rel, asan-rel
├── include/dw/             # public headers of the real tool (fd.hpp lives here)
├── src/                    # the real tool (empty until code has earned its place)
├── lab/                    # throwaway experiments and deliberately broken programs
├── tests/                  # automated checks (ctest)
├── bench/                  # recorded benchmark results
├── scripts/                # observe_leak.sh, bench_sanitizers.sh
├── docs/
│   ├── environment.md      # your machine record
│   └── notes/phase-0-ch1.md  # this note (later: phase-0-ch2.md, ...)
└── .vscode/                # editor configuration (see Step 1.4)
```

### 🧠 THINK

Why separate `lab/` from `src/`?

<details>
<summary>✅ Solution</summary>

`lab/` holds experiments where **broken code is the point**: leaks, overflows, naive readers. Mixing that with the real tool invites shipping a deliberate bug. `src/` receives code only after it has been measured and tested, which enforces your master document's rule that complexity must be earned. A nice side effect: the git history shows *why* each piece of `src/` exists, because the naive version sits in `lab/`.

</details>

### First build

```bash
cd debugging-wizard
git init && git add -A && git commit -m "Phase 0 chapter 1: repo skeleton"
cmake --preset debug
cmake --build --preset debug -j
ctest --preset debug
```

---

## Step 1.3 — CMake and Presets

### Concept

CMake is a **build-system generator**: it reads a description of targets and options and generates the actual build files (Makefiles here). Reproducibility comes from **presets**, which give each configuration a name and its own build directory.

| Preset | Build type | Sanitizers | Used for |
|--------|-----------|------------|----------|
| `debug` | `-O0 -g` | off | gdb, Valgrind, plain-run comparisons |
| `asan` | `-O0 -g` | ASan + UBSan (LSan comes with ASan) | catching bugs |
| `rel` | `-O2 -g` | off | benchmarks |
| `asan-rel` | `-O2 -g` | ASan + UBSan | measuring sanitizer cost |

Each preset builds into `build/<preset>/`, so all four coexist and you can compare binaries side by side.

The heart of the top-level `CMakeLists.txt` is one **interface target** that carries flags to every executable:

```cmake
add_library(dw_options INTERFACE)
target_include_directories(dw_options INTERFACE ${PROJECT_SOURCE_DIR}/include)
target_compile_options(dw_options INTERFACE -Wall -Wextra -Wpedantic -Wshadow)

if(DW_SANITIZE)
  target_compile_options(dw_options INTERFACE -fsanitize=address,undefined -fno-omit-frame-pointer)
  target_link_options(dw_options INTERFACE -fsanitize=address,undefined)
endif()
```

Every executable then says `target_link_libraries(name PRIVATE dw_options)` and inherits the same rules.

### 🧠 THINK

1. What is the difference between `-O0 -g` and `-O2`? Which do you want for debugging and which for benchmarking?
2. Why do sanitizer flags appear in **both** compile and link options?
3. Why use one build directory per preset instead of reconfiguring one directory?
4. Why put shared flags in one `dw_options` target instead of repeating them per executable?

<details>
<summary>✅ Solution</summary>

1. `-O0 -g` keeps code close to your source: variables live in memory, nothing is inlined, and both gdb and sanitizer reports map cleanly to lines. `-O2` inlines, reorders, and eliminates code, which is what real performance looks like. Debug timings are misleading for benchmarks, and optimized code is confusing to step through. `-O2 -g` (`RelWithDebInfo`) gives realistic speed and still readable reports.
2. Compile flags tell the compiler to **insert checks** around memory accesses. Link flags pull in the sanitizer **runtime library** (which replaces `malloc`/`free` and implements the checks). One without the other gives a link error or no checking.
3. Switching flags in one directory forces full rebuilds and risks stale objects with mixed instrumentation. Separate directories guarantee each binary was built the way its directory name says, and let you keep four configurations at once.
4. One place to change means no drift between executables. If `fd_test` were compiled without `-Wshadow` while `bugs` had it, "the warnings are clean" would mean different things in different places.

</details>


# Deep Dive: Sanitizer Compiler & Linker Flags

## Why Sanitizer Flags Go in Both Compile and Link Options

* **Compile Step (`target_compile_options`):** Injects code instrumentation (checks for memory safety and undefined behaviors) directly into the generated assembly instructions for every memory access. Without this, code compiles normally.
* **Link Step (`target_link_options`):** Links the required sanitizer runtime library (`libasan`, `libubsan`, etc.) and wraps standard functions like `malloc` and `free`. Omitting this leads to `undefined reference` linker errors.

---

## What `-fno-omit-frame-pointer` Does

* By default, optimized builds (`-O2` / `-O3`) omit the frame pointer (`rbp` on x86_64) to free up a general-purpose register.
* `-fno-omit-frame-pointer` forces the compiler to keep the frame pointer locked.
* **Why it matters:** When a sanitizer detects a crash, it needs to instantly unwind the call stack to print an accurate backtrace. Keeping the frame pointer makes stack-walking reliable, fast, and resilient even when debug symbols are missing or stripped.

---

## What Bugs Do Sanitizers Catch?

Your configuration (`-fsanitize=address,undefined`) activates **AddressSanitizer (ASan)**, **LeakSanitizer (LSan)**, and **UndefinedBehaviorSanitizer (UBSan)**:

### AddressSanitizer (ASan) & LeakSanitizer (LSan)

* **Buffer Overflows / Underflows:** Accessing heap, stack, or global arrays out of bounds.
* **Use-After-Free:** Reading or writing to memory after `free()` or `delete`.
* **Use-After-Scope:** Accessing local stack variables outside their defined code block.
* **Double / Invalid Free:** Freeing the same pointer twice or passing invalid pointers.
* **Memory Leaks:** Unreachable dynamic memory (`malloc`/`new`) that was never deallocated before program exit.

### UndefinedBehaviorSanitizer (UBSan)

* **Integer Overflows:** Signed arithmetic that wraps around or overflows limits.
* **Division by Zero:** Dividing integers by zero.
* **Null Pointer Dereferences:** Reading/writing through a null pointer.
* **Shift Out of Bounds:** Shifting by negative numbers or values exceeding type width.
* **Invalid Enums / Misaligned Pointers:** Casting illegal integer values to enums or accessing unaligned memory.

---

## Step 1.4 — VS Code as a Professional C++ Environment

### The decision

Visual Studio itself does not exist on Linux. The three serious options:

| Option | Code intelligence | Strengths | Trade-offs |
|--------|-------------------|-----------|------------|
| **VS Code + clangd + CMake Tools + gdb** (recommended, configured in this repo) | clangd (LLVM) reads `compile_commands.json`, so it sees the *exact* flags of every file | Fast, accurate, live `clang-tidy` diagnostics, free, lightweight, works over SSH | You assemble it from extensions (this repo does that for you) |
| VS Code + Microsoft C/C++ (cpptools) alone | Microsoft's IntelliSense engine | Closest to the "Visual Studio feel", easy start | Less accurate on complex CMake and template code, no `clang-tidy` integration |
| **CLion** (JetBrains) | Own engine, deep CMake integration | Best all-in-one C++ IDE, strong refactoring and debugger UI | Heavier; check JetBrains' current licence terms for non-commercial use |

**Recommendation:** VS Code with clangd as the language engine and the Microsoft C/C++ extension used **only as the gdb debugger front end**. This is what many professional C++ developers on Linux run, and everything is already configured in `.vscode/`. If you later want a heavier IDE, CLion opens the same CMake project without changes.

### How the pieces fit

```mermaid
flowchart LR
    P[CMakePresets.json] --> CT[CMake Tools extension]
    CT -->|configure and build| B[build/preset/]
    B -->|compile_commands.json| CC[clangd language server]
    CC -->|completion, errors, go-to-definition, clang-tidy| E[Editor]
    CT -->|selected target path| L[launch.json]
    L --> DBG[cpptools debug adapter]
    DBG -->|ptrace| G[gdb]
    G -->|controls| PROC[Your process]
```

**How to read this diagram:**

| Arrow | Meaning |
|-------|---------|
| Presets → CMake Tools | You pick a preset in the status bar. The extension runs `cmake --preset ...` for you. |
| Build → clangd | CMake writes `compile_commands.json` listing the exact compiler command for each file. clangd parses your code with those flags. |
| clangd → Editor | Live diagnostics, completion, hover types, go-to-definition. This replaces "IntelliSense". |
| CMake Tools → launch.json | `${command:cmake.launchTargetPath}` resolves to the executable currently selected in the status bar. |
| Debug adapter → gdb → process | gdb controls your process through the `ptrace` system call. |

### Setup

Install VS Code from Microsoft's official `.deb` or apt repository, or with `sudo snap install code --classic`. Then:

```bash
cd ~/debugging-wizard
code .
```

VS Code will offer the **recommended extensions** from `.vscode/extensions.json`: accept them. Or install by hand:

```bash
code --install-extension ms-vscode.cmake-tools
code --install-extension llvm-vs-code-extensions.vscode-clangd
code --install-extension ms-vscode.cpptools
code --install-extension bierner.markdown-mermaid
```

Then:

1. **Command Palette** (`Ctrl+Shift+P`) → `CMake: Select Configure Preset` → `debug`. The extension configures the project and writes `compile_commands.json` into the repo root.
2. Open `lab/bugs.cpp`. Hover over a variable, press `F12` on a function (go to definition), `Shift+F12` (find references), `F2` (rename symbol). This is your IntelliSense.
3. Build with the status-bar **Build** button (or `CMake: Build` in the palette).
4. In the status bar choose the launch target `bugs`, then press `F5` and choose **gdb: selected target + args** (edit its `args` to `uaf`).

Debug keys: `F9` toggle breakpoint, `F10` step over, `F11` step into, `Shift+F11` step out, `F5` continue.

### 🧪 EXPERIMENT: your first debugging session

1. Set a breakpoint on the `std::printf("uaf: ...")` line in `use_after_free()`.
2. Run under the debugger (`args: ["uaf"]`).
3. When it stops, look at the **Variables** pane for `a`, then step over the print.

### 🔬 PREDICT

Before stepping: what value will `a[0]` show after `delete[] a`? We assigned `7` earlier.

<details>
<summary>✅ Solution</summary>

**Not reliably 7.** `delete[]` returns the block to glibc's allocator, which writes its own bookkeeping (free-list pointers) into the first bytes of the freed block. So `a[0]` usually shows a **garbage-looking number** rather than 7. In a plain (unsanitized) test run of this repo it printed `1439461497`. Your value will differ.

The lesson is the reason use-after-free is dangerous: the program neither crashes nor prints the value you wrote. It silently reads allocator internals. In Step 1.8 ASan catches this at the exact read instruction.

(This corrects the earlier version of this note, which said it "usually prints 7". Testing disproved that. Predictions are hypotheses, including mine.)

</details>

### 🧠 THINK

1. Why does the project use **two** C++ extensions (clangd and cpptools), and why is cpptools' IntelliSense disabled in `settings.json`?
2. How does a debugger stop a running program at a line of source code? What does the CPU do?
3. Earlier you saw `LeakSanitizer does not work under ptrace (strace, gdb, etc)`. Why would ptrace matter to a leak checker?

<details>
<summary>✅ Solution</summary>

1. clangd does the *language* work (parsing, completion, diagnostics) and cpptools provides the *debug adapter* that talks to gdb. Two engines analyzing the same file would produce duplicate or conflicting suggestions and errors, so cpptools' engine is disabled and only its debugger is used.
2. gdb attaches with `ptrace`, then overwrites the first byte of the target instruction with a special one-byte trap instruction (`int3`, opcode `0xCC`). When the CPU executes it, it raises a breakpoint exception. The kernel turns that into a `SIGTRAP` delivered to the traced process, which stops and notifies gdb. gdb restores the original byte, lets you inspect state, and resumes. Breakpoints are literally a modified instruction in memory.
3. LSan's exit-time scan itself uses `ptrace` to freeze the process's threads while it reads their stacks and registers. A process can have only **one** tracer at a time, so if gdb or strace already holds it, LSan cannot. When debugging a sanitizer build, run with `ASAN_OPTIONS=detect_leaks=0` and keep ASan's memory-error checks.

</details>

### Troubleshooting

| Symptom | Likely cause and fix |
|---------|---------------------|
| Red squiggles everywhere or "file not found" for our headers | No `compile_commands.json` yet: run `CMake: Configure` first, then `clangd: Restart language server`. |
| Bogus errors inside standard headers | clangd is older than GCC 15's libstdc++. Try a newer `clangd` package, or send me the error text and we will add a clang-based preset. |
| F5 says the program does not exist | Build first, and check the launch target in the status bar. |
| Two sets of completions | cpptools IntelliSense is still enabled. Check `.vscode/settings.json` was loaded (open the folder itself, not a parent directory). |

---

## Step 1.5 — First Program: Observe It

`lab/hello.cpp`:

```cpp
#include <cstdio>
#include <unistd.h>

int main() {
    std::printf("hello from pid %d\n", static_cast<int>(getpid()));
    return 0;
}
```

```bash
./build/debug/lab/hello
strace -c ./build/debug/lab/hello        # summary of syscalls
strace ./build/debug/lab/hello 2>&1 | head -40
```

### 🔬 PREDICT

1. Roughly how many system calls: 1, about 5, or dozens?
2. Which syscall actually prints to the terminal?
3. Which syscall ends the process?

<details>
<summary>✅ Solution</summary>

1. **Dozens** (the exact number depends on your libc and loader). The program has one `printf`, but startup involves `execve`, the dynamic loader opening and mapping libraries (`openat`, `mmap`, `read`, `close`, `mprotect`), thread-local storage setup, and heap initialization (`brk`).
2. `write(1, "hello from pid ...", n)`. `printf` fills a user-space buffer. On a terminal (line-buffered) the newline triggers the flush. When stdout is redirected to a file or pipe it is fully buffered and `write` happens at exit.
3. `exit_group(0)`.

**Lesson:** one C++ statement is far removed from the syscalls it causes. When we sample `/proc` 1,000 times a second in Phase 1, syscalls are where the CPU cost hides.

</details>

### 🧪 EXPERIMENT: `/proc/self`

```bash
grep -E 'Name|Pid|VmSize|VmRSS' /proc/self/status
```

```mermaid
flowchart LR
    A[MMU + page tables] --> B[Kernel mm_struct: page counters]
    B --> C[procfs handler for /proc/PID/status]
    C --> D[read syscall]
    D --> E[Your tool: text to parsed number]
```

`VmSize` is the total size of virtual mappings; `VmRSS` counts pages resident in RAM. The kernel formats them as text **at the moment of each `read()`**. Nothing is stored in a file.

### 🧠 THINK

Why does `grep ... /proc/self/status` report the name and PID of `grep`, not your shell?

<details>
<summary>✅ Solution</summary>

`/proc/self` is a special link the kernel resolves to the PID of **the process doing the lookup**. The reader is `grep`, so you see `grep`'s own status. This is how Debugging Wizard will later observe itself (Phase 1, self-observation).

</details>

---

## Step 1.6 — The Leak Experiment: VSZ vs RSS

`lab/leak_bounded.cpp` allocates 1 MB per second, touches one byte of each block, and never frees. It is bounded so it exits normally (LSan only reports at exit).

```cpp
for (int i = 0; i < iterations; ++i) {
    char* p = static_cast<char*>(std::malloc(1024 * 1024));  // 1 MB
    if (!p) { return 1; }
    p[0] = 'x';                                              // touch one byte
    ...
    sleep(1);
    // deliberately never freed
}
```

Observe it with the helper script (arguments: binary, iterations, samples, interval in seconds):

```bash
scripts/observe_leak.sh build/debug/lab/leak_bounded 60 6 5
```

### 🔬 PREDICT (the original four questions)

1. Will the compiler warn about the leak with `-Wall -Wextra`?
2. Which grows, **VSZ** or **RSS**, and roughly how fast?
3. We touch only one byte of each 1 MB block. Does that change your answer to question 2?
4. Who first notices the leak: the compiler, the kernel, or nobody?

<details>
<summary>✅ Solution</summary>

1. **No.** Standard warnings do not track ownership across loop iterations. Overwriting a pointer is legal C++. (GCC's `-fanalyzer` may flag some cases but is not reliable for C++.)
2. **VSZ grows by roughly 1 MB per second.**
3. **Yes, this is the key insight.** A 1 MB `malloc` exceeds glibc's mmap threshold (128 KB by default), so it becomes a fresh anonymous `mmap`. The kernel reserves the *address range* immediately but supplies **physical pages lazily, on first touch**. Writing `p[0]` faults in one 4 KB page (glibc's chunk header shares that page). So **RSS grows only about 4 KB per second**, roughly 250 times slower than VSZ.
4. **Nobody**, until memory is exhausted or the process exits. The compiler cannot see it and the kernel does not judge. The kernel only reacts when memory runs out (reclaim, then the OOM killer). LSan reports at **exit**, so the original infinite-loop version would report nothing if killed with Ctrl-C.

</details>

> **📊 Reference data (sandbox, different machine).** Sampling every 2 seconds gave `VmSize` +2,056 kB per interval (**1,028 kB per second**) and `VmRSS` +8 kB per interval (**4 kB per second**). Why 1,028 kB rather than 1,024? The request is 1 MiB plus a 16-byte glibc header, and `mmap` rounds up to whole 4 KB pages, adding one extra page.

### 👀 OBSERVE (fill in with your numbers)

```
t=0s   VmSize: ______ kB   VmRSS: ______ kB
t=10s  VmSize: ______ kB   VmRSS: ______ kB
VmSize growth/s: ______      VmRSS growth/s: ______
```

### 🤔 EXPLAIN

If Debugging Wizard watched only **RSS**, how long would this leak take to look alarming? What would it miss by watching only VSZ?

<details>
<summary>✅ Solution</summary>

At 4 KB/s, RSS grows about 14 MB per hour and can hide for a long time. VSZ at about 1 MB/s (3.6 GB per hour) is loud, but VSZ alone misleads too: healthy programs reserve huge address ranges (thread stacks, mapped files, allocator arenas) without using them. A good detector tracks **several signals** (VSZ, RSS, mapping count, page faults) and their **trend**. This is the seed of Phase 3.

</details>

---

## Step 1.7 — Sanitizers: How They Work

| Tool | Catches | Mechanism | Typical cost |
|------|---------|-----------|--------------|
| **ASan** | heap/stack/global overflow, use-after-free, double free | Compiler instruments loads/stores to check **shadow memory** (1 shadow byte per 8 bytes). `malloc` is replaced to add **redzones** around blocks and a **quarantine** that delays reuse of freed blocks. | 1.5-3x time; memory highly workload dependent (see Step 1.10) |
| **LSan** | leaks | At **exit**, scans globals, stacks, registers for pointers to heap blocks. Unreachable blocks are leaks. On by default with ASan on Linux x86_64. | small until exit |
| **UBSan** | signed overflow, bad shifts, null deref, misalignment | Compiler inserts explicit checks at operations that could be undefined behavior. | low to moderate |
| **Valgrind** | invalid access, leaks, uninitialized reads | Runs your **unmodified** binary in a CPU emulator that tracks every byte. | 10-50x slower |

```bash
cmake --preset asan
cmake --build --preset asan -j
```

```mermaid
flowchart LR
    A[Your code: a index 4 assigned 1] --> B[Compiler inserts check]
    B --> C{Shadow memory: is this byte addressable?}
    C -- yes --> D[Access proceeds]
    C -- no --> E[ASan prints report and aborts]
```

**How to read this diagram:** the compiler rewrites each memory access into "check, then access." The check consults a shadow map the runtime maintains. Redzones poison the bytes just past an allocation, so an off-by-one lands in a poisoned byte and is caught.

### 🧠 THINK

1. Why can't we simply leave ASan on in production?
2. LSan reports at exit. What does that imply for a program that never exits normally?
3. Why can ASan and Valgrind not be combined on one binary?

<details>
<summary>✅ Solution</summary>

1. Every memory access gets a check and every allocation is padded and quarantined, which costs CPU and RAM. A diagnostic tool that multiplies its own footprint perturbs the system it observes. Sanitizers belong in development and CI. Debugging Wizard's *production* safety comes from design (RAII, bounded buffers) plus self-observation.
2. A daemon killed by a signal never reaches the exit-time scan. To test with LSan, give the program a clean shutdown path or run a bounded workload, as `leak_bounded` does.
3. Both take over memory allocation and track memory at a low level: ASan replaces `malloc` and maps a huge shadow region, Valgrind emulates the CPU. They conflict, so use the plain `debug` preset for Valgrind.

</details>

**Troubleshooting:** if ASan aborts at startup with a message about an unexpected memory mapping, that is a known interaction between some kernel address-space randomization settings and older sanitizer runtimes. Paste the message to me and we will diagnose it rather than guess.

---

## Step 1.8 — Break It: The Bug Zoo

`lab/bugs.cpp` contains four deliberate bugs selected by argument: `leak`, `overflow` (write one past a heap array), `uaf` (read freed memory), `ub` (signed integer overflow).

```bash
cmake --build --preset debug -j && cmake --build --preset asan -j

for b in leak overflow uaf ub; do echo "== ASan: $b =="; ./build/asan/lab/bugs $b; done
for b in leak overflow uaf ub; do echo "== plain: $b =="; ./build/debug/lab/bugs $b; done
for b in leak overflow uaf ub; do echo "== valgrind: $b =="; valgrind --leak-check=full ./build/debug/lab/bugs $b; done
```

### 🔬 PREDICT

Fill in before running (does each tool detect the bug, and what do you expect it to say?):

| Bug | Plain run | ASan/LSan/UBSan | Valgrind |
|-----|-----------|-----------------|----------|
| leak | ? | ? | ? |
| overflow | ? | ? | ? |
| use-after-free | ? | ? | ? |
| signed overflow | ? | ? | ? |

<details>
<summary>✅ Solution</summary>

| Bug | Plain run | ASan/LSan/UBSan | Valgrind |
|-----|-----------|-----------------|----------|
| leak | runs silently | LSan at exit: `detected memory leaks`, `Direct leak of 100 byte(s)` with the allocation stack | `definitely lost: 100 bytes in 1 blocks` |
| overflow | **appears to work** (prints 1) | ASan: `heap-buffer-overflow`, `WRITE of size 4` | `Invalid write of size 4` |
| use-after-free | runs, but prints **garbage** (allocator metadata, not your 7) | ASan: `heap-use-after-free`, `READ of size 4`, plus where it was freed | `Invalid read of size 4` |
| signed overflow | prints the wrapped value | UBSan: `runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'` | **not detected**: Valgrind sees an ordinary add instruction and knows nothing of C++ rules |

> **📊 Reference data.** The ASan/UBSan and plain columns above were observed in a real run while testing this repo (report headlines are exact). **Valgrind was not available in my sandbox**, so that column reflects its typical output. Confirm the wording on your machine and tell me if it differs.

**Lessons:**
- Memory bugs often **appear to work**. "It ran correctly" is not evidence of correctness.
- Tools see different layers: Valgrind sees *machine-level memory validity*, UBSan sees *language-level undefined behavior*.
- An ASan report gives **what** (bug type), **where** (the access), and **who allocated or freed** the memory. Read all three sections.

</details>

### ⚠️ BREAK IT

Change `a[4] = 1;` to `a[100] = 1;`. Predict whether ASan still catches it, then run.

<details>
<summary>✅ Solution</summary>

Often yes if the access lands in a poisoned redzone, but jumping **far** past an allocation can skip the redzone and land in other valid memory. ASan is strong, not a proof of correctness, and its redzones are finite. This is why tests, warnings, and RAII design work together with sanitizers.

</details>

---

## Step 1.9 — RAII: The Forgotten `close()`

**RAII (Resource Acquisition Is Initialization):** tie a resource's lifetime to an object's lifetime. Acquire in the constructor, release in the destructor. The compiler guarantees the destructor runs when the object leaves scope, including on early `return` and during exception unwinding.

### The broken version

`lab/fd_leak.cpp` opens `/proc/meminfo` in a loop and never closes it. Run it with a **lowered descriptor limit** so it fails quickly, and watch from a second terminal:

```bash
# terminal 1
(ulimit -n 64; ./build/debug/lab/fd_leak)

# terminal 2
watch -n1 'ls /proc/$(pgrep -n fd_leak)/fd | wc -l'
```

### 🔬 PREDICT

1. What happens to the fd count over time?
2. At roughly which iteration does `open` fail, and with what error?
3. Will ASan or LSan detect this bug?

<details>
<summary>✅ Solution</summary>

1. It rises by one per iteration (about 10 per second, since we sleep 0.1 s).
2. Descriptors 0, 1, 2 are already used (stdin, stdout, stderr), so with a limit of 64 the loop can open 61 files and fails at iteration **61** with `Too many open files` (`EMFILE`).
3. **No.** ASan and LSan track **heap memory**. A file descriptor is a small integer indexing a kernel table, so no leaked heap block exists. This is a whole category of leak that memory sanitizers cannot see, and only observation (`/proc/PID/fd`) or design (RAII) catches it. Debugging Wizard will monitor fd growth for this reason (Phase 2).

</details>

> **📊 Reference data.** With `ulimit -n 64` the program failed at exactly iteration 61, and the fd count was 18 at about 1.5 s and 48 at about 4.5 s, matching roughly 10 per second. One extra observation from the same testing: when I ran the **ASan build** with all descriptors exhausted, LeakSanitizer itself crashed with a fatal error at exit, while the same build with the same low limit but *unexhausted* descriptors reported leaks normally. A plausible explanation is that the leak checker needs descriptors of its own, but I did not prove the mechanism. The takeaway is safe either way: **when a resource is exhausted, the diagnostic tools can fail too**, a point that matters for Debugging Wizard's failure handling.

### Where the state lives

```mermaid
flowchart LR
    A[User: int fd = 3] --> B[Per-process fd table in kernel: entry 3 points to open file description]
    B --> C[Kernel file object: offset, flags, inode reference]
    C --> D[procfs: generates text on read]
```

**How to read this diagram:** the integer in your program is just an index. The real state (file offset, flags) is in the kernel. Forgetting `close()` leaks a kernel table slot and a kernel object, and none of it shows up in your heap.

### The fix: `include/dw/fd.hpp`

```cpp
namespace dw {

class Fd {
public:
    Fd() noexcept = default;
    explicit Fd(int fd) noexcept : fd_(fd) {}
    ~Fd() { reset(); }

    Fd(const Fd&) = delete;             // two owners would double-close
    Fd& operator=(const Fd&) = delete;

    Fd(Fd&& other) noexcept : fd_(other.release()) {}
    Fd& operator=(Fd&& other) noexcept {
        if (this != &other) { reset(other.release()); }
        return *this;
    }

    int get() const noexcept { return fd_; }
    bool valid() const noexcept { return fd_ >= 0; }

    int release() noexcept { int f = fd_; fd_ = -1; return f; }
    void reset(int fd = -1) noexcept {
        if (fd_ >= 0) { ::close(fd_); }
        fd_ = fd;
    }

private:
    int fd_ = -1;
};

}  // namespace dw
```

`lab/fd_raii.cpp` opens and closes `/proc/meminfo` 100,000 times using `dw::Fd`, and `tests/fd_test.cpp` checks the class's behavior. Run:

```bash
./build/debug/lab/fd_raii
ctest --preset debug
ctest --preset asan
```

### 🧠 THINK (line by line)

1. Why is the copy constructor `= delete`? What breaks if we remove it?
2. Why does the move constructor call `other.release()`?
3. What is `fd_ = -1` protecting against?
4. Does the destructor run if `main` returns early? If an exception is thrown?
5. In `fd_test.cpp`, why do we test with `fcntl(fd, F_GETFD)` after the scope ends?

<details>
<summary>✅ Solution</summary>

1. The default copy would give two `Fd` objects the same integer, and both destructors would call `close()`. The second `close` hits a number that may **already belong to a different file** opened in the meantime: a nasty, intermittent bug. Deleting copy makes ownership **unique** and turns the mistake into a compile error.
2. After a move, the source must stop owning the descriptor. `release()` returns the fd and sets the source's `fd_` to -1, so the source's destructor does nothing. Ownership transfers exactly once.
3. -1 means "not owning anything." It lets destructors and `reset()` tell "no resource" from a real descriptor, and makes a moved-from object safe to destroy.
4. **Yes to both.** Destructors of locals run on every scope exit: normal end, `return`, and stack unwinding. Manual `close()` calls are easy to skip on some path, and the compiler will never remind you.
5. `F_GETFD` on a closed descriptor fails with `EBADF`. It is a direct, kernel-level way to ask "is this descriptor still open?", so the test checks the *kernel's* state rather than trusting our own bookkeeping.

**Small detail:** the destructor ignores `close()`'s return value. That is acceptable for read-only files. For files we *write*, `close()` can report deferred write errors, which we handle in Chapter 4 (error handling).

</details>

---

## Step 1.10 — Measure: What Does Instrumentation Cost?

Never claim "sanitizers make it 2x slower" without measuring on **your** machine and **your** workload. `lab/bench.cpp` is an allocation-heavy workload (200 rounds of 20,000 small vectors).

```bash
cmake --preset rel     && cmake --build --preset rel -j
cmake --preset asan-rel && cmake --build --preset asan-rel -j
scripts/bench_sanitizers.sh 5        # wall time and max RSS, 5 runs each
```

Both builds use `-O2 -g`, so the comparison is fair.

### 🔬 PREDICT

| Config | Wall time | Max RSS |
|--------|-----------|---------|
| plain (`rel`) | ? | ? |
| ASan+UBSan (`asan-rel`) | ?x plain | ?x plain |

<details>
<summary>✅ Solution: what the reference run showed</summary>

> **📊 Reference data (sandbox, GCC 13, one run each).**
> | Config | Wall time | Max RSS |
> |--------|-----------|---------|
> | plain `-O2` | 0.51 s | 11.5 MB |
> | ASan+UBSan `-O2` | 1.97 s (**3.9x**) | 383.6 MB (**33x**) |

Time was in the expected 2-4x range, but memory was **far above the "2-3x" folklore**. Why?

**Hypothesis:** ASan's *quarantine*. To catch use-after-free, ASan does not reuse freed blocks right away; it holds up to a fixed budget (256 MB by default) of freed memory. Our workload frees millions of small blocks, so the quarantine fills.

**Experiment (one variable changed):** rerun with a smaller quarantine.

```bash
ASAN_OPTIONS=quarantine_size_mb=1 ./build/asan-rel/lab/bench
```

| Quarantine | Wall time | Max RSS |
|------------|-----------|---------|
| default (256 MB) | 1.93 s | 383.5 MB |
| 1 MB | 1.58 s | 18.8 MB |
| 0 | 1.51 s | 15.7 MB |

**Conclusion (evidence-supported):** the quarantine explains nearly all of the extra memory. It is a *fixed budget*, not a multiplier, which is why "ASan uses 2-3x memory" is unreliable for small programs (the fixed cost dominates) and for allocation-heavy ones (the budget fills). It also shows the trade-off: shrinking it saves memory but shortens the window in which ASan can catch a use-after-free.

**Method lessons:** repeat runs and look at the spread; compare like with like; change **one** variable at a time; record the machine in `docs/environment.md`. Your job now is to reproduce this on your Acer and see whether the same mechanism explains your numbers.

</details>

Put your results in `bench/phase0-ch1-sanitizer-cost.md`.

---

## Step 1.11 — Explain It Back, Knowledge Check, Completion

### 🎯 Explain-it-back (no notes)

> Why can a C++ program leak memory without any compiler warning or kernel complaint, and how do RAII, sanitizers, and `/proc` observation each catch (or prevent) a different part of the problem?

<details>
<summary>✅ Model answer</summary>

The compiler checks types and syntax, not runtime ownership, so a dropped pointer is legal code. The kernel hands out pages on request and reclaims everything only at exit, so it never objects to a long-lived process holding more and more. RAII **prevents** the bug by construction: ownership is tied to scope, so resources are released on every path. Sanitizers **detect** bugs in development: ASan at the instant of an invalid access, LSan for unreachable heap blocks at exit, UBSan for undefined language behavior. `/proc` observation catches what neither can, such as fd leaks and slow growth in a running process, by watching kernel-side counters over time. Each covers a different layer, and Debugging Wizard needs all three.

</details>

### Knowledge Check

**Level 1: Recall.** What is the difference between VSZ and RSS?

<details><summary>✅ Solution</summary>VSZ (VmSize) is the total size of the process's virtual mappings. RSS (VmRSS) is the part currently resident in physical RAM.</details>

**Level 2: Understanding.** Why did the leaking program grow VSZ about 250 times faster than RSS?

<details><summary>✅ Solution</summary>Each 1 MB `malloc` became an anonymous `mmap` that added 1 MB (plus a page) to the address space at once. Physical pages are allocated lazily on first touch, and we touched only one 4 KB page per block.</details>

**Level 3: Mechanism.** Where does the number in `VmRSS` originate?

<details><summary>✅ Solution</summary>The kernel keeps resident-page counters in the process's memory descriptor (`mm_struct`), updated as page faults populate page tables. `read()` on `/proc/PID/status` makes the procfs handler format those counters into text at that instant. The MMU and page tables are the hardware-facing truth about which pages are mapped.</details>

**Level 4: Experiment.** How would you prove the fd leak is real without any sanitizer?

<details><summary>✅ Solution</summary>Sample `ls /proc/PID/fd | wc -l` over time and show monotonic growth, then run the RAII version and show a flat count. Confirm with `strace -c -e trace=openat,close` (open/close imbalance) and by hitting `EMFILE` at the `ulimit -n` boundary.</details>

**Level 5: Debugging.** ASan reports nothing, but RSS grows steadily for hours. What remains possible?

<details><summary>✅ Solution</summary>
- The memory is still **reachable** (a cache, container, or queue growing without bound): not a leak by LSan's definition, but a functional leak.
- The program never exited cleanly, so LSan never scanned.
- Growth is outside the sanitized heap (mmap'd files, shared memory, allocator fragmentation).
- An uninstrumented library is allocating.

Distinguishing measurements: `/proc/PID/smaps` shows *which mapping* grows; heap profilers and allocation counters show who allocates. This is a Phase 3 investigation.
</details>

**Level 6: Systems reasoning.** A colleague says "it passes ASan and Valgrind, so it is memory safe." Give at least three reasons that is too strong.

<details><summary>✅ Solution</summary>
1. Sanitizers only check **executed paths**; untested inputs and rare error branches stay unchecked.
2. Neither catches fd, thread, or other kernel-resource leaks.
3. Redzone checks can miss far-out-of-bounds accesses.
4. Data races need ThreadSanitizer (which cannot be combined with ASan).
5. Sanitizer builds change timing, so some races vanish or appear.
6. Long-run behavior (fragmentation, slow growth) needs hours of observation, not a short test.
7. As we saw, when resources are exhausted the diagnostic tooling itself can fail.
</details>

### Exercises

1. **Warnings as errors.** Configure with `-DDW_WERROR=ON`, introduce an unused variable in a lab file, and confirm the build fails. Then remove it.
2. **Fix the leak.** Rewrite the loop in `leak_bounded.cpp` (in a copy, `leak_fixed.cpp`, registered in `lab/CMakeLists.txt`) with `std::unique_ptr<char[]>`. Show with `observe_leak.sh` that `VmSize` no longer grows. *Predict first, and think about what glibc does when a large mmap'd block is freed.*
3. **Mini Debugging Wizard.** Write `lab/fd_count.cpp`: given a PID, print the entry count of `/proc/PID/fd` once per second using `<filesystem>` or `opendir`. Run it against `fd_leak`. Which failures must it handle (permission denied, the process disappears)?
4. **Break the RAII class.** Remove `= delete` from the copy constructor in a scratch copy of `fd.hpp`, then write a small program that copies an `Fd` and observe the double-close with `strace -e trace=close`.
5. **Debug it.** Use the gdb session from Step 1.4 on `bugs overflow` in the `asan` preset with `ASAN_OPTIONS=detect_leaks=0`. Where does ASan abort, and what does gdb show at that point?

---

## Chapter Completion Criteria

I can:

- [ ] Explain source → ELF → process and redraw the Step 1.1 diagram from memory
- [ ] Explain where `VmRSS` comes from (hardware → kernel → `/proc` → `read()`)
- [ ] Configure, build, and test with CMake presets, and explain what each preset is for
- [ ] Get clangd completion, a clang-tidy diagnostic, and a gdb breakpoint working in VS Code
- [ ] Explain how a breakpoint works at the instruction level
- [ ] Predict, then measure, VSZ vs RSS for the leak program
- [ ] Identify all four bug-zoo bugs with sanitizers **and** Valgrind, and explain why Valgrind misses the UB case
- [ ] Demonstrate an fd leak and explain why ASan cannot see it
- [ ] Explain every line of `dw::Fd`, including why copying is deleted
- [ ] Report measured sanitizer overhead with method and explain the quarantine effect
- [ ] Answer all six knowledge-check levels without opening the solutions

---

## What This Unlocks Next

**Chapter 2** looks at a process's memory from the inside: reading `VmSize`, `VmRSS`, and `/proc/self/maps` from within C++. It uses `dw::Fd` from this chapter for the first real measurement code, and it is where `src/` receives its first file.

---

## Clarification Log

> Ask me anything about any step. Each question and answer can be appended here so this file becomes your complete study record.

| # | Step | Your question | Answer summary |
|---|------|---------------|----------------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
