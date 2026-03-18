# Concurrency vs Parallelism

---

## Core Objective

The goal of this document is to permanently resolve one of the most consequential and persistent misconceptions in software engineering: **concurrency and parallelism are not synonyms**. They describe different properties of a system, operate at different levels of abstraction, address different problems, require different tools, and fail in different ways.

Conflating them leads to:
- Applying parallelism (adding CPU cores) to I/O-bound systems that need concurrency (non-blocking design) — and seeing no improvement.
- Applying concurrency (async/await, event loops) to CPU-bound systems that need parallelism — and seeing no improvement.
- Over-engineering simple systems that need neither.
- Under-engineering complex systems that need both.

For a software architect, the distinction is not academic. It is the foundation of every performance decision, every threading model choice, every OS-level tuning strategy, and every system scaling conversation.

> **The one-sentence distinction:**
> Concurrency is about *dealing with* many things at once — it is a **design property**.
> Parallelism is about *doing* many things at once — it is an **execution property**.

---

## Fundamental Concepts

### Definitions Side-by-Side

| Property | Concurrency | Parallelism |
|---|---|---|
| **Nature** | Structural / Design | Executional / Physical |
| **Question answered** | "How is the program structured to manage multiple in-progress tasks?" | "How many tasks are physically executing at the same instant?" |
| **Requires multiple cores?** | No — works on a single core via interleaving | Yes — requires 2+ hardware execution units |
| **Primary concern** | Responsiveness, task coordination, correctness | Throughput, raw compute speed |
| **Failure mode** | Race conditions, deadlocks, starvation, callback hell | Data hazards, false sharing, Amdahl's Law limits |
| **Abstraction level** | Application architecture / software design | Hardware utilization / runtime execution |
| **Key question at design** | "Can tasks make progress while others wait?" | "Can independent computations run simultaneously?" |

---

### The Conceptual Boundary: ASCII Clarity

```
CONCURRENCY (1 CPU core, 2 tasks)
Timeline: ─────────────────────────────────────────────────►
           [Task A]  [Task B]  [Task A]  [Task B]  [Task A]
           CPU switches rapidly between A and B.
           At any given nanosecond, only ONE task executes.
           But over time, BOTH make progress. This is concurrency.

PARALLELISM (2 CPU cores, 2 tasks)
Core 1: ──[Task A ──────────────────────────────────]──────►
Core 2: ──[Task B ──────────────────────────────────]──────►
           Both tasks execute at the EXACT same instant.
           No switching. Genuine simultaneity. This is parallelism.

BOTH COMBINED (2 cores, 4 tasks — the real world)
Core 1: ──[Task A]──[Task C]──[Task A]──[Task C]──────────►
Core 2: ──[Task B]──[Task D]──[Task B]──[Task D]──────────►
           Core 1 is CONCURRENT across Tasks A and C.
           Core 2 is CONCURRENT across Tasks B and D.
           Core 1 and Core 2 run in PARALLEL with each other.
           This is the architecture of every modern server.
```

---

### The Relationship — Orthogonal Axes

Concurrency and parallelism are **orthogonal** — independent of each other. A program can be:

**Concurrent without Parallelism:**
A single-core machine running a Node.js web server. Thousands of connections are open simultaneously (concurrent). The CPU handles one event per loop iteration (not parallel). All tasks make progress by interleaving. No physical simultaneity.

**Parallel without Concurrency (rare):**
A GPU executing the same instruction on 10,000 data points at the exact same clock cycle (SIMD). Each execution unit runs identically — no coordination, no task switching, no communication between units. Pure simultaneous execution.

**Concurrent AND Parallel (the common case):**
A 16-core web server running 200 goroutines. Up to 16 goroutines execute in parallel (on 16 cores). The Go runtime schedules 200 goroutines concurrently onto those 16 cores. Parallelism = 16. Concurrency = 200.

**Neither (sequential):**
A single-threaded script that processes a list of items one at a time in a for loop. No concurrency, no parallelism. Correct, simple, but bounded.

```
                Concurrency
                    │
          YES       │       NO
          ──────────┼──────────
          Web server│  Sequential
   YES    with async│  SIMD math
Parallel  I/O + pool│  GPU shader
          ──────────┼──────────
          Single-   │ Single-
   NO     thread    │ thread
          event loop│ for loop
```

---

### Taxonomy of Execution Entities

Understanding which execution entity to use is the first architectural decision in any concurrent or parallel system.

| Entity | Scheduled by | Memory | Creation cost | Parallelism capable? | Best for |
|---|---|---|---|---|---|
| Process | OS kernel | Isolated | ~1ms, MBs | Yes (different cores) | Security/fault isolation |
| OS Thread | OS kernel | Shared heap | ~10–100µs, ~1–8MB | Yes | CPU-bound parallelism |
| Green Thread / Goroutine | Runtime (user-space) | Shared heap | ~1µs, ~2–8KB | Yes (M:N with OS threads) | High-concurrency I/O |
| Coroutine (stackless) | Application/event loop | Shared | Nanoseconds, bytes | No (single-threaded) | Async I/O pipelines |
| Fiber | Application (explicit) | Own stack | Microseconds, ~64KB | No (single-threaded) | Cooperative task systems |
| SIMD lane | CPU hardware | Registers | Zero (hardware) | Yes (data parallel) | Vectorized computation |
| GPU thread | GPU hardware scheduler | GPU VRAM | Batch launch | Yes (massively) | Data-parallel compute |

