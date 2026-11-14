# Module 13: Hybrid MPI + OpenMP Programming (Lecture 32)
# Subtopic 13.04: Worked Example: Complete Hybrid MPI + OpenMP Parallel Sum (Two-Level Hierarchical Domain Decomposition and Reduction)

---

## 1. Basic Idea

The **Two-Level Parallel Sum** is the canonical, standard examination model representing the entire hybrid programming philosophy. 

In this application:
- A massive global problem of size $N$ (e.g., summing $100{,}000{,}000$ integers) is computed across a distributed cluster of multi-core servers.
- The work distribution and mathematical reduction proceed through **two distinct hierarchical tiers**:

```
+-----------------------------------------------------------------------------------------+
|                  Two-Level Hierarchical Work Distribution & Reduction                   |
+-----------------------------------------------------------------------------------------+
| STEP 1: INTER-NODE DOMAIN DECOMPOSITION (MPI Tier)                                      |
|   Global Array (N Elements) is partitioned into P large slices:                         |
|     Slice size = N / P elements per MPI process                                         |
|                                                                                         |
| STEP 2: INTRA-NODE WORK-SHARING (OpenMP Tier)                                           |
|   Each MPI process partitions its local slice across T OpenMP threads:                  |
|     Sub-chunk = (N / P) / T elements per thread                                         |
|                                                                                         |
| STEP 3: TIER 1 REDUCTION (Intra-Node OpenMP Reduction)                                  |
|   Threads in each process sum their local chunks into a single process-local sum:       |
|     #pragma omp parallel for reduction(+:process_sum)                                   |
|                                                                                         |
| STEP 4: TIER 2 REDUCTION (Inter-Node MPI Reduction)                                     |
|   Each process sends its process_sum to Root (Rank 0) across the network:               |
|     MPI_Reduce(&process_sum, &grand_total, 1, MPI_LONG_LONG, MPI_SUM, 0, comm)          |
+-----------------------------------------------------------------------------------------+
```

This two-stage hierarchy completely eliminates network contention: instead of thousands of individual threads flooding the network with point-to-point packets, only $P$ processes communicate over the network fabric.

---

## 2. Why We Need It

### 1. Eliminating Network Flooding
Suppose a supercomputer job allocates 16 nodes with 64 cores each (total $1{,}024$ cores).
- If implemented with pure MPI, $1{,}024$ independent processes must participate in a global reduction across the network switches, traversing physical cables and generating $1{,}024$ network message envelopes.
- In the Hybrid MPI + OpenMP model:
  - 16 MPI processes are launched (1 per node).
  - Each process runs 64 OpenMP threads.
  - The first reduction stage occurs **entirely inside physical RAM** across on-chip CPU registers and L3 caches (nanosecond speed).
  - Only **16 aggregated values** are transmitted over the physical network fabric via `MPI_Reduce`.
  - Network traffic is reduced by **$98.4\%$**!

### 2. Perfect Mathematical Verification
Summing consecutive integers from $1$ to $N$ has a closed-form analytical solution established by Carl Friedrich Gauss:

$$\text{Sum}(N) = \sum_{i=1}^N i = \frac{N(N + 1)}{2}$$

For $N = 100{,}000{,}000$:
$$\text{Sum}(10^8) = \frac{10^8 \times (10^8 + 1)}{2} = \frac{10^{16} + 10^8}{2} = 5{,}000{,}000{,}050{,}000{,}000$$

This allows the program to verify its multi-tiered parallel computation against absolute ground-truth mathematics down to the single integer unit.

---

## 3. How It Works

### The Complete Two-Tier Execution Flow

```
                      GLOBAL DOMAIN: N = 100,000,000 elements
                                         |
                  +----------------------+----------------------+
                  | (MPI Rank 0)                                | (MPI Rank 1)
                  v                                             v
        Slice 0: [ 1 .. 50,000,000 ]                  Slice 1: [ 50,000,001 .. 100,000,000 ]
                  |                                             |
        +---------+---------+                         +---------+---------+
        | OpenMP Threads    |                         | OpenMP Threads    |
        v                   v                         v                   v
   [Thread 0]          [Thread 1]                [Thread 0]          [Thread 1]
   [1 .. 25M]          [25M .. 50M]              [50M .. 75M]        [75M .. 100M]
        \                   /                         \                   /
         +--------+--------+                           +--------+--------+
                  |                                             |
                  v                                             v
       TIER 1 REDUCTION (OpenMP):                    TIER 1 REDUCTION (OpenMP):
       process_sum_0 = 1,250,000,025,000,000         process_sum_1 = 3,750,000,025,000,000
                  \                                             /
                   +---------------------+---------------------+
                                         |
                                         v  MPI_Reduce(MPI_SUM, root=0)
                            TIER 2 REDUCTION (MPI Network):
                            GRAND TOTAL = 5,000,000,050,000,000
                                (Verified by Rank 0!)
```

