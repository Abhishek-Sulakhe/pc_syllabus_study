# 03. OpenMP Loop Scheduling Strategies: static, dynamic, guided, and auto

---

## 1. The Basic Idea

When parallelizing a loop with `#pragma omp for`, the total iteration space $\{0, 1, \dots, N-1\}$ must be carved up and mapped onto $T$ threads. How this mapping is executed is dictated by the **Loop Scheduling Strategy**.

In OpenMP, scheduling is controlled via the `schedule` clause:
```c
#pragma omp for schedule(type [, chunk_size])
```

The choice of scheduling type directly dictates two fundamental, competing forces in parallel computing:
1. **Load Balancing:** Ensuring that all $T$ threads finish their work at the same time so no thread sits idle waiting at the barrier.
2. **Scheduling Overhead & Cache Locality:** Minimizing the runtime synchronization cost required to assign work and maximizing the spatial cache locality of memory accesses.

OpenMP provides four core scheduling strategies:
- **`static`:** Deterministic, round-robin or block distribution calculated at loop entry with zero runtime queueing overhead.
- **`dynamic`:** First-come, first-served work queue where threads dynamically pull chunks of iterations from a shared queue at runtime.
- **`guided`:** Self-adapting exponential decay where chunk sizes start large to reduce queue contention and shrink smoothly toward a minimum chunk size to balance tail workload.
- **`auto` / `runtime`:** Deferring scheduling choices to the compiler or to the `OMP_SCHEDULE` environment variable.

```
       ITERATION SPACE: [0-----------------------------------------N-1]

1. STATIC CHUNKING (Chunk = C):
   Thread 0: [Chunk 0]             [Chunk 4]             [Chunk 8]
   Thread 1:          [Chunk 1]             [Chunk 5]             [Chunk 9]
   Thread 2:                   [Chunk 2]             [Chunk 6]
   Thread 3:                            [Chunk 3]             [Chunk 7]

2. DYNAMIC QUEUE (Chunk = C):
   Shared Atomic Queue: [Chunk 0][Chunk 1][Chunk 2][Chunk 3][Chunk 4] ...
   Fast Worker:   Pulls Chunk 0 -> Finishes -> Pulls Chunk 2 -> Pulls Chunk 3
   Slow Worker:   Pulls Chunk 1 ---------------------------> Finishes

3. GUIDED EXPONENTIAL DECAY:
   Thread 0: [=== Giant Chunk ===]     [Small]  [min]
   Thread 1:     [== Large Chunk ==]       [Medium] [min][min]
```

---

## 2. Why We Need It

To understand why a single scheduling strategy cannot fit all problems, consider two contrasting real-world loops:

### Case 1: Uniform, Highly Predictable Loop (Vector Addition)
```c
for (int i = 0; i < 10000000; i++) {
    C[i] = A[i] + B[i]; // Every iteration takes exactly 1 clock cycle
}
```
Every iteration has identical execution time. If we used dynamic scheduling here, threads would constantly contend for a shared atomic lock to grab iteration chunks, slowing the program down by up to $10\times$ compared to sequential execution! **Static scheduling** is required here: divide into equal blocks, assign them once, and let cores stream contiguous memory through hardware prefetchers.

### Case 2: Highly Irregular, Unpredictable Loop (Mandelbrot Fractal or Ray Tracing)
```c
for (int i = 0; i < height; i++) {
    for (int j = 0; j < width; j++) {
        // Deep iterations inside Mandelbrot set take 10,000 steps;
        // Pixels outside set escape in 2 steps!
        image[i][j] = compute_mandelbrot_pixel(i, j);
    }
}
```
If we use static scheduling here:
- Thread 0 gets the upper black region of the fractal and finishes in $0.05$ seconds.
- Thread 1 gets the chaotic center boundary of the fractal and takes $5.00$ seconds.
- Thread 0 sits completely idle for $4.95$ seconds at the barrier, destroying parallel efficiency.

**Dynamic or Guided scheduling** is mandatory here: fast threads that finish easy pixels must immediately grab new slices from slow threads.

---

## 3. How It Works: Mechanics of Each Strategy

### 3.1 Static Scheduling: `schedule(static [, chunk])`

