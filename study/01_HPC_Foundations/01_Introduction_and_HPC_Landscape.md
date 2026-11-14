# Module 01.01 — Introduction to Parallel Computing & the HPC Landscape

> **Prerequisites:** Basic knowledge of computer systems and standard C programming.  
> **Lecture Mapping:** Lecture 1 (Introduction, Course Overview, Evaluation, HPC Landscape)  
> **Exam Relevance:** MidSem (Foundational Concepts, Definitions) & Lab Orientation  

---

## 1. Understanding the Basic Idea

Imagine you are tasked with calculating the sum of 100 million numbers written on a chalkboard. 

If you work alone, you must read each number one by one, add it to your running total, and write down the result. Even if you are extremely fast—say, adding one number every second—the task will take you over three years of non-stop work.

Now imagine you invite 99 of your friends into the room. You divide the chalkboard into 100 equal sections of 1 million numbers each. Each person calculates their own local total simultaneously. After everyone finishes, you simply add the 100 partial sums together. What previously took three years can now theoretically be completed in less than two weeks.

```
Sequential Execution (One Worker):
Number 1 ──► Number 2 ──► Number 3 ──► ... ──► Number 100,000,000
[ Total Time: T ]

Parallel Execution (4 Workers Example):
Worker 0: Numbers [         1 to  25,000,000 ] ──► Partial Sum 0 ┐
Worker 1: Numbers [25,000,001 to  50,000,000 ] ──► Partial Sum 1 ┼──► Combine ──► Final Sum
Worker 2: Numbers [50,000,001 to  75,000,000 ] ──► Partial Sum 2 │    (Master)
Worker 3: Numbers [75,000,001 to 100,000,000 ] ──► Partial Sum 3 ┘
[ Total Time: ~ T / 4 + coordination overhead ]
```

This is the essence of **Parallel Computing**:
> **Parallel Computing** is a computational paradigm where a large problem is decomposed into discrete, smaller sub-problems that are executed simultaneously across multiple processing elements (cores, CPUs, or compute nodes) to obtain results faster or to solve larger problems.

**High-Performance Computing (HPC)** is the practice of aggregating computing power in a way that delivers vastly higher performance than one could get out of a standard desktop computer or workstation in order to solve complex problems in science, engineering, or business.

---

## 2. Why Do We Need It? The Death of the Free Lunch

For nearly three decades (from the 1970s until the mid-2000s), software developers enjoyed what computer scientist Herb Sutter famously called **"The Free Lunch."** 

If your sequential program was slow, you did not need to redesign your algorithms or rewrite your code. You simply waited 18 months, bought a new processor with a higher clock speed, and your existing program automatically ran almost twice as fast without changing a single line of source code.

```
Era of Clock Scaling (Up to ~2004):
Year      Processor       Clock Frequency      Transistor Count
1989      Intel 486       33 MHz               1.2 Million
1993      Pentium         66 MHz               3.1 Million
1999      Pentium III     500 MHz              9.5 Million
2004      Pentium 4       3.8 GHz              125 Million
------------------------------------------------------------------
2005+     The "Free Lunch" Ends! Clock speeds hit a hard wall (~3-4 GHz).
```

### Why Did Clock Scaling Stop? The Power Wall and Dennard Scaling Breakdown

To understand why parallel computing is now mandatory for every computer engineer, we must examine the physics of silicon semiconductors:

1. **CMOS Dynamic Power Dissipation:**
   The electrical power $P$ consumed by a complementary metal-oxide-semiconductor (CMOS) circuit is governed by:
   \[
   P = C \cdot V^2 \cdot f + P_{\text{static}}
   \]
   Where:
   - $C$ is the capacitive load being switched.
   - $V$ is the operating supply voltage.
   - $f$ is the clock switching frequency.
   - $P_{\text{static}}$ is leakage power (current leaking through transistors even when idle).

