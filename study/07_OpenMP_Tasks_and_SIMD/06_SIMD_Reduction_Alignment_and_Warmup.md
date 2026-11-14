# 06. SIMD Reduction, Alignment, Cache-Line Effects, and Benchmark Warmup

---

## 1. The Basic Idea

In high-performance computing, compiling a loop with `#pragma omp simd` is only the first step toward peak hardware performance. To extract the maximum theoretical FLOP/s from a modern processor, three final microarchitectural bottlenecks must be addressed:

1. **SIMD Vector Reduction:** Combining data elements within a single vector register using horizontal vector instructions without serializing the loop.
2. **Data Alignment & The `aligned` Clause:** Guaranteeing that memory buffers begin on physical **64-byte cache line boundaries**, allowing the CPU to execute high-speed aligned memory loads (`VMOVAPS`) rather than costly unaligned loads (`VMOVUPS`) that trigger split-cache-line stalls.
3. **Benchmark Warmup & Frequency Scaling:** Eliminating measurement noise caused by operating system demand paging (cold page faults), cold CPU caches, and Intel/AMD AVX frequency transition states.

```
                    THE ALIGNED vs. UNALIGNED MEMORY ACCESS
                    
  64-Byte Cache Line 0                64-Byte Cache Line 1
  [ 0 ............................ 63 ] [ 64 ........................... 127 ]
  
  CASE A: 64-BYTE ALIGNED LOAD (e.g., Address 0x00)
  [==== 64-Byte AVX-512 Load ====]
  -> Entire vector loaded from ONE cache line in ONE clock cycle! (VMOVAPS)

  CASE B: UNALIGNED SPLIT-LOAD (e.g., Address 0x30 = Byte 48)
              [==== 64-Byte AVX-512 Load ====]
              (16 bytes from Line 0) + (48 bytes from Line 1)
  -> Hardware must fetch TWO cache lines, split the data, and assemble it!
  -> Doubles L1 cache access latency and stalls vector pipeline! (VMOVUPS)
```

Mastering these concepts concludes our foundational journey through Part A of the curriculum, completing our mastery of shared-memory multi-core and vector architecture.

---

## 2. Why We Need It

### The Cost of Unaligned Memory Access
When an AVX-512 instruction loads 64 bytes of data starting at an unaligned address (such as `0x1004`):
1. **Split-Cache-Line Penalty:** The 64-byte payload straddles two distinct 64-byte cache lines. The CPU's Load Buffer must issue two separate cache queries, wait for both lines to arrive in L1, extract the matching bytes, and splice them together.
2. **Compiler Conservatism:** If the compiler does not know that pointers are aligned, it must generate conservative code with **Peel Loops** (running scalar iterations until an aligned address is reached). This adds binary bloat and branch instruction overhead.

### The `aligned` Clause
The `aligned(pointer : alignment)` clause informs the compiler:
```c
#pragma omp simd aligned(A, B, C : 64)
```
The developer guarantees that pointers `A`, `B`, and `C` are integer multiples of 64 bytes (`ptr % 64 == 0`). 
The compiler immediately strips away the peel loop and emits direct aligned vector instructions (`vmovaps / vmovapd`), achieving peak memory load throughput.

---

## 3. How It Works: SIMD Reductions & Horizontal Additions

### 3.1 SIMD Vector Reduction Mechanics
When computing a dot product or sum reduction using SIMD:
```c
double sum = 0.0;
#pragma omp simd reduction(+ : sum)
for (int i = 0; i < N; i++) {
    sum += A[i];
}
```

How does the single CPU core accumulate numbers without serializing execution?

1. **Vector Accumulator Allocation:** The compiler allocates a 256-bit or 512-bit vector register (e.g., `%ymm0` for AVX2 `double`) initialized to the identity element $\begin{bmatrix} 0 & 0 & 0 & 0 \end{bmatrix}$.
2. **Vertical Vector Accumulation:** On each clock cycle, the core loads 4 doubles and vertically adds them into the 4 lanes of `%ymm0`:

$$\begin{bmatrix} \text{Lane 0} \\ \text{Lane 1} \\ \text{Lane 2} \\ \text{Lane 3} \end{bmatrix}_{\text{new}} = \begin{bmatrix} \text{Lane 0} \\ \text{Lane 1} \\ \text{Lane 2} \\ \text{Lane 3} \end{bmatrix}_{\text{prev}} + \begin{bmatrix} A[i+0] \\ A[i+1] \\ A[i+2] \\ A[i+3] \end{bmatrix}$$