---

### The Bottleneck Classification

The most critical input to the concurrency-vs-parallelism decision is the **bottleneck type**.

| Bottleneck | Definition | Example | Solution |
|---|---|---|---|
| **I/O-bound** | Task spends most time waiting for external I/O (network, disk, DB, user) | HTTP server waiting for DB responses | Concurrency — overlap the waiting |
| **CPU-bound** | Task spends most time executing computations, no waiting | Image processing, ML training, encryption | Parallelism — add compute units |
| **Memory-bandwidth-bound** | Task saturates the memory bus, not CPU ALUs | Large matrix multiply on many cores | Parallelism (up to mem BW limit) + cache optimization |
| **Mixed** | Both I/O and CPU-intensive phases | Database server, game server | Both concurrency AND parallelism |

**Diagnosing your bottleneck**:
```bash
# CPU utilization per core (is CPU maxed?)
htop                          # interactive, per-core bars
mpstat -P ALL 1               # per-CPU stats every 1 second

# I/O wait time (is CPU idle waiting for I/O?)
iostat -x 1                   # disk I/O utilization
sar -u 1                      # %iowait column

# System call analysis (is time in syscalls waiting?)
strace -c -p <pid>            # count time per syscall
perf stat ./myprogram         # instructions, cache misses, cycles

# Network I/O
ss -s                         # socket statistics
nethogs                       # per-process network usage
```

---

## Procedure or Logic

### The Architectural Decision Framework

A software architect must answer these questions sequentially before choosing a concurrency or parallelism model.

**Step 1 — Classify the workload bottleneck**
- Profile first. Never guess. Use `perf`, `strace`, `htop`, `iostat`.
- `%iowait` high → I/O-bound → Concurrency is the primary tool.
- CPU cores at 100% → CPU-bound → Parallelism is the primary tool.
- Both → Mixed → Both are needed, with a thread pool bridging the two.

**Step 2 — Measure the parallel fraction (for CPU-bound work)**
- Profile which code paths are parallelizable (no sequential dependencies).
- Apply Amdahl's Law: `Speedup = 1 / (S + (1-S)/N)` where S = sequential fraction.
- If S = 30%, max speedup with infinite cores = 3.3×. If ROI is insufficient, don't parallelize.

**Step 3 — Assess shared state**
- List all data written by concurrent/parallel tasks.
- Each shared mutable variable is a synchronization requirement.
- Prefer: eliminate sharing (private copies) > atomics > fine-grained locks > coarse locks.
- More shared state → more synchronization overhead → less benefit from parallelism.

**Step 4 — Choose the execution model**

For **concurrency** (I/O-bound, high task count, responsiveness):
- Single-threaded event loop: Node.js, Redis, Nginx worker model. Zero thread overhead. No synchronization needed.
- Green threads / goroutines: Go, Java Virtual Threads. Lightweight, blocks without blocking OS thread. Best default for most server applications.
- Async/await coroutines: Python asyncio, Rust tokio, C# async. Good when language ecosystem supports it.
- Actor model: Erlang, Akka. Best for distributed, fault-tolerant, stateful concurrent systems.

For **parallelism** (CPU-bound, independent data, maximize throughput):
- Thread pool + work queue: Java ForkJoinPool, .NET ThreadPool, C++ `std::async`. General-purpose.
- Data parallelism (SIMD): AVX/SSE intrinsics, auto-vectorization, NumPy. Per-core throughput.
- Process-level parallelism: Python `multiprocessing`, Nginx worker processes. Bypass GIL, fault isolation.
- GPU parallelism: CUDA, OpenCL, Metal, WebGPU. Massively parallel data-parallel computation.
- Distributed parallelism: MPI, Spark, Dask. Multi-node parallel computation.

For **both** (mixed workloads):
- Event loop + thread pool: Node.js worker_threads / libuv thread pool. Async I/O + CPU offload.
- Goroutines + GOMAXPROCS: Go handles this naturally. Goroutines are concurrent; runtime uses multiple OS threads for parallelism.
- Reactive pipeline: Spring WebFlux, Akka Streams. Concurrent stream processing with parallel operators.
- Async + Parallel.For: C# async I/O with `Parallel.ForEach` for CPU phases.

**Step 5 — Account for task granularity**
- Task too fine-grained (< ~100µs): Threading overhead dominates. Sequential or batched.
- Task appropriately sized (> 1ms): Parallelism beneficial.
- Task coarse (> seconds): Parallelism with minimal workers. Focus on Amdahl's sequential fraction.

---

### Execution Model Comparison: Concurrency

