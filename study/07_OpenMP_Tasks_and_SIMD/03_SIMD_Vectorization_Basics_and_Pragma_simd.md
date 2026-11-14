# 03. SIMD Execution Model, #pragma omp simd, and Hardware Vector Extensions

---

## 1. The Basic Idea

In shared-memory multi-core computing, we utilize multiple processor cores concurrently through multithreading (`#pragma omp parallel`). However, modern microprocessors possess a second, orthogonal dimension of hardware parallelism located **inside each individual CPU core**: the **Vector Execution Units** operating on the **Single Instruction, Multiple Data (SIMD)** principle.

While a scalar CPU instruction adds two numbers:
$$c = a + b$$
A SIMD vector instruction loads an entire **packed vector register** containing $4$, $8$, or $16$ distinct floating-point numbers and adds them all simultaneously in a single CPU clock cycle:

$$\begin{bmatrix} c_0 \\ c_1 \\ c_2 \\ c_3 \end{bmatrix} = \begin{bmatrix} a_0 \\ a_1 \\ a_2 \\ a_3 \end{bmatrix} + \begin{bmatrix} b_0 \\ b_1 \\ b_2 \\ b_3 \end{bmatrix}$$

OpenMP 4.0 introduced the **`#pragma omp simd`** directive to give developers direct, portable control over hardware vectorization. It instructs the compiler to transform a sequential loop into a vectorized loop utilizing the CPU's widest SIMD registers, without requiring manual architecture-specific assembly or proprietary intrinsics.

```
       SCALAR INSTRUCTION EXECUTION (Traditional Core)
       Clock Cycle 1: c[0] = a[0] + b[0]
       Clock Cycle 2: c[1] = a[1] + b[1]
       Clock Cycle 3: c[2] = a[2] + b[2]
       Clock Cycle 4: c[3] = a[3] + b[3]  (Total: 4 Cycles for 4 elements)

       SIMD VECTOR INSTRUCTION EXECUTION (e.g., AVX 256-bit Register)
       Clock Cycle 1: [ c[0] | c[1] | c[2] | c[3] | c[4] | c[5] | c[6] | c[7] ]
                       = [ a[0..7] ] + [ b[0..7] ]  (Total: 1 Cycle for 8 elements!)
```

```
          THE TWO ORTHOGONAL AXES OF MODERN CPU PARALLELISM
          
                 AXIS 1: MULTI-CORE THREAD LEVEL (OpenMP Parallel)
             Core 0            Core 1            Core 2            Core 3
        +--------------+  +--------------+  +--------------+  +--------------+
        | SIMD Vectors |  | SIMD Vectors |  | SIMD Vectors |  | SIMD Vectors |
        | [512-bit ZMM]|  | [512-bit ZMM]|  | [512-bit ZMM]|  | [512-bit ZMM]|
        +--------------+  +--------------+  +--------------+  +--------------+
                 AXIS 2: DATA-LEVEL VECTORIZATION (#pragma omp simd)
```

Combining both axes (`#pragma omp parallel for simd`) exploits the full theoretical peak FLOP/s of modern supercomputer nodes.

---

## 2. Why We Need It: Overcoming Compiler Conservatism

Compilers such as GCC, Clang, and Intel oneAPI have built-in **Auto-Vectorization** optimization passes (`-O3 -ftree-vectorize`). However, auto-vectorizers are strictly bound by the ISO C/C++ abstract machine rules: the compiler **must prove that vectorizing a loop will never, under any circumstance, alter the sequential program behavior**.

In real-world C code, this strict requirement frequently prevents auto-vectorization due to **Pointer Aliasing**:

```c
void add_arrays(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i]; /* Will the compiler vectorize this? */
    }
}
```

### The Pointer Aliasing Dilemma
The compiler examines the pointers `a`, `b`, and `c`. In standard C:
- What if the caller passes `c = a + 1`?
- Then `c[0]` is the exact same memory location as `a[1]`.
- Modifying `c[0]` alters `a[1]` for the next iteration! This is a **Loop-Carried Memory Dependency**.
- If the compiler executes 8 iterations simultaneously in a SIMD vector register, `c[0]` through `c[7]` are computed using the *old* values of `a[1..7]`, producing an incorrect result!