2. **Dennard Scaling (1974):**
   Robert Dennard observed that as transistors shrank in physical dimensions, their power density remained constant because operating voltage ($V$) could be reduced in proportion to size:
   \[
   \text{Transistor Dimension} \propto \frac{1}{\kappa} \implies \text{Area} \propto \frac{1}{\kappa^2}, \quad V \propto \frac{1}{\kappa}, \quad f \propto \kappa
   \]
   This allowed chip designers to continuously increase clock frequency ($f$) without melting the chip!

3. **The Collapse of Dennard Scaling (~2004–2005):**
   As transistor gate oxides shrank to mere atomic layers (below 65 nm), quantum mechanical tunneling and sub-threshold leakage currents exploded. Operating voltage ($V$) could no longer be reduced without causing severe logic errors and circuit instability.
   
   Because $V$ became stuck around $\approx 1.0\text{ V}$, any further increase in clock frequency $f$ resulted in an unsustainable cubic increase in power density:
   \[
   P \propto f^3 \quad (\text{when } V \text{ is scaled with } f)
   \]

```
                  THE HISTORICAL "POWER WALL"
   Power Density
    (Watts/cm²)
        ▲
   1000 ┼                                      [Rocket Nozzle]
        │                                  /
    100 ┼                          /  [Nuclear Reactor Core]
        │                  /
     10 ┼          /  [Hot Plate]
        │     /
      1 ┼ /
        └────────────────────────────────────────► Time (Years)
         1980       1990        2000     2005
                                           │
                                     CRITICAL WALL:
                            Air cooling can no longer remove
                            more than ~100–150 W/cm² economically.
```

If chip makers had continued increasing clock frequencies at the historical rate, the surface of a consumer CPU would have reached the thermal power density of a nuclear reactor core or a rocket nozzle! 

### The Industry Shift: Multi-Core and Many-Core Architectures
Because individual cores could no longer be made faster by cranking the clock frequency, CPU architects made a historic pivot:
- **Instead of making one core run twice as fast, put two (and later 4, 8, 16, 64, 128+) cores on the same silicon die running at conservative clock frequencies (2.0–3.5 GHz).**
- This preserves the power budget while increasing theoretical throughput.
- **The catch:** A sequential program runs on exactly **one** core. The remaining cores sit completely idle.
- Therefore, **parallelism is no longer a luxury for supercomputing centers; it is the only way to exploit the performance of modern commodity hardware.**

---

## 3. How Does High-Performance Computing Work?

High-Performance Computing systems achieve massive computational power by coupling thousands of multi-core processors over high-speed networks.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          ANATOMY OF AN HPC CLUSTER                              │
└─────────────────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────┐  High-Speed Low-Latency Network  ┌─────────────────────────┐
 │       COMPUTE NODE 0    │  (e.g., InfiniBand HDR/NDR,      │       COMPUTE NODE 1    │
 │                         │   Slingshot, 100-400 Gbps)       │                         │
 │ ┌─────────────────────┐ │ ◄══════════════════════════════► │ ┌─────────────────────┐ │
 │ │ Socket 0 (CPU)      │ │                                  │ │ Socket 0 (CPU)      │ │
 │ │  Core 0   Core 1    │ │                                  │ │  Core 0   Core 1    │ │
 │ │  Core 2   Core 3    │ │                                  │ │  Core 2   Core 3    │ │
 │ └──────────┬──────────┘ │                                  │ └──────────┬──────────┘ │
 │            │ Shared Bus │                                  │            │ Shared Bus │
 │ ┌──────────┴──────────┐ │                                  │ ┌──────────┴──────────┐ │
 │ │ Local DRAM (RAM)    │ │                                  │ │ Local DRAM (RAM)    │ │
 │ │ [Private to Node 0] │ │                                  │ │ [Private to Node 1] │ │
 │ └─────────────────────┘ │                                  │ └─────────────────────┘ │
 └─────────────────────────┘                                  └─────────────────────────┘
              ▲                                                            ▲
              │                                                            │
              └────────────────────────┬───────────────────────────────────┘
                                       ▼
                       Shared Parallel Storage Subsystem
                          (Lustre, GPFS, Ceph - Petabytes)
