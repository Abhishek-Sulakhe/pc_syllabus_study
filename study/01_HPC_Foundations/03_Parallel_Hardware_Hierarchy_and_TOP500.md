# Module 01.03 — The Parallel Hardware Hierarchy & TOP500 Supercomputers

> **Prerequisites:** Module 01.01 (Introduction to Parallel Computing), Module 01.02 (The Seven Dwarfs)  
> **Lecture Mapping:** Lecture 1 (Parallel Hardware Hierarchy & Supercomputing Landscapes)  
> **Exam Relevance:** MidSem (Architecture Hierarchy, Benchmark Metrics, Ranking Methodologies) & EndSem  

---

## 1. Understanding the Basic Idea

When people hear the word **supercomputer**, they often imagine a single, enormous, science-fiction machine with a massive processor inside.

The physical reality is far more interesting:
> **A modern supercomputer is not one giant processor; it is an aggregation of tens of thousands of standard, high-performance processors arranged in an intricate, multi-layered hierarchy, communicating over an ultra-low-latency network.**

Parallelism does not occur at only one level. It exists simultaneously across multiple physical scales:
- Inside a **single CPU core**, multiple math operations execute in parallel across vector registers.
- Inside a **single CPU chip**, multiple cores execute concurrently.
- Inside a **single server motherboard (node)**, multiple CPU sockets and GPU accelerators share memory.
- Inside a **datacenter hall**, thousands of nodes coordinate over network cables to solve a single grand scientific problem.

Understanding this **hierarchy of parallelism** is essential. If your software parallelizes only across nodes but ignores multi-core threading, or parallelizes across cores but ignores vector SIMD registers, you are leaving 90% or more of the machine's computational capacity completely unused.

---

## 2. The Complete Parallel Hardware Hierarchy

To write high-performance code, an engineer must visualize the full architectural continuum from the microscopic silicon register up to the multi-megawatt warehouse cluster.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        THE PARALLEL HARDWARE HIERARCHY                          │
└─────────────────────────────────────────────────────────────────────────────────┘

 Level 5: WAREHOUSE / CLUSTER LEVEL (Massively Parallel Distributed Memory)
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ Thousands of Compute Nodes connected via high-speed network fabrics         │
 │ (InfiniBand HDR/NDR, Slingshot - 100 to 400 Gbps, Latency: 0.5–1.5 μs)      │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
 Level 4: NODE LEVEL (Shared-Memory SMP / ccNUMA System)
 ┌──────────────────────────────────────┴──────────────────────────────────────┐
 │ Single Motherboard / Blade: 2–4 CPU Sockets + 4–8 GPU Accelerators          │
 │ Shared DRAM (512 GB – 4 TB), Cache-Coherent Interconnect (UPI / Infinity)   │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
 Level 3: CHIP / SOCKET LEVEL (Multi-Core Silicon Die)
 ┌──────────────────────────────────────┴──────────────────────────────────────┐
 │ 32 to 128 Physical CPU Cores per Socket, Shared L3 Cache (e.g., 256 MB)    │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
 Level 2: CORE LEVEL (Simultaneous Multithreading & Pipelining)
 ┌──────────────────────────────────────┴──────────────────────────────────────┐
 │ Private L1 (32 KB) & L2 (1 MB) Caches, Out-of-Order Pipelined Execution     │
 │ 2 Logical Threads per Physical Core (SMT / Hyper-Threading)                 │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
 Level 1: INSTRUCTION & VECTOR LEVEL (SIMD - Single Instruction, Multiple Data)
 ┌──────────────────────────────────────┴──────────────────────────────────────┐
 │ 512-bit Vector Registers (AVX-512): 16 single-precision or 8 double-       │
 │ precision floating-point operations executed in a SINGLE clock cycle!       │
 └─────────────────────────────────────────────────────────────────────────────┘
```

### Detailed Layer Breakdown

#### Level 1: Vector / SIMD Parallelism (Micro-Architecture)
- **Scale:** Sub-nanosecond (cycle-level).
- **Mechanism:** Wide hardware registers (e.g., Intel AVX-512, ARM SVE) where one single assembly instruction applies the same arithmetic operation across multiple data elements packed into vector lanes simultaneously.
- **Programming Abstraction:** Compiler auto-vectorization, vector pragmas (`#pragma omp simd`), or vector intrinsics (`_mm512_add_pd`).

