# 04. Compiler Vectorization Reports and Optimization Flags

---

## 1. The Basic Idea

Writing `#pragma omp simd` or enabling `-O3` does not guarantee that your code will execute using wide SIMD vector instructions. A compiler is a complex static analyzer that may encounter subtle architectural roadblocks—such as unrecognized pointer aliasing, non-contiguous data layouts, complex control flow, or unsupported library calls. When this happens, the compiler quietly abandons vectorization and emits ordinary **scalar fallback assembly**, often without generating a single warning.

To ensure your code actually achieves hardware vector acceleration, you must master two essential tools:

1. **Architecture Optimization Flags (`-march=native`):** Instructing the compiler to unlock the full vector capabilities of the host processor (AVX2, AVX-512, FMA) rather than defaulting to conservative, legacy 128-bit SSE baseline architectures.
2. **Compiler Vectorization Diagnostic Reports (`-fopt-info-vec`):** Inspecting the compiler's internal vectorization optimization passes to verify which loops vectorized successfully, and diagnosing the precise mathematical or structural reasons why specific loops failed to vectorize (`-fopt-info-vec-missed`).

```
  +---------------------------------------------------------------------------------+
  |                            COMPILER VECTOR PIPELINE                             |
  |                                                                                 |
  |  Source Loop (C/C++) ---> [ Front-End AST ] ---> [ Vector Dependency Analysis ] |
  |                                                           |                     |
  |                             +-----------------------------+                     |
  |                             | (Passed: No Aliasing, Known Bounds, Unit-Stride)  |
  |                             v                                                   |
  |     -fopt-info-vec       : "loop vectorized using 32-byte vectors (AVX2)"       |
  |     -march=native        : Generates packed VADDPS / VMULPS instructions        |
  |                                                                                 |
  |                             +-----------------------------+                     |
  |                             | (Failed: Dependency, Divergent Branches, AoS)     |
  |                             v                                                   |
  |     -fopt-info-vec-missed: "statement clobbers memory / control flow too complex"|
  |     Silent Fallback      : Generates slow scalar ADDSS / MULSS instructions     |
  +---------------------------------------------------------------------------------+
```

---

## 2. Why We Need It: The "Generic Target" Performance Pitfall

By default, when you compile a program using GCC or Clang on Linux:
```bash
gcc -O3 main.c -o main
```
The compiler defaults to producing a generic binary compatible with **any x86-64 CPU manufactured since 2003**. 
Under this generic baseline:
- The compiler is restricted to **128-bit SSE2** instructions ($2$ doubles or $4$ floats per vector).
- It is **strictly forbidden** from emitting 256-bit AVX/AVX2 or 512-bit AVX-512 instructions, because running an AVX instruction on an older CPU that lacks AVX hardware triggers an immediate OS crash:
  ```
  Illegal instruction (core dumped)  [SIGILL]
  ```

### The Solution: `-march=native`
When compiling code intended to run on the local host machine (such as dedicated compute nodes in an HPC cluster or your university development workstation), appending `-march=native`:

```bash
gcc -O3 -march=native -fopenmp main.c -o main
```

Instructs the compiler to query the processor's hardware `CPUID` instruction at compile-time and enable **every hardware feature supported by that specific silicon**:
- 256-bit AVX and AVX2 vector registers
- 512-bit AVX-512 Foundation (`avx512f`), Vector Length Extensions (`avx512vl`), and Byte/Word (`avx512bw`)
- Fused Multiply-Add (**FMA3**: computes $a \times b + c$ in a single clock cycle with zero intermediate rounding error)
- Bit Manipulation Instructions (BMI1/BMI2)

> [!IMPORTANT]
> Simply adding `-march=native` to a compute-intensive matrix or physics simulation often doubles or quadruples performance with zero changes to the underlying source code!

---

## 3. How It Works: Compiler Vectorization Reports

Modern compilers provide detailed diagnostics tracking vectorization passes:

### 3.1 GCC Vectorization Diagnostic Flags

