# Parallelism

---

## Core Objective

Parallelism exists to solve one problem: **time**. A single CPU core, no matter how fast, can only execute one instruction per clock cycle. As problem sizes grow — petabytes of data to process, millions of pixels to render, trillions of model parameters to train — sequential execution becomes a fundamental physical bottleneck. Parallelism is the discipline of decomposing a problem so that independent portions execute **simultaneously** on multiple hardware units, reducing wall-clock time proportionally to the amount of independent work available.

For a software architect, mastering parallelism means understanding not just threading APIs, but the entire stack: how hardware executes instructions simultaneously, how the operating system manages multi-core resources, what synchronization costs, how to decompose problems correctly, and — critically — where parallelism provides diminishing returns or introduces correctness hazards that cost more than the performance gained.

The core promise of parallelism: **if a task takes T seconds on 1 core and the task is perfectly decomposable, it takes T/N seconds on N cores.** The core reality: no real task is perfectly decomposable. Understanding the gap between that promise and that reality is the entire discipline.

---

## Fundamental Concepts

### What Is Parallelism?

**Parallelism** is a runtime execution property. Multiple computations physically execute at the **exact same instant** on separate hardware execution units (cores, processors, GPUs, nodes). This is not the appearance of simultaneity — it is genuine, physical, simultaneous execution.

This distinguishes parallelism from **concurrency**, which is a design property — structuring a program to manage multiple tasks that may or may not run simultaneously. Concurrency can exist on a single core (via interleaving). Parallelism requires 2+ hardware execution units.

> A program can be concurrent without being parallel (single-core event loop).
> A program can be parallel without being concurrent (SIMD — same operation on all data).
> Most real systems are both.

---

### Types of Parallelism

#### 1. Data Parallelism
The **same operation** is applied to **different subsets of data** simultaneously. The data is partitioned, each partition assigned to a separate execution unit.

- Example: Applying a blur filter to a 4K image. The image is split into 8 horizontal bands. Each of 8 CPU cores processes its band independently.
- Example: Training a neural network with mini-batch data parallelism — each GPU receives a different subset of the training batch.
- Languages/tools: SIMD intrinsics, OpenMP `#pragma omp parallel for`, CUDA, NumPy vectorized operations, Java Streams `.parallelStream()`

```
Dataset: [A, B, C, D, E, F, G, H]
              ↓ split
Core 0: [A, B]  →  result[0,1]
Core 1: [C, D]  →  result[2,3]
Core 2: [E, F]  →  result[4,5]
Core 3: [G, H]  →  result[6,7]
              ↓ merge
Output: [r0, r1, r2, r3, r4, r5, r6, r7]
```

#### 2. Task Parallelism
**Different tasks** (functions, computations) execute simultaneously on different data or independently. Tasks are heterogeneous — each core may be doing something different.

- Example: A compiler pipeline — lexer, parser, and semantic analyzer can process different files simultaneously.
- Example: A game engine — physics simulation, audio processing, and AI pathfinding run on separate threads simultaneously.
- Tools: `std::thread`, Java `ForkJoinPool`, Go goroutines with worker pools, OpenMP `#pragma omp sections`

```
Core 0: [Task A: Physics simulation  ]
Core 1: [Task B: Audio processing    ]
Core 2: [Task C: AI pathfinding      ]
Core 3: [Task D: Network I/O         ]
         All executing simultaneously
```

#### 3. Pipeline Parallelism
Work is divided into **stages** (like an assembly line). Each stage processes a different item simultaneously. When Stage 1 finishes item 1 and passes it to Stage 2, Stage 1 immediately starts item 2.

- Example: Video encoding — decoding, color-space conversion, DCT compression, entropy coding as sequential stages. While Stage 3 compresses frame N, Stage 2 converts frame N+1, Stage 1 decodes frame N+2.
- Throughput: limited by the slowest stage (the bottleneck stage). Latency: sum of all stage times. Throughput: 1/max(stage time) items per second once the pipeline is full.

```
Time →   T1        T2        T3        T4        T5
Stage 1: [Frame 1] [Frame 2] [Frame 3] [Frame 4] [Frame 5]
Stage 2:           [Frame 1] [Frame 2] [Frame 3] [Frame 4]
Stage 3:                     [Frame 1] [Frame 2] [Frame 3]
```

#### 4. Instruction-Level Parallelism (ILP)
The CPU itself executes **multiple instructions from a single thread simultaneously** using out-of-order execution, superscalar pipelines, and speculative execution. This is invisible to the programmer — the hardware manages it automatically.

- **Superscalar execution**: A modern CPU has multiple execution units (ALUs, FPUs, load/store units). If instructions are independent, the CPU issues multiple instructions per clock cycle.
- **Out-of-order execution (OOO)**: The CPU reorders instructions to avoid stalls. If instruction 5 depends on instruction 4 (which is waiting for memory), the CPU executes instruction 6 first if it's independent.
- **Branch prediction**: The CPU predicts if/else outcomes and speculatively executes the predicted path to avoid pipeline stalls. Misprediction cost: ~15-20 cycles flushed.
- **Register renaming**: Eliminates false data hazards (WAR, WAW dependencies) by mapping architectural registers to a larger pool of physical registers.

Architectural implication: Write code with **predictable branches** and **independent instructions** to maximize ILP. Tight loops with data dependencies serialize through the CPU pipeline.

#### 5. Bit-Level Parallelism
Processing wider data in a single operation. A 64-bit CPU processes 64 bits per operation vs. a 32-bit CPU processing 32. SIMD extends this further: a 256-bit AVX2 register processes 8 × 32-bit floats in a single instruction.

