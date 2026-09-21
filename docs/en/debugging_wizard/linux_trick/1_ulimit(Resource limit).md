## Complete Guide: Linux Core Dumps, `ulimit`, and GDB Post-Mortem Debugging

---

### 1. What Is a Core Dump?

A **core dump** is a snapshot of a process’s address space and CPU register state written to disk at the exact moment the program terminates unexpectedly due to a fatal signal (e.g., `SIGSEGV`, `SIGABRT`).

#### What It Contains

* **CPU Registers:** Program Counter (`RIP`/`EIP`), Stack Pointer (`RSP`/`ESP`), Frame Pointer (`RBP`/`EBP`), and general-purpose registers.
* **Memory Snapshot:** Contents of the active execution stack, global/static data, heap allocations, and shared library mappings.
* **Metadata:** Active thread contexts, faulting memory addresses, and triggering signal details.

#### Signals That Trigger Core Dumps

| Signal | Description | Common Causes |
| --- | --- | --- |
| **`SIGSEGV`** | Segmentation Fault | Null pointer dereference, invalid memory access, buffer overflow. |
| **`SIGABRT`** | Abort | Triggered by `abort()`, failed `assert()`, or uncaught C++ exceptions. |
| **`SIGFPE`** | Floating Point Exception | Integer division by zero, invalid arithmetic operations. |
| **`SIGBUS`** | Bus Error | Unaligned memory access, physical address fault. |
| **`SIGILL`** | Illegal Instruction | Execution of invalid, corrupt, or unsupported CPU instructions. |

---

### 2. Managing Process Limits with `ulimit`

The `ulimit` command sets resource limits for the current shell session and child processes spawned from it.

#### Soft vs. Hard Limits

* **Soft Limit (`-S`):** The operational ceiling enforced by the kernel. Can be modified by non-root users up to the hard limit.
* **Hard Limit (`-H`):** The absolute maximum upper bound. Only superusers (`root`) can increase this limit.

#### Key Flags Summary

| Flag | Resource | Typical Usage Scenario |
| --- | --- | --- |
| **`-c`** | Core dump file size | `ulimit -c unlimited` enables saving crash dumps. |
| **`-n`** | Open file descriptors | Raised for high-concurrency servers (Nginx, databases). |
| **`-u`** | Max processes/threads | Prevents process exhaustion and fork bombs. |
| **`-s`** | Maximum stack size | Adjusted when deep recursion triggers stack overflow. |
| **`-v`** | Virtual memory limit | Prevents runaway memory allocation. |
| **`-t`** | Max CPU time (sec) | Kills tasks running beyond execution limits. |

---

### 3. Step-by-Step Workflow: Enabling and Capturing Core Dumps

By default, Linux limits core dump size to `0` and routes crash reports through system daemons like `systemd-coredump` or `Apport`. Follow these steps to generate a local file:

```bash
# 1. Allow unlimited core dump file size in the active terminal
ulimit -c unlimited

# 2. Configure the kernel to write core files to the local directory
sudo sysctl -w kernel.core_pattern=core

# 3. Run the target application to trigger the fault
./build/debug/lab/bugs overflow
# Output: Segmentation fault (core dumped)

# 4. Verify the core dump file was created
ls -l core*

```

---

### 4. Post-Mortem Analysis in GDB

To debug a crash, launch GDB by providing both the **unstripped executable** (compiled with `-g`) and the generated **core dump file**:

```bash
gdb ./build/debug/lab/bugs core

```

#### Essential GDB Commands for Core Dump Inspection

```text
(gdb) bt                  # Print the call stack (backtrace) up to the crash point
(gdb) frame <N>           # Switch context to stack frame N
(gdb) info locals         # Display all local variables in the current frame
(gdb) info args           # Display function input arguments
(gdb) p <variable>        # Print values of variables or pointers (e.g., p a[0])
(gdb) list                # Print surrounding lines of source code
(gdb) thread apply all bt # Print backtraces across all active threads

```

---

### 5. Troubleshooting Common Issues

#### Missing Core File (`No such file or directory`)

* **Cause 1:** `ulimit -c` is set to `0`. (Fix: `ulimit -c unlimited`).
* **Cause 2:** Kernel routes crash dumps to system service handlers (`systemd-coredump` / `apport`).
* *Option A (Direct GDB via systemd):* Run `coredumpctl gdb <binary_name>`.
* *Option B (Local file generation):* Run `sudo sysctl -w kernel.core_pattern=core`.



#### Stuck Pager Mode in GDB (`--Type <RET> for more...`)

If GDB pauses output and ignores typed commands:

1. Press `c` or `Enter` to complete the output.
2. Turn off pagination inside GDB:
```text
(gdb) set pagination off

```



---

### 6. Persistence Configuration

#### Permanent Core Pattern (`/etc/sysctl.conf`)

To make local core dumps default across system reboots:

```bash
echo "kernel.core_pattern = core.%e.%p" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

```

*(Pattern tokens: `%e` = executable name, `%p` = process ID).*

#### Permanent GDB Settings (`~/.gdbinit`)

To prevent pagination prompts and auto-enable debuginfod:

```bash
echo "set pagination off" >> ~/.gdbinit
echo "set debuginfod enabled on" >> ~/.gdbinit

```
