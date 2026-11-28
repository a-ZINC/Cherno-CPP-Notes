# PART 6 — select()

## Chapter 6.1 — select(): Waiting on Many Descriptors with One Call

### 🧠 One-Sentence Mental Model
> `select()` is ONE blocking system call that puts your thread to sleep on a whole SET of file descriptors at once and wakes it the moment ANY of them is ready (or a timeout expires) — it does not read, write, or accept anything for you; it only answers "which of these are ready right now?", and you then do the actual I/O.

### 🧒 Explain Like I'm Five
Imagine you're waiting for three friends to call. You could stand next to phone #1 forever (and miss the others), or run around checking all three every second (exhausting). `select()` is a little bell that rings when ANY of the three phones rings, and a tag on the bell tells you which one. You sleep until the bell rings, pick up the phone that rang, and go back to sleep.

### 🌍 Real-World Analogy
A restaurant host with a row of table-buzzers. Instead of walking the floor checking every table (busy-waiting, Chapter 5.4) or assigning one waiter to stare at each table (a thread per table, Chapter 4.8), one host sits at the desk and a board lights up the tables that need attention. The host handles whichever lights are on, then looks at the board again.

### ❓ The Problem
Part 4 gave us blocking I/O: simple, but one thread is frozen per wait. Part 5 gave us nonblocking I/O: no freezing, but if you just retry in a loop you spin at ~100% CPU (Chapter 5.4 measured 97–99%). We need the missing piece Chapter 5.1 promised: **a way to sleep until "something I care about changed" across MANY descriptors, from ONE thread.**

### 🔥 Why This Problem Matters
This is the hinge of all scalable server design. `select()` is the oldest and simplest member of the family (`select` → `poll` → `epoll`, Parts 6–8) that every event loop (Part 11), nginx, Redis, Node.js and the X server / Wayland compositors (Part 11.5) are built on. Learn the idea once here, in its simplest form, and the other two are variations on it.

### 🕰 Historical Context
`select()` appeared in 4.2BSD (1983) alongside the BSD sockets API, because network programs needed to watch a socket and the keyboard at the same time. It was later standardized by POSIX (header `<sys/select.h>`). Its fixed-size bitmask design (`fd_set`, 1024 bits in glibc) reflects 1980s assumptions of a process having a few dozen descriptors — the source of the limits and costs Chapters 6.2 and 6.6 examine.

### 💡 The Naive Solution
(a) One blocking thread per descriptor, or (b) nonblocking descriptors polled in a loop.

### ❌ Why the Naive Solution Fails
(a) costs a stack and a scheduler slot per connection and makes shared state need locks (Chapters 4.8–4.9). (b) burns a whole CPU doing nothing useful (Chapter 5.4).