| CPU Word Width | Integers per SIMD op (32-bit) | Floats per SIMD op |
|---|---|---|
| SSE2 (128-bit) | 4 | 4 |
| AVX2 (256-bit) | 8 | 8 |
| AVX-512 (512-bit) | 16 | 16 |

---

### Flynn's Taxonomy

A classification of computer architectures by instruction and data stream multiplicity. Every parallel system fits one category.

| Class | Full Name | Description | Example |
|---|---|---|---|
| **SISD** | Single Instruction, Single Data | Classic sequential CPU: one instruction, one data element | Old single-core CPUs |
| **SIMD** | Single Instruction, Multiple Data | One instruction operates on multiple data elements simultaneously | GPU shaders, AVX/SSE CPU, Intel AVX-512 |
| **MISD** | Multiple Instruction, Single Data | Multiple processors execute different instructions on the same data | Fault-tolerant systems, space shuttle flight control (redundant computation) |
| **MIMD** | Multiple Instruction, Multiple Data | Multiple processors run different instructions on different data independently | Multi-core CPUs, distributed clusters, cloud computing |

Modern CPUs are **MIMD** at the core level (each core runs its own instruction stream) and **SIMD** within each core (AVX/SSE units).

---

### Hardware Foundations

#### CPU Cores and the Memory Hierarchy

```
CPU DIE
┌─────────────────────────────────────────────────────────┐
│  Core 0          Core 1          Core 2          Core 3  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────┐  │
│  │ L1I 32KB │   │ L1I 32KB │   │ L1I 32KB │   │  ..  │  │
│  │ L1D 32KB │   │ L1D 32KB │   │ L1D 32KB │   │      │  │
│  │ L2 256KB │   │ L2 256KB │   │ L2 256KB │   │      │  │
│  └──────────┘   └──────────┘   └──────────┘   └──────┘  │
│                  L3 Cache (shared) 12MB                   │
└─────────────────────────────────────────────────────────┘
                        │
                   RAM (DDR5)
                  64-128+ GB
```

**Cache hierarchy latency** (approximate):
| Level | Latency | Size (typical) |
|---|---|---|
| L1 Cache | ~4 cycles | 32–64 KB per core |
| L2 Cache | ~12 cycles | 256 KB – 1 MB per core |
| L3 Cache | ~40 cycles | 8–96 MB shared |
| RAM (DRAM) | ~200 cycles | 16–512 GB |
| NVMe SSD | ~100,000 cycles | TB |

**Implication for parallelism**: Threads competing for the same L3 cache evict each other's working data. A parallel program that scales from 1→4 cores but causes cache thrashing may be slower than the single-threaded version.

#### Hyper-Threading / SMT (Simultaneous Multithreading)

A single physical core presents **2 logical cores** to the OS. The two logical cores share execution units, L1/L2 caches, and the TLB, but have separate register files and program counters. When one logical core stalls (waiting for memory), the other uses the idle execution units.

- **Benefit**: Better CPU utilization when threads stall on memory — throughput improves 20–30% for memory-bound workloads.
- **Risk**: For compute-bound workloads, two SMT threads compete for the same execution units, often providing zero benefit or slight regression.
- **Security implication**: SMT creates side-channel attack surface (Spectre, MDS attacks) because shared caches leak timing information between logical cores.

#### NUMA (Non-Uniform Memory Access)

In servers with multiple physical CPU sockets, each socket has RAM physically closer to it. Accessing "local" RAM (same socket) is fast (~4 cycles to L3, ~80ns to local DIMM). Accessing "remote" RAM (different socket) crosses the **interconnect** (Intel UPI, AMD Infinity Fabric) and costs 2–3× more.

```
Socket 0                      Socket 1
┌──────────────────┐          ┌──────────────────┐
│ Core 0-15        │◄────────►│ Core 16-31       │
│ Local RAM: 64GB  │  UPI/IF  │ Local RAM: 64GB  │
└──────────────────┘          └──────────────────┘
```

**NUMA-aware programming**: A thread on Core 0 accessing data allocated on Socket 1's RAM pays 2-3× latency. High-performance parallel programs use `numa_alloc_local()`, `mbind()`, and `numactl --cpunodebind=0 --membind=0` to keep data local to the socket running the computation.

---

### Processes vs Threads

| Property | Process | Thread |
|---|---|---|
| **Memory space** | Separate (isolated) virtual address space | Shared virtual address space within process |
| **Creation cost** | High (~1ms, OS must copy page tables, file descriptors) | Low (~10µs, no memory duplication) |
| **Communication** | IPC: pipes, sockets, shared memory, message queues | Direct shared memory (but requires synchronization) |
| **Crash isolation** | Yes — one process crash doesn't affect others | No — one thread crash (segfault) kills entire process |
| **Parallelism** | Yes — OS schedules processes on different cores | Yes — OS schedules threads on different cores |
| **Context switch cost** | Higher (TLB flush, new address space) | Lower (same address space) |
| **Use case** | Security isolation, fault isolation, multi-language | Low-overhead parallelism within one application |

**When to use processes for parallelism**: Browser architecture (Chrome uses process-per-tab for isolation), Python multiprocessing (bypass GIL), Nginx (worker processes).

**When to use threads for parallelism**: Java ForkJoinPool, C++ thread pools, game engine worker threads.

---

### Speedup, Efficiency, and Scalability

| Metric | Formula | Meaning |
|---|---|---|
| **Speedup (S)** | T₁ / Tₙ | How many times faster with N cores vs 1 core |
| **Efficiency (E)** | S / N | Fraction of cores doing useful work (1.0 = 100% efficient) |
| **Scalability** | How S changes as N increases | Does doubling cores double speed? |

