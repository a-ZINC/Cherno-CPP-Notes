# Chapter 01 · The 10,000-Foot View, Your Lab & Lumen Kickoff
 
> **Part 0 — Orientation** · ⏱️ ~6–8 hours (including VM install) · 🏗️ Milestone **m01**
> Prereqs: C++ basics, a terminal, Ubuntu 24.04+
 
---
 
## 🎯 Mission
 
**Understand:**
- [ ] The **story of a keypress** and the **story of a frame**, with every major component named
- [ ] The **four layers** of the Linux graphics & input stack, and which real Ubuntu programs live in each
- [ ] The difference between *protocols*, *servers*, *toolkits*, *APIs* and *drivers* (Wayland vs GNOME vs GTK vs OpenGL vs Mesa)
- [ ] DRM **primary vs render nodes**, **DRM master**, and why Ubuntu lets you open the GPU without `sudo` but not the keyboard
- [ ] How userspace talks to devices: `open`, `read`, `ioctl`, `mmap`, and the three C error conventions
**Build (Lumen m01):**
- [ ] A disposable **Ubuntu Server VM** where Lumen can take over the screen safely
- [ ] The **Lumen repository**: CMake module system, strict warnings, sanitizers
- [ ] **`lumen::core`**: `UniqueFd`, `CPtr<T, FreeFn>`, errno→exception helpers, a categorised logger
- [ ] **`lumen-info drm`**: lists GPUs, drivers, capabilities and connected monitors
- [ ] **`lumen-info input`**: lists and classifies input devices, and dumps raw kernel events
- [ ] **Catch2 tests**, **clang-format**, **GitHub Actions CI**, a README and your first **ADR**, all committed and tagged `m01`
---
 
# Part A — The big picture
 
## 🧠 Mental model: a theatre and a post office
 
Two pictures before any jargon.
 
### 🎭 Output is a theatre
 
| Theatre | Linux graphics |
|---|---|
| The **stage** the audience looks at | The **display panel** (your monitor) |
| **Painted backdrops** | **Framebuffers**: chunks of memory full of pixel values |
| The **stage manager**, who swaps backdrops only when the curtain is down, on a strict schedule | **KMS** (Kernel Mode Setting), which swaps buffers only during **vblank**, 60+ times a second |
| The **director**, who arranges every actor's backdrop into one scene | The **compositor** (on Ubuntu: GNOME's *Mutter*, inside `gnome-shell`). By Ch 22, **Lumen** |
| **Actors**, each painting their own small backdrop | **Applications** (Firefox, your terminal, a game) |
| The **paint workshop** with 1,000 painters working in parallel | The **GPU** |
| The **interpreter** turning the director's notes into workshop instructions | **Mesa** (the OpenGL/Vulkan driver in userspace) |
 
The key insight: **apps never paint on the stage directly.** They paint their own backdrop (a buffer) and hand it to the director (compositor). The director arranges everything into one final image, and the stage manager (KMS) puts it on stage at exactly the right moment.
 
### 📮 Input is a post office
 
| Post office | Linux input |
|---|---|
| A letter dropped in a mailbox | You press a key: the keyboard sends a **scancode** over USB |
| The sorting machine stamps a standard postcode on it | The kernel converts it to a standard **keycode** (`KEY_A` = 30) |
| The local depot that receives all mail | **evdev**: the `/dev/input/eventN` files |
| The clerk who knows the local customs: "this postcode means *this* street in *this* country" | **libinput** (makes sense of mice, touchpads, gestures) + **xkbcommon** (turns `KEY_A` into "a", "q" or "ф" depending on your layout) |
| The mail carrier who decides which house gets it | The **compositor** delivers the event to the window with **focus** |
| The resident who reads it and reacts | The **application** |
 
The key insight: **the kernel has no idea what letter you typed.** It only knows *which physical key position* went down. Turning that into a character is a userspace job, and in Ch 19 it becomes Lumen's job.
 
---
 
## 🗺️ The story of a keypress (and the frame that answers it)
 
You're in a terminal on Ubuntu. You press `A`. A moment later an `a` appears. Here is everything that happens in between. You'll implement or dissect every step during this series:
 
````mermaid
sequenceDiagram
    autonumber
    participant KB as ⌨️ Keyboard
    participant K as Kernel input<br/>(usbhid → evdev)
    participant M as Compositor<br/>(libinput + xkbcommon)
    participant T as Terminal app<br/>(GTK)
    participant G as Mesa + GPU
    participant D as Kernel DRM/KMS
    participant P as 🖥️ Panel
 
    KB->>K: USB HID report: usage 0x04 down
    K->>K: HID usage → keycode KEY_A (30)
    K->>M: input_event EV_KEY KEY_A 1, then EV_SYN, on /dev/input/event3
    M->>M: libinput: a keyboard key was pressed
    M->>M: xkbcommon: KEY_A + US layout → keysym 'a'
    M->>T: Wayland wl_keyboard.key(30, pressed) to the FOCUSED window
    T->>T: its own xkbcommon → 'a', appends text, schedules redraw
    T->>G: OpenGL/Vulkan draw calls into the window's buffer
    G-->>T: rendering finished (fence signals)
    T->>M: Wayland wl_surface.attach(buffer) + commit
    M->>G: composite all windows + cursor into one screen buffer
    M->>D: atomic commit: show this buffer at the next vblank
    D->>D: wait for the vertical blanking interval
    D->>P: display controller streams pixels over HDMI / eDP / DP
    P->>P: crystals / OLED cells change: you see 'a' ✨
    D-->>M: page-flip event: it is on screen now
````
 
Every step is a chapter, and a Lumen milestone:
 
| Step(s) | Topic | Chapter / milestone |
|---|---|---|
| 1–3 | HID, scancodes, keycodes, evdev | Ch 17 · m17 |
| (discovery, permissions) | udev, logind, seats | Ch 18 · m18 |
| 4–5 | libinput, xkbcommon | Ch 19 · m19 |
| 6, 10 | Wayland protocol | Ch 21–22 · m21–22 |
| 8–9 | GPU rendering, GL/Vulkan, Mesa | Ch 23–25 · m23–25 |
| 11 | Compositing | Ch 06 (math) · Ch 22 |
| 12–13, 16 | Atomic KMS, vblank, page flips | Ch 14–15 · m14–15 |
| 14 | Display timings, cables | Ch 08 · m08 |
| 15 | Panel technologies | Ch 08 |
| fences, buffer sharing | dma-buf, sync | Ch 16 · m16 |
 
### ⏱️ The latency budget
 
How long does all that take? A realistic picture for a 60 Hz screen:
 
| Stage | Typical time | Why |
|---|---|---|
| USB polling | 1–8 ms | Keyboards are polled at 125–1000 Hz |
| Kernel → compositor | < 0.1 ms | A `read()` on an fd |
| Compositor → app | < 1 ms | A Unix socket message |
| App thinks + renders | 1–10 ms | Depends on the app |
| Compositor composites | 1–5 ms | Usually on the GPU |
| **Waiting for vblank** | **0–16.7 ms** | A new frame can only go up at the next refresh (1000 / 60 = 16.7 ms) |
| Scanout | ~16 ms top → bottom | The panel is fed line by line |
| Panel response | 1–10 ms LCD · µs OLED | Physics of crystals and diodes |
 
**Total: roughly 20–60 ms** from finger to photon. In Ch 27 you'll measure Lumen's own budget.
 
> 🔮 **Predict:** you switch your monitor from 60 Hz to 144 Hz. Which rows shrink most?
> <details><summary>Reveal</summary>"Waiting for vblank" and "scanout". Both scale with the refresh period: 16.7 ms → 6.9 ms.</details>
 
---
 
## 🏛️ The four layers
 
Everything in this series sits in one of four layers. Here they are, filled in with **the actual names on an Ubuntu desktop**:
 
````mermaid
flowchart TB
    subgraph L4["4 · Applications"]
        A1["Firefox, GNOME Terminal, games, your programs"]
        A2["Toolkits: GTK, Qt, SDL, Chromium Ozone"]
    end
    subgraph L3["3 · System userspace"]
        S1["Compositor: Mutter inside gnome-shell<br/>(alternatives: KWin, Sway, Weston, soon Lumen)"]
        S2["Xwayland: runs old X11 apps"]
        S3["Mesa: OpenGL / Vulkan drivers<br/>(iris, radeonsi, llvmpipe, virgl…)"]
        S4["libinput, xkbcommon, libdrm, GBM"]
        S5["systemd-logind, udev"]
    end
    subgraph L2["2 · Kernel"]
        K1["DRM/KMS drivers: i915 / xe (Intel), amdgpu (AMD),<br/>nouveau or nvidia-drm (NVIDIA), virtio_gpu (VMs), simpledrm"]
        K2["Input: usbhid, hid-generic, i2c-hid, atkbd → evdev"]
        K3["dma-buf, sync files, DMA, IOMMU"]
    end
    subgraph L1["1 · Hardware"]
        H1["GPU: shader cores + display controller"]
        H2["Panel / monitor + cable: eDP, HDMI, DP"]
        H3["Keyboard, mouse, touchpad, touchscreen"]
    end
    L4 --> L3 --> L2 --> L1
````
 
How information crosses each boundary:
 
| Boundary | Mechanism | Example |
|---|---|---|
| App ↔ compositor | **Wayland protocol** over a Unix socket (or X11) | `wl_surface.commit` |
| App ↔ Mesa | **Function calls** into a shared library | `glDrawArrays()` |
| Userspace ↔ kernel | **System calls** on device files: `open`, `read`, `ioctl`, `mmap` | `ioctl(fd, DRM_IOCTL_MODE_ATOMIC, …)` |
| Kernel ↔ hardware | **Registers** (MMIO), **DMA**, **interrupts** | a vblank interrupt fires |
 
> 🧙 **Wizard rule #1:** *every* interaction between userspace and graphics/input hardware on Linux goes through a **file in `/dev`**. Learn to `open`, `read`, `ioctl` and `mmap` those files and nothing in this stack can hide from you. Lumen is built entirely on that rule.
 
**Where Lumen will live:** layer 3, right where Mutter sits today, talking down to layer 2 through `/dev/dri` and `/dev/input`, and up to apps through Wayland.
 
### NVIDIA on Ubuntu: a footnote you need
 
With NVIDIA's **proprietary driver** (very common on Ubuntu), the kernel part is `nvidia-drm` and OpenGL/Vulkan come from NVIDIA instead of Mesa. DRM/KMS and Wayland work the same way (`nvidia-drm.modeset=1`, the default on recent drivers), so everything applies except the Mesa internals in Ch 25. This is one more reason to develop in the VM, where the GPU is always `virtio_gpu` + Mesa.
 
---
 
## 📖 The who's-who: stop mixing up the categories
 
Most beginner confusion comes from comparing things that aren't the same *kind* of thing:
 
````mermaid
flowchart LR
    subgraph Protocols["📜 Protocols: languages"]
        P1["X11"]
        P2["Wayland"]
    end
    subgraph Servers["🖥️ Display servers / compositors"]
        S1["Xorg"]
        S2["Mutter (GNOME)"]
        S3["KWin (KDE)"]
        S4["Sway, Weston, Lumen"]
    end
    subgraph Desktops["🏠 Desktop environments"]
        D1["GNOME (Ubuntu default)"]
        D2["KDE Plasma"]
    end
    subgraph Toolkits["🧰 UI toolkits"]
        T1["GTK"]
        T2["Qt"]
        T3["SDL"]
    end
    subgraph APIs["🎨 Rendering API specs"]
        R1["OpenGL / OpenGL ES"]
        R2["Vulkan"]
        R3["EGL (glue)"]
    end
    subgraph Impl["⚙️ API implementations"]
        I1["Mesa"]
        I2["NVIDIA proprietary"]
    end
    subgraph KernelAPI["🐧 Kernel interfaces"]
        K1["DRM / KMS"]
        K2["fbdev (legacy)"]
        K3["evdev"]
    end
````
 
| ❌ "Should I use…" | ✅ Reality |
|---|---|
| "Wayland or GNOME?" | GNOME is a desktop whose compositor (Mutter) **speaks** Wayland. Not alternatives. |
| "GTK or Wayland?" | GTK is a toolkit that **uses** Wayland (or X11) to show windows. |
| "OpenGL or Mesa?" | OpenGL is a spec; Mesa **implements** it. |
| "DRM or KMS?" | KMS is the **display-control part** of DRM. DRM also covers rendering and memory. |
| "Xorg or X11?" | X11 is the protocol; Xorg is the most common server for it. |
| "Vulkan or Wayland?" | Different layers: Vulkan renders pixels; Wayland gets them to the compositor. Used together. |
 
---
 
## 📚 Starter glossary
 
Don't memorise these. Come back here whenever a word is unfamiliar.
 