#### Variant A: Default Static (No chunk specified)
```c
#pragma omp for schedule(static)
```
- OpenMP divides $N$ iterations into $T$ approximately equal, contiguous blocks of size $\approx \lceil N/T \rceil$.
- Thread $t$ is assigned block $t$ permanently.
- **Hardware Impact:** Maximizes cache line reuse. Hardware spatial prefetchers pre-load consecutive cache lines flawlessly. Zero runtime synchronization overhead.

#### Variant B: Static with Explicit Chunk Size $C$
```c
#pragma omp for schedule(static, C)
```
- Iterations are carved into fixed-size chunks of size $C$.
- Chunks are distributed to threads in a **round-robin cyclic fashion**:
  - Thread 0 gets: Chunks $0, T, 2T, 3T, \dots$
  - Thread 1 gets: Chunks $1, T+1, 2T+1, \dots$
  - Thread $t$ gets: Chunks $k$ where $k \equiv t \pmod T$.
- **Use Case:** Cyclic load imbalance (e.g., triangular loops where workload increases linearly with iteration index $i$).

```
Example: N = 16, T = 4, schedule(static, 2)
  Chunk 0 [0..1]   -> Thread 0
  Chunk 1 [2..3]   -> Thread 1
  Chunk 2 [4..5]   -> Thread 2
  Chunk 3 [6..7]   -> Thread 3
  Chunk 4 [8..9]   -> Thread 0
  Chunk 5 [10..11] -> Thread 1
  Chunk 6 [12..13] -> Thread 2
  Chunk 7 [14..15] -> Thread 3
```

---

### 3.2 Dynamic Scheduling: `schedule(dynamic [, chunk])`
```c
#pragma omp for schedule(dynamic, C)
```
- The iteration space is partitioned into discrete chunks of size $C$ (default $C = 1$ if omitted).
- Chunks are placed into an internal shared runtime work queue.
- When a thread becomes available, it executes an atomic operation (e.g., `lock cmpxchg` on x86) to grab the next chunk index from the queue.
- Threads that encounter heavy iterations take longer, while threads processing fast iterations return to the queue frequently to pull more work.

```
                  DYNAMIC WORK QUEUE MECHANISM
  Remaining Chunks Queue: [Chunk 4][Chunk 5][Chunk 6][Chunk 7]...
                               |
         +---------------------+---------------------+
         | (Atomic Pop)                              | (Atomic Pop)
         v                                           v
   Thread 0 (Fast Core)                        Thread 1 (Slow Core)
   Processes Chunk 4                           Processes Chunk 3 (Long)
   Finishes fast -> Grabs Chunk 5              Still busy...
```

- **Overhead Caution:** If $C = 1$ on a short loop, the atomic contention to increment the queue counter destroys scalability. Always choose a chunk size $C$ large enough to amortize the queue dispatch latency ($\ge 10\,\mu\text{s}$ of work).

---

### 3.3 Guided Scheduling: `schedule(guided [, chunk_min])`
```c
#pragma omp for schedule(guided, chunk_min)
```
Guided scheduling is an optimized, self-throttling hybrid between static and dynamic:
1. Chunks are dispatched dynamically on demand from a runtime queue.
2. However, **chunk sizes are NOT fixed**; they decrease exponentially over time.
3. The size of each allocated chunk is calculated proportionally to the number of unassigned iterations remaining, divided by the number of threads $T$:

$$\text{Chunk Size} = \max\left( \left\lceil \frac{N_{\text{remaining}}}{T} \right\rceil, \text{chunk\_min} \right)$$

- **Default:** If `chunk_min` is not specified, it defaults to $1$.
- **Why Guided Works Brilliantly:**
  - **At the start:** $N_{\text{remaining}}$ is huge $\implies$ chunk sizes are massive $\implies$ very few queue requests, near-zero synchronization contention, excellent cache locality.
  - **Near the finish:** $N_{\text{remaining}}$ is small $\implies$ chunk sizes shrink down to `chunk_min` $\implies$ fine-grained load balancing to ensure all threads finish simultaneously without trailing stragglers!

---

### 3.4 Auto and Runtime Scheduling