**1. Preemptive Multitasking (OS-managed threads)**
The OS forcefully interrupts a running thread after its time slice (quantum, typically 1–10ms) and switches to another. The thread has no say.

```
Thread A: ──[run]──[run]──[PREEMPT]──────────────[run]──►
Thread B: ────────────────[run]──[run]──[PREEMPT]──────►
                 ↑ OS decides switches ↑
```

Used by all modern OSes for thread scheduling. Threads never monopolize CPU. Cost: context switch overhead (~1–10µs). Risk: preemption at any point requires synchronization of all shared state.

**2. Cooperative Multitasking (event loops, coroutines)**
Tasks voluntarily yield control at defined points (I/O operations, explicit yield calls). The scheduler runs the next ready task.

```
Coroutine A: ──[run]──[await I/O]────────────────[resume]──►
Coroutine B: ──────────[run]──[await I/O]──[resume]────────►
                    ↑ task yields ↑  ↑ event fires ↑
```

Used by: Python `asyncio`, JavaScript event loop, Node.js, Go (partially). Lower overhead (no OS context switch). Risk: a task that never yields blocks the entire system.

**3. Actor-based Concurrency**
Tasks are isolated actors communicating only via asynchronous message passing. No shared state.

```
Actor A:  [state] ←── message ──── Actor B: [state]
            │                                  ↑
            └──── message ───────────────────►
```

Used by: Erlang, Elixir, Akka. Eliminates race conditions by design (no sharing). Scales to distributed systems naturally (a remote actor is identical to a local one, just with higher message latency).

---

### Execution Model Comparison: Parallelism

**1. Fork-Join**
```
Main: ──[fork]──────────────────────────[join]──► continue
              ├── Worker 0: [compute] ──┤
              ├── Worker 1: [compute] ──┤
              └── Worker 2: [compute] ──┘
```

**2. Pipeline**
```
Stage 1: [item1]──[item2]──[item3]──[item4]──►
Stage 2:          [item1]──[item2]──[item3]──►
Stage 3:                   [item1]──[item2]──►
         All stages execute simultaneously on different items
```

**3. MapReduce**
```
Input → [Partition] → Map(p1) Map(p2) Map(p3) → [Shuffle] → Reduce(k1) Reduce(k2) → Output
                       (parallel)                             (parallel)
```

---

### When to Use Concurrency vs Parallelism: Decision Table

| Scenario | Use | Why |
|---|---|---|
| Web server: 10,000 HTTP requests, each waits 50ms for DB | **Concurrency** | Bottleneck is I/O wait, not CPU. Overlapping waits is the gain. |
| Video transcoding: 4K, CPU-intensive | **Parallelism** | Bottleneck is compute. More cores = more throughput. |
| GUI application: user interactions + background load | **Concurrency** | UI thread must remain responsive. Background work deferred. |
| Matrix multiplication: N×N matrices | **Parallelism** | Pure compute, perfectly data-parallel. |
| Chat server: 50,000 simultaneous WebSocket connections | **Concurrency** | Each connection mostly idle (waiting for messages). |
| ML model training on GPUs | **Parallelism** | Massively parallel tensor operations. |
| API gateway: requests fan out to 5 microservices | **Concurrency** | Fan-out eliminates sequential waiting. |
| Database: complex analytical query on 10B rows | **Parallelism** | Parallel table scan, parallel hash join. |
| Real-time game server: physics + networking + AI | **Both** | I/O concurrency for networking + CPU parallelism for physics/AI. |
| Data pipeline: ingest + transform + load | **Both** | Concurrent I/O + parallel transformation stages. |

---

### Pitfall Decision Table

| Situation | Wrong Choice | Why It Fails | Correct Choice |
|---|---|---|---|
| 10,000 slow DB queries | Parallelize with 10,000 threads | ~10GB RAM in thread stacks; CPU idle 95% waiting for I/O | Concurrent async I/O |
| Compress 10GB of data | Single async coroutine | Async doesn't add CPU cycles | Parallel threads across cores |
| Web server on single-core VM | Multi-process parallelism | Can't parallelize on 1 physical core | Concurrent event loop |
| Matrix multiply on 64 cores | Single-threaded async design | No I/O to overlap; pure compute | Parallel SIMD/threads |
| Real-time audio playback | Many concurrent threads | Scheduling jitter causes audio glitches | Single high-priority real-time thread |
| Small 10ms tasks, parallelized | 32 threads with fork-join | Thread overhead > task duration | Sequential or coarser batching |
| CPU-heavy in async event loop | `await cpu_work()` | Blocks event loop for all other tasks | `await executor.run(cpu_work)` in thread pool |

---

## Practical Application / Examples

### Scenario 1: Web Server — Concurrency is the Answer

**Problem**: Build a web server handling 50,000 simultaneous connections. Each request queries a database (50ms) and an external API (100ms). Total time if sequential: 150ms per request. Server has 8 cores.