| Term | Plain-English meaning |
|---|---|
| **Pixel** | One sample of color in a picture |
| **Framebuffer** | A block of memory holding a whole picture's pixels |
| **Scanout** | Display hardware reading a framebuffer and sending it down the cable |
| **Refresh rate** | How many times per second the screen is redrawn (60 Hz, 144 Hz…) |
| **vblank** | The short pause between two refreshes: the safe moment to swap pictures |
| **Tearing** | Seeing half of one frame and half of the next, because the swap happened mid-scanout |
| **Page flip** | Atomically switching which framebuffer is scanned out, at vblank |
| **GPU** | A massively parallel processor specialised for drawing |
| **Display controller** | The (often separate) block that does scanout. *Not* the same as the GPU's shader cores! |
| **DRM** | *Direct Rendering Manager*, the kernel subsystem for GPUs and displays (nothing to do with copy protection) |
| **KMS** | *Kernel Mode Setting*, the part of DRM that sets resolution and refresh rate and chooses which buffer is shown |
| **GEM** | DRM's way of naming and managing GPU buffers |
| **dma-buf** | A file descriptor representing a buffer, shareable between devices and processes |
| **Compositor** | The program combining all windows into the final screen image |
| **Wayland** / **X11** | Modern / legacy protocol between apps and the display server |
| **Mesa** | The open-source OpenGL/Vulkan driver collection |
| **EGL / GBM** | Glue: EGL connects GL to surfaces; GBM allocates GPU buffers |
| **evdev** | The kernel's generic input event interface, `/dev/input/eventN` |
| **Keycode / keysym** | Physical key number (kernel) vs. meaning like 'a' (userspace) |
| **libinput** | Turns raw evdev events into sensible pointer/keyboard/touch events |
| **xkbcommon** | Maps keycodes to keysyms and text using a keyboard layout |
| **udev** | Discovers devices and creates `/dev` nodes |
| **logind** | systemd service that tracks sessions and hands out device access |
| **Seat** | A set of screens + input devices belonging to one session |
| **VT** | Virtual Terminal: the text consoles reached with `Ctrl+Alt+F3` |
| **ioctl** | "I/O control": the syscall for sending commands to a device |
| **RAII** | C++ idiom: an object owns a resource and its destructor releases it |
 
---
 
## 🔍 Quick experiments: explore the Ubuntu you're running
 
All **read-only and safe**. Run them on your **host** desktop.
 
### 1. What kind of session is this?
 
````bash
echo $XDG_SESSION_TYPE
loginctl show-session "$XDG_SESSION_ID" -p Type -p Seat -p VTNr -p Active -p Remote
````
 
> 🔮 **Predict first:** on stock Ubuntu, what does `XDG_SESSION_TYPE` print?
 
<details><summary>Reveal</summary>
`wayland`: you're in GNOME Shell, which contains the Mutter compositor. If you chose "Ubuntu on Xorg" at login, or an NVIDIA setup fell back, you'll see `x11`. Expect `Seat=seat0`, and `VTNr=2` (the GDM login screen sits on VT1).
</details>
### 2. Who's the compositor?
 
````bash
ps -eo pid,comm | grep -E 'gnome-shell|Xwayland|Xorg|kwin|sway'
````
 
`gnome-shell` (Mutter lives inside it), plus `Xwayland` if an old X11 app is running.
 
### 3. Your DRM devices
 
````bash
ls -l /dev/dri/ /dev/dri/by-path/
ls /sys/class/drm/
````
 
Typical laptop output:
 
````
crw-rw----+ 1 root video  226,   1 card1
crw-rw----+ 1 root render 226, 128 renderD128
 
card1  card1-eDP-1  card1-HDMI-A-1  card1-DP-1  renderD128  version
````
 
````mermaid
flowchart LR
    GPU["One GPU"] --> C["/dev/dri/card1<br/>PRIMARY node<br/>display control (KMS) + rendering<br/>group: video"]
    GPU --> R["/dev/dri/renderD128<br/>RENDER node<br/>rendering only, no display<br/>group: render"]
    C --> E1["card1-eDP-1<br/>(laptop panel)"]
    C --> E2["card1-HDMI-A-1"]
    C --> E3["card1-DP-1"]
````
 
- **`cardN`**, the *primary* node, can control screens. Only one process at a time can be **DRM master** and actually change what's displayed.
- **`renderDN`**, the *render* node, does GPU rendering only. Any number of unprivileged apps can share it safely; Firefox and games open this.
- `card1-eDP-1` etc. are **connectors**: the physical outputs.
> ❓ **Why `card1`, not `card0`?** Early in boot, a tiny driver called **simpledrm** claims `card0` to show the splash screen; the real GPU driver gets the next number. **Card numbers are not stable.** Never hardcode them (Lumen will use udev from m18).
 
````bash
for c in /sys/class/drm/card*-*; do
  echo "$(basename $c): $(cat $c/status)  $(head -1 $c/modes)"
done
````
 
### 4. Who owns the display right now?
 