This vertical addition proceeds at **peak register pipeline throughput** ($1$ vector per clock cycle).

3. **Horizontal Reduction Tree (Loop Exit):** At the end of the loop, the 4 partial sums inside `%ymm0` must be folded together into a single scalar double. The core executes a high-speed **Horizontal Vector Reduction Tree** using intra-register shuffle and add instructions:

```
                  HORIZONTAL VECTOR REDUCTION (AVX 4-Lane Double)
                  
  Register %ymm0: [ Lane 0: 25 ]  [ Lane 1: 30 ]  [ Lane 2: 15 ]  [ Lane 3: 10 ]
                         \              /                \              /
                          v            v                  v            v
  Shuffle & Add:       [ 25 + 30 = 55 ]                [ 15 + 10 = 25 ]
                                \                              /
                                 v                            v
  Final Horizontal Add:                  [ 55 + 25 = 80 ]
                                                |
                                                v
                                      Scalar 'sum' = 80.0
```

Notice that during $N$ iterations, zero scalar additions or cross-lane shuffles occurred! The logarithmic horizontal fold happened **only once** at loop exit.

---

### 3.2 Dynamic Memory Allocation with Alignment
In standard C, `malloc()` guarantees alignment only to `2 * sizeof(void*)` ($16$ bytes on 64-bit systems). 
Sixteen-byte alignment is sufficient for legacy 128-bit SSE, but **violates the 32-byte alignment required for AVX2 and the 64-byte alignment required for AVX-512 and CPU cache lines**!

To allocate properly aligned heap memory in modern C/C++:

```c
/* Standard C11 / C++17 Aligned Allocation */
#include <stdlib.h>

size_t alignment = 64; // 64 bytes (AVX-512 and Cache Line boundary)
size_t bytes = N * sizeof(double);

/* NOTE: In C11 aligned_alloc, 'bytes' MUST be an integral multiple of 'alignment'! */
bytes = ((bytes + alignment - 1) / alignment) * alignment;

double *A = (double*)aligned_alloc(alignment, bytes);
```

Other platform-specific allocation primitives:
- **POSIX:** `posix_memalign((void**)&ptr, 64, bytes);`
- **Intel / MSVC:** `_mm_malloc(bytes, 64);` (Freed using `_mm_free(ptr);`)

---

## 4. Benchmark Warmup and Transient Frequency Scaling

When measuring the performance of an OpenMP or SIMD kernel, the **first execution run is almost always drastically slower than subsequent runs**. If you do not perform **Benchmark Warmup**, your timing measurements will be completely invalid.

### The Four Root Causes of Cold Measurement Noise:
1. **Cold Page Faults (Demand Paging):** When you call `malloc()`, modern operating systems (Linux, Windows) do not allocate physical RAM frames immediately; they allocate virtual address ranges. Physical DRAM frames are assigned only when a core triggers a **Page Fault** upon writing to a page for the first time.
2. **Cold CPU Caches:** On the first execution, L1, L2, and L3 caches contain no application data. Every memory request is a high-latency DRAM read ($60\text{--}100\,\text{ns}$). In subsequent runs, hot data resides in L1/L2 caches ($1\text{--}4\,\text{ns}$).
3. **OS CPU Frequency Scaling (P-States & Turbo Boost):** Modern CPUs idle at low power frequencies (e.g., $1.2\,\text{GHz}$). When a heavy parallel compute kernel starts, the OS kernel's CPU governor (such as Linux `schedutil` or `powersave`) takes several milliseconds to detect the load and ramp frequencies up to max Turbo Boost ($4.5\text{--}5.5\,\text{GHz}$).
4. **AVX Frequency Throttling (Intel AVX License Levels):** Executing heavy 512-bit AVX-512 instructions consumes substantial current. Intel CPUs dynamically step down their clock multiplier to stay within Thermal Design Power (TDP) limits. This transition takes several microseconds.

