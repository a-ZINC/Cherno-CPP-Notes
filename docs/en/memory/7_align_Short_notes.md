# Alignment — The One Page You Actually Keep Open

## The one formula (type this without thinking)

```cpp
inline uintptr_t align_up(uintptr_t value, size_t alignment) {
    return (value + alignment - 1) & ~(alignment - 1);
}
```

That's it. Every alignment problem you'll ever hit reduces to calling this
correctly, on the right value, with the right alignment.

---

## The 3 questions (run this checklist any time you touch raw memory)

```
1. What is about to live at this address?          -> some type T
2. What does IT require?                             -> alignof(T)
3. Does my address satisfy that RIGHT NOW?            -> addr % alignof(T) == 0 ?
                                                          NO -> align_up() it
```

**Trigger to even ask this at all:** you're doing `malloc`, `char buffer[]`,
`reinterpret_cast`, placement `new`, writing an allocator, or touching
SIMD. Normal variables / `new T` / STL containers → skip this entirely,
already handled.

---

## The 5 lines you'll actually copy-paste

```cpp
// Get an over-aligned block from the OS/libc directly (size must be a
// multiple of alignment):
void* p = std::aligned_alloc(alignment, size);

// Round any address or size up to a required alignment:
uintptr_t a = align_up(raw, required_alignment);

// Guarantee a raw buffer is aligned for some type T:
alignas(T) char buffer[N];

// Compute correct array stride so EVERY element stays aligned, not just #0:
size_t stride = align_up(sizeof(T), alignof(T));

// Verify it, cheaply, anywhere, anytime (put this in a self-test):
assert(reinterpret_cast<uintptr_t>(ptr) % alignof(T) == 0);
```

---

## The one sentence that explains every bug in this whole topic

> **Aligning a container's START does not align something living INSIDE
> it, unless the offset between them is itself a multiple of the
> alignment you need.**

```
base % 16 == 0            <- container is fine
(base + 8) % 16 == 8       <- but +8 into it is NOT fine
(base + 16) % 16 == 0      <- +16 into it IS fine, because 16 is a
                               multiple of 16 and 8 is not
```

---

## When you're writing YOUR OWN allocator specifically

```
payload_offset = align_up(header_size, 16)     // not just header_size
total_block_size = align_up(header+payload+footer, 16)   // not just header+payload
```

Default to **16** unless you have a specific reason for more (SIMD/cache
line work sometimes wants 32/64) or know for certain you'll only ever
store things needing less.

---

## What happens if you get it wrong (so you remember WHY you're doing this)

```
SIMD aligned load on bad address        -> real SIGSEGV, immediately
misaligned scalar/struct access on x86  -> often silently "works" until
                                            it doesn't (different
                                            compiler/CPU/optimization level)
reinterpret_cast to misaligned T*       -> compiles fine, ALWAYS --
                                            danger deferred to first use
memcpy on raw bytes                     -> always safe, no alignment
                                            requirement, ever
```

---

## The absolute minimum version, if you remember nothing else

**`align_up(value, alignment) = (value + alignment - 1) & ~(alignment - 1)`
— call it on every offset before you place something manually in memory,
and default to 16 when in doubt.**