**Superlinear speedup** (S > N): Sometimes observed when parallelism allows the working dataset to fit into the combined L2/L3 caches of multiple cores, reducing memory bottlenecks. Not a violation of physics — it's a cache effect.

---

## Procedure or Logic

### Problem Decomposition for Parallelism

Before writing a single line of parallel code, a software architect must classify the problem:

**Step 1 — Identify independent work units**
Ask: "Which parts of this computation produce results that do not depend on the results of other parts?" Only independent work can be parallelized. A loop where iteration N+1 depends on the result of iteration N is a serial dependency chain — it cannot be parallelized.

```
PARALLELIZABLE:
for i in 0..N:
    result[i] = expensive_compute(input[i])   # each i is independent

NOT PARALLELIZABLE:
for i in 1..N:
    result[i] = result[i-1] + input[i]        # each i depends on previous
```

**Step 2 — Measure the sequential fraction**
Profile the program. Identify what percentage of execution time is in code that **cannot** be parallelized (global initialization, sequential aggregation, lock-protected sections). Apply Amdahl's Law (see Critical Considerations) to determine the theoretical maximum speedup. If 40% of the program is sequential, adding infinite cores yields at most 2.5× speedup.

**Step 3 — Choose a decomposition strategy**
- **Data decomposition**: Partition the dataset. Assign partitions to workers. Best for data-parallel problems (matrix ops, image processing, batch ML inference).
- **Task decomposition**: Identify independent tasks. Assign tasks to workers. Best for heterogeneous work (game engine systems, compiler phases).
- **Pipeline decomposition**: Identify sequential stages. Chain stages. Best for streaming workloads (video processing, ETL pipelines).
- **Recursive decomposition**: Divide problem in half recursively, solve halves in parallel, combine results. Best for divide-and-conquer algorithms (parallel merge sort, parallel quicksort, tree traversals).

**Step 4 — Choose granularity**
Granularity = size of each parallel work unit. Too fine: thread creation/scheduling overhead dominates. Too coarse: cores sit idle waiting for one slow task. Target granularity should produce tasks that take at least **10–100× the thread synchronization cost** (~1–10µs).

**Step 5 — Handle shared state**
List all data written by parallel tasks. Any shared mutable data requires synchronization. Design choices in priority order:
1. **Eliminate sharing**: give each worker its own copy, merge results at end.
2. **Use atomics**: for simple counters, flags — lock-free and fast.
3. **Use fine-grained locks**: protect only the minimum critical section.
4. **Use coarse locks as last resort**: simple but limits parallelism.

---

### Fork-Join Execution Model

The foundational pattern for task parallelism. A parent task **forks** into N child tasks that execute in parallel, then **joins** — waits for all children to complete before continuing.

```
Main Thread
     │
     ├─── fork ──────────────────────────────────────────
     │         │              │              │
     │      [Worker 0]    [Worker 1]     [Worker 2]
     │      process       process        process
     │      partition 0   partition 1   partition 2
     │         │              │              │
     └─── join ──────────────────────────────────────────
     │
  [Aggregate results]
     │
  [Continue]
```

Java's `ForkJoinPool` implements this with **work stealing** to handle uneven task sizes. The `RecursiveTask<V>` and `RecursiveAction` classes structure fork-join computation.

---

### Work Stealing Algorithm

**Problem**: In a fork-join pool with N threads, tasks are not always equal in duration. If thread 0 finishes its 10 tasks in 1s but thread 1 is still processing 1 large task taking 5s, threads 0–N-1 sit idle.

**Work stealing solution**:
- Each thread has its own **deque** (double-ended queue) of tasks.
- A thread pushes new tasks to the **front** (bottom) of its own deque.
- A thread pops work from the **front** of its own deque.
- When a thread's deque is **empty**, it **steals** a task from the **back** (top) of another randomly chosen thread's deque.
- Front/back separation minimizes contention: the owning thread and the stealing thread access opposite ends.

**Why it works**: Large tasks subdivided recursively tend to sit at the top of the deque — the coarser-grained work is stolen first, reducing steal frequency. Used by: Java `ForkJoinPool`, Intel TBB, Go's runtime scheduler, .NET ThreadPool.

---

### MapReduce Pattern

A parallel data processing pattern for large-scale data. Consists of three phases:

**Map phase**: Each worker applies a function to its partition, producing intermediate key-value pairs. All map tasks run in parallel.

**Shuffle phase**: Intermediate key-value pairs are grouped by key, redistributed to reducers. This is the network/I/O intensive phase in distributed systems.

**Reduce phase**: Each reducer aggregates all values for its assigned keys, producing final output. All reduce tasks run in parallel.

```
Input: [doc1, doc2, doc3, doc4, doc5, doc6]
           ↓ partition
Map:  [doc1,doc2] [doc3,doc4] [doc5,doc6]
        ↓              ↓            ↓
      {k:v, k:v}  {k:v, k:v}  {k:v, k:v}
                ↓ shuffle by key
Reduce: [all values for key A] → sum(A)
        [all values for key B] → sum(B)
```

Used by: Hadoop MapReduce, Apache Spark (generalized), Google's original MapReduce paper (2004).

---

### Pipeline Parallelism in Practice

**Throughput calculation**: If a pipeline has stages with durations [T₁, T₂, T₃, T₄], the pipeline throughput is `1 / max(T₁, T₂, T₃, T₄)` items per second once the pipeline is **full**. The slowest stage is the **bottleneck**. To improve throughput, either speed up the bottleneck or **replicate** the bottleneck stage with multiple parallel workers.

