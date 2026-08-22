# Virtual Memory — The Whole System, in Two Flows

**Rule to hold onto:** every access is really `translate → check cache → touch RAM/disk`.
Translation always happens the same way. Only the LAST step differs between a **read** and a **write**.

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