### ✅ The Better Solution
Let the kernel do the waiting for the whole set in a single call. Mechanically (Chapter 4.2's sleep sequence, generalized): the kernel checks each descriptor in your sets; if none is ready it puts the thread on the wait queue of EVERY one of them, sleeps, and any one of them waking it ends the call.

### 🧠 Core Concept
> **`select()` converts "N waits" into "1 wait over a set." You give it up to three sets — descriptors you want to READ from, WRITE to, and watch for EXCEPTIONAL conditions — plus a timeout. It blocks until at least one descriptor in the sets is ready or the timeout expires, then OVERWRITES your sets to contain only the ready ones and returns how many there are.**

### 📐 Deep Technical Explanation

#### 1. Syntax first (read this before the code)
```cpp
#include <sys/select.h>

int select(int nfds,
           fd_set *readfds,      // watch these for "read() would not block"
           fd_set *writefds,     // watch these for "write() would not block"
           fd_set *exceptfds,    // watch these for "exceptional condition" (rare: TCP urgent data)
           struct timeval *timeout);

// fd_set is an opaque bitmask. You never touch its bits directly; you use four macros:
void FD_ZERO(fd_set *set);              // clear every bit              -> "empty set"
void FD_SET(int fd, fd_set *set);       // turn on the bit for fd       -> "add fd"
void FD_CLR(int fd, fd_set *set);       // turn off the bit for fd      -> "remove fd"
int  FD_ISSET(int fd, fd_set *set);     // is fd's bit on?              -> "is fd in the set / is it ready?"

struct timeval { time_t tv_sec; suseconds_t tv_usec; };   // seconds + microseconds
```

| Parameter | Meaning | Common mistake |
|---|---|---|
| `nfds` | **highest-numbered fd in any set, PLUS ONE** | passing the *count* of fds, or forgetting the `+1` (Chapter 6.5) |
| `readfds` / `writefds` / `exceptfds` | input: what to watch. **Output: what is ready.** Pass `NULL` for a set you don't need | assuming they are unchanged after the call (they are not — Experiment 2) |
| `timeout` | `NULL` = wait forever; `{0,0}` = check and return immediately; otherwise wait at most that long | assuming it keeps its value (Linux reports the remaining time back; portable code must re-initialize it every call) |

**Return value**
- `> 0` — number of descriptors that are ready (summed over the three sets); inspect with `FD_ISSET`.
- `0` — the timeout expired with nothing ready.
- `-1` — error; check `errno`: `EINTR` (a signal arrived — not a failure, just retry), `EBADF` (some fd in a set is not open), `EINVAL` (bad `nfds` or timeout), `ENOMEM`.

#### 2. What "ready" means (the contract)
| Set | An fd is in it when… |
|---|---|
| **read** | a `read()`/`recv()` would **not block**: data is waiting, **or the peer closed (EOF)**, or an error is pending; for a **listening socket**, a connection is waiting to be `accept()`ed |
| **write** | a `write()`/`send()` would **not block** (some buffer space exists); for a socket being **connected**, the connect has finished — success *or* failure (Chapter 5.7) |
| **except** | an *exceptional condition* — in practice TCP out-of-band/urgent data. **It is NOT "errors."** Most programs pass `NULL` |

Two rules every later chapter relies on:
1. **Ready means "the call won't block", not "the call will transfer everything you ask."** A `write()` of 1 MB on a "writable" socket may still be partial (Chapter 5.3).
2. **`select()` only reports; you still do the I/O.** And because readiness can occasionally be stale or spurious, real programs keep the descriptors nonblocking (Part 5) so a surprise can never freeze the loop.

#### 3. The canonical loop (skeleton)
```cpp
for (;;) {
    fd_set rs; FD_ZERO(&rs);                 // 1. REBUILD the set every iteration (select destroys it)
    int maxfd = -1;
    for (int fd : watched) { FD_SET(fd, &rs); maxfd = std::max(maxfd, fd); }

    timeval tv{2, 0};                         // 2. re-initialize the timeout every iteration too
    int n = select(maxfd + 1, &rs, nullptr, nullptr, &tv);   // 3. ONE sleep for everything
    if (n < 0) { if (errno == EINTR) continue; /* real error */ break; }
    if (n == 0) { /* timeout: do periodic work */ continue; }

    for (int fd : watched)                    // 4. find who is ready, then do the real I/O
        if (FD_ISSET(fd, &rs)) handle(fd);
}
```
This rebuild-every-time shape is the single biggest reason `select()` costs more than its successors at scale (Chapter 6.6).

#### 4. Level-triggered
`select()` reports a descriptor as ready **for as long as the condition holds**. If 5 bytes sit in a pipe and you don't read them, the next `select()` returns immediately again, and again, until you drain them (Experiment 6). This "keep telling me until it's handled" behavior is called **level-triggered**; Part 8 contrasts it with edge-triggered `epoll`.

#### 5. How the kernel does it (preview; Chapter 6.6 goes deep)
The syscall copies your sets into the kernel, then loops over descriptors `0 … nfds-1`, asking each one's driver "are you ready?" through the `poll` entry of its `file_operations` (the dispatch table from Chapter 3.4). If none is ready it registers the thread on each descriptor's wait queue and sleeps; when woken it re-scans. Cost is therefore **proportional to `nfds`**, on every call.

On this machine, glibc's `select()` is implemented with the kernel's `pselect6` system call (observed with `strace` below).

### 🔬 Lab: the first `select()` program (compiled and run here)
**Predict before running** (answers follow the output):
1. A child writes to pipe A at 200 ms, to pipe B at 500 ms, and exits at 700 ms. How many times does the parent's `select()` return before both pipes are finished, and when?
2. A pipe whose writer exited — is it "readable" in the `select()` sense?
3. The parent is idle for 700 ms. Roughly how much CPU time does it use: ~700 ms, ~70 ms, or ~1 ms?

```cpp
// select_hello.cpp -- Chapter 6.1: the first select() program.
// Build: g++ -std=c++20 -O2 -Wall -o select_hello select_hello.cpp
#include <sys/select.h>
#include <sys/resource.h>
#include <sys/wait.h>
#include <unistd.h>
#include <fcntl.h>
#include <algorithm>
#include <cerrno>
#include <chrono>
#include <cstdio>
#include <cstring>

static double now_ms() {
    using namespace std::chrono;
    return duration<double, std::milli>(steady_clock::now().time_since_epoch()).count();
}
static double cpu_ms() {                       // user+system CPU this process has used so far
    rusage r{}; getrusage(RUSAGE_SELF, &r);
    return r.ru_utime.tv_sec * 1e3 + r.ru_utime.tv_usec / 1e3 + r.ru_stime.tv_sec * 1e3 + r.ru_stime.tv_usec / 1e3;
}

int main() {
    printf("sizeof(fd_set)=%zu bytes = %zu bits, FD_SETSIZE=%d\n\n", sizeof(fd_set), sizeof(fd_set) * 8, FD_SETSIZE);

    // ---------- Experiment 1: wait on TWO pipes at once with ONE call ----------
    int a[2], b[2];
    if (pipe(a) || pipe(b)) return 1;
    pid_t pid = fork();
    if (pid == 0) {                            // child = a slow "producer": A at 200 ms, B at 500 ms, exits at 700 ms
        close(a[0]); close(b[0]);
        usleep(200000); if (write(a[1], "hello from A", 12) < 0) _exit(1);
        usleep(300000); if (write(b[1], "hi from B", 9) < 0) _exit(1);
        usleep(200000); _exit(0);              // exiting closes both write ends -> readers will see EOF
    }
    close(a[1]); close(b[1]);

    puts("Experiment 1: one select() waiting on two pipes (producer writes A@200ms, B@500ms, exits@700ms)");
    double t0 = now_ms(), c0 = cpu_ms(); int calls = 0; int rd_fd[2] = {a[0], b[0]}; const char* nm[2] = {"A", "B"};
    while (rd_fd[0] >= 0 || rd_fd[1] >= 0) {
        fd_set rs; FD_ZERO(&rs);
        int maxfd = -1;
        for (int i = 0; i < 2; i++) if (rd_fd[i] >= 0) { FD_SET(rd_fd[i], &rs); maxfd = std::max(maxfd, rd_fd[i]); }
        timeval tv{2, 0};                      // give up after 2 s
        int n = select(maxfd + 1, &rs, nullptr, nullptr, &tv); calls++;
        if (n < 0) { perror("select"); return 1; }
        printf("  t=%6.1f ms  select() returned %d  ready:", now_ms() - t0, n);
        for (int i = 0; i < 2; i++) if (rd_fd[i] >= 0 && FD_ISSET(rd_fd[i], &rs)) printf(" %s(fd %d)", nm[i], rd_fd[i]);
        puts("");
        for (int i = 0; i < 2; i++) if (rd_fd[i] >= 0 && FD_ISSET(rd_fd[i], &rs)) {
            char buf[64]; ssize_t r = read(rd_fd[i], buf, sizeof buf);
            if (r > 0) printf("             read %zd bytes from %s: \"%.*s\"\n", r, nm[i], (int)r, buf);
            else { printf("             read() == 0 on %s -> EOF (a closed pipe counts as READABLE)\n", nm[i]); close(rd_fd[i]); rd_fd[i] = -1; }
        }
    }
    printf("  total: %d select() calls, wall %.0f ms, CPU used %.2f ms\n\n", calls, now_ms() - t0, cpu_ms() - c0);
    waitpid(pid, nullptr, 0);

    // ---------- Experiment 2: timeout, and the sets get OVERWRITTEN ----------
    int c[2];
    if (pipe(c)) return 1;
    puts("Experiment 2: nothing will ever arrive; timeout = 300 ms");
    fd_set rs; FD_ZERO(&rs); FD_SET(c[0], &rs);
    printf("  before: FD_ISSET(c)=%d\n", FD_ISSET(c[0], &rs) ? 1 : 0);
    timeval tv{0, 300000}; double t1 = now_ms();
    int n = select(c[0] + 1, &rs, nullptr, nullptr, &tv);
    printf("  select() returned %d after %.1f ms; after: FD_ISSET(c)=%d  <- the set was modified: you must rebuild it every loop\n\n", n, now_ms() - t1, FD_ISSET(c[0], &rs) ? 1 : 0);

    // ---------- Experiment 3: zero timeout = a pure readiness CHECK (Chapter 5.1's idea, for a set) ----------
    puts("Experiment 3: timeout {0,0} = check and return immediately");
    FD_ZERO(&rs); FD_SET(c[0], &rs); tv = {0, 0}; t1 = now_ms();
    n = select(c[0] + 1, &rs, nullptr, nullptr, &tv);
    printf("  returned %d after %.3f ms\n\n", n, now_ms() - t1);

    // ---------- Experiment 4: write-readiness ----------
    puts("Experiment 4: is the WRITE end of an empty pipe ready for writing?");
    fd_set ws; FD_ZERO(&ws); FD_SET(c[1], &ws); tv = {1, 0}; t1 = now_ms();
    n = select(c[1] + 1, nullptr, &ws, nullptr, &tv);
    printf("  returned %d after %.3f ms, FD_ISSET(write end)=%d (yes: there is buffer space, a write would not block)\n\n", n, now_ms() - t1, FD_ISSET(c[1], &ws) ? 1 : 0);

    // fill the pipe completely (nonblocking) and ask again
    int fl = fcntl(c[1], F_GETFL); fcntl(c[1], F_SETFL, fl | O_NONBLOCK);
    char junk[4096]; memset(junk, 'x', sizeof junk); size_t filled = 0;
    while (write(c[1], junk, sizeof junk) > 0) filled += sizeof junk;
    FD_ZERO(&ws); FD_SET(c[1], &ws); tv = {0, 200000}; t1 = now_ms();
    n = select(c[1] + 1, nullptr, &ws, nullptr, &tv);
    printf("Experiment 5: pipe now FULL (%zu bytes buffered) -> select(write) returned %d after %.1f ms (timed out: not writable)\n", filled, n, now_ms() - t1);

    // ---------- Experiment 6: select is LEVEL-triggered: it keeps reporting until you actually drain the data ----------
    int d[2];
    if (pipe(d) || write(d[1], "12345", 5) != 5) return 1;
    puts("\nExperiment 6: 5 bytes sit in a pipe; ask select() three times, reading only between the 2nd and 3rd");
    for (int k = 1; k <= 3; k++) {
        if (k == 3) { char tmp[16]; if (read(d[0], tmp, sizeof tmp) < 0) return 1; puts("  (read the 5 bytes)"); }
        FD_ZERO(&rs); FD_SET(d[0], &rs); tv = {0, 0};
        n = select(d[0] + 1, &rs, nullptr, nullptr, &tv);
        printf("  ask #%d -> select() returned %d\n", k, n);
    }
    return 0;
}
```

**Build and run:** `g++ -std=c++20 -O2 -Wall -o select_hello select_hello.cpp && ./select_hello`

### 📊 Measured output (this sandbox)
```
sizeof(fd_set)=128 bytes = 1024 bits, FD_SETSIZE=1024

Experiment 1: one select() waiting on two pipes (producer writes A@200ms, B@500ms, exits@700ms)
  t= 200.5 ms  select() returned 1  ready: A(fd 3)
             read 12 bytes from A: "hello from A"
  t= 500.6 ms  select() returned 1  ready: B(fd 5)
             read 9 bytes from B: "hi from B"
  t= 701.0 ms  select() returned 2  ready: A(fd 3) B(fd 5)
             read() == 0 on A -> EOF (a closed pipe counts as READABLE)
             read() == 0 on B -> EOF (a closed pipe counts as READABLE)
  total: 3 select() calls, wall 701 ms, CPU used 0.24 ms

Experiment 2: nothing will ever arrive; timeout = 300 ms
  before: FD_ISSET(c)=1
  select() returned 0 after 300.4 ms; after: FD_ISSET(c)=0  <- the set was modified: you must rebuild it every loop

Experiment 3: timeout {0,0} = check and return immediately
  returned 0 after 0.003 ms

Experiment 4: is the WRITE end of an empty pipe ready for writing?
  returned 1 after 0.001 ms, FD_ISSET(write end)=1 (yes: there is buffer space, a write would not block)

Experiment 5: pipe now FULL (65536 bytes buffered) -> select(write) returned 0 after 200.4 ms (timed out: not writable)

Experiment 6: 5 bytes sit in a pipe; ask select() three times, reading only between the 2nd and 3rd
  ask #1 -> select() returned 1
  ask #2 -> select() returned 1
  (read the 5 bytes)
  ask #3 -> select() returned 0
```
And the real system call, observed with `strace -e trace=pselect6`:
```
pselect6(6, [3 5], NULL, NULL, {tv_sec=2, tv_nsec=0}, NULL) = 1 (in [3], left {tv_sec=1, tv_nsec=800249024})
pselect6(6, [3 5], NULL, NULL, {tv_sec=2, tv_nsec=0}, NULL) = 1 (in [5], left {tv_sec=1, tv_nsec=699724140})
```
Note `6` = highest fd (5) + 1, and `left` = the kernel writing the unused part of the timeout back.

**Answers to the predictions**
1. **Three** returns: at ≈200 ms (A), ≈500 ms (B), ≈700 ms (both, as EOF). One call watched both pipes the whole time.
2. **Yes.** At 700 ms both pipes were reported readable; `read()` then returned 0 (EOF). A closed peer counts as "readable" so you can discover it — forgetting this is a classic infinite-loop bug.
3. **About 0.24 ms** of CPU in 701 ms of wall time (≈0.03%). Compare Chapter 5.4's spin loop at 97–99% CPU. The thread was asleep inside the kernel the entire time.

**What each experiment proved**
- **Exp. 1:** one sleeping call serves several descriptors; 3 syscalls instead of millions of retries.
- **Exp. 2:** timeout returned `0` after 300.4 ms, and the set was wiped (`FD_ISSET` went 1 → 0): **sets are in/out parameters.**
- **Exp. 3:** a `{0,0}` timeout is a pure readiness *check* — about 3 µs, never sleeps.
- **Exp. 4–5:** write-readiness is about buffer space: an empty pipe is writable instantly; a pipe stuffed with 65,536 bytes is not (timed out after 200 ms). This is exactly how a server learns a slow client's outgoing buffer is full (Chapter 5.6's backpressure).
- **Exp. 6:** level-triggered: two asks, two "ready" answers; after draining, `0`.