Because the compiler cannot prove that `a`, `b`, and `c` do not overlap, it errs on the side of caution: **it refuses to vectorize the loop**, leaving the code in slow scalar assembly!

### The `#pragma omp simd` Solution
When the programmer writes:
```c
#pragma omp simd
for (int i = 0; i < n; i++) {
    c[i] = a[i] + b[i];
}
```
The directive acts as a binding contract. The programmer asserts: *"I guarantee that iterations are independent and that there are no loop-carried dependencies that prevent vectorization."*
The compiler ignores potential aliasing risks and immediately emits packed vector instructions.

---

## 3. How It Works: Hardware Vector Extensions

Over the past three decades, microarchitectures have continuously doubled vector register widths to scale compute performance without increasing clock frequencies:

| Architecture Family | Register Name | Register Bit-Width | Single Precision (`float`) Capacity | Double Precision (`double`) Capacity | Introduced In |
|:---|:---:|:---:|:---:|:---:|:---:|
| **SSE / SSE4.2** | `XMM0`–`XMM15` | **128 bits** (16 bytes) | **4 floats** | **2 doubles** | Pentium III / Core 2 (1999) |
| **AVX / AVX2** | `YMM0`–`YMM15` | **256 bits** (32 bytes) | **8 floats** | **4 doubles** | Sandy Bridge / Haswell (2011) |
| **AVX-512** | `ZMM0`–`ZMM31` | **512 bits** (64 bytes) | **16 floats** | **8 doubles** | Skylake-X / Xeon Scalable (2017) |
| **ARM NEON** | `V0`–`V31` | **128 bits** (16 bytes) | **4 floats** | **2 doubles** | ARM Cortex-A (2005) |
| **ARM SVE** | `Z0`–`Z31` | Scalable ($128\text{--}2048$ bits) | Dynamic ($4\text{--}64$ floats) | Dynamic ($2\text{--}32$ doubles) | Fujitsu A64FX / Neoverse V1 |

```
                       HARDWARE VECTOR REGISTER SCALING
                       
  SSE (128-bit XMM):
  [ float 0 | float 1 | float 2 | float 3 ] (4 x 32-bit)

  AVX2 (256-bit YMM):
  [ f0 | f1 | f2 | f3 | f4 | f5 | f6 | f7 ] (8 x 32-bit)

  AVX-512 (512-bit ZMM):
  [ f0 | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10| f11| f12| f13| f14| f15] (16 x 32-bit)
```

---

## 4. Loop Vectorization Structure: Peel, Vector, and Epilogue

When a loop with trip count $N$ is vectorized by `#pragma omp simd` with vector register width $V$ (e.g., $V = 8$ for AVX `float`), $N$ is rarely an exact multiple of $V$, and memory addresses may not be aligned to 64-byte boundaries.

The compiler transforms the single loop into **three distinct execution stages**:

```
           ANATOMY OF A VECTORIZED LOOP IN COMPILED ASSEMBLY
           
  Memory Address: [ Unaligned Head ] [ 64-Byte Aligned Vector Core ] [ Remainder Tail ]
  Iterations:     0 ... K             K+1 .................... M      M+1 ........ N-1
                       |                             |                        |
                       v                             v                        v
                [ PEEL LOOP ]               [ VECTOR LOOP ]             [ EPILOGUE LOOP ]
                Scalar execution            Packed SIMD instructions    Scalar execution
                until address hits          (AVX/AVX-512 steps of V)    for remaining
                64-byte boundary.                                       N mod V elements.
```

1. **The Peel Loop (Scalar Prologue):**
   Executes iterations $0, 1, \dots$ one by one using standard scalar instructions until the target array memory pointer reaches a 32-byte or 64-byte aligned address.
2. **The Vectorized Core Loop (SIMD Engine):**
   Executes the vast majority of iterations in strides of $V$ (e.g., 8 or 16 iterations per step) using packed instructions (`VADDPS`, `VMULPD`).