````bash
sudo fuser -v /dev/dri/card*
sudo cat /sys/kernel/debug/dri/*/clients 2>/dev/null
````
 
The `clients` file lists every process with the device open; the `master` column shows **`gnome-shell`**. This is why, later, Lumen can't take over your desktop's screen: the director doesn't share the stage manager.
 
### 5. Why can you open `/dev/dri/card1` without sudo?
 
Notice the `+` in `crw-rw----+`. It means an **ACL** (Access Control List) is set on top of the normal permissions:
 
````bash
getfacl /dev/dri/card1
````
 
````
# owner: root
# group: video
user::rw-
user:yourname:rw-      ← you! added by systemd-logind
group::rw-
````
 
`systemd-logind` grants the user of the **active local session** access to the GPU, and takes it away when you switch user. Keyboards and mice in `/dev/input` do **not** get this ACL, because anyone who can read them can log every keystroke. A compositor gets them by *asking logind*. That's Ch 18 (and Lumen's m18).
 
### 6. Your input devices
 
````bash
grep -E '^(N|H):' /proc/bus/input/devices
ls -l /dev/input/by-id/ /dev/input/by-path/
````
 
Each `N: Name="..."` is one device, and its `H: Handlers=... eventN` line tells you which `/dev/input/eventN` file to read. One physical keyboard often appears as **several** devices (main keys, media keys, "system control"). That's normal.
 
### 7. Your monitor introduces itself: EDID
 
````bash
sudo apt install edid-decode
edid-decode /sys/class/drm/card1-eDP-1/edid    # pick a 'connected' connector
````
 
Manufacturer, model, physical size, supported modes: the monitor sends this blob over a tiny side channel in the cable. In m08 you write your own decoder for `lumen-info`.
 
---
 
# Part B — Your lab
 
## 🧪 Build your safe playground (QEMU VM)
 
From m12 onward, Lumen **takes over the screen** and **grabs input**. On your desktop that means fighting GNOME, and a bug can freeze your session. In a VM, Lumen is master of the VM's virtual screen, a hang just means closing a window, and your editor stays on the host.
 
````mermaid
flowchart LR
    subgraph Host["🖥️ Your Ubuntu desktop"]
        ED["VS Code<br/>(Remote-SSH)"]
        TERM["Terminal:<br/>ssh -p 2222 you@localhost"]
        QW["QEMU window =<br/>the VM's monitor"]
    end
    subgraph VM["📦 Ubuntu Server VM: no desktop"]
        SRC["~/lumen repo"]
        BIN["lumen / lumen-info<br/>DRM master on virtio_gpu"]
        VT["tty1 console"]
    end
    ED -- "edit over SSH" --> SRC
    TERM -- "build & run over SSH" --> BIN
    BIN -- "draws on" --> QW
    QW -- "keyboard/mouse when focused" --> BIN
````
 
**The trick:** you drive the VM over SSH, so even while Lumen owns the VM's screen and keyboard you still have a working terminal to start, stop and debug it.
 
### Step 1 · Install QEMU on the host
 
````bash
sudo apt install qemu-system-x86 qemu-utils
sudo usermod -aG kvm $USER      # hardware virtualization without sudo
# log out and back in, then:
ls -l /dev/kvm && groups | grep kvm
````
 
> No `/dev/kvm`? Enable **VT-x / AMD-V** in your BIOS. Without KVM it works, about 10× slower.
 
### Step 2 · Get Ubuntu Server and create a disk
 
Download the **Ubuntu Server** ISO (same version as your desktop is fine) from `ubuntu.com/download/server` into `~/vm/`:
 
````bash
mkdir -p ~/vm && cd ~/vm
qemu-img create -f qcow2 gfxlab.qcow2 30G
````
 
### Step 3 · Install
 
````bash
qemu-system-x86_64 \
  -enable-kvm -cpu host -smp 4 -m 4G \
  -drive file=gfxlab.qcow2,if=virtio \
  -cdrom ubuntu-*-live-server-amd64.iso -boot d \
  -nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22 \
  -device virtio-vga \
  -display gtk
````
 
In the installer, accept the defaults, set hostname `gfxlab`, and ✅ **tick "Install OpenSSH server"**. Close QEMU when it asks to reboot.
 
### Step 4 · Daily launch script
 
`~/vm/run-gfxlab.sh` (then `chmod +x ~/vm/run-gfxlab.sh`):
 
````bash
#!/usr/bin/env bash
# Launch the Lumen lab VM.
#  virtio-vga                     → 'virtio_gpu' DRM/KMS driver inside the VM
#  virtio-keyboard/mouse/tablet   → real evdev devices inside the VM
#  hostfwd 2222→22                → ssh -p 2222 you@localhost
cd "$(dirname "$0")"
exec qemu-system-x86_64 \
  -enable-kvm -cpu host -smp 4 -m 4G \
  -drive file=gfxlab.qcow2,if=virtio \
  -nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22 \
  -device virtio-vga \
  -device virtio-keyboard-pci \
  -device virtio-mouse-pci \
  -device virtio-tablet-pci \
  -display gtk,zoom-to-fit=off \
  "$@"
````
 
> 🖱️ Clicking into the QEMU window captures keyboard + mouse; **`Ctrl+Alt+G`** releases them.
> 🔭 In Ch 24 we switch to `-device virtio-vga-gl -display gtk,gl=on` for **real GPU acceleration** (virgl).
 
### Step 5 · Connect and install the toolbox
 
````bash
~/vm/run-gfxlab.sh &
ssh -p 2222 yourname@localhost
````
 
Inside the VM, install everything the whole series needs:
 
````bash
sudo apt update
sudo apt install -y \
  build-essential g++ cmake ninja-build pkg-config git gdb strace valgrind \
  catch2 clang-format clang-tidy \
  libdrm-dev libdrm-tests drm-info \
  libgbm-dev libegl-dev libgles-dev libvulkan-dev vulkan-tools mesa-utils \
  libinput-dev libinput-tools libudev-dev libxkbcommon-dev libseat-dev \
  libwayland-dev wayland-protocols wayland-utils weston \
  evtest edid-decode
````
 
| Packages | Used for |
|---|---|
| `build-essential g++ cmake ninja-build pkg-config git gdb strace valgrind` | building & debugging Lumen |
| `catch2 clang-format clang-tidy` | tests & code quality (m01) |
| `libdrm-dev libdrm-tests drm-info` | DRM/KMS (m13–m16); `libdrm-tests` gives **`modetest`** |
| `libgbm-dev libegl-dev libgles-dev libvulkan-dev vulkan-tools mesa-utils` | GPU renderer (m24–m25) |
| `libinput-dev libinput-tools libudev-dev libxkbcommon-dev libseat-dev` | input & sessions (m17–m19) |
| `libwayland-dev wayland-protocols wayland-utils weston` | Wayland (m21–m22) |
| `evtest edid-decode` | reference tools to compare `lumen-info` against |
 
Give your VM user device access:
 
````bash
sudo usermod -aG video,render,input $USER
exit   # log out of ssh, reconnect, then check:
groups
````
 
> ⚠️ **Security:** group `input` can read *every keystroke on the machine*. Fine in a throwaway VM. **Never do this on your real desktop.** There, use `sudo` until m18 teaches Lumen to ask logind properly.
 
### Step 6 · Bonus device: vkms
 
````bash
sudo modprobe vkms || { sudo apt install -y linux-modules-extra-$(uname -r) && sudo modprobe vkms; }
ls /dev/dri/
````
 
**vkms** (Virtual KMS) is a pure-software display driver: a fake display controller and monitor inside the kernel. You can modeset and page-flip, and even read back what *would* have been shown. Kernel developers test KMS with it, and from m15 Lumen's CI will too.
 
### Step 7 · Your first real tools
 
Inside the VM:
 
````bash
drm_info | less               # everything the DRM drivers expose, as a tree
modetest -c                   # connectors and their modes
modetest -p                   # CRTCs and planes
sudo evtest                   # pick a device, press keys
sudo libinput list-devices
sudo libinput debug-events    # move the mouse in the QEMU window
````
 
> 🎉 **The fun one.** With nothing else using the VM's screen, find the connector id with `modetest -c` and run:
> ```bash
> sudo modetest -M virtio_gpu -s <connector_id>:1280x800
> ```
> The QEMU window shows color bars. That's KMS in action. Press Enter in SSH to stop. In m14 Lumen does this itself.
 
---
 
# Part C — 🏗️ Lumen milestone m01
 
## What we're building today
 
````mermaid
flowchart TB
    subgraph Repo["lumen/ repository"]
        subgraph Build["Build & quality"]
            CM["CMakeLists.txt<br/>+ cmake/LumenHelpers.cmake"]
            CF[".clang-format"]
            CI[".github/workflows/ci.yml"]
        end
        subgraph Core["src/core → lumen::core"]
            UF["unique_fd.hpp<br/>UniqueFd, open_fd()"]
            CP["c_ptr.hpp<br/>CPtr‹T, FreeFn›"]
            ER["error.hpp<br/>check_syscall / check_neg_errno /<br/>check_ptr / retry_eintr"]
            LG["log.hpp<br/>info / warn / error / debug(Cat)"]
        end
        subgraph App["apps/lumen-info"]
            MAIN["main.cpp<br/>sub-command dispatch"]
            DRM["cmd_drm.cpp<br/>DRM nodes, caps, connectors"]
            INP["cmd_input.cpp<br/>list + classify + dump evdev"]
        end
        subgraph Tests["tests/unit (Catch2)"]
            T1["test_unique_fd / test_error / test_log"]
        end
        subgraph Docs["docs/"]
            ADR["adr/0001-language-build-and-layout.md"]
            ARCH["architecture.md"]
        end
    end
    App --> Core
    Tests --> Core
    DRM -->|"libdrm → ioctl"| KD["/dev/dri/*"]
    INP -->|"ioctl + read"| KI["/dev/input/event*"]
````
 
By the end of this part, inside the VM:
 
````
$ lumen-info drm
/dev/dri/card0
  driver : virtio_gpu 0.1.0  (virtio GPU)
  node   : primary (display + render)
  caps   : …
  master : YES (we may change what is on screen)
  KMS: 1 connectors, 1 encoders, 1 CRTCs
    ├─ connector 36  Virtual-1  connected  0mm×0mm, 25 modes, preferred 1280x800@60
 
$ lumen-info input
DEVICE               BUS        VID:PID    TYPE               NAME
/dev/input/event0    host       0000:0001  keyboard           AT Translated Set 2 keyboard
/dev/input/event2    virtual    0627:0001  keyboard           QEMU Virtio Keyboard
/dev/input/event3    virtual    0627:0001  mouse              QEMU Virtio Mouse
/dev/input/event4    virtual    0627:0001  absolute pointer   QEMU Virtio Tablet
````
 
---
 
## Step 1 · Create the repository
 
Work **inside the VM** (VS Code Remote-SSH: *Connect to Host…* → `ssh -p 2222 you@localhost`).
 
````bash
mkdir -p ~/lumen && cd ~/lumen
git init -b main
git config user.name  "Your Name"
git config user.email "you@example.com"
 
mkdir -p cmake \
         src/core/include/lumen/core src/core/src \
         apps/lumen-info \
         tests/unit \
         docs/adr \
         .github/workflows
````
 
The layout rule, used by every module we'll ever add:
 
````
src/<module>/include/lumen/<module>/*.hpp   ← public headers:  #include <lumen/core/log.hpp>
src/<module>/src/*.cpp                      ← implementation (private)
apps/<app>/                                 ← executables
tests/unit/                                 ← tests
````
 
> 🧙 **Why `include/lumen/core/` and not just `include/`?** So every include is namespaced: `<lumen/core/log.hpp>` can never collide with some other library's `log.hpp`. Real libraries (Boost, Qt, libinput's internals) all do this.
 
---
 
## Step 2 · The build system
 
### Top-level `CMakeLists.txt`
 
📄 **`CMakeLists.txt`**
 
````cmake
cmake_minimum_required(VERSION 3.22)
project(lumen
    VERSION 0.1.0
    DESCRIPTION "A from-scratch Linux display & input engine and Wayland compositor"
    LANGUAGES CXX)
 
# ---------------------------------------------------------------------------
# Global settings
# ---------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)            # -std=c++20, not -std=gnu++20
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)    # compile_commands.json for clangd / clang-tidy
 
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
endif()
 
option(LUMEN_WERROR   "Treat compiler warnings as errors"            ON)
option(LUMEN_SANITIZE "Build with AddressSanitizer + UBSan"          OFF)
option(LUMEN_TESTS    "Build the unit tests (needs Catch2 v3)"       ON)
 
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(LumenHelpers)
 
find_package(PkgConfig REQUIRED)
 
# ---------------------------------------------------------------------------
# Modules (libraries) — one add_subdirectory per module, added chapter by chapter
# ---------------------------------------------------------------------------
add_subdirectory(src/core)        # m01
 
# ---------------------------------------------------------------------------
# Applications
# ---------------------------------------------------------------------------
add_subdirectory(apps/lumen-info) # m01
 
# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------
if(LUMEN_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()
````
 
What the less obvious lines do:
 
| Line | Why |
|---|---|
| `CMAKE_CXX_EXTENSIONS OFF` | Use pure `-std=c++20`, not GNU extensions, for portable code |
| `CMAKE_EXPORT_COMPILE_COMMANDS ON` | Writes `build/compile_commands.json`, which lets **clangd** (VS Code IntelliSense) and **clang-tidy** understand your code |
| `option(LUMEN_SANITIZE …)` | Switch on ASan+UBSan with `-DLUMEN_SANITIZE=ON` |
| `include(LumenHelpers)` | Our own CMake functions (below) so every module follows the same rules |
| one `add_subdirectory` per module | Each chapter adds a line here. Reading this file tells you what Lumen can do |
 
### `cmake/LumenHelpers.cmake`
 
📄 **`cmake/LumenHelpers.cmake`**
 
````cmake
# LumenHelpers.cmake — small helpers shared by every module and app.
 
# Apply Lumen's warning and sanitizer policy to a target.
function(lumen_target_defaults target)
  target_compile_options(${target} PRIVATE
    -Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wsign-conversion
    -Wnon-virtual-dtor -Wold-style-cast -Woverloaded-virtual)
  if(LUMEN_WERROR)
    target_compile_options(${target} PRIVATE -Werror)
  endif()
  if(LUMEN_SANITIZE)
    target_compile_options(${target} PRIVATE -fsanitize=address,undefined -fno-omit-frame-pointer)
    target_link_options(${target} PRIVATE -fsanitize=address,undefined)
  endif()
endfunction()
 
# lumen_add_module(<name> SOURCES ... [DEPS ...])
#   Creates static library lumen_<name> with alias lumen::<name>.
#   Public headers live in src/<name>/include/lumen/<name>/ and are included as
#   #include <lumen/<name>/header.hpp>
function(lumen_add_module name)
  cmake_parse_arguments(ARG "" "" "SOURCES;DEPS" ${ARGN})
  add_library(lumen_${name} STATIC ${ARG_SOURCES})
  add_library(lumen::${name} ALIAS lumen_${name})
  target_include_directories(lumen_${name} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
  if(ARG_DEPS)
    target_link_libraries(lumen_${name} PUBLIC ${ARG_DEPS})
  endif()
  lumen_target_defaults(lumen_${name})
endfunction()
 
# lumen_add_app(<name> SOURCES ... [DEPS ...])
function(lumen_add_app name)
  cmake_parse_arguments(ARG "" "" "SOURCES;DEPS" ${ARGN})
  add_executable(${name} ${ARG_SOURCES})
  target_link_libraries(${name} PRIVATE ${ARG_DEPS})
  lumen_target_defaults(${name})
endfunction()
````
 
`lumen_add_module(core SOURCES …)` creates a static library `lumen_core` and an **alias** `lumen::core`. Always link to the `lumen::` name: if you typo it, CMake fails loudly instead of silently treating it as a system library called `-llumen_cor`.
 
The warning flags are strict on purpose. Graphics code is full of integer conversions (`uint32_t` widths, `int` strides, `size_t` sizes), and these flags catch the classic bugs:
 
| Flag | Catches |
|---|---|
| `-Wconversion` / `-Wsign-conversion` | `int stride = width * 4;` silently overflowing or going negative |
| `-Wshadow` | an inner `width` hiding the outer one |
| `-Wold-style-cast` | `(int)x` hiding dangerous conversions; we write `static_cast` |
| `-Wnon-virtual-dtor` | deleting a backend through a base pointer without a virtual destructor (m07!) |
| `-Werror` | a warning you ignore today is a bug you'll debug at 2 a.m. |
 
---
 
## Step 3 · Theory: how C++ talks to the kernel
 
Before writing `lumen::core`, let's understand what it wraps.
 
### Everything is a file descriptor
 
````mermaid
flowchart LR
    APP["Lumen"] -- "open('/dev/dri/card0')" --> FD["int fd = 3"]
    FD -- "read(fd, …)" --> E["receive a stream:<br/>input events, DRM events"]
    FD -- "ioctl(fd, REQUEST, &struct)" --> I["send a command,<br/>get an answer"]
    FD -- "mmap(fd, …)" --> M["map device/buffer memory<br/>(pixels!) into your address space"]
    FD -- "epoll(fd)" --> P["sleep until something<br/>happens (m11)"]
    FD -- "close(fd)" --> C["release"]
````
 
| Syscall | Purpose | Lumen uses it for |
|---|---|---|
| `open` | get a handle to a device | `/dev/dri/card0`, `/dev/input/event3`, `/dev/fb0` |
| `read` | receive a stream | 24-byte `input_event`s; DRM vblank/flip events |
| `ioctl` | "do this specific thing" | *everything* in DRM/KMS, input capabilities |
| `mmap` | make memory appear as a pointer | drawing pixels into framebuffers (m12–m13) |
| `epoll` | wait on many fds at once | the event loop (m11) |
 
### `ioctl` demystified
 
`ioctl(fd, request, &arg)` takes a **request number** and a **pointer to a struct**. The request number encodes its own meaning:
 
````c
// From <drm/drm.h>:
#define DRM_IOCTL_BASE      'd'
#define DRM_IOWR(nr, type)  _IOWR(DRM_IOCTL_BASE, nr, type)
#define DRM_IOCTL_VERSION   DRM_IOWR(0x00, struct drm_version)
````
 
````
 31  30 29          16 15          8 7            0
┌─────┬──────────────┬─────────────┬──────────────┐
│ dir │ sizeof(arg)  │ type: 'd'   │ number: 0x00 │
└─────┴──────────────┴─────────────┴──────────────┘
 dir: none / write / read / read+write (kernel's point of view)
````
 
So `DRM_IOCTL_VERSION` means "DRM subsystem, command 0, I'll pass a `struct drm_version` that you read *and* write back". Libraries like **libdrm** are mostly thin wrappers that fill these structs: `drmGetVersion(fd)` is just `ioctl(fd, DRM_IOCTL_VERSION, &v)` called twice (once to learn the string lengths, once to fill them in).
 
The input side is the same: `EVIOCGNAME(len)` = `_IOC(_IOC_READ, 'E', 0x06, len)`, i.e. "evdev command 6, read back `len` bytes of name".
 
### The three C error styles
 
The libraries under Lumen don't even agree on how to report errors:
 
| Style | Who | Example | Lumen helper |
|---|---|---|---|
| return `-1`, set `errno` | syscalls, much of libdrm | `open`, `ioctl`, `drmGetCap` | `check_syscall(ret, "what")` |
| return `-errno` directly | newer libdrm & libinput functions | `drmModeAtomicCommit` → `-EINVAL` | `check_neg_errno(ret, "what")` |
| return `NULL`, set `errno` | allocating getters | `drmModeGetResources` | `check_ptr(p, "what")` |
 
Mixing these up is a classic bug: reading `errno` after a function that *returned* `-EINVAL` gives you a stale, unrelated error. Our helpers make the convention explicit at every call site.
 
The errnos you'll meet constantly:
 
| errno | Usual meaning in our world |
|---|---|
| `EACCES` | wrong group/ACL, or **not DRM master** |
| `ENOENT` | device file doesn't exist |
| `EINVAL` | bad struct, or a combination the driver can't do |
| `EBUSY` | resource in use (e.g. a page flip already pending) |
| `EAGAIN` | nothing to read right now (non-blocking fd) |
| `EINTR` | a signal interrupted the call; retry |
 
---
 
## Step 4 · `lumen::core`
 
### `UniqueFd`: RAII for file descriptors
 
📄 **`src/core/include/lumen/core/unique_fd.hpp`**
 
````cpp
// lumen/core/unique_fd.hpp — RAII owner for a POSIX file descriptor.
//
// Every device we talk to in Lumen (/dev/dri/card*, /dev/input/event*, /dev/fb0,
// sockets, timerfds, epoll…) is an int file descriptor. UniqueFd guarantees each
// one is closed exactly once, even when exceptions fly.
#pragma once
 
#include <fcntl.h>
#include <unistd.h>
 
#include <string>
#include <utility>
 
namespace lumen {
 
class UniqueFd {
public:
    UniqueFd() noexcept = default;
    explicit UniqueFd(int fd) noexcept : fd_(fd) {}
    ~UniqueFd() { reset(); }
 
    // One owner only: copying would mean two close() calls on the same number.
    UniqueFd(const UniqueFd&) = delete;
    UniqueFd& operator=(const UniqueFd&) = delete;
 
    // Ownership can move (return from functions, store in containers).
    UniqueFd(UniqueFd&& other) noexcept : fd_(std::exchange(other.fd_, -1)) {}
    UniqueFd& operator=(UniqueFd&& other) noexcept {
        if (this != &other) reset(std::exchange(other.fd_, -1));
        return *this;
    }
 
    [[nodiscard]] int get() const noexcept { return fd_; }
    [[nodiscard]] bool valid() const noexcept { return fd_ >= 0; }
    explicit operator bool() const noexcept { return valid(); }
 
    // Give up ownership without closing (e.g. to hand the fd to a C library).
    [[nodiscard]] int release() noexcept { return std::exchange(fd_, -1); }
 
    // Close the current fd (if any) and optionally adopt a new one.
    void reset(int fd = -1) noexcept {
        if (fd_ >= 0) ::close(fd_);
        fd_ = fd;
    }
 
private:
    int fd_ = -1;
};
 
// open() that throws std::system_error on failure. O_CLOEXEC is always added so
// child processes never inherit our device fds.
[[nodiscard]] UniqueFd open_fd(const std::string& path, int flags);
 
}  // namespace lumen
````
 
📄 **`src/core/src/unique_fd.cpp`**
 
````cpp
// unique_fd.cpp — out-of-line helpers for lumen/core/unique_fd.hpp.
#include <lumen/core/error.hpp>
#include <lumen/core/unique_fd.hpp>
 
namespace lumen {
 
UniqueFd open_fd(const std::string& path, int flags) {
    const int fd = retry_eintr([&] { return ::open(path.c_str(), flags | O_CLOEXEC); });
    if (fd < 0) throw_errno("open " + path);
    return UniqueFd{fd};
}
 
}  // namespace lumen
````
 
Every design choice prevents a real bug:
 
````mermaid
stateDiagram-v2
    [*] --> Empty: default-constructed
    Empty --> Owning: constructed with fd, or reset(fd)
    Owning --> Owning: reset(other) closes old, adopts new
    Owning --> Empty: release() hands fd out, no close
    Owning --> Empty: moved-from (other object owns it now)
    Owning --> [*]: destructor closes()
    Empty --> [*]: destructor does nothing
````
 
- **Copy deleted:** two owners means two `close()` calls. The second may close an **unrelated** file that reused the same number. That's a heisenbug from hell.
- **Move allowed:** return it from functions, keep it in `std::vector`.
- **`O_CLOEXEC` always:** if Lumen launches an app (m22, m27), the child must not inherit our GPU or keyboard fds.
- **`retry_eintr`:** `open` can be interrupted by a signal; we just retry.
- **`[[nodiscard]]`:** `open_fd(path, flags);` without using the result would open and *immediately close* the file. The compiler now warns.
### `CPtr`: RAII for C library structs
 
📄 **`src/core/include/lumen/core/c_ptr.hpp`**
 
````cpp
// lumen/core/c_ptr.hpp — unique_ptr for structs allocated by C libraries.
//
// C libraries hand out pointers that must be released with a *specific* function:
//   drmModeGetResources()  →  drmModeFreeResources()
//   libinput_path_create_context() → libinput_unref()
//
//   using ResPtr = lumen::CPtr<drmModeRes, drmModeFreeResources>;
//   ResPtr res{drmModeGetResources(fd)};      // freed automatically
//
// The deleter is a stateless type, so CPtr is exactly the size of a raw pointer
// (unlike unique_ptr<T, decltype(&fn)>, which also stores a function pointer).
#pragma once
 
#include <memory>
 
namespace lumen {
 
template <auto FreeFn>
struct CDeleter {
    template <typename T>
    void operator()(T* p) const noexcept {
        if (p) FreeFn(p);
    }
};
 
template <typename T, auto FreeFn>
using CPtr = std::unique_ptr<T, CDeleter<FreeFn>>;
 
}  // namespace lumen
````
 
`template <auto FreeFn>` takes the **function itself** as a compile-time parameter. Compare:
 
````cpp
std::unique_ptr<drmModeRes, decltype(&drmModeFreeResources)> a{p, &drmModeFreeResources}; // 16 bytes
lumen::CPtr<drmModeRes, drmModeFreeResources>                b{p};                        //  8 bytes
````
 
Version `b` is shorter to write, can't be constructed with the wrong deleter, and is exactly as small as a raw pointer.
 
### `error.hpp`: one way to fail
 
📄 **`src/core/include/lumen/core/error.hpp`**
 
````cpp
// lumen/core/error.hpp — turning C error conventions into C++ exceptions.
//
// The C APIs under Lumen report errors in three different styles:
//   1. return -1 and set errno                 (open, ioctl, read, most syscalls)
//   2. return a negative errno directly         (many libdrm/libinput functions: -EINVAL)
//   3. return nullptr and set errno             (drmModeGetResources, …)
// These helpers normalise all three into std::system_error.
#pragma once
 
#include <cerrno>
#include <string>
#include <string_view>
#include <system_error>
 
namespace lumen {
 
// Throw std::system_error for `err` (defaults to the current errno).
[[noreturn]] inline void throw_errno(std::string_view what, int err = errno) {
    throw std::system_error(err, std::generic_category(), std::string(what));
}
 
// Style 1: `-1 + errno`. Returns the value unchanged on success so it can wrap calls:
//   int fd = check_syscall(::open(path, O_RDWR), "open");
template <typename T>
T check_syscall(T ret, std::string_view what) {
    if (ret < 0) throw_errno(what);
    return ret;
}
 
// Style 2: `-errno` returned directly (errno itself may be stale!).
template <typename T>
T check_neg_errno(T ret, std::string_view what) {
    if (ret < 0) throw_errno(what, static_cast<int>(-ret));
    return ret;
}
 
// Style 3: nullptr + errno.
template <typename T>
T* check_ptr(T* ptr, std::string_view what) {
    if (ptr == nullptr) throw_errno(what);
    return ptr;
}
 
// Retry a syscall while it fails with EINTR (interrupted by a signal).
template <typename F>
auto retry_eintr(F&& f) -> decltype(f()) {
    decltype(f()) ret;
    do {
        ret = f();
    } while (ret < 0 && errno == EINTR);
    return ret;
}
 
}  // namespace lumen
````
 
`std::system_error` carries an `std::error_code`, so callers can react to *specific* errors portably:
 
````cpp
catch (const std::system_error& e) {
    if (e.code() == std::errc::permission_denied) { /* explain groups/master */ }
}
````
 