### ❌ Common Misconceptions
- ❌ **"`select()` reads the data for me."** — No. It only reports readiness; you must still call `read()`/`recv()`.
- ❌ **"`nfds` is how many descriptors I'm watching."** — It's the **highest fd number + 1**. Watching just fd 900 requires `nfds = 901`, and the kernel scans every number below it.
- ❌ **"My `fd_set` is still valid for the next call."** — It is overwritten; rebuild it every iteration (Experiment 2).
- ❌ **"The except set reports errors."** — It reports out-of-band/urgent conditions. Errors and hangups show up as *readable* (and writable), and your next `read()`/`write()` returns the actual error.
- ❌ **"Readable means there is data."** — Also true for EOF and for a pending error; always check what `read()` returns (0 = EOF).
- ❌ **"A writable fd can take everything I write."** — Only "won't block for at least some bytes." Handle partial writes (Chapter 5.3).
- ❌ **"select() is obsolete and useless."** — It's slow at scale, but it is portable everywhere, trivially simple, and perfectly good for a handful of descriptors (a terminal plus a socket, a daemon with a few pipes).

### 🧙 Wizard Insight
`select()` doesn't make I/O faster; it makes **waiting** cheaper and *centralized*. The three-line idea — "ask the kernel which of these are ready, sleep if none" — is the whole foundation of event-driven programming. Everything that follows (`poll`, `epoll`, `kqueue`, `io_uring`'s readiness mode, every event loop) is the same question asked with a better data structure.

