# Debugging Wizard — Phase 0, Chapter 4
# Error Handling and Logging

> **How to use this note** (same as Chapters 1-3)
> 1. Read the short concept, then answer 🧠 THINK / 🔬 PREDICT **on paper first**.
> 2. Only then open the **✅ Solution**.
> 3. Run the experiment and record **your** numbers.
> 4. Ask me follow-up questions any time — they go in the **Clarification Log** at the end.
>
> **All code is in this note.** Every new file is listed in full in **Appendix A**.
>
> **Honesty note.** Every program and test in this chapter was compiled with the strict warning flags (zero warnings), run plain and under ASan/UBSan, and every number quoted below came from a real run — including the exact compiler messages in Step 4.2. CMake itself was not run in my sandbox — your first `cmake --preset debug` after adding this chapter is its real test. This chapter **does** add to `src/`: a new `Result<T>`-based file reader alongside Chapter 2's existing `read_file`, and a small logger — both additive, so every Chapter 1-3 program still compiles unchanged (verified).

---

## 0. Chapter Overview

### 0.1 What You Will Learn

- **Explain** why `dw_core`'s Chapter 1-3 pattern — return `bool`, write the real answer through an out-parameter — lets a caller silently ignore failure.
- **Use** `dw::Result<T>`, a small `[[nodiscard]]` type that turns "forgot to check" into a **compile error** under `DW_WERROR`, verified with the actual compiler output.
- **Reproduce** a real errno-clobbering bug: report the *wrong* error, consistently, then fix it with a one-line capture.
- **Distinguish** `ENOENT` from `EACCES` from other failures — not just "it failed" — using a real `Result`-returning reader tested against both.
- **Use** a minimal leveled logger, and explain a limitation it still has (not thread-safe) that the project will have to revisit.

### 0.2 Prerequisites

Chapters 1-3 complete. `dw_core` (Chapter 2) and `DW_WERROR` (Chapter 1) already exist.

### 0.3 Why This Matters to Debugging Wizard