**Naive parallel approach** (wrong): 50,000 OS threads.
- Each thread: ~1–8MB stack → 50–400GB RAM. Impossible.
- Each thread: idle 99% of the time (waiting for I/O). CPU utilization: ~1%.
- Context switch overhead with 50K threads: system becomes unresponsive.

**Correct concurrent approach**: Event loop + async I/O.
- Single event loop (or 8 worker event loops, one per core).
- Each request is a coroutine/goroutine. When it awaits the DB response, the event loop processes another request.
- Memory: each goroutine uses ~2–8KB. 50,000 goroutines = ~100–400MB. Feasible.
- CPU: kept busy processing events continuously, never idling on I/O.
- Total time for a single request: ~100ms (DB + API in parallel via concurrency). Same as with threads.

```
Goroutine (request handler):
  1. start DB query    → await (suspends goroutine, runs others)
  2. start API call    → await (suspends goroutine, runs others)
  3. DB response arrives → goroutine resumes
  4. API response arrives → goroutine resumes
  5. combine + respond
  Total: ~100ms (longest await), not 150ms (sequential)
```

**Key insight**: Concurrency here doesn't speed up any individual request's CPU work — there is no CPU work to speak of. It speeds up throughput by eliminating idle waiting.

---

### Scenario 2: Video Encoding — Parallelism is the Answer

**Problem**: Encode a 1-hour 4K video. Each frame requires heavy DCT, quantization, and entropy coding computation. 8-core machine.

**Naive concurrent approach** (wrong): Single-threaded async encoder.
- Async/await doesn't create CPU compute — it only avoids blocking on I/O.
- There is no I/O to await in pure encoding. All work is CPU computation.
- Result: same single-threaded performance. No improvement.

