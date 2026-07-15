# Linux Process Programming — From C to C++

> A chapter-wise study guide covering `fork()`, `wait()`, process IDs, pipes, and FIFOs.
> Each chapter first builds the **core OS concept**, then shows the **C way**, then shows
> **how a modern C++ engineer would harden the same code**. Read top to bottom — later
> chapters lean on vocabulary introduced earlier.

---

## Chapter 0 — Foundations You Need Before Touching `fork()`

Before any code, let's build the mental model. Skipping this is why `fork()` confuses
almost everyone the first time.

### What is a process?

A **process** is a running instance of a program. It owns:

- Its own **virtual address space** (code, stack, heap, globals) — isolated from every
  other process.
- A **process ID (PID)** — a unique integer the kernel uses to track it.
- A **parent process ID (PPID)** — the PID of whoever created it.
- A table of **open file descriptors** (small integers referencing open files, pipes,
  sockets, etc.).
- Kernel-tracked state: running, sleeping, zombie, etc.

A **thread**, in contrast, is a unit of execution *inside* a process. Multiple threads in
one process share the same address space (same heap, same globals) but each gets its own
stack. That's the big difference to keep in the back of your mind throughout this guide:
**processes duplicate memory, threads share it.** This series deliberately starts with
processes because the isolation makes the mental model of "who am I and what do I own"
much easier to reason about before you add shared-memory hazards from threads.

### What does `fork()` actually do?

`fork()` is a system call. Calling it **clones the calling process**: the kernel creates
a near-exact duplicate of your process — same code, same variable values, same open file
descriptors — and that duplicate (the **child**) starts running from the very next
instruction after the `fork()` call, in parallel with the original (the **parent**).

Key mental model: picture your program as a single line of execution. The instant you
call `fork()`, that line **splits into two independent lines**, both continuing from that
exact point. Everything *before* the call ran once. Everything *after* the call runs
**twice** — once per process — because there are now two processes executing it.

`fork()` returns a value, and that return value is *the only way* your code can tell
which line it's on:

| Where you are        | Return value of `fork()`                          |
|-----------------------|----------------------------------------------------|
| Inside the **child**  | `0`                                                |
| Inside the **parent** | The child's actual PID (a positive integer)        |
| `fork()` failed       | `-1` (no new process was created — e.g. resource limits hit) |

That's it — that's the entire trick behind every example in this guide. `fork()` doesn't
"tell" a process what it is; you infer it from the number it hands back.

### Why does memory look "copied" but independent?

When the child is created, its variables start out **identical in value** to the
parent's (same `n`, same `int i`, etc.) because the whole address space was duplicated.
But it's a **separate copy in separate physical memory** (modern kernels do this cheaply
via *copy-on-write* — physical pages are only actually duplicated when one side writes to
them). So the instant either process modifies a variable, the two copies diverge. There is
**no shared memory between parent and child by default** — that's exactly why later
chapters need pipes and FIFOs when the two processes actually need to exchange data.

---

## Chapter 1 — `fork()`: Creating Your First Child Process

### Concept

To use `fork()` on Linux you need `<unistd.h>` (this header is POSIX/Linux-specific, not
portable to Windows without an emulation layer like Cygwin/WSL).

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    printf("hello world\n");
    return 0;
}
```

This prints `hello world` once. Now insert a `fork()` before the `printf`:

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    fork();
    printf("hello world\n");
    return 0;
}
```

Now `hello world` prints **twice** — once from the parent, once from the child — because
both processes execute every line that comes after `fork()`.

### Reading the return value

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    int id = fork();
    printf("hello world from id %d\n", id);
    return 0;
}
```

You'll see two different numbers: one line prints `0` (the child), the other prints some
larger integer (the parent seeing the child's PID). Use that to branch:

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    int id = fork();

    if (id == 0) {
        printf("hello from child process\n");
    } else if (id > 0) {
        printf("hello from the main process\n");
    } else {
        // id == -1
        printf("fork failed\n");
    }
    return 0;
}
```

> **Senior-engineer habit:** always handle the `-1` case. Beginners write `if (id == 0)
> else` and silently assume `fork()` succeeded. In production code, a fork failure
> (usually `EAGAIN` — too many processes, or `ENOMEM`) must be handled, not ignored.

### The doubling trap: calling `fork()` multiple times

This is the single most common interview/exam gotcha:

```c
fork();
fork();
printf("hello world\n");
```

How many times does `hello world` print? **Four.** Each `fork()` doubles the number of
execution lines that reach it. Call it `n` times unconditionally and you get **2ⁿ**
processes, not `n`. Call it 4 times → 16 processes. People have accidentally taken down
shared servers by doing this inside a loop:

```c
for (int i = 0; i < 10; i++) {
    fork(); // DANGER: this creates 2^10 = 1024 processes, not 10!
}
```