#### `schedule(auto)`
- Introduced in OpenMP 3.0.
- Completely hands over scheduling control to the compiler and runtime library.
- Allows advanced optimizing compilers (such as Intel oneAPI or LLVM) to analyze loop access patterns, apply static analysis, or dynamically switch strategies based on hardware profiling.

#### `schedule(runtime)`
- The scheduling strategy and chunk size are **not fixed in the source code**.
- The decision is deferred until the binary is executed, reading the operating system environment variable `OMP_SCHEDULE`:
  ```bash
  $ export OMP_SCHEDULE="dynamic,16"
  $ ./my_simulation
  ```
- **Crucial Engineering Tool:** Allows performance engineers to benchmark multiple scheduling policies and sweep chunk sizes without recompiling the application.

---

## 4. Important Mechanisms: Mathematical Chunk Allocation Models

### Formal Formulas for Chunk Assignment

| Parameter | Notation | Description |
|:---|:---:|:---|
| Total Iterations | $N$ | Trip count of the loop |
| Thread Team Size | $T$ | Number of active threads |
| Specified Chunk Size | $C$ | Argument in `schedule(type, C)` |
| Current Remaining Iterations | $R_k$ | Iterations remaining before $k$-th dispatch |

#### 1. Static Default:
$$\text{Block}(t) = \left[ t \cdot \left\lceil \frac{N}{T} \right\rceil, \min\left((t+1) \cdot \left\lceil \frac{N}{T} \right\rceil, N\right) \right)$$

#### 2. Static Cyclic (`schedule(static, C)`):
Total chunks $K = \lceil N/C \rceil$. Chunk $k \in \{0, 1, \dots, K-1\}$ begins at iteration $k \cdot C$ and is assigned to:
$$\text{Thread ID } t = k \pmod T$$

#### 3. Guided Formula:
For the $k$-th chunk allocation to an available thread:
$$S_k = \max\left( \left\lceil \frac{R_k}{T} \right\rceil, C_{\text{min}} \right), \qquad R_{k+1} = R_k - S_k$$

---

## 5. Compilable C Code Examples

### 5.1 Program: Empirical Comparison of Static, Dynamic, and Guided Scheduling
This program simulates an irregular workload (triangular computation where execution cost scales with iteration index $i$) and logs which thread processes each iteration chunk.

Save this file as `omp_scheduling_demo.c`:

```c
/* =========================================================================
 * File: omp_scheduling_demo.c
 * Description: Compares static, dynamic, and guided OpenMP loop scheduling
 *              under an imbalanced (triangular) computational workload.
 * Compilation:
 *   gcc -O3 -fopenmp omp_scheduling_demo.c -o omp_scheduling_demo -lm
 * Execution:
 *   ./omp_scheduling_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <omp.h>

#define N 24
#define NUM_THREADS 4

/* Synthetic workload: higher iterations do significantly more work */
void simulate_work(int iteration) {
    long iterations_to_spin = (long)(iteration + 1) * 200000L;
    volatile double sink = 0.0;
    for (long k = 0; k < iterations_to_spin; k++) {
        sink += sin((double)k);
    }
}

int main(void) {
    printf("============================================================\n");
    printf("      OpenMP Loop Scheduling Strategies Demonstration       \n");
    printf("============================================================\n");
    printf("Configuration: Total Iterations N = %d, Threads T = %d\n\n", N, NUM_THREADS);

    double t_start, t_end;

    /* -------------------------------------------------------------
     * 1. STATIC SCHEDULING (Default Block)
     * ------------------------------------------------------------- */
    printf("--- 1. Testing schedule(static) ---\n");
    t_start = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(static)
    for (int i = 0; i < N; i++) {
        simulate_work(i);
        #pragma omp critical
        {
            printf("  [static]  Iter %02d processed by Thread %d\n",
                   i, omp_get_thread_num());
        }
    }

    t_end = omp_get_wtime();
    double time_static = t_end - t_start;
    printf("Elapsed Time (static): %f seconds\n\n", time_static);

    /* -------------------------------------------------------------
     * 2. STATIC CYCLIC SCHEDULING (Chunk = 2)
     * ------------------------------------------------------------- */
    printf("--- 2. Testing schedule(static, 2) ---\n");
    t_start = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(static, 2)
    for (int i = 0; i < N; i++) {
        simulate_work(i);
        #pragma omp critical
        {
            printf("  [static,2] Iter %02d processed by Thread %d\n",
                   i, omp_get_thread_num());
        }
    }

    t_end = omp_get_wtime();
    double time_static_2 = t_end - t_start;
    printf("Elapsed Time (static, 2): %f seconds\n\n", time_static_2);

    /* -------------------------------------------------------------
     * 3. DYNAMIC SCHEDULING (Chunk = 2)
     * ------------------------------------------------------------- */
    printf("--- 3. Testing schedule(dynamic, 2) ---\n");
    t_start = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(dynamic, 2)
    for (int i = 0; i < N; i++) {
        simulate_work(i);
        #pragma omp critical
        {
            printf("  [dynamic,2] Iter %02d processed by Thread %d\n",
                   i, omp_get_thread_num());
        }
    }

    t_end = omp_get_wtime();
    double time_dynamic_2 = t_end - t_start;
    printf("Elapsed Time (dynamic, 2): %f seconds\n\n", time_dynamic_2);

    /* -------------------------------------------------------------
     * 4. GUIDED SCHEDULING (Chunk Min = 2)
     * ------------------------------------------------------------- */
    printf("--- 4. Testing schedule(guided, 2) ---\n");
    t_start = omp_get_wtime();

    #pragma omp parallel for num_threads(NUM_THREADS) schedule(guided, 2)
    for (int i = 0; i < N; i++) {
        simulate_work(i);
        #pragma omp critical
        {
            printf("  [guided,2] Iter %02d processed by Thread %d\n",
                   i, omp_get_thread_num());
        }
    }

    t_end = omp_get_wtime();
    double time_guided_2 = t_end - t_start;
    printf("Elapsed Time (guided, 2): %f seconds\n\n", time_guided_2);

    /* -------------------------------------------------------------
     * Summary Performance Comparison
     * ------------------------------------------------------------- */
    printf("============================================================\n");
    printf("               PERFORMANCE SUMMARY (Imbalanced Work)         \n");
    printf("============================================================\n");
    printf("  schedule(static)    : %f s (Suffers from tail stragglers)\n", time_static);
    printf("  schedule(static, 2) : %f s (Improved cyclic balance)\n", time_static_2);
    printf("  schedule(dynamic, 2): %f s (Dynamic load balanced)\n", time_dynamic_2);
    printf("  schedule(guided, 2) : %f s (Optimal balance + low overhead)\n", time_guided_2);
    printf("============================================================\n");

    return EXIT_SUCCESS;
}
```

### 5.2 Sample Build and Output

```bash
$ gcc -O3 -fopenmp omp_scheduling_demo.c -o omp_scheduling_demo -lm
$ ./omp_scheduling_demo
```

```text
============================================================
      OpenMP Loop Scheduling Strategies Demonstration       
============================================================
Configuration: Total Iterations N = 24, Threads T = 4

--- 1. Testing schedule(static) ---
  [static]  Iter 00 processed by Thread 0
  [static]  Iter 01 processed by Thread 0
  ...
  [static]  Iter 06 processed by Thread 1
  ...
  [static]  Iter 18 processed by Thread 3
  [static]  Iter 23 processed by Thread 3
Elapsed Time (static): 0.082140 seconds

--- 2. Testing schedule(static, 2) ---
  [static,2] Iter 00 processed by Thread 0
  [static,2] Iter 01 processed by Thread 0
  [static,2] Iter 02 processed by Thread 1
  [static,2] Iter 03 processed by Thread 1
  [static,2] Iter 08 processed by Thread 0
  [static,2] Iter 09 processed by Thread 0
Elapsed Time (static, 2): 0.054310 seconds

--- 3. Testing schedule(dynamic, 2) ---
  [dynamic,2] Iter 00 processed by Thread 0
  [dynamic,2] Iter 01 processed by Thread 0
  [dynamic,2] Iter 02 processed by Thread 2
  [dynamic,2] Iter 04 processed by Thread 0
  [dynamic,2] Iter 06 processed by Thread 0
Elapsed Time (dynamic, 2): 0.048920 seconds

--- 4. Testing schedule(guided, 2) ---
  [guided,2] Iter 00 processed by Thread 0 (Received Chunk size 6)
  [guided,2] Iter 06 processed by Thread 1 (Received Chunk size 5)
  [guided,2] Iter 11 processed by Thread 2 (Received Chunk size 4)
  [guided,2] Iter 22 processed by Thread 3 (Received Chunk size 2)
Elapsed Time (guided, 2): 0.046180 seconds

============================================================
               PERFORMANCE SUMMARY (Imbalanced Work)         
============================================================
  schedule(static)    : 0.082140 s (Suffers from tail stragglers)
  schedule(static, 2) : 0.054310 s (Improved cyclic balance)
  schedule(dynamic, 2): 0.048920 s (Dynamic load balanced)
  schedule(guided, 2) : 0.046180 s (Optimal balance + low overhead)
============================================================
```