**Correct parallel approach**: Divide frames across 8 CPU cores.
- Frame-level parallelism: different threads encode independent frames simultaneously.
- Result: ~7–7.5× speedup (Amdahl's Law: sequential fraction ~5–10% for frame headers, multiplexing).

```
Core 0: [Frame 1 encode] [Frame 9  encode] [Frame 17 encode] ──►
Core 1: [Frame 2 encode] [Frame 10 encode] [Frame 18 encode] ──►
Core 2: [Frame 3 encode] [Frame 11 encode] [Frame 19 encode] ──►
...
Core 7: [Frame 8 encode] [Frame 16 encode] [Frame 24 encode] ──►
```

**Key insight**: Parallelism here reduces wall-clock time by distributing compute work. Concurrency would not help because there is nothing to interleave with — every nanosecond is spent computing.

---

### Scenario 3: Database Server — Both Are Required

**Problem**: PostgreSQL handling 500 concurrent queries. Some are simple key lookups (I/O-bound, fast). Some are complex analytical queries (CPU-bound, slow joins over billions of rows).

**Concurrency layer** (handles 500 simultaneous clients):
- 500 client connections are handled concurrently. While one client waits for disk I/O, another's query executes.
- Process-based concurrency (PostgreSQL uses processes, not threads, for isolation).
- The OS preemptive scheduler handles interleaving across processes.

**Parallelism layer** (speeds up expensive individual queries):
- For complex analytical queries, PostgreSQL's parallel query executor splits the work.
- Parallel sequential scan: 4 worker processes each scan 25% of the table simultaneously.
- Parallel hash join: 4 workers each build partial hash tables in parallel.
- Parallel aggregate: partial counts/sums per worker, aggregated at the gather node.

```
Client pool (500 connections) ──── Concurrency ────►
         │
         ├── Simple query → single process → done quickly
         └── Complex query → Parallel workers:
              Worker 0: scan rows 0-25%
              Worker 1: scan rows 25-50%
              Worker 2: scan rows 50-75%
              Worker 3: scan rows 75-100%
                    ↓ gather + aggregate ↓
               Final result to client
```

**Key insight**: Concurrency keeps the server responsive under many clients. Parallelism reduces latency for expensive individual queries. Neither alone is sufficient for a full-featured database server.

---

### Scenario 4: OS-Level View — What the OS Does

**On a quad-core machine running a web server, terminal, music player, and IDE simultaneously**:

**Concurrency (single core perspective)**:
- Each core runs one thread at a time.
- The OS scheduler switches between threads every 1–10ms.
- Your music player, IDE, web server, and terminal all appear to run simultaneously on each core because the switching is faster than human perception.
- This is **concurrency**: multiple tasks in progress, interleaved.

**Parallelism (multi-core perspective)**:
- Core 0: web server worker thread
- Core 1: IDE syntax highlighting thread
- Core 2: music player audio decoding thread
- Core 3: terminal shell process
- These four actually execute simultaneously. This is **parallelism**.

**The OS does both simultaneously**: Concurrency within each core (via the scheduler's time-slicing), Parallelism across cores (via multi-core hardware utilization). The Completely Fair Scheduler (CFS) on Linux manages both dimensions transparently.

---

## Operating Systems: Concurrency vs Parallelism Capabilities

### Linux

**For Concurrency:**
- `epoll`: O(1) I/O event notification. Handles 100,000+ file descriptors efficiently. The foundation of Nginx, Redis, Node.js, HAProxy.
- `io_uring` (Linux 5.1+): Submission/completion ring buffers eliminate per-operation syscall overhead. Highest-throughput async I/O available on any OS.
- `futex`: Fast userspace mutex — uncontended path is a single atomic operation, no kernel call. Makes concurrent locking cheap.
- CFS scheduler: Fair, low-latency scheduling for concurrent interactive tasks.
- `eventfd`, `timerfd`, `signalfd`: Integrate OS events into `epoll` loops cleanly.
- Tunable: `ulimit -n 1000000` (open files), `net.core.somaxconn` (TCP backlog).
- **Verdict for concurrency**: Best-in-class. Nginx reference architecture. Dominant in production.

**For Parallelism:**
- Full SMP support since kernel 2.0. Fine-grained kernel locking (no global BKL since 2.6.39).
- NUMA support: `numactl`, `libnuma`, `mbind()`. NUMA-aware allocator.
- CPU affinity: `taskset`, `sched_setaffinity`.
- Real-time scheduling: `SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE` for deterministic parallel tasks.
- cgroups cpuset: isolate workloads to specific CPU cores.
- OpenMP, MPI, Intel TBB: all first-class Linux citizens.
- 100% of Top500 supercomputers run Linux.
- **Verdict for parallelism**: Best-in-class for HPC and high-performance server parallelism.

**Overall: Linux is the dominant OS for both concurrency and parallelism in production systems.**

---

### Windows

**For Concurrency:**
- **IOCP (I/O Completion Ports)**: Completion-based async I/O. When I/O completes, a completion packet is queued and a thread pool worker processes it. Scales to tens of thousands of concurrent connections. Used internally by .NET's `ThreadPool` and `HttpListener`.
- **Overlapped I/O**: Every async I/O on Windows is "overlapped" — returns immediately, completes later via IOCP.
- **.NET async/await**: `Task`, `async`/`await`, `ConfigureAwait(false)`. The most ergonomic async model in any mainstream language. Backed by IOCP on Windows.
- **Windows Message Loop**: UI concurrency model. `PostMessage` (async, non-blocking), `SendMessage` (sync, blocking — avoid from UI thread).
- **Verdict for concurrency**: Excellent — especially for .NET ecosystem. IOCP is deeply integrated, async model is mature.

**For Parallelism:**
- **Thread Pool API**: `SubmitThreadpoolWork`, `TP_WORK`, `TP_IO`. Structured parallel work submission.
- **Processor Groups**: Systems with >64 logical processors use processor groups. APIs require explicit group management.
- **TPL (.NET)**: `Parallel.For`, `Parallel.ForEach`, PLINQ. Data and task parallelism with automatic partitioning.
- **SetThreadAffinityMask**: Pin threads to specific logical processors.
- **Concurrency Visualizer** (Visual Studio): Timeline-based visualization of thread execution, blocking, synchronization.
- **Verdict for parallelism**: Strong in the .NET ecosystem. Less control than Linux for native code.

---

### macOS

**For Concurrency:**
- **kqueue**: BSD-origin event notification. More powerful than epoll — unified API for sockets, files, processes, signals, timers in a single call.
- **GCD (Grand Central Dispatch)**: The most ergonomic high-level concurrency API. Dispatch queues (serial and concurrent), QoS classes, dispatch groups, dispatch barriers.
- **Swift Structured Concurrency**: `async`/`await`, `TaskGroup`, `AsyncStream`. Actors provide compile-time data-race safety.
- **NSRunLoop**: Foundation event loop, integrates with GCD, timers, input sources.
- **Verdict for concurrency**: Excellent developer ergonomics. Swift actors are the safest high-level concurrency model in any mainstream language.

**For Parallelism:**
- **Apple Silicon (M-series) heterogeneous cores**: P-cores (high-performance) + E-cores (efficiency). macOS scheduler assigns tasks to core type via QoS. Parallel workloads using `QOS_CLASS_USER_INITIATED` automatically run on P-cores.
- **Accelerate framework**: BLAS, LAPACK, vDSP — optimized vectorized parallel math using SIMD.
- **Metal Performance Shaders**: GPU-accelerated parallel computation.
- **Apple Neural Engine (ANE)**: Dedicated ML accelerator — parallel tensor operations at low power.
- **Verdict for parallelism**: Exceptional for heterogeneous compute (ML, graphics, signal processing) on Apple hardware. Limited for HPC workloads.

---

### FreeBSD

**For Concurrency:**
- **kqueue (origin)**: FreeBSD invented kqueue. Unified event notification — more orthogonal API than Linux's epoll for combining different event types.
- **sendfile()**: Zero-copy file-to-socket transfer. The same bytes that hit the page cache go directly to the NIC DMA buffer. Critical for high-concurrency content delivery.
- **Kernel TLS**: TLS encryption in kernel space eliminates userspace↔kernel copy overhead for TLS connections.
- **Netflix CDN**: Serves ~15% of global internet traffic using FreeBSD. Proves concurrent I/O at Tbps scale.
- **Verdict for concurrency**: Best-in-class for high-throughput static content delivery and CDN workloads.

**For Parallelism:**
- **ULE Scheduler**: Per-CPU run queues with work stealing. NUMA-aware. Designed for SMP from the ground up.
- **Fine-grained kernel locking**: The network stack processes packets in parallel with minimal cross-core locking. Multi-queue NIC support with RSS (Receive Side Scaling) — packets hashed to per-core queues.
- **cpuset**: `cpuset -l 0-7 ./program` pins processes/threads to specific cores.
- **Verdict for parallelism**: Strong for network-centric parallel workloads. Less tooling than Linux for HPC.

---

### OS Recommendation Matrix

| Use Case | Best OS | Key Tech | Reason |
|---|---|---|---|
| High-concurrency web server (>100K conn) | **Linux** | epoll, io_uring | Scalable async I/O, tunable kernel, dominant ecosystem |
| HPC / scientific parallel computing | **Linux** | OpenMP, MPI, NUMA, SMP | 100% of Top500, best HPC tooling |
| .NET concurrent/parallel services | **Windows** | IOCP, async/await, TPL | Native platform integration, mature tooling |
| CDN / high-throughput video streaming | **FreeBSD** | sendfile, kTLS, kqueue | Netflix-proven at Tbps, zero-copy stack |
| Apple platform app development | **macOS** | GCD, Swift actors, kqueue | First-class APIs, compile-time safety |
| ML inference on Apple hardware | **macOS** (Apple Silicon) | ANE, Metal, Accelerate | Heterogeneous parallel compute, battery efficiency |
| Real-time embedded concurrent | **Linux (PREEMPT_RT)** | SCHED_DEADLINE, futex | Deterministic low-latency scheduling |
| Secure concurrent server | **FreeBSD** | Capsicum + kqueue | Capability-based isolation per connection |

---

## Common Architectural Patterns

### Thread-per-Request (Apache MPM Prefork / early Tomcat)
One OS thread is created per incoming request. Simple to implement. Each request can block synchronously without affecting others.

```
Request 1 → Thread 1: [block on DB 50ms] [block on API 100ms] → respond
Request 2 → Thread 2: [block on DB 50ms] [block on API 100ms] → respond
```

**Problem**: 10,000 concurrent requests = 10,000 threads = 10–80GB RAM. OS scheduler struggles. Does not scale beyond a few thousand concurrent connections (the C10K problem).

**When appropriate**: Low concurrency, simple code, legacy systems.

---

### Event Loop + Thread Pool (Nginx, Node.js, Netty, Vert.x)
A single-threaded (or per-core) event loop handles all I/O concurrently. CPU-intensive work is offloaded to a bounded thread pool.

```
Event Loop (single thread):
  ← accepts connections
  ← reads request data (epoll/kqueue)
  → dispatches CPU work to thread pool
  ← receives thread pool results
  → writes responses (non-blocking)

Thread Pool (N threads, N = CPU count):
  ← picks up CPU work tasks
  → returns results to event loop
```

**Benefits**: Scales to millions of concurrent connections. CPU parallelism for compute work. No per-connection thread overhead.

**Used by**: Nginx, Node.js (with `worker_threads`), Netty (Java), Twisted (Python), Vert.x.

---

### Actor System (Erlang/Elixir, Akka)
Thousands to millions of lightweight isolated actors. Each handles its own concurrent state. Supervision trees restart failed actors.

```
Supervisor
   ├── Worker Actor 1: [mailbox] [private state]
   ├── Worker Actor 2: [mailbox] [private state]
   └── Worker Actor 3: [mailbox] [private state]

Actors communicate only via async messages.
No shared state → no data races.
Supervisor restarts crashed actors → fault tolerance.
```

**Benefits**: Location transparency (works identically local or distributed), fault tolerance by design, no shared-state synchronization needed.

**Used by**: WhatsApp (2 million connections per server with Erlang), Akka (JVM), Elixir Phoenix (high-concurrency web).

---

### Fork-Join Task Parallelism (ForkJoinPool, Go fan-out)
A parallel computation is recursively decomposed until tasks are small enough to compute directly. Results are merged up the tree.

```
solve(array[0..N]):
    if N < THRESHOLD: solve directly
    else:
        fork: solve(array[0..N/2])      ← parallel
        fork: solve(array[N/2..N])      ← parallel
        join: merge both results
```

**Benefits**: Automatic load balancing via work stealing. Scales to available cores. Ideal for divide-and-conquer algorithms.

**Used by**: Java `ForkJoinPool`, .NET `Parallel.For`, Intel TBB, Go goroutine fan-out.

---

### Reactive Pipeline (Kafka Streams, Spring WebFlux, Akka Streams)
Data flows through a pipeline of operators. Each operator is a concurrent processing stage. Back-pressure signals propagate upstream.

```
Source(rate: 10K/s)
  → filter(predicate)
  → map(transform)
  → groupBy(key)
  → window(10s)
  → aggregate(sum)
  → sink(database)
  ← backpressure ← (sink signals how fast it can consume)
```

**Benefits**: Handles unbounded streams, built-in backpressure, declarative composition.

**Used by**: Kafka Streams, Apache Flink, Spring WebFlux (Project Reactor), RxJava.

---

### Pattern Selection Matrix

| | **Shared Mutable State** | **Isolated State** |
|---|---|---|
| **I/O-bound** | Threads + locks + async I/O | Actor model, event loop, goroutines |
| **CPU-bound** | Thread pool + fine-grained locks | Fork-join, MapReduce, SIMD |

---

## Benchmarking and Measurement

### Measuring Concurrency Benefit

The primary metrics for concurrency effectiveness:

| Metric | Tool | What It Shows |
|---|---|---|
| **Throughput** (req/s) | `wrk`, `hey`, `k6`, `ab` | How many requests per second at given concurrency |
| **Latency percentiles** (p50/p95/p99) | `wrk`, `k6`, `vegeta` | Tail latency — where 95% or 99% of users land |
| **Connection count** | `ss -s` | How many sockets are in ESTABLISHED/TIME_WAIT state |
| **Event loop lag** | Node.js `perf_hooks.monitorEventLoopDelay()` | How backed-up the event loop is |
| **Goroutine count** | `runtime.NumGoroutine()` (Go) | Detect goroutine leaks |

```bash
# Benchmark with 1000 concurrent connections, 30 seconds
wrk -t8 -c1000 -d30s --latency http://localhost:8080/api/endpoint

# Throughput results output:
# Requests/sec: 125,432
# Latency 50%: 2.1ms, 95%: 8.7ms, 99%: 45.2ms
```

### Measuring Parallelism Benefit

| Metric | Tool | What It Shows |
|---|---|---|
| **Speedup ratio** | `time` (compare 1 core vs N cores) | How much faster with N workers |
| **CPU utilization** | `perf stat -e cycles,instructions` | Are cores doing real work or stalling? |
| **Per-core load** | `htop`, `mpstat -P ALL 1` | Is work evenly distributed? |
| **NUMA locality** | `numastat -p <pid>` | Are threads hitting local vs remote RAM? |
| **Cache miss rate** | `perf stat -e cache-misses,LLC-load-misses` | Cache efficiency under parallel load |
| **Memory bandwidth** | `likwid-bench`, `stream` benchmark | Are you hitting the memory bandwidth wall? |
| **Flamegraph** | `perf record -g + flamegraph.pl` | Where is actual time spent? |

```bash
# Measure parallelism efficiency
time ./program --threads=1    # baseline
time ./program --threads=4    # with 4 cores
time ./program --threads=8    # with 8 cores
# speedup = baseline_time / N_thread_time
# efficiency = speedup / N_threads (target: > 0.7)

# CPU hardware utilization
perf stat -e cycles,instructions,cache-misses,LLC-load-misses ./program
```

### The Speedup Measurement Protocol

1. Run single-threaded: record wall time T₁.
2. Run with N threads: record wall time Tₙ.
3. Speedup S = T₁ / Tₙ.
4. Efficiency E = S / N. If E < 0.6, investigate: synchronization overhead? Memory bandwidth? False sharing? Load imbalance?
5. Plot speedup vs N. Where does the curve flatten? That's your effective parallelism limit (Amdahl's sequential fraction).

---

## Critical Considerations

### 1. Concurrency Is Not Free — The Complexity Tax

Every concurrent system introduces non-determinism. Code that looks correct when read sequentially may be broken when executed concurrently. The complexity is not in writing the code — it's in **reasoning about all possible interleavings**. For N concurrent tasks with M shared variables, the state space is factorial. This is why:
- Formal methods (TLA+, model checkers) exist.
- Immutable data structures are valuable (Clojure, Haskell, functional core/imperative shell).
- Message passing eliminates sharing (Go channels, actor model).
- Rust's type system prevents data races at compile time.

### 2. Parallelism Has Diminishing Returns (Amdahl's Law)

Every parallel program has a sequential fraction S (initialization, result aggregation, lock-protected sections). Adding cores beyond `1/S` provides no benefit.

| Sequential Fraction | Max speedup (∞ cores) | Cores needed for 90% of max speedup |
|---|---|---|
| 5% | 20× | ~10 cores |
| 10% | 10× | ~5 cores |
| 25% | 4× | ~3 cores |
| 50% | 2× | ~2 cores |

**Architect implication**: Reducing S is often more valuable than adding hardware. Every global lock and serial aggregation step is a hidden sequential fraction.

### 3. False Sharing Silently Destroys Parallel Performance

Two threads on different cores modify different variables that happen to share a **64-byte cache line**. The CPU cache coherence protocol forces the cache line to bounce between cores on every write, serializing what should be independent work.

```c
// Dangerous: both counters likely on same cache line
struct { int counter_a; int counter_b; } counters;

// Fixed: pad to separate cache lines
struct { int counter_a; char _pad[60]; int counter_b; } counters;
```

### 4. The async/await CPU Trap

`async`/`await` and event loops are for I/O-bound work. A CPU-heavy computation placed directly in an async function blocks the event loop, freezing all other concurrent tasks.

```python
# WRONG: blocks event loop for all other coroutines
async def handler():
    result = heavy_computation()    # blocks entire event loop
    return result

# CORRECT: offload CPU work to thread pool
async def handler():
    result = await loop.run_in_executor(None, heavy_computation)
    return result
```

### 5. The Thundering Herd Problem

When a lock is released or a condition fires, all waiting threads wake simultaneously. Only one proceeds; the rest block immediately again. At scale (thousands of threads), this creates a burst of wasted context switches and cache pollution.

**Solutions**: Use `signal()` instead of `broadcast()` when only one thread should proceed. Use work-stealing queues instead of a single shared queue. Use `SO_REUSEPORT` to give each worker process its own accept queue.

### 6. Observability in Concurrent Systems

A concurrent system processing thousands of simultaneous requests is opaque without instrumentation. A bug that manifests as a race condition at 10,000 req/s is invisible at 10 req/s.

**Mandatory practices**:
- **Correlation IDs**: Each request gets a unique ID. All log entries for that request include the ID. Makes it possible to trace a single request through concurrent processing.
- **Distributed tracing**: OpenTelemetry, Jaeger, Zipkin. Trace a request as it crosses async boundaries, thread pools, and services.
- **Thread/goroutine IDs in logs**: Identify which execution unit produced which log entry.

### 7. Testing Concurrent Code

Deterministic unit tests miss concurrency bugs because they run deterministically. The bug depends on timing.

**Strategies**:
- **Race detectors**: ThreadSanitizer (C/C++), `go test -race` (Go), Helgrind.
- **Stress testing**: Run the same test 10,000 times concurrently. Amplifies timing windows.
- **Property-based testing**: Generate random operation sequences. Verify invariants hold for all sequences.
- **Chaos engineering**: Inject delays, kill threads, simulate partial failures to expose race conditions in production-like conditions.
- **Formal verification**: TLA+, SPIN model checker. AWS found critical bugs in DynamoDB using TLA+ before they shipped.

### 8. Mechanical Sympathy — Hardware Awareness

Concurrent and parallel programs that ignore hardware reality leave significant performance on the table:
- **Cache line size**: 64 bytes on x86/ARM. Padding prevents false sharing.
- **NUMA topology**: Threads should access memory on their local socket.
- **Write combining**: Sequential writes to the same cache line are faster than scattered writes.
- **Branch prediction**: Unpredictable branches in hot parallel loops stall the pipeline.
- **Memory ordering**: Relaxed atomics are far cheaper than seq_cst on ARM (requires memory barriers on every seq_cst operation).

---

## Critical Synthesis Note

**The meta-architectural insight:**
Concurrency and parallelism exist at **different layers of abstraction**, and this is precisely why conflating them causes architectural mistakes. Concurrency is expressible on paper — in a flowchart, in pseudocode, in a whiteboard diagram — before any hardware is chosen. Parallelism is a statement about what physical hardware is doing at a specific nanosecond. This means:

1. **Design for concurrency first.** A well-structured concurrent design can run on 1 core or 1000 cores. A poorly-structured sequential design cannot be parallelized without fundamental rewriting.
2. **Parallelism is a scaling lever applied to a concurrent design.** You don't add parallelism to a sequential program — you add it to an already-concurrent decomposition of the problem.
3. **The correct order of concern is**: Correctness → Concurrency model → Parallelism degree. Most performance problems are solved by getting the concurrency model right before touching core count.

**Cross-disciplinary connection — Cognitive Science:**
Humans exhibit concurrency without parallelism: we switch attention between multiple concerns (planning, speaking, walking) rapidly enough that they appear simultaneous. But genuine dual-processing — two independent cognitive processes at full capacity simultaneously — is limited (the psychological refractory period). Human "multitasking" is actually rapid task switching (concurrency), not true parallel processing. Engineers who understand this distinction perform better at architectural reasoning about concurrent systems: they recognize that most "simultaneous" requirements can be met by concurrency (overlapping progress) rather than true parallelism (simultaneous compute). This prevents over-engineering.

**Knowledge gap — Hardware Transactional Memory (HTM):**
Intel TSX (Transactional Synchronization Extensions) and IBM POWER's HTM allow the CPU to speculatively execute code as a transaction. If no conflicting writes occur, the transaction commits without any lock. On conflict, it aborts and retries. HTM blurs the concurrency/parallelism boundary: the execution is parallel (multiple cores, no locks), but the programming model is sequential within a transaction. The interaction between HTM, the OS scheduler, and virtual machine hypervisors (which can cause spurious aborts via interrupts) creates subtle correctness issues that are not yet fully characterized. An architect building lock-free data structures should investigate HTM's guarantees and failure modes — this is an underexplored area with significant performance potential and non-obvious hazards.
