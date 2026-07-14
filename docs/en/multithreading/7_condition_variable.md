# C++ Multithreading Notes — Part 2: Persistent Worker Threads & Condition Variables

Continuation of `cpp_multithreading_notes.md`. Covers the "big reveal" for why
batching into many small pieces was slow, then the design and full
implementation of a persistent worker-thread pool using `std::condition_variable`.

---

## 1. The Reveal: Why Smaller Batches Were Slower

Recap from Part 1: splitting the same total dataset into 10,000 pieces instead
of 1 made processing ~2.5× slower, even though the total work and thread
count per batch were identical.

**The answer: creating and destroying threads is expensive.**

- Creating a `std::thread`/`std::jthread` is a **heavyweight OS operation** —
  it calls into the operating system, which has to build a thread context and
  allocate a new thread stack (often ~1 MB).
- This is much more expensive than allocating something like a `std::vector`.
- In the 10,000-subset case, **thread creation/destruction overhead was
  measured as ~1.5× more expensive than the actual data processing itself.**
- Spawning `10,000 × 4` threads over the run (one fresh batch of 4 worker
  threads per subset) dominates the total time.

**The fix:** don't create a new batch of threads for every chunk of work.
Instead, create a fixed pool of worker threads **once**, keep them alive for
the whole program, and repeatedly **dispatch new jobs** to them, then wait for
them to signal completion. Reusing live threads amortizes the expensive
creation cost across all the work instead of paying it per batch.

**Trade-off:** this requires proper synchronization between a "master" thread
(dispatching jobs) and "worker" threads (executing jobs and signaling
completion) — more complex code, but much faster when you must process data
in small increments (e.g., streaming input, or per-frame game updates).

---

## 2. Synchronization Options

### Option A: Polling (naive, not used in the end)

- Shared job data + a flag, protected by a mutex.
- Worker sleeps for a fixed interval (e.g. 10 ms), wakes up, locks the mutex,
  checks the flag, does work if there's a job, sets a "done" flag, goes back
  to sleep.
- Master does the mirror image: writes job data + sets flag, sleeps, wakes,
  polls the "done" flag.

