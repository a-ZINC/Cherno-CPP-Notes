# PART 4 — Blocking I/O

## Chapter 4.7 — Blocking Socket I/O

> This chapter is written so you never need to look anything else up for socket programming basics. It's long on purpose — read it once, slowly, top to bottom, and you'll have the entire mental model plus every syscall you need for the rest of this course.

### 🧠 One-Sentence Mental Model
> Blocking `accept()`/`recv()`/`send()` on a socket run the exact same four-step block/wait/wake mechanism as file I/O (Chapters 4.2, 4.6), just dispatched through the SOCKET `file_operations` implementation (Chapter 3.8) instead of the regular-file one — and this chapter's project is the single-threaded blocking TCP echo server every later chapter in this course (through Part 14) will keep rebuilding with progressively better concurrency strategies.

### 🧒 Explain Like I'm Five
`accept()` is like standing at a door waiting for the next customer to walk in — you don't do anything else until someone arrives. `recv()` is like waiting for that customer to actually say something once they're at your counter. Both are pure waiting, exactly like waiting for a book from the library basement (Chapter 4.6) — just waiting for a person instead of a disk.

### 🌍 Real-World Analogy
A single toll booth operator: they wait for a car to pull up (`accept()`), then wait for the driver to hand over payment (`recv()`), hand back change (`send()`), and only then are free to look for the next car. One operator, one car at a time, zero multitasking — this IS a legitimate design (Chapter 4.1) for low traffic, and it's exactly what this chapter's server does.

### ❓ The Problem
Chapter 4.6 grounded the abstract blocking mechanism in file I/O. This chapter does the same for the networking case that every server chapter for the rest of this course builds on: what does a real, minimal, correct blocking TCP server (and client!) look like, and precisely where does it block?

### 🔥 Why This Problem Matters
Every socket API you'll use in Parts 5-14 (`select`, `epoll`, `io_uring`) is a variation on managing MULTIPLE instances of these exact same three calls (`accept`/`recv`/`send`) without needing one whole thread per connection. You cannot appreciate what those mechanisms buy you without first having this single-threaded baseline clearly in hand.

### 🕰 Historical Context
The Berkeley sockets API (`socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`, `recv()`) was introduced in 4.2BSD Unix in 1983, deliberately designed to look and feel like ordinary file descriptors (Part 3's "everything is a file" philosophy) so that existing blocking-I/O intuition (and even generic tools like `read()`/`write()`) would transfer directly to network programming with minimal new concepts.

### 💡 The Naive Solution
Write the simplest possible server: `accept()` one connection, fully serve it with blocking `recv()`/`send()` calls, `close()` it, then loop back to `accept()` the next one.

### ❌ Why the Naive Solution Fails
It's not "wrong" — it's the correct starting point — but it can only ever serve ONE client at a time; a second client that connects while the first is still being served simply waits in the kernel's connection backlog queue until the first is done, however long that takes (Chapters 4.8-4.9 fix exactly this).

### ✅ The Better Solution (this chapter's honest scope)
Build the single-connection-at-a-time server correctly and completely first — it's the necessary foundation, and it IS the right answer for genuinely low-concurrency use cases (Chapter 4.1's principle, applied).

### 🧠 Core Concept
> **`accept()` blocks until a new connection arrives in the listening socket's queue; `recv()` blocks until bytes arrive on that specific connected socket (or the peer closes it); `send()` usually does NOT block unless the kernel's send buffer is full (Chapter 3.8's backpressure). All three, when they do block, use the identical mechanism traced since Chapter 4.2 — only the specific wait queue and wakeup trigger (Part 1's NIC/DMA/interrupt chain, Chapter 3.8) differ from the file case.**

---

## 🔌 Socket Programming — The Complete Reference

Everything below is self-contained. Read it once, in order — every later part of this course (select, poll, epoll, TCP/IP, HTTP, io_uring) reuses these exact same calls and structures without re-explaining them.