#### Level 2: Instruction-Level Parallelism (ILP) & Multithreading (SMT)
- **Scale:** Core level.
- **Mechanism:** Superscalar execution units that issue multiple independent scalar instructions per cycle, combined with hardware multithreading (Hyper-Threading) to keep execution pipelines full when one thread is stalled on memory.

#### Level 3: Multi-Core Sockets
- **Scale:** Single silicon chip.
- **Mechanism:** Dozens of complete, independent CPU cores fabricated onto one piece of silicon (or multiple chiplets), sharing a massive Last-Level Cache (L3).
- **Programming Abstraction:** OpenMP threads, POSIX threads (Pthreads).

#### Level 4: Compute Nodes (SMP / ccNUMA)
- **Scale:** Single server blade.
- **Mechanism:** Multiple CPU sockets connected to dedicated local memory banks. Accessing another socket's memory is possible via hardware cache-coherence protocols, but incurs higher latency (**Cache-Coherent Non-Uniform Memory Access - ccNUMA**).

#### Level 5: Multi-Node Clusters & Supercomputers
- **Scale:** Full machine room.
- **Mechanism:** Autonomous nodes with **completely private memory address spaces**, connected by dedicated low-latency, high-bandwidth interconnects (InfiniBand, Cray Slingshot, Omni-Path).
- **Programming Abstraction:** Message Passing Interface (MPI).

---

## 3. How Supercomputers are Ranked: The TOP500 List

Published twice a year (in June at ISC in Germany and in November at SC in the United States) since 1993, the **TOP500 list** (`top500.org`) ranks the 500 most powerful supercomputers in the world.

```
       ERA OF COMPUTATIONAL SPEED
  Throughput
    (FLOP/s)
       ▲
   10¹⁸ ┼───────────────────────────────────────────────────────── [EXASCALE: Frontier (2022)]
        │                                                     /
   10¹⁵ ┼───────────────────────────────── [PETASCALE: Roadrunner (2008)]
        │                             /
   10¹² ┼────────── [TERASCALE: ASCI Red (1997)]
        │       /
   10⁹  ┼── [GIGASCALE: Cray-2 (1985)]
        └─────────────────────────────────────────────────────────► Time (Years)
         1985         1995        2005        2015        2025
```

### The Primary Benchmark: High-Performance LINPACK (HPL)
To be eligible for the TOP500, a supercomputing site must run the **HPL (High-Performance LINPACK)** benchmark.

#### What does HPL actually do?
1. Generates a massive dense system of linear algebraic equations:
   \[
   A \cdot x = b
   \]
   Where $A$ is a dense $N \times N$ matrix of 64-bit floating-point numbers ($N$ is chosen by the site to be as large as physical memory permits—frequently millions of rows and columns!).
2. Solves the system using **LU decomposition with partial pivoting** in double precision (64-bit IEEE 754 arithmetic).
3. Measures the total wall-clock time $T$ and computes the sustained floating-point throughput:
   \[
   \text{Sustained FLOPS } (R_{\max}) = \frac{\frac{2}{3} N^3 + 2 N^2}{T}
   \]

### Key Metrics Reported on TOP500

| Metric Name | Symbol | Definition & Significance |
|:---|:---:|:---|
| **Theoretical Peak** | $R_{\text{peak}}$ | The absolute maximum floating-point operations the hardware could perform if every core and vector unit executed at 100% capacity at base/turbo clock frequency without any cache misses or communication delays. |
| **Sustained LINPACK**| $R_{\max}$ | The actual, measured floating-point throughput achieved while running the real HPL benchmark code. |
| **LINPACK Efficiency**| $\frac{R_{\max}}{R_{\text{peak}}}$ | The percentage of the theoretical machine capacity delivered in practice (typically $65\%\text{ to }85\%$ on well-balanced systems). |
| **Problem Size** | $N$ | The order of the linear system matrix used in the benchmark run. |
| **Power Consumption**| $P$ | Total electrical power consumed by the system during the benchmark run, measured in Kilowatts ($\text{kW}$) or Megawatts ($\text{MW}$). |

---

## 4. Why LINPACK is Not Enough: The Rise of HPCG

