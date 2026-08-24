# Virtual Memory — The Whole System, in Two Flows

**Rule to hold onto:** every access is really `translate → check cache → touch RAM/disk`.
Translation always happens the same way. Only the LAST step differs between a **read** and a **write**.

<img width="1408" height="768" alt="Gemini_Generated_Image_4s5ubn4s5ubn4s5u" src="https://github.com/user-attachments/assets/88a50494-cc7a-47e5-afc4-3db858839bf2" />


---

## CASE 1 — READ: we want data from `0xA3F`

```
we want data from 0xA3F
 └─ split address: VPN = upper bits, VPO = 0xA3F (offset, rides along unchanged)

 └─ MMU goes to TLB with VPN
      VPN is split again → TLBT (tag) + TLBI (index)
      TLBI picks the SET (TLB has ~64 entries, grouped into sets)
      search that set's few entries, comparing TLBT against each entry's tag
      each TLB entry holds: [valid | tag | PPN]

      CASE A — TLB HIT (tag matched, valid=1)
         └─ got PPN straight from the TLB — no page-table walk needed
         └─ go straight to "CHECK CACHE" below

      CASE B — TLB MISS (no matching valid entry in that set)
         └─ MMU's page-table walker takes CR3 (base of THIS process's page table) + VPN
         └─ walks the table (1 level shown here; really K levels — Ch.3, Section 8)
         └─ fetches the PTE from RAM at that computed PTE address
         └─ RAM returns the PTE: [valid bit | PPN or disk-location]

         now check the PTE's valid bit (same check TLB-hit path skipped):
            valid = 0  → page NOT in RAM. Look at what's stored instead:
               ├─ NULL / no vm_area_struct covers this address
               │      → SEGMENTATION FAULT (you were never allowed to touch this)
               └─ a real disk location is stored here
                      → PAGE FAULT (legal address, just not resident right now)
                      → OS page-fault handler runs:
                           1. picks a VICTIM page currently in RAM
                           2. victim DIRTY? → write it back to disk first
                              victim CLEAN?  → just discard it, disk copy already correct
                           3. reads the REQUESTED page from disk into that freed frame
                           4. updates the PTE: valid=1, PPN = new frame
                           5. loads this translation into the TLB
                      → RESTART the faulting instruction from scratch
                      → this time: TLB now has it → HIT → continue below

            valid = 1  → PPN is right here, already in RAM → continue below

 └─ CHECK CACHE (this step runs on BOTH the TLB-hit and TLB-miss paths —
    it's a separate question from "is it in RAM")
      combine PPN + VPO = physical address
      split physical address → CT (tag) + CI (index) + CO (offset within line)
      look up L1 using CI:
         HIT  (CT matches) → data is already sitting in the 64-byte line → return it
         MISS → fetch the ONE 64-byte line containing this address
                 (not the whole 4 KB page — that already happened above, if at all)
                 from L2 → L3 → RAM (whichever level has it), load it into L1

 └─ return the requested bytes to the CPU register — READ COMPLETE
```

---

## CASE 2 — WRITE: we want to write to `0xA3F`

**Everything above (TLB → page table → page fault → cache) happens exactly the same way first.**
A write needs a *resolved, cached* physical address too, before it can write anything.
The difference only shows up in **one extra permission check**, and in what happens *after* the write:

```
we want to WRITE to 0xA3F
 └─ (run the ENTIRE translation flow from Case 1 above, unchanged, to get:
      a valid PTE, a physical address, and the target 64-byte line loaded into L1)

 └─ NEW STEP — check the PTE's write permission bit (R/W bit), not just its valid bit
      R/W = 1 (writable)
         └─ perform the write directly into the L1 cache line
         └─ mark that cache line DIRTY (differs from RAM now — write-back policy)
         └─ mark the PTE's dirty bit too (MMU sets this automatically on x86)
         └─ WRITE COMPLETE — nothing touched RAM or disk yet.
             RAM gets updated only when this dirty line is evicted from cache.
             Disk gets updated only when this dirty PAGE is evicted from RAM.
             (If the process exits first, these writes may NEVER reach disk —
              being lazy here is a deliberate performance win, Section 2.)

      R/W = 0 (read-only) → hardware raises a PROTECTION FAULT, not a normal page fault
         └─ OS fault handler checks WHY this page is read-only:
              ├─ genuinely illegal (e.g. writing to .text) → SEGMENTATION FAULT / crash
              └─ marked COPY-ON-WRITE (this page is shared, e.g. right after fork())
                     → allocate a NEW physical frame
                     → copy the shared page's contents into it
                     → update THIS process's PTE: point at the new frame, R/W = 1
                     → RESTART the write instruction
                     → this time: R/W = 1 → normal write path above → succeeds,
                       writing only into THIS process's own private copy
                     → the other process's page is completely untouched
```