### 🧠 Quiz
**Q1.** Does `select()` return the data that arrived?
<details><summary>Answer</summary>No. It returns only the number of ready descriptors and modifies your sets to mark them. You then call read()/recv() yourself.</details>

**Q2.** You watch fds 4, 9 and 12. What `nfds` do you pass?
<details><summary>Answer</summary>13 (highest fd + 1). Passing 3 (the count) would make the kernel ignore fds ≥ 3 entirely.</details>

**Q3.** After `select()` returns 0, what do your `fd_set`s contain?
<details><summary>Answer</summary>They are cleared (no ready descriptors). If you reuse them in the next call without rebuilding, you will be watching nothing.</details>

**Q4.** A peer closes its end of a socket. Which set reports it, and what does read() return?
<details><summary>Answer</summary>The read set (the fd becomes readable), and read()/recv() returns 0 meaning EOF.</details>

**Q5.** If you don't read the data after select() reports readable, what does the next select() do?
<details><summary>Answer</summary>Returns immediately again with the same fd ready — select is level-triggered.</details>

### 📌 Short Notes (Quick Reference)
- `select(nfds, &read, &write, &except, &timeout)`: sleep on a SET of fds until one is ready or timeout; returns count ready, 0 on timeout, -1 on error.
- `nfds` = highest fd + 1. Sets are overwritten on return → rebuild every loop. Re-initialize the timeout every call.
- Macros: `FD_ZERO`, `FD_SET`, `FD_CLR`, `FD_ISSET`. `fd_set` is 1024 bits (128 bytes) here.
- Read-ready includes EOF, errors, and a pending `accept()`; write-ready means buffer space (or connect finished); except-ready means urgent data, not errors.
- Level-triggered: keeps reporting until you drain/handle.
- You still do the I/O; keep fds nonblocking so stale readiness can't hang the loop.
- Idle cost is near zero (0.24 ms CPU in 701 ms measured) — the thread sleeps in the kernel.
- On Linux/glibc, `select()` is carried out by the `pselect6` syscall.

### 🔗 What This Connects To Next
**Previous:** Part 5, Chapter 5.7 — Nonblocking `connect()` (end of Part 5)
**Current:** Part 6, Chapter 6.1 — `select()`: Waiting on Many Descriptors with One Call
**Next:** Part 6, Chapter 6.2 — `fd_set` Internals: bitmasks, `FD_SETSIZE`, the 1024 limit, and why `nfds` exists