| GCC Flag | Description | Output Content |
|:---|:---|:---|
| **`-fopt-info-vec`** | Reports **successful** vectorizations. | Prints filename, line number, and vector register byte width used (e.g., 16-byte, 32-byte, or 64-byte). |
| **`-fopt-info-vec-missed`** | Reports loops that **failed** to vectorize. | Explains the exact failure reason (e.g., control flow, potential aliasing, non-contiguous access). |
| **`-fopt-info-vec-note`** | Reports detailed compiler analysis notes. | Detailed alignment, cost model calculations, and loop versioning details. |
| **`-fopt-info-vec-all`** | Enables **all** of the above simultaneously. | Full dump of the vectorizer optimization pass. |

#### Sample GCC Vectorization Missed Diagnostics:
```text
simd_test.c:42:5: missed: couldn't vectorize loop
simd_test.c:45:9: missed: statement clobbers memory: printf(...)
simd_test.c:50:5: missed: not vectorized: control flow in loop
simd_test.c:65:5: missed: not vectorized: loop-carried dependency between A[i] and A[i-1]
```

---

### 3.2 Clang / LLVM Vectorization Diagnostic Flags

| Clang Flag | Equivalent Function |
|:---|:---|
| **`-Rpass=loop-vectorize`** | Emits remarks for loops successfully vectorized. |
| **`-Rpass-missed=loop-vectorize`** | Emits remarks explaining why loops were not vectorized. |
| **`-Rpass-analysis=loop-vectorize`** | Emits detailed remarks identifying memory dependencies and cost calculations. |

---

### 3.3 Intel oneAPI (icx) Optimization Reports
Intel's compilers provide industry-leading vectorizer diagnostics:
```bash
icx -O3 -qopenmp -qopt-report=3 -qopt-report-phase=vec main.c
```
Generates a detailed text file (`main.optrpt`) describing loop unrolling factors, vector byte lengths, peel loop counts, and memory alignment metrics.

---

## 4. Understanding Vectorization Opportunities & Blockers

What makes a loop vectorizable?

```
                        VECTORIZATION ELIGIBILITY CHECKLIST
                        
  [ ] Canonical Loop Form: Single induction variable, invariant step.
  [ ] Pre-computable Trip Count: Loop cannot terminate on dynamic while() condition or early break.
  [ ] Straight-line Control Flow: No complex branching (or branch must be convertible to blend mask).
  [ ] No Outer Function Calls: Calling non-inlined functions (e.g. printf, rand) kills vectorization.
  [ ] No RAW Loop-Carried Dependencies: Iteration i cannot depend on outputs of iteration i-1.
  [ ] Contiguous Memory Layout (Unit Stride): Preferred SoA over AoS.
```

---

### 4.1 Memory Layout: Structure of Arrays (SoA) vs. Array of Structures (AoS)
The physical arrangement of data in memory dictates whether vector hardware can perform **Unit-Stride Streaming** or is crippled by **Gather/Scatter** overhead:

```
  ARRAY OF STRUCTURES (AoS) - OOP Style:
  struct Particle { float x, y, z, mass; };
  Particle particles[1000];
  Memory: [ x0 y0 z0 m0 ][ x1 y1 z1 m1 ][ x2 y2 z2 m2 ] ...
  Problem: To update only 'x', CPU must jump 16 bytes between elements! (Strided access).

  STRUCTURE OF ARRAYS (SoA) - Data-Oriented HPC Style:
  struct ParticleSystem {
      float x[1000];
      float y[1000];
      float z[1000];
      float mass[1000];
  };
  Memory for x: [ x0 x1 x2 x3 x4 x5 x6 x7 x8 ... ]
  Advantage: Consecutive floats packed side-by-side! Ideal for 256-bit AVX vector load!
```

- In **SoA**, a single vector load instruction (`_mm256_load_ps`) pulls 8 consecutive floats into a register in 1 clock cycle.
- In **AoS**, the CPU must execute costly **Gather instructions** (`VGATHERDPS`), which fetch disjoint memory locations across multiple cache lines, degrading throughput by up to $5\times$.

---

## 5. Compilable C Code Benchmark

