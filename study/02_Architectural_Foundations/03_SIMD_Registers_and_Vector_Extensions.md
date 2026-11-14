# Module 02.03 — SIMD Registers, Vector Instructions & Architectural Extensions (SSE, AVX, AVX-512)

> **Prerequisites:** Module 02.01 (Pipelining & Stored-Program Model), Module 02.02 (ILP & Superscalar Execution)  
> **Lecture Mapping:** Lecture 2 (Architectural Foundations: SIMD Registers, SSE, AVX, AVX-512)  
> **Exam Relevance:** MidSem (Register Width Calculations, Vector Lane Mapping, SIMD vs MIMD) & EndSem  

---

## 1. Understanding the Basic Idea

Suppose you are a painter hired to paint a row of four wooden fence posts white.
- **The Scalar Painter:** You pick up a standard 4-inch paintbrush. You dip it in paint, walk over to Post 0, and paint it. Then you dip again, walk to Post 1, and paint it. You repeat this for Post 2 and Post 3. To paint 4 posts, you made **4 distinct painting actions**.
- **The Vector (SIMD) Painter:** You construct a specialized, wide roller tool holding **four paint heads attached to a single handle**. You dip all four heads into the paint tray at once, apply the tool to the fence, and push down **once**. In a single physical stroke, all 4 posts are painted simultaneously!

```
Scalar Execution (SISD - 1 Operation per Instruction):
Instruction 1: Add a[0] + b[0] ──► c[0]
Instruction 2: Add a[1] + b[1] ──► c[1]
Instruction 3: Add a[2] + b[2] ──► c[2]
Instruction 4: Add a[3] + b[3] ──► c[3]
[ 4 Instructions Issued, 4 Clock Cycles ]

SIMD Vector Execution (1 Instruction, Multiple Data Elements):
Instruction:   VADDPS ymm0, ymm1, ymm2
               ┌──────────┬──────────┬──────────┬──────────┐
Vector Reg A:  │   a[3]   │   a[2]   │   a[1]   │   a[0]   │
               └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
                    │ +        │ +        │ +        │ +
               ┌────┴─────┬────┴─────┬────┴─────┬────┴─────┐
Vector Reg B:  │   b[3]   │   b[2]   │   b[1]   │   b[0]   │
               └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
                    ▼          ▼          ▼          ▼
               ┌──────────┬──────────┬──────────┬──────────┐
Vector Reg C:  │   c[3]   │   c[2]   │   c[1]   │   c[0]   │
               └──────────┴──────────┴──────────┴──────────┘
[ 1 Instruction Issued, 1 Clock Cycle, 4 Operations Completed! ]
```

This is **Single Instruction, Multiple Data (SIMD)**. 

Instead of operating on a single scalar number, the processor operates on **vectors** of numbers packed contiguously into extra-wide hardware registers.

---

## 2. Why Do We Need SIMD? The Energy & Area Efficiency of Vector Processing

In Module 02.02, we learned that building wider superscalar out-of-order execution cores hits the **ILP Wall** and burns massive amounts of electrical power.
Fetching, decoding, renaming, and tracking 4 independent scalar instructions requires thousands of complex transistors that generate heat.

### The Brilliant Efficiency of SIMD
Why fetch and decode 4 separate instructions when they all do the exact same operation (`ADD`) on adjacent data in memory?
- **Control Overhead Elimination:** A SIMD instruction fetches and decodes **exactly one instruction** from memory. The decode logic broadcasts that single opcode across 4, 8, or 16 parallel arithmetic lanes.
- **Energy Conservation:** By eliminating 75% to 90% of the instruction fetch/decode energy, SIMD directs almost all electrical power into actual floating-point math.
- **Throughput Multiplication:** A CPU core with 512-bit vector registers can compute **16 single-precision floating-point additions in a single clock cycle**, delivering up to a **$16\times$ speedup** over scalar code on the exact same core without changing the clock frequency!

---

## 3. The Evolution of x86 Vector Extensions: From MMX to AVX-512