**Buffer sizing**: Between stages, a buffer (queue) holds items. Buffer too small → fast upstream stage blocks waiting for slow downstream. Buffer too large → unbounded memory growth if upstream is consistently faster. Target buffer size: `upstream_rate × expected_stage_latency`.

---

### Synchronization Primitives

#### Mutex (Mutual Exclusion Lock)
A binary lock that ensures only one thread executes the **critical section** at a time. When Thread A holds the mutex, Thread B attempting to acquire it is **blocked** (put to sleep by the OS) until Thread A releases it.

```
Thread A:  acquire(mutex) → [critical section] → release(mutex)
Thread B:  acquire(mutex) → BLOCKED → ... → [woken up] → [critical section] → release(mutex)
```

- **Blocking cost**: OS context switch on contention (~1–10µs). Appropriate when critical section holds for > ~1µs.
- **Ownership**: A mutex is owned by the thread that acquired it. Only the owner can release it.
- **Priority inversion risk**: Low-priority thread holds mutex, high-priority thread blocks on it, medium-priority threads run instead — high-priority thread starves.

#### Spinlock
A lock where the waiting thread **busy-waits** in a loop checking if the lock is available, instead of sleeping.

```c
while (atomic_test_and_set(&lock)) { /* spin */ }
// critical section
atomic_clear(&lock);
```

- **When better than mutex**: Critical section is **very short** (< ~200ns, a few instructions). If Thread B would sleep and wake up in the same time it spins, spinning is faster (avoids OS context switch).
- **When worse than mutex**: Critical section is long. Spinning wastes CPU cycles doing no work.
- **Used in**: OS kernel code (interrupt handlers), lock implementations themselves.

#### Read-Write Lock (RWLock)
Allows multiple concurrent readers OR one exclusive writer. Readers never block other readers; a writer blocks all readers and other writers.

```
Readers: [R1 reading] [R2 reading] [R3 reading]   ← all concurrent
Writer:  [          blocked          ] → [W writing] → [R1, R2, R3 resume]
```

- **Use case**: Frequently-read, rarely-written data structures (routing tables, configuration, caches).
- **Writer starvation risk**: In a read-heavy system, a constant stream of new readers can starve a waiting writer indefinitely. Solutions: writer-preferred RWLocks, writer queuing.

#### Semaphore
An integer counter with two atomic operations:
- **P (wait/down)**: Decrement counter. If counter would go negative, block until another thread increments it.
- **V (signal/up)**: Increment counter. If threads are blocked, wake one.

```
Counting semaphore(3):  Limits concurrent access to a resource pool of 3
Binary semaphore(1):    Equivalent to a mutex (but without ownership semantics)
```

- **Key difference from mutex**: A semaphore has no ownership — any thread can signal it. Used for signaling between threads (producer signals consumer) rather than mutual exclusion.
- **Use case**: Connection pool limiting (max 10 DB connections), throttling parallel tasks, bounded buffer in producer-consumer.

#### Barrier
A synchronization point where all participating threads must arrive before any of them continue. Used in iterative parallel algorithms where each iteration depends on the complete results of the previous.

```
Core 0: [phase 1 work] ──────────────────── BARRIER ──► [phase 2 work]
Core 1: [phase 1 work] ─────── BARRIER ───────────────► [phase 2 work]
Core 2: [phase 1 work] ─────────────────── BARRIER ────► [phase 2 work]
                                                ↑
                              All must arrive before any proceed
```

#### Atomic Operations and Memory Ordering

**Atomic operations** execute as a single indivisible hardware instruction, requiring no lock. The CPU provides hardware support (LOCK prefix on x86, exclusive monitor on ARM).

- **CAS (Compare-And-Swap)**: `if (current == expected) { current = new_val; return true; } else { return false; }` — the foundation of all lock-free algorithms.
- **Fetch-And-Add**: Atomically increment a value and return the old value.
- **Test-And-Set**: Set a bit and return its old value — the basis of spinlocks.

**Memory ordering** — why it matters: Modern CPUs and compilers **reorder instructions** to optimize performance (out-of-order execution, store buffers). A write on Core 0 may not be visible to Core 1 without a **memory barrier**.

| Memory Order | Meaning |
|---|---|
| `relaxed` | No ordering guarantees. Only atomicity of the single operation. Fastest. Use for counters where order doesn't matter. |
| `acquire` | All memory operations **after** this load (in program order) happen after the load. Used when reading a flag that guards data. |
| `release` | All memory operations **before** this store (in program order) happen before the store. Used when writing data before setting a flag. |
| `acq_rel` | Both acquire and release. Used for read-modify-write operations (CAS, fetch-add). |
| `seq_cst` | Total global ordering of all seq_cst operations. Slowest. Default for `std::atomic`. Use when in doubt. |

#### Cache Coherence: MESI Protocol

Every cache line in a multicore system is tracked with one of four states:

| State | Meaning | Can Read? | Can Write? |
|---|---|---|---|
| **M (Modified)** | This core has the only copy, it has been written (dirty) | Yes | Yes |
| **E (Exclusive)** | This core has the only copy, it is clean (same as RAM) | Yes | Yes (transitions to M) |
| **S (Shared)** | Multiple cores have clean copies | Yes | No (must invalidate others first) |
| **I (Invalid)** | This core's copy is stale — must fetch from another cache or RAM | No | No |

**False sharing**: Two threads on different cores each modify **different variables** that happen to reside on the **same 64-byte cache line**. Core 0's write invalidates Core 1's cache line (even though Core 1 doesn't use Core 0's variable). Core 1 must reload the entire cache line. This creates invisible serialization that can make parallel code slower than sequential.

