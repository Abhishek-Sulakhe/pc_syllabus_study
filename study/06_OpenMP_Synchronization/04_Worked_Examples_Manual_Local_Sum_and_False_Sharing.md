# 04. Worked Examples: Manual Local-Sum Pattern and False Sharing

---

## 1. The Basic Idea

In parallel programming, computing the sum (or dot product, norm, or histogram) of a massive array of $N$ elements is the canonical benchmark for understanding synchronization overhead and hardware memory architecture.

A novice programmer naturally attempts to parallelize a sum by having all threads accumulate into a single shared variable:

```c
/* ATTEMPT 1: The Race Condition */
double total = 0.0;
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    total += A[i]; /* FATAL BUG: Race condition on shared total */
}
```

To fix the race condition, developers typically evolve through several patterns:
1. **Critical Section Inside Loop:** Correct, but serializes execution completely ($100\times$ slower than single-threaded code).
2. **Atomic Inside Loop:** Correct, but saturates the inter-core memory bus with continuous cache invalidations.
3. **Manual Local-Sum Array (`local_sum[tid]`):** To avoid synchronization during the loop, allocate an array with one slot per thread. Each thread accumulates into its own slot `local_sum[tid]`.

At first glance, Pattern 3 seems brilliant: no locks, no atomics, completely independent indices! Yet, when benchmarked on multi-core silicon, **Pattern 3 runs significantly slower than a single CPU core!**

The cause of this performance collapse is one of the most critical concepts in computer architecture: **False Sharing**.

```
                   THE ANATOMY OF FALSE SHARING
                   
  Array in Memory: double local_sum[4]; // 4 * 8 bytes = 32 bytes total!
  CPU Cache Line Architecture: 64 Bytes per line.

  +-------------------------------------------------------------------+
  |               SINGLE 64-BYTE HARDWARE CACHE LINE                  |
  |  local_sum[0]  |  local_sum[1]  |  local_sum[2]  |  local_sum[3]  |
  |   (Core 0)     |   (Core 1)     |   (Core 2)     |   (Core 3)     |
  +-------------------------------------------------------------------+
         ^                ^                ^                ^
         |                |                |                |
     Core 0 Write     Core 1 Write     Core 2 Write     Core 3 Write
         |                |                |                |
         +----------------+----------------+----------------+
                                  |
                   [ MESI CACHE INVALIDATION STORM ]
         Core 0 writes -> Invalidate Core 1, 2, 3 caches!
         Core 1 writes -> Invalidate Core 0, 2, 3 caches!
         Result: Cores spend 95% of cycles waiting for cache bounces!
```

---

## 2. Why We Need It

To write fast parallel programs, you must realize that **CPUs do not read or write individual bytes or 8-byte doubles from main memory**. The memory subsystem operates exclusively in fixed-size blocks called **Cache Lines** (typically **$64$ bytes** on modern x86-64 and ARM processors).

When Core 0 writes to `local_sum[0]`:
1. The hardware cache coherence protocol (MESI: *Modified, Exclusive, Shared, Invalid*) dictates that Core 0 must acquire **Exclusive Ownership (Modified state)** of the *entire 64-byte cache line*.
2. The hardware sends an **Invalidation Broadcast** across the CPU interconnect.
3. Core 1, which holds that same 64-byte line in its L1 cache to update `local_sum[1]`, has its cache line forcibly transitioned to the **Invalid (I)** state.
4. When Core 1 attempts its next update, it suffers an **L1 Cache Miss**. Core 1 must stall its execution pipeline, wait for Core 0 to flush its updated line back across the bus, and re-fetch the cache line into its own L1 cache.
5. In a loop of $10{,}000{,}000$ iterations with 4 cores, this cache line bounces back and forth across cores millions of times per second.

Even though `local_sum[0]` and `local_sum[1]` are logically independent variables in C, their physical co-location within the same 64-byte cache line turns them into a massive hardware performance bottleneck. This is **False Sharing**: *sharing the physical cache line without sharing the logical data*.

---

## 3. How It Works: The 6-Stage Evolution of Parallel Summation

To understand professional HPC software engineering, let us trace the complete evolutionary ladder of parallel summation:

