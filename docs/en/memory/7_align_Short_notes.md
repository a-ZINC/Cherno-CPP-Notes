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

**`alignof`** is a query (like `sizeof`), and **`alignas`** is a command.

### `alignof` (The Query)

It asks the compiler: *"What is the alignment requirement (in bytes) for this type?"*

```cpp
size_t alignment = alignof(Vector4); // Returns 16
size_t int_align = alignof(int);     // Usually returns 4

```

Use it when you need to dynamically check or enforce alignment in your code (like passing it into your `align_up` function).

### `alignas` (The Command)

It tells the compiler: *"Force this variable, array, or struct to be aligned to this specific byte boundary, even if it normally wouldn't need to be."*

```cpp
// Force a normal float array to align to 32 bytes for AVX-256
alignas(32) float buffer[8]; 

// Force a struct to be 16-byte aligned for SIMD
struct alignas(16) Vector4 {
    float x, y, z, w;
};

```

Use it when you are declaring data structures and want to guarantee they land on safe memory boundaries automatically.

---

Ready to jump back to **Challenge 2**?
*If Item #0 starts at `0x1000` and `sizeof(WeirdItem)` is 20, where does raw pointer math land Item #1, and how do you fix it with `align_up`?*

## The absolute minimum version, if you remember nothing else

**`align_up(value, alignment) = (value + alignment - 1) & ~(alignment - 1)`
— call it on every offset before you place something manually in memory,
and default to 16 when in doubt.**


## Memory Alignment: The Essential Notes

### The Universal Formula

```cpp
inline uintptr_t align_up(uintptr_t value, size_t alignment) {
    return (value + alignment - 1) & ~(alignment - 1);
}

```

### Why Alignment Matters

* **Hardware Enforcement:** SIMD instructions (SSE, AVX, AVX-512) demand strict 16-, 32-, or 64-byte alignment. Feeding them an unaligned address triggers a hardware fault (`SIGSEGV`).
* **Performance Optimization:** Aligned memory prevents cache-line splits, allowing the CPU to fetch data in a single clock cycle without extra bus overhead.

### Compiler Automation vs. Manual Intervention

* **Automatic:** Standard variables, `new T`, and STL containers are automatically aligned by the compiler.
* **Manual:** Required when bypassing the compiler—such as writing custom memory allocators, parsing raw byte buffers/network streams, doing pointer arithmetic, or using placement `new`.

### The Golden Rule of Offsets

> Aligning the start of a buffer **does not** automatically align what lives inside it. Every time you advance a pointer past a header or a variable-length payload, you must explicitly re-align.

### The Toolkit

* **`alignof(T)`**: Queries a type's alignment requirement in bytes.
* **`alignas(N)`**: Forces the compiler to align a struct or variable to a specific byte boundary.
* **Array Stride Formula**:
```cpp
size_t stride = align_up(sizeof(T), alignof(T));

```


Ensures every item in a packed array or custom pool stays safely aligned, preventing padding/size mismatches from breaking subsequent elements.