```c
// BAD: counter_a and counter_b on same cache line
struct { int counter_a; int counter_b; } shared;

// GOOD: pad to separate cache lines
struct { int counter_a; char pad[60]; int counter_b; } shared;
```

---

### Race Conditions

#### Definition
A **race condition** is a defect where the program output depends on the non-deterministic timing and interleaving of operations. The canonical example:

```
Thread A:  read counter (=5)
                              ← context switch here
Thread B:  read counter (=5), increment to 6, write counter (=6)
                              ← context switch back
Thread A:  increment 5 to 6, write counter (=6)   ← Thread B's write lost!

Expected: counter = 7
Actual:   counter = 6
```

#### Data Race vs Race Condition
These are distinct concepts that are frequently conflated:

| Term | Definition | Example |
|---|---|---|
| **Data race** | Concurrent unsynchronized access to the same memory location, where at least one access is a write. **Undefined behavior** in C/C++. | Two threads write to the same `int` without synchronization |
| **Race condition** | A logical bug caused by incorrect ordering assumptions. Can exist **even with proper synchronization**. | Check-then-act: checking if a file exists, then creating it — file could be created by another process between check and create |

All data races are a form of race condition, but not all race conditions involve data races. A correctly synchronized program can still have race conditions if the synchronization is at the wrong logical level.

#### Prevention
1. **Synchronization** (mutexes, atomics): Serialize access to shared mutable state.
2. **Immutability**: Immutable data can be safely read by unlimited concurrent threads with zero synchronization cost.
3. **Thread-local storage**: Give each thread its own copy. `thread_local` (C++11), `ThreadLocal<T>` (Java), goroutine-local patterns.
4. **Message passing**: Eliminate shared mutable state — tasks communicate by passing data, not by sharing it.
5. **Functional purity**: Pure functions with no side effects are trivially thread-safe.

#### Detection Tools
| Tool | Platform | Method | What It Detects |
|---|---|---|---|
| **ThreadSanitizer (TSan)** | Linux/macOS | Compile-time instrumentation | Data races at runtime |
| **Helgrind** | Linux/macOS | Valgrind plugin | Lock order violations, data races |
| **Intel Inspector** | Linux/Windows | Binary instrumentation | Data races, deadlocks |
| **Go race detector** | All | `go run -race` | Data races in Go programs |
| **Visual Studio Concurrency Visualizer** | Windows | ETW tracing | Thread contention, blocking |

---

### Deadlocks

A **deadlock** is a state where two or more threads are permanently blocked, each waiting for a resource held by the other. The system makes no further progress.

**Four Coffman Conditions** — all four must hold simultaneously for deadlock to occur:
1. **Mutual exclusion**: Resources cannot be shared (only one thread at a time).
2. **Hold and wait**: A thread holds at least one resource while waiting for another.
3. **No preemption**: Resources cannot be forcibly taken from a thread.
4. **Circular wait**: A cycle exists in the resource-waiting graph (A waits for B, B waits for A).

**Prevention** (eliminate at least one condition):
- **Lock ordering**: Always acquire multiple locks in a globally consistent order (e.g., by memory address). Eliminates circular wait.
- **Try-lock with timeout**: `try_lock()` — if acquisition fails, release all held locks and retry. Eliminates hold-and-wait.
- **Lock-free design**: Eliminate mutual exclusion entirely using atomics.

**Livelock**: Two threads continuously change state in response to each other but make no progress. Like two people in a hallway both stepping aside in the same direction. Solution: randomized backoff.

**Starvation**: A thread is perpetually denied CPU time because higher-priority or more-frequent threads always win scheduling decisions.

---

## Operating Systems and Parallelism

### Linux

Linux is the dominant OS for parallel computing — 100% of the Top500 supercomputers run Linux. It provides the most complete and tunable parallelism infrastructure of any general-purpose OS.

**SMP (Symmetric Multiprocessing)**: Linux has full SMP support since kernel 2.0 (1996). All cores share a single memory space visible to the kernel. The kernel itself is parallelized with fine-grained locking (Big Kernel Lock removed in 2.6, replaced with per-subsystem locks).

**NUMA support**: `numactl`, `libnuma`, `mbind()`, `set_mempolicy()` system calls enable NUMA-aware memory allocation. The kernel's NUMA balancer (`numa_balancing`) migrates memory pages toward the CPU accessing them most frequently.

```bash
# Run program on NUMA node 0, using only node 0's memory
numactl --cpunodebind=0 --membind=0 ./myprogram

# Show NUMA memory statistics
numastat -p myprogram
```

**CPU Affinity**: Pinning threads to specific cores prevents cache-invalidating migrations.
```bash
# Pin process to cores 0-3
taskset -c 0-3 ./myprogram

# Set affinity programmatically
sched_setaffinity(pid, sizeof(cpu_set_t), &cpu_set);
```

**Scheduling Policies for Parallel Workloads**:
| Policy | Description | Use Case |
|---|---|---|
| `SCHED_OTHER` (CFS) | Default fair scheduling | General parallel workloads |
| `SCHED_FIFO` | Real-time, run until blocked or yields, highest priority | Hard real-time parallel tasks |
| `SCHED_RR` | Real-time, round-robin with time slice | Soft real-time parallel tasks |
| `SCHED_DEADLINE` | EDF scheduling with deadline, runtime, period | Tasks with strict timing requirements |
| `SCHED_BATCH` | Lower priority, no preemption penalty | Background batch parallel jobs |