### The logger
 
📄 **`src/core/include/lumen/core/log.hpp`**
 
````cpp
// lumen/core/log.hpp — tiny leveled + categorised logger.
//
//   lumen::log::info("found {} connectors", n);
//   lumen::log::debug(lumen::log::Cat::Drm, "ioctl {} -> {}", name, ret);
//
// Debug output is off by default and enabled per category with an env var:
//   LUMEN_DEBUG=drm,input ./lumen
//   LUMEN_DEBUG=all       ./lumen
#pragma once
 
#include <cstdint>
#include <format>
#include <string_view>
#include <utility>
 
namespace lumen::log {
 
enum class Level : uint8_t { Debug, Info, Warn, Error };
 
// Categories are bit flags so several can be enabled at once.
enum class Cat : uint32_t {
    Core = 1u << 0,
    Drm = 1u << 1,
    Input = 1u << 2,
    Session = 1u << 3,
    Render = 1u << 4,
    Server = 1u << 5,
    All = 0xFFFF'FFFFu,
};
 
// Parse "drm,input" / "all" / "" into a bitmask. Unknown names are ignored.
uint32_t parse_categories(std::string_view spec);
 
// Read LUMEN_DEBUG once (call at startup; safe to call again to re-read).
void init_from_env();
 
// Override programmatically (tests, command-line flags).
void set_enabled_categories(uint32_t mask);
[[nodiscard]] bool enabled(Cat cat);
 
// Low-level sink: writes "[level] [category] message" to stderr.
void write(Level level, std::string_view category, std::string_view message);
 
template <typename... Args>
void info(std::format_string<Args...> fmt, Args&&... args) {
    write(Level::Info, {}, std::format(fmt, std::forward<Args>(args)...));
}
template <typename... Args>
void warn(std::format_string<Args...> fmt, Args&&... args) {
    write(Level::Warn, {}, std::format(fmt, std::forward<Args>(args)...));
}
template <typename... Args>
void error(std::format_string<Args...> fmt, Args&&... args) {
    write(Level::Error, {}, std::format(fmt, std::forward<Args>(args)...));
}
 
std::string_view category_name(Cat cat);
 
template <typename... Args>
void debug(Cat cat, std::format_string<Args...> fmt, Args&&... args) {
    if (!enabled(cat)) return;  // cheap check first: no formatting cost when off
    write(Level::Debug, category_name(cat), std::format(fmt, std::forward<Args>(args)...));
}
 
}  // namespace lumen::log
````
 
📄 **`src/core/src/log.cpp`**
 
````cpp
// log.cpp — implementation of lumen/core/log.hpp.
#include <lumen/core/log.hpp>
 
#include <unistd.h>
 
#include <array>
#include <atomic>
#include <cstdio>
#include <cstdlib>
#include <string>
 
namespace lumen::log {
namespace {
 
struct CatName {
    Cat cat;
    std::string_view name;
};
 
constexpr std::array kCategories{
    CatName{Cat::Core, "core"},     CatName{Cat::Drm, "drm"},
    CatName{Cat::Input, "input"},   CatName{Cat::Session, "session"},
    CatName{Cat::Render, "render"}, CatName{Cat::Server, "server"},
};
 
std::atomic<uint32_t> g_enabled{0};
 
std::string_view trim(std::string_view s) {
    while (!s.empty() && (s.front() == ' ' || s.front() == '\t')) s.remove_prefix(1);
    while (!s.empty() && (s.back() == ' ' || s.back() == '\t')) s.remove_suffix(1);
    return s;
}
 
}  // namespace
 
uint32_t parse_categories(std::string_view spec) {
    uint32_t mask = 0;
    while (!spec.empty()) {
        const auto comma = spec.find(',');
        const auto token = trim(spec.substr(0, comma));
        if (token == "all") {
            mask |= static_cast<uint32_t>(Cat::All);
        } else {
            for (const auto& c : kCategories)
                if (token == c.name) mask |= static_cast<uint32_t>(c.cat);
        }
        if (comma == std::string_view::npos) break;
        spec.remove_prefix(comma + 1);
    }
    return mask;
}
 
void init_from_env() {
    const char* env = std::getenv("LUMEN_DEBUG");
    set_enabled_categories(env ? parse_categories(env) : 0);
}
 
void set_enabled_categories(uint32_t mask) {
    g_enabled.store(mask, std::memory_order_relaxed);
}
 
bool enabled(Cat cat) {
    return (g_enabled.load(std::memory_order_relaxed) & static_cast<uint32_t>(cat)) != 0;
}
 
std::string_view category_name(Cat cat) {
    for (const auto& c : kCategories)
        if (c.cat == cat) return c.name;
    return "?";
}
 
void write(Level level, std::string_view category, std::string_view message) {
    static const bool color = ::isatty(STDERR_FILENO) != 0;
    const char* tag = "";
    const char* col = "";
    switch (level) {
        case Level::Debug:
            tag = "debug";
            col = "\033[2m";
            break;  // dim
        case Level::Info:
            tag = "info";
            col = "\033[36m";
            break;  // cyan
        case Level::Warn:
            tag = "warn";
            col = "\033[33m";
            break;  // yellow
        case Level::Error:
            tag = "error";
            col = "\033[31m";
            break;  // red
    }
    std::string line;
    if (color) line += col;
    line += '[';
    line += tag;
    line += ']';
    if (!category.empty()) {
        line += " [";
        line += category;
        line += ']';
    }
    line += ' ';
    line += message;
    if (color) line += "\033[0m";
    line += '\n';
    std::fputs(line.c_str(), stderr);
}
 
}  // namespace lumen::log
````
 
Design notes:
- **`std::format_string<Args...>`** checks format strings **at compile time**: `log::info("{} {}", x)` with a missing argument is a compile error, not a crash.
- **`debug()` checks `enabled()` before formatting**, so turned-off debug logs cost one atomic load. That matters when we log every vblank (60×/s) or every mouse event (1000×/s).
- **Categories are bit flags**, so `LUMEN_DEBUG=drm,input` enables two at once with one `&` test.
- **`std::atomic`**: from m09 worker threads log too, so the flags must be safe to read concurrently.
- One `fputs` per line keeps lines from different threads from interleaving mid-line.
`src/core/CMakeLists.txt`:
 
📄 **`src/core/CMakeLists.txt`**
 
````cmake
lumen_add_module(core
  SOURCES
    src/log.cpp
    src/unique_fd.cpp
)
````
 
---
 
## Step 5 · `lumen-info`: skeleton
 
`lumen-info` is Lumen's X-ray machine. Every chapter adds a sub-command, and you'll use it to check your understanding against reality.
 
📄 **`apps/lumen-info/commands.hpp`**
 
````cpp
// commands.hpp — sub-commands of lumen-info. Each returns a process exit code.
#pragma once
 
#include <span>
#include <string_view>
 
namespace lumen::info {
 
using Args = std::span<const std::string_view>;
 
int cmd_drm(Args args);    // lumen-info drm [NODE]
int cmd_input(Args args);  // lumen-info input [DEVICE]
 
}  // namespace lumen::info
````
 
📄 **`apps/lumen-info/main.cpp`**
 
````cpp
// lumen-info — inspect the graphics & input devices Lumen will drive.
//
//   lumen-info drm               list every DRM node: driver, caps, connectors
//   lumen-info drm /dev/dri/card0
//   lumen-info input             list every evdev device with its type
//   lumen-info input /dev/input/event3   dump raw events (Ctrl+C to stop)
//
// It grows every chapter: EDID decoding (ch08), full KMS tree (ch13), …
#include "commands.hpp"
 
#include <lumen/core/log.hpp>
 
#include <cstdio>
#include <exception>
#include <string_view>
#include <vector>
 
namespace {
 
constexpr std::string_view kVersion = "0.1.0 (m01)";
 
void usage() {
    std::puts(
        "usage: lumen-info <command> [args]\n"
        "\n"
        "commands:\n"
        "  drm   [NODE]     list DRM devices (or one node): driver, caps, connectors\n"
        "  input [DEVICE]   list input devices, or dump raw events from one\n"
        "  version          print version\n"
        "\n"
        "environment:\n"
        "  LUMEN_DEBUG=drm,input,all   enable debug logging per category");
}
 
}  // namespace
 
int main(int argc, char** argv) {
    lumen::log::init_from_env();
 
    // Convert argv to string_views once; nicer to work with than char**.
    std::vector<std::string_view> args(argv + 1, argv + argc);
    if (args.empty() || args[0] == "help" || args[0] == "--help" || args[0] == "-h") {
        usage();
        return args.empty() ? 1 : 0;
    }
 
    const std::string_view cmd = args[0];
    const lumen::info::Args rest{args.data() + 1, args.size() - 1};
 
    try {
        if (cmd == "drm") return lumen::info::cmd_drm(rest);
        if (cmd == "input") return lumen::info::cmd_input(rest);
        if (cmd == "version" || cmd == "--version") {
            std::printf("lumen-info %.*s\n", static_cast<int>(kVersion.size()), kVersion.data());
            return 0;
        }
        lumen::log::error("unknown command '{}'", cmd);
        usage();
        return 1;
    } catch (const std::exception& e) {
        // Anything a command didn't handle ends up here with a readable message,
        // e.g. "open /dev/dri/card0: Permission denied".
        lumen::log::error("{}", e.what());
        return 1;
    }
}
````
 
📄 **`apps/lumen-info/CMakeLists.txt`**
 
````cmake
pkg_check_modules(LIBDRM REQUIRED IMPORTED_TARGET libdrm)
 
