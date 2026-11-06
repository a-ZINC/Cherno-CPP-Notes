# Chapter 15 — Labs, Quiz, Glossary, Cheat Sheet

## 15.1 Hands-on labs (do them; this is where you become a legend)

### Lab 1 — Reserve vs commit vs touch (Ch 2)
Run `vm.c` from Chapter 2. Fill:

| Step | VmSize | VmRSS |
|---|---|---|
| after reserve | | |
| after commit | | |
| after touch 100MB | | |

Then run `HeapWatch.java` with `-Xms64m -Xmx4g` and with `-XX:+AlwaysPreTouch`. Explain the difference.

### Lab 2 — Mini JVM (Ch 3)
Add `ISUB`, `IMUL`, and `IF_ICMPLT` opcodes; compute 10!. Compare with `javap -c` output of the same loop in Java.

### Lab 3 — Direct memory OOM (Ch 9)
```java
import java.nio.ByteBuffer; import java.util.*;
public class Direct { public static void main(String[] a){
  List<ByteBuffer> l = new ArrayList<>();
  while (true) l.add(ByteBuffer.allocateDirect(1 << 20));
} }
```
`java -Xmx256m -XX:MaxDirectMemorySize=128m -XX:NativeMemoryTracking=summary Direct`
While it runs: `jcmd <pid> VM.native_memory summary`. Observe heap tiny, native growing, then `OutOfMemoryError: Direct buffer memory`.

### Lab 4 — Heap leak + MAT (Ch 5, 12, 14)
```java
import java.util.*;
public class Leak {
  static final Map<Long, byte[]> CACHE = new HashMap<>();
  public static void main(String[] a) throws Exception {
    long i = 0;
    while (true) { CACHE.put(i++, new byte[10_000]); if (i % 1000 == 0) Thread.sleep(10); }
  }
}
```
`java -Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp -Xlog:gc:file=/tmp/gc.log Leak`
Tasks: watch `jstat -gcutil <pid> 1s`; find the staircase in `gc.log`; open the `.hprof` in MAT; find `Leak.CACHE` via **Path to GC Roots**.

### Lab 5 — Thread leak (Ch 8)
```java
public class Threads { public static void main(String[] a) throws Exception {
  while (true) { new Thread(() -> { try { Thread.sleep(Long.MAX_VALUE);} catch(Exception e){} }).start(); Thread.sleep(5);} } }
```
Watch `grep Threads /proc/<pid>/status`; thread dump grouped by name; end state: `unable to create native thread`. Repeat in Docker with `--pids-limit 300`.

### Lab 6 — Metaspace growth (Ch 6)
Generate classes in a loop (ByteBuddy, or `Proxy.newProxyInstance` with a new `ClassLoader` each time, keeping references). Run with `-XX:MaxMetaspaceSize=64m`. Observe `OutOfMemoryError: Metaspace` and `jcmd <pid> VM.metaspace`.

### Lab 7 — Container reality (Ch 10)
```bash
docker run --rm -m 512m eclipse-temurin:21 java -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|UseSerialGC|UseG1GC|ActiveProcessorCount"
docker run --rm -m 512m --cpus=1 eclipse-temurin:21 java -XshowSettings:system -version
```
Then run Lab 3 with `-XX:MaxRAMPercentage=70` and no direct limit until the container is OOMKilled: `docker inspect <id> --format '{{.State.OOMKilled}}'`.

### Lab 8 — GC comparison (Ch 5, 11)
Same workload, same `-Xmx2g`, under Serial / Parallel / G1 / ZGC:

| GC | Throughput | p99 | Max pause | RSS |
|---|---|---|---|---|
| Serial | | | | |
| Parallel | | | | |
| G1 | | | | |
| ZGC | | | | |

### Lab 9 — JIT in action (Ch 7)
Run a hot loop with `-Xint`, default, and `-XX:TieredStopAtLevel=1`; add `-XX:+PrintCompilation`. Find tier transitions and a `made not entrant`.

### Lab 10 — Profile a real service (Ch 13)
2 min JFR + CPU/alloc/wall async-profiler graphs on staging under load. Write 3 findings and 1 fix with before/after numbers.

## 15.2 Quiz (answers hidden)

**1.** Explain "Java heap is virtual memory" using the words reserve, commit, touch.
<details><summary>Answer</summary>The JVM `mmap`s a PROT_NONE range of size `-Xmx` (reserve, costs no RAM), makes the first `-Xms` usable (commit), and RAM is consumed only when pages are written (touch → page faults). RSS counts touched pages. The JVM's allocator and GC manage objects inside that range.</details>