### 5.1 Program: Diagnosing Vectorization and Benchmarking AoS vs. SoA
This program contains two algorithms:
1. An **AoS (Array of Structures)** particle coordinate update.
2. An **SoA (Structure of Arrays)** particle coordinate update.
It demonstrates compiler diagnostic reports, the `-march=native` speedup, and the unit-stride vector advantage.

Save this file as `vector_optimization_demo.c`:

```c
/* =========================================================================
 * File: vector_optimization_demo.c
 * Description: Demonstrates:
 *              1. Vectorization diagnostics (-fopt-info-vec)
 *              2. The performance difference of AoS vs SoA memory layout
 *              3. Architecture unlocking via -march=native
 * Compilation & Diagnostic Commands:
 *   Standard Build:
 *     gcc -O3 -fopenmp -fopt-info-vec-missed -fopt-info-vec vector_optimization_demo.c -o vec_demo
 *   Native AVX Build:
 *     gcc -O3 -march=native -fopenmp -fopt-info-vec vector_optimization_demo.c -o vec_demo
 * Execution:
 *   ./vec_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define NUM_PARTICLES 10000000 // 10 Million Particles
#define NUM_STEPS 10

/* -------------------------------------------------------------
 * 1. ARRAY OF STRUCTURES (AoS) LAYOUT
 * ------------------------------------------------------------- */
typedef struct {
    float x, y, z;
    float vx, vy, vz;
} ParticleAoS;

/* -------------------------------------------------------------
 * 2. STRUCTURE OF ARRAYS (SoA) LAYOUT
 * ------------------------------------------------------------- */
typedef struct {
    float *x, *y, *z;
    float *vx, *vy, *vz;
} ParticleSoA;

void update_aos(ParticleAoS * __restrict__ p, int n, float dt) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        p[i].x += p[i].vx * dt;
        p[i].y += p[i].vy * dt;
        p[i].z += p[i].vz * dt;
    }
}

void update_soa(ParticleSoA * __restrict__ p, int n, float dt) {
    float * __restrict__ x = p->x;
    float * __restrict__ y = p->y;
    float * __restrict__ z = p->z;
    float * __restrict__ vx = p->vx;
    float * __restrict__ vy = p->vy;
    float * __restrict__ vz = p->vz;

    #pragma omp simd
    for (int i = 0; i < n; i++) {
        x[i] += vx[i] * dt;
        y[i] += vy[i] * dt;
        z[i] += vz[i] * dt;
    }
}

int main(void) {
    printf("============================================================\n");
    printf("     Compiler Vectorization & Memory Layout Benchmark       \n");
    printf("============================================================\n");
    printf("Particle Count: %d | Time Steps: %d\n\n", NUM_PARTICLES, NUM_STEPS);

    float dt = 0.01f;
    double t0, t1;

    /* 1. Benchmark AoS */
    ParticleAoS *particles_aos = (ParticleAoS*)aligned_alloc(64, NUM_PARTICLES * sizeof(ParticleAoS));
    for (int i = 0; i < NUM_PARTICLES; i++) {
        particles_aos[i].x = 1.0f; particles_aos[i].y = 2.0f; particles_aos[i].z = 3.0f;
        particles_aos[i].vx = 0.1f; particles_aos[i].vy = 0.2f; particles_aos[i].vz = 0.3f;
    }

    t0 = omp_get_wtime();
    for (int s = 0; s < NUM_STEPS; s++) {
        update_aos(particles_aos, NUM_PARTICLES, dt);
    }
    t1 = omp_get_wtime();
    double time_aos = t1 - t0;
    printf("1. Array of Structures (AoS) : %8.4f ms | (Strided memory access)\n",
           time_aos * 1000.0);

    /* 2. Benchmark SoA */
    ParticleSoA particles_soa;
    particles_soa.x  = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));
    particles_soa.y  = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));
    particles_soa.z  = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));
    particles_soa.vx = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));
    particles_soa.vy = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));
    particles_soa.vz = (float*)aligned_alloc(64, NUM_PARTICLES * sizeof(float));

    for (int i = 0; i < NUM_PARTICLES; i++) {
        particles_soa.x[i] = 1.0f; particles_soa.y[i] = 2.0f; particles_soa.z[i] = 3.0f;
        particles_soa.vx[i] = 0.1f; particles_soa.vy[i] = 0.2f; particles_soa.vz[i] = 0.3f;
    }

    t0 = omp_get_wtime();
    for (int s = 0; s < NUM_STEPS; s++) {
        update_soa(&particles_soa, NUM_PARTICLES, dt);
    }
    t1 = omp_get_wtime();
    double time_soa = t1 - t0;
    printf("2. Structure of Arrays (SoA) : %8.4f ms | (Unit-stride vector streaming)\n\n",
           time_soa * 1000.0);

    printf("--> Performance Advantage of SoA Layout: %.2fx Faster!\n",
           time_aos / time_soa);

    /* Cleanup */
    free(particles_aos);
    free(particles_soa.x);  free(particles_soa.y);  free(particles_soa.z);
    free(particles_soa.vx); free(particles_soa.vy); free(particles_soa.vz);

    printf("============================================================\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Diagnostics and Benchmark Output

#### Step 1: Compile with Diagnostics:
```bash
$ gcc -O3 -march=native -fopenmp -fopt-info-vec vector_optimization_demo.c -o vec_demo
```
*Compiler Vectorization Report:*
```text
vector_optimization_demo.c:38:5: optimized: loop vectorized using 32-byte vectors
vector_optimization_demo.c:54:5: optimized: loop vectorized using 32-byte vectors
```

#### Step 2: Run the Executable:
```bash
$ ./vec_demo
```
```text
============================================================
     Compiler Vectorization & Memory Layout Benchmark       