Over the past 25 years, microprocessor architects have continually widened the CPU's internal vector registers to meet the soaring computational demands of graphics, multimedia, scientific computing, and artificial intelligence.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    THE EVOLUTION OF VECTOR REGISTER WIDTHS                      │
└─────────────────────────────────────────────────────────────────────────────────┘

 1997: MMX (64-bit)
 ┌────────────────────────┐
 │ 64 bits (MM0 - MM7)    │  [Integer Only: 8 bytes or 4 shorts or 2 ints]
 └────────────────────────┘
 
 1999: SSE / SSE2 / SSE3 / SSE4 (128-bit)
 ┌────────────────────────────────────────────────┐
 │ 128 bits (XMM0 - XMM15)                        │  [4 single-precision floats OR]
 └────────────────────────────────────────────────┘  [2 double-precision floats]
 
 2011: AVX / AVX2 (256-bit)
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 256 bits (YMM0 - YMM15)                                                        │  [8 single-precision floats OR]
 └────────────────────────────────────────────────────────────────────────────────┘  [4 double-precision floats]
 
 2015: AVX-512 (512-bit)
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 512 bits (ZMM0 - ZMM31)                                                                                                        │  [16 single-precision floats OR]
 └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘  [8 double-precision floats]
```

---

### Detailed Architectural Specifications

| Vector Extension | Introduced | Register Name | Register Count | Width (Bits) | Max 32-bit `float` Ops/Cycle | Max 64-bit `double` Ops/Cycle | Key Architectural Enhancements |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| **MMX** | 1997 | `mm0`–`mm7` | 8 | 64 bits | 0 (Integer only) | 0 (Integer only) | First x86 SIMD; aliased onto x87 FPU stack |
| **SSE** | 1999 | `xmm0`–`xmm7` | 8 | 128 bits | 4 floats | 0 | Dedicated vector registers; single-precision floats |
| **SSE2** | 2001 | `xmm0`–`xmm15`| 16 | 128 bits | 4 floats | 2 doubles | Added 64-bit IEEE double-precision support |
| **AVX** | 2011 | `ymm0`–`ymm15`| 16 | 256 bits | 8 floats | 4 doubles | Doubled width; introduced 3-operand non-destructive syntax (`VADDPS dest, src1, src2`) |
| **AVX2** | 2013 | `ymm0`–`ymm15`| 16 | 256 bits | 16 floats (with FMA) | 8 doubles (with FMA) | Full 256-bit integer vector support; FMA3 integration |
| **AVX-512** | 2015 | `zmm0`–`zmm31`| 32 | 512 bits | 32 floats (with FMA) | 16 doubles (with FMA) | Doubled registers to 32; added 8 opmask registers (`k0`–`k7`), per-element masking |

---

## 4. Vector Register Packing: Data Types and Lane Mapping

To understand vector arithmetic, visualize how memory variables map into the bit lanes of a register:

### 128-bit Register (`xmm`)
- **Single-Precision (32-bit floats):** Holds $\frac{128}{32} = \mathbf{4\text{ floats}}$ (`float[0]`, `float[1]`, `float[2]`, `float[3]`).
- **Double-Precision (64-bit doubles):** Holds $\frac{128}{64} = \mathbf{2\text{ doubles}}$ (`double[0]`, `double[1]`).

### 256-bit Register (`ymm`)
- **Single-Precision:** Holds $\frac{256}{32} = \mathbf{8\text{ floats}}$.
- **Double-Precision:** Holds $\frac{256}{64} = \mathbf{4\text{ doubles}}$.

### 512-bit Register (`zmm`)
- **Single-Precision:** Holds $\frac{512}{32} = \mathbf{16\text{ floats}}$.
- **Double-Precision:** Holds $\frac{512}{64} = \mathbf{8\text{ doubles}}$.

```
Bit: 511           448 447           384 383           320 319           256 255           192 191           128 127            64 63              0
    ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────────┐