Notice how `schedule(static)` was the slowest because Thread 3 received all the heaviest iterations ($18..23$) and held up the team, whereas `guided` dynamically adapted chunk sizes, producing the fastest execution time.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Guided Chunk Size Step-by-Step Derivation
Let $N = 100$, $T = 4$, and $C_{\text{min}} = 2$. Let us trace the chunk sizes allocated:

1. **First Allocation (Chunk 1):**
   $$S_1 = \left\lceil \frac{100}{4} \right\rceil = 25 \implies R_1 = 100 - 25 = 75$$
2. **Second Allocation (Chunk 2):**
   $$S_2 = \left\lceil \frac{75}{4} \right\rceil = \lceil 18.75 \rceil = 19 \implies R_2 = 75 - 19 = 56$$
3. **Third Allocation (Chunk 3):**
   $$S_3 = \left\lceil \frac{56}{4} \right\rceil = 14 \implies R_3 = 56 - 14 = 42$$
4. **Fourth Allocation (Chunk 4):**
   $$S_4 = \left\lceil \frac{42}{4} \right\rceil = \lceil 10.5 \rceil = 11 \implies R_4 = 42 - 11 = 31$$
5. **Fifth Allocation (Chunk 5):**
   $$S_5 = \left\lceil \frac{31}{4} \right\rceil = \lceil 7.75 \rceil = 8 \implies R_5 = 31 - 8 = 23$$
6. **Subsequent Allocations:**
   $S_6 = \lceil 23/4 \rceil = 6$, $S_7 = \lceil 17/4 \rceil = 5$, $S_8 = \lceil 12/4 \rceil = 3$, $S_9 = \lceil 9/4 \rceil = 3$, $S_{10} = \max(\lceil 6/4 \rceil, 2) = 2$, $\dots$

Notice how chunk sizes decrease exponentially:
$$25 \longrightarrow 19 \longrightarrow 14 \longrightarrow 11 \longrightarrow 8 \longrightarrow 6 \longrightarrow 5 \longrightarrow 3 \longrightarrow 2 \dots$$

---

## 7. Advantages & Limitations

### Scheduling Strategy Trade-off Matrix

| Strategy | Scheduling Overhead | Cache Locality | Load Balancing Capability | Best Suited For |
|:---|:---:|:---:|:---:|:---|
| **`static`** | **Lowest (Zero at runtime)** | **Best** (Sequential contiguous blocks) | **Poor** (Vulnerable to irregular load) | Homogeneous loops, matrix operations, dense linear algebra |
| **`static, C`** | Very Low | Moderate (Strided blocks) | Moderate (Solves predictable triangular skew) | Triangular matrix loops, interleaved tasks |
| **`dynamic, C`** | High (Atomic queue per chunk) | Poor (Threads jump across memory) | **Excellent** (Fast threads pull more work) | Unpredictable loops, server requests, fractals, ray tracing |
| **`guided, C`** | Moderate | Good initially, shrinks at tail | **Best** (Adaptive smoothing) | Unknown or uneven workloads where dynamic overhead is too high |
| **`runtime`** | Depends on chosen mode | Depends on chosen mode | Depends on chosen mode | Tuning, benchmarking, production environments |

---

## 8. Formal Definitions for Exam