**CFS (Completely Fair Scheduler)** for multi-core: Maintains per-core run queues. Performs **load balancing** — migrating threads from overloaded cores to idle ones — using the topology-aware scheduler domains (cores → NUMA nodes → sockets).

**cgroups cpuset**: Isolate parallel applications to specific cores, preventing interference from other workloads.
```bash
# Create a cgroup with cores 4-7 for a parallel job
cgcreate -g cpuset:/parallel_job
echo "4-7" > /sys/fs/cgroup/cpuset/parallel_job/cpuset.cpus
echo "0"   > /sys/fs/cgroup/cpuset/parallel_job/cpuset.mems
cgexec -g cpuset:/parallel_job ./parallel_program
```

**Huge Pages for Parallel Workloads**: Large parallel datasets benefit from 2MB huge pages (vs default 4KB). Huge pages reduce TLB pressure — with 4KB pages, a thread accessing 1GB of data needs 262,144 TLB entries. With 2MB huge pages, it needs only 512.

**Profiling Parallel Programs on Linux**:
```bash
perf stat -e cycles,instructions,cache-misses,LLC-load-misses ./myprogram
perf record -g ./myprogram && perf report
htop                    # visual per-core CPU utilization
numastat                # NUMA hit/miss rates
```

**Parallel I/O with io_uring**: Linux 5.1+ provides `io_uring` — a high-performance async I/O interface using shared ring buffers between kernel and userspace. Eliminates system call overhead per I/O operation, enabling truly parallel I/O across many file descriptors.

---

### Windows

**Thread Pool API (Vista+)**: Windows provides `CreateThreadpoolWork`, `SubmitThreadpoolWork` — a structured thread pool backed by the I/O Completion Port infrastructure. Automatically sizes the pool to available cores.

**I/O Completion Ports (IOCP)**: Windows's mechanism for parallel async I/O. Threads in the pool block on `GetQueuedCompletionStatus`. When an async I/O completes, the kernel posts a completion packet to the IOCP queue. Idle threads wake to process completions.

**SetThreadAffinityMask**: Pins a thread to specific logical processors.
```c
SetThreadAffinityMask(GetCurrentThread(), 0b00001111); // Cores 0-3
```

**Processor Groups** (Windows 7+): Systems with more than 64 logical processors are organized into processor groups. Thread affinity APIs must specify which group. The `SetThreadGroupAffinity` function assigns threads to specific processor groups.

**.NET Task Parallel Library (TPL)**: `Parallel.For`, `Parallel.ForEach`, and PLINQ (Parallel LINQ) implement data and task parallelism on top of the `ThreadPool`. `Parallel.ForEach` handles load balancing internally using range partitioning and chunk stealing.

```csharp
// Data parallelism with automatic partitioning
Parallel.For(0, N, i => {
    result[i] = ExpensiveCompute(input[i]);
});

// PLINQ - parallel LINQ query
var results = data.AsParallel()
                  .WithDegreeOfParallelism(Environment.ProcessorCount)
                  .Select(x => Transform(x))
                  .ToArray();
```

**Visual Studio Concurrency Visualizer**: Timeline view showing thread execution, blocking, synchronization events — essential for diagnosing parallel performance on Windows.

---

### macOS

**Grand Central Dispatch (GCD / libdispatch)**: Apple's high-level concurrency framework. Manages a thread pool sized to available cores. Developers submit work as **blocks** to **dispatch queues**:
- `dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0)` — parallel queue for user-initiated work.
- `dispatch_async(queue, ^{ /* work */ })` — submit work without blocking caller.
- `dispatch_apply(N, queue, ^(size_t i){ /* parallel loop body */ })` — parallel for loop.

**Quality-of-Service classes** direct the OS scheduler:
| QoS Class | Description | Core Assignment (Apple Silicon) |
|---|---|---|
| `QOS_CLASS_USER_INTERACTIVE` | UI work, animations | P-cores (performance) |
| `QOS_CLASS_USER_INITIATED` | User-triggered computation | P-cores |
| `QOS_CLASS_DEFAULT` | Default background work | P-cores or E-cores |
| `QOS_CLASS_UTILITY` | Long-running, user visible | E-cores (efficiency) |
| `QOS_CLASS_BACKGROUND` | Backup, indexing | E-cores |

**Apple Silicon (M-series) Heterogeneous Parallelism**: Apple M1/M2/M3/M4 chips have separate **P-cores** (high-performance, high-power) and **E-cores** (low-performance, low-power). macOS's scheduler uses QoS to assign tasks to the appropriate core type. Parallel algorithms that mix latency-sensitive and background work can take advantage of this heterogeneity automatically.

**Accelerate Framework**: Apple's vectorized math library — BLAS, LAPACK, vDSP — uses SIMD intrinsics internally, providing highly optimized parallel computation for matrix operations, FFTs, and signal processing.

**Metal Performance Shaders**: GPU-accelerated parallel computation. For architects building ML pipelines on Apple hardware, Metal provides direct access to the GPU's thousands of shader cores.

---

### FreeBSD

**ULE Scheduler**: FreeBSD's default scheduler since 5.x. Designed specifically for SMP workloads. Features per-CPU run queues (reduces lock contention vs a global run queue), interactive task detection, and NUMA awareness. Significantly outperforms the old 4BSD scheduler on multi-core workloads.

**Fine-grained kernel locking**: FreeBSD's SMP implementation replaced the Giant Lock (a single global kernel lock) with hundreds of fine-grained locks for individual kernel subsystems, enabling true kernel-level parallelism.