3. **The Epilogue Loop (Scalar Remainder):**
   Handles the remaining $N \pmod V$ iterations that do not fill a complete vector register, using scalar instructions or masked vector stores.

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of `#pragma omp simd` vs Scalar Execution
This program compares scalar array execution against `#pragma omp simd` and combined multi-core SIMD (`#pragma omp parallel for simd`).

Save this file as `omp_simd_intro.c`:

```c
/* =========================================================================
 * File: omp_simd_intro.c
 * Description: Demonstrates OpenMP SIMD vectorization:
 *              1. Sequential scalar computation
 *              2. Core-level vectorization via #pragma omp simd
 *              3. Combined multi-core vectorization via #pragma omp parallel for simd
 * Compilation:
 *   gcc -O3 -fopenmp -fopt-info-vec omp_simd_intro.c -o omp_simd_intro -lm
 * Execution:
 *   ./omp_simd_intro
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <omp.h>

#define N 16000000 // 16 Million elements (~64 MB per array)
#define NUM_THREADS 4

/* Disabling vectorization on this function to establish true scalar baseline */
__attribute__((optimize("no-tree-vectorize")))
void compute_scalar(float * __restrict__ a, float * __restrict__ b, float * __restrict__ c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] * 2.5f + b[i] * 1.5f;
    }
}

/* Enforcing SIMD vectorization via OpenMP */
void compute_simd(float *a, float *b, float *c, int n) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        c[i] = a[i] * 2.5f + b[i] * 1.5f;
    }
}

/* Combining Multi-Core Thread Parallelism + Single-Core Vectorization */
void compute_parallel_simd(float *a, float *b, float *c, int n) {
    #pragma omp parallel for simd num_threads(NUM_THREADS)
    for (int i = 0; i < n; i++) {
        c[i] = a[i] * 2.5f + b[i] * 1.5f;
    }
}

int main(void) {
    printf("============================================================\n");
    printf("     OpenMP SIMD Execution Model & Vectorization Demo       \n");
    printf("============================================================\n");
    printf("Vector Size: %d single-precision floats (%.2f MB)\n\n",
           N, (double)(N * sizeof(float)) / (1024 * 1024));

    /* Allocate memory aligned to 64-byte cache/AVX-512 boundaries */
    float *A = (float*)aligned_alloc(64, N * sizeof(float));
    float *B = (float*)aligned_alloc(64, N * sizeof(float));
    float *C = (float*)aligned_alloc(64, N * sizeof(float));

    for (int i = 0; i < N; i++) {
        A[i] = 1.0f;
        B[i] = 2.0f;
        C[i] = 0.0f;
    }

    double t0, t1;

    /* -------------------------------------------------------------
     * 1. SCALAR BASELINE (No vectorization)
     * ------------------------------------------------------------- */
    t0 = omp_get_wtime();
    compute_scalar(A, B, C, N);
    t1 = omp_get_wtime();
    double time_scalar = t1 - t0;
    printf("1. Pure Scalar Execution       : %8.4f ms | Sample C[0]=%.1f | Speedup = 1.00x\n",
           time_scalar * 1000.0, C[0]);

    /* -------------------------------------------------------------
     * 2. SINGLE-CORE SIMD (#pragma omp simd)
     * ------------------------------------------------------------- */
    t0 = omp_get_wtime();
    compute_simd(A, B, C, N);
    t1 = omp_get_wtime();
    double time_simd = t1 - t0;
    printf("2. Single-Core #pragma omp simd: %8.4f ms | Sample C[0]=%.1f | Speedup = %.2fx\n",
           time_simd * 1000.0, C[0], time_scalar / time_simd);

    /* -------------------------------------------------------------
     * 3. MULTI-CORE + SIMD (#pragma omp parallel for simd)
     * ------------------------------------------------------------- */
    t0 = omp_get_wtime();
    compute_parallel_simd(A, B, C, N);
    t1 = omp_get_wtime();
    double time_par_simd = t1 - t0;
    printf("3. Multi-Core (%d Th) + SIMD   : %8.4f ms | Sample C[0]=%.1f | Speedup = %.2fx\n",
           NUM_THREADS, time_par_simd * 1000.0, C[0], time_scalar / time_par_simd);

    printf("============================================================\n");
    free(A);
    free(B);
    free(C);
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp -fopt-info-vec omp_simd_intro.c -o omp_simd_intro -lm
```
*Compiler Vectorization Report Output:*
```text
omp_simd_intro.c:31:5: optimized: loop vectorized using 32-byte vectors
omp_simd_intro.c:40:5: optimized: loop vectorized using 32-byte vectors
```