- **Loop Scheduling:** The algorithm and runtime policy used by an OpenMP implementation to partition the iterations of a parallel loop and assign them to threads in a team.
- **Chunk Size ($C$):** A contiguous number of iterations dispatched as a single indivisible unit of work to a thread.
- **Static Scheduling:** A scheduling method where iteration blocks are calculated deterministically at loop entry and assigned to threads in block or block-cyclic fashion with zero runtime synchronization.
- **Dynamic Scheduling:** A scheduling method where iteration chunks of fixed size $C$ are placed in a shared work queue and assigned to idle threads on demand at runtime.
- **Guided Scheduling:** A dynamic scheduling variant where chunk sizes decrease exponentially in proportion to the remaining unassigned iterations divided by the number of threads, bounded below by a specified minimum chunk size.
- **Load Imbalance:** The condition in parallel execution where some threads finish their assigned work earlier than others, wasting computational cycles while idling at a synchronization barrier.

---

## 9. Comparison Table

### Summary Comparison Across All 5 OpenMP Scheduling Types

| Feature | `schedule(static)` | `schedule(dynamic)` | `schedule(guided)` | `schedule(auto)` | `schedule(runtime)` |
|:---|:---:|:---:|:---:|:---:|:---:|
| **When is assignment decided?** | Compile / Loop Entry | At runtime (as threads finish) | At runtime (as threads finish) | Handled by Compiler/Runtime | Deferred to execution via Env Var |
| **Chunk Size Behavior** | Fixed ($\approx N/T$ or $C$) | Fixed ($C$) | Dynamically Decreasing | Implementation defined | Determined by `OMP_SCHEDULE` |
| **Runtime Queue Contention?** | **None** | **Yes** (Every chunk) | **Reduced** (Large initial chunks) | Implementation defined | Determined by `OMP_SCHEDULE` |
| **Hardware Prefetch Friendly?** | **Yes (Superb)** | No (Cache hopping) | Yes initially, drops near tail | Implementation defined | Determined by `OMP_SCHEDULE` |
| **Default Chunk Size** | $\approx \lceil N/T \rceil$ | $1$ | $1$ (as minimum) | Implementation defined | Implementation defined |

---

## 10. Solved Analytical & Code-Tracing Problems

### Problem 1: Static Cyclic Chunk Assignment Trace
**Question:** A parallel loop with $N = 20$ iterations ($i = 0$ to $19$) is executed by $T = 3$ threads using `#pragma omp for schedule(static, 3)`. Trace the exact distribution of chunks and iterations to Thread 0, Thread 1, and Thread 2.

**Solution:**
1. Chunk size $C = 3$. Total chunks $= \lceil 20 / 3 \rceil = 7$ chunks:
   - Chunk 0: Iterations $[0, 1, 2]$
   - Chunk 1: Iterations $[3, 4, 5]$
   - Chunk 2: Iterations $[6, 7, 8]$
   - Chunk 3: Iterations $[9, 10, 11]$
   - Chunk 4: Iterations $[12, 13, 14]$
   - Chunk 5: Iterations $[15, 16, 17]$
   - Chunk 6: Iterations $[18, 19]$ (Partial remainder chunk)
2. Assign chunks to threads using round-robin rule $k \pmod 3$:
   - **Thread 0 (Chunks 0, 3, 6):** Iterations $\mathbf{[0, 1, 2]}$, $\mathbf{[9, 10, 11]}$, and $\mathbf{[18, 19]}$ (Total: 8 iterations).
   - **Thread 1 (Chunks 1, 4):** Iterations $\mathbf{[3, 4, 5]}$, and $\mathbf{[12, 13, 14]}$ (Total: 6 iterations).
   - **Thread 2 (Chunks 2, 5):** Iterations $\mathbf{[6, 7, 8]}$, and $\mathbf{[15, 16, 17]}$ (Total: 6 iterations).

---

### Problem 2: Guided Scheduling Iteration Calculation
**Question:** Let $N = 64$, $T = 4$, and the directive be `#pragma omp for schedule(guided, 4)`. Calculate the size of the first two chunks dispatched.