**2.** Name the two "heaps" in a JVM process.
<details><summary>Answer</summary>The Java heap (object memory managed by GC) and the native/C heap (malloc, used by JVM internals, JNI libs, direct buffers).</details>

**3.** Why is Java `new` fast?
<details><summary>Answer</summary>Per-thread TLAB: pointer bump, no lock; freeing is bulk and proportional to survivors.</details>

**4.** Why can't Java expose raw pointers?
<details><summary>Answer</summary>The GC moves objects; raw addresses would dangle; arithmetic would break safety.</details>

**5.** What is the card table for?
<details><summary>Answer</summary>Tracking old→young references so a young GC doesn't scan the whole old generation.</details>

**6.** Name 5 memory regions in a JVM process besides the Java heap.
<details><summary>Answer</summary>Metaspace, code cache, thread stacks, direct/Netty buffers, GC structures, malloc arenas/JNI, mapped files.</details>

**7.** `-Xmx5g` in a 6 GiB pod, repeatedly OOMKilled with heap 2.5 GiB used. Why?
<details><summary>Answer</summary>Only ~1 GiB left for non-heap memory; native bursts exceed the cgroup limit.</details>

**8.** How does a JIT speculate safely?
<details><summary>Answer</summary>It compiles under assumptions from the profile (guards); if violated, it deoptimizes back to the interpreter.</details>

**9.** `volatile int c; c++` from many threads: correct?
<details><summary>Answer</summary>No: read-modify-write isn't atomic. Use `AtomicInteger`/`LongAdder`.</details>

**10.** Shallow vs retained size?
<details><summary>Answer</summary>Shallow = the object's own bytes; retained = bytes freed if it were collected (its exclusively-held subtree).</details>

**11.** What proves a heap leak in metrics?
<details><summary>Answer</summary>A steadily rising after-GC floor (`jvm_gc_live_data_size_bytes`).</details>

**12.** Heap dump 300 MB, RSS 5 GB. Where to look?
<details><summary>Answer</summary>Off-heap: NMT diff, thread count, direct/Netty metrics, `pmap`, glibc arenas.</details>

**13.** Exit code 137 means?
<details><summary>Answer</summary>SIGKILL, typically the cgroup OOM killer; total RSS exceeded the limit; no Java stack trace.</details>

**14.** Which C++ tools map to: perf, valgrind, gdb `bt`, TSan?
<details><summary>Answer</summary>async-profiler/JFR; heap dump+MAT / Netty leak detector; `jcmd Thread.print`; jcstress/JFR lock events.</details>

**15.** Why not copy 30 GC flags from a blog?
<details><summary>Answer</summary>Flags are version- and workload-specific; many are obsolete; without a baseline you can't tell if they help.</details>

## 15.3 A–Z glossary

| Term | Plain meaning |
|---|---|
| **AOT / JIT** | Compile before running / while running |
| **AppCDS** | Archive of loaded classes for faster startup |
| **Arena** | Big chunk an allocator carves up |
| **Barrier (write/load)** | Extra instructions around reference stores/loads that help the GC |
| **Bytecode** | Instructions of the imaginary JVM CPU |
| **C1 / C2** | Fast JIT / heavy JIT |
| **Card table** | Byte map marking changed heap regions |
| **cgroup** | Linux resource-limit group (container) |
| **ClassLoader** | Object that locates and loads classes |
| **Code cache** | Memory for JIT machine code |
| **Commit / Reserve / Touch** | Make usable / claim addresses / actually use RAM |
| **Compressed oops** | 4-byte object references |
| **Deopt** | Fall back from compiled to interpreted code |
| **Direct buffer** | Java object owning native memory |
| **Dominator** | Object that solely retains others |
| **Eden / Survivor / Old** | Young areas and the old area of the heap |
| **Escape analysis** | Proof an object never leaves a method |
| **GC root** | Starting point for reachability (stack vars, statics) |
| **Happens-before** | Guarantee that one thread sees another's writes |
| **Heap dump** | Snapshot of all Java objects |
| **Humongous** | G1 object ≥ half a region |
| **IHOP** | Heap % at which G1 starts concurrent marking |
| **Inlining** | Copy callee body into caller |
| **JFR / JMC** | Flight recorder / its GUI |
| **JMH** | Microbenchmark harness |
| **JMM** | Java Memory Model |
| **JNI / FFM** | Calling native code from Java |
| **Klass** | JVM's internal class description (≈ vtable+RTTI) |
| **Mark word** | Header word: hash, age, lock bits |
| **Metaspace** | Native memory holding class metadata |
| **Monitor** | Lock behind `synchronized` |
| **NMT** | JVM native memory tracking |
| **OOMKilled** | Container killed by kernel for exceeding memory |
| **OSR** | Switch to compiled code mid-loop |
| **Page cache** | RAM caching file data |
| **Pinning** | Virtual thread stuck on its carrier |
| **Promotion** | Moving a survivor from young to old |
| **Retained size** | Memory freed if an object were collected |
| **RSS / VSZ** | Real resident memory / total virtual size |
| **Safepoint** | Place where threads can be stopped for the VM |
| **STW** | Stop-the-world pause |
| **TLAB** | Thread-local allocation buffer |
| **TTSP** | Time to safepoint |
| **Warm-up** | Time until JIT has compiled the hot code |
| **Working set** | Memory metric K8s uses; closest to OOM risk |