lumen_add_app(lumen-info
  SOURCES
    main.cpp
    cmd_drm.cpp
    cmd_input.cpp
  DEPS
    lumen::core
    PkgConfig::LIBDRM
)
````
 
The top-level `try/catch` in `main` is the **error boundary**: commands throw `std::system_error` with context ("open /dev/dri/card0: Permission denied"), and `main` turns that into one clean error line and exit code 1. No error code goes unchecked, and none needs to be passed back by hand.
 
---
 
## Step 6 · `lumen-info drm`
 
### The plan
 
````mermaid
flowchart TD
    A["scan /dev/dri for card* / renderD*"] --> B["open_fd(node, O_RDWR)"]
    B -->|"system_error EACCES"| X["print hint: groups / getfacl"]
    B --> C["drmGetVersion → DRM_IOCTL_VERSION"]
    C -->|"fails"| NX["not a DRM device"]
    C --> D["drmGetCap × 6 → DRM_IOCTL_GET_CAP"]
    D --> E{"primary node?"}
    E -->|"render node"| F["done: render nodes can't drive displays"]
    E -->|"card node"| G["drmIsMaster"]
    G --> H["drmModeGetResources → MODE_GETRESOURCES"]
    H --> I["for each connector: drmModeGetConnector<br/>type, status, size, preferred mode"]
````
 
### The code
 
📄 **`apps/lumen-info/cmd_drm.cpp`**
 
````cpp
// cmd_drm.cpp — `lumen-info drm`: say hello to every DRM device.
//
// For each /dev/dri/card* and /dev/dri/renderD* node:
//   * ask the kernel driver its name and version     (DRM_IOCTL_VERSION)
//   * query a few capabilities                       (DRM_IOCTL_GET_CAP)
//   * for primary (card) nodes: are we DRM master? which connectors exist,
//     and is a monitor plugged into them?            (DRM_IOCTL_MODE_GETRESOURCES, …)
#include "commands.hpp"
 
#include <lumen/core/c_ptr.hpp>
#include <lumen/core/log.hpp>
#include <lumen/core/unique_fd.hpp>
 
#include <xf86drm.h>
#include <xf86drmMode.h>
 
#include <algorithm>
#include <cstdint>
#include <cstdio>
#include <filesystem>
#include <string>
#include <system_error>
#include <vector>
 
namespace lumen::info {
namespace {
 
namespace fs = std::filesystem;
 
// RAII aliases for the libdrm structs we use (see lumen/core/c_ptr.hpp).
using VersionPtr = CPtr<drmVersion, drmFreeVersion>;
using ResourcesPtr = CPtr<drmModeRes, drmModeFreeResources>;
using ConnectorPtr = CPtr<drmModeConnector, drmModeFreeConnector>;
 
struct CapInfo {
    uint64_t id;
    const char* name;
    const char* meaning;
};
 
constexpr CapInfo kCaps[] = {
    {DRM_CAP_DUMB_BUFFER, "DUMB_BUFFER", "CPU-drawable scanout buffers (ch13)"},
    {DRM_CAP_PRIME, "PRIME", "dma-buf sharing: 1=import 2=export 3=both (ch16)"},
    {DRM_CAP_TIMESTAMP_MONOTONIC, "TIMESTAMP_MONOTONIC", "vblank timestamps use CLOCK_MONOTONIC"},
    {DRM_CAP_ASYNC_PAGE_FLIP, "ASYNC_PAGE_FLIP", "flip without waiting for vblank (ch14)"},
    {DRM_CAP_CURSOR_WIDTH, "CURSOR_WIDTH", "hardware cursor size in pixels (ch14)"},
    {DRM_CAP_ADDFB2_MODIFIERS, "ADDFB2_MODIFIERS", "framebuffers with tiling modifiers (ch10)"},
};
 
const char* connection_str(drmModeConnection c) {
    switch (c) {
        case DRM_MODE_CONNECTED:
            return "connected";
        case DRM_MODE_DISCONNECTED:
            return "disconnected";
        default:
            return "unknown";
    }
}
 
const char* node_type_str(int type) {
    switch (type) {
        case DRM_NODE_PRIMARY:
            return "primary (display + render)";
        case DRM_NODE_RENDER:
            return "render (render only)";
        default:
            return "other";
    }
}
 
void print_connectors(int fd) {
    ResourcesPtr res{drmModeGetResources(fd)};
    if (!res) {
        std::printf("  KMS: none (this GPU cannot drive displays)\n");
        return;
    }
    std::printf("  KMS: %d connectors, %d encoders, %d CRTCs\n", res->count_connectors,
                res->count_encoders, res->count_crtcs);
 
    for (int i = 0; i < res->count_connectors; ++i) {
        ConnectorPtr conn{drmModeGetConnector(fd, res->connectors[i])};
        if (!conn) continue;
        const char* type = drmModeGetConnectorTypeName(conn->connector_type);
        std::printf("    ├─ connector %u  %s-%u  %s", conn->connector_id, type ? type : "?",
                    conn->connector_type_id, connection_str(conn->connection));
        if (conn->connection == DRM_MODE_CONNECTED) {
            std::printf("  %umm×%umm, %d modes", conn->mmWidth, conn->mmHeight, conn->count_modes);
            // Find the mode the monitor marks as preferred (usually its native resolution).
            for (int m = 0; m < conn->count_modes; ++m) {
                const drmModeModeInfo& mode = conn->modes[m];
                if (mode.type & DRM_MODE_TYPE_PREFERRED) {
                    std::printf(", preferred %s@%u", mode.name, mode.vrefresh);
                    break;
                }
            }
        }
        std::printf("\n");
    }
}
 
void inspect_node(const fs::path& path) {
    std::printf("%s\n", path.c_str());
 
    UniqueFd fd;
    try {
        fd = open_fd(path.string(), O_RDWR);
    } catch (const std::system_error& e) {
        std::printf("  cannot open: %s\n", e.what());
        if (e.code() == std::errc::permission_denied)
            std::printf("  hint: check `groups` (video/render) or `getfacl %s`\n", path.c_str());
        std::printf("\n");
        return;
    }
    log::debug(log::Cat::Drm, "opened {} as fd {}", path.string(), fd.get());
 
    VersionPtr ver{drmGetVersion(fd.get())};
    if (!ver) {
        // DRM_IOCTL_VERSION failed: whatever this file is, it is not a DRM device.
        std::printf("  not a DRM device (DRM_IOCTL_VERSION failed)\n\n");
        return;
    }
    std::printf("  driver : %s %d.%d.%d  (%s)\n", ver->name, ver->version_major, ver->version_minor,
                ver->version_patchlevel, ver->desc);
    const int type = drmGetNodeTypeFromFd(fd.get());
    std::printf("  node   : %s\n", node_type_str(type));
 
    std::printf("  caps   :\n");
    for (const auto& cap : kCaps) {
        uint64_t value = 0;
        if (drmGetCap(fd.get(), cap.id, &value) == 0)
            std::printf("    %-20s = %-4llu %s\n", cap.name, static_cast<unsigned long long>(value),
                        cap.meaning);
        else
            std::printf("    %-20s = n/a  %s\n", cap.name, cap.meaning);
    }
 
    if (type == DRM_NODE_PRIMARY) {
        const bool master = drmIsMaster(fd.get()) != 0;
        std::printf("  master : %s\n",
                    master ? "YES (we may change what is on screen)"
                           : "no (another process, e.g. gnome-shell, owns the display)");
        print_connectors(fd.get());
    }
    std::printf("\n");
}
 
std::vector<fs::path> find_nodes() {
    std::vector<fs::path> nodes;
    const fs::path dri{"/dev/dri"};
    std::error_code ec;
    for (const auto& entry : fs::directory_iterator(dri, ec)) {
        const std::string name = entry.path().filename().string();
        if (name.starts_with("card") || name.starts_with("renderD")) nodes.push_back(entry.path());
    }
    std::sort(nodes.begin(), nodes.end());
    return nodes;
}
 
}  // namespace
 
int cmd_drm(Args args) {
    if (!args.empty()) {
        inspect_node(fs::path{args[0]});
        return 0;
    }
    const auto nodes = find_nodes();
    if (nodes.empty()) {
        log::error("no DRM nodes in /dev/dri (no GPU driver loaded? try: sudo modprobe vkms)");
        return 1;
    }
    for (const auto& n : nodes) inspect_node(n);
    return 0;
}
 
}  // namespace lumen::info
````
 
Things to notice:
- **`VersionPtr ver{drmGetVersion(fd.get())}`**: allocated by libdrm, freed by `CPtr` on every path, including early `return`s.
- **The preferred mode** is the one with `DRM_MODE_TYPE_PREFERRED`, normally the panel's native resolution. Don't just take `modes[0]` and hope.
- **`mmWidth/mmHeight`**: the physical size, read from EDID. A VM reports 0 because there's no physical panel. On real monitors, physical size ÷ pixels gives you DPI (Ch 02).
### Build and run
 
````bash
cd ~/lumen
cmake -S . -B build -G Ninja
cmake --build build
./build/apps/lumen-info/lumen-info drm
````
 
Expected in the VM (versions differ):
 
````
/dev/dri/card0
  driver : virtio_gpu 0.1.0  (virtio GPU)
  node   : primary (display + render)
  caps   :
    DUMB_BUFFER          = 1    CPU-drawable scanout buffers (ch13)
    PRIME                = 3    dma-buf sharing: 1=import 2=export 3=both (ch16)
    TIMESTAMP_MONOTONIC  = 1    vblank timestamps use CLOCK_MONOTONIC
    ASYNC_PAGE_FLIP      = 0    flip without waiting for vblank (ch14)
    CURSOR_WIDTH         = 64   hardware cursor size in pixels (ch14)
    ADDFB2_MODIFIERS     = 0    framebuffers with tiling modifiers (ch10)
  master : YES (we may change what is on screen)
  KMS: 1 connectors, 1 encoders, 1 CRTCs
    ├─ connector 36  Virtual-1  connected  0mm×0mm, 25 modes, preferred 1280x800@60
 
/dev/dri/renderD128
  driver : virtio_gpu 0.1.0  (virtio GPU)
  node   : render (render only)
  caps   : …
````
 
> 🔮 **Predict, then run on your HOST desktop** (copy the repo over or clone it there; `libdrm-dev` + `catch2` needed). What will `master` say there, and why?
 
<details><summary>Reveal</summary>
`no`, because gnome-shell (Mutter) is already master of your card, and there can be only one. In the VM nothing else had `card0` open, so **the first process to open a primary node automatically becomes master**. That's you. On the host you'll also see your real driver: `i915`/`xe`, `amdgpu`, `nouveau` or `nvidia-drm`. If you load vkms in the VM, `lumen-info drm` shows two cards, one per driver.
</details>
### 👀 X-ray it with `strace`
 
````bash
strace -e trace=openat,ioctl ./build/apps/lumen-info/lumen-info drm 2>&1 | grep -A4 'card0"'
````
 
````
openat(AT_FDCWD, "/dev/dri/card0", O_RDWR|O_CLOEXEC) = 3
ioctl(3, DRM_IOCTL_VERSION, 0x7ffd…) = 0
ioctl(3, DRM_IOCTL_VERSION, 0x7ffd…) = 0
ioctl(3, DRM_IOCTL_GET_CAP, {capability=DRM_CAP_DUMB_BUFFER, value=1}) = 0
````
 
strace decodes DRM ioctls by name. You can see `DRM_IOCTL_VERSION` twice, exactly as explained in Step 3. Try it later on `weston` or `modetest` (`strace -f -e ioctl …`) to see what *they* ask the kernel. This is the single most useful debugging trick in this series.
 
---
 
## Step 7 · `lumen-info input`
 
### Theory: how evdev describes events
 
Every input event is one fixed-size struct, from `<linux/input.h>`:
 
````c
struct input_event {
    struct timeval time;   // when (sec + µs)
    __u16 type;            // category: EV_KEY, EV_REL, EV_ABS, EV_SYN, EV_MSC…
    __u16 code;            // which: KEY_A, BTN_LEFT, REL_X, ABS_MT_POSITION_X…
    __s32 value;           // key: 0 up, 1 down, 2 autorepeat · axis: delta or position
};                         // 24 bytes on 64-bit
````
 
Events arrive in **frames** terminated by `EV_SYN / SYN_REPORT`, meaning "everything since the last SYN happened at one instant":
 
````mermaid
sequenceDiagram
    participant Mouse
    participant evdev as /dev/input/event3
    participant Lumen as lumen-info
    Mouse->>evdev: EV_REL REL_X +3
    Mouse->>evdev: EV_REL REL_Y -1
    Mouse->>evdev: EV_SYN SYN_REPORT 0
    evdev->>Lumen: read() → 3 × 24 bytes
    Note over Lumen: one frame: moved right 3 and up 1<br/>at the same instant. Never apply X<br/>without waiting for its SYN!
````
 
### Theory: asking a device what it is
 
Before reading events, you can ask a device **what it can send**. `EVIOCGBIT(type, len)` fills a bitmap where bit N = "I can send code N":
 
````mermaid
flowchart LR
    Q0["EVIOCGBIT(0)<br/>which TYPES?"] --> T["bit EV_KEY ✔<br/>bit EV_REL ✔<br/>bit EV_ABS ✘"]
    T --> Q1["EVIOCGBIT(EV_KEY)<br/>which KEYS?"]
    T --> Q2["EVIOCGBIT(EV_REL)<br/>which AXES?"]
    Q1 --> K["BTN_LEFT ✔ BTN_RIGHT ✔<br/>KEY_A ✘"]
    Q2 --> R["REL_X ✔ REL_Y ✔ REL_WHEEL ✔"]
    K --> V["⇒ it's a MOUSE"]
    R --> V
````
 