---

## 4. Important Mechanisms and Functions

### 1. Hierarchical Range Partitioning Formula

For a global index range $[1 \dots N]$ partitioned across $P$ ranks (Rank $r \in [0, P-1]$):

$$\text{Chunk Size } S = \frac{N}{P}$$
$$\text{Rank Start Index } I_{\text{start}}(r) = r \cdot S + 1$$
$$\text{Rank End Index } I_{\text{end}}(r) = (r + 1) \cdot S$$

Within Process $r$, OpenMP automatically divides the local range $[I_{\text{start}}(r) \dots I_{\text{end}}(r)]$ across its $T$ threads using `#pragma omp parallel for`.

---

### 2. The Hybrid OpenMP Reduction Clause

```c
long long process_sum = 0;

#pragma omp parallel for default(none) \
                         shared(start_idx, end_idx) \
                         reduction(+:process_sum)
for (long long i = start_idx; i <= end_idx; ++i) {
    process_sum += i;
}
```

- Each OpenMP thread allocates a private accumulator initialized to `0`.
- Threads sum their assigned loop iterations concurrently without lock contention.
- Upon loop termination, OpenMP combines the private thread accumulators into the single shared variable `process_sum`.

---

### 3. The Global MPI Reduction Call

```c
long long grand_total = 0;

MPI_Reduce(&process_sum, &grand_total, 1, MPI_LONG_LONG,
           MPI_SUM, 0 /* root */, MPI_COMM_WORLD);
```
- Each process contributes its local `process_sum`.
- The network subsystem executes a binomial tree reduction.
- Rank 0 receives the final `grand_total`.

---

## 5. Compilable C Code Examples

### Example 1: The Canonical Complete Hybrid MPI + OpenMP Parallel Sum
This complete, production-grade program computes the sum of the first $100{,}000{,}000$ integers across a cluster, implements the full two-level hierarchy, measures execution timing, and validates correctness against Gauss's formula.