```
       CPU EXECUTION LATENCY TIMELINE
       
  Start Run 1 (Cold):   [ Page Faults ] [ Cache Misses ] [ Frequency Ramp ] -> High Latency (Slow)
  Start Run 2 (Warm):   [ Hot L1/L2 ] [ Steady Turbo Boost State ] ---------> True Hardware Limit!
```

---

## 5. Compilable C Code Benchmark

### 5.1 Program: Demonstration of Alignment, SIMD Reduction, and Warmup Effects
This benchmark measures:
1. The latency difference between **Cold Start** and **Warm Steady-State** execution.
2. The performance difference between **Aligned 64-Byte Access** and **Unaligned Memory Access**.
3. Vector reduction throughput.

Save this file as `omp_simd_alignment_demo.c`:

```c
/* =========================================================================
 * File: omp_simd_alignment_demo.c
 * Description: Demonstrates:
 *              1. SIMD reduction and horizontal register folds
 *              2. Memory alignment via aligned_alloc and aligned() clause
 *              3. The performance cost of unaligned split-cache-line accesses
 *              4. The Benchmark Warmup phenomenon
 * Compilation:
 *   gcc -O3 -fopenmp -march=native omp_simd_alignment_demo.c -o simd_align_demo
 * Execution:
 *   ./simd_align_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <omp.h>

#define N 16000000 // 16 Million doubles (~128 MB)
#define NUM_WARMUP_RUNS 3
#define NUM_BENCHMARK_RUNS 5

/* SIMD Dot Product with Aligned Contract */
double dot_product_aligned(const double * __restrict__ a,
                           const double * __restrict__ b,
                           int n) {
    double total = 0.0;

    /* Assert to compiler that pointers are 64-byte aligned */
    #pragma omp simd reduction(+ : total) aligned(a, b : 64)
    for (int i = 0; i < n; i++) {
        total += a[i] * b[i];
    }
    return total;
}

/* SIMD Dot Product with Unaligned Pointers */
double dot_product_unaligned(const double * __restrict__ a,
                             const double * __restrict__ b,
                             int n) {
    double total = 0.0;

    /* NO alignment clause: Compiler must use unaligned VMOVUPD */
    #pragma omp simd reduction(+ : total)
    for (int i = 0; i < n; i++) {
        total += a[i] * b[i];
    }
    return total;
}

int main(void) {
    printf("============================================================\n");
    printf("    SIMD Reduction, Alignment & Benchmark Warmup Demo       \n");
    printf("============================================================\n");
    printf("Vector Length: %d doubles (%.2f MB per array)\n\n",
           N, (double)(N * sizeof(double)) / (1024 * 1024));

    /* 1. Allocate 64-byte aligned memory buffers */
    size_t alloc_bytes = ((N * sizeof(double) + 63) / 64) * 64;
    double *A_aligned = (double*)aligned_alloc(64, alloc_bytes);
    double *B_aligned = (double*)aligned_alloc(64, alloc_bytes);

    printf("[Memory Architecture Inspection]\n");
    printf("  Pointer A Address: %p (%s 64-byte aligned)\n",
           (void*)A_aligned, ((uintptr_t)A_aligned % 64 == 0) ? "PERFECTLY" : "NOT");
    printf("  Pointer B Address: %p (%s 64-byte aligned)\n\n",
           (void*)B_aligned, ((uintptr_t)B_aligned % 64 == 0) ? "PERFECTLY" : "NOT");

    /* -------------------------------------------------------------
     * DEMO 1: The Benchmark Warmup Phenomenon
     * ------------------------------------------------------------- */
    printf("--- 1. Cold Start vs. Warm Steady-State Execution ---\n");

    double t0 = omp_get_wtime();
    /* FIRST TOUCH: Triggers OS Page Faults and Cold L1/L2 Cache Misses */
    for (int i = 0; i < N; i++) {
        A_aligned[i] = 1.0;
        B_aligned[i] = 2.0;
    }
    double t1 = omp_get_wtime();
    printf("  Run 1 (Cold Start - Page Faults & Cold Caches) : %8.4f ms\n",
           (t1 - t0) * 1000.0);

    t0 = omp_get_wtime();
    /* SECOND TOUCH: Pages mapped, steady-state CPU Turbo Boost */
    for (int i = 0; i < N; i++) {
        A_aligned[i] = 1.0;
        B_aligned[i] = 2.0;
    }
    t1 = omp_get_wtime();
    printf("  Run 2 (Warm Steady-State - Pages Already Mapped): %8.4f ms\n\n",
           (t1 - t0) * 1000.0);

    /* -------------------------------------------------------------
     * DEMO 2: SIMD Reduction Benchmark (Aligned vs Unaligned)
     * ------------------------------------------------------------- */
    printf("--- 2. Benchmarking Aligned vs Unaligned SIMD Reduction ---\n");

    /* Create unaligned pointers (Offset by 1 double = 8 bytes, breaking 64B alignment) */
    const double *A_unaligned = A_aligned + 1;
    const double *B_unaligned = B_aligned + 1;
    int unaligned_n = N - 1;

    /* Warmup CPU execution pipelines */
    for (int w = 0; w < NUM_WARMUP_RUNS; w++) {
        dot_product_aligned(A_aligned, B_aligned, N);
        dot_product_unaligned(A_unaligned, B_unaligned, unaligned_n);
    }

    /* Timed Aligned Runs */
    double time_aligned_sum = 0.0;
    double res_aligned = 0.0;
    for (int r = 0; r < NUM_BENCHMARK_RUNS; r++) {
        t0 = omp_get_wtime();
        res_aligned = dot_product_aligned(A_aligned, B_aligned, N);
        t1 = omp_get_wtime();
        time_aligned_sum += (t1 - t0);
    }
    double avg_aligned = (time_aligned_sum / NUM_BENCHMARK_RUNS) * 1000.0;

    /* Timed Unaligned Runs */
    double time_unaligned_sum = 0.0;
    double res_unaligned = 0.0;
    for (int r = 0; r < NUM_BENCHMARK_RUNS; r++) {
        t0 = omp_get_wtime();
        res_unaligned = dot_product_unaligned(A_unaligned, B_unaligned, unaligned_n);
        t1 = omp_get_wtime();
        time_unaligned_sum += (t1 - t0);
    }
    double avg_unaligned = (time_unaligned_sum / NUM_BENCHMARK_RUNS) * 1000.0;

    printf("  Aligned SIMD Reduction   : %8.4f ms | Dot Product = %.1f\n",
           avg_aligned, res_aligned);
    printf("  Unaligned SIMD Reduction : %8.4f ms | Dot Product = %.1f\n",
           avg_unaligned, res_unaligned);
    printf("  -> Aligned Speedup Advantage: %.2fx Faster\n",
           avg_unaligned / avg_aligned);

    free(A_aligned);
    free(B_aligned);

    printf("\n============================================================\n");
    printf("Benchmark Complete.\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp -march=native omp_simd_alignment_demo.c -o simd_align_demo
$ ./simd_align_demo
```