To create exactly 3 processes (1 parent forking twice, each fork guarded so children
don't fork again):

```c
int id = fork();
if (id != 0) {
    // only the parent forks again
    fork();
}
```

**Rule of thumb:** if you want to fan out to *N* worker processes, only ever call
`fork()` from the **original parent**, in a loop, and make sure children immediately
check "am I the child?" and skip the loop/further forking.

### C++ upgrade

`fork()` itself is still a raw POSIX syscall in C++ — there's no `std::fork`. The value
C++ adds is **safety around what you do with the result**:

```cpp
#include <unistd.h>
#include <iostream>
#include <stdexcept>

pid_t safe_fork() {
    pid_t pid = fork();
    if (pid == -1) {
        throw std::runtime_error("fork() failed");
    }
    return pid;
}

int main() {
    try {
        pid_t id = safe_fork();
        if (id == 0) {
            std::cout << "hello from child process\n";
        } else {
            std::cout << "hello from the main process\n";
        }
    } catch (const std::exception& e) {
        std::cerr << e.what() << '\n';
        return 1;
    }
}
```

**Caveats a senior engineer will flag immediately:**

- `fork()` only duplicates the calling **thread**. If your C++ program is multi-threaded,
  only the thread that called `fork()` survives in the child; other threads simply vanish
  — locks they held stay locked forever. This is a classic multi-threaded + `fork()` bug.
- C++ objects with non-trivial destructors get duplicated too — both parent and child now
  own "copies" of the same RAII resource (e.g. two `std::fstream` handles pointing at the
  same underlying file descriptor number space). Be deliberate about which process closes
  what, exactly like the pipe examples in later chapters.
- Throwing exceptions across a `fork()` boundary is fine (each process has its own stack),
  but never let an exception unwind past `main()` in the child without an explicit
  `_exit()` — otherwise both processes might try to run the same cleanup/global
  destructors, double-flushing buffers or double-closing fds.

---

## Chapter 2 — `wait()`: Synchronizing Parent and Child

### The problem: unordered execution

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    int n;
    int id = fork();

    if (id == 0) n = 1;   // child prints 1..5
    else        n = 6;    // parent prints 6..10

    for (int i = n; i < n + 5; i++) {
        printf("%d\n", i);
        fflush(stdout);
    }
    return 0;
}
```

You'd hope for `1 2 3 4 5 6 7 8 9 10`. Instead you get something like `1 6 2 7 3 8 4 9 5
10` — or any other interleaving. **Why:** once `fork()` splits execution, the OS
scheduler decides how to timeslice the two independent processes. There is no guarantee
about ordering between them. This is a fundamental truth of concurrent execution, not a
bug: **the kernel scheduler owns ordering, not your source code.**

`fflush(stdout)` is used here because `printf` buffers by default — without flushing, you
might not even see output in the order it was *generated* internally, since the C library
batches writes.

### The fix: `wait()`

`wait()` (from `<sys/wait.h>`) blocks the calling process until **any one of its child
processes finishes**. Call it only in the parent:

```c
#include <sys/wait.h>

int id = fork();
int n = (id == 0) ? 1 : 6;

if (id != 0) {
    wait(NULL);   // parent waits for the child to fully finish first
}

for (int i = n; i < n + 5; i++) {
    printf("%d\n", i);
    fflush(stdout);
}
```

Now you deterministically get `1 2 3 4 5` (child) followed by `6 7 8 9 10` (parent),
every single run.

> **Core concept — why this matters beyond printing numbers:** `wait()` is how a parent
> avoids **zombie processes** (Chapter 4) and how it collects a child's **exit status**.
> Any time you fork off work, the parent is responsible for eventually calling `wait()` —
> exactly like a thread's `join()`, which you'll meet if this series moves to threads.

### C++ upgrade

```cpp
#include <sys/wait.h>
#include <unistd.h>
#include <cstdio>
#include <stdexcept>

class ChildProcess {
public:
    explicit ChildProcess(pid_t pid) : pid_(pid) {}

    // RAII: destructor waits automatically if nobody did it explicitly —
    // this prevents "I forgot to wait()" leaks (zombies) by construction.
    ~ChildProcess() {
        if (pid_ > 0 && !waited_) {
            int status;
            waitpid(pid_, &status, 0);
        }
    }

    int join() {
        int status;
        waitpid(pid_, &status, 0);
        waited_ = true;
        return status;
    }

    ChildProcess(const ChildProcess&) = delete;            // don't allow copies
    ChildProcess& operator=(const ChildProcess&) = delete;  // — one owner per PID

private:
    pid_t pid_;
    bool waited_ = false;
};
```

Using it reads almost exactly like `std::thread::join()`:

```cpp
pid_t id = fork();
if (id == 0) {
    // child work...
    _exit(0);
}
ChildProcess child(id);
child.join();   // equivalent to wait(), but explicit and self-documenting
```

**Why this is the "senior" version:** it makes forgetting to `wait()` a compile-time-shaped
habit rather than a runtime leak — the destructor is your safety net. It also mirrors the
API shape of `std::thread`, so if your team later migrates part of this to threads instead
of processes, the calling code barely changes.

In the context of the `waitpid` function, these arguments are used to manage how the parent process waits for a child process to finish.

The function signature is:
`pid_t waitpid(pid_t pid, int *wstatus, int options);`

Here is the breakdown of the parameters you asked about:

---

### 1. The `status` (`int *wstatus`)

The `status` argument is a **pointer to an integer** where `waitpid` will store information about *how* the child process terminated.

It is not just a simple return code; it is a bit-mask. You should **not** inspect the integer directly. Instead, you must use POSIX-defined **macros** to extract the data from it:

* **`WIFEXITED(status)`**: Returns true if the child terminated normally (e.g., calling `exit()` or returning from `main`).
* **`WEXITSTATUS(status)`**: If `WIFEXITED` is true, this macro extracts the actual exit code (the value passed to `exit()`).
* **`WIFSIGNALED(status)`**: Returns true if the child was terminated by a signal (e.g., a crash like `SIGSEGV` or `SIGKILL`).
* **`WTERMSIG(status)`**: If `WIFSIGNALED` is true, this macro tells you which signal killed the process.

**Example usage:**

```cpp
int status;
waitpid(pid_, &status, 0);