The bitmap is an array of `unsigned long`. Bit `N` lives in word `N / 64` at position `N % 64` (on 64-bit). Our `BitArray<MaxCode>` template wraps that arithmetic. `KEY_MAX` is 0x2ff (767), so the key bitmap is 12 words = 96 bytes.
 
The classification rules (keyboard = has `KEY_A`, `KEY_Z`, `KEY_SPACE`; touchscreen = absolute axes + `INPUT_PROP_DIRECT`, etc.) are simplified versions of what **udev's `input_id`** builtin does to tag devices `ID_INPUT_KEYBOARD=1`, which libinput then trusts. Check with `udevadm info /dev/input/event3`.
 
### The code
 
📄 **`apps/lumen-info/cmd_input.cpp`**
 
````cpp
// cmd_input.cpp — `lumen-info input`: list evdev devices, or dump raw events.
//
// No library needed: <linux/input.h>, open(), ioctl() and read().
//   * listing uses EVIOCGNAME / EVIOCGID / EVIOCGBIT / EVIOCGPROP to ask each device
//     what it is and which event types/codes it can produce
//   * dumping read()s struct input_event records (24 bytes each on 64-bit)
#include "commands.hpp"
 
#include <lumen/core/error.hpp>
#include <lumen/core/log.hpp>
#include <lumen/core/unique_fd.hpp>
 
#include <linux/input.h>
#include <sys/ioctl.h>
 
#include <algorithm>
#include <array>
#include <csignal>
#include <cstdio>
#include <filesystem>
#include <string>
#include <system_error>
#include <vector>
 
namespace lumen::info {
namespace {
 
namespace fs = std::filesystem;
 
// ---------------------------------------------------------------------------
// Capability bitmaps. The kernel describes "which keys/axes exist" as arrays of
// unsigned long where bit N means "code N is supported".
// ---------------------------------------------------------------------------
template <std::size_t MaxCode>
class BitArray {
public:
    static constexpr std::size_t kBitsPerWord = 8 * sizeof(unsigned long);
    [[nodiscard]] bool test(std::size_t bit) const {
        return bit <= MaxCode && ((words_[bit / kBitsPerWord] >> (bit % kBitsPerWord)) & 1UL) != 0;
    }
    unsigned long* data() { return words_.data(); }
    [[nodiscard]] std::size_t size_bytes() const { return sizeof(words_); }
 
private:
    std::array<unsigned long, MaxCode / kBitsPerWord + 1> words_{};
};
 
struct DeviceInfo {
    fs::path path;
    std::string name = "?";
    input_id id{};
    BitArray<EV_MAX> types;
    BitArray<KEY_MAX> keys;
    BitArray<REL_MAX> rels;
    BitArray<ABS_MAX> abs;
    BitArray<INPUT_PROP_MAX> props;
};
 
// EVIOCGBIT(type, len): "which codes of event `type` can you send?" (type 0 = which types)
template <std::size_t N>
void query_bits(int fd, unsigned type, BitArray<N>& bits) {
    if (::ioctl(fd, EVIOCGBIT(type, bits.size_bytes()), bits.data()) < 0)
        log::debug(log::Cat::Input, "EVIOCGBIT({}) failed: errno {}", type, errno);
}
 
DeviceInfo probe(const fs::path& path) {
    UniqueFd fd = open_fd(path.string(), O_RDONLY | O_NONBLOCK);
    DeviceInfo d;
    d.path = path;
    std::array<char, 256> name{};
    if (::ioctl(fd.get(), EVIOCGNAME(name.size() - 1), name.data()) >= 0) d.name = name.data();
    ::ioctl(fd.get(), EVIOCGID, &d.id);
    query_bits(fd.get(), 0, d.types);
    query_bits(fd.get(), EV_KEY, d.keys);
    query_bits(fd.get(), EV_REL, d.rels);
    query_bits(fd.get(), EV_ABS, d.abs);
    ::ioctl(fd.get(), EVIOCGPROP(d.props.size_bytes()), d.props.data());
    return d;
}
 
// A rough guess at what kind of device this is: the same kind of heuristics that
// udev's input_id builtin (and therefore libinput) applies. Ch17/18 do this properly.
std::string classify(const DeviceInfo& d) {
    std::vector<std::string> kinds;
    const bool has_key = d.types.test(EV_KEY);
    if (has_key && d.keys.test(KEY_A) && d.keys.test(KEY_Z) && d.keys.test(KEY_SPACE))
        kinds.emplace_back("keyboard");
    if (d.types.test(EV_REL) && d.rels.test(REL_X) && d.rels.test(REL_Y) && d.keys.test(BTN_LEFT))
        kinds.emplace_back("mouse");
    if (d.types.test(EV_ABS) && d.abs.test(ABS_X) && d.abs.test(ABS_Y)) {
        if (d.props.test(INPUT_PROP_DIRECT))
            kinds.emplace_back("touchscreen");
        else if (d.keys.test(BTN_TOOL_FINGER) && !d.keys.test(BTN_TOOL_PEN))
            kinds.emplace_back("touchpad");
        else if (d.keys.test(BTN_TOOL_PEN) || d.keys.test(BTN_STYLUS))
            kinds.emplace_back("tablet");
        else if (d.keys.test(BTN_LEFT) || d.keys.test(BTN_TOUCH))
            kinds.emplace_back("absolute pointer");
        else if (d.keys.test(BTN_GAMEPAD) || d.keys.test(BTN_JOYSTICK))
            kinds.emplace_back("joystick");
    }
    if (kinds.empty() && has_key) kinds.emplace_back("buttons/keys");
    if (kinds.empty()) kinds.emplace_back("other");
 
    std::string out;
    for (const auto& k : kinds) out += (out.empty() ? "" : "+") + k;
    return out;
}
 
const char* bus_name(uint16_t bus) {
    switch (bus) {
        case BUS_USB:
            return "usb";
        case BUS_I8042:
            return "ps/2";
        case BUS_BLUETOOTH:
            return "bluetooth";
        case BUS_I2C:
            return "i2c";
        case BUS_VIRTUAL:
            return "virtual";
        case BUS_HOST:
            return "host";
        default:
            return "other";
    }
}
 
int event_number(const fs::path& p) {
    // "event12" -> 12, so we can sort event2 before event10.
    const std::string s = p.filename().string();
    return std::stoi(s.substr(5));
}
 
int list_devices() {
    std::vector<fs::path> paths;
    std::error_code ec;
    for (const auto& e : fs::directory_iterator("/dev/input", ec))
        if (e.path().filename().string().starts_with("event")) paths.push_back(e.path());
    std::sort(paths.begin(), paths.end(), [](const fs::path& a, const fs::path& b) {
        return event_number(a) < event_number(b);
    });
 
    if (paths.empty()) {
        log::error("no /dev/input/event* devices found");
        return 1;
    }
 
    std::printf("%-20s %-10s %-9s  %-18s %s\n", "DEVICE", "BUS", "VID:PID", "TYPE", "NAME");
    int denied = 0;
    for (const auto& p : paths) {
        try {
            const DeviceInfo d = probe(p);
            char vidpid[16];
            std::snprintf(vidpid, sizeof vidpid, "%04x:%04x", d.id.vendor, d.id.product);
            std::printf("%-20s %-10s %-9s  %-18s %s\n", p.c_str(), bus_name(d.id.bustype), vidpid,
                        classify(d).c_str(), d.name.c_str());
        } catch (const std::system_error& e) {
            if (e.code() == std::errc::permission_denied) ++denied;
            std::printf("%-20s (%s)\n", p.c_str(), e.code().message().c_str());
        }
    }
    if (denied > 0)
        std::printf(
            "\n%d device(s) not readable: run with sudo, or (lab VM only!) join group 'input'\n",
            denied);
    return 0;
}
 
// ---------------------------------------------------------------------------
// Event dumping
// ---------------------------------------------------------------------------
volatile std::sig_atomic_t g_quit = 0;
void on_sigint(int) {
    g_quit = 1;
}
 
const char* type_name(uint16_t type) {
    switch (type) {
        case EV_SYN:
            return "EV_SYN";
        case EV_KEY:
            return "EV_KEY";
        case EV_REL:
            return "EV_REL";
        case EV_ABS:
            return "EV_ABS";
        case EV_MSC:
            return "EV_MSC";
        case EV_LED:
            return "EV_LED";
        case EV_REP:
            return "EV_REP";
        default:
            return "EV_?";
    }
}
 
// A handful of names so the dump is readable. Ch17 generates the full table
// from <linux/input-event-codes.h> at build time.
std::string code_name(uint16_t type, uint16_t code) {
    struct Name {
        uint16_t code;
        const char* name;
    };
    static constexpr Name kKeys[] = {{KEY_ESC, "KEY_ESC"},
                                     {KEY_A, "KEY_A"},
                                     {KEY_S, "KEY_S"},
                                     {KEY_D, "KEY_D"},
                                     {KEY_W, "KEY_W"},
                                     {KEY_Q, "KEY_Q"},
                                     {KEY_Z, "KEY_Z"},
                                     {KEY_SPACE, "KEY_SPACE"},
                                     {KEY_ENTER, "KEY_ENTER"},
                                     {KEY_LEFTSHIFT, "KEY_LEFTSHIFT"},
                                     {KEY_RIGHTSHIFT, "KEY_RIGHTSHIFT"},
                                     {KEY_LEFTCTRL, "KEY_LEFTCTRL"},
                                     {KEY_LEFTALT, "KEY_LEFTALT"},
                                     {KEY_CAPSLOCK, "KEY_CAPSLOCK"},
                                     {BTN_LEFT, "BTN_LEFT"},
                                     {BTN_RIGHT, "BTN_RIGHT"},
                                     {BTN_MIDDLE, "BTN_MIDDLE"},
                                     {BTN_TOUCH, "BTN_TOUCH"},
                                     {BTN_TOOL_FINGER, "BTN_TOOL_FINGER"}};
    static constexpr Name kRel[] = {
        {REL_X, "REL_X"}, {REL_Y, "REL_Y"}, {REL_WHEEL, "REL_WHEEL"}, {REL_HWHEEL, "REL_HWHEEL"}};
    static constexpr Name kAbs[] = {{ABS_X, "ABS_X"},
                                    {ABS_Y, "ABS_Y"},
                                    {ABS_MT_SLOT, "ABS_MT_SLOT"},
                                    {ABS_MT_POSITION_X, "ABS_MT_POSITION_X"},
                                    {ABS_MT_POSITION_Y, "ABS_MT_POSITION_Y"},
                                    {ABS_MT_TRACKING_ID, "ABS_MT_TRACKING_ID"}};
    static constexpr Name kSyn[] = {{SYN_REPORT, "SYN_REPORT"}, {SYN_DROPPED, "SYN_DROPPED"}};
    static constexpr Name kMsc[] = {{MSC_SCAN, "MSC_SCAN"}};
 
    auto find = [code](const auto& table) -> const char* {
        for (const auto& n : table)
            if (n.code == code) return n.name;
        return nullptr;
    };
    const char* name = nullptr;
    switch (type) {
        case EV_KEY:
            name = find(kKeys);
            break;
        case EV_REL:
            name = find(kRel);
            break;
        case EV_ABS:
            name = find(kAbs);
            break;
        case EV_SYN:
            name = find(kSyn);
            break;
        case EV_MSC:
            name = find(kMsc);
            break;
        default:
            break;
    }
    return name ? name : "code " + std::to_string(code);
}
 
const char* key_state(int32_t value) {
    switch (value) {
        case 0:
            return "released";
        case 1:
            return "PRESSED";
        case 2:
            return "autorepeat";
        default:
            return "?";
    }
}
 
int dump_events(const std::string& path) {
    const DeviceInfo info = probe(path);
    UniqueFd fd = open_fd(path, O_RDONLY);
 
    // Install SIGINT *without* SA_RESTART, so a blocking read() returns EINTR
    // on Ctrl+C instead of silently restarting. (std::signal on glibc sets SA_RESTART!)
    struct sigaction sa {};
    sa.sa_handler = on_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    ::sigaction(SIGINT, &sa, nullptr);
 
    std::printf("Reading %s: \"%s\" [%s]   (Ctrl+C to stop)\n", path.c_str(), info.name.c_str(),
                classify(info).c_str());
    std::printf("sizeof(struct input_event) = %zu bytes\n\n", sizeof(input_event));
 
    std::array<input_event, 64> buf{};
    while (g_quit == 0) {
        const ssize_t n = ::read(fd.get(), buf.data(), sizeof buf);
        if (n < 0) {
            if (errno == EINTR) continue;  // loop condition sees g_quit
            throw_errno("read " + path);
        }
        const auto count = static_cast<std::size_t>(n) / sizeof(input_event);
        for (std::size_t i = 0; i < count; ++i) {
            const input_event& ev = buf[i];
            std::printf("%ld.%06ld  %-6s %-20s %6d", static_cast<long>(ev.input_event_sec),
                        static_cast<long>(ev.input_event_usec), type_name(ev.type),
                        code_name(ev.type, ev.code).c_str(), ev.value);
            if (ev.type == EV_KEY) std::printf("   %s", key_state(ev.value));
            if (ev.type == EV_SYN && ev.code == SYN_REPORT)
                std::printf("   ──── end of report ────");
            std::printf("\n");
        }
        std::fflush(stdout);
    }
    std::printf("\nstopped.\n");
    return 0;
}
 
}  // namespace
 
int cmd_input(Args args) {
    if (args.empty()) return list_devices();
    return dump_events(std::string{args[0]});
}
 
}  // namespace lumen::info
````
 
Three subtle points worth studying:
 