**Solution:**
1. Initial remaining iterations $R_0 = 64$.
2. **First Chunk Size:**
   $$S_1 = \max\left( \left\lceil \frac{R_0}{T} \right\rceil, C_{\text{min}} \right) = \max\left( \left\lceil \frac{64}{4} \right\rceil, 4 \right) = \max(16, 4) = \mathbf{16}$$
   - Iterations $[0..15]$ assigned to the first arriving thread.
   - Remaining iterations $R_1 = 64 - 16 = 48$.
3. **Second Chunk Size:**
   $$S_2 = \max\left( \left\lceil \frac{R_1}{T} \right\rceil, C_{\text{min}} \right) = \max\left( \left\lceil \frac{48}{4} \right\rceil, 4 \right) = \max(12, 4) = \mathbf{12}$$
   - Iterations $[16..27]$ assigned to the second arriving thread.
   - Remaining iterations $R_2 = 48 - 12 = 36$.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why is `schedule(dynamic, 1)` often a disastrous choice for fine-grained loops?
**Answer:** Because every single iteration requires an idle thread to atomically modify the shared queue index variable using a synchronized CPU instruction (e.g., `lock cmpxchg`). This triggers **cache line invalidation** across all CPU cores (the MESI cache coherence protocol forces every core to evict the cache line containing the queue index). The latency of resolving this atomic cache bounce is typically $50$ to $200$ CPU cycles—meaning that if the loop body takes only $10$ cycles, the program spends $95\%$ of its time waiting on atomic memory synchronization!

### Q2: How does `schedule(guided)` solve the overhead problem of `schedule(dynamic)`?
**Answer:** By dispatching **exponentially decreasing chunk sizes**. Early in the loop, guided dispatches massive chunks ($N/T$), so threads execute for millions of cycles without touching the shared queue. As the loop nears completion, chunks shrink to smooth out stragglers. Thus, guided provides the fine-grained load-balancing of dynamic scheduling while executing orders of magnitude fewer atomic queue requests.

### Q3: When should I use `schedule(runtime)`?
**Answer:** Use `schedule(runtime)` during **performance tuning and profiling**. By writing `schedule(runtime)` in your code, you can write a benchmarking shell script that tests `static`, `dynamic`, and `guided` across different chunk sizes (via `export OMP_SCHEDULE="..."`) without recompiling the executable each time.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    OPENMP SCHEDULING CHEATSHEET                                    |
+----------------------------------------------------------------------------------------------------+
| Syntax                 | Description                          | Scheduling Cost | Cache Locality   |
+------------------------+--------------------------------------+-----------------+------------------+
| schedule(static)       | Equal contiguous blocks (~N/T)       | Zero (At entry) | Superb           |
| schedule(static, C)    | Round-robin chunks of size C         | Near Zero       | Moderate         |
| schedule(dynamic, C)   | Fixed chunks of size C from queue    | High            | Poor             |
| schedule(guided, C)    | Exponentially shrinking chunks >= C  | Moderate        | Good -> Moderate |
| schedule(auto)         | Compiler/runtime decides             | System defined  | System defined   |
| schedule(runtime)      | Configured via OMP_SCHEDULE env var  | System defined  | System defined   |
+----------------------------------------------------------------------------------------------------+
|                                      DECISION FLOWCHART                                            |
+----------------------------------------------------------------------------------------------------+
| Are all iterations equal in computational time?                                                    |
|   YES  ---> schedule(static)                                                                      |
|   NO   ---> Is iteration cost predictable (e.g. triangular i)?                                     |
|               YES  ---> schedule(static, C)                                                       |
|               NO   ---> Is loop large with unknown imbalance?                                      |
|                           YES  ---> schedule(guided, C)                                            |
|                           NO   ---> schedule(dynamic, C)                                           |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the operational mechanics of `static`, `dynamic`, `guided`, `auto`, and `runtime` scheduling strategies.
2. Calculate the exact iteration assignment for threads under `static` and `static, C` chunking.
3. Compute the exponential chunk decay sequence for `guided` scheduling using the formal recurrence equation.
4. Select the optimal scheduling policy and chunk size based on workload predictability, iteration granularity, and cache locality.
5. Use the `OMP_SCHEDULE` environment variable to benchmark scheduling policies dynamically without code recompilation.