```c
/*
 * File: hybrid_parallel_sum.c
 * Compilation:
 *   mpicc -O2 -fopenmp hybrid_parallel_sum.c -o hybrid_parallel_sum
 *
 * Execution (Syllabus setup: 2 MPI processes x 4 OpenMP threads = 8 total threads):
 *   export OMP_NUM_THREADS=4
 *   mpirun -np 2 ./hybrid_parallel_sum
 */

#include <mpi.h>
#include <omp.h>
#include <stdio.h>
#include <stdlib.h>

#define N 100000000LL /* 100 Million Elements (Requires 64-bit long long) */

int main(int argc, char** argv) {
    int rank, size;
    int provided;

    /* Step 1: Initialize MPI with Funneled thread support */
    MPI_Init_thread(&argc, &argv, MPI_THREAD_FUNNELED, &provided);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int num_threads = omp_get_max_threads();

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  CANONICAL HYBRID MPI + OpenMP TWO-LEVEL PARALLEL SUM           \n");
        printf("  Global Problem Size (N)   : %lld elements                      \n", N);
        printf("  MPI Processes (P)         : %d                                 \n", size);
        printf("  OpenMP Threads/Process (T): %d                                 \n", num_threads);
        printf("  Total Concurrent Workers  : %d threads across cluster          \n", size * num_threads);
        printf("=================================================================\n\n");
    }

    /* Synchronize start timer across all nodes */
    MPI_Barrier(MPI_COMM_WORLD);
    double t_start = MPI_Wtime();

    /* =========================================================================
     * STAGE 1: TIER 1 DOMAIN DECOMPOSITION (Inter-Node Partitioning)
     * ========================================================================= */
    long long chunk_per_rank = N / size;
    long long rank_start = (long long)rank * chunk_per_rank + 1LL;
    long long rank_end   = (rank == size - 1) ? N : (rank_start + chunk_per_rank - 1LL);

    printf("[MPI Rank %d] Assigned Slice: [%lld to %lld] (%lld items)\n",
           rank, rank_start, rank_end, rank_end - rank_start + 1LL);

    /* =========================================================================
     * STAGE 2: TIER 2 INTRA-NODE COMPUTATION & OpenMP REDUCTION
     * ========================================================================= */
    long long process_local_sum = 0;

    #pragma omp parallel for default(none) \
                             shared(rank_start, rank_end) \
                             reduction(+:process_local_sum) \
                             schedule(static)
    for (long long i = rank_start; i <= rank_end; ++i) {
        process_local_sum += i;
    }

    printf("  [MPI Rank %d] Tier 1 OpenMP Reduction Complete. Local Sum = %lld\n",
           rank, process_local_sum);

    /* =========================================================================
     * STAGE 3: TIER 2 INTER-NODE MPI REDUCTION
     * ========================================================================= */
    long long global_grand_total = 0;

    MPI_Reduce(
        &process_local_sum,    /* Input buffer from this process */
        &global_grand_total,   /* Output buffer on Root */
        1,                     /* Element count */
        MPI_LONG_LONG,         /* Datatype */
        MPI_SUM,               /* Operator */
        0,                     /* Root rank */
        MPI_COMM_WORLD         /* Communicator */
    );

    double t_end = MPI_Wtime();

    /* =========================================================================
     * STAGE 4: MATHEMATICAL VERIFICATION ON ROOT (Rank 0)
     * ========================================================================= */
    if (rank == 0) {
        /* Gauss's Formula: N * (N + 1) / 2 */
        long long exact_analytical_sum = (N * (N + 1LL)) / 2LL;
        long long discrepancy = global_grand_total - exact_analytical_sum;

        printf("\n=================================================================\n");
        printf("  FINAL HYBRID EXECUTION RESULTS & VERIFICATION                  \n");
        printf("=================================================================\n");
        printf("  Computed Grand Total   : %lld\n", global_grand_total);
        printf("  Exact Analytical Sum   : %lld\n", exact_analytical_sum);
        printf("  Absolute Discrepancy   : %lld\n", discrepancy);
        printf("  Total Execution Time   : %.4f seconds\n", t_end - t_start);
        printf("-----------------------------------------------------------------\n");

        if (discrepancy == 0) {
            printf(">>> VERIFICATION SUCCESS: 100%% Bit-Perfect Mathematical Match! <<<\n");
        } else {
            printf(">>> VERIFICATION FAILED: Calculation mismatch! <<<\n");
        }
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 2 MPI processes and 4 OpenMP threads each) ---
=================================================================
  CANONICAL HYBRID MPI + OpenMP TWO-LEVEL PARALLEL SUM           
  Global Problem Size (N)   : 100000000 elements                      
  MPI Processes (P)         : 2                                 
  OpenMP Threads/Process (T): 4                                 
  Total Concurrent Workers  : 8 threads across cluster          
=================================================================

[MPI Rank 0] Assigned Slice: [1 to 50000000] (50000000 items)
[MPI Rank 1] Assigned Slice: [50000001 to 100000000] (50000000 items)
  [MPI Rank 0] Tier 1 OpenMP Reduction Complete. Local Sum = 1250000025000000
  [MPI Rank 1] Tier 1 OpenMP Reduction Complete. Local Sum = 3750000025000000

=================================================================
  FINAL HYBRID EXECUTION RESULTS & VERIFICATION                  
=================================================================
  Computed Grand Total   : 5000000050000000
  Exact Analytical Sum   : 5000000050000000
  Absolute Discrepancy   : 0
  Total Execution Time   : 0.0482 seconds
-----------------------------------------------------------------
>>> VERIFICATION SUCCESS: 100% Bit-Perfect Mathematical Match! <<<
=================================================================
```

---

## 6. Mathematical Formulations & Scaling Proof

### Arithmetic Overflow Analysis

In parallel summation benchmarks, selecting proper integer bit-widths is vital.