While HPL is the historical standard, modern supercomputing scientists recognized a critical limitation:
> **LINPACK (Dense Linear Algebra) is too optimistic.** It exhibits near-ideal data reuse, regular unit-stride memory accesses, and hides memory latency completely.

In 2014, Jack Dongarra (co-founder of LINPACK) and Michael Heroux created the **High Performance Conjugate Gradients (HPCG)** benchmark.

### What is HPCG?
- HPCG solves an elliptic partial differential equation discretized on a 3D regular grid using an iterative Conjugate Gradient solver preconditioned with a multi-grid method.
- Its mathematical core relies on **Sparse Matrix-Vector multiplication (SpMV)** and vector dot products.
- Memory access is irregular and indirect. Arithmetic intensity is extremely low ($< 0.25\text{ FLOPs/byte}$).

### The Eye-Opening Comparison: HPL vs. HPCG on Frontier

Let us examine the world's first verified Exascale supercomputer, **Frontier** (located at Oak Ridge National Laboratory, USA):

```
                   FRONTIER SUPERCOMPUTER BENCHMARK COMPARISON
   Throughput
   (PFLOP/s)
      ▲
 1200 ┼────────────────────────────── R_peak: 1,714.8 PFLOP/s (Theoretical Peak)
      │
 1100 ┼────────────────── R_max (HPL LINPACK): 1,194.0 PFLOP/s (69.6% of Peak)
      │
      │
      │   [Massive Reality Gap]
      │
      │
   20 ┼── HPCG: 14.05 PFLOP/s (Only 0.82% of Peak!)
    0 ┴─────────────────────────────────────────────────────────────────────────►
```

#### Why Such a Dramatic Drop?
- On **HPL**, Frontier delivers **$1.194\text{ ExaFLOPS}$** ($1,194\text{ PetaFLOPS}$), utilizing almost $70\%$ of its theoretical peak.
- On **HPCG**, the exact same machine delivers only **$14.05\text{ PetaFLOPS}$**—barely **$0.82\%$** of its theoretical peak!
- **The Lesson:** For algorithms that do not have high arithmetic reuse, $99\%$ of the supercomputer's compute silicon sits idle waiting for memory transfers from DRAM!

---

## 5. Green500: The Energy Efficiency Metric

As supercomputers grew into multi-megawatt facilities, raw performance alone became unsustainable. In 2007, the **Green500 list** was introduced to rank the TOP500 supercomputers according to **Energy Efficiency**:

\[
\boxed{\text{Energy Efficiency} = \frac{R_{\max}}{\text{Electrical Power Consumed}} = \frac{\text{GFLOP/s}}{\text{Watt}} \quad \text{or} \quad \frac{\text{MFLOP/s}}{\text{Watt}}}
\]

### Modern Energy Milestones
- In 2010, the most energy-efficient supercomputers achieved $\approx 1\text{ GFLOPS/Watt}$.
- By 2024, modern GPU-accelerated clusters achieve over **$65\text{ to }70\text{ GFLOPS/Watt}$**!
- If an Exascale supercomputer were built using 2010 technology ($1\text{ GFLOPS/W}$), it would consume:
  \[
  \text{Power} = \frac{10^{18}\text{ FLOPS}}{10^9\text{ FLOPS/W}} = 10^9\text{ Watts} = \mathbf{1,000\text{ Megawatts} = 1\text{ Gigawatt!}}
  \]
  That equals the electrical output of an entire commercial nuclear power station. Thanks to energy-efficient multi-core/GPU architectures, modern Exascale systems operate around $20\text{ to }30\text{ MW}$.

---

## 6. Practical Implementation: Querying the Local Hardware Topology

To understand the hardware hierarchy on an actual machine, you can write a C program that queries core counts, thread contexts, and OpenMP runtime properties.

### System Hardware Topology Inspection in C (`inspect_hardware.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <unistd.h>