## 15.4 One-page cheat sheet

```
PROCESS = Java heap (mmap'd, -Xmx) + Metaspace + Code cache + Thread stacks
          + Direct/Netty + GC structures + malloc/JNI/JVM internals
RESERVE (free) → COMMIT (allowed) → TOUCH (RSS, counts toward pod limit)
JAVA new  = bump pointer in TLAB    GC = mark/copy survivors, free the rest
LEAK SIGNAL = rising after-GC floor (jvm_gc_live_data_size_bytes)
HEAP FLAT + RSS UP = off-heap: threads / direct / Metaspace / native (NMT diff, pmap)
137 = OOMKilled (kernel).  Java OOM = JVM's own limit.  Read the message.
POD: Xmx ≈ 55-65% of limit (Netty/gRPC), cap Metaspace/direct/stack, MALLOC_ARENA_MAX=2
JIT: interpreter → C1 → C2, deopt on broken guesses, warm-up minutes
TOOLS: jcmd | jstat | MAT | NMT | JFR | async-profiler | JMH | kubectl debug
```

## 15.5 Study plan (2 weeks → legend)

| Day | Do |
|---|---|
| 1 | Ch 1–2, Lab 1 |
| 2 | Ch 3, Lab 2 |
| 3 | Ch 4–5, Lab 4 |
| 4 | Ch 5 logs: read a real `gc.log`; Lab 8 |
| 5 | Ch 6–7, Labs 6, 9 |
| 6 | Ch 8, Lab 5 |
| 7 | Ch 9, Lab 3 |
| 8 | Ch 10, Lab 7; fill your pod worksheet |
| 9 | Ch 11; audit your flags |
| 10 | Ch 12; practise jcmd/MAT/NMT on staging |
| 11 | Ch 13, Lab 10 |
| 12 | Ch 14; build the 8-panel dashboard |
| 13–14 | 24–48 h soak test; walk the decision tree; write findings |

## 15.6 Legend checklist
- [ ] Draw the JVM process memory map from memory.
- [ ] Explain reserve/commit/touch and run `vm.c`.
- [ ] Write mark-sweep and explain why Java uses copying/compaction.
- [ ] Compute object sizes by hand and verify with JOL.
- [ ] Read a GC log and extract floor, alloc rate, p99 pause.
- [ ] Explain JIT tiers, inlining, deopt.
- [ ] Map `volatile/synchronized/Atomic` to `atomic/mutex`.
- [ ] Fill a pod memory budget and justify each flag.
- [ ] Diagnose an off-heap leak with NMT + Netty detector.
- [ ] Produce CPU, alloc and wall flame graphs and act on them.
- [ ] Solve the 0.95 → 1.0 case with data, not guesses.

## 15.7 Further reading
- *Java Performance* (Scott Oaks); *Optimizing Cloud Native Java* (Ben Evans); *The Garbage Collection Handbook*.
- OpenJDK HotSpot docs/wiki; JEP 439 (Generational ZGC), JEP 444 (Virtual Threads).
- Aleksey Shipilëv's *JVM Anatomy Quarks*; Brendan Gregg's flame graph/perf pages.

Prev: [14](14-leaks-and-case-study.md) · Back to [Index](00-README.md)