Let $N = 100{,}000{,}000 = 10^8$.
The maximum sum is:
$$\text{Sum}(10^8) = \frac{10^8 \times (10^8 + 1)}{2} \approx 5 \times 10^{15}$$

#### Bit-Width Capacities:
1. **Standard 32-bit Signed Integer (`int`):**
   $$\text{Max Value} = 2^{31} - 1 = 2{,}147{,}483{,}647 \approx 2.14 \times 10^9$$
   Since $5 \times 10^{15} \gg 2.14 \times 10^9$, using `int` triggers catastrophic **integer overflow**, producing garbage negative numbers!

2. **Standard 64-bit Signed Integer (`long long` / `int64_t`):**
   $$\text{Max Value} = 2^{63} - 1 = 9{,}223{,}372{,}036{,}854{,}775{,}807 \approx 9.22 \times 10^{18}$$
   Since $5 \times 10^{15} \ll 9.22 \times 10^{18}$, `long long` holds the result with over **$1{,}800\times$ headroom**.

*Rule:* Always use `long long` and `MPI_LONG_LONG` for summation benchmarks where $N \ge 65{,}536$.

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Architecture Review

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Two-Level Reduction       | - Reduces network messages by P_node    | - Requires coordination of two separate |
|                           | - Leverages ultra-fast CPU register sum |   reduction syntaxes                    |
|                           | - Minimizes cluster interconnect latency|                                         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Funneled Sizing Pattern   | - 100% safe: master thread communicates | - Worker threads sit idle during the    |
|                           | - Maximizes OpenMP core efficiency      |   final MPI_Reduce call                 |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Two-Level Reduction:** A hierarchical reduction technique in hybrid parallel computing where OpenMP threads reduce data locally within each multi-core process using shared memory, and MPI subsequently reduces the per-process totals across cluster nodes using network communication.
>
> **Gauss's Summation Formula:** The mathematical identity $\sum_{i=1}^N i = \frac{N(N+1)}{2}$, used in high-performance computing as an analytical ground truth to verify the numerical correctness of parallel summation algorithms.
>
> **Hierarchical Domain Decomposition:** The algorithmic division of a problem domain across two distinct hardware tiers: coarse-grained slices assigned to distributed MPI ranks, and fine-grained sub-slices assigned to local OpenMP threads.
>
> **Arithmetic Overflow in Reductions:** A failure mode occurring when accumulator variables lack sufficient bit-width (e.g., using 32-bit `int` instead of 64-bit `long long`), causing the aggregated parallel sum to exceed register capacity and produce corrupted negative values.

---

## 9. Comparison Tables

### Pure MPI Sum vs Pure OpenMP Sum vs Hybrid Sum

| Dimension | Pure MPI Sum | Pure OpenMP Sum | Hybrid MPI + OpenMP Sum |
| :--- | :--- | :--- | :--- |
| **Max Process/Thread Scope**| $1{,}024$ MPI ranks | $64$ OpenMP threads (1 Node) | $16$ MPI ranks $\times$ $64$ OpenMP threads |
| **Network Reduction Size** | $1{,}024$ processes | Zero (Shared memory only) | **16 processes** |
| **Memory Buffer Footprint** | $1{,}024$ buffers | 1 shared buffer | 16 buffers |
| **Reduction Mechanisms** | `MPI_Reduce` only | `#pragma omp reduction` only | **Both** (OpenMP Tier 1 $\to$ MPI Tier 2) |
| **Scalability Limit** | Limited by network latency | Limited by single node RAM | **Exascale Supercomputers** |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Calculating Intermediate Slices for a Given Rank and Thread
**Question:** In the hybrid sum example with $N = 100{,}000{,}000$, $P = 2$ MPI processes, and $T = 4$ OpenMP threads per process:
1. What index slice is assigned to MPI Rank 1?
2. If OpenMP uses a static schedule with equal chunks, what exact index range is computed by **Thread 2 on MPI Rank 1**?

**Solution:**
1. **MPI Rank 1 Slice:**
   - Elements per rank = $100{,}000{,}000 / 2 = 50{,}000{,}000$.
   - Rank 1 Range = $[50{,}000{,}001 \dots 100{,}000{,}000]$.
