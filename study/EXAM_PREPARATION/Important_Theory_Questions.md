# Master University Theory Examination Question Bank

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 1–32)  
**Applicability:** University MidSem (Lectures 1–16) & EndSem (Lectures 1–32) Examinations  
**Grading Standard:** Every question is presented in formal university examination format with marks weighting ([6 to 12 Marks]), expected evaluation criteria, complete model answers, architectural ASCII diagrams, and key grading points.

---

## Table of Contents
1. [HPC Architecture, Hardware & Performance Scaling (Q1 – Q5)](#1-hpc-architecture-hardware--performance-scaling-q1--q5)
2. [OpenMP Foundations, Work-Sharing & Synchronization (Q6 – Q11)](#2-openmp-foundations-work-sharing--synchronization-q6--q11)
3. [OpenMP Tasks & SIMD Vectorization (Q12 – Q13)](#3-openmp-tasks--simd-vectorization-q12--q13)
4. [POSIX Threads & Concurrency Control Primitives (Q14 – Q18)](#4-posix-threads--concurrency-control-primitives-q14--q18)
5. [Distributed Memory, MPI & Hybrid Programming (Q19 – Q24)](#5-distributed-memory-mpi--hybrid-programming-q19--q24)

---

## 1. HPC Architecture, Hardware & Performance Scaling (Q1 – Q5)

### Question 1 [8 Marks]
**State and explain Flynn's Taxonomy of parallel computer architectures. Provide a neat architectural diagram and a real-world computing example for each category.**

#### Evaluation Rubric:
- Definition & criterion of classification (2 Marks)
- Description of 4 classes with diagrams (4 Marks)
- Real-world hardware examples for each (2 Marks)

#### Model Answer:
Flynn's Taxonomy (proposed by Michael J. Flynn in 1972) classifies all computer systems based on the multiplicity of hardware **Instruction Streams** and **Data Streams** active at any given instant.

```
                        INSTRUCTION STREAMS
                      Single             Multiple
                +-------------------+-------------------+
       Single   |       SISD        |       MISD        |
DATA            | (Von Neumann CPU) | (Systolic Array)  |
STREAMS         +-------------------+-------------------+
      Multiple  |       SIMD        |       MIMD        |
                | (Vector / GPUs)   | (Multicore / HPC) |
                +-------------------+-------------------+
```

1. **SISD (Single Instruction, Single Data):**
   - **Mechanism:** A single control unit fetches a single instruction stream from memory, and a single arithmetic logic unit (ALU) operates on a single data stream stored in memory.
   - **Diagram:** `[Control Unit] ---> [ALU / Processor] <---> [Memory]`
   - **Example:** Classic single-core Von Neumann processors (e.g., Intel 8086, early Pentium).

2. **SIMD (Single Instruction, Multiple Data):**
   - **Mechanism:** A single central control unit broadcasts one machine instruction to multiple parallel processing elements. Each processing element executes that identical instruction synchronously on distinct data elements.
   - **Diagram:**
     ```
     [Control Unit] ======(Broadcasts 1 Instruction)======>
           |                   |                   |
           v                   v                   v
        [ALU 0]             [ALU 1]             [ALU 2]
           ^                   ^                   ^
           | (Data 0)          | (Data 1)          | (Data 2)
       [Data Mem 0]        [Data Mem 1]        [Data Mem 2]
     ```
   - **Example:** Modern vector extensions (Intel AVX-512, ARM NEON) and GPU Streaming Multiprocessors (SMs) executing warp instructions.

3. **MISD (Multiple Instruction, Single Data):**
   - **Mechanism:** Multiple independent processing units execute distinct instructions simultaneously on the same single data stream.
   - **Diagram:** `[Data Stream] ---> [ALU 1] ---> [ALU 2] ---> [ALU 3] ---> Result`
   - **Example:** Highly specialized systolic arrays, pipeline image processors, and redundant space-shuttle flight-control computers (voting systems).

4. **MIMD (Multiple Instruction, Multiple Data):**
   - **Mechanism:** Multiple independent processors execute distinct instruction streams asynchronously on distinct data sets.
   - **Diagram:**
     ```
     [Control Unit 0] -> [ALU 0] <-> [Memory 0]
     [Control Unit 1] -> [ALU 1] <-> [Memory 1]
     [Control Unit 2] -> [ALU 2] <-> [Memory 2]
     ```
   - **Example:** Modern multi-core x86/ARM processors (Intel Xeon, AMD EPYC), supercomputing clusters, and distributed cloud datacenters.

---

### Question 2 [10 Marks]
**(a) Compare Uniform Memory Access (UMA) and Cache-Coherent Non-Uniform Memory Access (ccNUMA) architectures with diagrams.**  
**(b) Explain why NUMA-awareness is essential for multicore performance scaling and describe the "first-touch" memory allocation policy.**

#### Evaluation Rubric:
- (a) Definitions, diagrams, and comparison of UMA vs ccNUMA (5 Marks)
- (b) NUMA performance impact, inter-socket crossbar latency, and First-Touch policy explanation (5 Marks)

#### Model Answer:

**(a) UMA vs. ccNUMA Architectures:**

```
UMA (Symmetric Multiprocessing):
+--------+  +--------+  +--------+  +--------+
| Core 0 |  | Core 1 |  | Core 2 |  | Core 3 |
+---+----+  +---+----+  +---+----+  +---+----+
    |           |           |           |
====+===========+===========+===========+==== System Bus / Crossbar
                      |
              +-------+-------+
              | Shared Memory | (Equal Latency to all Cores)
              +---------------+

ccNUMA (Non-Uniform Memory Access):
   [ SOCKET 0 / NUMA Node 0 ]                 [ SOCKET 1 / NUMA Node 1 ]
+--------+  +--------+  +--------+         +--------+  +--------+  +--------+
| Core 0 |  | Core 1 |  | Core 2 |         | Core 3 |  | Core 4 |  | Core 5 |
+---+----+  +---+----+  +---+----+         +---+----+  +---+----+  +---+----+
    \___________|___________/                  \___________|___________/
                |                                          |
        +-------+-------+                          +-------+-------+
        | Local Memory  | <=== Inter-Socket ====>  | Local Memory  |
        |    (DRAM 0)   |       Interconnect       |    (DRAM 1)   |
        +---------------+      (UPI / Infinity)    +---------------+
```

- **UMA:** All processors share physical memory through a centralized bus. Access latency ($\tau_{\text{mem}}$) and bandwidth are uniform for every core. However, as core counts exceed 8–16, the shared bus becomes saturated, creating an unscalable bottleneck.
- **ccNUMA:** Physical memory is divided into independent domains, each directly attached to a CPU socket. Hardware maintains cache coherency across domains. Accessing local DRAM exhibits low latency ($\approx 70\text{ ns}$), while accessing remote DRAM across the inter-socket interconnect (UPI/Infinity Fabric) incurs a 2x–3x latency penalty ($\approx 150\text{ ns}$) and constrained bandwidth.

**(b) NUMA Awareness and the First-Touch Policy:**
- **Why NUMA Awareness Matters:** If OpenMP threads allocated on Socket 1 frequently read and write data residing in Socket 0's memory, memory traffic saturates the inter-socket link, cutting effective memory bandwidth by up to 60%.
- **First-Touch Policy:** In Linux, physical DRAM pages are not allocated when `malloc()` is executed; `malloc()` merely reserves virtual address space. The physical page is mapped to the NUMA domain of the specific CPU core that **first writes to that memory page**.
- **Engineering Rule:** To achieve NUMA locality, data arrays must be initialized in parallel using the exact same OpenMP loop scheduling and thread mapping as will be used in the subsequent compute phase:
  ```c
  #pragma omp parallel for schedule(static)
  for (int i = 0; i < N; i++) {
      A[i] = 0.0; /* Thread that computes element i also first-touches it! */
  }
  ```

---

### Question 3 [10 Marks]
**(a) State and mathematically derive Amdahl's Law.**  
**(b) State and mathematically derive Gustafson's Law.**  
**(c) Contrast Strong Scaling and Weak Scaling with an illustrative graph.**

#### Evaluation Rubric:
- (a) Mathematical derivation and asymptotic limit of Amdahl's Law (3 Marks)
- (b) Mathematical derivation of Gustafson's Law (3 Marks)
- (c) Strong vs Weak scaling definition, comparison, and scaling graph (4 Marks)

#### Model Answer:

**(a) Derivation of Amdahl's Law (Strong Scaling):**
Let $T(1)$ be sequential execution time. Suppose a fraction $s$ ($0 \le s \le 1$) is strictly serial, and the remaining fraction $p = 1 - s$ can be parallelized across $N$ processors with linear efficiency. The total problem workload is fixed.
The execution time on $N$ processors is:
$$T(N) = s \cdot T(1) + \frac{(1 - s) \cdot T(1)}{N}$$
Speedup $S(N)$ is defined as $\frac{T(1)}{T(N)}$:
$$S_{\text{Amdahl}}(N) = \frac{T(1)}{s \cdot T(1) + \frac{1 - s}{N} \cdot T(1)} = \frac{1}{s + \frac{1 - s}{N}}$$
Taking the asymptotic limit as $N \to \infty$:
$$\lim_{N \to \infty} S_{\text{Amdahl}}(N) = \frac{1}{s}$$
*Significance:* If only $5\%$ of code is serial ($s = 0.05$), maximum possible speedup on infinite processors is $20\times$.

**(b) Derivation of Gustafson's Law (Weak Scaling):**
Gustafson observed that in real HPC, problem size scales with processor count to keep execution time constant.
Let parallel execution time on $N$ processors be normalized to $T(N) = 1$. Let $s$ be the measured serial fraction on the parallel machine, and $1 - s$ be the parallel fraction ($s + (1 - s) = 1$).
To run this scaled problem on a single processor, the serial part takes $s$, while the parallel part requires $(1 - s) \times N$:
$$T(1) = s + (1 - s) \cdot N$$
Speedup $S_{\text{Gustafson}}(N)$ is:
$$S_{\text{Gustafson}}(N) = \frac{T(1)}{T(N)} = \frac{s + (1 - s) \cdot N}{1} = s + (1 - s) \cdot N = N + s(1 - N)$$
*Significance:* Scaled speedup grows linearly with $N$, demonstrating that massively parallel computing is viable.

**(c) Strong Scaling vs. Weak Scaling:**
- **Strong Scaling:** Problem size is fixed; processor count $N$ increases; goal is minimizing time-to-solution. Governed by Amdahl's Law; speedup plateaus at $1/s$.
- **Weak Scaling:** Work per processor is fixed; total problem size grows linearly with $N$; goal is solving larger scientific models. Governed by Gustafson's Law; speedup scales linearly.

```
Speedup
  ^
  |                                   / Gustafson's Law (Weak Scaling)
  |                                  /  (Linear Scaled Speedup)
  |                                 /
  |                                /
  |                      ---------+-- Amdahl's Law (Strong Scaling)
  |                 . - '   Asymptotic Limit = 1/s
  |            . - '
  |       . - '
  +---------------------------------------------> Number of Processors (N)
```

---

### Question 4 [8 Marks]
**Explain the Seven Dwarfs of numerical methods defined by UC Berkeley. Why are they significant in HPC hardware evaluation?**

#### Model Answer:
The "Seven Dwarfs" (formulated by Asanović et al. at UC Berkeley) are seven algorithmic patterns that capture the computational, memory access, and communication characteristics of scientific and engineering computing:

1. **Dense Linear Algebra:** Operations on dense matrices and vectors (BLAS 1/2/3, LAPACK). Compute-bound; characterized by regular unit-stride memory access and high arithmetic intensity ($O(N^3)$ operations on $O(N^2)$ data).
2. **Sparse Linear Algebra:** Operations on matrices populated primarily by zeros (SpMV, CG solvers). Memory-bandwidth bound; characterized by indirect array indexing (`A[col[i]]`) and low arithmetic intensity.
3. **Spectral Methods (FFT):** Operations in the frequency domain (1D/2D/3D Fast Fourier Transforms). Characterized by all-to-all communication patterns, butterfly shuffle stages, and non-local memory strides.
4. **N-Body Methods:** Simulating interactions between all pairs of discrete particles (gravitational, electrostatic). Computational complexity is $O(N^2)$ (direct) or $O(N \log N)$ (hierarchical Barnes-Hut / Fast Multipole Method).
5. **Structured Grids:** Computations on regular multi-dimensional arrays where grid points are updated based on a local spatial neighborhood (stencils). Characterized by high spatial locality and regular halo exchanges.
6. **Unstructured Grids:** Computations on irregular geometric meshes (finite elements, aerodynamics). Characterized by pointer chasing, irregular gather/scatter operations, and unpredictable cache performance.
7. **MapReduce / Monte Carlo:** Embarrassingly parallel, independent calculations with zero inter-task communication, followed by an associative reduction of results. High parallel efficiency.

**Significance in HPC:** Instead of optimizing architectures for hundreds of individual scientific codes, hardware architects and benchmark suites (SPEC, CORAL) evaluate processors against these 7 patterns, guaranteeing that hardware enhancements translate across diverse computational sciences.

---

### Question 5 [6 Marks]
**What is the Roofline Model? Define Arithmetic Intensity. Explain how the model identifies whether a kernel is compute-bound or memory-bound.**

#### Model Answer:
The **Roofline Model** is an intuitive visual performance model that bounds a program's attainable floating-point performance ($P_{\text{attainable}}$) on a given multicore processor as a function of its **Arithmetic Intensity**.

```
Attainable Performance (FLOP/s)
  ^
  |                  PEAK COMPUTE PERFORMANCE (R_peak)
  |               +--------------------------------------- (Compute-Bound Ceiling)
  |              / 
  |             /  
  |            /   
  |           /    SLOPE = Memory Bandwidth (B)
  |          /     (Memory-Bound Ceiling)
  |         /
  |        /
  |       /  |
  +-------+--+--------------------------------------------> Arithmetic Intensity
          0  I_crit                                         (FLOPs/Byte)
```

1. **Arithmetic Intensity ($I$):**
   The ratio of total floating-point operations executed to total bytes transferred between CPU cache and main DRAM:
   $$I = \frac{\text{Floating-Point Operations (FLOPs)}}{\text{DRAM Data Movement (Bytes)}} \quad \left[\frac{\text{FLOPs}}{\text{Byte}}\right]$$

2. **The Roofline Formula:**
   $$P_{\text{attainable}} = \min\left( R_{\text{peak}}, \; I \times B_{\text{memory}} \right)$$

3. **Classification:**
   - **Machine Balance Point ($I_{\text{crit}}$):** $I_{\text{crit}} = \frac{R_{\text{peak}}}{B_{\text{memory}}}$.
   - **Memory-Bound Regime ($I < I_{\text{crit}}$):** The kernel lies on the slanted roof ($P = I \times B$). Performance is limited by memory bandwidth. Increasing CPU clock speed or vector units yields zero speedup. Optimization requires cache blocking, loop tiling, and register reuse.
   - **Compute-Bound Regime ($I \ge I_{\text{crit}}$):** The kernel lies on the horizontal flat roof ($P = R_{\text{peak}}$). Performance is limited by arithmetic hardware pipelines. Optimization requires vectorization (SIMD/AVX-512) and instruction pipelining.

---

## 2. OpenMP Foundations, Work-Sharing & Synchronization (Q6 – Q11)

### Question 6 [8 Marks]
**Explain the Fork-Join Execution Model in OpenMP with an architectural diagram. Describe the runtime behavior when entering and exiting a `#pragma omp parallel` construct.**

#### Model Answer:
OpenMP applications follow the **Fork-Join Execution Model**:

```
Master Thread
============== (Sequential Execution)
      |
      | encounters #pragma omp parallel
      v
    FORK ===== (Team of Threads Created / Activated)
    +---------------+---------------+---------------+
    |               |               |               |
    v               v               v               v
 [Thread 0]     [Thread 1]     [Thread 2]     [Thread 3]
 (Master)        (Worker)       (Worker)       (Worker)
    |               |               |               |
    |               |               |               |
    v               v               v               v
    JOIN ===== (Implicit Barrier Synchronization)
    +---------------+---------------+---------------+
      |
      v
Master Thread resumes sequential execution
```

1. **Sequential Initial State:** The application begins execution as a single thread of execution, termed the **Initial/Master Thread**.
2. **The Fork Phase:** Upon encountering `#pragma omp parallel`:
   - The master thread queries the runtime for the requested thread count $T$ (via `num_threads` clause, `omp_set_num_threads()`, or `OMP_NUM_THREADS`).
   - A team of $T$ threads is created (or recruited from an existing dormant thread pool to avoid OS thread-creation overhead).
   - Each thread is assigned a unique integer ID (`0` to $T-1$), where Thread 0 is the master thread.
   - The enclosed structured block of code is replicated and executed concurrently by all $T$ threads.
3. **The Join Phase:** At the closing brace `}` of the parallel construct:
   - An **implicit barrier synchronization** is automatically executed; every thread must reach this point before any thread can proceed.
   - The worker threads are synchronized, suspended, and returned to the thread pool.
   - Only the master thread continues executing the subsequent serial code.

---

### Question 7 [8 Marks]
**Explain the differences between `shared`, `private`, and `firstprivate` data scoping clauses in OpenMP. Why is `default(none)` considered an essential software engineering practice?**

#### Model Answer:

| Scoping Clause | Memory Allocation Location | Initialization Behavior | Value upon Construct Exit |
| :--- | :--- | :--- | :--- |
| **`shared(var)`** | Global heap or master stack. All threads share one physical address. | Retains original value prior to parallel region. | Any modifications made by any thread persist globally. |
| **`private(var)`** | Allocated locally on each thread's private execution stack. | **Uninitialized** upon entry (contains garbage memory values). | **Undefined** after parallel region; original variable is unchanged. |
| **`firstprivate(var)`**| Allocated locally on each thread's private stack. | Automatically initialized with master thread's value prior to fork. | **Undefined** after parallel region; original variable is unchanged. |

**Why `default(none)` is an Essential Best Practice:**
By default, OpenMP uses implicit scoping rules: variables defined outside a parallel region are implicitly scoped as `shared` (except loop index variables).
- **The Risk:** If a programmer uses a temporary local variable (such as an index `j` or accumulator `temp`) without explicitly declaring it `private`, OpenMP silently scopes it as `shared`. All threads write concurrently to that single memory address, causing catastrophic data races that produce silent numerical corruption.
- **The Solution:** Specifying `default(none)` disables all implicit scoping heuristics. The compiler refuses to compile the program and raises a compilation error if any referenced variable lacks an explicit scoping clause (`shared`, `private`, `firstprivate`, or `reduction`). This catches race conditions at compile time.

---

### Question 8 [10 Marks]
**Compare the four loop scheduling strategies in OpenMP: `static`, `dynamic`, `guided`, and `auto`. Provide their syntax, chunk assignment mechanisms, and recommended workload characteristics.**

#### Model Answer:

| Scheduling Strategy | Syntax | Chunk Allocation Mechanism | Runtime Overhead | Ideal Workload Profile |
| :--- | :--- | :--- | :--- | :--- |
| **Static** | `schedule(static, chunk)` | Iterations are divided into fixed chunks and assigned round-robin at compile/launch time. | **Negligible** (No locks or queues). | Perfectly balanced, uniform loops (e.g., dense vector additions, matrix multiplication). |
| **Dynamic** | `schedule(dynamic, chunk)` | Chunks are placed in a shared runtime queue. Threads lock the queue and pull a chunk dynamically upon finishing previous work. | **High** (Atomic queue synchronization per chunk). | Highly irregular, unpredictable loop execution times (e.g., Mandelbrot set, ray tracing). |
| **Guided** | `schedule(guided, min_chunk)` | Chunk size decreases exponentially over time ($Chunk \approx \frac{\text{Remaining}}{2 \cdot T}$). Threads pull smaller chunks as the loop nears completion. | **Moderate** (Fewer synchronization events than dynamic). | Irregular workloads where large initial chunks amortize overhead and fine final chunks prevent load imbalance. |
| **Auto** | `schedule(auto)` | The compiler and runtime library are granted total freedom to choose the optimal scheduling policy. | Implementation dependent. | Portable code where compiler optimizations are tuned to target architecture. |

---

### Question 9 [8 Marks]
**Differentiate between `#pragma omp critical` and `#pragma omp atomic`. What are the performance and architectural implications of each?**

#### Model Answer:

| Feature | `#pragma omp critical` | `#pragma omp atomic` |
| :--- | :--- | :--- |
| **Scope of Protection** | Can protect an arbitrary multi-line block of C statements, function calls, and pointer updates. | Protects strictly **a single basic read-modify-write operation** (e.g., `x += expr`, `x++`). |
| **Underlying Mechanism** | Implements software mutual exclusion using general runtime mutex locks (`pthread_mutex`). | Implements hardware-level atomic instructions supported directly by CPU instruction sets. |
| **Hardware Instructions** | Software lock/unlock function calls (`libgomp` mutex calls). | Emits CPU hardware bus lock instructions (e.g., x86 `LOCK XADD`, `CMPXCHG`). |
| **Thread Concurrency** | Completely serializes execution; all other threads block and sleep. | High hardware throughput; cache coherency hardware handles pipelining. |
| **Performance Overhead** | High ($\approx 50\text{ ns} - 200\text{ ns}$ per invocation). | Ultra-low ($\approx 5\text{ ns} - 15\text{ ns}$ per invocation). |

**Architectural Recommendation:** Whenever an operation can be expressed as a single binary update (`x += val`), always prefer `atomic` over `critical`. For global aggregations, `reduction` is superior to both because it eliminates synchronization entirely during the loop.

---

### Question 10 [10 Marks]
**(a) What is False Sharing in shared-memory multiprocessors? Explain its root microarchitectural cause using the MESI cache coherency protocol.**  
**(b) Write a C code snippet illustrating false sharing and show how to eliminate it.**

#### Model Answer:

**(a) Root Microarchitectural Cause:**
False sharing occurs when two or more threads running on different physical CPU cores modify independent variables that happen to reside within the **same physical cache line** (typically 64 bytes).
- Even though the threads do not logically access the same variable, the hardware cache coherency protocol (such as MESI: Modified, Exclusive, Shared, Invalid) operates at the **granularity of entire cache lines**, not individual bytes.
- When Core 0 writes to Variable A, Core 0's cache line transitions to the **Modified (M)** state, which invalidates the identical 64-byte cache line in Core 1's L1/L2 cache (transitioning it to **Invalid (I)**).
- When Core 1 attempts to write to Variable B, it suffers a cache miss, forcing an expensive bus snooping transaction and cache-line transfer across the interconnect.
- The cache line continuously bounces between cores ("cache line ping-ponging"), stalling execution pipelines and causing a $5\times - 20\times$ performance slowdown.

```
+---------------------------------------------------------------+
|               64-Byte Physical Cache Line                     |
+-------------------------------+-------------------------------+
|  sum[0] (Modified by Core 0)  |  sum[1] (Modified by Core 1)  |
+-------------------------------+-------------------------------+
           |                                   |
    Core 0 writes (Invalidates line)     Core 1 writes (Invalidates line)
           \----------------- PING-PONG ----------------/
```

**(b) Flawed Code vs. Solution:**

*Flawed Code (Severe False Sharing):*
```c
double sum[MAX_THREADS]; /* Elements are contiguous in memory! */
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    for (int i = tid; i < N; i += num_threads) {
        sum[tid] += A[i]; /* Multiple cores writing to same 64-byte line! */
    }
}
```

*Solution (Thread-Local Accumulator):*
```c
double global_sum = 0.0;
#pragma omp parallel
{
    double local_sum = 0.0; /* Allocated on thread's private stack! */
    #pragma omp for
    for (int i = 0; i < N; i++) {
        local_sum += A[i];
    }
    #pragma omp atomic
    global_sum += local_sum; /* Single atomic update at loop end */
}
```

---

### Question 11 [6 Marks]
**Explain the OpenMP `reduction` clause. Why must reduction operators possess mathematical associativity and commutativity?**

#### Model Answer:
The OpenMP `reduction(op:list)` clause accumulates values across threads without synchronization overhead:
1. **Local Accumulation:** The OpenMP runtime allocates a private copy of the reduction variable on each thread's stack, automatically initialized to the operator's mathematical identity ($0$ for `+`, $1$ for `*`).
2. **Lock-Free Execution:** Each thread updates its private accumulator during the loop without mutexes or atomics.
3. **Tree Reduction:** Upon loop completion, the runtime combines all thread-local accumulators into the original shared variable.

**Mathematical Requirements:**
- **Associativity:** $(a \oplus b) \oplus c = a \oplus (b \oplus c)$. Parallel reduction divides the iteration space arbitrarily and reduces sub-results hierarchically in a tree. Non-associative operations produce inconsistent results depending on chunk distribution.
- **Commutativity:** $a \oplus b = b \oplus a$. Because threads complete non-deterministically due to OS scheduling, sub-results are combined in arbitrary order.
- *Exam Caveat:* IEEE-754 floating-point addition is mathematically associative, but computer floating-point addition is **not associative** due to finite-precision roundoff error ($ (a+b)+c \ne a+(b+c) $). OpenMP parallel reductions on `float`/`double` may yield slight variations in least-significant bits compared to sequential execution.

---

## 3. OpenMP Tasks & SIMD Vectorization (Q12 – Q13)

### Question 12 [8 Marks]
**Contrast Task Parallelism with Data/Loop Parallelism in OpenMP. Under what algorithmic conditions is `#pragma omp task` preferred over `#pragma omp for`?**

#### Model Answer:

| Feature | Data Parallelism (`#pragma omp for`) | Task Parallelism (`#pragma omp task`) |
| :--- | :--- | :--- |
| **Execution Model** | Loop-centric; iteration space is partitioned across a static team of threads. | Graph-centric; discrete work units are packaged and queued dynamically. |
| **Control Flow** | Rigid; requires canonical `for` loops with known iteration counts upfront. | Highly flexible; supports recursive functions, pointer chasing, dynamic graphs. |
| **Data Structure** | Contiguous indexable structures (arrays, dense matrices). | Non-indexable structures (linked lists, binary trees, DAGs). |
| **Thread Assignment**| Threads process assigned iteration chunks synchronously. | Any idle thread in the team steals and executes available tasks from the task pool. |

**When Task Parallelism is Preferred:**
1. **Recursive Algorithms:** Divide-and-conquer algorithms (e.g., Quicksort, recursive Fibonacci, tree traversals) where sub-tasks are spawned dynamically:
   ```c
   #pragma omp task
   quicksort(left, pivot - 1);
   #pragma omp task
   quicksort(pivot + 1, right);
   ```
2. **Pointer-Based Data Structures:** Iterating through linked lists or graph networks where total node count is unknown prior to traversal.
3. **Unbalanced Irregular Computations:** Applications where sub-problems have dramatically varying execution times that cannot be predicted by static/dynamic loop chunking.

---

### Question 13 [6 Marks]
**Explain the operation of `#pragma omp simd`. What is the significance of the `aligned` clause in vector optimization?**

#### Model Answer:
The `#pragma omp simd` directive instructs the compiler to vectorize a loop into hardware SIMD instructions (e.g., AVX2, AVX-512) without spawning multiple threads:
- It asserts to the compiler that loop iterations have **no loop-carried data dependencies** that prevent vectorization.
- The compiler maps multiple iterations to vector registers (e.g., executing 8 double-precision additions in a single 512-bit instruction).

**Significance of the `aligned` Clause:**
Modern CPU architectures achieve maximum memory throughput when data pointers align with hardware cache-line and vector register byte boundaries (e.g., 64-byte alignment for AVX-512).
- If pointers are unaligned, the processor must execute two unaligned memory loads (`_mm512_loadu_pd`), merge the halves, and handle potential split-cache-line penalties.
- Specifying `#pragma omp simd aligned(A, B: 64)` guarantees to the compiler that arrays `A` and `B` start at 64-byte aligned memory addresses, allowing the compiler to emit high-speed aligned vector load instructions (`_mm512_load_pd`), yielding up to a $20\% - 40\%$ speedup.

---

## 4. POSIX Threads & Concurrency Control Primitives (Q14 – Q18)

### Question 14 [8 Marks]
**Compare POSIX Threads (Pthreads) and OpenMP across programming model, abstraction level, programmer responsibilities, and performance trade-offs.**

#### Model Answer:

| Comparison Dimension | POSIX Threads (Pthreads) | OpenMP |
| :--- | :--- | :--- |
| **Standard Level** | Low-level OS library API (`<pthread.h>`). | High-level directive-based language extension (`#pragma omp`). |
| **Thread Management** | Manual: Explicit `pthread_create()`, `pthread_join()`, stack allocation. | Automatic: Managed via compiler pragmas and runtime fork-join pools. |
| **Work Partitioning** | Manual: Programmer must manually calculate loop index bounds per thread. | Automatic: Handled by work-sharing constructs (`#pragma omp for`). |
| **Portability** | Universal across POSIX-certified Unix systems; complex on Windows. | Universally supported across GCC, Clang, Intel, and MSVC compilers. |
| **Overhead** | Minimal runtime overhead; fine-grained control over thread attributes. | Slight abstraction overhead from runtime library (`libgomp`). |
| **Best Suited For** | Long-running asynchronous service threads, complex server daemons, event loops. | Scientific numerical computing, loop-intensive matrix operations, HPC kernels. |

---

### Question 15 [8 Marks]
**(a) Describe the complete lifecycle of a POSIX thread with a state diagram.**  
**(b) Differentiate between Joinable and Detached threads. What is a "zombie thread"?**

#### Model Answer:

**(a) Thread Lifecycle State Diagram:**
```
[Unborn] ---> pthread_create()
                 |
                 v
             [READY] <==== (OS Scheduler Time Slice) ====> [RUNNING]
                 |                                             |
          pthread_mutex_lock()                          Thread Function Returns
          pthread_cond_wait()                           or pthread_exit()
                 |                                             |
                 v                                             v
             [BLOCKED]                                    [TERMINATED]
                 |                                             |
         (Condition Signaled /                                 v
          Mutex Unlocked)                             [ZOMBIE / RESOURCE CLEANUP]
                                                      (Reclaimed via pthread_join
                                                       or auto-reclaimed if detached)
```

**(b) Joinable vs. Detached Threads & Zombie Threads:**
- **Joinable Thread (Default):** The OS retains the thread's exit code, return value pointer, and kernel data structures after termination until another thread invokes `pthread_join(tid, &retval)`.
- **Detached Thread:** The OS automatically and immediately reclaims all memory and kernel thread table structures the instant the thread terminates. Created via `pthread_detach(tid)` or `pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)`.
- **Zombie Thread:** A joinable thread that has completed execution but has not been joined by any parent thread. It continues to consume operating system thread table entries and stack memory. If an application repeatedly spawns joinable threads without joining them, it leaks resources until the OS thread table is exhausted (`EAGAIN: Resource temporarily unavailable`).

---

### Question 16 [10 Marks]
**Explain the Condition Variable synchronization mechanism in Pthreads. Why must `pthread_cond_wait()` always release the mutex atomically, and why must it always be evaluated inside a `while` loop?**

#### Model Answer:
A Condition Variable (`pthread_cond_t`) is a signaling primitive that enables threads to sleep without consuming CPU cycles until an application-defined predicate becomes true.

**The Canonical Condition Variable Pattern:**
```c
pthread_mutex_lock(&mutex);
while (!predicate_is_true) {
    pthread_cond_wait(&cond, &mutex);
}
/* Execute critical section safely */
pthread_mutex_unlock(&mutex);
```

**Why Mutex Release Must Be Atomic:**
`pthread_cond_wait(&cond, &mutex)` performs two actions in a single indivisible hardware operation:
1. It unlocks the associated `mutex`.
2. It suspends the calling thread on the condition variable's wait queue.
- *The Fatal Danger (The Lost Wakeup Problem):* If these operations were not atomic, a thread could unlock the mutex, get preempted by the OS before enrolling on the wait queue, and allow a signaling thread to run. The signaling thread modifies the state and signals the condition. When the waiting thread resumes, it enters the wait queue, missing the signal entirely and sleeping forever (deadlock).

**Why the `while` Loop Pattern is Mandatory:**
A condition variable check must **never** use an `if` statement:
1. **Spurious Wakeups:** The operating system kernel and POSIX standard permit condition variables to awaken occasionally without an explicit signal. A `while` loop re-evaluates the predicate upon wakeup.
2. **Stolen State (Mesa Semantics):** When a signaling thread issues `pthread_cond_signal()`, a waiting thread is unblocked, but it does not run immediately; it must wait to re-acquire the mutex. Before it acquires the mutex, another thread may intervene, acquire the lock, and invalidate the condition (e.g., consume the buffer item). The `while` loop ensures the woken thread re-checks the predicate before proceeding.

---

### Question 17 [10 Marks]
**Compare Mutexes, Semaphores, Read-Write Locks, and Spinlocks. Provide a decision matrix for selecting the appropriate synchronization primitive in concurrent software design.**

#### Model Answer:

| Primitive | Operating System Mechanism | Ownership Property | Resource Blocking Type | Recommended Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Mutex** (`pthread_mutex_t`) | Sleep-lock; unscheduled by OS kernel into wait queue. | **Strict Ownership:** Only the thread that locked the mutex can unlock it. | Binary (0 or 1 thread). | General-purpose mutual exclusion for multi-statement critical sections. |
| **Semaphore** (`sem_t`) | Atomic integer counter with kernel blocking queue. | **No Ownership:** Any thread can signal (`sem_post`) a semaphore. | Counting ($0 \dots N$ resources). | Producer-Consumer bounded buffers, signaling events, resource pools. |
| **Read-Write Lock** (`pthread_rwlock_t`) | Shared read locks; exclusive write locks. | Ownership enforced for writers. | Concurrent readers OR single exclusive writer. | Read-heavy workloads (e.g., routing tables, in-memory caches, lookup databases). |
| **Spinlock** (`pthread_spinlock_t`) | User-space busy-waiting; tight CPU polling loop. | Thread that locked must unlock. | Binary. | Ultra-short critical sections on dedicated multiprocessors ($t_{\text{wait}} < 2\text{ \textmu s}$). |

---

### Question 18 [8 Marks]
**Write the complete pseudocode/C code solving the Bounded-Buffer Producer-Consumer problem using POSIX counting semaphores and a mutex. Explain how deadlock is prevented.**

#### Model Answer:

```c
#include <pthread.h>
#include <semaphore.h>

#define BUFFER_SIZE 8

int buffer[BUFFER_SIZE];
int in = 0, out = 0;

sem_t empty_slots;  /* Tracks available empty buffer slots (init: BUFFER_SIZE) */
sem_t full_slots;   /* Tracks filled buffer slots (init: 0) */
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void producer(int item) {
    sem_wait(&empty_slots);         /* Wait for available slot */
    pthread_mutex_lock(&mutex);      /* Enter critical section */
    
    buffer[in] = item;
    in = (in + 1) % BUFFER_SIZE;
    
    pthread_mutex_unlock(&mutex);    /* Exit critical section */
    sem_post(&full_slots);           /* Signal new item available */
}

int consumer(void) {
    int item;
    sem_wait(&full_slots);           /* Wait for available item */
    pthread_mutex_lock(&mutex);      /* Enter critical section */
    
    item = buffer[out];
    out = (out + 1) % BUFFER_SIZE;
    
    pthread_mutex_unlock(&mutex);    /* Exit critical section */
    sem_post(&empty_slots);          /* Signal freed slot */
    return item;
}
```

**Deadlock Prevention Explanation:**
Notice the ordering of locks: `sem_wait()` is called **before** `pthread_mutex_lock()`.
- If a programmer mistakenly reverses this order (`pthread_mutex_lock(&mutex)` before `sem_wait(&empty_slots)`), a catastrophic deadlock occurs when the buffer is full:
  - The producer acquires the mutex, then blocks on `sem_wait(&empty_slots)`.
  - The consumer attempts to read an item to free a slot, but blocks forever trying to acquire the mutex held by the sleeping producer!
- Calling `sem_wait()` first ensures that threads only contend for the mutex when a valid buffer slot is guaranteed to exist.

---

## 5. Distributed Memory, MPI & Hybrid Programming (Q19 – Q24)

### Question 19 [8 Marks]
**Explain the Distributed-Memory Message Passing Model. Define the roles of Communicators (`MPI_Comm`), Process Rank, and Process Size in MPI.**

#### Model Answer:
In the **Distributed-Memory Message Passing Model**, an application consists of multiple autonomous processes executing on distinct physical compute nodes:
- Each process possesses its own private, isolated virtual address space.
- A process cannot directly read or write variables stored in another process's memory.
- All inter-process communication, coordination, and data movement occurs exclusively via explicit message transmission across a network interconnect (e.g., InfiniBand, Ethernet).

**Fundamental MPI Concepts:**
1. **Communicator (`MPI_Comm`):** An opaque handle representing an isolated communication universe that encapsulates a specific group of processes and a unique communication context. It ensures that message traffic generated in one algorithmic library cannot be intercepted or confused with messages in another library. `MPI_COMM_WORLD` is the predefined default communicator encompassing all launched processes.
2. **Process Rank:** A unique, zero-indexed integer identifier ($0, 1, \dots, \text{Size}-1$) assigned to each process within a communicator. Obtained via `MPI_Comm_rank(comm, &rank)`. Programs use conditional branching on rank (`if (rank == 0)`) to implement Master-Worker patterns.
3. **Process Size:** The total number of processes participating in a communicator. Obtained via `MPI_Comm_size(comm, &size)`. Used to calculate domain decomposition boundaries.

---

### Question 20 [10 Marks]
**(a) Differentiate between Blocking and Non-Blocking communication in MPI.**  
**(b) Explain how non-blocking calls (`MPI_Isend`, `MPI_Irecv`) combined with `MPI_Wait` facilitate computation-communication overlap (latency hiding).**

#### Model Answer:

**(a) Blocking vs. Non-Blocking Communication:**

| Feature | Blocking (`MPI_Send` / `MPI_Recv`) | Non-Blocking (`MPI_Isend` / `MPI_Irecv`) |
| :--- | :--- | :--- |
| **Return Semantics** | Returns only when the user's buffer can be safely reused or modified. | Returns **immediately**, before the data transfer has completed. |
| **Tracking Handle** | None; completion is implicit upon return. | Returns an opaque request handle (`MPI_Request`). |
| **CPU Concurrency** | CPU is blocked or idles during network packet transmission. | CPU returns to execution immediately, enabling concurrent local compute. |
| **Completion Check** | Not required. | Mandatory via explicit calls to `MPI_Wait()` or `MPI_Test()`. |
| **Deadlock Risk** | High risk of cyclic circular wait deadlocks. | Extremely low; initiates both directions concurrently. |

**(b) Computation-Communication Overlap (Latency Hiding):**

```
Blocking Execution:
[ Send/Recv Comm (100 us) ] ---> [ Compute Interior Grid (150 us) ] = 250 us Total

Non-Blocking Overlapped Execution:
[ MPI_Isend / Irecv ]
|--- Background Network Transfer (100 us) ---|
+--------------------------------------------+
| Compute Interior Grid (150 us)             | ---> [ MPI_Wait ] = 150 us Total
+--------------------------------------------+
```

By initiating non-blocking boundary exchanges (`MPI_Isend`/`MPI_Irecv`), the network interface card (NIC) handles packet routing via Direct Memory Access (DMA) in the background while the CPU concurrently updates the interior numerical mesh. When the CPU reaches `MPI_Wait()`, the network transfer is already finished, completely eliminating communication overhead from the critical path.

---

### Question 21 [12 Marks]
**Compare the following MPI collective communication operations in terms of communication topology, data movement, and root process involvement: `MPI_Bcast`, `MPI_Scatter`, `MPI_Gather`, `MPI_Allgather`, `MPI_Reduce`, `MPI_Allreduce`, and `MPI_Barrier`.**

#### Model Answer:

```
MPI_Bcast:      [ D0 ] -> Root transmits identical buffer D0 to all ranks
MPI_Scatter:    [ D0 | D1 | D2 | D3 ] -> Root splits array; sends D_i to Rank i
MPI_Gather:     Rank i sends D_i -> Root concatenates [ D0 | D1 | D2 | D3 ]
MPI_Allgather:  Every rank contributes D_i -> Every rank receives [ D0 | D1 | D2 | D3 ]
MPI_Reduce:     Rank i contributes D_i -> Root receives D0 (+) D1 (+) D2 (+) D3
MPI_Allreduce:  Rank i contributes D_i -> Every rank receives D0 (+) D1 (+) D2 (+) D3
MPI_Barrier:    Zero data transferred -> Temporal execution alignment across all ranks
```

| Collective Routine | Direction | Data Movement Pattern | Root Process Required? | Data Transferred per Rank |
| :--- | :--- | :--- | :---: | :--- |
| **`MPI_Bcast`** | One-to-All | Identical buffer replicated from root to all processes. | **Yes** | $m$ bytes |
| **`MPI_Scatter`** | One-to-All | Contiguous array partitioned into equal chunks; chunk $i$ sent to rank $i$. | **Yes** | $m / P$ bytes |
| **`MPI_Gather`** | All-to-One | Chunks collected from each rank and concatenated at root in rank order. | **Yes** | $m / P$ bytes |
| **`MPI_Allgather`** | All-to-All | Every process gathers chunks; all processes receive the full assembled array. | **No** | $m$ bytes |
| **`MPI_Reduce`** | All-to-One | Data collected and reduced via associative operator; result stored at root. | **Yes** | Scalar / array result |
| **`MPI_Allreduce`** | All-to-All | Data collected, reduced via operator; identical result stored at **all** ranks. | **No** | Scalar / array result |
| **`MPI_Barrier`** | Synchronization | No data transferred; all processes block until all have called the routine. | **No** | **0 bytes** |

---

### Question 22 [8 Marks]
**Explain Dynamic Process Management in MPI. Describe the workflow by which two independent MPI applications connect dynamically using ports, published service names, and `ompi-server`.**

#### Model Answer:
MPI-2 introduced **Dynamic Process Management**, enabling independently launched MPI applications (or dynamic clients) to establish communication channels at runtime without sharing a common initial `MPI_COMM_WORLD`.

```
 SERVER APPLICATION                           CLIENT APPLICATION
====================                         ====================
 1. MPI_Open_port()
    (Generates Port String)
 2. MPI_Publish_name() === [ompi-server] <=== 3. MPI_Lookup_name()
    (Publishes "Service")   (Name Server)        (Discovers Port String)
 4. MPI_Comm_accept() <====================== 5. MPI_Comm_connect()
         \                                          /
          \=======> [ INTERCOMMUNICATOR ] <========/
                    (Two-way cross-group messaging)
```

**Step-by-Step Connection Workflow:**
1. **Name Service Startup:** A central discovery daemon (`ompi-server`) is started on the network cluster, maintaining a global registry of published service names and network ports.
2. **Server Port Creation (`MPI_Open_port`):** The server application calls `MPI_Open_port()` to allocate a dynamic network socket address string.
3. **Publishing Service Name (`MPI_Publish_name`):** The server publishes the port string to `ompi-server` under a human-readable service identifier (e.g., `"HeatSolverService"`).
4. **Client Service Lookup (`MPI_Lookup_name`):** An independently launched client application queries `ompi-server` using `MPI_Lookup_name("HeatSolverService")` to retrieve the server's network port string.
5. **Connection & Intercommunicator Creation:**
   - The server calls `MPI_Comm_accept(port_name, ..., &intercomm)`.
   - The client calls `MPI_Comm_connect(port_name, ..., &intercomm)`.
   - The two applications complete a TCP/InfiniBand handshake, establishing a new **Intercommunicator** that enables direct point-to-point and collective message passing across the separate programs.

---

### Question 23 [8 Marks]
**Explain the four MPI thread safety levels initialized via `MPI_Init_thread()`. What are the programmer constraints and runtime guarantees of each level?**

#### Model Answer:
When combining MPI with multithreading (such as OpenMP or Pthreads), the application must initialize MPI using:
```c
int MPI_Init_thread(int *argc, char ***argv, int required, int *provided);
```

The four standardized thread safety levels are:

1. **`MPI_THREAD_SINGLE`:**
   - **Guarantee:** Pure single-threaded execution.
   - **Constraint:** The application must execute as a single thread. Spawning OpenMP threads is strictly prohibited.

2. **`MPI_THREAD_FUNNELED`:**
   - **Guarantee:** The process may be multithreaded (spawning OpenMP parallel regions).
   - **Constraint:** **Only the master thread** (thread ID 0) is permitted to make MPI calls. All MPI routines must be enclosed within `#pragma omp master` or executed outside parallel regions.

3. **`MPI_THREAD_SERIALIZED`:**
   - **Guarantee:** Any OpenMP thread in the process may make MPI calls.
   - **Constraint:** Multiple threads **must not issue MPI calls concurrently**. The programmer is strictly responsible for serializing access using critical sections (`#pragma omp critical`) or mutexes.

4. **`MPI_THREAD_MULTIPLE`:**
   - **Guarantee:** Full multithreaded concurrency. Any thread may issue MPI calls simultaneously without application-level locking.
   - **Constraint:** The MPI library internally enforces thread safety (using internal mutexes). May incur slight communication latency overhead due to internal lock contention.

---

### Question 24 [10 Marks]
**(a) What is Hybrid MPI + OpenMP computing, and why does it match modern supercomputer architectures?**  
**(b) Explain the importance of thread pinning (affinity), socket-level mapping, and how to avoid physical core over-subscription.**

#### Model Answer:

**(a) The Hybrid Computing Model:**
Modern supercomputers are hierarchically structured: they consist of thousands of discrete compute nodes interconnected by a high-speed network (InfiniBand), where each individual node contains multiple multi-core CPU sockets sharing a common physical memory (NUMA SMP).
- **The Hybrid Paradigm:** Uses **MPI for inter-node communication** across network links, and **OpenMP for intra-node shared-memory multithreading** across CPU cores within each node.
- **Architectural Match:** Eliminates the memory footprint overhead of pure MPI (which duplicates ghost cells, buffer queues, and OS process tables on every single core), while avoiding the scalability limits of pure OpenMP (which cannot cross physical node boundaries).

**(b) Affinity, Mapping, and Over-Subscription Prevention:**

1. **NUMA Socket-Level Mapping:** Modern servers feature 2 to 4 CPU sockets, each with its own local memory controller. Best practice dictates launching **1 MPI process per CPU socket**, and spawning OpenMP worker threads matching the physical core count of that socket ($T = C_{\text{socket}}$). This guarantees that 100% of thread memory accesses hit local DRAM ($f_{\text{local}} = 1.0$).
2. **Thread Pinning (`OMP_PROC_BIND`):** If threads are not bound to cores, the operating system kernel dynamically migrates threads between cores. This destroys L1/L2 cache locality and forces memory accesses across high-latency inter-socket links. Setting `export OMP_PROC_BIND=close` and `export OMP_PLACES=cores` pins each worker thread permanently to its dedicated physical core.
3. **Preventing Core Over-Subscription:**
   - Over-subscription occurs when $M \times T > C_{\text{phys}}$.
   - Running more threads than physical hardware pipelines forces the OS kernel to execute continuous context switches, causing pipeline stalls and cache thrashing.
   - **Golden Rule:** Always enforce:
     $$\text{Ranks per Node } (M_{\text{node}}) \times \text{Threads per Rank } (T_{\text{omp}}) = \text{Physical Cores per Node } (C_{\text{node}})$$

---
*End of Master University Theory Examination Question Bank. All questions provide complete, full-credit model answers.*
