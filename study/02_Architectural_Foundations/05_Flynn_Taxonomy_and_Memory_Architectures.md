# Module 02.05 — Flynn’s Taxonomy, Memory Architectures (UMA vs. ccNUMA) & OpenMP Preview

> **Prerequisites:** Module 02.03 (SIMD Vectorization), Module 02.04 (Memory Hierarchy & Bandwidth)  
> **Lecture Mapping:** Lecture 2 (Flynn's Classification, UMA vs. ccNUMA, Shared vs. Distributed Memory, OpenMP Preview)  
> **Exam Relevance:** MidSem (Core Theory Questions, Taxonomic Classifications, ccNUMA Diagrams) & EndSem  

---

## 1. Understanding the Basic Idea

Suppose you are the director of an orchestral music concert.
- You could have **one conductor** leading **one solo violinist** playing a single melody.
- Or you could have **one conductor** leading **twenty violinists** who all play the exact same notes in unison to produce a powerful, amplified sound.
- Or you could have **four independent conductors**, each directing their own separate section (violins, cellos, brass, and percussion), each playing completely different parts of a complex symphony simultaneously.

In 1972, Stanford University professor Michael J. Flynn asked:
> **How can we systematically classify every conceivable computer architecture that has ever been built or ever will be built?**

He realized that all computation can be reduced to two fundamental streams:
1. **The Instruction Stream:** The sequence of operations being executed.
2. **The Data Stream:** The sequence of operands being manipulated.

By considering whether each stream is **Single** or **Multiple**, Flynn created the famous **Flynn’s Taxonomy**—the universal classification system of parallel computer architectures.

---

## 2. Flynn’s Classical Taxonomy (1972)

Flynn’s classification divides all computer systems into a $2 \times 2$ matrix based on the multiplicity of hardware instruction streams and data streams:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           FLYNN'S 2x2 TAXONOMY MATRIX                           │
└─────────────────────────────────────────────────────────────────────────────────┘

                            DATA STREAM MULTIPLICITY
                     Single Data (SD)          Multiple Data (MD)
                 ┌─────────────────────────┬─────────────────────────┐
      Single     │          SISD           │          SIMD           │
   Instruction   │ (Single Instruction,    │ (Single Instruction,    │
       (SI)      │  Single Data)           │  Multiple Data)         │
                 │                         │                         │
INSTRUCTION      │ Classical Sequential    │ Vector Processors &     │
 STREAM          │ Uniprocessor (e.g., 486)│ GPUs (AVX-512, CUDA)    │
MULTIPLICITY     ├─────────────────────────┼─────────────────────────┤
      Multiple   │          MISD           │          MIMD           │
   Instruction   │ (Multiple Instruction,  │ (Multiple Instruction,  │
       (MI)      │  Single Data)           │  Multiple Data)         │
                 │                         │                         │
                 │ Space Shuttle Fault-    │ Modern Multi-Core CPUs  │
                 │ Tolerant Redundant CPUs │ & Supercomputer Clusters│
                 └─────────────────────────┴─────────────────────────┘
```

---

### Deep Dive into the Four Classes

```
1. SISD (Single Instruction, Single Data):
   Control Unit ──► [ Instruction Stream ] ──► Processor Core ◄──► [ Data Stream ] ◄──► Memory
   (One instruction operates on one data item at a time. Traditional uniprocessors.)

2. SIMD (Single Instruction, Multiple Data):
                    ┌──► ALU 0 ◄──► Data Element 0
   Control Unit ────┼──► ALU 1 ◄──► Data Element 1
   (1 Instruction)  ├──► ALU 2 ◄──► Data Element 2
                    └──► ALU 3 ◄──► Data Element 3
   (One instruction broadcast to multiple ALUs operating on packed vector registers.)

3. MISD (Multiple Instruction, Single Data):
   Control Unit 0 ──► ALU 0 ──┐
   Control Unit 1 ──► ALU 1 ──┼──► [ SINGLE SHARED DATA STREAM ]
   Control Unit 2 ──► ALU 2 ──┘
   (Multiple instructions operate on the same data item. Extremely rare; used for voting/fault tolerance.)

4. MIMD (Multiple Instruction, Multiple Data):
   Control Unit 0 ──► ALU 0 ◄──► Data Stream 0 (Core 0 running Thread A)
   Control Unit 1 ──► ALU 1 ◄──► Data Stream 1 (Core 1 running Thread B)
   Control Unit 2 ──► ALU 2 ◄──► Data Stream 2 (Core 2 running Thread C)
   (Autonomous cores executing completely different instructions on completely different data!)
```

#### 1. SISD (Single Instruction, Single Data)
- **Mechanism:** A single processor core executes a single sequential instruction stream operating on data stored in a single memory space.
- **Examples:** Classical mainframe computers, early Intel 8086 / 80486 processors, simple 8-bit microcontrollers (PIC, ATmega328P).

#### 2. SIMD (Single Instruction, Multiple Data)
- **Mechanism:** A single control unit issues a single machine instruction that is executed simultaneously by multiple processing units, each operating on distinct elements of a vector.
- **Execution:** Lockstep execution (zero thread divergence).
- **Examples:** Intel SSE, AVX, AVX-512 vector units; NVIDIA / AMD GPU Streaming Multiprocessors (Warp execution); historical vector supercomputers (Cray-1).

#### 3. MISD (Multiple Instruction, Single Data)
- **Mechanism:** Multiple processing units execute different instructions on the exact same stream of data.
- **Rarity:** Mathematically uncommon in commercial software.
- **Examples:** Flight control computers on the NASA Space Shuttle and Boeing commercial aircraft, where 3 to 4 redundant independent processors run different versions of flight stabilization software on identical sensor data to compare outputs and "vote" on decisions to guarantee safety against cosmic-ray bit flips or software bugs. Systolic arrays in DSP filters are also classified here.

#### 4. MIMD (Multiple Instruction, Multiple Data)
- **Mechanism:** Multiple autonomous processor cores operate completely independently, each fetching its own instructions and processing its own data.
- **Dominance:** MIMD is the **architectural foundation of virtually all modern parallel computers and supercomputers**.
- **Sub-classification:** MIMD systems are categorized by how their processors access memory:
  - **Shared-Memory MIMD (SMP / UMA / ccNUMA)**
  - **Distributed-Memory MIMD (Clusters / Supercomputers)**

---

## 3. Shared-Memory Memory Architectures: UMA vs. ccNUMA

Within MIMD shared-memory systems, all processor cores can directly address and access all physical memory. 
However, **how physical DRAM is connected to the CPU sockets** creates two radically different performance profiles.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      UMA vs. ccNUMA ARCHITECTURES                               │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. UMA (UNIFORM MEMORY ACCESS - Symmetric Multiprocessing / SMP):
 
     Core 0         Core 1         Core 2         Core 3
    ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐
    │ L1/L2│       │ L1/L2│       │ L1/L2│       │ L1/L2│
    └──┬───┘       └──┬───┘       └──┬───┘       └──┬───┘
       │              │              │              │
    ═══╧══════════════╧══════════════╧══════════════╧═════ SHARED MEMORY BUS
                             │
                             ▼
                    ┌─────────────────┐
                    │  CENTRAL DRAM   │  [ Every core has IDENTICAL latency ]
                    │  (Common RAM)   │  [ Bottleneck: Bus Contention!      ]
                    └─────────────────┘

 2. ccNUMA (CACHE-COHERENT NON-UNIFORM MEMORY ACCESS):
 
        SOCKET 0 (NUMA Node 0)                        SOCKET 1 (NUMA Node 1)
   ┌──────────────────────────────┐              ┌──────────────────────────────┐
   │ Core 0   Core 1   Core 2 ... │              │ Core 32  Core 33  Core 34 ...│
   │  L3 Cache (Shared Socket 0)  │              │  L3 Cache (Shared Socket 1)  │
   └──────────────┬───────────────┘              └──────────────┬───────────────┘
                  │                                             │
           LOCAL ACCESS (Fast: ~60 ns)                   LOCAL ACCESS (Fast: ~60 ns)
                  ▼                                             ▼
         ┌─────────────────┐      High-Speed Socket     ┌─────────────────┐
         │ LOCAL DRAM (S0) │ ◄════════════════════════► │ LOCAL DRAM (S1) │
         └─────────────────┘      Interconnect Bus      └─────────────────┘
                  ▲            (UPI / Infinity Fabric)          ▲
                  │                                             │
                  └──────── REMOTE ACCESS (Slow: ~120 ns!) ─────┘
```

---

### Detailed Architecture Comparison

#### Uniform Memory Access (UMA)
- **Concept:** All processors share physical memory through a single shared centralized bus or crossbar switch.
- **Symmetric Multiprocessing (SMP):** Every processor has identical access privileges and **identical access latency** to every memory location.
- **Advantages:** Extremely simple programming model. No need to worry about where arrays are physically placed in memory.
- **The Fatal Flaw (Scalability Limit):** The shared memory bus quickly saturates. As you add more cores (typically beyond 8 to 16 cores), cores spend more time fighting for bus arbitration than executing code. UMA cannot scale to high core counts!

#### Cache-Coherent Non-Uniform Memory Access (ccNUMA)
- **Concept:** Physical memory is partitioned and distributed locally to individual CPU sockets (called **NUMA Nodes**).
- **Non-Uniform Access Latency:**
  - **Local Memory Access:** When Core 0 reads from Socket 0's DRAM, access is direct and fast ($\approx 60\text{ ns}$).
  - **Remote Memory Access:** When Core 0 reads from Socket 1's DRAM, the request must travel across the inter-socket interconnect (Intel Ultra Path Interconnect - UPI, or AMD Infinity Fabric). This incurs **$2\times\text{ to }3\times$ higher latency** ($\approx 120\text{ to }180\text{ ns}$) and consumes interconnect bandwidth!
- **Cache-Coherence ("cc"):** Dedicated hardware snooping and directory protocols ensure that if Core 0 modifies a variable cached by Core 32 on another socket, Core 32's cache is automatically invalidated. The programmer sees a single unified address space.
- **Scalability:** Scales efficiently to hundreds of cores on multi-socket server blades.

---

## 4. Shared-Memory Systems vs. Distributed-Memory Systems

At the macro scale, parallel computing is divided into two primary programming and hardware models:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              SHARED MEMORY vs. DISTRIBUTED MEMORY COMPARISON                    │
└─────────────────────────────────────────────────────────────────────────────────┘

 SHARED-MEMORY SYSTEM (SMP / NUMA)           DISTRIBUTED-MEMORY SYSTEM (Cluster)
 
 ┌─────────────────────────────────┐        ┌───────────────┐     ┌───────────────┐
 │       COMMON ADDRESS SPACE      │        │ NODE 0 MEMORY │     │ NODE 1 MEMORY │
 │                                 │        │ (Private RAM) │     │ (Private RAM) │
 │  Variable X = 100               │        └───────┬───────┘     └───────┬───────┘
 └───────▲─────────▲─────────▲─────┘                │                     │
         │         │         │               Core 0 │              Core 1 │
      Core 0    Core 1    Core 2            ┌───────┴───────┐     ┌───────┴───────┐
                                            │ Process 0     │     │ Process 1     │
 Communication:                             └───────┬───────┘     └───────┬───────┘
 Direct memory reads/writes to                      │   Network Message   │
 shared global addresses.                           └──► (MPI_Send/Recv) ─┘
```

### Grand Comparison Matrix

| Architectural Feature | Shared-Memory System (SMP / ccNUMA) | Distributed-Memory System (HPC Cluster) |
|:---|:---|:---|
| **Memory Address Space** | **Single, unified global address space** shared by all processors | **Disjoint, independent private address spaces** per node |
| **Physical Hardware** | Multi-core, multi-socket server blade | Thousands of independent servers in server racks |
| **Data Communication** | Direct memory load/store instructions (`x = y`) | Explicit network messages (`MPI_Send`, `MPI_Recv`) |
| **Hardware Interconnect**| High-speed on-board bus (UPI, Infinity Fabric) | High-speed network fabric (InfiniBand, Slingshot, RoCE)|
| **Communication Latency**| Nanoseconds ($10\text{ to }150\text{ ns}$) | Microseconds ($0.5\text{ to }5\ \mu\text{s}$) — $100\times\text{ slower!}$ |
| **Scalability Limit** | Moderate (typically 2 to 8 sockets; $\le 512$ cores) | Massive (scales to thousands of nodes; $> 1,000,000$ cores) |
| **Primary Challenge** | Race conditions, false sharing, lock contention | Network communication overhead, distributed deadlocks |
| **Primary Programming Model**| **OpenMP**, POSIX Threads (Pthreads) | **MPI (Message Passing Interface)** |

---

## 5. Preview of OpenMP: The Shared-Memory Bridge

How does a programmer write software for shared-memory (UMA and ccNUMA) machines?

Historically, developers had to write low-level thread code using POSIX Threads (Pthreads), manually allocating thread stacks, managing thread IDs, and handling mutex locks (which we study thoroughly in Part B).

In 1997, the computing industry unified to create **OpenMP (Open Multi-Processing)**:
> **OpenMP** is an open-standard API for shared-memory parallel programming that enables C, C++, and Fortran compilers to automatically parallelize code through high-level compiler directives (`#pragma omp`).

### The Elegance of OpenMP
Instead of rewriting your algorithm from scratch, you annotate existing sequential loops:

```c
// Sequential C Code:
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i];
}

// OpenMP Parallel C Code:
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i];
}
```

The compiler interprets `#pragma omp parallel for` by:
1. Spawning a team of worker threads across available CPU cores (Fork).
2. Slicing the $N$ iterations into equal chunks across the thread team.
3. Synchronizing threads at the conclusion of the loop (Join).
4. Utilizing the shared physical address space so all threads directly read `a` and `b` and write `c` with zero networking code.

We dive deep into OpenMP architecture, data scoping, and fork-join mechanics starting in **Module 04**.

---

## 6. Practical Implementation: Observing ccNUMA Latency Penalty in C

Here is a C program demonstrating the **NUMA latency penalty**. It allocates a large memory buffer, touches it on one socket to bind physical DRAM pages locally (First-Touch policy), and then measures access time from the local socket versus a remote socket.

### NUMA Latency Exploration (`numa_latency.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <omp.h>

#define BUFFER_SIZE (128 * 1024 * 1024) // 128 Million ints = 512 MB

int main(void) {
    printf("============================================================\n");
    printf("        OBSERVING SHARED-MEMORY NUMA ALLOCATION             \n");
    printf("============================================================\n");

    // Allocate virtual memory buffer (no physical pages mapped yet!)
    int *data = (int *)malloc(BUFFER_SIZE * sizeof(int));
    if (!data) {
        fprintf(stderr, "Allocation failed!\n");
        return 1;
    }

    printf("Buffer Size: 512 MB\n");
    printf("Available OpenMP Threads: %d\n", omp_get_max_threads());

    // -------------------------------------------------------------
    // FIRST-TOUCH POLICY DEMONSTRATION:
    // In Linux/ccNUMA, physical DRAM pages are not assigned when malloc()
    // is called; they are assigned to the NUMA node of the thread that
    // WRITES to them FIRST!
    // -------------------------------------------------------------
    double t0 = omp_get_wtime();

    // Parallel initialization: each thread initializes its own chunk,
    // ensuring local NUMA DRAM page placement!
    #pragma omp parallel for schedule(static)
    for (int i = 0; i < BUFFER_SIZE; i++) {
        data[i] = 1;
    }

    double t1 = omp_get_wtime();
    printf("Parallel Local First-Touch Init Time: %.4f seconds\n", t1 - t0);

    // Read traversal benchmark
    double t2 = omp_get_wtime();
    long long sum = 0;

    #pragma omp parallel for schedule(static) reduction(+:sum)
    for (int i = 0; i < BUFFER_SIZE; i++) {
        sum += data[i];
    }

    double t3 = omp_get_wtime();
    printf("Parallel Read Traversal Time        : %.4f seconds (Sum: %lld)\n", 
           t3 - t2, sum);
    printf("============================================================\n");

    free(data);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp numa_latency.c -o numa_latency
./numa_latency
```

#### The Linux "First-Touch" NUMA Policy
- In Linux, `malloc()` merely reserves virtual address space; **no physical memory is allocated!**
- The physical DRAM page is allocated only when a memory address is written to for the first time.
- The operating system allocates the page on the **NUMA node physically closest to the core executing that first write**.
- If Master Thread 0 initializes the entire array sequentially in a single-threaded loop, the entire $512\text{ MB}$ array will be mapped to **Socket 0's DRAM**. When worker threads on Socket 1 later access their portions in parallel, all their reads must crawl across the remote interconnect bus, bottlenecking performance!

---

## 7. Important Comparisons

### UMA (SMP) vs. ccNUMA

| Feature | UMA (Uniform Memory Access) | ccNUMA (Cache-Coherent NUMA) |
|:---|:---|:---|
| **Memory Latency** | Identical for all cores to all memory addresses | Non-uniform (Fast local DRAM, slow remote DRAM) |
| **Physical Layout** | Single centralized memory controller & bus | Memory distributed locally across multiple sockets |
| **Scalability** | Poor (bus saturates at $> 8\text{ to }16$ cores) | High (scales to hundreds of cores per server blade)|
| **Hardware Cost** | Inexpensive, commodity desktop motherboards | Complex multi-socket server motherboards |
| **Software Sensitivity**| Insensitive to thread and memory placement | Highly sensitive to thread affinity and data placement |
| **Primary Bottleneck** | Centralized bus contention | Interconnect bandwidth (UPI / Infinity Fabric) |

---

## 8. Exam-Ready Definitions

### Flynn’s Taxonomy
> **Flynn’s Taxonomy** is a classification system for computer architectures proposed by Michael J. Flynn in 1972 that categorizes computing systems into four classes (SISD, SIMD, MISD, MIMD) based on the multiplicity of instruction streams and data streams processed simultaneously.

### Symmetric Multiprocessing (SMP / UMA)
> **Symmetric Multiprocessing (SMP)** is a computer architecture where two or more identical processors are connected to a single, shared main memory with uniform access latency, controlled by a single operating system instance.

### ccNUMA (Cache-Coherent Non-Uniform Memory Access)
> **ccNUMA** is a shared-memory multiprocessing architecture where physical memory is distributed among processor nodes, resulting in variable access latencies depending on memory location, with hardware protocols maintaining cache coherence across all processors.

### First-Touch Policy
> The **First-Touch Policy** is an operating system memory management strategy in NUMA systems whereby physical memory pages are allocated on the NUMA node physically local to the thread that first writes to the page, rather than the thread that allocated it.

---

## 9. Solved Problems

### Problem 1: Classifying Architectures Using Flynn’s Taxonomy
Classify each of the following computing systems into its proper Flynn taxonomy category and justify your answer:
1. An Intel Core i9 processor executing an OpenMP parallel loop where each core runs different tasks.
2. An AVX-512 vector execution unit processing 8 double-precision numbers simultaneously.
3. A classic single-core Intel 80386 processor from 1985.
4. A flight control system with three redundant CPUs executing distinct control algorithms on identical gyroscope sensor readings.
5. An NVIDIA H100 GPU executing a CUDA warp across 32 threads in lockstep.

#### Solution:
1. **Intel Core i9 running OpenMP tasks:**
   - **Category: MIMD (Multiple Instruction, Multiple Data).**
   - *Justification:* Multiple independent cores each execute their own instruction streams on different data.
2. **AVX-512 Vector Execution:**
   - **Category: SIMD (Single Instruction, Multiple Data).**
   - *Justification:* A single vector instruction (`VADDPD`) is broadcast across multiple 64-bit vector lanes in a single core.
3. **Intel 80386 Processor:**
   - **Category: SISD (Single Instruction, Single Data).**
   - *Justification:* A single uniprocessor core fetching one instruction operating on one scalar data word.
4. **Flight Control Redundant Voting System:**
   - **Category: MISD (Multiple Instruction, Single Data).**
   - *Justification:* Multiple distinct instruction programs execute simultaneously on the exact same stream of sensor input data.
5. **NVIDIA GPU Warp Execution:**
   - **Category: SIMD (or SIMT - Single Instruction, Multiple Threads).**
   - *Justification:* 32 threads in a warp execute the same instruction in lockstep across different data elements.

---

### Problem 2: NUMA Access Latency Penalty Calculation
A dual-socket ccNUMA server has Socket 0 and Socket 1.
- **Local Memory Access Latency:** $60\text{ ns}$.
- **Remote Memory Access Latency:** $150\text{ ns}$ (across the inter-socket interconnect).

A parallel scientific code runs with 64 threads (32 threads on Socket 0, 32 threads on Socket 1). 
Due to poor memory initialization, all array data was allocated on **Socket 0's local DRAM**.
As a result:
- Socket 0 threads perform $100\%$ local memory accesses.
- Socket 1 threads perform $100\%$ remote memory accesses.

1. Calculate the average memory latency experienced by Socket 0 threads.
2. Calculate the average memory latency experienced by Socket 1 threads.
3. Calculate the overall average memory access latency across all 64 threads.
4. If proper NUMA first-touch initialization is implemented such that all threads access 100% local memory, calculate the overall average memory access latency and the percentage reduction in latency.

#### Solution:
1. **Socket 0 Latency:**
   \[
   L_{\text{Socket 0}} = \mathbf{60\text{ ns}}
   \]
2. **Socket 1 Latency:**
   \[
   L_{\text{Socket 1}} = \mathbf{150\text{ ns}}
   \]
3. **Overall Server Average Latency (Unoptimized):**
   \[
   L_{\text{avg}} = \frac{32 \times 60\text{ ns} + 32 \times 150\text{ ns}}{64} = \frac{1,920 + 4,800}{64} = \frac{6,720}{64} = \mathbf{105\text{ ns}}
   \]
4. **Optimized Latency (First-Touch NUMA Placement):**
   - Both Socket 0 and Socket 1 threads now access 100% local DRAM:
   \[
   L_{\text{optimized}} = \mathbf{60\text{ ns}}
   \]
   - Percentage reduction in memory latency:
   \[
   \text{Reduction} = \frac{105\text{ ns} - 60\text{ ns}}{105\text{ ns}} \times 100\% = \frac{45}{105} \times 100\% \approx \mathbf{42.86\%}
   \]
   *Insight:* Proper NUMA page allocation cuts average memory latency by nearly half!

---

## 10. Conceptual Examination Questions & Answers

### Q1: Why is cache coherence necessary in ccNUMA systems, and what problem does it solve?
**Model Answer:**
In a ccNUMA system, multiple processor sockets share a single logical address space, and each socket possesses its own local private caches (L1, L2, L3). 

Without cache coherence, the **Cache Inconsistency Problem** occurs:
1. Socket 0 reads variable `X = 5` from DRAM and stores it in its local L1 cache.
2. Socket 1 also reads `X = 5` from DRAM and stores it in its local L1 cache.
3. Socket 0 updates `X = 10` in its local cache.
4. If Socket 1 reads `X`, it will read the stale cached value `5`, leading to silent algorithmic corruption!

Hardware cache coherence protocols (such as MESI/MOESI) monitor and broadcast cache state changes across the interconnect. When Socket 0 writes to `X`, the coherence protocol immediately sends an invalidation signal to Socket 1, ensuring that all processors always observe a consistent, up-to-date view of shared memory.

---

### Q2: What is the primary architectural limitation of UMA (SMP) systems that led to the adoption of NUMA?
**Model Answer:**
The primary limitation of Uniform Memory Access (UMA) is the **shared memory bus bottleneck**.

In UMA, all processor cores connect to a single centralized memory controller and DRAM pool across a common shared bus. As the core count increases, the aggregate demand for memory bandwidth rapidly exceeds the maximum physical bandwidth of the single bus. 

Cores spend increasing amounts of time stalled in arbitration queues waiting for bus access, resulting in diminishing performance returns beyond 8 to 16 cores. 

NUMA overcomes this bottleneck by distributing memory controllers and physical DRAM channels locally across multiple sockets, multiplying aggregate memory bandwidth linearly with socket count.

---

## 11. Quick Revision Sheet

- **Flynn’s Taxonomy Matrix:**
  - **SISD:** Single core uniprocessor.
  - **SIMD:** Vector registers (SSE/AVX), GPU warps.
  - **MISD:** Spacecraft fault-tolerant voting systems.
  - **MIMD:** Multi-core CPUs and supercomputers.
- **UMA vs. ccNUMA:**
  - **UMA:** Centralized RAM, uniform access latency, limited scalability ($\le 16$ cores).
  - **ccNUMA:** Distributed local RAM per socket, fast local access ($\approx 60\text{ ns}$), slow remote access ($\approx 150\text{ ns}$), scales to hundreds of cores.
- **Shared Memory vs. Distributed Memory:**
  - **Shared Memory:** Unified address space, communication via load/store, OpenMP / Pthreads.
  - **Distributed Memory:** Disjoint address spaces, communication via network messages, MPI.
- **Linux First-Touch Rule:**
  \[
  \boxed{\text{Physical RAM page assigned to the NUMA node of the thread that WRITES to it first!}}
  \]
- **Common Exam Trap:**
  - *Trap:* Thinking NUMA means distributed memory without a shared address space.
  - *Reality:* NUMA is a **shared-memory system**. Any core can directly read/write any address; only the access *latency* is non-uniform!

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 03 (Lectures 3–5: Parallel Performance & Scaling Laws), verify that you can independently:
1. Reproduce Flynn’s 2x2 taxonomy matrix and classify any given computer architecture.
2. Draw the block diagrams for UMA (SMP) and ccNUMA multi-socket architectures.
3. Explain the Linux First-Touch memory allocation policy and explain how it impacts ccNUMA performance.
4. Compare Shared-Memory and Distributed-Memory paradigms across address spaces, communication mechanisms, and scaling limits.
5. Provide a clear preview of the OpenMP shared-memory programming model and explain why it abstracts away low-level thread mechanics.