*Execution Output:*
```bash
$ ./omp_simd_intro
```
```text
============================================================
     OpenMP SIMD Execution Model & Vectorization Demo       
============================================================
Vector Size: 16000000 single-precision floats (61.04 MB)

1. Pure Scalar Execution       :  36.8120 ms | Sample C[0]=5.5 | Speedup = 1.00x
2. Single-Core #pragma omp simd:   9.2140 ms | Sample C[0]=5.5 | Speedup = 4.00x
3. Multi-Core (4 Th) + SIMD   :   2.8910 ms | Sample C[0]=5.5 | Speedup = 12.73x
============================================================
```

### Key Analytical Takeaways:
1. **Single-Core Vector Speedup:** On a 256-bit AVX CPU, `compute_simd` achieved exactly **$4.00\times$ speedup** over scalar code on a single CPU core!
2. **Multiplicative Speedup:** When combining 4 threads with SIMD (`parallel for simd`), speedup jumped to **$12.73\times$** ($4\text{ cores} \times 4\text{ SIMD factor} \approx 16\times$ theoretical peak, constrained by memory bandwidth).

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Theoretical Vector Speedup Factor ($S_{\text{SIMD}}$)
The maximum theoretical speedup achievable via SIMD vectorization is strictly dictated by the **Vector Register Width** $W_{\text{reg}}$ divided by the **Data Type Width** $W_{\text{type}}$:

$$S_{\text{SIMD, max}} = V_{\text{width}} = \frac{W_{\text{register}}}{W_{\text{data\_type}}}$$

#### Examples:
- **AVX2 (256-bit registers):**
  - For `double` (64-bit): $V = 256 / 64 = \mathbf{4\times \text{ speedup}}$
  - For `float` (32-bit): $V = 256 / 32 = \mathbf{8\times \text{ speedup}}$
  - For `short` (16-bit int): $V = 256 / 16 = \mathbf{16\times \text{ speedup}}$
- **AVX-512 (512-bit registers):**
  - For `float` (32-bit): $V = 512 / 32 = \mathbf{16\times \text{ speedup}}$

---

### 6.2 Total Speedup under Combined Parallelism
When combining $p$ thread cores with vector width $V$ across problem with parallel fraction $f$:

$$S(p, V) = \frac{1}{(1 - f) + \frac{f}{p \cdot V}}$$

If $f = 0.98$, running on $p = 8$ cores with AVX-512 single-precision ($V = 16$):
$$p \cdot V = 8 \times 16 = 128\times \text{ compute concurrency!}$$
$$S(8, 16) = \frac{1}{0.02 + \frac{0.98}{128}} = \frac{1}{0.02 + 0.00765} = \frac{1}{0.02765} \approx \mathbf{36.16\times \text{ speedup!}}$$

---

## 7. Advantages & Limitations

### Advantages of `#pragma omp simd`
1. **Hardware Acceleration without Assembly:** Eliminates the need to write raw intrinsics (`_mm256_add_ps`).
2. **Overrides Pointer Aliasing:** Instructs the compiler to vectorize even when pointer overlap cannot be disproven statically.
3. **Architecture Portability:** The identical C loop compiles to SSE on older x86, AVX-512 on modern Xeon servers, and NEON/SVE on ARM chips.

### Limitations & Traps
1. **Vectorization Hazards:** If the programmer asserts `#pragma omp simd` on a loop that *actually does* contain a true cross-iteration dependency ($A[i] = A[i-1] + B[i]$), the compiler blindly vectorizes it, silently producing corrupted data!
2. **Control Flow Divergence:** Loops containing heavy branching (`if/else`) vectorize poorly because SIMD hardware must compute both paths and use blend masks.
3. **Memory Bandwidth Bottleneck:** SIMD processes numbers so fast that memory-bound loops hit the DRAM bandwidth wall (Roofline limit) almost immediately.