1. **`O_NONBLOCK` when probing.** We only want ioctls, and never want to block, so the probe fd is non-blocking. The dump fd is blocking, because we want `read()` to sleep until an event arrives.
2. **`sigaction` without `SA_RESTART`.** If you used `std::signal`, glibc installs the handler *with* `SA_RESTART`. The kernel would then silently restart the blocked `read()` after Ctrl+C, and the program wouldn't quit until you pressed another key. Without `SA_RESTART`, `read()` returns `EINTR`, the loop sees `g_quit` and exits. This exact bug is in many real programs.
3. **`ev.input_event_sec` instead of `ev.time.tv_sec`.** These macros keep working on 32-bit systems after the year-2038 `time_t` change, where the struct layout differs.
### Run it
 
````bash
./build/apps/lumen-info/lumen-info input
````
 
Find the `QEMU Virtio Keyboard` line, then dump it:
 
````bash
./build/apps/lumen-info/lumen-info input /dev/input/event2
````
 
**Click into the QEMU window** and press `A`:
 
````
Reading /dev/input/event2: "QEMU Virtio Keyboard" [keyboard]   (Ctrl+C to stop)
sizeof(struct input_event) = 24 bytes
 
1762800000.123456  EV_KEY KEY_A                     1   PRESSED
1762800000.123456  EV_SYN SYN_REPORT                0   ──── end of report ────
1762800000.201234  EV_KEY KEY_A                     0   released
1762800000.201234  EV_SYN SYN_REPORT                0   ──── end of report ────
````
 
Experiments:
- **Hold `A`.** After ~250 ms, `autorepeat` events (value 2) appear at ~30/s. That's the kernel's repeat. *(Ch 19 twist: compositors usually ignore these and do their own.)*
- **Mouse** (`QEMU Virtio Mouse`): `EV_REL REL_X/REL_Y`, i.e. *relative* motion.
- **Tablet** (`QEMU Virtio Tablet`): `EV_ABS ABS_X/ABS_Y` from 0 to 32767, i.e. *absolute* position, like a touchscreen. QEMU uses it so the guest cursor follows your real cursor exactly.
- On your **host** (with `sudo`): try your laptop's touchpad and watch the `ABS_MT_*` multitouch flood when you use two fingers.
> 🔮 **Predict:** you press `Shift+A`. How many `EV_KEY` events arrive, and does any of them say "capital A"?
 
<details><summary>Reveal</summary>
Four: `KEY_LEFTSHIFT 1`, `KEY_A 1`, `KEY_A 0`, `KEY_LEFTSHIFT 0`, each followed by `SYN_REPORT`. Nothing says "capital A", because the kernel just reports two physical keys. Shift + that key = "A" is decided in userspace, by xkbcommon in m19. The post office analogy in action.
</details>
> 🔮 **Predict:** why might you see `EV_MSC MSC_SCAN` before each `EV_KEY` on a real USB keyboard?
 
<details><summary>Reveal</summary>
`MSC_SCAN` carries the raw **scancode** (for USB, the HID usage, e.g. `0x70004` for A) *before* the kernel's translation to `KEY_A`. It's the "letter before the postcode was stamped". `udev hwdb` can remap scancodes to different keycodes, which is how laptop vendors' odd keys get fixed (Ch 17).
</details>
---
 
## Step 8 · Tests
 
Lumen's rule: **every module ships with tests.** For `core` we test the behaviours that would cause nasty bugs if they broke.
 
📄 **`tests/CMakeLists.txt`**
 
````cmake
find_package(Catch2 3 REQUIRED)
include(Catch)
 
add_executable(lumen-tests
  unit/test_unique_fd.cpp
  unit/test_error.cpp
  unit/test_log.cpp
)
target_link_libraries(lumen-tests PRIVATE lumen::core Catch2::Catch2WithMain)
lumen_target_defaults(lumen-tests)
 
catch_discover_tests(lumen-tests)   # each TEST_CASE becomes its own ctest entry
````
 
📄 **`tests/unit/test_unique_fd.cpp`**
 
````cpp
// test_unique_fd.cpp — prove UniqueFd closes exactly once and moves correctly.
#include <lumen/core/unique_fd.hpp>
 
#include <catch2/catch_test_macros.hpp>
 
#include <fcntl.h>
#include <unistd.h>
 
#include <cerrno>
#include <system_error>
 
namespace {
 
// An fd is "open" if fcntl(F_GETFD) succeeds on it.
bool is_open(int fd) {
    return ::fcntl(fd, F_GETFD) != -1 || errno != EBADF;
}
 
}  // namespace
 
TEST_CASE("UniqueFd closes its descriptor when destroyed", "[core][unique_fd]") {
    int raw[2];
    REQUIRE(::pipe(raw) == 0);
    {
        lumen::UniqueFd a{raw[0]};
        lumen::UniqueFd b{raw[1]};
        REQUIRE(is_open(raw[0]));
        REQUIRE(is_open(raw[1]));
    }
    CHECK_FALSE(is_open(raw[0]));
    CHECK_FALSE(is_open(raw[1]));
}
 
TEST_CASE("UniqueFd move transfers ownership", "[core][unique_fd]") {
    int raw[2];
    REQUIRE(::pipe(raw) == 0);
    lumen::UniqueFd keep{raw[1]};
 
    lumen::UniqueFd a{raw[0]};
    lumen::UniqueFd b{std::move(a)};
    CHECK_FALSE(a.valid());  // NOLINT(bugprone-use-after-move): testing moved-from state
    CHECK(b.get() == raw[0]);
 
    lumen::UniqueFd c;
    c = std::move(b);
    CHECK_FALSE(b.valid());  // NOLINT(bugprone-use-after-move)
    CHECK(c.get() == raw[0]);
    CHECK(is_open(raw[0]));
}
 
TEST_CASE("UniqueFd release gives up ownership without closing", "[core][unique_fd]") {
    int raw[2];
    REQUIRE(::pipe(raw) == 0);
    lumen::UniqueFd keep{raw[1]};
    int released = -1;
    {
        lumen::UniqueFd a{raw[0]};
        released = a.release();
        CHECK_FALSE(a.valid());
    }
    CHECK(is_open(released));
    ::close(released);
}
 
TEST_CASE("open_fd sets O_CLOEXEC and throws on failure", "[core][unique_fd]") {
    lumen::UniqueFd fd = lumen::open_fd("/dev/null", O_RDONLY);
    REQUIRE(fd.valid());
    CHECK((::fcntl(fd.get(), F_GETFD) & FD_CLOEXEC) != 0);
 
    try {
        (void)lumen::open_fd("/definitely/not/here", O_RDONLY);
        FAIL("expected std::system_error");
    } catch (const std::system_error& e) {
        CHECK(e.code() == std::errc::no_such_file_or_directory);
    }
}
````
 
Testing trick: `pipe()` gives us two real fds without needing any device, and `fcntl(fd, F_GETFD)` failing with `EBADF` proves an fd is closed.
 
📄 **`tests/unit/test_error.cpp`**
 
````cpp
// test_error.cpp — the three C error styles all become std::system_error.
#include <lumen/core/error.hpp>
 
#include <catch2/catch_test_macros.hpp>
 
#include <fcntl.h>
#include <unistd.h>
 
#include <cerrno>
#include <system_error>
 
TEST_CASE("check_syscall passes values through and throws on -1", "[core][error]") {
    CHECK(lumen::check_syscall(42, "ok") == 42);
 
    errno = 0;
    try {
        lumen::check_syscall(::open("/definitely/not/here", O_RDONLY | O_CLOEXEC), "open");
        FAIL("expected throw");
    } catch (const std::system_error& e) {
        CHECK(e.code() == std::errc::no_such_file_or_directory);
    }
}
 
TEST_CASE("check_neg_errno uses the returned value, not errno", "[core][error]") {
    errno = 0;  // stale errno must not matter
    try {
        lumen::check_neg_errno(-EINVAL, "drmSomething");
        FAIL("expected throw");
    } catch (const std::system_error& e) {
        CHECK(e.code() == std::errc::invalid_argument);
    }
    CHECK(lumen::check_neg_errno(0, "fine") == 0);
}
 
TEST_CASE("check_ptr throws on nullptr", "[core][error]") {
    int x = 1;
    CHECK(lumen::check_ptr(&x, "ok") == &x);
 
    errno = ENOMEM;
    int* null = nullptr;
    CHECK_THROWS_AS(lumen::check_ptr(null, "alloc"), std::system_error);
}
 
TEST_CASE("retry_eintr retries only on EINTR", "[core][error]") {
    int calls = 0;
    const int ret = lumen::retry_eintr([&] {
        ++calls;
        if (calls < 3) {
            errno = EINTR;
            return -1;
        }
        return 7;
    });
    CHECK(ret == 7);
    CHECK(calls == 3);
 
    calls = 0;
    const int fail = lumen::retry_eintr([&] {
        ++calls;
        errno = EIO;
        return -1;
    });
    CHECK(fail == -1);
    CHECK(calls == 1);
}
````
 
📄 **`tests/unit/test_log.cpp`**
 
````cpp
// test_log.cpp — LUMEN_DEBUG category parsing.
#include <lumen/core/log.hpp>
 
#include <catch2/catch_test_macros.hpp>
 
using lumen::log::Cat;
using lumen::log::parse_categories;
 
namespace {
constexpr uint32_t bit(Cat c) {
    return static_cast<uint32_t>(c);
}
}  // namespace
 
TEST_CASE("parse_categories handles lists, spaces, all and junk", "[core][log]") {
    CHECK(parse_categories("") == 0);
    CHECK(parse_categories("drm") == bit(Cat::Drm));
    CHECK(parse_categories("drm,input") == (bit(Cat::Drm) | bit(Cat::Input)));
    CHECK(parse_categories(" drm , input ") == (bit(Cat::Drm) | bit(Cat::Input)));
    CHECK(parse_categories("all") == bit(Cat::All));
    CHECK(parse_categories("nonsense,drm") == bit(Cat::Drm));
}
 
TEST_CASE("enabled() follows set_enabled_categories", "[core][log]") {
    lumen::log::set_enabled_categories(bit(Cat::Input));
    CHECK(lumen::log::enabled(Cat::Input));
    CHECK_FALSE(lumen::log::enabled(Cat::Drm));
    lumen::log::set_enabled_categories(0);
    CHECK_FALSE(lumen::log::enabled(Cat::Input));
}
````
 
Run them:
 
````bash
cmake --build build && ctest --test-dir build --output-on-failure
````
 
````
100% tests passed, 0 tests failed out of 10
````
 
And with sanitizers, which catch leaks, use-after-free and undefined behaviour:
 
````bash
cmake -S . -B build-asan -G Ninja -DLUMEN_SANITIZE=ON
cmake --build build-asan && ctest --test-dir build-asan
````
 
---
 
## Step 9 · Code quality, CI and docs
 
### `.clang-format`
 
📄 **`.clang-format`**
 
````yaml
# Lumen code style — run: clang-format -i $(git ls-files '*.cpp' '*.hpp')
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: Inline
AllowShortIfStatementsOnASingleLine: WithoutElse
DerivePointerAlignment: false
PointerAlignment: Left
IncludeBlocks: Regroup
IncludeCategories:
  - Regex: '^"'                 # this app's own headers: "commands.hpp"
    Priority: 1
  - Regex: '^<lumen/'           # Lumen module headers
    Priority: 2
  - Regex: '^<catch2/'          # third-party C++ libraries
    Priority: 3
  - Regex: '^<.*\.h>'           # C / POSIX / kernel / C-library headers
    Priority: 4
  - Regex: '^<'                 # C++ standard library
    Priority: 5
````
 
````bash
clang-format -i $(git ls-files '*.cpp' '*.hpp')   # format everything
````
 
The `IncludeCategories` keep every file's includes in the same order: own headers, Lumen modules, third-party C++, C/POSIX/kernel headers, then the standard library. Readers can see a file's dependencies at a glance.
 
### `.gitignore`
 
📄 **`.gitignore`**
 
````text
/build*/
/.cache/
compile_commands.json
*.ppm
*.png
!tests/golden/**/*.png
````
 
### GitHub Actions CI
 
📄 **`.github/workflows/ci.yml`**
 
````yaml
# CI: build + unit tests on every push and pull request.
# Two configurations: a normal Debug build, and one with ASan + UBSan.
name: ci
 
on:
  push:
  pull_request:
 
jobs:
  build-and-test:
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      matrix:
        sanitize: [OFF, ON]
    name: build (sanitize=${{ matrix.sanitize }})
    steps:
      - uses: actions/checkout@v4
 
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            g++ cmake ninja-build pkg-config catch2 libdrm-dev
 
      - name: Configure
        run: cmake -S . -B build -G Ninja -DLUMEN_SANITIZE=${{ matrix.sanitize }}
 
      - name: Build
        run: cmake --build build
 
      - name: Test
        run: ctest --test-dir build --output-on-failure
 
      - name: Smoke-test lumen-info
        run: ./build/apps/lumen-info/lumen-info version
 
  format:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Check clang-format
        run: |
          sudo apt-get install -y clang-format
          clang-format --dry-run --Werror $(git ls-files '*.cpp' '*.hpp')
````
 
Three jobs run on every push: a normal build + tests, a sanitizer build + tests, and a formatting check. A green badge on your README tells a reviewer instantly that the project builds and passes. Note what CI can **not** do yet: GitHub's runners have no GPU or `/dev/dri`. In m15 we solve that with vkms.
 
### Your first ADR
 
An **Architecture Decision Record** is a short note on *what* was decided and *why*. When you're asked "why C++ and not Rust?" or "why atomic KMS?" in an interview, these are your answers, already written.
 
📄 **`docs/adr/0001-language-build-and-layout.md`**
 
````markdown
# ADR 0001 — Language, build system and repository layout
 
- **Status:** accepted
- **Date:** milestone m01
 
## Context
 