---

## 12. FULL WORKED EXAMPLE #1 — tracing `int x = 10;` (a WRITE, first touch)

```cpp
int x = 10;   // x is a local variable — lives on the STACK
              // (a private, demand-zero region, Section 10)
```

Say the compiler assigns `x` the virtual address `0x00007fff'a000'1a34`, and this is the **very first time** this stack page has ever been touched.

```
1.  CPU executes:  MOV [0x00007fffa0001a34], 10
2.  Split VA:  VPN = upper 36 bits,  VPO = 0xA34 (lower 12 bits)
3.  Check L1 TLB for this VPN → MISS (first touch, nothing cached)
4.  Walk 4-level page table via CR3 → PML4 → PDPT → PD → PT
5.  Final-level PTE: Valid bit = 0
       → this stack page has NEVER been backed by real physical memory
6.  PAGE FAULT raised
7.  OS fault handler: checks vm_area_struct — is this address inside the STACK region? YES, legal.
8.  This is a DEMAND-ZERO page (no file backs it).
       OS finds a free physical frame, fills it with ZEROS.
9.  OS updates the PTE: valid=1, PPN = new frame, R/W=1 (writable — it's stack memory)
10. OS ALSO loads this new translation into the TLB
11. Faulting instruction RESTARTS FROM SCRATCH
12. CPU re-issues the SAME virtual address
13. Check TLB again → HIT this time (just installed in step 10)!
       Get PPN directly, no page-table walk needed.
14. Combine PPN + VPO = Physical Address
15. Check L1 d-cache at this physical address → MISS (freshly zeroed page never cached)
16. Fetch the 64-byte cache line containing this address from DRAM (~200 cycles)
17. THE ACTUAL WRITE HAPPENS: value 10 is stored at offset 0xA34 within this cache line, in L1
18. Cache line marked DIRTY (differs from what's in DRAM now)
19. Instruction complete. x now holds 10 — ONLY in L1 cache (and eventually DRAM).
       NOTHING has been written to disk yet.
```

### State of every layer after this ONE instruction

| Layer | State after `x = 10` |
|---|---|
| **TLB** | Valid VPN→PPN translation cached — next nearby stack access is a TLB hit |
| **Page table (PTE)** | valid=1, points to the new physical frame, R/W=1 |
| **L1 cache** | Holds the 64-byte line containing `x`, marked **dirty**, value 10 at the right offset |
| **DRAM** | Still holds *old* content until the dirty line eventually gets evicted/flushed (write-back, Section 2) |
| **Disk / swap** | Untouched. May *never* be touched if this stack frame is popped before eviction pressure forces it |


If another thread (running on a different core) tries to read or write the variable `x`:

1. **Cache Coherency Protocol (like MESI / MOESI):** The CPU cores use a hardware coherency protocol. When Core B wants `x`, its cache controller sends a request across the interconnect.
2. **State Transition:** Core A's L1 cache sees that another core wants the data it has marked as **Modified (Dirty)**. Core A must intercept this request. It changes its own cache line state to **Shared** (or **Invalid** if Core B is writing to it) and flushes the updated value (`10`) out of its L1/L2 cache hierarchy.
3. **Updating RAM / L3:** Depending on the exact cache architecture and write-back policy, the updated value is typically written down to the shared **L3 cache** (or directly to **DRAM** depending on whether it's an exclusive or inclusive cache hierarchy) so that Core B can fetch the correct, up-to-date value of `10`.

### If this had instead been a write to a **copy-on-write** page

Steps 5–8 change: the PTE would already be **valid** (page exists, cached, shared) but marked **read-only**. The fault raised is a **protection fault**, not a not-present fault, and the handler's job is "allocate new frame, copy contents, repoint PTE, mark read-write" instead of "zero-fill a new frame." Same overall shape (fault → handler → fix PTE → restart), different specific repair — which is exactly why the fault handler must check *which kind* of fault occurred (Section 6) before deciding what to do.

---

## 13. FULL WORKED EXAMPLE #2 — tracing `int y = x;` (a READ, then a WRITE, in the same instruction)

Now assume: the page containing `x` is **already TLB-cached and dirty in L1** (from Example 1). The page that will hold `y` is a **brand-new, never-touched** stack slot.

```cpp
int y = x;   // READ x, WRITE the value into a NEW variable y
```

### Reading x (the source)

```
1. CPU issues x's virtual address
2. Split VA → VPN_x + VPO_x
3. Check TLB for VPN_x → HIT (installed during Example 1) — no table walk needed!
4. Combine PPN_x + VPO_x = Physical Address of x
5. Check L1 cache at that address → HIT (line is already resident, dirty, from Example 1)
6. Return the value 10 to the CPU — fast path, both TLB and cache hit
```

### Writing y (the destination — brand new page)

```
7.  CPU issues y's virtual address
8.  Split VA → VPN_y + VPO_y
9.  Check TLB for VPN_y → MISS (never touched before)
10. Walk 4-level page table → final PTE: Valid = 0
11. PAGE FAULT → handler checks vm_area_struct → inside STACK region, legal
12. Demand-zero page: allocate a fresh frame, zero-fill it
13. Update PTE: valid=1, PPN = new frame, R/W=1
14. Load translation into TLB
15. RESTART the instruction from scratch
16. Re-issue VA for y → TLB HIT this time
17. Combine PPN_y + VPO_y = Physical Address
18. L1 cache check → MISS (freshly zeroed page never cached) → fetch 64-byte line from DRAM
19. THE WRITE HAPPENS: value 10 (read from x) is stored at y's offset in this new cache line
20. Line marked DIRTY
```

**Why the read and the write take completely different paths through the *same* diagram, one instruction apart:**
- **Reading `x`** hits *everywhere* (TLB hit, cache hit) because it's re-touching memory from the immediately preceding instruction — pure temporal locality.
- **Writing `y`** misses *everywhere* (TLB miss → page fault → cache miss) because it's the very first touch of a page that has never been backed by physical memory at all. It must go through the *entire* demand-zero-page machinery of Section 6 before the write can even happen.

Same mechanism, same diagram — but locality (or the total lack of it) determines which branches get taken.

---

## The one-paragraph version

**Read:** split the address → look in the TLB → if missing, walk the page table in RAM →
if the page isn't resident, page-fault it in from disk (evicting a victim if needed) →
once you have a physical address, check the cache for its 64-byte line → fetch it if missing → return the bytes.

**Write:** identical path to get a valid, cached physical address — **plus one extra check**:
is this page actually writable? If yes, write into the cache line, mark it dirty, and let write-back
policy lazily push it to RAM/disk later. If the page is read-only because of copy-on-write, take a
protection fault, duplicate the page privately, mark it writable, and retry. If it's read-only for a
real reason (like code memory), that's a crash, not a fault to recover from.

## Hardware vs software, one more time

| Does the work | Layer |
|---|---|
| Splitting the address, TLB lookup, page-table walk, cache lookup, dirty/valid bit bookkeeping | **Hardware** (MMU, TLB, cache controller — automatic, every access) |
| Deciding *what to do* when a fault is raised (evict which victim, zero-fill vs. load-from-disk vs. COW-copy vs. segfault) | **Software** (OS page-fault handler — only runs when hardware raises an exception) |
| The page table's bytes themselves | **Data** — sits in hardware RAM, but its content is written only by OS software; hardware only ever reads it |