---

## 8. Formal Definitions for Exam

- **SIMD (Single Instruction, Multiple Data):** A hardware parallel computing architecture classified under Flynn's Taxonomy where a single processor execution unit executes one instruction simultaneously across multiple data elements packed into vector registers.
- **`#pragma omp simd`:** An OpenMP directive instructing the compiler to vectorize the enclosed canonical `for` loop using hardware SIMD vector instructions, assuming no loop-carried dependencies exist.
- **Vector Register:** A wide CPU register (e.g., 128-bit XMM, 256-bit YMM, 512-bit ZMM) capable of holding multiple scalar values simultaneously.
- **Pointer Aliasing:** The condition in C/C++ where two or more pointers reference overlapping regions of memory, preventing auto-vectorization due to potential loop-carried dependencies.
- **Peel Loop:** A scalar prologue loop generated by vectorizing compilers to process initial unaligned array elements until memory addresses reach optimal cache/SIMD alignment boundaries.
- **Epilogue Loop:** A scalar remainder loop generated by vectorizing compilers to process the trailing $N \pmod V$ iterations that do not fill a full vector register.

---

## 9. Comparison Table

### Multithreading vs. SIMD Vectorization

| Dimension | Multithreading (`#pragma omp parallel`) | SIMD Vectorization (`#pragma omp simd`) |
|:---|:---|:---|
| **Hardware Entity** | Multiple CPU Cores / SMT Hardware Threads | Vector Execution Units (ALUs) inside ONE core |
| **Control Flow** | Asynchronous (Each thread has private Program Counter)| Synchronous (Single instruction across all vector lanes)|
| **Communication** | Via Shared RAM and L3 Cache | Direct register packing inside CPU core |
| **Memory Bandwidth Cost**| Heavy (Cache coherence bus traffic) | Low (Contiguous streaming loads) |
| **Overhead to Launch** | $\approx 1\text{--}5\,\mu\text{s}$ (Fork-join team coordination) | **$0\,\mu\text{s}$ (Single instruction stream)** |

---

## 10. Solved Analytical Problems

### Problem 1: Loop-Carried Dependency Identification
**Question:** Analyze the following two loops. Determine which loop can be legally vectorized with `#pragma omp simd` and which cannot, justifying your answer using distance vectors.

```c
/* Loop 1 */
for (int i = 1; i < N; i++) {
    A[i] = A[i-1] + B[i];
}

/* Loop 2 */
for (int i = 0; i < N - 1; i++) {
    A[i] = A[i+1] + B[i];
}
```

**Solution:**
1. **Loop 1 Analysis:**
   - Iteration $i$ writes to `A[i]`.
   - Iteration $i+1$ reads from `A[i]`.
   - This is a **True Data Dependency (RAW - Read-After-Write)** with distance $+1$.
   - **Verdict: ILLEGAL TO VECTORIZE.** If the hardware processes 4 elements simultaneously ($i = 1, 2, 3, 4$), $A[2]$ will read the *old* value of $A[1]$ before the vector instruction finishes writing the *new* value of $A[1]$. Vectorizing this loop produces mathematically wrong answers.
2. **Loop 2 Analysis:**
   - Iteration $i$ reads from `A[i+1]`.
   - Iteration $i+1$ writes to `A[i+1]`.
   - This is an **Anti-Dependency (WAR - Write-After-Read)** with distance $-1$.
   - **Verdict: LEGAL TO VECTORIZE WITH SIMD.** In a SIMD register, all loads are executed into vector registers *before* the vector stores write back to memory! Therefore, reading `A[i+1]` before writing `A[i]` is safe within a vector lane.

---

### Problem 2: Calculating Vector Register Utilization
**Question:** An engineer executes a loop with $N = 35$ iterations of double-precision floating-point numbers on an AVX-512 processor (512-bit registers). 
1. How many `double` values fit in an AVX-512 vector register?
2. How many iterations are processed in the main vector loop?
3. How many iterations are relegated to the scalar epilogue loop?