**Downsides:**
- Wasteful CPU/wake-ups from constantly polling.
- Adds latency: if the worker finishes right after a poll check, the result
  sits unused until the next poll interval (e.g., up to 10 ms wasted "with
  your thumb up your butt").

### Option B: Condition variables (`std::condition_variable`) — the good way

A condition variable lets one thread **signal** another rather than requiring
it to poll. Key facts:

- **Must always be used together with a mutex.** This is a hard requirement,
  not a suggestion.
- Threads call `cv.wait(lock, predicate)` to sleep until notified **and**
  the predicate returns true.
- Another thread modifies shared state (under the mutex), releases the mutex,
  then calls `cv.notify_one()` (wake one waiter) or `cv.notify_all()` (wake
  everyone waiting).
- **Sequence for a waiting thread:**
  1. Acquire the mutex (must be a `std::unique_lock`, not `std::lock_guard`,
     because it needs to be unlockable/relockable).
  2. Check the condition/predicate *before* sleeping (in case the event
     already happened — otherwise you'd sleep forever waiting for a
     notification that already fired).
  3. Call `wait(lock, predicate)`. Internally this atomically unlocks the
     mutex and sleeps; on wake, it atomically reacquires the mutex, then
     re-checks the predicate.
  4. While awake, you hold the mutex — until you either release it manually
     or go back to sleep.
- **Spurious wakeups are possible**: a thread can wake up without ever being
  notified. This is why you *always* re-check the condition/predicate after
  waking, even though logically the only reason you'd expect to wake up is a
  real notification.
- **Sequence for the notifying thread:**
  1. Acquire the mutex.
  2. Modify shared memory.
  3. Release the mutex.
  4. Call `notify_one()`/`notify_all()` — ideally *after* releasing the lock,
     otherwise the woken thread just immediately blocks on the mutex again
     (wasteful).

---

## 3. Design: Worker + MasterControl

### `Worker` (one per worker thread) encapsulates:

| Member | Purpose |
|---|---|
| `MasterControl* p_master` | back-pointer so the worker can signal "I'm done" |
| `std::jthread thread` | the actual OS thread, started at construction |
| `std::condition_variable cv` | used by the *master* to wake this worker |
| `std::mutex mtx` | always paired with the condition variable |
| `std::span<int> input` | the current job's data (shared memory) |
| `int* p_output` | where to write the result; `nullptr` means "no job" |
| `bool dying` | flag telling the worker to terminate |

Public interface (only these are called from outside):
- `SetJob(std::span<int> data, int* p_out)` — lock, write `input`/`p_output`,
  unlock, notify.
- `Kill()` — lock, set `dying = true`, unlock, notify.

Private:
- `Run()` — the function the `jthread` executes: loop forever, wait on the
  condition variable until `p_output != nullptr` or `dying`, then either
  break (if dying) or process the job and call back into `MasterControl` to
  signal completion.

### `MasterControl` (one, shared by all workers) encapsulates:

| Member | Purpose |
|---|---|
| `std::condition_variable cv` | used by *workers* to wake the master |
| `std::mutex mtx` | paired with the condition variable |
| `std::unique_lock<std::mutex> lock` | held by the main/master thread across waits |
| `size_t worker_count` | total number of workers (constant) |
| `size_t done_count` | shared counter, protected by `mtx` |

Public interface:
- `SignalDone()` — called by a worker when it finishes a job: lock, increment
  `done_count`, if `done_count == worker_count` then notify; unlock (via
  scope/lock_guard semantics).
- `WaitForAllDone()` — called by the main thread: `cv.wait(lock, predicate)`
  where predicate is `done_count == worker_count`; once woken, reset
  `done_count` to 0 for the next round.

### Why `std::unique_ptr<Worker>` in a `std::vector`?

Workers **must have a stable address** in memory, because each worker's
`jthread` captures `this` (a pointer to the worker) when it starts. If workers
were stored by value in a `std::vector<Worker>`, a reallocation (e.g. when the
vector grows) would move/copy them to new addresses, leaving the thread's
captured pointer **dangling**. Using `std::vector<std::unique_ptr<Worker>>`
guarantees each `Worker` lives at a fixed heap address for its whole lifetime.

---

## 4. Result

With the persistent worker pool (still splitting the dataset into the same
10,000 subsets), the time dropped from **~2.5 seconds back down to ~1.1
seconds** — matching the "do it all in one big batch" baseline. This confirms
the earlier hypothesis: the slowdown was purely thread creation/destruction
overhead, not the cost of processing in smaller pieces or the extra looping
logic.

---

## 5. Practice Code — Full Worker Pool Implementation

This reconstructs the design above as a compilable C++20 program. It reuses
`GenerateDataSets()` / `ProcessChunk()`-style helpers from Part 1 notes.

```cpp
// worker_pool_demo.cpp
// Compile (g++):  g++ -std=c++20 -O2 -pthread worker_pool_demo.cpp -o pool_demo
// Compile (MSVC): cl /std:c++20 /O2 /EHsc worker_pool_demo.cpp

#include <chrono>
#include <condition_variable>
#include <iostream>
#include <memory>
#include <mutex>
#include <random>
#include <span>
#include <thread>
#include <vector>

constexpr size_t k_data_set_size = 50'000'000;
constexpr size_t k_num_data_sets = 4;      // == worker_count
constexpr size_t k_num_subsets = 10'000;   // how finely we slice each dataset

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

std::vector<std::vector<int>> GenerateDataSets()
{
    std::vector<std::vector<int>> data_sets(k_num_data_sets);
    std::mt19937 rng{ 1337 };
    std::uniform_int_distribution<int> dist{ 0, 100 };
    for (auto& set : data_sets)
    {
        set.resize(k_data_set_size);
        for (auto& v : set) v = dist(rng);
    }
    return data_sets;
}

void ProcessChunk(std::span<const int> chunk, int& out_sum)
{
    long long total = 0;
    for (int v : chunk) total += v;
    out_sum = static_cast<int>(total); // demo only; watch for overflow IRL
}

// ---------------- MasterControl ----------------
class MasterControl
{
public:
    explicit MasterControl(size_t worker_count)
        : lock_(mtx_), worker_count_(worker_count)
    {
        // lock_ starts by owning mtx_ (locked), mirroring the video's design.
    }

    // Called by a worker when it finishes its job.
    void SignalDone()
    {
        std::lock_guard<std::mutex> lg(mtx_);
        ++done_count_;
        if (done_count_ == worker_count_)
        {
            cv_.notify_one();
        }
    }

    // Called by the master/main thread to block until all workers are done.
    void WaitForAllDone()
    {
        cv_.wait(lock_, [this] { return done_count_ == worker_count_; });
        done_count_ = 0; // reset for next round
    }

private:
    std::condition_variable cv_;
    std::mutex mtx_;
    std::unique_lock<std::mutex> lock_;
    size_t worker_count_;
    size_t done_count_ = 0;
};

// ---------------- Worker ----------------
class Worker
{
public:
    explicit Worker(MasterControl* p_master)
        : p_master_(p_master), thread_(&Worker::Run_, this)
    {
    }

    // Assign a new job. Called by the main/master thread.
    void SetJob(std::span<int> data, int* p_out)
    {
        {
            std::lock_guard<std::mutex> lg(mtx_);
            input_ = data;
            p_output_ = p_out;
        }
        cv_.notify_one();
    }

    // Tell the worker to terminate after any current job.
    void Kill()
    {
        {
            std::lock_guard<std::mutex> lg(mtx_);
            dying_ = true;
        }
        cv_.notify_one();
    }

private:
    void Run_()
    {
        std::unique_lock<std::mutex> lock(mtx_);
        while (true)
        {
            cv_.wait(lock, [this] { return p_output_ != nullptr || dying_; });

            if (dying_)
            {
                break;
            }

            // We hold mtx_ here; process the job.
            ProcessChunk(std::span<const int>(input_), *p_output_);

            // Clear job markers before signaling completion.
            p_output_ = nullptr;
            input_ = {};

            p_master_->SignalDone();
        }
    }

    MasterControl* p_master_;
    std::condition_variable cv_;
    std::mutex mtx_;
    std::span<int> input_;
    int* p_output_ = nullptr;
    bool dying_ = false;
    std::jthread thread_; // declared last so it's constructed last (after cv_/mtx_ exist)
};

int main()
{
    constexpr size_t worker_count = k_num_data_sets;

    auto data_sets = GenerateDataSets();
    Timer timer;

    MasterControl mctrl(worker_count);

    std::vector<std::unique_ptr<Worker>> workers;
    workers.reserve(worker_count);
    for (size_t i = 0; i < worker_count; ++i)
    {
        workers.push_back(std::make_unique<Worker>(&mctrl));
    }

    const size_t subset_size = k_data_set_size / k_num_subsets;
    std::vector<int> partial_sums(worker_count, 0);
    long long grand_total = 0;

    for (size_t sub = 0; sub < k_num_subsets; ++sub)
    {
        for (size_t j = 0; j < worker_count; ++j)
        {
            int* base = data_sets[j].data() + sub * subset_size;
            std::span<int> chunk(base, subset_size);
            workers[j]->SetJob(chunk, &partial_sums[j]);
        }
        mctrl.WaitForAllDone();

        for (int s : partial_sums) grand_total += s;
    }

    for (auto& w : workers) w->Kill();
    workers.clear(); // destructors join the jthreads

    std::cout << "[worker pool] time: " << timer.Peek()
              << "s  total: " << grand_total << "\n";

    return 0;
}
```

### Things to try

1. Run this and compare its time against the Part 1 `DoSmallies(10'000)`
   version — you should see it drop back close to the ~1.1s baseline.
2. Sweep `k_num_subsets` (1, 100, 1,000, 10,000, 100,000) with this pool-based
   version and confirm the time stays roughly flat, unlike the naive
   spawn-per-batch version.
3. Deliberately remove the predicate re-check (simulate not handling spurious
   wakeups) and reason about what could go wrong — you likely won't observe a
   bug on your platform, but it's undefined-by-spec behavior to skip it.
4. Extend `MasterControl`/`Worker` to report per-worker timing to identify
   any load imbalance across workers.
5. Try `notify_all()` instead of `notify_one()` in `SignalDone()` where it
   doesn't matter (only master waits there) vs. where it would matter (if
   multiple threads could be waiting on the same condition variable).