if (WIFEXITED(status)) {
    int exit_code = WEXITSTATUS(status);
    printf("Child exited with code: %d\n", exit_code);
}

```

---

### 2. The `0` (the `options` parameter)

The `0` is the `options` argument. When you pass `0`, you are telling the system to **perform a standard, blocking wait**.

* **Blocking behavior:** If the child process is still running, the parent process will stop and "sleep" at this line until the child process finishes.
* **Other common options:**
* **`WNOHANG`**: This makes the call **non-blocking**. If the child is still running, `waitpid` returns immediately with a value of `0` instead of waiting. This is useful if you want to check on a child process without pausing your main program's loop.
* **`WUNTRACED`**: Also returns if the child has stopped (but not terminated) because of a signal.



---

### Summary Table

| Parameter | Type | Purpose |
| --- | --- | --- |
| `pid_` | `pid_t` | The specific Process ID of the child you want to wait for. |
| `&status` | `int *` | A memory address where the system writes information about the child's exit state. |
| `0` | `int` | The option flag. `0` means "wait until the child is done." |

**Why it matters:** Without calling `waitpid` (or `wait`), you end up with **"Zombie Processes"**. When a child process dies, it stays in the system process table as a "zombie" so the parent can read its exit status. If the parent never calls `waitpid`, the zombie stays there forever, consuming a process ID slot.

---

## Chapter 3 — Visualizing What Actually Happens in Memory

This chapter is conceptual (no new syntax), but it's the piece that makes everything else
click, so don't skip it.

Walk `int id = fork();` step by step:

1. **Before `fork()`:** one process, one copy of every variable.
2. **`fork()` executes:** the kernel makes a second process. It copies the *entire*
   variable state as it existed at that instant — same `n`, same `i`, same everything —
   into a **new, independent memory space**.
3. **The one difference the kernel introduces:** the return value of `fork()` itself.
   Child gets `0`. Parent gets the child's real PID (e.g. `4300`).
4. **From this point on**, each process's copy of every variable is completely
   independent. If the child does `n = 1;`, the parent's `n` is untouched — they are not
   pointers to shared memory, they are two distinct copies living at (conceptually) the
   same virtual address but backed by different physical pages.
5. **Both processes execute the identical subsequent lines of code** — same `if`
   statements, same loops — but because their copy of `id` differs, they take different
   branches and produce different output.

Two conclusions to internalize permanently:

- **Memory is copied value-by-value at the moment of `fork()`, then diverges.** There is
  no live synchronization after that point — if you need synchronization or data
  exchange, you must build it explicitly (Chapters 6–9: pipes and FIFOs).
- **Both processes run the *same compiled code*.** The `if (id == 0)` branches are what
  give each process its distinct personality. This is different from, say, spawning two
  different executables — `fork()` clones *this* program, not launches a different one
  (that's what `exec()`-family functions are for, which pair with `fork()` in real-world
  shells and servers but aren't covered in this transcript).

**C++ framing:** this is precisely why capturing C++ objects by reference/pointer across a
`fork()` and expecting the child to see the parent's future mutations is a bug — there is
no "future" shared state. If two processes must share live, mutable state, you need
explicit OS-level shared memory (`shm_open`/`mmap`) or message passing (pipes/FIFOs/
sockets), never a bare pointer captured before the fork.

---

## Chapter 4 — Process IDs, Parent IDs, and Zombie/Orphan Processes

### Concept: every process has an identity and a lineage

Every process on Linux (and any Unix-like system) has:

- A **PID** (`getpid()`) — its own unique ID.
- A **PPID** (`getppid()`) — the PID of whoever forked it.

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    printf("current id %d, parent id %d\n", getpid(), getppid());
    return 0;
}
```

The **only** process without a "real" creator is the very first process the kernel starts
at boot (historically PID 1, `init`/`systemd`). Every other process traces its lineage
back to it.

### Zombies: what happens if the parent doesn't `wait()`

If a child finishes execution but its parent never calls `wait()`, the child becomes a
**zombie**: the process has already terminated (freed its own memory, closed its own
files) but the kernel keeps a tiny leftover record — just enough to hold its **exit
status** — until some parent collects it via `wait()`. A zombie **cannot be killed** with
`kill`/`SIGKILL`, because it isn't actually running anymore; there's nothing left to kill.
Only `wait()`-ing (by its parent) removes the zombie entry from the process table. Enough
accumulated zombies exhaust the finite PID table — a real, if slow-building, resource
leak.