ZMM:│    double[7]    │    double[6]    │    double[5]    │    double[4]    │    double[3]    │    double[2]    │    double[1]    │    double[0]    │
    └─────────────────┴─────────────────┴─────────────────┴─────────────────┴─────────────────┴─────────────────┴─────────────────┴─────────────────┘
                                                                            │                                 │
                                                                            ▼                                 ▼
                                                                Upper 256 bits = YMM               Lower 128 bits = XMM
```

Notice that x86 vector registers are **aliased**:
- The lower 128 bits of `ymm0` is `xmm0`.
- The lower 256 bits of `zmm0` is `ymm0`.

---

## 5. AVX-512 Advanced Architectural Features

AVX-512 is not simply wider AVX2; it introduced revolutionary architectural mechanisms designed specifically for supercomputing workloads:

### 1. Opmask Registers (`k0` through `k7`)
In scalar code, conditional operations use branches (`if-else`). But how do you execute an `if` statement when a vector register holds 16 different numbers, and only 9 of them meet the condition?
You cannot branch for half a register!

AVX-512 introduced **Predication (Masking)**:
```assembly
VCMPPS k1, zmm1, zmm2, 1  ; Compare zmm1 < zmm2 element-by-element.
                          ; Result is 16 bits written to mask register k1!
VADDPS zmm3 {k1}, zmm4, zmm5 ; Add only where k1 bit is 1!
                              ; Other vector lanes remain unchanged!
```

```
Element Index:   15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
Mask Reg (k1):    1  0  1  1  0  0  1  0  1  1  0  0  1  1  0  1
Vector Ops:       ▲  │  ▲  ▲  │  │  ▲  │  ▲  ▲  │  │  ▲  ▲  │  ▲
Executed:        YES NO YES YES NO NO YES NO YES YES NO NO YES YES NO YES
```

### 2. Doubling the Register File (32 Registers)
AVX-512 doubled the number of accessible registers from 16 (`ymm0`–`ymm15`) to **32 (`zmm0`–`zmm31`)**.
This dramatically reduces **register spilling** (swapping registers to the stack), keeping critical loops entirely inside register silicon.

### 3. Embedded Broadcast
An instruction can read a single scalar from memory and broadcast it across all 16 vector lanes on the fly without requiring an explicit separate broadcast instruction:
```assembly
VMULPD zmm1, zmm2, [rax]{1to8} ; Multiplies 8 doubles in zmm2 by 1 scalar double at [rax]
```

---

## 6. Practical Implementation: Scalar vs. AVX2 Vector Intrinsics in C

There are three ways to use SIMD in C:
1. **Auto-vectorization:** Relying on compiler flags (`-O3 -march=native`).
2. **OpenMP SIMD:** Using `#pragma omp simd` (covered in Module 07).
3. **SIMD Intrinsics:** Explicitly writing low-level vector assembly functions directly in C using `<immintrin.h>`.

The program below demonstrates an explicit **AVX2 vector addition** of two floating-point arrays using C intrinsics.

### AVX2 Vector Addition Benchmark (`avx2_vector_add.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <immintrin.h> // Essential header for x86 SIMD Intrinsics (SSE/AVX/AVX2/AVX-512)

#define N 32000000 // 32 Million floats (~128 MB per array)

// -----------------------------------------------------------------
// 1. Pure Scalar Addition (1 addition at a time)
// -----------------------------------------------------------------
void scalar_add(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// -----------------------------------------------------------------
// 2. Explicit AVX2 Vector Addition (8 additions per instruction)
// -----------------------------------------------------------------
void avx2_vector_add(float *a, float *b, float *c, int n) {
    // Process 8 floats per loop step
    for (int i = 0; i < n; i += 8) {
        // Load 256 bits (8 floats) from memory into YMM vector registers
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);

        // Perform 8 parallel single-precision additions in ONE cycle
        __m256 vc = _mm256_add_ps(va, vb);

        // Store the 256-bit result back to memory
        _mm256_storeu_ps(&c[i], vc);
    }
}