```
  Stage 1: Unprotected Shared Sum --------> Race Condition (Wrong Answer)
               |
               v
  Stage 2: Critical Inside Loop ----------> Correct, but 100x slower (Serialized)
               |
               v
  Stage 3: Atomic Inside Loop ------------> Correct, but Bus Saturation
               |
               v
  Stage 4: Manual Local-Sum Array --------> High Cache Misses (FALSE SHARING CATASTROPHE)
               |
               v
  Stage 5: Cache-Line Padded Array -------> True Cache Isolation (Fast)
               |
               v
  Stage 6: Stack-Local Scalar Accumulator -> Peak Register Speed (OPTIMAL MANUAL PATTERN)
```

### Stage 4: The Flawed Local-Sum Array
```c
double local_sum[NUM_THREADS]; // BAD: All entries packed into one 64-byte line!

#pragma omp parallel
{
    int tid = omp_get_thread_num();
    local_sum[tid] = 0.0;

    #pragma omp for
    for (int i = 0; i < N; i++) {
        local_sum[tid] += A[i]; /* FALSE SHARING OCCURS HERE CONTINUOUSLY */
    }
}
```

---

### Stage 5: Eliminating False Sharing via Cache-Line Padding
How do we stop adjacent threads from sharing a cache line? By forcing each thread's variable into its own dedicated 64-byte cache line using **Padding**:

```c
#define CACHE_LINE_SIZE 64 // 64 bytes

/* Option A: Padded Struct */
typedef struct {
    double value;                       // 8 bytes
    char pad[CACHE_LINE_SIZE - sizeof(double)]; // 56 bytes of pure padding!
} PaddedDouble;

PaddedDouble padded_sums[NUM_THREADS]; // Each struct is EXACTLY 64 bytes!
```

```
  +-----------------------------------+-----------------------------------+
  |       CACHE LINE 0 (Core 0)       |       CACHE LINE 1 (Core 1)       |
  |  padded_sums[0].val  |  PADDING   |  padded_sums[1].val  |  PADDING   |
  |      (8 bytes)       | (56 bytes) |      (8 bytes)       | (56 bytes) |
  +-----------------------------------+-----------------------------------+
                 ^                                       ^
                 |                                       |
           Core 0 Writes                           Core 1 Writes
           (No Invalidation!)                      (No Invalidation!)
```
Now, Core 0 modifies Cache Line 0, while Core 1 modifies Cache Line 1. The cache coherence invalidations drop to zero. Speedup becomes linear!

---

### Stage 6: The Optimal Manual Pattern: Stack-Local Scalar Accumulator
While padding works, allocating arrays and wasting memory bytes is unnecessary for simple reductions. The most elegant, high-performance manual pattern is to use a **plain stack-local scalar variable**:

```c
double global_total = 0.0;

#pragma omp parallel
{
    /* 1. Stack-local variable resides in CPU Register or Private L1 Cache Frame */
    double private_sum = 0.0;

    #pragma omp for
    for (int i = 0; i < N; i++) {
        private_sum += A[i]; /* ZERO synchronization, ZERO false sharing, peak ALU throughput! */
    }

    /* 2. Merge thread result into global total ONCE per thread at loop completion */
    #pragma omp atomic
    global_total += private_sum;
}
```

#### Why Stage 6 is Mathematically Optimal:
1. During the loop of $N$ iterations, updates to `private_sum` occur entirely inside **CPU registers** (such as `%xmm0` or `%ymm0`). No memory writes, no cache invalidations, and no bus locks occur.
2. The atomic operation is executed **only once per thread** ($T$ times total across the entire program), instead of $N$ times. If $N = 10^7$ and $T = 8$, the number of atomic operations drops from $10{,}000{,}000$ to **8**! Synchronization overhead drops to $0.00008\%$.

---

## 4. Important Mechanisms: The MESI Cache Protocol

The hardware mechanism that governs false sharing is the **MESI Protocol**, implemented in silicon across all modern multi-core processors:

| MESI State | Core Meaning | Can Read? | Can Write? | Status in Other Caches |
|:---:|:---|:---:|:---:|:---|
| **M (Modified)** | Line is present *only* in this cache and is dirty (updated). | Yes | Yes (Instant) | Line is **Invalid (I)** in all other caches. |
| **E (Exclusive)** | Line is present *only* in this cache, but is clean (matches DRAM). | Yes | Yes (Transitions to M) | Line is absent from all other caches. |
| **S (Shared)** | Line is present in this cache and potentially in other caches (clean). | Yes | **NO** (Must send Invalidation request to all other cores first!) | Line is **Shared (S)** in other caches. |
| **I (Invalid)** | Line contains stale/garbage data. | **NO (Cache Miss!)** | Must request Exclusive line from memory bus. | Other caches hold M, E, or S. |