> **Golden rule, worth memorizing:** *every process that calls `fork()` must eventually
> call `wait()` (or `waitpid()`) for each child it created* — no exceptions, in any
> production code.

### Orphans: what happens if the parent dies first

If the **parent terminates before the child**, the child doesn't vanish — it gets
**re-parented** to a different existing process (traditionally PID 1 / `init`, or on
modern systems whatever your process supervisor is, e.g. `systemd`). This is the kernel's
way of guaranteeing every process always has *some* parent to eventually `wait()` for it.
You can observe this by having the child sleep (so it outlives the parent) and printing
`getppid()` before and after the parent exits — the PPID value changes once the original
parent is gone.

### `wait()`'s return value is useful, not just a blocking call

```c
#include <sys/wait.h>
#include <errno.h>

int res = wait(NULL);
if (res == -1) {
    printf("no children to wait for\n");
} else {
    printf("finished execution: %d\n", res); // the PID that was reaped
}
```

`wait()` returns the **PID of the child it just reaped**, or `-1` if there was nothing to
wait for (check `errno == ECHILD` to distinguish "no children" from a genuine error).

### C++ upgrade

Wrap the (PID, waited?) pair from Chapter 2's `ChildProcess` class with richer status
decoding, using the POSIX macros instead of raw integers:

```cpp
#include <sys/wait.h>
#include <optional>
#include <iostream>

struct ExitInfo {
    pid_t pid;
    bool exited_normally;
    int exit_code_or_signal;
};

std::optional<ExitInfo> reap_any_child() {
    int status;
    pid_t pid = waitpid(-1, &status, 0);   // -1 == "any child", same as wait(NULL)
    if (pid == -1) {
        return std::nullopt;               // ECHILD: nothing left to reap
    }
    if (WIFEXITED(status)) {
        return ExitInfo{pid, true, WEXITSTATUS(status)};
    }
    return ExitInfo{pid, false, WTERMSIG(status)}; // died from a signal
}
```

This is the shape production process-supervisors (think: a tiny `init`/reaper) actually
use — decoding *why* a child died (clean exit vs. killed by signal) rather than treating
`wait()`'s status as an opaque int, which is a common beginner mistake carried over
straight from C.

---

## Chapter 5 — Multiple `fork()` Calls and Building a Process Tree

### Concept: forking creates a tree, not a list

Call `fork()` twice **unconditionally** and you don't get 2 extra processes — you get a
small **binary tree** of 4 total processes, because the second `fork()` runs in *both* of
the processes produced by the first one.

```c
int id1 = fork();
int id2 = fork();
```

Work out who's who by reading the `(id1, id2)` pairs as tree coordinates:

| Process                              | `id1`         | `id2`         |
|---------------------------------------|---------------|---------------|
| Original root process                 | `X` (child's real PID) | `Y` (its own child's real PID) |
| First fork's child, second fork's parent | `0`        | `Y`           |
| First fork's child's child (a leaf)   | `0`           | `0`           |
| Root's second child (a leaf)          | `X`           | `0`           |

**Reading rule:** if an ID variable is `0`, you are the *child* created by that particular
`fork()` call. If it's non-zero, you are that call's *parent* and hold the real PID of the
process it spawned. A leaf process (no further children) is characterized by **all** its
ID variables being `0`.

```c
if (id1 == 0) {
    if (id2 == 0) {
        printf("we are process Y\n"); // deepest leaf
    } else {
        printf("we are process X\n"); // child of root, but itself a parent
    }
} else {
    if (id2 == 0) {
        printf("we are process Z\n"); // root's second child, a leaf
    } else {
        printf("we are the parent process\n"); // the original root
    }
}
```

### The trap of `wait(NULL)` with more than one child

`wait(NULL)` waits for **any single** child to finish — not *all* children. The original
root process here has *two* children (from `id1`'s fork and `id2`'s fork). A single
`wait(NULL)` call returns as soon as the *first* one finishes and then the parent
proceeds — potentially exiting while the second child is still running. The fix is to
loop until there's truly nothing left:

```c
#include <errno.h>
#include <sys/wait.h>

while (wait(NULL) != -1) {
    printf("waited for a child to finish\n");
}
if (errno != ECHILD) {
    // a genuine error, not just "no more children"
    perror("wait");
}
```

### The classic loop-of-`fork()` disaster

```c
for (int i = 0; i < 10; i++) {
    fork();   // creates 2^10 = 1024 processes, NOT 10!
}
```

This has genuinely crashed real systems (fork-bombing your own machine by accident). The
fix: only ever call `fork()` from the **original parent**, and make children bail out of
the loop immediately:

```c
for (int i = 0; i < 10; i++) {
    int id = fork();
    if (id == 0) break;   // children stop forking further; only the parent continues the loop
}
```

### C++ upgrade

Model the tree explicitly with a small pool/manager instead of hand-tracking `id1`,
`id2`, `id3`... variables — this scales the "which am I" logic past 2 forks without the
combinatorial `if`-nesting shown above:

```cpp
#include <vector>
#include <unistd.h>
#include <sys/wait.h>
#include <stdexcept>

class ProcessPool {
public:
    // Spawns `count` children from THIS process only (never re-entered by children).
    explicit ProcessPool(int count) {
        for (int i = 0; i < count; ++i) {
            pid_t pid = fork();
            if (pid == -1) throw std::runtime_error("fork failed");
            if (pid == 0) {
                is_child_ = true;
                return;   // a child never continues this loop -> no combinatorial explosion
            }
            children_.push_back(pid);
        }
    }

    bool is_child() const { return is_child_; }

    void wait_for_all() {
        for (pid_t pid : children_) {
            waitpid(pid, nullptr, 0);
        }
    }

private:
    std::vector<pid_t> children_;
    bool is_child_ = false;
};
```

```cpp
int main() {
    ProcessPool pool(3); // exactly 3 worker children, never 2^3
    if (pool.is_child()) {
        // worker body
        _exit(0);
    }
    pool.wait_for_all(); // waits for ALL of them, not just the first
}
```

This directly fixes both bugs from this chapter: it can never accidentally double-fork
(children return immediately), and it waits for every child it tracked, not just one.

---

## Chapter 6 — Pipes: One-Way Communication Between Related Processes

### Concept: `fork()` copies memory, it doesn't share it — pipes bridge that gap