```text
============================================================
    SIMD Reduction, Alignment & Benchmark Warmup Demo       
============================================================
Vector Length: 16000000 doubles (122.07 MB per array)

[Memory Architecture Inspection]
  Pointer A Address: 0x55b410a40000 (PERFECTLY 64-byte aligned)
  Pointer B Address: 0x55b418a40000 (PERFECTLY 64-byte aligned)

--- 1. Cold Start vs. Warm Steady-State Execution ---
  Run 1 (Cold Start - Page Faults & Cold Caches) :  38.4120 ms
  Run 2 (Warm Steady-State - Pages Already Mapped):  11.1200 ms

--- 2. Benchmarking Aligned vs Unaligned SIMD Reduction ---
  Aligned SIMD Reduction   :  10.8910 ms | Dot Product = 32000000.0
  Unaligned SIMD Reduction :  12.4180 ms | Dot Product = 31999998.0
  -> Aligned Speedup Advantage: 1.14x Faster

============================================================
Benchmark Complete.
```

### Critical Observations:
1. **The Warmup Reality:** Cold Start took **$38.41\,\text{ms}$**, while Warm Steady-State took **$11.12\,\text{ms}$** ($3.45\times$ faster!). Measuring without warmup captures OS kernel paging latency rather than CPU vector speed.
2. **Alignment Advantage:** The aligned loop was **$1.14\times$ faster** than the unaligned loop because the hardware avoided split-cache-line stalls across 16 million memory loads.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Horizontal Vector Reduction Latency
For a SIMD register of width $V$ (e.g., $V = 8$ for AVX2 `float`), a horizontal reduction requires:

$$\text{Horizontal Tree Rounds} = \lceil \log_2 V \rceil$$

Inside the CPU, this compiles into $\log_2(V)$ intra-register shuffle and add operations:
- For AVX (4 doubles): $\log_2(4) = 2$ shuffle steps ($\approx 6$ clock cycles).
- For AVX-512 (16 floats): $\log_2(16) = 4$ shuffle steps ($\approx 12$ clock cycles).

Because this $O(\log V)$ latency is executed only **once** after all $N$ iterations finish, its amortized cost per iteration approaches zero:

$$\text{Effective Reduction Overhead per Iteration} = \frac{O(\log_2 V)}{N} \to 0$$

---

## 7. Advantages & Limitations

### The `aligned` Clause
- **Advantages:** Strips away peel loops; guarantees single-cycle aligned vector instructions (`vmovapd`); eliminates split-cache-line pipeline stalls.
- **Dangers:** If you assert `aligned(A : 64)` on a pointer that is *not actually aligned*, the CPU executes `vmovaps` on an unaligned address, triggering an immediate hardware **General Protection Fault (Segmentation Fault / Crash)**!

---

## 8. Formal Definitions for Exam

- **SIMD Reduction:** A vector computing operation where individual vector lanes accumulate partial sums independently in a vector register during the loop, followed by a horizontal tree reduction at loop exit.
- **Horizontal Vector Reduction:** An intra-register reduction algorithm that uses vector shuffle and permutation instructions to sum the individual lanes of a single vector register into a scalar value in $O(\log_2 V)$ steps.
- **`aligned` Clause:** An OpenMP SIMD clause (`aligned(list : alignment)`) asserting that specified pointers are aligned to physical memory byte boundaries matching `alignment`.
- **Split-Cache-Line Access:** A memory read or write where an unaligned vector payload spans across two distinct 64-byte hardware cache lines, requiring the CPU memory subsystem to perform two cache accesses and stitch the data together.
- **Benchmark Warmup:** The practice of executing a computational kernel several times prior to recording timing metrics to ensure OS page tables are populated, caches are hot, and the CPU has reached steady-state Turbo Boost frequencies.

---

## 9. Comparison Table

### Summary Comparison: Aligned vs. Unaligned Vector Memory Operations

| Characteristic | Aligned Vector Memory Access | Unaligned Vector Memory Access |
|:---|:---:|:---:|
| **Physical Starting Address** | Address is multiple of 32 or 64 bytes (`addr % 64 == 0`) | Arbitrary memory address |
| **x86 Assembly Opcode** | `VMOVAPS` / `VMOVAPD` (Aligned Packed) | `VMOVUPS` / `VMOVUPD` (Unaligned Packed) |
| **Compiler Loop Structure** | Pure vector core loop (No peel loop required) | Peel loop + Vector loop + Epilogue loop |
| **Cache Line Behavior** | Exactly 1 cache line per load | Can straddle 2 cache lines (Split load) |
| **Crash Risk if Misaligned?** | **YES (Segmentation Fault / SIGSEGV)** | **NO** (Hardware handles smoothly) |

---

## 10. Solved Analytical Problems

### Problem 1: Calculating Alignment Validity
**Question:** An engineer allocates an array using `malloc`:
```c
double *ptr = (double*)malloc(1000 * sizeof(double));
```
Assuming `malloc` returns pointer address `0x7ffee0400030`:
1. Is `ptr` aligned to 16 bytes?
2. Is `ptr` aligned to 32 bytes (AVX2)?
3. Is `ptr` aligned to 64 bytes (AVX-512 / Cache Line)?
4. What happens if the engineer marks this pointer as `aligned(ptr : 64)`?

**Solution:**
1. **16-byte alignment:**
   $0x30 = 48$ in decimal. $48 / 16 = 3$ (Remainder 0) $\implies$ **YES, aligned to 16 bytes**.
2. **32-byte alignment:**
   $48 / 32 = 1.5$ (Remainder 16) $\implies$ **NO, misaligned for AVX2**.
3. **64-byte alignment:**
   $48 / 64 = 0$ (Remainder 48) $\implies$ **NO, misaligned for AVX-512**.