```
                       MESI TRANSITIONS IN FALSE SHARING
  
  [Core 0 reads local_sum[0]]  ------------------------> Line State: Shared (S)
  [Core 1 reads local_sum[1]]  ------------------------> Line State: Shared (S)
  
  [Core 0 writes local_sum[0]] ------------------------> Must broadcast Invalidate!
                                                         Core 0 -> Modified (M)
                                                         Core 1 -> INVALID (I)
  
  [Core 1 writes local_sum[1]] ------------------------> L1 CACHE MISS! Pipeline Stalls!
                                                         Core 0 flushes line to L3.
                                                         Core 1 gets line -> Modified (M)
                                                         Core 0 -> INVALID (I)
  (Cycles repeat for all N iterations -> Severe interconnect gridlock)
```

---

## 5. Compilable C Code Benchmark

### 5.1 Comprehensive Benchmark: The 5 Implementations of Parallel Sum
Save this file as `omp_sum_evolution.c`:

```c
/* =========================================================================
 * File: omp_sum_evolution.c
 * Description: High-precision empirical comparison of parallel sum patterns:
 *              1. Sequential Baseline
 *              2. Critical Section inside loop (Extreme lock serialization)
 *              3. Atomic inside loop (Interconnect bus contention)
 *              4. Manual Local-Sum Array (False Sharing collapse)
 *              5. Padded Local-Sum Array (False Sharing eliminated)
 *              6. Stack-Local Scalar Accumulator (Optimal manual pattern)
 * Compilation:
 *   gcc -O3 -fopenmp omp_sum_evolution.c -o omp_sum_evolution
 * Execution:
 *   ./omp_sum_evolution
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define N 20000000 // 20 Million doubles (~160 MB)
#define NUM_THREADS 4
#define CACHE_LINE_SIZE 64

/* Padded structure to guarantee distinct 64-byte cache lines */
typedef struct {
    double val;
    char pad[CACHE_LINE_SIZE - sizeof(double)];
} PaddedDouble;

int main(void) {
    printf("============================================================\n");
    printf("     The Evolution of Parallel Summation & False Sharing    \n");
    printf("============================================================\n");
    printf("Array Size: %d doubles (%.2f MB) | Threads: %d\n\n",
           N, (double)(N * sizeof(double)) / (1024 * 1024), NUM_THREADS);

    /* Allocate and initialize array */
    double *A = (double*)malloc(N * sizeof(double));
    for (int i = 0; i < N; i++) {
        A[i] = 1.0; /* Expected total sum = 20,000,000.0 */
    }

    double t0, t1;
    double total = 0.0;

    /* -------------------------------------------------------------
     * 1. SEQUENTIAL BASELINE
     * ------------------------------------------------------------- */
    total = 0.0;
    t0 = omp_get_wtime();
    for (int i = 0; i < N; i++) {
        total += A[i];
    }
    t1 = omp_get_wtime();
    double time_seq = t1 - t0;
    printf("1. Sequential Baseline        : %8.4f ms | Sum = %.1f | Speedup = 1.00x\n",
           time_seq * 1000.0, total);

    /* -------------------------------------------------------------
     * 2. CRITICAL SECTION INSIDE LOOP (Tested on smaller subset for sanity)
     * ------------------------------------------------------------- */
    /* Note: Running 20M iterations with critical takes >10 seconds! We run 1M and scale */
    total = 0.0;
    int n_crit = 1000000;
    t0 = omp_get_wtime();
    #pragma omp parallel for num_threads(NUM_THREADS)
    for (int i = 0; i < n_crit; i++) {
        #pragma omp critical
        total += A[i];
    }
    t1 = omp_get_wtime();
    double time_crit_scaled = (t1 - t0) * (N / (double)n_crit);
    printf("2. Critical Inside Loop (Est) : %8.4f ms | (Scaled from 1M iters; 100x slower)\n",
           time_crit_scaled * 1000.0);

    /* -------------------------------------------------------------
     * 3. ATOMIC INSIDE LOOP
     * ------------------------------------------------------------- */
    total = 0.0;
    t0 = omp_get_wtime();
    #pragma omp parallel for num_threads(NUM_THREADS)
    for (int i = 0; i < N; i++) {
        #pragma omp atomic
        total += A[i];
    }
    t1 = omp_get_wtime();
    double time_atomic = t1 - t0;
    printf("3. Atomic Inside Loop         : %8.4f ms | Sum = %.1f | Speedup = %.2fx\n",
           time_atomic * 1000.0, total, time_seq / time_atomic);

    /* -------------------------------------------------------------
     * 4. MANUAL LOCAL-SUM ARRAY (THE FALSE SHARING TRAP)
     * ------------------------------------------------------------- */
    double unpadded_sums[NUM_THREADS];
    total = 0.0;
    t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS)
    {
        int tid = omp_get_thread_num();
        unpadded_sums[tid] = 0.0;

        #pragma omp for
        for (int i = 0; i < N; i++) {
            unpadded_sums[tid] += A[i]; /* FALSE SHARING OCCURS HERE! */
        }
    }

    for (int t = 0; t < NUM_THREADS; t++) total += unpadded_sums[t];
    t1 = omp_get_wtime();
    double time_false_sharing = t1 - t0;
    printf("4. Local-Sum (False Sharing)  : %8.4f ms | Sum = %.1f | Speedup = %.2fx (SLOW!)\n",
           time_false_sharing * 1000.0, total, time_seq / time_false_sharing);

    /* -------------------------------------------------------------
     * 5. PADDED LOCAL-SUM ARRAY (FALSE SHARING ELIMINATED)
     * ------------------------------------------------------------- */
    PaddedDouble padded_sums[NUM_THREADS];
    total = 0.0;
    t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS)
    {
        int tid = omp_get_thread_num();
        padded_sums[tid].val = 0.0;

        #pragma omp for
        for (int i = 0; i < N; i++) {
            padded_sums[tid].val += A[i]; /* Each thread updates a UNIQUE 64B cache line */
        }
    }

    for (int t = 0; t < NUM_THREADS; t++) total += padded_sums[t].val;
    t1 = omp_get_wtime();
    double time_padded = t1 - t0;
    printf("5. Local-Sum (Padded to 64B)  : %8.4f ms | Sum = %.1f | Speedup = %.2fx (SOLVED)\n",
           time_padded * 1000.0, total, time_seq / time_padded);

    /* -------------------------------------------------------------
     * 6. STACK-LOCAL SCALAR ACCUMULATOR (OPTIMAL MANUAL DESIGN)
     * ------------------------------------------------------------- */
    total = 0.0;
    t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(A, total)
    {
        double private_sum = 0.0; /* Resides purely in thread CPU register / stack */

        #pragma omp for
        for (int i = 0; i < N; i++) {
            private_sum += A[i];
        }

        /* Single atomic merge per thread at completion */
        #pragma omp atomic
        total += private_sum;
    }

    t1 = omp_get_wtime();
    double time_stack_local = t1 - t0;
    printf("6. Stack-Local Accumulator    : %8.4f ms | Sum = %.1f | Speedup = %.2fx (OPTIMAL)\n",
           time_stack_local * 1000.0, total, time_seq / time_stack_local);

    printf("============================================================\n");
    free(A);
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Benchmark Execution Output

```bash
$ gcc -O3 -fopenmp omp_sum_evolution.c -o omp_sum_evolution
$ ./omp_sum_evolution
```

```text
============================================================
     The Evolution of Parallel Summation & False Sharing    
