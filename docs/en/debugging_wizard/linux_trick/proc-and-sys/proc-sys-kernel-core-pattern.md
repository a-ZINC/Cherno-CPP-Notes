---
topic: proc-and-sys
tags: /proc/sys/kernel/core_pattern, core_pattern, sysctl, core dump, apport, systemd-coredump, coredumpctl
related: ../crash-debugging/core-dumps.md, proc-self-limits.md, ../process-resources/ulimit.md
updated: 2026-09-21
---
# `/proc/sys/kernel/core_pattern`

## 1. What it is

A **system-wide setting** that tells the kernel **where and how to write a core dump** when a process dies from a fatal signal. It is a single line of text. Unlike the `/proc/self/...` files, this one is **writable** (as root) and changes behaviour for the whole machine.

| Access | Command |
|--------|---------|
| Read | `cat /proc/sys/kernel/core_pattern` or `sysctl kernel.core_pattern` |
| Change until reboot | `sudo sysctl -w kernel.core_pattern=core.%e.%p` |
| Change permanently | A file in `/etc/sysctl.d/`, then `sudo sysctl --system` |

**Where it acts:** at crash time the kernel reads this pattern, checks the process's core-size limit (`Max core file size` in [limits](proc-self-limits.md)), and then either writes a file or starts a handler program.

## 2. The two forms

| Value | Meaning |
|-------|---------|
| A file name or path, for example `core` or `/tmp/core.%e.%p` | The kernel **writes a file**. A relative name is created in the crashing process's **current directory** |
| Starts with a pipe: `\|/path/to/handler args...` | The kernel **starts the handler and pipes the dump into it**. No local file appears. Ubuntu's default hands dumps to Apport; many other systems use `systemd-coredump` |

**Verified:** in the test environment the value was plain `core`, and a crash left a 448 KiB `core` file in the current directory.

## 3. Pattern tokens

| Token | Expands to |
|-------|-----------|
| `%e` | Executable name |
| `%p` | Process ID (as seen in the process's own namespace) |
| `%t` | Time of the dump (seconds since the epoch) |
| `%s` | Number of the signal that killed it |
| `%u` | User ID |
| `%h` | Hostname |
| `%%` | A literal `%` |

Example: `core.%e.%p.%t` produces names like `core.crash.4211.1790000000`.

## 4. Commands you actually use

```bash
cat /proc/sys/kernel/core_pattern                       # ALWAYS save the old value first
sudo sysctl -w kernel.core_pattern=/tmp/core.%e.%p      # use an absolute path: easy to find
sudo sysctl -w kernel.core_pattern='<the old value>'    # restore
coredumpctl list                                        # when systemd-coredump is the handler
coredumpctl gdb <name>                                  # open the latest dump in gdb
```

## 5. Gotchas

- **Changing it affects every process on the machine**, and may disable the system's crash reporter (Apport). Save and restore the old value.
- **It is not enough on its own.** The process's core limit must also allow a dump (`ulimit -c unlimited`), or nothing is written.
- **A relative pattern writes into the crashing process's current directory**, which may be somewhere unexpected or unwritable. Use an absolute path.
- **It can revert after reboot** if a service (Apport) sets it again. A drop-in file in `/etc/sysctl.d/` persists across reboots only if nothing overrides it later.
- **Dumps may contain secrets** (passwords, keys in memory). Choose a directory with restricted access.

## 6. Related

- [core-dumps](../crash-debugging/core-dumps.md): the full workflow, from crash to GDB.
- [proc-self-limits](proc-self-limits.md): `Max core file size`.
- [ulimit](../process-resources/ulimit.md): `-c`.
