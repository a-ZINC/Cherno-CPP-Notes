# PART 2 — Linux Process + Kernel Foundations

## Chapter 2.4 — User Space

### 🧠 One-Sentence Mental Model
> User space is where every ordinary program runs, deliberately and permanently restricted by real CPU hardware — not by a polite software agreement — from touching devices directly, reading other processes' memory, or executing privileged instructions.

### 🧒 Explain Like I'm Five
Imagine a museum where visitors (user-space programs) can walk around and look at anything on display, but there's a red velvet rope around certain areas — the control room, the vault, the wiring closet — and the rope isn't just a polite suggestion, it's electrified. Visitors physically cannot cross it, no matter how badly they want to, because the museum's own security system (the CPU hardware) actively enforces the boundary, not because visitors are trusted to behave.

### 🌍 Real-World Analogy
Think of an airport. Passengers (user-space code) can walk through the terminal, use the shops, sit at gates — but they cannot walk onto the tarmac, into the control tower, or into the baggage-handling machinery, even if they really want to and even if nobody's watching at a given moment. It's not that passengers have agreed not to — the doors themselves are physically locked and require special credentials (kernel-level privilege) that ordinary tickets simply don't grant. The airport doesn't hope passengers behave; it makes misbehavior physically impossible through the design of the building itself.

### ❓ The Problem
Chapter 2.3 showed that virtual memory gives every process its own isolated address space — but that alone doesn't explain how a process is stopped from doing something *catastrophic*, like directly reprogramming a device's MMIO registers (Part 1 Chapter 1.6) or disabling interrupts system-wide. Address-space isolation protects memory; something else has to protect *instructions* — some way of saying "this specific operation is too dangerous for ordinary code to perform, ever, no exceptions."

### 🔥 Why This Problem Matters
If any running program could execute *any* instruction, including ones that reprogram hardware or bypass memory protection, then virtual memory's isolation guarantee (Chapter 2.3) would be worthless — a malicious or simply buggy program could just directly manipulate the MMU's own configuration and read anything it wanted. The privilege boundary this chapter describes is what makes every other safety guarantee in this course actually hold up in practice, not just in theory.

### 🕰 Historical Context
Early, simpler CPUs had no privilege levels at all — any running code could execute any instruction, which was fine for single-program, trusted-operator machines but catastrophic once systems started running multiple, independently-written, potentially buggy or hostile programs simultaneously. Hardware-enforced privilege levels (rings on x86, similarly-named mechanisms on other architectures) were introduced specifically to let an operating system safely run untrusted code *at all* — this hardware feature is the actual precondition for every multi-user, multi-process operating system that followed.

### 💡 The Naive Solution
Trust that application code simply won't try to do anything dangerous, enforced only by convention or by the compiler refusing to generate certain instructions.

### ❌ Why the Naive Solution Fails
Compilers can be bypassed (hand-written assembly, or a sufficiently motivated bug), and "trust" is not a security boundary against either malice or ordinary bugs. Any purely software-level restriction, with no hardware backing, could in principle be defeated by code clever or buggy enough to reach the underlying instruction stream directly. Real isolation needs to be enforced by something a running program *cannot* simply route around — which means it needs to be enforced by the CPU itself, at the hardware level, checked before every privileged instruction actually executes.

### ✅ The Better Solution
Build the restriction into the CPU's own silicon: a hardware mode bit (the "current privilege level") that the processor checks before executing certain instructions, refusing them entirely — with a hardware fault, not a software opinion — if the current level isn't sufficient.

### 🧠 Core Concept
> **x86 CPUs implement hardware privilege rings (0 = most privileged, 3 = least). The kernel runs in ring 0; ordinary application code runs in ring 3. This is a real hardware mode bit the CPU checks before executing certain instructions — not a software convention that determined code could bypass.**

### 📐 Deep Technical Explanation