2. **Thread 2 on Rank 1 Range:**
   - Local elements = $50{,}000{,}000$.
   - Elements per thread = $50{,}000{,}000 / 4 = 12{,}500{,}000$.
   - Thread 0 on Rank 1: $[50{,}000{,}001 \dots 62{,}500{,}000]$
   - Thread 1 on Rank 1: $[62{,}500{,}001 \dots 75{,}000{,}000]$
   - **Thread 2 on Rank 1: $[75{,}000{,}001 \dots 87{,}500{,}000]$**
   - Thread 3 on Rank 1: $[87{,}500{,}001 \dots 100{,}000{,}000]$

---

### Problem 2: Detecting the Missing OpenMP Reduction Clause
**Question:** A student writes the following hybrid loop:
```c
long long process_sum = 0;
#pragma omp parallel for
for (long long i = start; i <= end; ++i) {
    process_sum += i; /* Missing reduction clause */
}
MPI_Reduce(&process_sum, &grand_total, 1, MPI_LONG_LONG, MPI_SUM, 0, comm);
```
What error occurs and what will be the computed result?

**Solution:**
- **The Defect:** The variable `process_sum` is shared across all OpenMP threads, but no `reduction(+:process_sum)` or atomic synchronization was specified.
- Multiple threads concurrently execute `process_sum += i` (Read-Modify-Write data race).
- Updates are silently lost due to multi-core cache line race conditions.
- `process_sum` will be substantially smaller than the true mathematical sum, and will produce non-deterministic, random numbers on every run.
- **The Fix:** Add `reduction(+:process_sum)` to the OpenMP pragma.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why is `MPI_Reduce` called outside the OpenMP parallel region in this worked example?
**Answer:** Because the application operates under the **`MPI_THREAD_FUNNELED`** model. In this model, only the master thread is permitted to invoke MPI communication calls. Placing `MPI_Reduce` outside the OpenMP parallel region guarantees that only the master thread executes the reduction, completely preventing thread race conditions in the MPI transport layer and avoiding the high locking overhead of `MPI_THREAD_MULTIPLE`.

### Q2: What would happen if `MPI_Reduce` were placed INSIDE the OpenMP parallel region without `#pragma omp master`?
**Answer:** Every OpenMP thread in the process would invoke `MPI_Reduce` simultaneously! If there are 4 threads per rank and 2 ranks, the MPI runtime would receive 8 separate reduction calls instead of 2. Under `MPI_THREAD_FUNNELED`, this produces an immediate crash or race condition. Under `MPI_THREAD_MULTIPLE`, it produces incorrect mathematical results because the communicator expects only 2 participating ranks.

### Q3: Why does this program use `schedule(static)` for the OpenMP loop?
**Answer:** Because the computational workload of summing integers (`process_sum += i`) is **perfectly uniform** across all iterations (each iteration takes exactly one addition cycle). Static scheduling divides the loop iterations into equal contiguous chunks at compile time with zero runtime scheduling overhead, maximizing CPU L1 data cache prefetching and instruction pipelining.

---

## 12. Quick Revision Sheet

- **Canonical Architecture:**
  1. Inter-Node: Domain decomposition ($N/P$) across MPI ranks.
  2. Intra-Node: Work-sharing loop across OpenMP threads.
  3. Tier 1 Reduction: OpenMP `#pragma omp parallel for reduction(+:sum)`.
  4. Tier 2 Reduction: `MPI_Reduce(&sum, &total, 1, MPI_LONG_LONG, MPI_SUM, 0, comm)`.
- **Gauss Formula Verification:** $\sum_{i=1}^N i = \frac{N(N+1)}{2}$.
- **Data Type Rule:** Always use 64-bit `long long` and `MPI_LONG_LONG` to prevent integer overflow.
- **Thread Safety Contract:** `MPI_THREAD_FUNNELED` allows master thread to execute `MPI_Reduce` safely after OpenMP threads join.

---

## 13. What You Should Be Able to Do

1. **Write, compile, and execute the complete canonical hybrid parallel sum** program from memory.
2. **Calculate exact mathematical bounds and slice ranges** for arbitrary process and thread counts.
3. **Verify parallel numerical output** using Gauss's closed-form summation formula.
4. **Diagnose and eliminate OpenMP data races** in multi-threaded accumulator loops.
5. **Defend the network scaling advantages** of two-level hierarchical reductions in oral examinations.