int main(void) {
    printf("============================================================\n");
    printf("       PARALLEL HARDWARE TOPOLOGY QUERY UTILITY             \n");
    printf("============================================================\n");

    // 1. Query OS-level CPU availability
    long num_online_processors = sysconf(_SC_NPROCESSORS_ONLN);
    long num_configured_processors = sysconf(_SC_NPROCESSORS_CONF);

    printf("OS Detected Online Logical Cores     : %ld\n", num_online_processors);
    printf("OS Configured Total Cores           : %ld\n", num_configured_processors);

    // 2. Query OpenMP Runtime Environment
    int omp_max_threads = omp_get_max_threads();
    int omp_num_procs   = omp_get_num_procs();

    printf("\nOpenMP Environment:\n");
    printf("  omp_get_num_procs() (Available Cores) : %d\n", omp_num_procs);
    printf("  omp_get_max_threads() (Default Team)  : %d\n", omp_max_threads);

    // 3. Test active parallel region execution
    printf("\nSpawning Thread Team Across Hardware Hierarchy:\n");
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        int total_threads = omp_get_num_threads();

        // Print thread execution report (executed concurrently)
        #pragma omp critical
        {
            printf("  [Hardware Thread %2d of %2d] active and executing on core.\n", 
                   tid, total_threads);
        }
    }

    printf("============================================================\n");
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp inspect_hardware.c -o inspect_hardware
./inspect_hardware
```

*Expected Output (on an 8-core / 16-thread workstation):*
```
============================================================
       PARALLEL HARDWARE TOPOLOGY QUERY UTILITY             
============================================================
OS Detected Online Logical Cores     : 16
OS Configured Total Cores           : 16

OpenMP Environment:
  omp_get_num_procs() (Available Cores) : 16
  omp_get_max_threads() (Default Team)  : 16

Spawning Thread Team Across Hardware Hierarchy:
  [Hardware Thread  0 of 16] active and executing on core.
  [Hardware Thread  3 of 16] active and executing on core.
  [Hardware Thread  1 of 16] active and executing on core.
  ...
