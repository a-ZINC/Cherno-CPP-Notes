# C++ Multithreading Notes — Batching Work Across Threads

Notes from "Chili" (Chili's Programming) multithreading video: splitting a big
dataset into smaller chunks processed repeatedly (like frames in a game loop),
and the mystery of why smaller batches run *slower* despite doing the same
total amount of work.

---

## 1. Context / Goal

Previous videos covered:
- Synchronizing workloads across threads and gathering results.
- Hidden/"false" synchronization from cache-line sharing (accessing data at
  nearby memory addresses causes contention even without explicit locks).

This video's goal: instead of processing one **huge** dataset once, process
the **same total amount of data** but split into many **smaller batches** run
repeatedly — like a game doing many small updates per second instead of one
giant update per second.

Example: instead of processing 50,000,000 elements once per second, process
500,000 elements, 100 times per second. Same throughput, smaller granularity,
needed so a UI/game can redraw at a good frame rate instead of feeling like a
slideshow.

---

## 2. Key C++ tool: `std::span`

`std::span<T>` (C++20) is a **non-owning view** over a contiguous range of
elements (array, vector, part of a vector, raw pointer range, etc.). It gives
you `begin()`/`end()` so it works in range-based for loops, without copying
data and without needing a fixed compile-time size.

Why it's useful here: it lets a worker function accept "a chunk of the
dataset" without caring whether that chunk is a whole vector or just a slice
of one.

**Gotcha:** `std::thread`'s constructor forwards arguments through a template,
and this does *not* implicitly convert a container to `std::span` the way a
normal function call would. You have to construct the `std::span` explicitly:

```cpp
// This FAILS to compile (no matching overloaded function):
std::thread t(process_data_set, data_vector);

// This WORKS — construct the span explicitly:
std::thread t(process_data_set, std::span(data_vector));
```

---

## 3. `std::jthread` vs `std::thread`

`std::jthread` (C++20) is like `std::thread`, but:
- It automatically **joins** in its destructor (no need to remember `.join()`).
- It also supports cooperative cancellation via `std::stop_token` (not covered
  in this transcript, but worth knowing).

This matters when you're spinning up and tearing down a batch of worker
threads on every iteration of a loop — using `jthread` and just letting a
`vector<jthread>` go out of scope (or calling `.clear()`) joins everything for
you automatically.

---

## 4. The structure of the demo program

Two "modes," selectable by a command-line flag:

- `do_biggie()` — process the entire dataset (e.g. 50,000,000 × 4 = 200,000,000
  elements) in one pass, split across a fixed number of worker threads. Baseline
  timing: **~1.1 seconds**.
- `do_smallies()` — take the *same* total dataset, but slice it into `N` equal
  subsets, and for each subset spin up a fresh batch of worker threads,
  process it, join, accumulate a running total, then move to the next subset.

Shared logic (generating the dataset) is factored into its own function,
`generate_data_set()`, so both modes use identical dataset-construction code.
Note: returning a local `std::vector` by value from a function does **not**
need `std::move` — Return Value Optimization (RVO) already avoids the copy,
and is typically *more* efficient than an explicit move.

---

## 5. The experiment & the mystery

Same total data, same 4 worker threads either way — only the **granularity**
of batching changes:

| Subsets the data is divided into | Time taken |
|---|---|
| 1 (i.e. same as `do_biggie`)     | ~1.1 s |
| 100                              | ~1.19 s |
| 1,000                            | ~1.3 s |
| 10,000                           | ~2.5 s |

Dividing the exact same workload into more, smaller batches makes it roughly
**2.5× slower**, even though:
- Total element count processed is identical.
- Number of worker threads used per batch is identical (still 4).
- The extra "for" loop driving the batching is trivially cheap — far too
  cheap to explain a 2.5× slowdown by itself.

**Open question posed at the end of the video:** what is actually causing the
slowdown as batches get smaller and more numerous? (Answer promised in the
next video — likely candidates to think about: thread creation/destruction
overhead per batch, OS scheduling overhead, cache warm-up/locality effects
resetting on every batch, or synchronization/join costs repeated many times.)

---

## 6. Practice Code

Below is a self-contained reconstruction of the demo you can compile and
experiment with (requires C++20 for `std::span` and `std::jthread`).

```cpp
// multithread_batching_demo.cpp
// Compile (g++):  g++ -std=c++20 -O2 -pthread multithread_batching_demo.cpp -o demo
// Compile (MSVC): cl /std:c++20 /O2 /EHsc multithread_batching_demo.cpp
//
// Usage:
//   ./demo            -> runs the "biggie" (single big batch) version
//   ./demo --small    -> runs the "smallies" (many small batches) version

#include <chrono>
#include <cstring>
#include <iostream>
#include <random>
#include <span>
#include <thread>
#include <vector>

constexpr size_t k_data_set_size = 50'000'000;
constexpr size_t k_num_data_sets = 4;
constexpr size_t k_num_workers = 4;

// A simple RAII stopwatch, mirroring the "chili timer" from the video.
class Timer
{
public:
    Timer() : start_(std::chrono::steady_clock::now()) {}
    double Peek() const
    {
        return std::chrono::duration<double>(
                   std::chrono::steady_clock::now() - start_)
            .count();
    }

private:
    std::chrono::steady_clock::time_point start_;
};

// Generates the four datasets we'll crunch through.
std::vector<std::vector<int>> GenerateDataSets()
{
    std::vector<std::vector<int>> data_sets(k_num_data_sets);
    std::mt19937 rng{ 1337 }; // fixed seed for repeatable timing
    std::uniform_int_distribution<int> dist{ 0, 100 };

    for (auto& set : data_sets)
    {
        set.resize(k_data_set_size);
        for (auto& val : set)
        {
            val = dist(rng);
        }
    }
    return data_sets; // RVO -- no need for std::move
}

// Worker: sums up a span (a "chunk") of one dataset into an output slot.
void ProcessChunk(std::span<const int> chunk, long long& out_sum)
{
    long long total = 0;
    for (int v : chunk)
    {
        total += v;
    }
    out_sum = total;
}

// ---------- MODE 1: one giant batch ----------
void DoBiggie()
{
    auto data_sets = GenerateDataSets();
    Timer timer;

    std::vector<long long> partial_sums(k_num_data_sets, 0);
    {
        std::vector<std::jthread> workers;
        workers.reserve(k_num_data_sets);
        for (size_t i = 0; i < k_num_data_sets; ++i)
        {
            workers.emplace_back(
                ProcessChunk,
                std::span<const int>(data_sets[i]),
                std::ref(partial_sums[i]));
        }
        // jthreads auto-join when `workers` goes out of scope here.
    }

    long long grand_total = 0;
    for (auto s : partial_sums) grand_total += s;

    std::cout << "[biggie] time: " << timer.Peek()
              << "s  total: " << grand_total << "\n";
}

// ---------- MODE 2: many small batches ----------
// num_subsets controls how finely each dataset is sliced.
void DoSmallies(size_t num_subsets)
{
    auto data_sets = GenerateDataSets();
    Timer timer;

    const size_t subset_size = k_data_set_size / num_subsets;
    long long grand_total = 0;

    for (size_t sub = 0; sub < num_subsets; ++sub)
    {
        std::vector<long long> partial_sums(k_num_data_sets, 0);
        {
            std::vector<std::jthread> workers;
            workers.reserve(k_num_data_sets);
            for (size_t j = 0; j < k_num_data_sets; ++j)
            {
                const int* base = data_sets[j].data() + sub * subset_size;
                std::span<const int> chunk(base, subset_size);
                workers.emplace_back(
                    ProcessChunk, chunk, std::ref(partial_sums[j]));
            }
            // auto-join at end of this scope, every iteration
        }
        for (auto s : partial_sums) grand_total += s;
    }

    std::cout << "[smallies x" << num_subsets << "] time: " << timer.Peek()
              << "s  total: " << grand_total << "\n";
}

int main(int argc, char** argv)
{
    bool use_small = (argc > 1 && std::strcmp(argv[1], "--small") == 0);

    if (use_small)
    {
        // Try changing this to 1, 100, 1000, 10000 and compare timings,
        // just like in the video.
        DoSmallies(10'000);
    }
    else
    {
        DoBiggie();
    }

    return 0;
}
```

### Suggested experiments

1. Run `DoBiggie()` and note the baseline time.
2. Run `DoSmallies(n)` for `n = 1, 100, 1'000, 10'000` and record the times —
   you should reproduce the same slowdown pattern from the video (identical
   work, much slower as `n` grows).
3. Instrument `DoSmallies` to separately time:
   - thread creation + destruction per batch,
   - the actual `ProcessChunk` work,
   - the outer loop bookkeeping,

   to start narrowing down where the extra time is going.
4. Try reusing a fixed thread pool (created once, outside the loop) instead of
   creating new `jthread`s every batch, and compare timings — this is a
   strong candidate for "fixing" the slowdown and a good lead-in to the next
   video's answer.

---

## 7. Takeaways

- `std::span` is great for passing "a view into a chunk of data" to worker
  functions without copying — but you must construct it explicitly when
  passing through `std::thread`/`std::jthread` constructors.
- `std::jthread` auto-joins on destruction, which is very convenient when
  workers are created and destroyed repeatedly in a loop.
- Splitting identical total work into smaller, more frequent batches can
  significantly hurt throughput — even though the algorithmic work is
  unchanged. The likely culprits (to verify in practice) are per-batch thread
  creation/teardown overhead and repeated synchronization/join costs, rather
  than the trivial cost of the outer loop itself.
- Real-world systems (games, streaming pipelines) often *need* small, frequent
  batches for responsiveness — so this overhead is a real practical problem
  worth understanding and mitigating (e.g., thread pools) rather than an
  academic curiosity.