4. **Consequence of `aligned(ptr : 64)`:**
   The compiler will emit aligned load instruction `VMOVAPD`. When the CPU execution unit encounters an unaligned address (`0x30`), the hardware triggers an unaligned memory fault $\implies$ **Immediate Program Crash (Segmentation Fault)**.

---

### Problem 2: Designing a Complete Scientific Microbenchmark Protocol
**Question:** List the mandatory 4-step protocol required to benchmark an OpenMP / SIMD kernel accurately without measurement distortion.

**Solution:**
1. **Step 1: First-Touch Initialization:**
   Allocate arrays and initialize all data in a separate pre-loop. This forces the OS kernel to map physical DRAM frames, eliminating page fault overhead from subsequent timings.
2. **Step 2: Microbenchmark Warmup:**
   Execute the target compute kernel $2$ to $5$ times without recording timestamps. This warms L1/L2 caches and forces CPU frequency governors into high-power Turbo Boost states.
3. **Step 3: High-Precision Wall-Clock Timing:**
   Use monotonic high-resolution timers (`omp_get_wtime()`) rather than CPU process clocks (`clock()`).
4. **Step 4: Statistical Repetition:**
   Execute the kernel for $K \ge 5$ iterations, computing the **minimum** or **median** runtime (discarding outlier spikes caused by OS context switches and background interrupts).

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why do modern CPUs support unaligned vector loads (`VMOVUPS`) without crashing, if aligned loads are faster?
**Answer:** Because real-world applications frequently process unaligned buffers (such as parsing network packets or streaming video frames). Microarchitecture designers added hardware circuitry to x86 and ARM chips allowing `VMOVUPS` to execute at nearly the same speed as `VMOVAPS` *provided the memory access does not cross a 64-byte cache line boundary*. However, when an unaligned load crosses a cache line boundary, hardware stalls are unavoidable.

### Q2: Why is `aligned_alloc` preferred over custom pointer arithmetic?
**Answer:** Custom pointer bumping (e.g. `char *buf = malloc(N + 64); double *p = (buf + 63) & ~63;`) works, but tracking the original `buf` pointer to pass to `free()` is error-prone. In contrast, C11 `aligned_alloc(alignment, size)` returns a naturally aligned pointer that can be passed directly to standard `free()`.

### Q3: Does OpenMP provide an aligned allocation API?
**Answer:** Yes! OpenMP 5.0 introduced **`omp_alloc()`** and **`omp_aligned_alloc()`**, allowing developers to allocate memory with explicit alignment and even specify memory types (such as High-Bandwidth Memory - HBM or Non-Volatile Memory - NVDIMM).

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    SIMD ALIGNMENT & WARMUP CHEAT SHEET                             |
+----------------------------------------------------------------------------------------------------+
| Directive Clause       | #pragma omp simd aligned(ptr : 64) reduction(+ : sum)                     |
| Memory Allocation      | aligned_alloc(64, bytes) (C11) or posix_memalign (POSIX)                  |
| Misalignment Danger    | Marking unaligned pointer as aligned -> CRASH (General Protection Fault)  |
| Horizontal Fold        | O(log2 V) shuffle tree across register lanes; executed ONCE at loop exit. |
+----------------------------------------------------------------------------------------------------+
|                                   THE 4 RULES OF BENCHMARKING                                      |
+----------------------------------------------------------------------------------------------------+
| 1. First-Touch All Data| Force OS demand-paging page faults before starting timers.                |
| 2. Run Warmup Passes   | Warm up L1/L2 caches and ramp CPU governor to maximum Turbo Boost.        |
| 3. Use omp_get_wtime() | Monotonic wall-clock timer (Never use clock() for multi-threaded code!).  |
| 4. Repeat and Average  | Run K >= 5 times and report minimum or median execution time.             |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain how SIMD vector reductions execute vertical lane accumulations during loops and horizontal tree reductions at loop exit.
2. Allocate 64-byte aligned memory using C11 `aligned_alloc` and apply the `aligned` clause to OpenMP SIMD directives.
3. Diagram and explain the Split-Cache-Line performance penalty of unaligned memory loads.
4. Explain why cold-start measurements are distorted by page faults, cold caches, and CPU frequency scaling.
5. Structure a rigorous scientific microbenchmark protocol featuring first-touch initialization, warmup iterations, and monotonic timing.