A **pipe** is an in-kernel, in-memory byte buffer with two ends: a **read end** and a
**write end**, each exposed to your program as a **file descriptor** (a small integer
handle, the same kind of handle you'd get from opening a real file).

```c
#include <unistd.h>

int fd[2];
if (pipe(fd) == -1) {
    perror("pipe failed");
    return 1;
}
// fd[0] = read end, fd[1] = write end
```

**Critical ordering:** you must call `pipe()` **before** `fork()`. File descriptors are
inherited across `fork()` — after forking, both processes hold their own copies of `fd[0]`
and `fd[1]`, each independently open/closeable, both pointing at the *same* underlying
kernel pipe. That inheritance is precisely what makes a pipe usable for parent↔child
communication: it's the one piece of "shared state" the kernel maintains for you, in
contrast to everything else that diverges post-fork (Chapter 3).

### Writing and reading

```c
write(fd[1], &x, sizeof(x));   // write end
read(fd[0], &y, sizeof(y));    // read end
```

Both `read()`/`write()` return the number of bytes transferred, or `-1` on error — always
check this:

```c
if (write(fd[1], &x, sizeof(x)) == -1) {
    perror("write");
    return 2;
}
```

### Discipline: close the ends you don't use

A pipe conceptually needs exactly **one writer and one reader**. Since both processes
inherit *both* ends, good practice is to close whichever end you don't personally use in
each process:

```c
int id = fork();

if (id == 0) {
    close(fd[0]);                 // child only writes -> close read end
    int x;
    scanf("%d", &x);
    write(fd[1], &x, sizeof(x));
    close(fd[1]);
} else {
    close(fd[1]);                 // parent only reads -> close write end
    int y;
    read(fd[0], &y, sizeof(y));
    close(fd[0]);
    printf("got from child process: %d\n", y);
}
```

Because both fds were inherited, that's **4 total close() calls** across both processes
(2 per process) to fully tear the pipe down cleanly — not closing unused ends is a common
source of pipes that never signal EOF properly, because a lingering open write-end
convinces the reader more data might still be coming.

### C++ upgrade

Wrap the raw fds in RAII so "forgot to close" becomes structurally impossible, and give
read/write type-safe, retry-aware helpers instead of raw `void*` byte-blasting:

```cpp
#include <unistd.h>
#include <stdexcept>
#include <cstring>

class Pipe {
public:
    Pipe() {
        if (pipe(fds_) == -1) throw std::runtime_error("pipe() failed");
    }
    ~Pipe() { close_read(); close_write(); }

    Pipe(const Pipe&) = delete;
    Pipe& operator=(const Pipe&) = delete;

    void close_read()  { if (fds_[0] != -1) { close(fds_[0]); fds_[0] = -1; } }
    void close_write() { if (fds_[1] != -1) { close(fds_[1]); fds_[1] = -1; } }

    template <typename T>
    void write_value(const T& value) {
        ssize_t n = write(fds_[1], &value, sizeof(T));
        if (n != sizeof(T)) throw std::runtime_error("short/failed write");
    }

    template <typename T>
    T read_value() {
        T value{};
        ssize_t n = read(fds_[0], &value, sizeof(T));
        if (n != sizeof(T)) throw std::runtime_error("short/failed read");
        return value;
    }

private:
    int fds_[2];
};
```

```cpp
Pipe p;
pid_t id = fork();
if (id == 0) {
    p.close_read();
    int x = 44;
    p.write_value(x);
    p.close_write();
    _exit(0);
} else {
    p.close_write();
    int y = p.read_value<int>();
    std::cout << "got from child process: " << y << '\n';
}
```

**Why this is safer than the C version:** the destructor guarantees the fds close even if
an exception is thrown mid-function (no more "forgot the 4th `close()` call" bugs), and
`write_value`/`read_value` throw on short reads/writes instead of silently reading garbage
padding — the raw C version in the transcript doesn't check for *partial* reads/writes,
which is a real bug lurking in "successful-looking" code (a `write()` can legally
transfer fewer bytes than requested, especially for larger payloads).

---

## Chapter 7 — Practical Pipe Use Case: Parallel Sum of an Array

### Concept: splitting work across processes for real parallelism

This ties Chapters 1, 2, and 6 together into an actual useful pattern: divide an array in
half, let the **child** sum one half while the **parent** sums the other half
*simultaneously* (true parallelism if you have multiple CPU cores — unlike threads which
still need explicit synchronization for shared data, here each process safely owns its
own half with zero risk of a data race, precisely because memory isn't shared).

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int array[] = {1, 2, 3, 4, 1, 3};
    int array_size = sizeof(array) / sizeof(int);

    int fd[2];
    if (pipe(fd) == -1) return 1;

    int id = fork();
    if (id == -1) return 2;

    int start, end;
    if (id == 0) {
        start = 0;
        end = array_size / 2;
    } else {
        start = array_size / 2;
        end = array_size;
    }

    int sum = 0;
    for (int i = start; i < end; i++) sum += array[i];
    printf("calculated partial sum: %d\n", sum);

    if (id == 0) {
        close(fd[0]);
        if (write(fd[1], &sum, sizeof(sum)) == -1) return 3;
        close(fd[1]);
    } else {
        close(fd[1]);
        int sum_from_child;
        if (read(fd[0], &sum_from_child, sizeof(sum_from_child)) == -1) return 4;
        close(fd[0]);

        int total_sum = sum + sum_from_child;
        printf("total sum is %d\n", total_sum);
        wait(NULL);   // reap the child so it doesn't become a zombie
    }
    return 0;
}
```

**Why `wait(NULL)` matters here specifically:** the parent finishes reading from the pipe
before the child has necessarily fully exited — reading data doesn't guarantee the child
process has terminated. `wait()` guarantees the parent doesn't itself exit (and get
reaped by `init`) while its child lingers unreaped.

**Scaling note called out directly in the lesson:** this pattern is genuinely useful when
the array is huge and the *work per element* is expensive — summing is a toy example.
The homework extension (splitting into 3+ workers with more pipes) is exactly the
generalization industrial map-reduce-style batch processors use, just with more pipes/
children and a final reduction step in the coordinator process.

### C++ upgrade

Use `std::span`-like slicing (or plain iterators), `std::accumulate`, and the `Pipe`/
`ChildProcess` RAII wrappers from earlier chapters so the "business logic" (summing) isn't
tangled with fd bookkeeping:

```cpp
#include <numeric>
#include <vector>
#include <iostream>

long sum_range(const std::vector<int>& data, size_t start, size_t end) {
    return std::accumulate(data.begin() + start, data.begin() + end, 0L);
}

int main() {
    std::vector<int> data{1, 2, 3, 4, 1, 3};
    size_t mid = data.size() / 2;

    Pipe pipe;
    pid_t id = fork();
    if (id == -1) throw std::runtime_error("fork failed");

    if (id == 0) {
        pipe.close_read();
        long partial = sum_range(data, 0, mid);
        pipe.write_value(partial);
        pipe.close_write();
        _exit(0);
    }

    pipe.close_write();
    long my_partial = sum_range(data, mid, data.size());
    long child_partial = pipe.read_value<long>();
    pipe.close_read();

    ChildProcess(id).join();
    std::cout << "total sum: " << (my_partial + child_partial) << '\n';
}
```

**Senior-engineer note on generalizing to N workers:** if you go beyond 2 processes
(the homework in the transcript), don't hand-roll N pipes — that's what the
`ProcessPool` class from Chapter 5 is for. In real systems you'd typically reach for
`std::async`/thread pools instead of raw `fork()` once the unit of work is small and
shared-nothing isn't a hard requirement — processes shine specifically when you want
strong memory isolation (e.g. one crashing worker can't corrupt another's heap), which
threads cannot give you.

---

## Chapter 8 — FIFOs (Named Pipes): Communication Between *Unrelated* Processes

### Concept: the one limitation of pipes, and how FIFOs remove it

A regular pipe (Chapter 6) only works between processes that share a common ancestor who
created the pipe before forking — the fds are inherited, not discoverable from outside
that process tree. A **FIFO** (First-In-First-Out; also called a **named pipe**) removes
that restriction: it's a special file *type* that lives in the filesystem (so it has a
path, visible to `ls`), but behaves like a pipe — bytes written on one end come out the
other end, in order, and it does **not** actually store data on disk.

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <errno.h>

if (mkfifo("myfifo1", 0777) == -1) {
    if (errno != EEXIST) {
        perror("could not create FIFO file");
        return 1;
    }
    // EEXIST just means it's already there from a previous run — that's fine
}
```

`0777` is the Unix permission bits (octal) — here, readable/writable/executable by
everyone. Checking `errno == EEXIST` (from `<errno.h>`) specifically lets you safely
re-run a program that creates its own FIFO without erroring every subsequent launch.

### Opening blocks until both ends are present — this is the defining feature

```c
#include <fcntl.h>

int fd = open("myfifo1", O_WRONLY);   // BLOCKS until some other process opens it O_RDONLY
```

This is documented kernel behavior (see `man 2 open`, the FIFO section): **opening a FIFO
for reading blocks until another process opens it for writing, and vice versa.** This
"rendezvous" property is exactly why FIFOs are useful for coordination between
independently-launched programs — no shared ancestry, no pre-arranged fds, just a known
filesystem path both sides agree on. If you open for `O_RDWR` (both directions) instead,
the open call won't block, because it's already satisfying "someone has it open for
reading" and "someone has it open for writing" from the same single call — useful for
quick testing but not the normal pattern for genuine two-party communication.

### C++ upgrade

```cpp
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cerrno>
#include <stdexcept>
#include <string>

class Fifo {
public:
    explicit Fifo(std::string path) : path_(std::move(path)) {
        if (mkfifo(path_.c_str(), 0666) == -1 && errno != EEXIST) {
            throw std::runtime_error("mkfifo failed: " + path_);
        }
    }

    int open_for_write() { return open_checked(O_WRONLY); }
    int open_for_read()  { return open_checked(O_RDONLY); }

private:
    int open_checked(int flags) {
        int fd = open(path_.c_str(), flags);
        if (fd == -1) throw std::runtime_error("open failed on FIFO: " + path_);
        return fd;
    }
    std::string path_;
};
```

**Why bother wrapping something this thin?** Because `mkfifo`'s "ignore `EEXIST`" logic is
exactly the kind of one-liner that gets copy-pasted incorrectly (forgetting the `errno`
check, or checking the wrong constant) across a codebase. Centralizing it in one
constructor means every caller gets the correct idempotent behavior for free, and the
exception type documents *which path* failed, unlike a bare `perror()`.

---

## Chapter 9 — Sending Real Data Through a FIFO Between Two Separate Programs

### Concept: FIFOs work exactly like pipes once both ends are open

Once past the `open()` rendezvous (Chapter 8), read/write on a FIFO behaves identically to
a pipe:

**Program A (generator):**
```c
int fd = open("sum", O_WRONLY);   // blocks until Program B opens for reading

int array[5] = { /* random numbers */ };
write(fd, array, 5 * sizeof(int));  // one call, writes the whole array at once
close(fd);
```

**Program B (reader):**
```c
int fd = open("sum", O_RDONLY);   // blocks until Program A opens for writing

int array[5];
read(fd, array, 5 * sizeof(int));  // one call reads everything sitting in the buffer

int sum = 0;
for (int i = 0; i < 5; i++) sum += array[i];
printf("result is %d\n", sum);
```

### Batch reads/writes vs. one-at-a-time — an important performance lesson

The lesson explicitly contrasts writing 5 integers with **5 separate `write()` calls** vs.
**1 call writing `5 * sizeof(int)` bytes**. Both are *functionally* correct, but the
single batched call is far more efficient — fewer system calls, fewer context switches
into the kernel. On the read side, it doesn't actually matter whether you call `read()`
five times or once for `5 * sizeof(int)` bytes — the kernel's FIFO buffer holds whatever
was written and lets you drain it however you like; reads and writes on a FIFO don't have
to match granularity 1:1. The practical takeaway: **batch your writes when you control the
producer**, since that's where the system-call overhead savings actually happen.

> **A subtlety worth internalizing:** the FIFO has a finite kernel buffer. You *can* write
> more data than the reader has consumed so far (up to that buffer's capacity) — the
> writer isn't required to wait for the reader to catch up byte-for-byte, only the initial
> `open()` rendezvous is synchronous. If the buffer fills up, `write()` will then block
> until the reader drains some of it.

### C++ upgrade

Build on the `Fifo` class from Chapter 8 with typed, whole-array transfer helpers:

```cpp
#include <vector>
#include <stdexcept>
#include <unistd.h>

template <typename T>
void write_all(int fd, const std::vector<T>& data) {
    ssize_t expected = data.size() * sizeof(T);
    ssize_t written = write(fd, data.data(), expected);
    if (written != expected) throw std::runtime_error("short write to FIFO");
}

template <typename T>
std::vector<T> read_all(int fd, size_t count) {
    std::vector<T> data(count);
    ssize_t expected = count * sizeof(T);
    ssize_t got = read(fd, data.data(), expected);
    if (got != expected) throw std::runtime_error("short read from FIFO");
    return data;
}
```

```cpp
// generator program
Fifo fifo("sum");
int fd = fifo.open_for_write();
std::vector<int> numbers = {8, 18, 10, 89, 24};
write_all(fd, numbers);
close(fd);
```

```cpp
// reader program
Fifo fifo("sum");
int fd = fifo.open_for_read();
auto numbers = read_all<int>(fd, 5);
long total = std::accumulate(numbers.begin(), numbers.end(), 0L);
std::cout << "result is " << total << '\n';
close(fd);
```

Note `write_all`/`read_all` explicitly check the transferred byte count against what was
*expected* — the raw C version in the transcript trusts `read`/`write` to always move the
full requested size, which (as flagged in Chapter 6) isn't guaranteed by POSIX in
general, even though it usually holds true in practice for small, local, blocking FIFOs
like these.

---

## Chapter 10 — Two-Way Communication and the Self-Read Race Condition

### Concept: one pipe can't safely do both directions at once

Goal: parent generates a random number, sends it to the child; child multiplies it by 4
and sends the result back; parent prints it.

**The naive (broken) approach — one pipe, both directions:**

```c
int p1[2];
pipe(p1);
int id = fork();

if (id == 0) {
    int x;
    read(p1[0], &x, sizeof(x));
    x *= 4;
    write(p1[1], &x, sizeof(x));
} else {
    int y = rand() % 10;
    write(p1[1], &y, sizeof(y));
    read(p1[0], &y, sizeof(y));
    printf("result is %d\n", y);
}
```

This *sometimes* works and sometimes **hangs forever** or prints the wrong value. Why:
after `fork()`, **both** processes inherited **both** ends of the same single pipe. There
is no rule that says "only the process that didn't write to a pipe end may read from it."
So it's entirely possible for the **parent to write, then immediately read back its own
just-written value**, before the child ever gets scheduled to read it — the child then
blocks forever trying to read a number nobody will ever send it (the parent already
consumed it). Whether this happens depends on the OS scheduler's timing (in the
transcript, adding a slow `printf` between the write and read incidentally gave the child
enough of a scheduling window to win the race — which is a fragile, accidental fix, not a
real one).

### The real fix: two pipes, one direction each

```c
int p1[2]; // child -> parent
int p2[2]; // parent -> child
pipe(p1);
pipe(p2);

int id = fork();

if (id == 0) {
    close(p1[0]); close(p2[1]);   // child: only writes p1, only reads p2

    int x;
    read(p2[0], &x, sizeof(x));
    x *= 4;
    write(p1[1], &x, sizeof(x));

    close(p1[1]); close(p2[0]);
} else {
    close(p1[1]); close(p2[0]);   // parent: only reads p1, only writes p2

    int y = rand() % 10;
    write(p2[1], &y, sizeof(y));
    read(p1[0], &y, sizeof(y));
    printf("result is %d\n", y);

    close(p1[0]); close(p2[1]);
}
```

Now there is **no ambiguity** about who can read what: `p1` only ever flows child→parent,
`p2` only ever flows parent→child. This is guaranteed correct regardless of scheduling —
it doesn't depend on lucky timing from a `printf` call. **This is the general rule for
bidirectional IPC with pipes: you always need two unidirectional pipes, never one pipe
used both ways between the same two parties.**

### C++ upgrade

Compose two `Pipe` objects (Chapter 6) into a `Channel` type that structurally prevents
the self-read race by only exposing direction-correct methods to each side:

```cpp
class Channel {
public:
    Channel() = default;

    // Call once, before fork(), from the parent.
    struct Ends {
        Pipe to_child;    // parent writes, child reads
        Pipe to_parent;   // child writes, parent reads
    };

    static Ends create() { return Ends{}; }
};
```

```cpp
auto ends = Channel::create();
pid_t id = fork();

if (id == 0) {
    ends.to_child.close_write();
    ends.to_parent.close_read();

    int x = ends.to_child.read_value<int>();
    x *= 4;
    ends.to_parent.write_value(x);
    _exit(0);
}

ends.to_child.close_read();
ends.to_parent.close_write();

int y = rand() % 10;
ends.to_child.write_value(y);
int result = ends.to_parent.read_value<int>();
std::cout << "result is " << result << '\n';

ChildProcess(id).join();
```

**Why this design is genuinely better, not just prettier:** the `to_child` pipe's type
system usage is *directional by naming convention* — a reviewer instantly sees "parent
writes to `to_child`, child reads from `to_child`" without cross-referencing raw `p1[0]`
vs `p2[1]` indices, which is exactly the kind of off-by-one/wrong-index mistake that
causes the deadlocks this chapter is about. If you outgrow raw pipes entirely for this
kind of bidirectional request/response pattern, the natural next step (beyond this
transcript) is a Unix domain socket with `socketpair()`, which gives you one bidirectional
fd instead of managing two pipes by hand — worth knowing as the production-grade successor
to this pattern.

---

## Summary Table — Concept → Tool → When to Reach for It

| Need                                                    | Tool                          | Chapter |
|----------------------------------------------------------|-------------------------------|---------|
| Run code in a new, memory-isolated process                | `fork()`                      | 1       |
| Guarantee ordering / avoid zombies                        | `wait()` / `waitpid()`        | 2, 4    |
| Understand why parent/child variables diverge              | copy-on-write memory model    | 3       |
| Identify a process or its lineage                          | `getpid()` / `getppid()`      | 4       |
| Fan out to several children safely                         | guarded `fork()` loop / process pool | 5 |
| Send data one-way between **related** processes            | `pipe()`                       | 6, 7    |
| Send data between **unrelated** processes (own executables) | `mkfifo()` + `open()`          | 8, 9    |
| Send data **both ways** between two processes              | two pipes (or, later, a socket pair) | 10 |

### The single biggest idea to carry forward

`fork()` gives you **parallelism through isolation** — no shared memory, no data races,
at the cost of needing explicit tools (pipes/FIFOs) whenever processes must talk. Threads
(the next natural topic after processes, per the intro to this transcript) flip that
trade-off: cheap communication via shared memory, but now *you* own preventing data races
via locks/atomics. Understanding *why* processes need pipes at all is what makes threads'
shared-memory hazards make sense later — you're seeing the two ends of the same spectrum.