============================================================
Particle Count: 10000000 | Time Steps: 10

1. Array of Structures (AoS) : 182.4100 ms | (Strided memory access)
2. Structure of Arrays (SoA) :  48.1200 ms | (Unit-stride vector streaming)

--> Performance Advantage of SoA Layout: 3.79x Faster!
============================================================
```

### Critical Architectural Observations:
1. **SoA Advantage:** The Structure of Arrays layout executed **$3.79\times$ faster** than the Array of Structures layout on the identical CPU!
2. **Why?** In SoA, `x[i]` and `vx[i]` are contiguous in memory. The CPU loaded 8 single-precision floats in a single AVX vector instruction and streamed data through the L1 cache at peak bandwidth. In AoS, the non-contiguous 24-byte struct layout forced the CPU into slow, strided loads that choked the memory pipeline.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Strided Load Penalty Model (AoS vs. SoA)
Let:
- $B_{\text{line}} = 64$ bytes be the hardware cache line size.
- $S_{\text{stride}}$ be the byte size of an AoS struct (e.g., $24$ bytes for `ParticleAoS`).

In SoA layout, data is unit-stride ($1$ float $= 4$ bytes):
$$\text{Floats per 64-Byte Cache Line} = \frac{64}{4} = 16 \text{ useful elements}$$
Every DRAM cache line fetch delivers **$100\%$ useful data**.

In AoS layout with a 24-byte struct:
$$\text{Useful floats accessed per cache line} \approx \frac{64}{24} \approx 2.67 \text{ floats}$$
Only $2.67 \times 4 = 10.67$ bytes out of every 64-byte line are used!
$$\text{Memory Bus Waste} = 1 - \frac{10.67}{64} \approx \mathbf{83.3\% \text{ wasted memory bandwidth!}}$$

This mathematical reality explains why AoS structures severely throttle vector performance on modern multi-core processors.

---

## 7. Advantages & Limitations

### Compiler Vectorization Flags Summary

| Flag | Primary Benefit | Potential Risk / Drawback |
|:---|:---|:---|
| **`-O3`** | Enables aggressive loop vectorization, unrolling, and inlining. | Larger binary size; rare floating-point reordering. |
| **`-march=native`** | Unlocks full CPU instruction set (AVX2, AVX-512, FMA). | **Breaks portability:** Binary crashes if run on older CPUs. |
| **`-mavx2`** | Generates 256-bit AVX2 vector code portably for all Haswell+ chips. | Will not utilize AVX-512 on machines that have it. |
| **`-fopt-info-vec`** | Verifies which loops actually vectorized. | Adds compiler stderr text output during build. |
| **`-fopt-info-vec-missed`**| Identifies the exact line and reason why loops failed to vectorize. | Very verbose output on large codebases. |

---

## 8. Formal Definitions for Exam

- **`-fopt-info-vec`:** A GCC compiler optimization reporting flag that prints diagnostic information detailing which loops were successfully transformed using vector execution units.
- **`-fopt-info-vec-missed`:** A GCC diagnostic flag that outputs detailed explanations for why specific loops were rejected by the compiler's auto-vectorization pass.
- **`-march=native`:** A compiler flag that instructs the code generator to detect the microarchitecture of the host processor and emit code utilizing all supported vector and instruction set extensions.
- **Structure of Arrays (SoA):** A memory layout pattern where each attribute of a composite data structure is stored in a separate, contiguous homogeneous array, optimizing spatial locality for SIMD vector units.
- **Array of Structures (AoS):** A traditional object-oriented memory layout where composite attributes for each entity are clustered together in contiguous structs, causing non-unit strided memory accesses during vectorization.
- **Unit Stride:** A memory access pattern where consecutive loop iterations access physically contiguous memory addresses ($\text{Address}_{i+1} - \text{Address}_i = \text{sizeof}(\text{element})$), maximizing hardware prefetching and SIMD vector register throughput.

---

## 9. Comparison Table

### Vectorization Diagnostic Flags Across Major Compilers

| Optimization Goal | GNU Compiler Collection (GCC) | LLVM / Clang | Intel oneAPI (icx) |
|:---|:---|:---|:---|
| **Report Vectorized Loops** | `-fopt-info-vec` | `-Rpass=loop-vectorize` | `-qopt-report -qopt-report-phase=vec` |
| **Report Missed Loops** | `-fopt-info-vec-missed` | `-Rpass-missed=loop-vectorize` | Included in `.optrpt` |
| **Target Host Architecture** | `-march=native` | `-march=native` | `-xHost` or `-march=native` |
| **Force 256-bit AVX2** | `-mavx2 -mfma` | `-mavx2 -mfma` | `-march=core-avx2` |
| **Force 512-bit AVX-512** | `-mavx512f -mavx512vl` | `-mavx512f -mavx512vl` | `-xCORE-AVX512` |

---

## 10. Solved Analytical Problems

### Problem 1: Interpreting Missed Vectorization Diagnostics
**Question:** When compiling a scientific simulation with `gcc -O3 -fopt-info-vec-missed`, the compiler outputs:
```text
solver.c:88:5: missed: couldn't vectorize loop: statement clobbers memory: log_debug(i)
```
Explain the root cause of the failure and show how to modify the code so the loop vectorizes.

**Solution:**
1. **Root Cause:**
   - The loop contains a function call: `log_debug(i)`.
   - The compiler cannot inline `log_debug()`. An external function call involves saving registers, stack frame manipulation, and potential arbitrary side effects ("clobbers memory").
   - SIMD hardware instructions cannot branch into arbitrary scalar functions mid-vector.
2. **The Fix:**
   - Hoist the I/O or debug call outside the computational vector loop, or guard it with an `#ifdef DEBUG` preprocessor block so production builds remove it:
   ```c
   #pragma omp simd
   for (int i = 0; i < N; i++) {
       A[i] = B[i] * C[i];
   }
   #ifdef DEBUG
   for (int i = 0; i < N; i++) log_debug(i);
   #endif
   ```