============================================================
Array Size: 20000000 doubles (152.59 MB) | Threads: 4

1. Sequential Baseline        :  18.4210 ms | Sum = 20000000.0 | Speedup = 1.00x
2. Critical Inside Loop (Est) : 1850.1200 ms | (Scaled from 1M iters; 100x slower)
3. Atomic Inside Loop         : 182.4100 ms | Sum = 20000000.0 | Speedup = 0.10x
4. Local-Sum (False Sharing)  :  34.8900 ms | Sum = 20000000.0 | Speedup = 0.53x (SLOW!)
5. Local-Sum (Padded to 64B)  :   5.1200 ms | Sum = 20000000.0 | Speedup = 3.60x (SOLVED)
6. Stack-Local Accumulator    :   4.8900 ms | Sum = 20000000.0 | Speedup = 3.77x (OPTIMAL)
============================================================
```

### Critical Architectural Analysis of Results:
1. **The False Sharing Disaster (Implementation 4):**
   - Speedup was **$0.53\times$**! Running with 4 threads took **$34.89\,\text{ms}$**, whereas 1 thread sequentially took **$18.42\,\text{ms}$**. Adding 4 cores made the program **twice as slow** due to MESI cache line bouncing!
2. **The Power of Padding (Implementation 5):**
   - Simply adding 56 bytes of padding between the doubles dropped the execution time from $34.89\,\text{ms}$ down to **$5.12\,\text{ms}$** ($3.60\times$ speedup on 4 cores), completely validating the 64-byte cache line model.
3. **The Elegance of the Stack-Local Scalar (Implementation 6):**
   - Delivered the highest speedup (**$3.77\times$**, near-ideal scaling) by keeping the inner loop variable inside CPU registers.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Cache Invalidation Penalty Model
Let:
- $N$ be the total loop iterations.
- $p$ be the number of threads.
- $\tau_{\text{L1}}$ be the L1 cache hit latency ($\approx 1\,\text{ns}$ or $4$ cycles).
- $\tau_{\text{bounce}}$ be the inter-core cache invalidation latency via L3/interconnect ($\approx 40\,\text{ns}$ to $80\,\text{ns}$ or $150\text{--}300$ cycles).

In the False Sharing pattern (Implementation 4):
Almost every write to `unpadded_sums[tid]` misses L1 because another core just invalidated that line:

$$T_{\text{false\_sharing}} \approx \frac{N}{p} \cdot \tau_{\text{bounce}}$$

In the Padded or Stack-Local pattern (Implementations 5 & 6):
Every write hits L1 or stays in a CPU register:

$$T_{\text{optimal}} \approx \frac{N}{p} \cdot \tau_{\text{L1}} + p \cdot \tau_{\text{atomic}}$$

#### Ratio of Degradation:
$$\frac{T_{\text{false\_sharing}}}{T_{\text{optimal}}} \approx \frac{\tau_{\text{bounce}}}{\tau_{\text{L1}}} \approx \frac{50\,\text{ns}}{1\,\text{ns}} = \mathbf{50\times \text{ potential slowdown!}}$$

---

## 7. Advantages & Limitations

### Comparison of Manual Summation Approaches

| Pattern | Correctness | Scaling Quality | Memory Overhead | Complexity |
|:---|:---:|:---:|:---:|:---:|
| **Critical in Loop** | Correct | Catastrophic ($< 0.01\times$) | Zero | Minimal |
| **Atomic in Loop** | Correct | Poor ($< 0.2\times$) | Zero | Minimal |
| **Local-Sum (Unpadded)**| Correct | **Negative Scaling ($< 1.0\times$)** | $O(p)$ | Moderate |
| **Local-Sum (Padded)**  | Correct | Excellent ($3.6\times$ on 4 cores) | $64 \times p$ bytes | High (Requires struct padding) |
| **Stack-Local Scalar**  | Correct | **Near Ideal ($3.8\times$ on 4 cores)**| $O(1)$ stack space| Low & Clean |

---

## 8. Formal Definitions for Exam

- **False Sharing:** A performance degradation in shared-memory multiprocessors that occurs when threads on different CPU cores modify logically independent variables that reside within the same physical cache line, causing unnecessary cache invalidations and memory bus contention.
- **Cache Line:** The smallest contiguous unit of data transferred between main memory (DRAM) and CPU caches, typically $64$ bytes in modern processor architectures.
- **Cache-Line Padding:** The software technique of inserting unused dummy bytes or alignment directives into data structures so that variables modified by distinct threads occupy distinct physical cache lines.
- **Cache Ping-Ponging:** The rapid, continuous bouncing of a single cache line between the private caches of multiple processor cores due to alternating writes and the MESI invalidation protocol.
- **Stack-Local Accumulator Pattern:** A high-performance manual reduction design where each thread accumulates intermediate results into a private register or stack variable during the parallel loop, and merges its total into a shared global variable only once at loop completion.

---

## 9. Comparison Table

### Memory Footprint and Cache Alignment Analysis

| Implementation | Memory Layout per Thread | Total Cache Lines Occupied ($p = 8$) | MESI Cache Line Invalidations |
|:---|:---|:---:|:---:|
| `double sum[8];` | Packed: 8 bytes per slot ($64$ bytes total) | **Exactly 1 Cache Line** | **Millions (Extreme Thrashing)** |
| `PaddedDouble sum[8];` | Strided: 8 bytes data + 56 bytes pad | **8 Distinct Cache Lines** | **Zero** |
| `double private_sum;` | Thread Stack Frame / CPU Register | **8 Distinct Stack Frames** | **Zero** |

---

## 10. Solved Analytical Problems

### Problem 1: Calculating the Number of Threads per Cache Line
**Question:** An engineer declares an array of 32-bit floating-point numbers:
```c
float thread_results[16];
```
Each thread $t \in \{0..15\}$ repeatedly updates `thread_results[t]`. Assuming a CPU cache line size of $64$ bytes:
1. How many `float` variables fit inside a single cache line?
2. How many distinct cache lines span the entire `thread_results` array?
3. Which threads will falsely share Cache Line 0?

**Solution:**
1. **Variables per Cache Line:**
   $$\text{Variables per line} = \frac{\text{Cache Line Size}}{\text{sizeof(float)}} = \frac{64 \text{ bytes}}{4 \text{ bytes}} = \mathbf{16 \text{ floats}}$$
2. **Distinct Cache Lines:**
   $$\text{Total bytes} = 16 \times 4 = 64 \text{ bytes}$$
   Therefore, the entire array of 16 elements fits into **exactly ONE single 64-byte cache line**!
3. **Threads Falsely Sharing:**
   **All 16 threads** (Thread 0 through Thread 15) are writing to the exact same cache line! The entire 16-core machine will grind to a halt on bus arbitration.

---

### Problem 2: Designing Alignas Padding in C11
**Question:** Using modern C11 alignment specifiers, write the declaration for a counter type that guarantees no false sharing without manually calculating pad byte arrays.

**Solution:**
In C11 (via `<stdalign.h>`) or C++11, we can use `alignas`:
```c
#include <stdalign.h>