============================================================
```

---

## 7. Important Comparisons

### HPL (LINPACK) vs. HPCG Benchmark

| Feature | HPL (High-Performance LINPACK) | HPCG (High Performance Conjugate Gradients) |
|:---|:---|:---|
| **Mathematical Problem** | Dense Linear System ($A \cdot x = b$) | Sparse Linear System from 3D PDE ($A_{\text{sparse}} x = b$) |
| **Algorithmic Dwarf** | Dwarf 1: Dense Linear Algebra | Dwarf 2: Sparse Linear Algebra & Dwarf 5 (Stencils) |
| **Memory Access Pattern** | Contiguous, regular, unit-stride | Irregular, indirect indexing, non-contiguous |
| **Arithmetic Intensity** | High ($\mathcal{O}(N)$ arithmetic reuse) | Very Low ($< 0.25\text{ FLOPs/byte}$) |
| **Hardware Stressed** | Floating-point ALU/SIMD execution units | DRAM Memory Bandwidth & Network Latency |
| **Efficiency Achieved** | High ($65\%\text{ to }85\%$ of $R_{\text{peak}}$) | Low ($0.5\%\text{ to }3\%$ of $R_{\text{peak}}$) |
| **Primary Goal** | Showcases maximum theoretical compute power | Represents real scientific application performance |

---

## 8. Exam-Ready Definitions

### Theoretical Peak Performance ($R_{\text{peak}}$)
> **Theoretical Peak Performance ($R_{\text{peak}}$)** is the absolute upper mathematical limit of floating-point operations a computing system can perform per second, calculated as the product of total processor cores, operating clock frequency, and the maximum number of floating-point operations executable per clock cycle per core.

### Sustained Performance ($R_{\max}$)
> **Sustained Performance ($R_{\max}$)** is the actual, measured rate of floating-point execution (in FLOP/s) achieved by a parallel system when running a real benchmark application (such as HPL LINPACK) over its entire execution duration.

### Green500
> The **Green500** is a biannual supercomputer ranking that orders the systems of the TOP500 list according to their energy efficiency, measured as sustained floating-point throughput per unit of electrical power consumed ($\text{GFLOP/s per Watt}$).

### Cache-Coherent Non-Uniform Memory Access (ccNUMA)
> **Cache-Coherent Non-Uniform Memory Access (ccNUMA)** is a multi-processor shared-memory architecture in which all processors share a single logical address space, but physical memory access times vary depending on whether the memory bank is physically local to the accessing processor socket or remote, with hardware mechanisms maintaining cache consistency across sockets.

---

## 9. Solved Problems

### Problem 1: Calculating $R_{\text{peak}}$ for an HPC Cluster
A university purchases an HPC cluster consisting of $64$ dual-socket compute nodes. 
- Each CPU socket contains an Intel Xeon processor with $36$ physical cores.
- The base clock frequency of each core is $2.4\text{ GHz}$.
- Each core is equipped with two 512-bit AVX-512 FMA (Fused Multiply-Add) vector execution units. A single 512-bit FMA instruction computes $8$ double-precision (64-bit) multiply-adds per cycle, which counts as $16$ floating-point operations per FMA unit per cycle (i.e., $2 \times 16 = 32\text{ FLOPs/cycle/core}$).

Calculate:
1. The theoretical peak performance ($R_{\text{peak}}$) of a single node in TFLOP/s.
2. The total theoretical peak performance ($R_{\text{peak}}$) of the entire 64-node cluster in PFLOP/s.
3. If the cluster achieves an $R_{\max}$ of $265\text{ TFLOP/s}$ on the HPL benchmark, calculate its LINPACK parallel efficiency.

#### Solution:
#### 1. Single Node Peak Performance
- Total cores per node:
  \[
  \text{Cores/Node} = 2\text{ sockets} \times 36\text{ cores/socket} = 72\text{ cores}
  \]
- Floating-point capability per core per second:
  \[
  \text{FLOPs/sec/core} = 2.4 \times 10^9\text{ cycles/sec} \times 32\text{ FLOPs/cycle} = 7.68 \times 10^{10}\text{ FLOP/s} = 76.8\text{ GFLOP/s}
  \]
- Single Node $R_{\text{peak}}$:
  \[
  R_{\text{peak}}(\text{node}) = 72 \times 76.8\text{ GFLOP/s} = 5,529.6\text{ GFLOP/s} = \mathbf{5.5296\text{ TFLOP/s}}
  \]

#### 2. Entire Cluster Peak Performance
- Total cluster peak throughput:
  \[
  R_{\text{peak}}(\text{cluster}) = 64\text{ nodes} \times 5.5296\text{ TFLOP/s} = \mathbf{353.8944\text{ TFLOP/s}} \approx \mathbf{0.3539\text{ PFLOP/s}}
  \]

#### 3. LINPACK Parallel Efficiency
- Efficiency formula:
  \[
  \text{Efficiency} = \frac{R_{\max}}{R_{\text{peak}}} = \frac{265.0\text{ TFLOP/s}}{353.8944\text{ TFLOP/s}} \approx 0.7488 = \mathbf{74.88\%}
  \]

---

### Problem 2: Energy Efficiency and Green500 Ranking
Supercomputer A delivers $R_{\max} = 500\text{ PFLOP/s}$ and consumes $12.5\text{ Megawatts}$.  
Supercomputer B delivers $R_{\max} = 800\text{ PFLOP/s}$ and consumes $25.0\text{ Megawatts}$.

1. Which supercomputer will rank higher on the **TOP500 list**?
2. Which supercomputer will rank higher on the **Green500 list**?

#### Solution:
1. **TOP500 Ranking (governed strictly by $R_{\max}$):**
   - Supercomputer A: $R_{\max} = 500\text{ PFLOP/s}$
   - Supercomputer B: $R_{\max} = 800\text{ PFLOP/s}$
   - **Supercomputer B ranks higher on TOP500** because its raw sustained throughput is greater ($800 > 500$).

2. **Green500 Ranking (governed strictly by GFLOP/s per Watt):**
   - Energy Efficiency of Supercomputer A:
     \[
     \text{Eff}_A = \frac{500 \times 10^{15}\text{ FLOP/s}}{12.5 \times 10^6\text{ W}} = 40 \times 10^9\text{ FLOP/s/W} = \mathbf{40.0\text{ GFLOP/s per Watt}}
     \]
   - Energy Efficiency of Supercomputer B:
     \[
     \text{Eff}_B = \frac{800 \times 10^{15}\text{ FLOP/s}}{25.0 \times 10^6\text{ W}} = 32 \times 10^9\text{ FLOP/s/W} = \mathbf{32.0\text{ GFLOP/s per Watt}}
     \]
   - **Supercomputer A ranks higher on Green500** because it delivers more compute per watt consumed ($40.0 > 32.0\text{ GFLOP/s/W}$).

---

## 10. Conceptual Examination Questions & Answers

### Q1: Explain why a modern parallel programmer must write hybrid code (e.g., MPI + OpenMP) rather than relying on a single parallel programming model.
**Model Answer:**
A modern supercomputer embodies a two-tier hardware hierarchy:
1. **Across Compute Nodes:** Nodes have disjoint, private physical memory spaces connected via a network fabric. Communication requires explicit network message transfer.
2. **Within a Single Compute Node:** Multiple CPU sockets and cores share a common physical DRAM address space with hardware cache coherence.

If a programmer uses **pure OpenMP**, the program cannot run across multiple nodes because OpenMP requires a unified shared address space. 

Conversely, if a programmer uses **pure MPI**, they must launch an independent process on every single core of the node. This creates severe performance drawbacks:
- It duplicates read-only data structures in every core's memory, causing memory exhaustion.
- It forces intra-node communication through the software network stack rather than direct shared-memory reads/writes.

Therefore, the **Hybrid Model (MPI + OpenMP)** is optimal: MPI handles coarse-grained communication *between* nodes, while OpenMP handles fine-grained thread parallelism *within* each node's shared memory.

---

### Q2: Why is the gap between $R_{\text{peak}}$ and $R_{\max}$ larger on GPU-accelerated clusters than on CPU-only clusters?
**Model Answer:**
GPU accelerators possess massive theoretical floating-point execution parallelism (tens of thousands of arithmetic ALUs) operating at high clock rates, resulting in very large theoretical peak numbers ($R_{\text{peak}}$). 

However, achieving that peak requires keeping every ALU continuously fed with data. The PCIe or NVLink bus connecting the CPU host to the GPU accelerator has limited bandwidth compared to on-chip GPU SRAM. Furthermore, communication overheads between distributed nodes, non-uniform memory access, and non-vectorizable sequential portions of the code prevent the hardware from running at full capacity continuously. 

Because the GPU's compute throughput is so immense relative to its memory and interconnect feeding bandwidth, real-world benchmarks suffer a higher percentage of idle stall cycles, widening the gap between $R_{\text{peak}}$ and $R_{\max}$.

---

## 11. Quick Revision Sheet

- **The 5-Level Parallel Hierarchy:**
  \[
  \boxed{\text{Vector (SIMD)} \longrightarrow \text{Core (SMT)} \longrightarrow \text{Socket (Multi-Core)} \longrightarrow \text{Node (ccNUMA)} \longrightarrow \text{Cluster (Distributed MPI)}}
  \]
- **Theoretical Peak Performance Equation:**
  \[
  \boxed{R_{\text{peak}} = N_{\text{cores}} \times f_{\text{clock}} \times \left(\frac{\text{FLOPs}}{\text{cycle/core}}\right)}
  \]
- **LINPACK Efficiency Equation:**
  \[
  \boxed{\text{Efficiency} = \frac{R_{\max}}{R_{\text{peak}}} \times 100\%}
  \]
- **Green500 Metric:**
  \[
  \boxed{\text{Green Metric} = \frac{R_{\max}}{\text{Power (Watts)}} \quad [\text{GFLOP/s per Watt}]}
  \]
- **Benchmark Contrast:**
  - **HPL (LINPACK):** Dense LU factorization, high arithmetic intensity, measures compute ceiling.
  - **HPCG:** Sparse Conjugate Gradient, low arithmetic intensity, measures memory/interconnect reality.
- **Common Exam Trap:**
  - *Trap:* Confusing $R_{\text{peak}}$ with $R_{\max}$.
  - *Correction:* $R_{\text{peak}}$ is a theoretical calculation on paper based on hardware specs; $R_{\max}$ is the actual experimentally measured number from a real benchmark run.

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 01.04, verify that you can independently:
1. Draw and explain the 5-layer parallel hardware hierarchy from vector registers to multi-node clusters.
2. Calculate the theoretical peak performance ($R_{\text{peak}}$) of any multi-core CPU or cluster given its core count, clock speed, and vector instruction capacity.
3. Define $R_{\max}$ and calculate parallel efficiency on the HPL benchmark.
4. Contrast HPL and HPCG and explain why machines achieve vastly lower scores on HPCG.
5. Define the Green500 metric and solve numerical trade-offs between compute power and energy efficiency.