```

### The Three Paradigms of HPC Software
To program this hardware hierarchy, three distinct paradigms are combined:
1. **Instruction & Vector Level (SIMD):** Vector instructions (AVX-512) operating on data packed into 512-bit registers within a single core cycle.
2. **Shared-Memory Multi-Threading (OpenMP & Pthreads):** Multiple threads executing concurrently inside a single compute node, sharing a common physical DRAM address space.
3. **Distributed-Memory Multi-Processing (MPI):** Multiple autonomous processes running across distinct physical nodes, communicating explicitly by sending and receiving network messages.

---

## 4. Course Structure & Evaluation Framework

To master this vast landscape, this course is organized into two balanced halves:

### Evaluation Pattern
| Component | Weight | Scope | Assessment Type |
|:---|:---:|:---|:---|
| **End Semester Examination** | **40%** | Comprehensive (Lectures 1–32) | Theory, Architecture, Derivations, Protocols |
| **Laboratory Examination** | **40%** | Practical (OpenMP, Pthreads, MPI, Hybrid) | Compiling, Debugging, Benchmarking C/C++ Code |
| **Mid Semester Examination** | **20%** | Part A (Lectures 1–16) | Foundations, Scaling Models, OpenMP Fundamentals |

### The Two Pillars of the Course

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               COURSE TIMELINE                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

 PART A: TILL MIDSEM (Lectures 1–16)
 ├── Module 01: HPC Landscape, Seven Dwarfs, TOP500 & Power Wall
 ├── Module 02: Computer Architecture, ILP, SMT, SIMD, Flynn's Taxonomy & ccNUMA
 ├── Module 03: Performance Scaling Laws (Amdahl, Gustafson), Bottlenecks & Benchmarking
 ├── Module 04: OpenMP Basics, Fork-Join Architecture & Data Scoping (shared/private)
 ├── Module 05: OpenMP Work-Sharing (`for`, `sections`, `single`) & Loop Scheduling
 ├── Module 06: OpenMP Synchronization (`critical`, `atomic`, barriers, locks, reductions)
 └── Module 07: OpenMP Tasks & SIMD Vectorization (`#pragma omp simd`)
                            │
               ★★★ MID-SEMESTER EXAMINATION ★★★
                            │
 PART B: AFTER MIDSEM (Lectures 17–32)
 ├── Module 08: POSIX Standard & Pthreads Creation, Attributes, Joining & Detaching
 ├── Module 09: Pthreads Synchronization (Mutexes, CondVars, Semaphores, RW-Locks, Spinlocks)
 ├── Module 10: MPI Foundations, Distributed Memory & SPMD Lifecycle (`mpicc`/`mpirun`)
 ├── Module 11: MPI Point-to-Point (Blocking vs Non-Blocking), Dynamic Ports & Troubleshooting
 ├── Module 12: MPI Collective Operations (Broadcast, Scatter, Gather, Reduce, Barrier)
 └── Module 13: Hybrid Supercomputing: MPI + OpenMP Architecture & Parallel Sum
                            │
            ★★★ LAB EXAM (40%) & END-SEM EXAM (40%) ★★★
```

---

## 5. Practical Implementation: The Sequential vs. Parallel Contrast

To understand what parallel programming looks like in practice, consider a C program that computes the sum of an array of $100,000,000$ double-precision floating-point numbers.

### Sequential Baseline Implementation (`sequential_sum.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N 100000000