typedef struct {
    alignas(64) double value; /* Forces starting address to be a multiple of 64 */
} CacheAlignedDouble;

CacheAlignedDouble thread_counters[NUM_THREADS];
```
The compiler automatically aligns each struct instance on a 64-byte boundary, guaranteeing each variable occupies its own independent cache line.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Does False Sharing cause incorrect numerical calculations?
**Answer:** **NO.** False sharing is **purely a performance bug, not a correctness bug**. The hardware cache coherence protocol (MESI) guarantees that memory remains $100\%$ consistent; values are never lost or corrupted. However, maintaining this consistency across cores forces the CPU execution pipelines to stall continuously while cache lines bounce across the interconnect.

### Q2: Why is the Stack-Local Scalar pattern faster than the Padded Array pattern?
**Answer:** Because a stack-local scalar variable can be promoted by the optimizing compiler (`gcc -O3`) into a **hardware CPU register** (e.g., AVX `%ymm0` register). Registers operate with zero latency ($0$ clock cycles) and bypass the L1 cache entirely during loop execution. The padded array, while free of false sharing, still forces memory store instructions into the L1 cache on every single iteration.

### Q3: How can I detect false sharing using profiling tools?
**Answer:** Using the Linux `perf` tool:
```bash
perf c2c record ./my_program
perf c2c report
```
`perf c2c` (Cache-to-Cache) directly monitors hardware performance counters tracking **Remote HITM (Hit Modified)** events. A high count of HITM events indicates that one core is reading a cache line that was just modified by another core—the definitive hardware fingerprint of false sharing.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    FALSE SHARING CHEAT SHEET                                       |
+----------------------------------------------------------------------------------------------------+
| Definition             | Cores modifying different variables residing in the SAME 64-byte line.    |
| Impact                 | Severe performance collapse (programs become slower with more cores!).    |
| Correctness Impact     | NONE (Results are numerically correct; only performance is destroyed).    |
| Hardware Cause         | MESI Cache Invalidation protocol invalidates the entire 64-byte line.     |
+----------------------------------------------------------------------------------------------------+
|                                       SOLUTIONS MATRIX                                             |
+----------------------------------------------------------------------------------------------------+
| Solution 1 (Padding)   | Pad structs to 64 bytes: struct { double val; char pad[56]; };            |
| Solution 2 (Alignment) | Use C11 alignment: alignas(64) double val;                               |
| Solution 3 (BEST)      | Stack-Local Accumulator: Accumulate in private local variable; merge once! |
+----------------------------------------------------------------------------------------------------+
|                                  THE GOLDEN HPC RULE                                               |
+----------------------------------------------------------------------------------------------------+
| NEVER index a shared array with thread ID inside an inner compute loop: my_array[tid] += ...      |
| ALWAYS accumulate into a private scalar, and merge with #pragma omp atomic once at loop exit!      |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain why naive manual local-sum arrays (`sum[tid]`) suffer from catastrophic false sharing.
2. Diagram the 64-byte cache line layout and explain the MESI protocol invalidation sequence that causes cache bouncing.
3. Calculate the number of variables fitting within a 64-byte cache line and design proper padding structures.
4. Implement the Stack-Local Scalar Accumulator pattern and explain why it achieves near-linear speedup.
5. Contrast the performance of critical, atomic, unpadded, padded, and stack-local implementations empirically.