### 1. What a socket actually IS (recap + new detail)

A socket is a file descriptor (Chapter 3.1) whose `file_operations` table (Chapter 3.4) dispatches to the kernel's network stack instead of a filesystem. Creating one with `socket()` does NOT connect you to anything yet — it just allocates the kernel object and hands you an fd, exactly like `open()` allocates a `struct file` without necessarily touching a device yet.

```mermaid
flowchart LR
    A["socket()"] --> B["fd (unconnected)"]
    B --> C{"Server or client?"}
    C -->|Server| D["bind() + listen() + accept()"]
    C -->|Client| E["connect()"]
    D --> F["send()/recv() on the connection"]
    E --> F
    F --> G["close() / shutdown()"]
```

### 2. `socket()` — every argument, every common value

```c
int socket(int domain, int type, int protocol);
```

**`domain`** — which address family:

| Constant | Meaning | When to use it |
|---|---|---|
| `AF_INET` | IPv4 | The overwhelming default for internet/LAN servers |
| `AF_INET6` | IPv6 | Modern dual-stack servers; can also accept IPv4 via mapped addresses |
| `AF_UNIX` (= `AF_LOCAL`) | Local machine only, addressed by a filesystem path | IPC between processes on the SAME machine — no network stack at all, faster than loopback TCP (Chapter 4.9's isolation discussion, Section 9 below) |

**`type`** — the communication style:

| Constant | Meaning |
|---|---|
| `SOCK_STREAM` | Reliable, ordered, connection-oriented byte STREAM — TCP, when paired with `AF_INET`. This chapter's entire focus. |
| `SOCK_DGRAM` | Unreliable, unordered, connectionless MESSAGES — UDP, when paired with `AF_INET`. Covered briefly in Section 8; full depth in Part 9. |
| `SOCK_RAW` | Direct access to IP-layer packets, bypassing TCP/UDP — needs root, used by tools like `ping`; out of scope for this course's application-level focus. |

**`protocol`** — almost always `0` ("pick the default for this domain+type combo," which resolves to TCP for `AF_INET`+`SOCK_STREAM`, UDP for `AF_INET`+`SOCK_DGRAM`). You'd only ever set this explicitly for unusual protocols.

**Return value:** a new fd on success; `-1` and `errno` set on failure (e.g. `EMFILE` — this process has hit its open-fd limit, `EPROTONOSUPPORT` — invalid domain/type/protocol combination).

### 3. The address structures — `sockaddr` and its family

Every socket function that takes an address uses a generic pointer type, `struct sockaddr*`, but you almost never construct one directly — you construct the SPECIFIC structure for your address family and cast it. This is a deliberate C-era polymorphism trick, worth understanding once so it never confuses you again:

```c
struct sockaddr {
    sa_family_t sa_family;   // AF_INET, AF_INET6, AF_UNIX...
    char        sa_data[14]; // opaque, family-specific bytes
};

// what you ACTUALLY fill in for IPv4:
struct sockaddr_in {
    sa_family_t    sin_family;   // AF_INET
    in_port_t      sin_port;     // port, in NETWORK byte order (Section 4)
    struct in_addr sin_addr;     // the 32-bit IPv4 address
};

// for IPv6:
struct sockaddr_in6 {
    sa_family_t     sin6_family;  // AF_INET6
    in_port_t       sin6_port;
    struct in6_addr sin6_addr;    // 128-bit IPv6 address
    // + scope/flow fields, rarely touched directly
};

// for AF_UNIX:
struct sockaddr_un {
    sa_family_t sun_family;         // AF_UNIX
    char        sun_path[108];      // a filesystem path, e.g. "/tmp/mysocket"
};
```

You fill in the SPECIFIC struct (`sockaddr_in`, say), then pass `(struct sockaddr*)&addr` to `bind()`/`connect()`/`accept()` — the function reads `sa_family` first to know which concrete layout the rest of the bytes actually have. This is why every one of these calls also takes a `socklen_t` length argument — the generic pointer alone doesn't tell the kernel how many bytes are actually valid to read.

`struct sockaddr_storage` is a fixed-size structure large enough to hold ANY address type — useful when writing code that doesn't know in advance whether it'll get an IPv4 or IPv6 peer (e.g., the output of `accept()` when the listening socket is dual-stack).

### 4. Byte order — `htons`, `htonl`, `ntohs`, `ntohl`

Different CPU architectures store multi-byte integers with the bytes in different orders (Part 1 territory: **little-endian** x86 stores the least-significant byte first; some other architectures are **big-endian**, most-significant byte first). Network protocols had to pick ONE fixed order so any two machines can talk regardless of their own native order — they chose big-endian, historically called **"network byte order."**

```c
addr.sin_port = htons(8080);        // "host to network, SHORT (16-bit)" -- for ports
addr.sin_addr.s_addr = htonl(ip32); // "host to network, LONG (32-bit)" -- for raw IPv4 addresses
uint16_t port = ntohs(addr.sin_port);  // the reverse, when READING a received value
```

On a little-endian machine (essentially all consumer x86/ARM today), `htons(8080)` genuinely swaps the two bytes; on a (rare) big-endian machine it's a no-op — the function is defined to always do the right thing regardless, which is why you must ALWAYS call it rather than assuming your machine's order matches the network's. Forgetting this is a classic bug: your server silently binds to the byte-swapped port number instead of the one you intended.

### 5. Turning text IP addresses into structures — `inet_pton`, `inet_ntop`, and the special constants

```c
struct in_addr ip;
inet_pton(AF_INET, "192.168.1.10", &ip);   // text -> binary ("presentation TO network")

char text[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &ip, text, sizeof(text)); // binary -> text ("network TO presentation")
```

Two special IPv4 constants you'll use constantly:
- **`INADDR_ANY`** (`0.0.0.0`) — "bind to ALL of this machine's network interfaces" (Ethernet, WiFi, loopback, all of them) — what this chapter's server uses, since it doesn't care which interface a client arrives on.
- **`INADDR_LOOPBACK`** (`127.0.0.1`) — the loopback interface, reachable only from THIS machine — useful for local-only testing/services.

### 6. The modern, portable way to resolve addresses — `getaddrinfo()`

Hardcoding `AF_INET` and manually filling `sockaddr_in` (as this chapter's example code does, for teaching clarity) works, but real-world production code almost always uses `getaddrinfo()` instead, because it transparently handles IPv4 vs. IPv6, DNS hostname resolution, and service-name-to-port lookup (`"http"` → `80`) all in one call:

```c
struct addrinfo hints{}, *result;
hints.ai_family   = AF_UNSPEC;    // AF_UNSPEC = "give me IPv4 OR IPv6, whichever works"
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags    = AI_PASSIVE;   // "I'm a server; fill in INADDR_ANY-equivalent for me"

int rc = getaddrinfo(nullptr, "8080", &hints, &result);
if (rc != 0) { fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rc)); exit(1); }

// result is a LINKED LIST of candidate addresses (a hostname can resolve
// to multiple IPs) -- try each until one works:
int fd = -1;
for (struct addrinfo* p = result; p != nullptr; p = p->ai_next) {
    fd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
    if (fd < 0) continue;
    if (bind(fd, p->ai_addr, p->ai_addrlen) == 0) break;   // success
    close(fd); fd = -1;
}
freeaddrinfo(result);   // ALWAYS free the list once you're done with it
```
This chapter's project code uses the simpler, manual `sockaddr_in` approach specifically so every field is visible and explained — but know that `getaddrinfo()` is what you reach for once you need real portability (Part 9 uses it more).

### 7. The full server-side call sequence, with every error code that matters

```mermaid
sequenceDiagram
    participant Server
    participant Kernel
    participant NIC as NIC (Part 1)

    Server->>Kernel: socket() -> fd
    Server->>Kernel: setsockopt(SO_REUSEADDR)
    Server->>Kernel: bind(fd, local_addr)
    Server->>Kernel: listen(fd, backlog)
    Server->>Kernel: accept(fd)
    Note over Server,Kernel: BLOCKS -- no connection queued yet<br/>(Ch 4.2's mechanism, joined to listen_fd's wait queue)
    NIC->>Kernel: SYN packet arrives, TCP handshake completes
    Kernel-->>Server: accept() returns new connected_fd (Ch 4.5's wakeup)
    Server->>Kernel: recv(connected_fd, buf, len)
    Note over Server,Kernel: BLOCKS if no data yet
    NIC->>Kernel: data packet arrives (Part 1's DMA/interrupt chain)
    Kernel-->>Server: recv() returns with data
    Server->>Kernel: send(connected_fd, response, len)
    Note over Server,Kernel: usually returns immediately --<br/>only blocks if the send buffer is full
```

```c
int socket(int domain, int type, int protocol);
```
Covered fully in Section 2.

```c
int setsockopt(int fd, int level, int optname, const void *optval, socklen_t optlen);
```
`level` is almost always `SOL_SOCKET` for these (a few, like `TCP_NODELAY` below, use `IPPROTO_TCP` instead):

| `optname` | What it does | Why you'd set it |
|---|---|---|
| `SO_REUSEADDR` | Lets `bind()` succeed on a port still in the brief `TIME_WAIT` state (Section 10) after a just-restarted server | Without it, quickly restarting a server often fails with `EADDRINUSE` — this chapter's code sets this one |
| `SO_REUSEPORT` | Lets MULTIPLE independent sockets bind the SAME port, kernel load-balances connections across them | Multi-process servers, each running its own `accept()` loop, no shared listening socket needed (Part 21) |
| `SO_KEEPALIVE` | Kernel periodically probes an idle connection | Detect a peer that silently vanished (crash, network partition) rather than waiting forever |
| `SO_RCVBUF` / `SO_SNDBUF` | Explicitly set the kernel receive/send buffer size | Tune throughput on high-latency links (Chapter 3.8's backpressure buffers; Part 21 quantifies the trade-off) |
| `TCP_NODELAY` (level `IPPROTO_TCP`) | Disables **Nagle's algorithm** — a default TCP behavior that batches small writes together for a few milliseconds before sending, to reduce packet overhead | Set this when you need every small message sent IMMEDIATELY (e.g., an interactive protocol) — trades a small amount of network efficiency for lower latency |

```c
int bind(int fd, const struct sockaddr *addr, socklen_t addrlen);
```
Assigns a local address (IP + port) to the socket. Common failure: `EADDRINUSE` — something is already bound to this exact IP+port combination (see `SO_REUSEADDR` above for the common "just restarted my server" case of this).

```c
int listen(int fd, int backlog);
```
Marks the socket as passive (will `accept()` connections, will never `connect()` out). **`backlog`** is the maximum number of FULLY-established connections the kernel will queue, waiting for your process to call `accept()`, before it starts refusing new ones — exactly the queue Chapter 4.7's earlier experiment showed a second client sitting in while the server was busy with the first. (Technically, modern Linux actually maintains TWO internal queues — an incomplete-handshake SYN queue and a completed-handshake accept queue — `backlog` sizes the second one; Part 9 covers the handshake itself in full.)

```c
int accept(int fd, struct sockaddr *addr, socklen_t *addrlen);
```
Pulls the next fully-established connection off the accept queue. `addr`/`addrlen` (often `nullptr`/`nullptr` when you don't need it) are filled in with the CONNECTING CLIENT's address if non-null — useful for logging or IP-based filtering. **Return value:** a BRAND NEW file descriptor for this one connection — the original listening `fd` is untouched and keeps listening for more.

```c
ssize_t recv(int fd, void *buf, size_t len, int flags);
ssize_t send(int fd, const void *buf, size_t len, int flags);
```
Behave like `read()`/`write()` (Chapter 4.6) with an extra `flags` argument — `0` for ordinary behavior, or OR'd combinations of:

| Flag | Effect |
|---|---|
| `MSG_DONTWAIT` | Make just THIS ONE call nonblocking, without changing the socket's overall mode — a lightweight preview of Part 5's `O_NONBLOCK` |
| `MSG_WAITALL` (recv only) | Don't return until the FULL requested length has arrived (or error/EOF) — disables "short read" for this one call |
| `MSG_PEEK` (recv only) | Look at incoming data WITHOUT removing it from the receive buffer — a later `recv()` sees the same bytes again |
| `MSG_NOSIGNAL` (send only, Linux) | Suppress `SIGPIPE` (Section 11) on send-to-a-closed-peer; get `EPIPE` instead |

### 8. Client-side: `connect()`, and a complete matching client program

Everything above was server-side. The client side is actually SIMPLER — no `bind()`/`listen()`/`accept()` needed (the kernel auto-assigns a local port via an implicit bind inside `connect()`):

```c
int connect(int fd, const struct sockaddr *addr, socklen_t addrlen);
```
Initiates the TCP three-way handshake (SYN → SYN-ACK → ACK, full detail in Part 9) to the given remote address. **This call BLOCKS** (Chapter 4.2's mechanism again) until the handshake completes or fails. Common failures:
- `ECONNREFUSED` — nothing is listening on that port at that address (no process has it bound + listening).
- `ETIMEDOUT` — no response at all within the OS's connection timeout (firewall silently dropping packets, or the host is unreachable).
- `ENETUNREACH` / `EHOSTUNREACH` — routing-level failure, no path to that network/host at all.

**Project: a complete, matching TCP client** (test the server below with this instead of `nc`, to see explicit error handling):

```cpp
// blocking_echo_client.cpp
// Compile: g++ -O2 -std=c++20 blocking_echo_client.cpp -o blocking_echo_client
// Run:     ./blocking_echo_client 127.0.0.1 9000

#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <cstdlib>

int main(int argc, char** argv) {
    if (argc < 3) { fprintf(stderr, "usage: %s <ip> <port>\n", argv[0]); return 1; }

    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) { perror("socket"); return 1; }

    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(atoi(argv[2]));           // Section 4: byte order matters
    if (inet_pton(AF_INET, argv[1], &server_addr.sin_addr) != 1) {  // Section 5
        fprintf(stderr, "invalid IP address: %s\n", argv[1]);
        return 1;
    }

    if (connect(fd, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");   // e.g. "Connection refused" if nothing's listening
        return 1;
    }
    printf("Connected. Type a message and press Enter (Ctrl-D to quit).\n");

    char line[4096];
    while (fgets(line, sizeof(line), stdin)) {
        size_t len = strlen(line);
        ssize_t sent_total = 0;
        while ((size_t)sent_total < len) {
            ssize_t s = send(fd, line + sent_total, len - sent_total, 0);
            if (s < 0) { perror("send"); close(fd); return 1; }
            sent_total += s;
        }

        char buf[4096];
        ssize_t n = recv(fd, buf, sizeof(buf), 0);   // BLOCKS here until the echo arrives
        if (n <= 0) { printf("Server closed the connection.\n"); break; }
        fwrite(buf, 1, n, stdout);
    }
    close(fd);
    return 0;
}
```

**Predict before running:** what happens if you run this client WITHOUT starting the server first?
<details><summary>Click to reveal the answer</summary>
`connect()` returns `-1` immediately (or after a short delay) with `errno == ECONNREFUSED`, because the OS on the target machine actively rejects the connection attempt (a TCP RST packet) when nothing is listening on that port — you do NOT get a generic "timeout," which only happens when there's no response at all (a firewall silently d
