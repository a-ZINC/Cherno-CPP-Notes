To understand why your `Storage Per Thread` approach (using a `std::vector<int> cnts(4)`) is so fast compared to the `Mutex` approach, we have to look at the **Hardware Cache Architecture** and how it interacts with your code.

The core of the performance gap lies not just in the data distribution, but in how **"smart"** modern CPU cores are at hiding inefficiencies.

---

### The Memory Hierarchy Diagram

When you have 4 threads, each thread typically runs on its own physical **CPU Core**. Each core has its own private **L1/L2 Cache**.

```text
[ Core 1 ]    [ Core 2 ]    [ Core 3 ]    [ Core 4 ]
    |             |             |             |
[ L1 Cache ]  [ L1 Cache ]  [ L1 Cache ]  [ L1 Cache ]
    |             |             |             |
====================================================== <--- Hardware Bus / Interconnect
    |
[ L3 Cache (Shared) ]
    |
[ RAM (Main Memory) ]

```

---

### The Two Scenarios

#### 1. The Mutex Scenario (The "Traffic Jam")

In your `simpleMultiThreadMutex` test, all 4 cores are trying to update **one single variable** (`cnt`).

* **Action:** Core 1 locks the mutex and writes to `cnt`.
* **Coherency Trigger:** The CPU hardware uses the **MESI protocol**. Core 1 tells all other cores: *"I am writing to this memory address; invalidate your copies!"*
* **The "Stop":** When Core 2 tries to write to the same `cnt`, it finds its cache line is "Invalid." It must wait for the hardware to sync the value from Core 1.
* **The OS Overhead:** On top of this, the `std::mutex` makes the OS "pause" the thread if it's waiting for the lock, adding huge delays.

#### 2. Your `Storage Per Thread` Scenario

In `simpleMultiThreadStoragePerThread`, each thread is working on `cnts[i]`.

* **The Reality of False Sharing:** You are correct—`cnts[0], [1], [2], [3]` are all inside the **same 64-byte cache line**.
* **Why it's still fast:** Even though they are in the same cache line, they are **different bytes** within that line. When Core 1 writes to `cnts[0]`, the hardware marks the entire cache line as "Modified" in Core 1's cache. If Core 2 tries to write to `cnts[1]`, the hardware does have to do a coherency handshake.

---

### The "Smart Core" Factor: Why it doesn't affect you

You might wonder: *If they are fighting over the same cache line, why isn't performance destroyed?* The answer is that modern **CPU cores are incredibly smart.** They use several advanced techniques to mask these hardware stalls:

#### 1. Out-of-Order Execution (OoO)

Modern CPU cores do not execute your code like a mindless robot reading a list. They have a massive "Instruction Window." If Core 2 needs to write to `cnts[1]` but the cache line is currently being held by Core 1, the core does not just sit idle. It looks ahead at your code, finds other instructions (like your `rand()` math) that don't depend on that write, and executes them first.

#### 2. Latency Hiding

Because your math (`(double)rand() / RAND_MAX`) is so complex and takes so long, the "handshake" (the cache line moving from Core 1 to Core 2) happens entirely in the background. The CPU core effectively says: *"I'll request the cache line now, and while the hardware bus handles that 50-nanosecond delay, I'll go do 200 cycles of math."* By the time the math is done, the cache line has arrived, and the core commits the write without ever actually stopping.

#### 3. Pipeline Efficiency

The CPU pipeline is designed to keep moving. In your `Storage Per Thread` example, the cores are not blocked; they are simply experiencing a slightly delayed write. The CPU "smartly" reorders the execution so that the pipeline stays full.

---

### Why the "Coherency" isn't stopping your code

| Feature | Mutex Scenario | Storage Per Thread Scenario |
| --- | --- | --- |
| **Sync Type** | **OS Level (Software)** | **Hardware Level (MESI)** |
| **Delay** | High (Context switches) | Tiny (Hardware bus signals) |
| **Wait** | Thread is "Blocked" by OS | Pipeline "Stall" (Microseconds) |
| **Frequency** | 10 million times | Only when cache lines ping-pong |

### Visualizing the "Hardware Handshake"

Imagine the CPU Interconnect as a high-speed messaging system:

1. **Core 1** wants to update `cnts[0]`.
2. **Core 1** sends a "Request for Ownership" (RFO) message.
3. **Core 2, 3, 4** see the message and invalidate their local copy of that cache line.
4. **Core 1** performs the math and writes the value.
5. **Later:** **Core 2** wants to update `cnts[1]`. It sends an RFO, and the line moves to **Core 2**.

Because your math (the random number generator) is so "heavy," the **Hardware Interconnect** is finished with its handshake long before the next thread is ready to write again. You aren't "seeing" the coherency cost because the hardware and the smart scheduling of the CPU cores are so much faster than your math loop.

### How to "Break" it

If you want to see the performance drop, you need to make the **handshake frequency higher than the math time**.

If you remove the `rand()` math and just do `cnts[i]++` in a tight loop, the cores will fight over that cache line millions of times per second. That is when you will see the "False Sharing" penalty destroy your performance because the Hardware Bus will become completely saturated with "Invalidate" messages, and even the "smart" out-of-order execution won't be able to find enough work to hide the delay.

> **In summary:** Your `Storage Per Thread` approach works because you are **distributing the data** across the cores. Even if they are sitting on the same physical cache line, the CPU hardware and the **smart out-of-order execution units** are doing a better job of managing that than the OS kernel is at managing a `std::mutex`.

The distinction between "OS managing a queue" (Mutex) vs "CPU hardware and smart logic managing a cache line" (Storage Per Thread) creates the massive efficiency gap you see in your results.