**CPU affinity**:
```bash
cpuset -l 0-3 -p <pid>       # Pin process to cores 0-3
cpuset -l 0,2 ./myprogram    # Run with affinity to cores 0 and 2
```

**Parallel network stack**: FreeBSD's network stack is designed for parallel processing. Network interface card (NIC) receive side scaling (RSS) distributes incoming packets across multiple cores by hashing the flow (src IP + dst IP + ports). Each core processes its own flow queue without contention. This is why Netflix chose FreeBSD for its CDN nodes — the network stack achieves multi-hundred-gigabit-per-second throughput.

---

### OS Recommendation for Parallel Workloads

| Use Case | Best OS | Key Technology | Reason |
|---|---|---|---|
| HPC / scientific computing | **Linux** | OpenMP, MPI, NUMA, SMP | Dominates Top500, best HPC tooling, NUMA control |
| Parallel server (CPU-bound) | **Linux** | pthreads, cgroups cpuset | Fine-grained control, SCHED_DEADLINE |
| .NET parallel applications | **Windows** | TPL, PLINQ, IOCP | Native .NET integration |
| ML inference (Apple hardware) | **macOS** | Metal, ANE, Accelerate | Heterogeneous P/E-cores, Apple Neural Engine |
| Parallel network processing | **FreeBSD** | RSS, zero-copy, ULE | Netflix-proven at Tbps scale |
| Embedded real-time parallel | **Linux (PREEMPT_RT)** | SCHED_DEADLINE, CPU isolation | Deterministic scheduling, isolcpus kernel param |

---

## Practical Application / Examples

### Example 1: Parallel Matrix Multiplication

**Problem**: Multiply two N×N matrices. Naïve sequential implementation: O(N³).

**Parallel strategy (data decomposition)**: Each row of the result matrix C is independent. Assign rows to threads.

```
C[i][j] = sum(A[i][k] * B[k][j]) for k in 0..N

Thread 0: computes rows 0..N/4
Thread 1: computes rows N/4..N/2
Thread 2: computes rows N/2..3N/4
Thread 3: computes rows 3N/4..N
```

**Cache-aware tiling (critical for performance)**: The naïve parallel version still has poor cache behavior. **Tiled (blocked) matrix multiplication** divides matrices into cache-fitting blocks (e.g., 64×64 floats = 16KB, fits in L1). Threads process blocks rather than rows, dramatically reducing cache misses.

```c
// OpenMP parallel tiled matrix multiply
#pragma omp parallel for schedule(dynamic)
for (int ii = 0; ii < N; ii += TILE_SIZE)
  for (int jj = 0; jj < N; jj += TILE_SIZE)
    for (int kk = 0; kk < N; kk += TILE_SIZE)
      for (int i = ii; i < min(ii+TILE_SIZE, N); i++)
        for (int j = jj; j < min(jj+TILE_SIZE, N); j++)
          for (int k = kk; k < min(kk+TILE_SIZE, N); k++)
            C[i][j] += A[i][k] * B[k][j];
```

Speedup achieved: Near-linear on 8-16 cores for large matrices. The bottleneck shifts to memory bandwidth at high core counts.

---

### Example 2: PostgreSQL Parallel Query Execution

PostgreSQL 9.6+ implements parallel query execution — a real-world parallel system embedded in a production database.

**Parallel Sequential Scan**: A table scan is partitioned among multiple **parallel worker processes** (not threads — PostgreSQL uses processes for isolation). Each worker scans a portion of the heap file. A **gather node** at the top of the query plan collects rows from all workers.

**Parallel Hash Join**: The build phase (hashing the smaller table) is executed in parallel by multiple workers, each building a partial hash table. Workers then probe their partial hash tables with tuples from the probe side.

**Parallel Aggregate**: `COUNT(*)`, `SUM()` are computed as partial aggregates per worker, then combined at the gather node.

**Configuration**:
```sql
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.1;
EXPLAIN SELECT count(*) FROM large_table WHERE condition;
-- Look for "Gather" nodes in the plan
```

**Architect implication**: Parallel query execution helps CPU-bound queries (complex aggregations, large joins) but not I/O-bound queries (the I/O is still the bottleneck). The overhead of worker process creation means small queries are slower with parallelism enabled.

---

### Example 3: Video Encoding Parallelism (FFmpeg)

FFmpeg parallelizes at multiple levels simultaneously:

**Frame-level parallelism**: Independent frames (I-frames and some B-frames) are encoded in parallel across threads. Each thread encodes a complete frame.

**Slice-level parallelism**: A single frame is divided into horizontal **slices**. Multiple threads encode different slices of the same frame simultaneously.

**Tile-based parallelism** (HEVC/H.265, AV1): Frames are divided into rectangular tiles. Tiles are encoded in parallel with minimal dependency.

```bash
# FFmpeg: use all available cores for encoding
ffmpeg -i input.mp4 -c:v libx264 -threads 0 output.mp4

# Explicit thread count
ffmpeg -i input.mp4 -c:v libx264 -threads 8 output.mp4
```

**Architect note**: Video encoding is near-embarrassingly parallel for I-frames but has dependencies for P-frames and B-frames (which depend on reference frames). The parallel fraction is ~70–85%, limiting speedup to ~4–6× on 8 cores per Amdahl's Law.

---

## Critical Considerations

### 1. Amdahl's Law — The Hard Limit

Any program contains a **sequential fraction S** (code that cannot be parallelized) and a parallel fraction `1 - S`.

```
Maximum Speedup = 1 / (S + (1-S)/N)
  Where N = number of cores
```