---

## 6. Common Bug Called Out in the Video

The original code accumulated a "grand total" incorrectly — it forgot to
reset per-worker partial sums to zero each iteration and forgot to actually
sum into a running "grand total" variable across iterations. It still
produced the *correct final number* by coincidence: since each partial-sum
slot kept accumulating across iterations without being cleared, and the
"grand total" was recomputed as a full sum of all slots on the *last*
iteration, the intermediate accumulation happened to land on the right value
anyway.

**Lesson:** getting the "right answer" doesn't always confirm your code does
what you *intended* — always double check that data is being reset/cleared
where you expect, especially in loops with shared/reused buffers.

---

## 7. Key Takeaways

- Thread creation/destruction is a heavyweight OS-level operation — expensive
  enough that creating tens of thousands of threads can dominate your
  program's runtime.
- If you must process data in small increments repeatedly, **create your
  worker threads once and reuse them**, dispatching new jobs via
  synchronization instead of spawning/joining new threads every round.
- `std::condition_variable` requires a paired mutex, must be waited on with a
  predicate (to survive spurious wakeups and to avoid missed
  wake-ups/deadlock), and is far more efficient than manual polling.
- Encapsulating the synchronization details inside `Worker`/`MasterControl`
  classes keeps the calling code (`SetJob`, `Kill`, `WaitForAllDone`) simple
  and safe, even though the internals are intricate.
- Store polymorphic/address-sensitive objects (like workers whose threads
  capture `this`) behind `std::unique_ptr` inside containers, so container
  reallocation never invalidates captured pointers.