Chapters 1-3 built instruments that read `/proc` and diagnose memory bugs. Every one of those instruments can itself fail — a process vanishes mid-read, a permission is denied, a file format changes. So far, failure has meant "print to `stderr` and `return 1`" from `main`. A tool meant to run for hours, watching *other* processes that can disappear at any moment (Phase 2's whole premise), needs a real answer to "what happened, and can I tell the difference between reasons?" This chapter builds that answer, and gives Debugging Wizard its first way to *report* findings instead of just printing numbers to a terminal.

### 0.4 Step Map

| Step | Focus |
|------|-------|
| 4.1 | Why `bool` + out-parameter lets errors vanish |
| 4.2 | `[[nodiscard]]`: making "forgot to check" a compile error |
| 4.3 | The errno race: reporting confidently, and wrongly |
| 4.4 | `dw::Result<T>` in `dw_core`: `read_file_result` |
| 4.5 | A minimal logger |
| 4.6 | Explain-it-back, knowledge check, completion |

### 0.5 Project Conventions (unchanged)

Code lives in the note. Predict before you run. Change one variable at a time. Shell for short glue only; measurement code is C++. Record the machine in `docs/environment.md`.

---

## Step 4.1 — Why `bool` + Out-Parameter Lets Errors Vanish

Chapter 2's `dw::read_file` has this shape:

```cpp
bool read_file(const char* path, std::string& out);
```

It works, and every test in Chapters 1-3 passed. But look at how easy it is to call **wrong**:

```cpp
std::string text;
read_file("/proc/self/status", text);   // return value not checked
// ... text is used here, possibly empty or stale, and nothing said so
```

This compiles with **zero warnings** under `-Wall -Wextra -Wpedantic -Wshadow` — the exact flags this whole project builds with.

### 🧠 THINK

1. `read_file` already follows good discipline internally (Chapter 2: loop until EOF, retry on `EINTR`, fail loudly on a bad format). Given that, what's actually wrong with its *interface*?
2. A `bool` return **can** be checked. Why doesn't "the information is technically available" solve the problem in practice?
3. `dw::MemSnapshot` fields default to `-1` for "missing," a design Chapter 2 specifically chose over `0`. Does the `bool` return of `read_mem_snapshot` have an equivalent way to say *why* it failed?

<details>
<summary>✅ Solution</summary>

1. The **implementation** is careful; the **interface** makes that care optional for the caller. Nothing stops a caller from ignoring the return value, and the compiler has no way to know that ignoring it is a mistake here specifically (as opposed to, say, ignoring the return value of `printf`, which is usually fine).
2. "Can be checked" and "will be checked" are different properties. A tool built to run for hours, written under deadline pressure, edited by someone unfamiliar with the codebase — any of these make an *optional* check something that eventually gets skipped. The project's own rule from Chapter 1 applies here too: relying on discipline instead of a mechanism is how bugs survive review.
3. **No.** `bool` collapses every possible failure — file doesn't exist, no permission, truncated read, bad format — into the same single bit: `false`. Chapter 2's `-1` sentinel distinguishes "missing" from "zero" for one *field*; nothing in the current interface distinguishes one *kind of failure* from another for the whole call.

</details>

---

## Step 4.2 — `[[nodiscard]]`: Making "Forgot to Check" a Compile Error

`dw::Result<T>` (Appendix A.1) is a small type that holds either a `T` or an `Error`, marked `[[nodiscard]]`:

```cpp
struct Error {
    int code = 0;          // usually errno
    std::string message;   // human-readable context
};

template <typename T>
class [[nodiscard]] Result {
public:
    static Result success(T value);
    static Result failure(Error e);
    static Result failure(int code, std::string message);

    bool ok() const noexcept;
    explicit operator bool() const noexcept;
    const T& value() const&;     // throws std::bad_variant_access if !ok()
    const Error& error() const&;
    T value_or(T fallback) const&;
};
```

`[[nodiscard]]` is a **language attribute**, not a warning flag — it fires with **no** `-W` flags at all, and every compiler in this project's toolchain understands it.

### 🔬 PREDICT

Given this file:

```cpp
#include "dw/proc_self.hpp"

int main() {
    dw::read_file_result("/proc/self/status");  // return value dropped
    return 0;
}
```

1. Does `g++ -std=c++17 -c` (no `-W` flags at all) warn about this?
2. Does adding `DW_WERROR` (i.e. `-Werror`) turn it into a build **failure**, or just a louder warning?
3. What does the fixed version — one that checks the result — need to look like to compile silently under `-Werror`?

<details>
<summary>✅ Solution</summary>

1. **Yes**, with zero `-W` flags:
   ```
   warning: ignoring returned value of type 'dw::Result<std::string>',
   declared with attribute 'nodiscard' [-Wunused-result]
   ```
   This is verified, exact compiler output. `[[nodiscard]]` is checked independent of `-Wall`.
2. **A build failure**, verified:
   ```
   error: ignoring returned value of type 'dw::Result<std::string>',
   declared with attribute 'nodiscard' [-Werror=unused-result]
   cc1plus: all warnings being treated as errors
   ```
   exit code 1. This is Chapter 1's `DW_WERROR` option doing exactly its job — `cmake --preset debug -DDW_WERROR=ON` would now refuse to build any file that drops a `Result`.
3. **Actually use the return value** — assign it to a variable and check it:
   ```cpp
   auto r = dw::read_file_result("/proc/self/status");
   if (!r) {
       std::fprintf(stderr, "read failed: %s\n", dw::to_string(r.error()).c_str());
       return 1;
   }
   ```
   Verified: compiles with **zero** warnings under the same `-Werror` build.

</details>

### 🧠 THINK

`bool` return values (like Chapter 2's `read_file`) are **not** `[[nodiscard]]`-protected in this codebase, but `Result<T>` is. Why not simply mark every function `[[nodiscard]]`, `bool`-returning ones included?

<details>
<summary>✅ Solution</summary>

Nothing stops you from adding `[[nodiscard]]` to a `bool`-returning function too — some codebases do exactly that. The reason this project introduces `Result<T>` instead of just decorating the old signature: `[[nodiscard]]` only solves *"did you look?"*, not *"what did you learn?"*. A checked `bool` still only tells you pass/fail. `Result<T>` solves both at once — the type change is what makes `.error().code` and `.error().message` available at all, and marking it `[[nodiscard]]` closes the other gap for free. Chapter 2's `read_file` is left as-is deliberately (see Step 4.4) rather than silently changed, so nothing already written against it breaks.

</details>

---

## Step 4.3 — The errno Race: Reporting Confidently, and Wrongly

`errno` is a single, per-thread variable that **any** failing libc or syscall wrapper can overwrite — including ones you call *after* the failure you actually care about, while getting ready to report it.

📄 **Full file:** Appendix A.4 (`lab/errno_race.cpp`).

```cpp
void buggy_version() {
    int fd = ::open("/proc/999999/status", O_RDONLY);   // fails: ENOENT
    if (fd < 0) {
        std::fprintf(stderr, "about to report the error...\n");
        ::open("/proc/1/maps", O_RDONLY);                // ALSO fails: EACCES here
        std::fprintf(stderr, "failed: %s\n", std::strerror(errno));  // reads errno NOW
    }
}
```

```bash
./build/debug/lab/errno_race
```

### 🔬 PREDICT

1. The file `/proc/999999/status` doesn't exist (`ENOENT`). What error message do you expect `buggy_version` to print?
2. Is this reproducible every time, or does it depend on scheduling/timing luck?
3. What's the minimal fix?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data** (3 repeats, this sandbox — where `/proc/1/maps` gives `EACCES` due to a container `ptrace` restriction, see `linux-notes/proc-and-sys/proc-self-fd.md`'s neighbor notes on permission checks):
> ```
> [buggy] open of /proc/999999/status failed: Permission denied
> [fixed] open of /proc/999999/status failed: No such file or directory
> ```
> Identical across all 3 runs.

1. You'd reasonably expect **"No such file or directory"** (`ENOENT`), since that's the call that actually failed.
2. **It prints the wrong message — "Permission denied" — every single time**, not intermittently. This is not a race in the threading sense (no two threads involved); it's deterministic given the same two syscalls run in the same order. The intervening `open("/proc/1/maps", ...)` also fails, with `EACCES`, and **that** is the errno value still sitting there when `strerror(errno)` finally runs. The bug is real and 100% reproducible, which is precisely why it's dangerous: it doesn't look flaky, it looks like confident, wrong output.
3. Capture `errno` into a local **immediately** after the call that can set it — before any other call, including logging calls, runs:
   ```cpp
   const int saved_errno = errno;   // right here, nothing else in between
   ```
   Verified: the fixed version correctly prints "No such file or directory" in all 3 runs, regardless of what the intervening `open()` does.

</details>

### 🤔 EXPLAIN

`Result<T>::failure(...)` in Step 4.2 takes an explicit `int code` argument rather than reading a global `errno` internally. Connect this design choice directly to the bug you just reproduced.

<details>
<summary>✅ Solution</summary>

If `Result::failure()` read `errno` internally instead of taking it as a parameter, it would reintroduce exactly this bug at one remove — the *moment* you construct the `Result` still has to be immediately after the failing call, and if `Result::failure()`'s own constructor does any work that could touch `errno` first (allocating the `std::string` message, for instance — allocation can, rarely, fail and touch errno on some paths), you're back to the same race. Requiring the caller to pass `code` explicitly forces the capture to happen at the *call site*, right next to the syscall, which is exactly where Step 4.4's `read_file_result` does it: `const int saved_errno = errno;` is the very next line after `::open(...)` returns.

</details>

---

## Step 4.4 — `dw::Result<T>` in `dw_core`: `read_file_result`

`include/dw/proc_self.hpp` gains a second file reader, **alongside** Chapter 2's `read_file` — not replacing it, so nothing already written against it needs to change:

```cpp
bool read_file(const char* path, std::string& out);              // Chapter 2, unchanged
Result<std::string> read_file_result(const char* path);          // new
```

📄 **Full files:** Appendix A.2 (updated `proc_self.hpp`), A.3 (updated `proc_self.cpp`).

```bash
cmake --build --preset debug -j
ctest --preset debug
```

### 🔬 PREDICT

Given what Step 4.3 just taught about capturing `errno` immediately:

1. Reading `/proc/999999/status` (doesn't exist) — what `code` and roughly what `message` do you expect?
2. Reading `/proc/1/maps` in this sandbox (Step 4.3's other failing case) — same question.
3. Do the two failures produce **different** `code` values, proving the function actually distinguishes them rather than collapsing everything to "failed"?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data (verified, direct calls):**
> ```
> read_file_result("/proc/999999/status"):
>   code=2  message="open failed: /proc/999999/status: No such file or directory"
>
> read_file_result("/proc/1/maps"):
>   code=13 message="open failed: /proc/1/maps: Permission denied"
> ```

1. `code=2` (`ENOENT`), message naming the path and "No such file or directory."
2. `code=13` (`EACCES`), message naming the path and "Permission denied."
3. **Yes — 2 versus 13, genuinely different**, confirmed by `tests/result_and_log_test.cpp`'s check 5. This is the entire point: a caller can now write `if (r.error().code == ENOENT) { /* handle missing */ } else { /* handle something else */ }` instead of only ever knowing "it didn't work."

</details>

### 🧠 THINK

The test for this (`result_and_log_test.cpp`, check 5) doesn't hard-assert that `/proc/1/maps` fails — it only checks the error code *if* the read failed. Why not just assert `EACCES` directly, given that's what this sandbox produces?

<details>
<summary>✅ Solution</summary>

Whether `/proc/1/maps` is readable depends on the **environment** — container restrictions, whether the test runs as root without a restrictive seccomp profile, ptrace policy (`/proc/sys/kernel/yama/ptrace_scope`) — not on anything `read_file_result` controls. A test that hard-codes "this must be `EACCES`" would fail on a machine where it happens to succeed, for reasons having nothing to do with a real bug. The test checks what `read_file_result` is actually responsible for (correctly reporting *whichever* error occurs, and that it differs from the unrelated `ENOENT` case) without asserting a fact about the machine it happens to run on. This is the same discipline as Chapter 2's permission-check exercises: predict, but verify facts about your own machine before asserting them as universal.

</details>

---

## Step 4.5 — A Minimal Logger

`dw::log_impl` (call it through the `DW_LOG_*` macros) writes leveled, timestamped messages to `stderr`, with cheap filtering:

```cpp
DW_LOG_DEBUG("a debug message, arg=%d", 7);
DW_LOG_WARN("a warning: %s", "something's off");
```

📄 **Full files:** Appendix A.5 (`log.hpp`), A.6 (`log.cpp`).

```bash
./build/debug/lab/errno_race   # unrelated to logging, but already built -- or use the test below
ctest --preset debug -R result_and_log_test --output-on-failure
```

### 🔬 PREDICT

1. Four calls, one at each level (`Debug`, `Info`, `Warn`, `Error`), with the level left at its **true default** — no call to `set_log_level` at all. How many print, and which?
2. After calling `dw::set_log_level(dw::LogLevel::Warn)`, then logging one `Debug`, one `Info`, and one `Warn` message — how many print now?
3. `log_impl` takes `...` (varargs) and is declared with `__attribute__((format(printf, 4, 5)))`. What does that buy you that a plain varargs function doesn't?

<details>
<summary>✅ Solution</summary>

> **📊 Reference data — true default, no `set_log_level` call (verified):**
> ```
> [INFO ] probe.cpp:5: an info message
> [WARN ] probe.cpp:6: a warning: something's off
> [ERROR] probe.cpp:7: an error: code=42
> ```
> **📊 Reference data — level raised to `Warn`, then Debug/Info/Warn logged (verified):**
> ```
> [WARN ] probe.cpp:30: should still print
> ```

1. **3 of the 4 print — `Info`, `Warn`, `Error`.** The default level is `LogLevel::Info`, and `Debug` sits below it, so the `Debug` call is silently dropped at zero cost (one atomic load, no formatting, no `fprintf`). This matches `log.cpp`'s declared default (`std::atomic<LogLevel> g_level{LogLevel::Info};`) exactly.
2. **Only the `Warn` message** — verified: after raising the level, both the `Debug` and `Info` calls produced no output at all, and the `Warn` call printed normally.
3. GCC/Clang then check your format string against your actual arguments **at compile time**, the same way they do for `printf` itself — a mismatched `%d` against a `std::string` argument becomes a compiler warning (and under `DW_WERROR`, a build failure) instead of a runtime surprise. Exercise 3 asks you to trigger this yourself.

</details>

### 🧠 THINK

The logger's implementation comment says: *"NOT thread-safe yet: two threads logging at once can interleave their fprintf output."* Given that this project has already discussed adding threads later (the sampling engine, Phase 8's performance lab), why write a logger with a known limitation instead of making it thread-safe now?

<details>
<summary>✅ Solution</summary>

This is the project's own rule, applied to its own tooling: **complexity must be earned through evidence**, not added because it might be needed. Nothing in Phase 0 is multithreaded yet, so a lock or a per-thread buffer would be complexity with no current benefit and a real cost — contention, another thing to get subtly wrong, another thing to test. The honest move is what the comment does: state the limitation plainly, in the code, so it's found by design instead of by surprise later. When Phase 8 (or earlier) actually introduces threads, fixing this logger is one of the concrete, measurable tasks that phase's own THINK questions will walk through — with a real workload to justify *how* to fix it (a mutex? a lock-free ring buffer? that choice should be measured, not guessed now).

</details>

---

## Step 4.6 — Explain It Back, Knowledge Check, Completion

### 🎯 Explain-it-back (no notes)

> Explain the errno-race bug from Step 4.3 to someone who has never seen it: what does `errno` actually guarantee, what does the buggy code assume that isn't guaranteed, and why does the fix work?

<details>
<summary>✅ Model answer</summary>

`errno` is a single per-thread variable that a failing library or system call sets to explain *its own* failure. It guarantees nothing about calls made *after* it — any subsequent call, including an unrelated one made while preparing to report the first error, can overwrite it if that call also fails. The buggy code assumes `errno` stays put until you get around to reading it, which is true only if literally nothing else runs in between — an assumption that's easy to violate by adding "just one more check" or a logging call before the final report. The fix works because it reads `errno` into a local variable on the very next line after the failing call, before anything else has a chance to run, so what gets reported is frozen at the moment it was actually true.

</details>

### Knowledge Check

**Level 1: Recall.** What does `[[nodiscard]]` do, and does it require `-Wall` to take effect?

<details><summary>✅ Solution</summary>It makes the compiler warn when a function's return value is discarded without being used. Verified: it fires with zero <code>-W</code> flags at all — it's a language attribute checked independently of warning-flag groups, not a member of <code>-Wall</code>.</details>

**Level 2: Understanding.** Why does `Result<T>::failure` take `int code` as an explicit parameter instead of reading `errno` itself inside the function?

<details><summary>✅ Solution</summary>To force the capture to happen at the call site, immediately after the failing syscall — exactly where <code>read_file_result</code> does it (<code>const int saved_errno = errno;</code> on the very next line). If <code>Result::failure</code> read <code>errno</code> internally, the same race from Step 4.3 could reappear one level removed, inside the constructor's own work.</details>

**Level 3: Mechanism.** In Step 4.3's buggy version, exactly which call set the `errno` value that finally got printed?

<details><summary>✅ Solution</summary>The <em>second</em> <code>open("/proc/1/maps", O_RDONLY)</code> call — the "let me also check this" step added between the real failure and the point where <code>strerror(errno)</code> finally reads it. Verified: it set <code>errno</code> to 13 (EACCES) in this sandbox, which is what printed, even though the call actually being reported on was the first <code>open()</code>, which had set <code>errno</code> to 2 (ENOENT).</details>

**Level 4: Experiment.** How would you prove, without reading the source, that `read_file_result`'s two failure cases in Step 4.4 are genuinely distinguished rather than both just mapping to some generic "failed" code?

<details><summary>✅ Solution</summary>Call it against both paths and compare <code>.error().code</code> directly, exactly what <code>result_and_log_test.cpp</code>'s check 5 does: if the two codes are equal, the function is collapsing distinct failures into one signal (Step 4.1's original problem); if they differ — verified here as 2 vs 13 — it's preserving the distinction.</details>

**Level 5: Debugging.** A caller checks `if (!result) { report generic failure }` and never looks at `.error().code`. Is this a bug in `Result<T>`, in `read_file_result`, or somewhere else?

<details><summary>✅ Solution</summary>Neither — <code>Result&lt;T&gt;</code>'s <code>[[nodiscard]]</code> only guarantees the return value is looked at, not that every field of it is used well. This caller has satisfied the compiler (no warning, no <code>-Werror</code> failure) while still discarding real information. It's a design gap in <em>that calling code</em>, not a flaw in the type — the same way a checked <code>bool</code> in Step 4.1 satisfies "did you check" without satisfying "did you learn anything." <code>[[nodiscard]]</code> raises the floor; it doesn't set the ceiling.</details>

**Level 6: Systems reasoning.** This chapter added `Result<T>` as a *second* way to report failure, alongside Chapter 2's `bool`-returning functions, rather than converting everything at once. Chapter 3's bug-signature table made a similar point about needing more than one fact to reach a conclusion. What property do these two design choices share?

<details><summary>✅ Solution</summary>
Both resist collapsing distinct information into one signal too early. Chapter 3 showed that a single <code>RssAnon</code> snapshot can't distinguish leak from growth from fragmentation — you need the trend, or an external check, not just one number treated as a verdict. This chapter shows the parallel problem on the error-handling side: a <code>bool</code> collapses every kind of failure into one bit, and converting <em>everything</em> to <code>Result&lt;T&gt;</code> in one pass, under time pressure, risks doing the conversion hastily and losing exactly the distinctions (ENOENT vs EACCES) that make it worth doing at all. Keeping both interfaces lets each caller migrate deliberately, verified one at a time — the same "earn complexity, don't jump to Version G" discipline the whole project runs on, applied here to the project's own code instead of to `/proc`.
</details>

### Exercises

1. **Convert one caller.** Pick one program from Chapter 2 or 3 that currently calls `dw::read_file` (for example inside `read_mem_snapshot` itself) and change it to use `read_file_result` instead, handling the `Error` explicitly. Does the behavior change for any existing test?
2. **A third failure mode.** `read_file_result`'s current error paths are `open()` failing and `read()` failing. Find a way to make `read()` fail with something other than a permission or existence problem (hint: a file that changes type between `open()` and `read()`) and see what `code` comes out.
3. **Format-string safety.** Deliberately call `DW_LOG_INFO("value: %s", 42)` (a `%s` with an `int` argument). What does the compiler say, and does it say it under **zero** `-W` flags, some, or only under `-Wall`?
4. **Make the logger thread-safe, and measure the cost.** Add a `std::mutex` around the `fprintf` calls in `log_impl`, then use `run_measure` (Chapter 1) to compare logging throughput before and after, single-threaded. Is there a measurable cost even with no contention?
5. **`value_or` in practice.** Find a real spot in this chapter's code where `Result<T>::value_or` would be a reasonable simplification over an explicit `if (!r) { ... } else { ... }`, and explain why it is, or isn't, actually a good idea there.

---

## Chapter Completion Criteria

I can:

- [ ] Explain why `bool` + out-parameter lets a caller ignore failure with zero warnings
- [ ] Explain what `[[nodiscard]]` guarantees and reproduce the exact warning-vs-error behavior under `DW_WERROR`
- [ ] Reproduce the errno-race bug, explain exactly which call clobbers `errno`, and fix it with an immediate capture
- [ ] Explain why `Result::failure` takes `code` as a parameter instead of reading `errno` internally
- [ ] Show that `read_file_result` distinguishes `ENOENT` from `EACCES` with real, different `.error().code` values
- [ ] Explain the logger's level filtering and its known thread-safety limitation, and why that limitation was left in deliberately
- [ ] Explain why this chapter kept `read_file` and added `read_file_result` rather than replacing the old function
- [ ] Answer all six knowledge-check levels without opening the solutions

---

## What This Unlocks Next

**Chapter 5 (testing and benchmarking).** This chapter's tests (`result_and_log_test.cpp`) followed the same hand-rolled `CHECK` macro pattern as Chapters 1-2. Chapter 5 turns that pattern into a real, reusable convention — and gives the benchmark style Chapter 2 introduced (`bench/`, self-contained `chrono`+`getrusage` timing) a proper home alongside it, so every later phase stops hand-rolling one.

---

## Clarification Log

> Ask me anything about any step. Each question and answer can be appended here so this file becomes your complete study record.

| # | Step | Your question | Answer summary |
|---|------|---------------|----------------|
| 1 | | | |
| 2 | | | |
| 3 | | | |

---

## Appendix A — Complete Code, File by File

Everything new or changed in this chapter is on this page; no download is needed. Each listing is the exact file that was compiled and tested while preparing this note (the CMake files excepted, see the honesty note at the top).

Create or replace each file at the path in its heading.

### A.1 `include/dw/result.hpp`

**New.** `dw::Result<T>` and `dw::Error`: the `[[nodiscard]]` result type.

```cpp
#pragma once

#include <string>
#include <utility>
#include <variant>

namespace dw {

// A syscall-shaped error: a numeric code (usually errno, 0 if not
// syscall-related) plus a human-readable message giving context a bare
// errno cannot ("opening /proc/1/maps failed", not just "Permission
// denied" -- strerror alone never says WHICH file).
struct Error {
    int code = 0;
    std::string message;
};

// Formats an Error the way you'd want it printed: just the message, since by
// convention (see dw::read_file_result) the message already embeds
// strerror(code) where that's meaningful. code==0 means "not a syscall
// failure"; the message alone should explain it.
inline std::string to_string(const Error& e) { return e.message; }

// Result<T>: either a T, or an Error explaining why there isn't one.
// [[nodiscard]] makes the compiler warn -- and with DW_WERROR, FAIL THE
// BUILD -- if a caller drops a Result without checking it. This replaces
// dw_core's Chapter 1-3 pattern of "return bool, write the real answer
// through an out-parameter", which a caller can silently ignore (the bool
// return by itself is NOT nodiscard-enforced the same way a whole result
// object can be).
template <typename T>
class [[nodiscard]] Result {
public:
    static Result success(T value) { return Result(std::in_place_index<0>, std::move(value)); }
    static Result failure(Error e) { return Result(std::in_place_index<1>, std::move(e)); }
    static Result failure(int code, std::string message) {
        return failure(Error{code, std::move(message)});
    }

    bool ok() const noexcept { return data_.index() == 0; }
    explicit operator bool() const noexcept { return ok(); }

    // Precondition: ok(). Calling these on a failed Result throws
    // std::bad_variant_access -- loud, not a silent wrong-branch read.
    const T& value() const& { return std::get<0>(data_); }
    T&& value() && { return std::get<0>(std::move(data_)); }

    const Error& error() const& { return std::get<1>(data_); }

    // Never throws: returns `fallback` instead of the value on failure.
    T value_or(T fallback) const& {
        return ok() ? std::get<0>(data_) : std::move(fallback);
    }

private:
    Result(std::in_place_index_t<0>, T v) : data_(std::in_place_index<0>, std::move(v)) {}
    Result(std::in_place_index_t<1>, Error e) : data_(std::in_place_index<1>, std::move(e)) {}

    std::variant<T, Error> data_;
};

}  // namespace dw
```

### A.2 `include/dw/proc_self.hpp`

**Updated (Chapter 2 → 4).** Adds `read_file_result`, includes `dw/result.hpp`; `read_file` and every Chapter 2 declaration are unchanged.

```cpp
#pragma once

#include <string>
#include <vector>

#include "dw/result.hpp"

namespace dw {

// Read a whole (small) file. Loops until end-of-file and retries on EINTR.
// Returns false on any error; `out` is then unspecified.
//
// Chapter 4 note: this bool-returning form is kept, unchanged, because
// Chapters 1-3's code calls it. See read_file_result below for the
// Result<T>-based form Chapter 4 introduces, which reports WHY a read
// failed (ENOENT vs EACCES vs something else) instead of just that it did.
bool read_file(const char* path, std::string& out);

// Same job as read_file, but reports what went wrong. errno is captured
// IMMEDIATELY after the failing call (see Chapter 4, Step 4.3's errno-race
// experiment for why "immediately" is load-bearing, not a style choice).
Result<std::string> read_file_result(const char* path);

// A snapshot of THIS process's memory accounting, as the kernel reports it.
// A field is -1 when the kernel did not provide it.
struct MemSnapshot {
    long vm_size_kb = -1;    // VmSize: total virtual address space ("VSZ")
    long vm_rss_kb = -1;     // VmRSS: pages resident in RAM ("RSS")
    long vm_hwm_kb = -1;     // VmHWM: peak RSS so far
    long rss_anon_kb = -1;   // resident anonymous pages (heap, stack, anonymous mmap)
    long rss_file_kb = -1;   // resident file-backed pages (code, libraries, mapped files)
    long rss_shmem_kb = -1;  // resident shared-memory pages
    long threads = -1;
    long minor_faults = -1;  // page faults served without disk I/O (getrusage)
    long major_faults = -1;  // page faults that needed disk I/O    (getrusage)
};

bool read_mem_snapshot(MemSnapshot& out);

// One line of /proc/self/maps.
struct MapEntry {
    unsigned long start = 0;
    unsigned long end = 0;
    std::string perms;  // for example "r-xp"
    unsigned long offset = 0;
    unsigned long inode = 0;
    std::string path;  // "" for anonymous mappings, else a file or "[heap]", "[stack]", ...

    unsigned long size_kb() const { return (end - start) / 1024; }
};

bool read_maps(std::vector<MapEntry>& out);

// The mapping that contains `addr`, or nullptr.
const MapEntry* find_map(const std::vector<MapEntry>& maps, const void* addr);

}  // namespace dw
```

### A.3 `src/proc_self.cpp`

**Updated (Chapter 2 → 4).** Adds the `read_file_result` implementation right after `read_file`; every Chapter 2 function body is unchanged.

```cpp
#include "dw/proc_self.hpp"

#include <cerrno>
#include <charconv>
#include <cstdio>
#include <cstring>
#include <fcntl.h>
#include <string_view>
#include <sys/resource.h>
#include <system_error>
#include <utility>
#include <unistd.h>

#include "dw/fd.hpp"

namespace dw {

bool read_file(const char* path, std::string& out) {
    out.clear();
    Fd fd(::open(path, O_RDONLY | O_CLOEXEC));
    if (!fd.valid()) {
        return false;
    }
    char buf[4096];
    for (;;) {
        const ssize_t n = ::read(fd.get(), buf, sizeof buf);
        if (n > 0) {
            out.append(buf, static_cast<std::size_t>(n));
        } else if (n == 0) {
            return true;  // end of file
        } else if (errno != EINTR) {
            return false;  // real error; EINTR means "interrupted, try again"
        }
    }
}

Result<std::string> read_file_result(const char* path) {
    Fd fd(::open(path, O_RDONLY | O_CLOEXEC));
    if (!fd.valid()) {
        const int saved_errno = errno;  // capture BEFORE any other call can clobber it
        return Result<std::string>::failure(
            saved_errno, std::string("open failed: ") + path + ": " + std::strerror(saved_errno));
    }

    std::string out;
    char buf[4096];
    for (;;) {
        const ssize_t n = ::read(fd.get(), buf, sizeof buf);
        if (n > 0) {
            out.append(buf, static_cast<std::size_t>(n));
        } else if (n == 0) {
            return Result<std::string>::success(std::move(out));  // end of file
        } else if (errno != EINTR) {
            const int saved_errno = errno;
            return Result<std::string>::failure(
                saved_errno, std::string("read failed: ") + path + ": " + std::strerror(saved_errno));
        }
        // EINTR: loop and retry, same as read_file above
    }
}

namespace {

// Parse the integer after "Key:" (leading blanks skipped). -1 if it is not a number.
long parse_number(std::string_view text) {
    while (!text.empty() && (text.front() == ' ' || text.front() == '\t')) {
        text.remove_prefix(1);
    }
    long value = -1;
    const auto res = std::from_chars(text.data(), text.data() + text.size(), value);
    return res.ec == std::errc() ? value : -1;
}

}  // namespace

bool read_mem_snapshot(MemSnapshot& out) {
    out = MemSnapshot{};

    std::string text;
    if (!read_file("/proc/self/status", text)) {
        return false;
    }

    std::string_view rest(text);
    while (!rest.empty()) {
        const std::size_t nl = rest.find('\n');
        const std::string_view line = rest.substr(0, nl);  // npos: the whole remainder
        rest = (nl == std::string_view::npos) ? std::string_view{} : rest.substr(nl + 1);

        const std::size_t colon = line.find(':');
        if (colon == std::string_view::npos) {
            continue;
        }
        const std::string_view key = line.substr(0, colon);
        const std::string_view value = line.substr(colon + 1);

        if (key == "VmSize") {
            out.vm_size_kb = parse_number(value);
        } else if (key == "VmRSS") {
            out.vm_rss_kb = parse_number(value);
        } else if (key == "VmHWM") {
            out.vm_hwm_kb = parse_number(value);
        } else if (key == "RssAnon") {
            out.rss_anon_kb = parse_number(value);
        } else if (key == "RssFile") {
            out.rss_file_kb = parse_number(value);
        } else if (key == "RssShmem") {
            out.rss_shmem_kb = parse_number(value);
        } else if (key == "Threads") {
            out.threads = parse_number(value);
        }
    }

    rusage ru{};
    if (::getrusage(RUSAGE_SELF, &ru) == 0) {
        out.minor_faults = static_cast<long>(ru.ru_minflt);
        out.major_faults = static_cast<long>(ru.ru_majflt);
    }
    return true;
}

bool read_maps(std::vector<MapEntry>& out) {
    out.clear();

    std::string text;
    if (!read_file("/proc/self/maps", text)) {
        return false;
    }

    std::size_t pos = 0;
    while (pos < text.size()) {
        std::size_t nl = text.find('\n', pos);
        if (nl == std::string::npos) {
            nl = text.size();
        }
        const std::string line = text.substr(pos, nl - pos);  // sscanf needs a NUL-terminated string
        pos = nl + 1;
        if (line.empty()) {
            continue;
        }

        // Format: start-end perms offset dev inode   [path]
        unsigned long start = 0;
        unsigned long end = 0;
        unsigned long offset = 0;
        unsigned long inode = 0;
        char perms[8] = {};
        char dev[32] = {};
        int consumed = -1;
        const int fields = std::sscanf(line.c_str(), "%lx-%lx %7s %lx %31s %lu %n", &start, &end,
                                       perms, &offset, dev, &inode, &consumed);
        if (fields < 6) {
            return false;  // unexpected format: better to fail loudly than to guess
        }

        MapEntry e;
        e.start = start;
        e.end = end;
        e.perms = perms;
        e.offset = offset;
        e.inode = inode;
        if (consumed >= 0 && static_cast<std::size_t>(consumed) < line.size()) {
            e.path = line.substr(static_cast<std::size_t>(consumed));  // may contain spaces
        }
        out.push_back(std::move(e));
    }
    return true;
}

const MapEntry* find_map(const std::vector<MapEntry>& maps, const void* addr) {
    const auto a = reinterpret_cast<unsigned long>(addr);
    for (const auto& m : maps) {
        if (a >= m.start && a < m.end) {
            return &m;
        }
    }
    return nullptr;
}

}  // namespace dw
```

### A.4 `lab/errno_race.cpp`

**New.** Step 4.3: the errno-clobbering bug, buggy and fixed versions side by side.

```cpp
// The errno race: errno is a single, PER-THREAD variable that any failing
// libc/syscall wrapper can overwrite. If you don't capture it IMMEDIATELY
// after the call that set it, something else can clobber it before you
// look -- and you'll report the wrong error with total confidence.
//
// Usage: errno_race        (runs the buggy version, then the fixed version)
#include <cerrno>
#include <cstdio>
#include <cstring>
#include <fcntl.h>
#include <unistd.h>

namespace {

// Note: errno is guaranteed meaningful only right after a call that FAILS.
// A call that SUCCEEDS is not required to touch it at all -- so the
// "innocent" step in between must itself fail to reliably clobber errno.
// Real code clobbers it this way constantly: log a message (which may open
// a log file), check another resource, clean something up -- any of which
// can fail for its OWN reason before you get back to the original error.

// BUGGY: errno is read only when we finally print it, one unrelated failing
// call after the failure that actually set it.
void buggy_version() {
    int fd = ::open("/proc/999999/status", O_RDONLY);  // fails: ENOENT (2)
    if (fd < 0) {
        std::fprintf(stderr, "[buggy] about to report the error...\n");
        // An unrelated permission check -- exactly the kind of "let me also
        // verify X" step real error-handling code adds -- that itself
        // fails, and silently overwrites errno with ITS OWN failure code.
        ::open("/proc/1/maps", O_RDONLY);  // fails: EACCES (13) in this sandbox
        std::fprintf(stderr, "[buggy] open of /proc/999999/status failed: %s\n",
                     std::strerror(errno));  // WRONG: reports EACCES, not ENOENT
    }
}

// FIXED: errno is captured in a local variable in the very next line after
// the call that can set it, before anything else runs.
void fixed_version() {
    int fd = ::open("/proc/999999/status", O_RDONLY);  // fails: ENOENT (2)
    if (fd < 0) {
        const int saved_errno = errno;  // captured BEFORE any other call
        std::fprintf(stderr, "[fixed] about to report the error...\n");
        ::open("/proc/1/maps", O_RDONLY);  // still fails, still clobbers errno -- doesn't matter now
        std::fprintf(stderr, "[fixed] open of /proc/999999/status failed: %s\n",
                     std::strerror(saved_errno));  // correct: reports ENOENT
    }
}

}  // namespace

int main() {
    buggy_version();
    fixed_version();
    return 0;
}
```

### A.5 `include/dw/log.hpp`

**New.** The `DW_LOG_*` macros and `dw::LogLevel`.

```cpp
#pragma once

namespace dw {

enum class LogLevel { Debug = 0, Info = 1, Warn = 2, Error = 3 };

// Messages below this level are dropped (cheaply: one atomic load, no
// formatting work). Default: Info.
void set_log_level(LogLevel level);

// Not for direct use -- go through the DW_LOG_* macros below, which supply
// __FILE__ and __LINE__ automatically. printf-style format string.
void log_impl(LogLevel level, const char* file, int line, const char* fmt, ...)
#if defined(__GNUC__)
    __attribute__((format(printf, 4, 5)))  // lets the compiler check the format string itself
#endif
    ;

}  // namespace dw

#define DW_LOG_DEBUG(...) ::dw::log_impl(::dw::LogLevel::Debug, __FILE__, __LINE__, __VA_ARGS__)
#define DW_LOG_INFO(...) ::dw::log_impl(::dw::LogLevel::Info, __FILE__, __LINE__, __VA_ARGS__)
#define DW_LOG_WARN(...) ::dw::log_impl(::dw::LogLevel::Warn, __FILE__, __LINE__, __VA_ARGS__)
#define DW_LOG_ERROR(...) ::dw::log_impl(::dw::LogLevel::Error, __FILE__, __LINE__, __VA_ARGS__)
```

### A.6 `src/log.cpp`

**New.** The logger implementation.

```cpp
#include "dw/log.hpp"

#include <atomic>
#include <cstdarg>
#include <cstdio>
#include <ctime>

namespace dw {
namespace {

// NOT thread-safe yet: a plain atomic protects the level itself, but two
// threads logging at once can interleave their fprintf output. Revisit
// when the project adds real concurrency (see linux-notes on threading).
std::atomic<LogLevel> g_level{LogLevel::Info};

const char* level_name(LogLevel level) {
    switch (level) {
        case LogLevel::Debug:
            return "DEBUG";
        case LogLevel::Info:
            return "INFO ";
        case LogLevel::Warn:
            return "WARN ";
        case LogLevel::Error:
            return "ERROR";
    }
    return "?????";
}

}  // namespace

void set_log_level(LogLevel level) { g_level.store(level, std::memory_order_relaxed); }

void log_impl(LogLevel level, const char* file, int line, const char* fmt, ...) {
    if (static_cast<int>(level) < static_cast<int>(g_level.load(std::memory_order_relaxed))) {
        return;  // cheap early exit: no formatting, no syscall
    }

    timespec ts{};
    clock_gettime(CLOCK_REALTIME, &ts);
    tm tm_buf{};
    localtime_r(&ts.tv_sec, &tm_buf);
    char time_buf[16];
    std::strftime(time_buf, sizeof time_buf, "%H:%M:%S", &tm_buf);

    std::fprintf(stderr, "%s.%03ld [%s] %s:%d: ", time_buf, ts.tv_nsec / 1000000L,
                 level_name(level), file, line);

    va_list args;
    va_start(args, fmt);
    std::vfprintf(stderr, fmt, args);
    va_end(args);

    std::fprintf(stderr, "\n");
}

}  // namespace dw
```

### A.7 `src/CMakeLists.txt`

**Replaces the Chapter 2 version.** Adds `log.cpp` to `dw_core`.

```cmake
# The real tool. Code arrives here only after it has earned its place.
add_library(dw_core STATIC proc_self.cpp log.cpp)
target_link_libraries(dw_core PUBLIC dw_options)
```

### A.8 `lab/CMakeLists.txt`

**Replaces the Chapter 3 version.** Registers `errno_race`.

```cmake
# Lab = throwaway experiments and deliberately broken programs.
# Code graduates to src/ only after it has earned its place.
function(dw_lab name)
  add_executable(${name} ${name}.cpp)
  target_link_libraries(${name} PRIVATE dw_options)
endfunction()

dw_lab(hello)
dw_lab(leak_bounded)
dw_lab(bugs)
dw_lab(fd_leak)
dw_lab(fd_raii)
dw_lab(bench)
dw_lab(errno_race)
dw_lab(observe_pid)
dw_lab(run_measure)

# Programs that use the real tool's library (dw_core).
function(dw_lab_core name)
  add_executable(${name} ${name}.cpp)
  target_link_libraries(${name} PRIVATE dw_core)
endfunction()

dw_lab_core(maps_summary)
dw_lab_core(touch_pages)
dw_lab_core(heap_vs_mmap)
dw_lab_core(leak_revisited)
dw_lab_core(growth_bounded)
dw_lab_core(fragmentation)
dw_lab_core(excess_alloc)
```

### A.9 `tests/CMakeLists.txt`

**Replaces the Chapter 2 version.** Registers `result_and_log_test`.

```cmake
add_executable(fd_test fd_test.cpp)
target_link_libraries(fd_test PRIVATE dw_options)
add_test(NAME fd_test COMMAND fd_test)

add_executable(proc_self_test proc_self_test.cpp)
target_link_libraries(proc_self_test PRIVATE dw_core)
add_test(NAME proc_self_test COMMAND proc_self_test)

add_executable(result_and_log_test result_and_log_test.cpp)
target_link_libraries(result_and_log_test PRIVATE dw_core)
add_test(NAME result_and_log_test COMMAND result_and_log_test)
```

### A.10 `tests/result_and_log_test.cpp`

**New.** Tests for `Result<T>`, `read_file_result`, and the logger.

```cpp
// Tests for dw::Result<T>, dw::to_string(Error), and dw::read_file_result.
#include <cerrno>
#include <cstdio>
#include <cstdlib>
#include <string>

#include "dw/log.hpp"
#include "dw/proc_self.hpp"
#include "dw/result.hpp"

#define CHECK(cond)                                                                        \
    do {                                                                                   \
        if (!(cond)) {                                                                     \
            std::fprintf(stderr, "CHECK failed: %s (%s:%d)\n", #cond, __FILE__, __LINE__); \
            std::exit(1);                                                                  \
        }                                                                                  \
    } while (0)

int main() {
    // 1. Result<T>: success carries a value, is truthy, ok() agrees with operator bool.
    {
        auto r = dw::Result<int>::success(42);
        CHECK(r.ok());
        CHECK(static_cast<bool>(r));
        CHECK(r.value() == 42);
        CHECK(r.value_or(-1) == 42);
    }

    // 2. Result<T>: failure carries an Error, is falsy, value_or falls back.
    {
        auto r = dw::Result<int>::failure(ENOENT, "test failure");
        CHECK(!r.ok());
        CHECK(!static_cast<bool>(r));
        CHECK(r.error().code == ENOENT);
        CHECK(r.error().message == "test failure");
        CHECK(r.value_or(-1) == -1);
        CHECK(dw::to_string(r.error()) == "test failure");
    }

    // 3. read_file_result: success reads real content.
    {
        auto r = dw::read_file_result("/proc/self/status");
        CHECK(r.ok());
        CHECK(r.value().size() > 200);
        CHECK(r.value().find("VmRSS:") != std::string::npos);
    }

    // 4. read_file_result: a missing file fails with ENOENT, not some other code.
    {
        auto r = dw::read_file_result("/proc/this/does/not/exist");
        CHECK(!r.ok());
        CHECK(r.error().code == ENOENT);
        CHECK(r.error().message.find("No such file") != std::string::npos);
    }

    // 5. Two independent failures produce two independent, correctly-labelled
    //    errors -- this is the property the errno-race bug violates.
    {
        auto missing = dw::read_file_result("/proc/999999/status");
        auto denied = dw::read_file_result("/proc/1/maps");
        // In an environment where /proc/1/maps happens to be readable (e.g. no
        // ptrace restriction), this second call succeeds instead -- that is a
        // fact about the environment, not a bug in read_file_result, so only
        // check it when it actually failed.
        CHECK(missing.error().code == ENOENT);
        if (!denied.ok()) {
            CHECK(denied.error().code != missing.error().code);
        }
    }

    // 6. Logging doesn't crash at any level, and filtering doesn't throw.
    {
        dw::set_log_level(dw::LogLevel::Debug);
        DW_LOG_DEBUG("debug %d", 1);
        DW_LOG_INFO("info %d", 2);
        DW_LOG_WARN("warn %d", 3);
        DW_LOG_ERROR("error %d", 4);
        dw::set_log_level(dw::LogLevel::Info);  // restore default for anything run after
    }

    std::puts("result_and_log_test: all checks passed");
    return 0;
}
```

### A.11 Build, test, and run

```bash
cmake --build --preset debug -j
ctest --preset debug
./build/debug/lab/errno_race
```

**Expected:** the build finishes with no warnings; `ctest` reports `fd_test`, `proc_self_test`, and `result_and_log_test` all passed (`result_and_log_test` prints `result_and_log_test: all checks passed`); `errno_race` prints the buggy version reporting the wrong error and the fixed version reporting the right one.

**Try the compile-error demo yourself** (Step 4.2), outside the normal build:

```bash
cat > /tmp/ignored_result.cpp << 'CPPEOF'
#include "dw/proc_self.hpp"
int main() {
    dw::read_file_result("/proc/self/status");  // return value dropped
    return 0;
}
CPPEOF
g++ -std=c++17 -Iinclude -Werror -c /tmp/ignored_result.cpp -o /tmp/ignored_result.o
```

**Expected:** a build failure quoting `-Werror=unused-result`, matching Step 4.2's reference output exactly.

If the CMake step errors, copy the full message and ask me. Then: `git add -A && git commit -m "Phase 0 chapter 4: Result<T>, errno-race fix, minimal logger"`.