**Solution:**
1. **Vector Register Capacity ($V$):**
   $$V = \frac{512 \text{ bits}}{64 \text{ bits/double}} = \mathbf{8 \text{ doubles per register}}$$
2. **Main Vector Loop:**
   $$\text{Vector Chunks} = \left\lfloor \frac{35}{8} \right\rfloor = 4 \text{ chunks}$$
   $$\text{Iterations Processed in Vector Loop} = 4 \times 8 = \mathbf{32 \text{ iterations}} \quad (i = 0 \dots 31)$$
3. **Scalar Epilogue Loop:**
   $$\text{Remainder Iterations} = 35 \pmod 8 = \mathbf{3 \text{ iterations}} \quad (i = 32, 33, 34)$$

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why does `#pragma omp simd` not create any threads?
**Answer:** Because SIMD operates **within a single execution core**. It does not spawn OS threads or awaken thread pools. Instead, it instructs the compiler's code generator to emit packed SIMD assembly instructions (such as `vaddps` or `vmulpd`) instead of scalar instructions (`addss` or `mulsd`). The single calling thread executes these vector instructions sequentially on its local vector ALUs.

### Q2: What is the difference between `#pragma omp for` and `#pragma omp for simd`?
**Answer:**
- `#pragma omp for` divides loop iterations across multiple **CPU cores** (multithreading). Inside each core, iterations are still computed sequentially one scalar number at a time.
- `#pragma omp for simd` performs **both**: it distributes chunks of iterations across multiple CPU cores, and *within each core*, it vectorizes the assigned chunk across the core's SIMD execution units.

### Q3: What is the purpose of the C99 `restrict` keyword in vectorization?
**Answer:** The `restrict` keyword (`float * __restrict__ ptr`) is a C language contract promising the compiler that the pointer `ptr` does not point to memory that overlaps with any other pointer in that scope. This eliminates the pointer aliasing dilemma, allowing the compiler to auto-vectorize loops without needing `#pragma omp simd`.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    OPENMP SIMD CHEAT SHEET                                         |
+----------------------------------------------------------------------------------------------------+
| Directive              | #pragma omp simd                                                          |
| Hardware Target        | Vector registers inside ONE CPU core (XMM, YMM, ZMM, NEON).               |
| Thread Creation        | ZERO threads created! Operates on a single thread.                        |
| Register Widths        | SSE = 128-bit (4 floats)   | AVX2 = 256-bit (8 floats)                    |
|                        | AVX-512 = 512-bit (16 fl) | ARM NEON = 128-bit (4 floats)                 |
| Combined Construct     | #pragma omp parallel for simd (Multi-core team + Core vectorization)       |
+----------------------------------------------------------------------------------------------------+
|                                  LOOP TRANSFORMATION ANATOMY                                       |
+----------------------------------------------------------------------------------------------------+
| 1. Peel Loop           | Scalar loop: Runs initial unaligned elements until memory hits 64B bound. |
| 2. Vector Loop         | Core SIMD loop: Processes data in strides of V (e.g. 8 or 16 per clock).   |
| 3. Epilogue Loop       | Scalar loop: Cleans up the remaining N mod V iterations at the tail.      |
+----------------------------------------------------------------------------------------------------+
|                                   THE CARDINAL HAZARD                                              |
+----------------------------------------------------------------------------------------------------+
| RAW Dependency Danger  | Never put #pragma omp simd on loops with True Dependencies (A[i]=A[i-1])!  |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the architectural difference between multi-core thread parallelism and single-core SIMD vector parallelism.
2. Identify hardware vector register widths across SSE (128-bit), AVX2 (256-bit), and AVX-512 (512-bit).
3. Explain how pointer aliasing prevents auto-vectorization and how `#pragma omp simd` overrides compiler conservatism.
4. Diagram the 3-stage loop structure (peel loop, vector core, epilogue loop) emitted by vectorizing compilers.
5. Calculate maximum theoretical vector speedup $S_{\text{SIMD}}$ based on register bit-width and data type size.
6. Diagnose illegal loop-carried dependencies (RAW) that prohibit SIMD vectorization.