int main(void) {
    printf("============================================================\n");
    printf("       DEMONSTRATING AVX2 256-BIT VECTOR INTRINSICS         \n");
    printf("============================================================\n");

    // Allocate memory aligned to 32-byte boundaries for AVX2
    float *a = (float *)aligned_alloc(32, N * sizeof(float));
    float *b = (float *)aligned_alloc(32, N * sizeof(float));
    float *c = (float *)aligned_alloc(32, N * sizeof(float));

    if (!a || !b || !c) {
        fprintf(stderr, "Error: Memory allocation failed!\n");
        return 1;
    }

    for (int i = 0; i < N; i++) {
        a[i] = 1.5f;
        b[i] = 2.5f;
    }

    struct timespec start, end;

    // Test Scalar
    clock_gettime(CLOCK_MONOTONIC, &start);
    scalar_add(a, b, c, N);
    clock_gettime(CLOCK_MONOTONIC, &end);
    double scalar_time = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("Scalar Loop Execution Time : %.4f seconds (Result: %.2f)\n", scalar_time, c[0]);

    // Test AVX2 Vector Intrinsics
    clock_gettime(CLOCK_MONOTONIC, &start);
    avx2_vector_add(a, b, c, N);
    clock_gettime(CLOCK_MONOTONIC, &end);
    double vector_time = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("AVX2 Vector Execution Time : %.4f seconds (Result: %.2f)\n", vector_time, c[0]);
    printf("\nVector Speedup Achieved    : %.2fx faster!\n", scalar_time / vector_time);
    printf("============================================================\n");

    free(a); free(b); free(c);
    return 0;
}
```

#### Compilation & Execution
Note the mandatory architecture flag `-mavx2` instructing GCC to generate AVX2 assembly:
```bash
gcc -O2 -mavx2 avx2_vector_add.c -o avx2_vector_add
./avx2_vector_add
```

*Expected Output (on an Intel x86-64 CPU with AVX2 support):*
```
============================================================
       DEMONSTRATING AVX2 256-BIT VECTOR INTRINSICS         
============================================================
Scalar Loop Execution Time : 0.0482 seconds (Result: 4.00)
AVX2 Vector Execution Time : 0.0135 seconds (Result: 4.00)

Vector Speedup Achieved    : 3.57x faster!
============================================================
```

#### Why Was the Speedup $3.57\times$ Rather Than $8.0\times$?
While arithmetic was computed $8\times$ faster, streaming 32 million floats from main memory saturated the CPU's memory bandwidth bus (the Memory Wall). Vector math runs so fast that memory delivery becomes the primary bottleneck!

---

## 7. Important Comparisons

### SIMD Vectorization vs. MIMD Multithreading

| Dimension | SIMD Vectorization (Module 02.03) | MIMD Multithreading (OpenMP / Pthreads) |
|:---|:---|:---|
| **Flynn Classification** | Single Instruction, Multiple Data | Multiple Instruction, Multiple Data |
| **Hardware Scope** | Inside a **single CPU core** (vector registers)| Across **multiple CPU cores** or sockets |
| **Instruction Streams**| Exactly **ONE instruction stream** | Multiple **autonomous instruction streams** |
| **Synchronization** | Implicit, zero-cycle lockstep execution in hardware | Explicit software barriers, mutexes, locks required |
| **Data Locality** | Requires contiguous, packed data arrays | Can execute completely different functions on distinct data |
| **Programming Model** | Vector pragmas (`#pragma omp simd`), intrinsics| Thread pragmas (`#pragma omp parallel`), Pthreads API |

---

## 8. Exam-Ready Definitions

### Single Instruction, Multiple Data (SIMD)
> **Single Instruction, Multiple Data (SIMD)** is an architectural classification wherein a single machine instruction initiates the concurrent execution of the same arithmetic or logical operation across multiple distinct data elements stored in wide vector registers.

### Vector Lane
> A **Vector Lane** is an independent, parallel sub-channel within a SIMD hardware execution unit responsible for performing the arithmetic operation on one specific element of a packed vector register.

### AVX (Advanced Vector Extensions)
> **Advanced Vector Extensions (AVX)** is an x86 instruction set architecture extension introduced by Intel and AMD that expanded vector register widths from 128 bits (`XMM`) to 256 bits (`YMM`) and 512 bits (`ZMM`), supporting high-throughput floating-point and integer vector computation.