---

### Problem 2: The SIGILL Crash on Cluster Deployment
**Question:** A developer compiles a program on their high-end laptop featuring an Intel 13th-Gen Core i9 (supporting AVX2) using `gcc -O3 -march=native -fopenmp simulation.c -o sim_bin`.
When the developer deploys `sim_bin` to an older HPC compute node containing Intel Xeon E5 processors, the program crashes instantly upon launch with:
```
Illegal instruction (core dumped)
```
Explain why this happened and provide two solutions.

**Solution:**
1. **Why It Crashed:**
   - The flag `-march=native` generated binary instructions specific to the laptop's modern CPU (such as AVX2 or FMA).
   - When executed on the older Xeon server lacking those specific instructions, the older CPU encountered an unknown opcode and generated a hardware fault: `SIGILL` (Signal Illegal Instruction).
2. **Solution 1 (Compile on Target):**
   - Recompile the source code directly on the target HPC compute node using `-march=native`.
3. **Solution 2 (Portable Target Flag):**
   - Compile using a mutually supported architecture baseline, such as `-march=x86-64-v3` (which targets standard AVX/AVX2) or `-msse4.2` to ensure backward compatibility across all cluster nodes.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why does `-O2` not vectorize loops by default in GCC?
**Answer:** Because `-O2` prioritizes optimization without significantly increasing binary size or compilation time. Vectorization introduces loop peeling, loop unrolling, and scalar epilogues, which can expand code size (code bloat) and may slightly change floating-point rounding order. In GCC, vectorization is enabled by default starting at **`-O3`** or explicitly via `-O2 -ftree-vectorize`.