| Sequential Fraction | Max Speedup (4 cores) | Max Speedup (16 cores) | Max Speedup (∞ cores) |
|---|---|---|---|
| 5% | 3.48× | 9.54× | 20× |
| 10% | 3.08× | 6.40× | 10× |
| 25% | 2.29× | 3.37× | 4× |
| 50% | 1.60× | 1.88× | 2× |

**Architectural implication**: Before adding cores, reduce the sequential fraction. Every synchronization point, global lock, and serial aggregation step is a sequential fraction. This is more valuable than adding hardware.

### 2. Gustafson's Law — The Counterpoint

Amdahl's Law assumes a **fixed problem size**. Gustafson's Law observes: with more processors, we solve **larger problems** in the same time. As problem size grows, the parallel fraction dominates. This justifies massive parallelism for scalable workloads (big data, climate simulation, ML training at scale).

### 3. The Overhead Crossover Point

Parallelism has overhead: thread creation, work distribution, synchronization, result aggregation. For small inputs, this overhead exceeds the benefit. Always benchmark:
- A task taking 100µs sequentially with 10µs thread overhead → parallelism on 2 cores: 50µs + 10µs = 60µs. Slower.
- A task taking 100ms sequentially → parallelism on 2 cores: 50ms + 10µs = 50.01ms. Faster.

Rule of thumb: Task duration should be at least **100× the synchronization cost** for parallelism to be beneficial.

### 4. Memory Bandwidth Wall

Modern CPUs can compute far faster than RAM can supply data. Beyond a certain number of cores, adding more cores does not improve performance because all cores are competing for the same memory bus bandwidth. This is the **memory bandwidth wall** — common in data-parallel scientific computing.

**Roofline model**: A compute-performance model that identifies whether a workload is:
- **Compute-bound**: Adding more FLOPS (cores, SIMD) helps.
- **Memory-bandwidth-bound**: Adding FLOPS doesn't help. Must reduce memory access (improve cache reuse, compression, reduced precision).

### 5. Non-Determinism and Reproducibility

Parallel programs are inherently **non-deterministic** — the same input can produce different outputs on different runs because thread scheduling is non-deterministic. This breaks reproducibility in:
- Floating-point reductions (addition is not associative in floating point — different orderings produce different results).
- Debug builds (heisenbugs: bugs that disappear when you add logging).
- ML training (non-deterministic gradient accumulation produces different model weights).

Mitigation: deterministic parallel algorithms (reduce trees with fixed topology), seed-controlled random number generators per thread, reproducibility modes in ML frameworks (`torch.use_deterministic_algorithms(True)`).

### 6. GPU Parallelism Limitations

GPU compute is massively parallel (thousands of CUDA cores) but has hard constraints:
- **Kernel launch overhead**: ~5–20µs per CUDA kernel launch. Fine-grained tasks can be dominated by launch overhead.
- **PCIe bandwidth bottleneck**: Transferring data between CPU memory and GPU memory via PCIe (16 GB/s for PCIe 4.0 x16) is often the limiting factor, not GPU compute.
- **Thread divergence**: In a GPU warp (32 threads executing the same instruction), if-else branches cause serialization — threads taking the other branch must wait. All threads in a warp execute the same instruction at the same time.
- **Shared memory limits**: GPU shared memory (SRAM) per SM (streaming multiprocessor) is 48–164KB. Workloads must fit data into shared memory to avoid slow global memory access.

### 7. The Non-Scalability of Locks Under Contention

A lock under heavy contention becomes a **serialization point** that converts parallel work into sequential work. If 16 threads all contend for the same mutex, only 1 runs at a time in the critical section — effective parallelism = 1. Solutions:
- **Lock striping**: Instead of 1 lock for a hash table, use N locks (one per bucket or bucket range). Reduces contention by factor N.
- **Read-copy-update (RCU)**: Linux kernel technique — readers never lock, writers create a new copy and atomically swap the pointer. Readers see consistent data without any locking cost.
- **Lock-free data structures**: CAS-based stacks, queues, and hash maps that never block.

---

## Critical Synthesis Note

**The architectural meta-insight**: Most engineers approach parallelism as a **performance optimization applied to existing sequential code**. This is the wrong mental model. The correct model is that parallelism is a **design constraint** that must be considered at the problem decomposition phase. A parallel-unfriendly algorithm is far more expensive to fix post-hoc than a parallel-friendly one designed from the start. The sequential fraction S in Amdahl's Law is primarily determined at design time by architectural choices — the number of global locks, the structure of aggregation steps, the degree of shared mutable state — not at implementation time.

**Cross-disciplinary connection — Biology**: The architecture of the vertebrate cerebellum is embarrassingly parallel: ~70 billion granule cells compute independently and in parallel, feeding into ~15 million Purkinje cells. Evolution solved the same design problem engineers face — massive parallel throughput with minimal coordination overhead. The biological solution was radical localization: each Purkinje cell processes input only from its local granule cells. The engineering analog is **data locality**: design parallel systems so each worker's data is physically close (in cache, on the same NUMA node), minimizing cross-worker communication.

**Knowledge gap — automatic parallelization**: Despite decades of research, compilers still cannot automatically parallelize most loops. The problem is **alias analysis** — the compiler cannot statically determine whether two pointers point to the same memory location, so it must conservatively assume they do and refuse to parallelize. Advances in **alias-free programming models** (Rust's ownership system, Fortran's `PURE` functions, SPMD-style programming) and **polyhedral optimization** show promise, but the general automatic parallelization of arbitrary code remains an unsolved problem. An architect who designs software with explicit parallelism annotations (OpenMP, CUDA, Rust's Rayon) is still far ahead of what automatic tools can achieve.