Lumen talks to the Linux kernel (DRM/KMS, evdev, fbdev) and to C libraries
(libdrm, libinput, libudev, xkbcommon, libwayland, EGL, GBM). It must be fast
enough to composite full-HD frames at 60 Hz on the CPU, and readable as a
learning project and portfolio piece.
 
## Decision
 
1. **C++20**, compiled with g++ (clang also supported). C++20 gives us
   `std::span`, `std::format`, concepts and designated initialisers, all of
   which map well onto C structs and ioctls.
2. All C resources are owned by **RAII** types (`UniqueFd`, `CPtr<T, FreeFn>`).
   No raw `close()` or `drmModeFree*()` in application code.
3. C error conventions are converted to **`std::system_error`** at the
   boundary (`check_syscall`, `check_neg_errno`, `check_ptr`).
4. **CMake** with one static library per module (`lumen::core`, `lumen::pixel`,
   …), created through `lumen_add_module()` so all modules share one warning
   and sanitizer policy. External libraries are found with **pkg-config**.
5. Warnings are errors (`-Werror`, including `-Wconversion`); CI builds both a
   normal and an ASan + UBSan configuration.
6. **Catch2 v3** for unit tests, one `TEST_CASE` per behaviour, discovered by
   CTest.
 
## Layout
 
```
src/<module>/include/lumen/<module>/*.hpp   public headers
src/<module>/src/*.cpp                      implementation
apps/<app>/                                 executables
tests/unit/                                 unit tests
docs/adr/                                   decisions like this one
```
 
Dependencies only point "down": `server → render/input/backend → … → core`.
`core` depends on nothing but the C++ standard library and POSIX.
 
## Consequences
 
- Contributors need a C++20 compiler (Ubuntu 22.04's g++ 11 lacks
  `std::format`; Ubuntu 24.04+ is required).
- Exceptions are used for errors, so code that runs inside C callbacks
  (libwayland, libinput) must catch at the callback boundary, since an exception
  must never unwind through C frames. This will matter from m19 onwards.
````
 
### `docs/architecture.md` and `README.md`
 
📄 **`docs/architecture.md`**
 
````markdown
# Lumen architecture
 
*This document grows with every milestone.*
 
## Modules
 
| Module | Since | Responsibility |
|---|---|---|
| `core` | m01 | RAII wrappers (`UniqueFd`, `CPtr`), errno → exceptions, logging |
 
## Applications
 
| App | Since | Purpose |
|---|---|---|
| `lumen-info` | m01 | Inspect DRM devices (`drm`) and input devices (`input`) |
 
## Dependency rule
 
Modules may only depend on modules *below* them:
 
```
server
  ├── render ── compose ── raster ── pixel
  ├── input ── session
  └── backend ── display
                     └────────────── core   (everything depends on core)
```
````
 
📄 **`README.md`**
 
````markdown
# Lumen
 
**A from-scratch Linux display & input engine and Wayland compositor, in modern C++20.**
 
Lumen is being built one milestone at a time while learning the Linux graphics and
input stack, from pixels and color math up to DRM/KMS, libinput, Wayland and the GPU.
 
## Status
 
| Milestone | Feature | State |
|---|---|---|
| m01 | Repo, `lumen::core`, `lumen-info drm` / `input`, tests, CI | ✅ |
| m02 | `Image`, PPM output, aliasing demo | ⏳ |
 
## Build (Ubuntu 24.04+)
 
```bash
sudo apt install g++ cmake ninja-build pkg-config catch2 libdrm-dev
cmake -S . -B build -G Ninja
cmake --build build
ctest --test-dir build
```
 
Options: `-DLUMEN_SANITIZE=ON` (ASan + UBSan), `-DLUMEN_WERROR=OFF`, `-DLUMEN_TESTS=OFF`.
 
## Try it
 
```bash
./build/apps/lumen-info/lumen-info drm            # GPUs, drivers, capabilities, monitors
./build/apps/lumen-info/lumen-info input          # keyboards, mice, touchpads…
sudo ./build/apps/lumen-info/lumen-info input /dev/input/event3   # raw events
LUMEN_DEBUG=drm ./build/apps/lumen-info/lumen-info drm            # with debug logs
```
 
## Documentation
 
- [Architecture](docs/architecture.md)
- [Architecture Decision Records](docs/adr/)
````
 
---
 
## Step 10 · Commit and tag m01
 
````bash
cd ~/lumen
clang-format --dry-run --Werror $(git ls-files -co --exclude-standard '*.cpp' '*.hpp')
cmake --build build && ctest --test-dir build
git add .
git commit -m "feat(core,lumen-info): m01 project kickoff
 
- CMake module system with strict warnings and sanitizer option
- lumen::core: UniqueFd, CPtr, errno helpers, categorised logger
- lumen-info drm: DRM nodes, driver, caps, connectors, master status
- lumen-info input: device listing/classification and raw event dump
- Catch2 unit tests, clang-format, GitHub Actions CI, ADR 0001"
git tag -a m01 -m "Milestone 01: project kickoff"
````
 
Then create an empty repo on GitHub and push:
 
````bash
git remote add origin git@github.com:<you>/lumen.git
git push -u origin main --tags
````
 
Open the **Actions** tab and watch CI go green ✅.
 
> 🧙 **Commit message style:** `type(scope): summary` (*Conventional Commits*): `feat`, `fix`, `refactor`, `test`, `docs`. It reads well in `git log --oneline`, and tools can generate changelogs from it. One tag per milestone means `git diff m13 m14` shows exactly what a chapter added: a great way to revise.
 
---
 
## 🐛 Break it on purpose
 
Deliberate bugs teach you what real failures look like.
 
**1. Permission denied, explained.**
````bash
sudo -u nobody ./build/apps/lumen-info/lumen-info drm
````
→ `cannot open: open /dev/dri/card0: Permission denied` + the hint. From m14, the *same* errno also appears when you try to modeset **without being master**. Two different causes, one symptom.
 
**2. A partial event read.** In `dump_events`, change the read size to `10`:
````cpp
const ssize_t n = ::read(fd.get(), buf.data(), 10);
````
→ `read /dev/input/event2: Invalid argument`. evdev refuses reads smaller than one `input_event`. The kernel enforces struct boundaries; you can never get half an event.
 
**3. The SA_RESTART trap.** Replace the `sigaction` block with `std::signal(SIGINT, on_sigint);`. Run the dump, press Ctrl+C, and… nothing, until you press another key in the QEMU window. Now you've *felt* why the comment is there.
 
**4. Leak a struct, catch it with ASan.** In `inspect_node`, replace `VersionPtr ver{…}` with a raw pointer that's never freed:
````cpp
drmVersion* ver = drmGetVersion(fd.get());
````
Rebuild `build-asan` and run `lumen-info drm` → LeakSanitizer reports the `drmGetVersion` allocation with a stack trace. This is why every C struct goes into a `CPtr`.
 
**5. Break a test.** In `UniqueFd::reset`, comment out the `::close`. Run `ctest` → `UniqueFd closes its descriptor when destroyed` fails. Your tests just caught an fd leak that would otherwise exhaust the process's fd limit after a few thousand hotplugs.
 
**6. Fight GNOME for the stage.** On your **host** desktop:
````bash
modetest -s <connector>:<mode>
````
→ fails on `SetCrtc` with *Permission denied*: Mutter is master. Press `Ctrl+Alt+F3`, log in on the text console, and try again: it works, because switching VT makes GNOME **drop** master. (`Ctrl+Alt+F2` or `F1` to go back.) Lumen will do this dance properly in m18.
 
---
 
## 🧙 Wizard corner
 
- **Primary vs render node is a security boundary.** Anything that only renders (a browser tab, a game) should use `renderD*`: it can't touch the display or grab other apps' buffers by global name. Only the compositor needs `card*`.
- **First opener becomes master.** If no master exists, the first `open()` of a primary node makes you master. Compositors call `drmDropMaster` / `drmSetMaster` on VT switch, coordinated by logind (m18).
- **Stable paths:** `/dev/dri/by-path/pci-0000:00:02.0-card` and `/dev/input/by-id/usb-…-event-kbd` survive reboots and replugs; `card1` and `event5` don't.
- **`udevadm info /dev/input/event3`** shows `ID_INPUT_KEYBOARD=1`, `ID_INPUT_TOUCHPAD=1`, the seat and the tags. libinput's whole device classification starts from these properties, and you just reimplemented a mini version of it.
- **Headers are documentation.** Skim `/usr/include/linux/input-event-codes.h`, `/usr/include/drm/drm_mode.h` and `/usr/include/xf86drmMode.h` now. They're surprisingly readable, and you'll live in them.
- **Elixir** (`elixir.bootlin.com`) lets you click through kernel source. Look up `evdev_read` and see the `count < input_event_size()` check that produced your `EINVAL` in Break-it #2.
- **Exceptions vs C callbacks:** an exception must never unwind through C stack frames (libwayland, libinput callbacks). ADR 0001 already notes it; from m19 every callback will have its own `try/catch` boundary.
---
 
## ✅ Self-check
 
<details>
<summary>1. Name the four layers of the stack and one Ubuntu component in each.</summary>
Hardware (GPU, panel, keyboard) · Kernel (i915/amdgpu/virtio_gpu, evdev) · System userspace (Mutter/gnome-shell, Mesa, libinput, logind) · Applications (Firefox, GTK apps). Lumen lives in layer 3.
</details>
<details>
<summary>2. card0 vs renderD128?</summary>
`card0` is the primary node: display control (KMS) + rendering, with one master at a time. `renderD128` is the render node: rendering only, no display control, safe for many unprivileged clients.
</details>
<details>
<summary>3. Why does lumen-info drm say "master: YES" in the VM but "no" on your desktop?</summary>
On the desktop, gnome-shell is already master, and there can be only one per device. In the VM nobody else had the card open, so the first opener (lumen-info) automatically became master.
</details>
<details>
<summary>4. Why can you open /dev/dri/card1 without sudo on Ubuntu, but not /dev/input/event*?</summary>
systemd-logind adds an ACL granting the active local session's user access to GPU nodes. Input nodes get no such ACL, since reading them would allow keylogging. Compositors obtain input fds from logind instead (m18).
</details>
<details>
<summary>5. Does the kernel know which character you typed?</summary>
No. Only keycodes (physical keys). Layouts, Shift and dead keys are applied in userspace by xkbcommon.
</details>
<details>
<summary>6. What does SYN_REPORT mean, and why must you wait for it?</summary>
It ends one frame of events that happened at the same instant. Applying REL_X before seeing its matching REL_Y (or a touch's X before its Y) produces jagged, wrong motion.
</details>
<details>
<summary>7. Name the three C error conventions and Lumen's helper for each.</summary>
`-1` + errno → `check_syscall` · `-errno` returned → `check_neg_errno` · `NULL` + errno → `check_ptr`.
</details>
<details>
<summary>8. Why is UniqueFd's copy constructor deleted, and why does open_fd add O_CLOEXEC?</summary>
Copying would cause a double `close()`, possibly closing an unrelated reused fd. `O_CLOEXEC` stops child processes (apps Lumen launches later) from inheriting device fds.
</details>
<details>
<summary>9. Why does the dump use sigaction instead of std::signal?</summary>
glibc's `signal()` sets `SA_RESTART`, so a blocking `read()` would restart after Ctrl+C and the program wouldn't exit until the next input event. Without `SA_RESTART`, `read()` returns `EINTR` and the loop exits.
</details>
<details>
<summary>10. What does the ioctl request number encode?</summary>
Direction (none/read/write/both), argument size, a type letter (`'d'` for DRM, `'E'` for evdev) and a command number.
</details>
<details>
<summary>11. Why does `log::debug` check enabled() before formatting?</summary>
So disabled debug logs cost only one atomic load. Lumen will log in hot paths (every vblank, every input event), and formatting strings there would waste CPU even when nobody reads them.
</details>
---
 
## 🏁 Definition of done: m01
 
- [ ] Explored my host: session type, compositor, DRM nodes, master, ACLs, input devices, EDID
- [ ] VM installed; `run-gfxlab.sh` works; `ssh -p 2222` works; toolbox installed
- [ ] Saw `modetest` color bars in the QEMU window
- [ ] `lumen/` repo created with the layout above
- [ ] `cmake --build build` succeeds with **zero warnings** (they're errors!)
- [ ] `ctest` passes in both `build` and `build-asan`
- [ ] `lumen-info drm` run in the VM **and** on the host; I can explain the master difference
- [ ] `lumen-info input` lists and classifies devices; dump shows keyboard, mouse and tablet events and autorepeat
- [ ] Traced `lumen-info drm` with `strace`
- [ ] Did at least three "Break it on purpose" exercises
- [ ] Pushed to GitHub, CI green ✅, tag `m01` pushed
---
 
## 📚 Go further
 
- Bootlin, *Understanding the Linux Graphics Stack*: the "Software Aspects → System-agnostic overview" and "Linux kernel overview" slides (this chapter expands them)
- Kernel docs: *GPU Driver Developer's Guide → Userland interfaces* (`docs.kernel.org/gpu/drm-uapi.html`), especially "Primary Nodes, DRM Master and Authentication" and "Render nodes"
- Kernel docs: *Input subsystem → Event codes* (`docs.kernel.org/input/event-codes.html`)
- `man 2 ioctl`, `man 2 open`, `man 7 signal` (the section on interrupted system calls and `SA_RESTART`)
- Catch2 v3 docs: *Tutorial* and *Assertion macros*
- *Architecture Decision Records* by Michael Nygard (the original ADR blog post)
---
 
> **Next → Chapter 02 · Light, Pixels & Sampling.** We zoom all the way in: what a pixel actually *is*, why photos of brick walls ripple (aliasing!), and the theorem that decides how many pixels you need. Lumen gets its first real module, `lumen::pixel`, and starts producing images. (milestone m02