### FMA3 (Fused Multiply-Add 3-Operand)
> **FMA3** is a vector instruction format that computes $d = (a \times b) + c$ in a single hardware cycle with three register operands, executing two floating-point operations with a single rounding step.

---

## 9. Solved Problems

### Problem 1: Register Capacity and Vector Speedup
A computational physicist is porting a hydrodynamics simulation loop from scalar code to vector code.
The loop iterates $N = 10,000,000$ times.
Each iteration computes four 64-bit IEEE double-precision floating-point arithmetic operations on array elements.

1. How many double-precision numbers can be stored simultaneously in:
   - An SSE register (`XMM`)?
   - An AVX2 register (`YMM`)?
   - An AVX-512 register (`ZMM`)?
2. Calculate the theoretical minimum number of vector instructions required to complete the $10,000,000$ iterations using AVX-512.
3. If scalar execution requires $1\text{ cycle}$ per operation, calculate the theoretical speedup of the AVX-512 implementation assuming an ideal memory subsystem.

#### Solution:
#### 1. Register Capacity for 64-bit Doubles
- **SSE (`XMM` = 128 bits):**
  \[
  \text{Capacity} = \frac{128\text{ bits}}{64\text{ bits/double}} = \mathbf{2\text{ doubles}}
  \]
- **AVX2 (`YMM` = 256 bits):**
  \[
  \text{Capacity} = \frac{256\text{ bits}}{64\text{ bits/double}} = \mathbf{4\text{ doubles}}
  \]
- **AVX-512 (`ZMM` = 512 bits):**
  \[
  \text{Capacity} = \frac{512\text{ bits}}{64\text{ bits/double}} = \mathbf{8\text{ doubles}}
  \]

#### 2. Minimum Vector Instructions for AVX-512
- In scalar execution, $10,000,000$ iterations require $10,000,000$ loop operations.
- With AVX-512, each vector instruction processes **$8$ loop iterations simultaneously**:
  \[
  \text{Vector Loop Iterations} = \frac{10,000,000}{8} = \mathbf{1,250,000\text{ vector loop steps}}
  \]
- Since each iteration has 4 operations, the total vector arithmetic instructions required is:
  \[
  \text{Total Vector Instructions} = 1,250,000 \times 4 = \mathbf{5,000,000\text{ instructions}}
  \]
  *(Scalar code would have required $40,000,000$ instructions—an $8\times$ reduction!)*

#### 3. Theoretical Speedup
\[
S = \frac{\text{Scalar Instructions}}{\text{Vector Instructions}} = \frac{40,000,000}{5,000,000} = \mathbf{8.0\times}
\]

---

### Problem 2: Peak FLOPs Comparison across Vector Generations
An Intel core operates at a stable base frequency of $3.0\text{ GHz}$. 
The core features two identical execution ports capable of executing vector instructions.
Calculate the single-core peak floating-point throughput ($R_{\text{peak}}$) for 32-bit single-precision floats under:
1. **Scalar Code (No SIMD):** 1 float operation per cycle per port.
2. **SSE (128-bit):** 4 float operations per cycle per port.
3. **AVX2 with FMA (256-bit):** 8 floats $\times 2$ (FMA) per cycle per port.
4. **AVX-512 with FMA (512-bit):** 16 floats $\times 2$ (FMA) per cycle per port.

#### Solution:
Peak throughput formula: $R_{\text{peak}} = f \times (\text{Ports}) \times (\text{FLOPs/Cycle/Port})$.
- $f = 3.0 \times 10^9\text{ Hz}$.

1. **Scalar:**
   \[
   R = 3.0 \times 10^9 \times 2 \times 1 = \mathbf{6.0\text{ GFLOP/s}}
   \]
2. **SSE:**
   \[
   R = 3.0 \times 10^9 \times 2 \times 4 = \mathbf{24.0\text{ GFLOP/s}} \quad (4\times\text{ scalar})
   \]
3. **AVX2 with FMA:**
   \[
   R = 3.0 \times 10^9 \times 2 \times 16 = \mathbf{96.0\text{ GFLOP/s}} \quad (16\times\text{ scalar})
   \]