int main(void) {
    // 1. Allocate memory for 100 million double-precision numbers (~800 MB)
    double *data = (double *)malloc(N * sizeof(double));
    if (data == NULL) {
        fprintf(stderr, "Error: Memory allocation failed!\n");
        return 1;
    }

    // 2. Initialize the array
    for (int i = 0; i < N; i++) {
        data[i] = 1.0; // Sum should equal exactly 100,000,000.0
    }

    // 3. Time the sequential accumulation
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    double total_sum = 0.0;
    for (int i = 0; i < N; i++) {
        total_sum += data[i];
    }

    clock_gettime(CLOCK_MONOTONIC, &end);

    // 4. Calculate elapsed wall-clock time in seconds
    double elapsed_time = (end.tv_sec - start.tv_sec) + 
                          (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("Sequential Result : %f\n", total_sum);
    printf("Execution Time    : %.4f seconds\n", elapsed_time);

    free(data);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 sequential_sum.c -o sequential_sum
./sequential_sum
```
*Expected Output on a modern Intel Core i7:*
```
Sequential Result : 100000000.000000
Execution Time    : 0.1180 seconds
```

---

### Parallel OpenMP Implementation (`openmp_sum.c`)

Here is how the identical workload is parallelized across 4 physical cores using OpenMP:

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define N 100000000

int main(void) {
    double *data = (double *)malloc(N * sizeof(double));
    if (data == NULL) {
        fprintf(stderr, "Error: Memory allocation failed!\n");
        return 1;
    }

    for (int i = 0; i < N; i++) {
        data[i] = 1.0;
    }

    // High-resolution OpenMP timer
    double start_time = omp_get_wtime();

    double total_sum = 0.0;

    // Direct the compiler to distribute the loop iterations across 4 threads
    #pragma omp parallel for num_threads(4) reduction(+:total_sum)
    for (int i = 0; i < N; i++) {
        total_sum += data[i];
    }

    double end_time = omp_get_wtime();

    printf("Parallel Result   : %f\n", total_sum);
    printf("Execution Time    : %.4f seconds\n", end_time - start_time);

    free(data);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp openmp_sum.c -o openmp_sum
./openmp_sum
```
*Expected Output:*
```
Parallel Result   : 100000000.000000
Execution Time    : 0.0342 seconds
```

#### Key Observations
1. **Notice the `#pragma omp` directive:** We did not write complex socket code or manual assembly. The compiler divided the 100 million iterations among 4 threads ($25,000,000$ iterations per thread).
2. **Speedup:** The time dropped from $\approx 0.1180\text{ s}$ to $\approx 0.0342\text{ s}$—a speedup of $\approx 3.45\times$ on 4 cores!
3. **Why wasn't it exactly $4.0\times$?** Because of thread creation overhead, memory bandwidth saturation on the shared memory bus, and cache effects (which we explore in Modules 02 and 03).

---

## 6. Important Formulas & Physical Principles

### 1. Fundamental Performance
Performance is defined as the amount of valid computational work completed per unit of wall-clock time:
\[
\boxed{P = \frac{W}{T}}
\]
Where:
- $P$ is performance (e.g., operations per second).
- $W$ is total work accomplished (e.g., number of floating-point arithmetic operations).
- $T$ is execution time in seconds.

### 2. CMOS Dynamic Power Dissipation
The power consumed when switching digital logic states:
\[
\boxed{P_{\text{dynamic}} = \alpha \cdot C \cdot V^2 \cdot f}
\]
Where:
- $\alpha$ is the activity factor ($0 < \alpha \le 1$, the fraction of transistors switching per cycle).
- $C$ is total circuit capacitive load in Farads ($\text{F}$).
- $V$ is supply voltage in Volts ($\text{V}$).
- $f$ is clock frequency in Hertz ($\text{Hz}$).

### 3. Theoretical Peak Floating-Point Throughput
The maximum theoretical floating-point capacity of a computing system:
\[
\boxed{\text{Peak FLOP/s} = N_{\text{sockets}} \times \left(\frac{N_{\text{cores}}}{\text{socket}}\right) \times f_{\text{clock}} \times \left(\frac{\text{FLOPs}}{\text{cycle/core}}\right)}
\]
*Example:* A dual-socket server with two 32-core AMD EPYC processors running at a base clock of $2.5\text{ GHz}$, where each core can execute two 256-bit AVX2 FMA (Fused Multiply-Add) operations per cycle (16 double-precision FLOPs per cycle):
\[
\text{Peak FLOP/s} = 2 \times 32 \times (2.5 \times 10^9) \times 16 = 2,560,000,000,000\text{ FLOP/s} = 2.56\text{ TFLOP/s}
\]

---

## 7. Real-World Applications of HPC

HPC is the engine of modern scientific discovery and industrial engineering:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           GRAND CHALLENGE HPC DOMAINS                           │
└─────────────────────────────────────────────────────────────────────────────────┘
         │
         ├── 1. Climate & Weather Prediction
         │      Simulating Navier-Stokes fluid dynamics and atmospheric chemistry
         │      across a global 3D grid with millions of grid cells.
         │
         ├── 2. Molecular Dynamics & Drug Discovery
         │      Simulating electrostatic and Lennard-Jones forces between millions
         │      of atoms to design vaccines and therapeutics (e.g., NAMD, GROMACS).
         │
         ├── 3. Astrophysics & Cosmology
         │      N-Body gravitational simulations modeling galactic collision and
         │      dark matter evolution across billions of light years.
         │
         ├── 4. Computational Fluid Dynamics (CFD) & Aerodynamics
         │      Simulating airflow over commercial aircraft and Formula 1 race cars
         │      to reduce drag without expensive physical wind-tunnel prototypes.
         │
         └── 5. Artificial Intelligence & Deep Learning
                Distributed training of multi-hundred-billion parameter transformer
                models across tens of thousands of GPUs over InfiniBand networks.
```

---

## 8. Exam-Ready Definitions

### High-Performance Computing (HPC)
> **High-Performance Computing (HPC)** is the practice of aggregating computational components (such as processors, memory, and network interconnects) into massively parallel clusters to perform complex mathematical calculations at speeds orders of magnitude higher than standard desktop computers.

### Parallel Computing
> **Parallel Computing** is a computational paradigm in which multiple processing units concurrently execute distinct sub-tasks of a larger problem, communicating and synchronizing as necessary to produce the final solution in reduced wall-clock time.

### The Power Wall
> **The Power Wall** refers to the physical limit on microprocessor clock frequency scaling reached around 2004–2005, caused by the inability of traditional air-cooling techniques to dissipate the heat generated by sub-threshold leakage currents and high switching frequencies at sub-65 nm transistor dimensions.

### Dennard Scaling
> **Dennard Scaling** is the semiconductor scaling principle stating that as transistors become smaller, their power density remains constant because operating voltage and current scale downwards proportionally with linear transistor dimensions, allowing clock frequencies to increase without increasing power consumption per unit area.

---

## 9. Important Comparisons

### Commodity Personal Computer vs. HPC Supercomputing Cluster

| Characteristic | Commodity Desktop PC | HPC Supercomputer / Cluster |
|:---|:---|:---|
| **Primary Design Goal** | Low cost, responsiveness for single-user interactive tasks | Maximum aggregate compute throughput and data throughput |
| **Processor Count** | 1 CPU socket (4 to 16 cores) | Thousands of multi-core CPUs + GPU accelerators ($10^4$ to $10^7$ cores) |
| **Memory Architecture** | Single shared DRAM (16–64 GB), UMA bus | Distributed memory: terabytes to petabytes across private-node DRAMs |
| **Interconnect** | PCIe bus, standard 1 Gbps / 2.5 Gbps Ethernet | Specialized ultra-low latency fabrics (InfiniBand, Slingshot, 100–400 Gbps, $<1\ \mu\text{s}$ latency) |
| **Power Consumption** | 65 W to 500 W | 1 Megawatt to 30 Megawatts (equivalent to a small city) |
| **Programming Model** | Sequential code, POSIX threads, standard OS libraries | Hybrid MPI (across nodes) + OpenMP / CUDA (within nodes) |
| **Storage System** | Local SATA / NVMe SSD (single filesystem) | High-speed Parallel Distributed Filesystems (Lustre, GPFS) scaling to 100s of GB/s |

---

## 10. Solved Problems

### Problem 1: The Consequences of Frequency Scaling Under Fixed Voltage
A processor manufacturer considers releasing a single-core CPU clocked at $6.0\text{ GHz}$. The current processor operates stably at $3.0\text{ GHz}$ with a supply voltage of $1.1\text{ V}$ and dissipates $95\text{ W}$ of dynamic power. Because of transistor leakage constraints, operating at $6.0\text{ GHz}$ requires increasing the voltage to $1.35\text{ V}$. 

Assuming the capacitive load $C$ remains constant, calculate the dynamic power dissipated by the new processor.

#### Solution:
1. **Recall the dynamic power formula:**
   \[
   P = C \cdot V^2 \cdot f
   \]
2. **Formulate the ratio between the new state ($P_2$) and the baseline state ($P_1$):**
   \[
   \frac{P_2}{P_1} = \frac{C \cdot V_2^2 \cdot f_2}{C \cdot V_1^2 \cdot f_1} = \left(\frac{V_2}{V_1}\right)^2 \times \left(\frac{f_2}{f_1}\right)
   \]
3. **Substitute the given values:**
   - $V_1 = 1.1\text{ V}, \quad V_2 = 1.35\text{ V}$
   - $f_1 = 3.0\text{ GHz}, \quad f_2 = 6.0\text{ GHz}$
   - $P_1 = 95\text{ W}$

   \[
   \frac{P_2}{95} = \left(\frac{1.35}{1.1}\right)^2 \times \left(\frac{6.0}{3.0}\right)
   \]
   \[
   \left(\frac{1.35}{1.1}\right)^2 = (1.2273)^2 \approx 1.5062
   \]
   \[
   \frac{f_2}{f_1} = 2.0
   \]
   \[
   P_2 = 95 \times 1.5062 \times 2.0 = 95 \times 3.0124 \approx \mathbf{286.18\text{ W}}
   \]
4. **Conclusion:**
   Doubling the clock frequency caused the power dissipation to triple! A standard consumer cooling solution rated for $95\text{ W}$ would experience catastrophic thermal throttling or hardware failure. This illustrates the physical reality of the Power Wall.

---

### Problem 2: Supercomputer Operating Electricity Cost
The *Frontier* supercomputer at Oak Ridge National Laboratory consumes approximately $22.7\text{ Megawatts}$ of electrical power continuously under full LINPACK load. 

Assuming an industrial electricity tariff of $\$0.08$ per kilowatt-hour ($\text{kWh}$), calculate:
1. The total energy consumed in kilowatt-hours over a single 30-day month.
2. The monthly electricity bill required simply to power the system (excluding cooling overhead).

#### Solution:
1. **Convert power from Megawatts to Kilowatts:**
   \[
   P = 22.7\text{ MW} = 22.7 \times 10^3\text{ kW} = 22,700\text{ kW}
   \]
2. **Calculate total hours in 30 days:**
   \[
   \text{Hours} = 30\text{ days} \times 24\frac{\text{hours}}{\text{day}} = 720\text{ hours}
   \]
3. **Calculate total energy in $\text{kWh}$:**
   \[
   \text{Energy} = 22,700\text{ kW} \times 720\text{ hours} = \mathbf{16,344,000\text{ kWh}}
   \]
4. **Calculate total monthly electricity cost:**
   \[
   \text{Cost} = 16,344,000\text{ kWh} \times \$0.08/\text{kWh} = \mathbf{\$1,307,520}
   \]
   \[
   \boxed{\text{Monthly Cost} \approx \$1.31\text{ Million USD}}
   \]
5. **Architectural Insight:** At over $\$15\text{ Million}$ per year in electricity alone, energy efficiency (measured in Gigaflops per Watt) is as crucial an engineering metric as raw computational speed.

---

## 11. Conceptual Examination Questions & Answers

### Q1: Why did CPU manufacturers switch to multi-core architectures instead of continuing to increase single-core clock speeds?
**Model Answer:**
CPU manufacturers hit the physical barrier known as the **Power Wall** around 2004–2005. Dynamic power consumption in CMOS circuits is proportional to the switching frequency and the square of the voltage ($P \propto C \cdot V^2 \cdot f$). 

Historically, Dennard scaling allowed clock frequency to rise while decreasing voltage without increasing power density. However, below 65 nm, sub-threshold leakage currents prevented further voltage reductions. Continuing to increase clock speeds would have led to uncontrollable thermal dissipation exceeding the cooling limits of silicon ($>100\text{ W/cm}^2$). 

Consequently, architects transitioned to **multi-core designs**, placing multiple lower-frequency, energy-efficient cores on a single chip to achieve higher aggregate throughput via parallelism while keeping total power within acceptable thermal limits.

---

### Q2: What is "The Free Lunch" in computing, and why did it end?
**Model Answer:**
"The Free Lunch" was the era prior to 2005 when sequential software applications automatically doubled in performance every 18 months without modifying the source code, driven entirely by exponential increases in single-core processor clock frequencies and architectural enhancements (pipelining, superscalar execution, out-of-order execution).

The Free Lunch ended because single-core clock speed scaling halted due to the **Power Wall**, **Instruction-Level Parallelism (ILP) wall**, and the **Memory Wall**. As a result, software developers can no longer rely on hardware to speed up sequential code; performance gains now require restructuring algorithms to run concurrently across multiple cores and processors.

---

### Q3: Explain why a sequential program cannot take advantage of a modern 64-core processor.
**Model Answer:**
A traditional sequential program consists of a single instruction stream executing on a single thread context. The operating system assigns this thread to **one physical or logical core**. 

Because sequential code contains no concurrency instructions (such as OpenMP pragmas, POSIX thread calls, or MPI message routines), the remaining 63 cores on the chip receive no instructions and remain idle. Thus, the program utilizes only $\frac{1}{64}\text{th}$ ($\approx 1.56\%$) of the chip's theoretical processing capacity.

---

## 12. Quick Revision Sheet

- **Parallel Computing Core Formula:**
  \[
  \boxed{\text{Decompose Problem} \longrightarrow \text{Execute Concurrently} \longrightarrow \text{Combine Partial Results}}
  \]
- **Performance Definition:**
  \[
  \boxed{P = \frac{\text{Work}}{\text{Time}}}
  \]
- **CMOS Dynamic Power Equation:**
  \[
  \boxed{P_{\text{dynamic}} = C \cdot V^2 \cdot f}
  \]
- **The Core Crisis That Created Modern HPC:**
  \[
  \boxed{\text{Transistors shrunk} \longrightarrow \text{Leakage current exploded} \longrightarrow \text{Voltage could not drop} \longrightarrow \text{Clock stopped at } \sim 3\text{--}4\text{ GHz} \longrightarrow \text{Multi-Core Era}}
  \]
- **Evaluation Weightage Checklist:**
  - End Semester Exam: **40%**
  - Lab Practical Exam: **40%**
  - Mid Semester Exam: **20%**
- **Common Exam Trap:**
  - *Trap:* Thinking that adding more cores makes sequential programs run faster.
  - *Reality:* Sequential code only executes on a single core; only parallelized software benefits from additional cores.

---

## 13. What You Should Be Able to Do After This Topic

Before moving to Module 01.02, verify that you can independently:
1. Explain the physical breakdown of Dennard Scaling and derive why power increases cubically with clock frequency if voltage scales with frequency.
2. State the exact three-component evaluation distribution of this course.
3. Write, compile, and run the sequential vs. OpenMP baseline array summation programs in C using GCC flags `-O3` and `-fopenmp`.
4. Solve numerical calculations involving processor dynamic power dissipation ratios and supercomputer electricity costs.
5. Provide precise, word-for-word examination definitions of High-Performance Computing, Parallel Computing, the Power Wall, and Dennard Scaling.