**What "checked by hardware" actually means, mechanically:** certain instructions are defined, in the CPU's own instruction-decoding logic, to be **privileged** — meaning the CPU's control unit (Part 1 Chapter 1.1) checks the current privilege level as part of decoding the instruction, before execution. Examples include: instructions that disable/enable interrupts globally, instructions that modify `CR3` (Part 1's static map — changing which page table is active), and instructions that directly access certain MMIO regions reserved for kernel/driver use.

```
[HARDWARE] CPU decodes an instruction that requires ring 0.
Current privilege level is checked (a real, physical CPU register/flag).
   IF current level == 0 (kernel) -> instruction executes normally.
   IF current level == 3 (user)   -> instruction is REFUSED.
                                      CPU raises a GENERAL PROTECTION FAULT
                                      (a hardware exception, mechanically
                                      similar in KIND to Part 1's page-fault
                                      mechanism, but signaling "not allowed"
                                      rather than "not currently resident").
                                      Kernel's fault handler typically
                                      responds by killing the offending
                                      process (delivering SIGSEGV or similar).
```

This is precisely why "just don't write buggy privileged instructions" is not the actual safety mechanism here — even a perfectly well-intentioned program that somehow ended up trying to execute a ring-0-only instruction from ring 3 would be stopped by hardware, unconditionally, every single time, with zero reliance on the program's own good behavior.

**What user space cannot do, and the precise reason each restriction holds:**
- **Cannot directly read/write another process's memory** — enforced by Chapter 2.3's separate page tables: there is no valid virtual-to-physical translation from process A's page table into process B's physical frames at all, so there's no address A could even construct to reach B's memory, privilege rings aside.
- **Cannot directly issue MMIO writes to a device** (Part 1 Chapter 1.6) — the physical address ranges MMIO registers live at are typically mapped only into the kernel's portion of the address space, and even where mapped, the *instructions* needed to safely sequence device programming are often privileged; only the kernel driver, running in ring 0, is permitted to touch them.
- **Cannot directly modify its own page table or `CR3`** — must ask the kernel (via a syscall, Chapter 2.6) to make any change to its own memory mappings; this is exactly why `mmap()`, `munmap()`, and even ordinary heap growth via `brk()`/`sbrk()` are syscalls rather than plain instructions a program executes on its own.

### 🏗 Architecture

```mermaid
flowchart TB
    subgraph Ring3["Ring 3 -- User Space (unprivileged)"]
        APP["Your C++ application code"]
        LIBC["libc / other libraries"]
    end
    subgraph Ring0["Ring 0 -- Kernel Space (privileged)"]
        KERN["Kernel code: scheduler, drivers, fault handlers (Part 1)"]
    end
    APP --> LIBC
    LIBC -.syscall instruction, Ch 2.6.-> KERN
    APP -.attempts a privileged instruction directly.-> FAULT["GENERAL PROTECTION FAULT<br/>(hardware refuses, unconditionally)"]
```

**How to read this diagram:** the solid arrow shows the *only* sanctioned path from user code toward privileged operations — through a library function that ultimately issues a proper `syscall` instruction (Chapter 2.6 builds this transition in full). The dotted arrow at the bottom shows what happens if code tries to skip that sanctioned path and execute a privileged instruction directly from ring 3 — the CPU itself refuses, hard, every time, regardless of what the code "intended."

### ❌ Common Misconceptions
- ❌ **"Privilege levels are a software/OS-level convention."** — They're a real, physical CPU feature (a hardware register tracking current privilege level, checked as part of instruction decoding) — an OS chooses *how* to use rings, but cannot make ring 3 code execute a ring-0-only instruction through any software trick; the refusal happens in silicon.
- ❌ **"A user-space program could, with enough cleverness, directly write to a device's registers if it really tried."** — Barring a genuine security vulnerability (like Meltdown, Chapter 2.5's gotcha), this is specifically what the hardware privilege check exists to make impossible — not merely discouraged, but mechanically refused.
- ❌ **"Isolation between processes comes entirely from privilege rings."** — Rings restrict *which instructions* can run; separate page tables (Chapter 2.3) restrict *which memory* is even reachable. Both mechanisms work together — rings alone wouldn't stop one ring-3 process from reading another ring-3 process's memory if they somehow shared a page table; it's virtual memory doing that specific job.
- ❌ **"System calls are just regular function calls into 'privileged' library code."** — A syscall is a genuine, hardware-mediated privilege-level transition (Chapter 2.6 in full) — fundamentally different in kind from an ordinary function call, which never changes the CPU's privilege level at all.

### 🧙 Wizard Insight
Every "sandbox," "container," or "permission system" you'll ever encounter in real systems work is, at its foundation, built on top of this one hardware fact: user-space code cannot execute privileged instructions, full stop, enforced by silicon. Containers add extra layers (namespaces, cgroups) entirely in *software*, on top of this same hardware ring boundary — they don't replace it. When evaluating any claimed security boundary in a system, the first useful question a wizard asks is: "is this actually enforced by hardware (like ring 3), or is it a software convention that a sufficiently privileged bug or exploit could route around?" — the ring boundary is the gold-standard example of the former.

### 🧠 Quiz
**Q1.** What physically stops a user-space program from executing an instruction that modifies `CR3` directly?
<details><summary>Answer</summary>The CPU's own hardware checks the current privilege level as part of decoding that instruction; from ring 3, the instruction is refused and a General Protection Fault is raised — this is enforced in silicon, not by software convention.</details>

**Q2.** Why isn't process isolation (Chapter 2.3's separate page tables) enough on its own, without privilege rings?
<details><summary>Answer</summary>Separate page tables restrict which memory addresses translate to real data, but say nothing about which INSTRUCTIONS a program can execute. Without rings, a program could still execute instructions that reconfigure hardware or the memory system itself (like changing CR3, or writing to a device's MMIO registers) even though it can't directly read another process's normal memory.</details>

**Q3.** What kind of hardware exception does an attempted privileged instruction from ring 3 raise, and what typically happens next?
<details><summary>Answer</summary>A General Protection Fault — the kernel's fault handler typically responds by terminating the offending process (delivering a signal such as SIGSEGV), similar in overall shape to how Part 1's page-fault handler responds to an illegal memory access, though it's a different specific kind of hardware exception.</details>

### 📌 Short Notes (Quick Reference)
- User space = ring 3 (unprivileged); kernel space = ring 0 (privileged) — a real, hardware-enforced CPU mode, not a software convention.
- Privileged instructions (disabling interrupts, modifying CR3, direct MMIO to protected regions) are checked by the CPU's own decode logic; attempting them from ring 3 → General Protection Fault, unconditionally.
- Privilege rings and virtual memory (Chapter 2.3) are two separate, complementary protections: rings restrict *which instructions* run; page tables restrict *which memory* is reachable.
- The only sanctioned path from user space toward privileged operations is a syscall (Chapter 2.6) — a genuine, hardware-mediated transition, not an ordinary function call.

### 🔗 What This Connects To Next
**Previous:** Part 2, Chapter 2.3 — Virtual Memory
**Current:** Part 2, Chapter 2.4 — User Space
**Next:** Part 2, Chapter 2.5 — Kernel Space (the privileged side of this exact same boundary, and why it's mapped into every process regardless)