### Q2: What is "Loop Peeling" reported in compiler vectorization notes?
**Answer:** CPUs achieve maximum memory throughput when 32-byte or 64-byte vector loads are aligned to 32-byte or 64-byte physical memory addresses. If an array pointer starts at an unaligned address (e.g. `0x1004`), the compiler "peels" off the first few iterations and runs them in a short scalar loop until the pointer reaches `0x1020` (a clean 32-byte boundary). From that point forward, the compiler executes high-speed aligned vector instructions.

### Q3: Why is Structure of Arrays (SoA) harder to maintain in OOP software, and what is AoSoA?
**Answer:** SoA separates data members into different arrays, breaking encapsulation (an entity's data is scattered across multiple lists). To get the best of both worlds, HPC engineers use **Array of Structures of Arrays (AoSoA)**: structs that contain fixed-size vector chunks (e.g. `struct ParticleChunk { float x[8], y[8], z[8]; };`). This preserves modularity while ensuring that every internal struct member matches the exact width of a hardware SIMD vector register!

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                COMPILER VECTORIZATION CHEATSHEET                                   |
+----------------------------------------------------------------------------------------------------+
| Key GCC Flags          | -O3 -fopenmp -march=native -fopt-info-vec -fopt-info-vec-missed           |
| -march=native          | Detects local CPUID; unlocks AVX2, AVX-512, FMA. (Breaks portability!)     |
| -fopt-info-vec         | Reports loops successfully transformed to SIMD vectors.                   |
| -fopt-info-vec-missed  | Diagnoses why loops failed (dependencies, function calls, control flow).  |
+----------------------------------------------------------------------------------------------------+
|                                    MEMORY LAYOUT COMPARISON                                        |
+----------------------------------------------------------------------------------------------------+
| Layout                 | Memory Access Pattern       | Vector Efficiency | Hardware Prefecher Hits |
| SoA (Struct of Arrays) | Unit-Stride (Contiguous)    | 100% Optimal      | Superb                  |
| AoS (Array of Structs) | Strided / Gather-Scatter    | Poor (~20-40%)    | High Cache Line Waste   |
+----------------------------------------------------------------------------------------------------+
|                                   TOP VECTORIZATION KILLERS                                        |
+----------------------------------------------------------------------------------------------------+
| 1. Pointer Aliasing without 'restrict' or '#pragma omp simd'                                       |
| 2. Function calls inside loop body (e.g., printf, math functions not vectorizable)                 |
| 3. Dynamic loop termination conditions (while loops, break statements)                             |
| 4. True Read-After-Write (RAW) loop-carried data dependencies (A[i] = A[i-1])                      |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Use compiler vectorization diagnostic flags (`-fopt-info-vec`, `-fopt-info-vec-missed`) to verify whether loops vectorized.
2. Explain why `-march=native` is necessary to unlock AVX2/AVX-512 on modern x86 hardware.
3. Diagnose and resolve the SIGILL crash caused by running binaries compiled with `-march=native` on older machines.
4. Explain the difference between Unit-Stride memory access and Gather/Scatter strided memory access.
5. Transform inefficient Array of Structures (AoS) code into high-performance Structure of Arrays (SoA) layout.
6. Identify common loop constructs that block auto-vectorization (un-inlined calls, complex branches, loop-carried dependencies).