4. **AVX-512 with FMA:**
   \[
   R = 3.0 \times 10^9 \times 2 \times 32 = \mathbf{192.0\text{ GFLOP/s}} \quad (32\times\text{ scalar!})
   \]

*Conclusion:* The exact same core running at the exact same $3.0\text{ GHz}$ clock frequency is **$32\times$ faster** when utilizing AVX-512 vector units compared to basic scalar execution!

---

## 10. Conceptual Examination Questions & Answers

### Q1: Why is memory alignment critical for SIMD vector instructions?
**Model Answer:**
SIMD instructions load and store data in wide blocks (16 bytes for SSE, 32 bytes for AVX, 64 bytes for AVX-512). 

Hardware memory controllers and cache lines are structured in 64-byte chunks.
- If data is **aligned** (i.e., its starting memory address is an exact multiple of the vector width, such as address $0\text{x}1000$ mod 64 = 0), a 512-bit vector load can be serviced by fetching **exactly one single cache line** in a single memory cycle.
- If data is **unaligned** (e.g., straddling across two adjacent cache lines), the CPU must perform **two distinct memory loads**, merge the bytes using internal shifter circuits, and handle potential split-lock or page-boundary faults.

This causes a severe performance penalty. Consequently, high-performance code uses aligned memory allocation functions (`aligned_alloc(64, size)`) and aligned vector load instructions (`_mm512_load_pd` instead of `_mm512_loadu_pd`).

---

### Q2: What is "AVX Downclocking" (or Frequency Throttling), and why does it occur on AVX-512 processors?
**Model Answer:**
Executing 512-bit wide vector instructions simultaneously activates thousands of ALUs, drawing massive instantaneous electrical currents and generating high localized power density (heat) on the silicon die.

To prevent the CPU from exceeding its Thermal Design Power (TDP) or suffering voltage droop (which could crash the chip), processor power control hardware automatically **downclocks (reduces)** the operating clock frequency by $10\%\text{ to }20\%$ while executing heavy AVX-512 instructions. 

In poorly optimized code containing only sporadic AVX-512 instructions mixed with scalar code, the CPU downclocks the entire core, causing the surrounding scalar code to execute slower! Thus, vectorization must be dense and sustained to outweigh the frequency downclocking penalty.

---

## 11. Quick Revision Sheet

- **SIMD Core Idea:**
  \[
  \boxed{\text{One Opcode Fetch} \longrightarrow \text{Broadcast to } K \text{ Vector Lanes} \longrightarrow K \text{ Calculations Completed in 1 Cycle}}
  \]
- **Register Widths & Capacities:**
  - **SSE (128-bit):** 4 single floats (32-bit) OR 2 double floats (64-bit).
  - **AVX / AVX2 (256-bit):** 8 single floats OR 4 double floats.
  - **AVX-512 (512-bit):** 16 single floats OR 8 double floats.
- **FMA Peak Factor:**
  \[
  \boxed{\text{1 FMA Instruction} = 2\text{ FLOPs (Multiply + Add)}}
  \]
- **AVX-512 Key Superpowers:**
  1. 32 Architectural registers (`zmm0`–`zmm31`).
  2. Opmask registers (`k0`–`k7`) for branchless vector predication.
  3. Embedded scalar broadcasting (`{1to8}`).
- **Common Exam Trap:**
  - *Trap:* Confusing SIMD with multi-core parallelism.
  - *Reality:* SIMD operates **inside a single core** via wide registers; multi-core executes independent instruction streams across **separate physical cores**.

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 02.04, verify that you can independently:
1. Explain the SIMD execution model and draw a vector lane execution diagram.
2. State the bit-widths and register names for MMX, SSE, AVX, AVX2, and AVX-512.
3. Calculate how many `float` or `double` operands fit into any vector register.
4. Calculate single-core theoretical peak GFLOP/s with and without FMA.
5. Explain the purpose of Opmask registers in AVX-512 and describe why memory alignment matters for vector throughput.
