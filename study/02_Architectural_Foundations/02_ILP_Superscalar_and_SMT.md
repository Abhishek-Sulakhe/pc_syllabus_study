# Module 02.02 — Instruction-Level Parallelism (ILP), Superscalar Processors & Simultaneous Multithreading (SMT)

> **Prerequisites:** Module 02.01 (Stored-Program Model & Instruction Pipelining)  
> **Lecture Mapping:** Lecture 2 (Architectural Foundations: ILP, Superscalar Architecture, SMT / Hyper-Threading)  
> **Exam Relevance:** MidSem (Definitions, Hardware Mechanisms, Superscalar vs SMT Comparisons) & EndSem  

---

## 1. Understanding the Basic Idea

In Module 02.01, we explored how a pipelined CPU overlaps instruction stages, aiming for an ideal throughput of **1 instruction completed per clock cycle** ($\text{CPI} = 1.0$).

Now ask a bolder question:
> **Why stop at 1 instruction per cycle? Can a single CPU core execute two, four, or eight instructions simultaneously in the very same clock cycle?**

Imagine a supermarket checkout area.
- A standard pipelined processor has **one cashier** who works on one customer at a time, moving them smoothly from scanning to bagging to paying.
- A **Superscalar Processor** is like opening **four parallel checkout lanes** inside the same store. If four customers have independent carts with no shared items, the store can check out 4 customers in the exact same minute!

```
Single-Issue Pipelined Core (1 instruction/cycle max):
Cycle 1: [ Inst 1 ]
Cycle 2: [ Inst 2 ]
Cycle 3: [ Inst 3 ]

4-Way Superscalar Core (Up to 4 instructions/cycle):
Cycle 1: [ Inst 1 ] [ Inst 2 ] [ Inst 3 ] [ Inst 4 ]
Cycle 2: [ Inst 5 ] [ Inst 6 ] [ Inst 7 ] [ Inst 8 ]
```

And what happens if Customer 1 realizes they forgot their wallet and must wait for someone to bring money? In a regular checkout line, that lane freezes completely. 
**Simultaneous Multithreading (SMT)** solves this: while Customer 1 is frozen waiting for their wallet (a memory stall), the cashier immediately starts scanning items for Customer 2 from a completely different shopping thread!

---

## 2. Instruction-Level Parallelism (ILP) & Bernstein’s Conditions

Before a CPU can execute multiple instructions in parallel, it must verify that doing so will not alter the mathematical outcome of the program.

> **Instruction-Level Parallelism (ILP)** is a measure of how many operations in a computer program can be executed simultaneously without violating algorithmic correctness.

### When Can Two Instructions Execute in Parallel? Bernstein’s Conditions (1966)
Arthur Bernstein formalized the mathematical conditions required for two segments of code (or instructions $I_1$ and $I_2$) to execute concurrently.

Let:
- $I(I_1)$ be the **Input set** (variables/registers read by $I_1$).
- $O(I_1)$ be the **Output set** (variables/registers written by $I_1$).
- $I(I_2)$ be the **Input set** read by $I_2$.
- $O(I_2)$ be the **Output set** written by $I_2$.

Two instructions $I_1$ and $I_2$ can execute in parallel if and only if **all three** of the following conditions hold simultaneously:

\[
\boxed{
\begin{aligned}
1. \quad I(I_1) \cap O(I_2) &= \emptyset \quad &\text{(No Anti-Dependency: WAR)} \\
2. \quad O(I_1) \cap I(I_2) &= \emptyset \quad &\text{(No True Data Dependency: RAW)} \\
3. \quad O(I_1) \cap O(I_2) &= \emptyset \quad &\text{(No Output Dependency: WAW)}
\end{aligned}
}
\]

```
              BERNSTEIN'S DATA DEPENDENCY VIOLATIONS
   
   1. True Dependency (RAW)        2. Anti-Dependency (WAR)       3. Output Dependency (WAW)
      Read-After-Write               Write-After-Read               Write-After-Write
      
      I1: R1 = R2 + R3               I1: R4 = R1 + R5               I1: R1 = R2 + R3
             │                                │                            │
             ▼                                ▼                            ▼
      I2: R6 = R1 + R7               I2: R1 = R6 + R7               I2: R1 = R8 + R9
      (I2 reads what I1 writes!)     (I2 overwrites what I1 reads!) (Both write to R1!)
```

- **RAW (True Data Dependency):** A fundamental algorithmic dependency. Cannot be eliminated; $I_2$ must wait for $I_1$.
- **WAR and WAW (Name Dependencies):** These are artificial dependencies caused by a shortage of register names! Modern superscalar CPUs eliminate WAR and WAW hazards automatically using **Register Renaming**.

---

## 3. How Superscalar Processors Work

A **Superscalar Processor** implements dynamic parallelism at runtime using dedicated hardware logic on the CPU die.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     ANATOMY OF AN OUT-OF-ORDER SUPERSCALAR CORE                 │
└─────────────────────────────────────────────────────────────────────────────────┘

                ┌─────────────────────────────────────────────────┐
                │          INSTRUCTION FETCH (IF) & DECODE        │
                │        Fetches 4 to 8 instructions / cycle      │
                └────────────────────────┬────────────────────────┘
                                         │
                                         ▼
                ┌─────────────────────────────────────────────────┐
                │       REGISTER RENAMING & ALLOCATION TABLE      │
                │ Maps architectural registers (R1..R32) to 128+  │
                │ physical internal registers (eliminates WAR/WAW)│
                └────────────────────────┬────────────────────────┘
                                         │
                                         ▼
                ┌─────────────────────────────────────────────────┐
                │         REORDER BUFFER & RESERVATION STATIONS   │
                │ Instructions wait here until operands ready.    │
                │ Dispatched OUT-OF-ORDER as soon as inputs ready!│
                └───────┬────────────┬────────────┬───────────────┘
                        │            │            │
         ┌──────────────┘            │            └──────────────┐
         ▼                           ▼                           ▼
 ┌───────────────┐           ┌───────────────┐           ┌───────────────┐
 │ Integer ALU 0 │           │ Integer ALU 1 │           │ Floating-Point│
 │ & Branch Unit │           │ & Vector SIMD │           │ Load/Store    │
 └───────┬───────┘           └───────┬───────┘           └───────┬───────┘
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     ▼
                ┌─────────────────────────────────────────────────┐
                │             IN-ORDER RETIRE / COMMIT            │
                │ Instructions commit results to architectural    │
                │ state in original program order (preserves bugs)│
                └─────────────────────────────────────────────────┘
```

### The Magic of Out-of-Order (OoO) Execution
Consider this sequence of instructions:
```assembly
I1: LOAD  R1, [Memory_Address_A]  ; Slow DRAM fetch! (Stalls for 200 cycles)
I2: ADD   R2, R1, R3              ; Depends on I1 (RAW). Must wait!
I3: ADD   R4, R5, R6              ; Completely independent!
I4: MULT  R7, R8, R9              ; Completely independent!
```
- In an **In-Order processor**, instruction $I_3$ and $I_4$ are blocked behind $I_2$, even though they have nothing to do with $R_1$! The entire CPU sits completely frozen for 200 cycles.
- In an **Out-of-Order (OoO) Superscalar processor**, the hardware notices that $I_3$ and $I_4$ have all their inputs ready. It bypasses the stalled instructions $I_1$ and $I_2$, executes $I_3$ and $I_4$ immediately, and holds their results in a Reorder Buffer until $I_1$ and $I_2$ finally complete!

### The Limit of ILP: The "ILP Wall"
Why can't we build a 32-issue superscalar CPU?
1. **Dependency Chains:** Real code has an average natural ILP of only $2\text{ to }4$ instructions. Finding 32 independent instructions in a single thread is statistically impossible in standard software.
2. **Exponential Hardware Complexity:** The logic required to compare dependencies between $N$ instructions grows as $\mathcal{O}(N^2)$. A 32-wide issue window would require over 1,000 comparator circuits operating at sub-nanosecond speeds, consuming enormous electrical power.
3. This limitation is known as the **ILP Wall**.

---

## 4. Simultaneous Multithreading (SMT / Hyper-Threading)

Even with sophisticated out-of-order execution, modern wide superscalar cores frequently suffer from **execution unit underutilization**:
- Some cycles execute 3 instructions.
- Some cycles execute 1 instruction.
- Many cycles execute **0 instructions** because all in-flight instructions are stalled waiting for cache misses or branch resolutions.

```
       CORE EXECUTION RESOURCE WASTAGE (HORIZONTALLY & VERTICALLY)
 Issue Slots ──►  ALU 0   ALU 1   FPU 0   FPU 1   MEM 0   MEM 1
 Cycle 1:        [  I1 ] [  I2 ] [     ] [     ] [     ] [     ]  ◄── Partial Use
 Cycle 2:        [     ] [     ] [     ] [     ] [     ] [     ]  ◄── Horizontal Waste
 Cycle 3:        [     ] [     ] [     ] [     ] [     ] [     ]  ◄── Vertical Waste (Stall)
 Cycle 4:        [  I3 ] [     ] [  I4 ] [     ] [  I5 ] [     ]
```

### Enter SMT: Merging Two Threads into One Core
**Simultaneous Multithreading (SMT)** (commercialized by Intel as **Hyper-Threading**) is a hardware architecture technique where:
> A single physical CPU core presents itself to the operating system as **two (or more) logical cores**, sharing the physical execution pipelines, ALU units, and caches, while replicating only the architectural state.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│               SIMULTANEOUS MULTITHREADING (SMT) EXECUTION                       │
└─────────────────────────────────────────────────────────────────────────────────┘

 Replicated per Logical Thread:               Shared Core Execution Units:
 ┌───────────────────────────┐                ┌────────────────────────────────┐
 │ Thread Context 0:         │                │ 4 Integer ALUs                 │
 │ Program Counter 0 (PC₀)   │ ═════════╗     │ 2 Floating-Point Vector Units  │
 │ Architectural Regs (RAX..)│          ║     │ 2 Load/Store Memory Units      │
 └───────────────────────────┘          ╠═══► │ L1 Data & Instruction Caches   │
 ┌───────────────────────────┐          ║     │ L2 Unified Cache               │
 │ Thread Context 1:         │ ═════════╝     │ Out-of-Order Execution Window  │
 │ Program Counter 1 (PC₁)   │                └────────────────────────────────┘
 │ Architectural Regs (RBX..)│
 └───────────────────────────┘
```

### How SMT Eliminates Idle Cycles
In an SMT processor, instructions from **Thread 0** and **Thread 1** are dispatched into the execution pipelines simultaneously:

```
 Issue Slots ──►  ALU 0   ALU 1   FPU 0   FPU 1   MEM 0   MEM 1
 Cycle 1:        [ T0 ]  [ T0 ]  [ T1 ]  [     ] [ T1 ]  [     ]
 Cycle 2:        [ T1 ]  [     ] [ T0 ]  [ T0 ]  [     ] [ T1 ]
 Cycle 3:        [ T1 ]  [ T1 ]  [ T1 ]  [     ] [ T0 ]  [ T0 ]
 Cycle 4:        [ T0 ]  [ T0 ]  [ T1 ]  [ T1 ]  [ T0 ]  [ T1 ]
                 ▲
                 Nearly 100% of execution slots are filled!
```

---

## 5. Physical Cores vs. Logical Cores: When Does SMT Help & When Does It Hurt?

Many beginners believe that enabling Hyper-Threading turns an 8-core CPU into a true 16-core CPU. **This is fundamentally untrue.**

### The Silicon Cost vs. Speedup Reality
- Adding a true second physical core requires **$100\%$ more silicon area** and almost doubles power consumption.
- Adding SMT to an existing core requires only **$\approx 5\%$ more silicon area** (just duplicating registers and program counters).
- In typical mixed integer/server workloads, SMT delivers a **$20\%\text{ to }30\%$ throughput boost**—an incredible performance-per-dollar triumph.

### When SMT Hurts in Scientific HPC Workloads
In high-performance computing, supercomputing centers frequently **disable SMT (Hyper-Threading)** in the BIOS! Why?

1. **Cache Thrashing:**
   Both logical threads share the small $32\text{ KB}$ L1 data cache. If both threads run memory-intensive algorithms, they constantly evict each other's cache lines, causing cache miss rates to skyrocket.
2. **Execution Unit Saturation:**
   If Thread 0 is already running heavy AVX-512 vector code, it is already using 100% of the core's vector execution units. Thread 1 has no free arithmetic units to use; it simply competes with Thread 0, cutting Thread 0's throughput in half while adding context-switching jitter.
3. **Non-Deterministic Tail Latency:**
   In distributed MPI simulations where thousands of cores must synchronize at a barrier, the entire supercomputer runs at the speed of the slowest core. The interference caused by SMT creates performance jitter, dragging down the entire cluster.

---

## 6. Practical Implementation: Observing Superscalar ILP in C

Here is a C program demonstrating how superscalar multiple-issue execution units exploit loop unrolling to double instruction throughput.

### Superscalar Execution Benchmark (`superscalar_bench.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N 200000000

int main(void) {
    printf("============================================================\n");
    printf("     DEMONSTRATING SUPERSCALAR MULTI-ISSUE PARALLELISM      \n");
    printf("============================================================\n");

    struct timespec start, end;

    // -------------------------------------------------------------
    // Test 1: Single Accumulator (Serialization: 1 instruction per cycle limit)
    // -------------------------------------------------------------
    double sum = 0.0;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < N; i++) {
        sum += 1.0000001; // Continuous RAW dependency on 'sum'
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double time_single = (end.tv_sec - start.tv_sec) + 
                         (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("Single Accumulator Time    : %.4f seconds (Sum: %.2f)\n", 
           time_single, sum);

    // -------------------------------------------------------------
    // Test 2: Dual Independent Accumulators (Superscalar Dual-Issue)
    // -------------------------------------------------------------
    double sum1 = 0.0, sum2 = 0.0;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < N; i += 2) {
        sum1 += 1.0000001; // Issued to ALU 0
        sum2 += 1.0000001; // Issued to ALU 1 concurrently!
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double time_dual = (end.tv_sec - start.tv_sec) + 
                       (end.tv_nsec - start.tv_nsec) / 1e9;

    double final_sum2 = sum1 + sum2;
    printf("Dual Accumulators Time      : %.4f seconds (Sum: %.2f)\n", 
           time_dual, final_sum2);

    // -------------------------------------------------------------
    // Test 3: Four Independent Accumulators (4-Way Superscalar Issue)
    // -------------------------------------------------------------
    double s1 = 0.0, s2 = 0.0, s3 = 0.0, s4 = 0.0;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < N; i += 4) {
        s1 += 1.0000001;
        s2 += 1.0000001;
        s3 += 1.0000001;
        s4 += 1.0000001;
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double time_quad = (end.tv_sec - start.tv_sec) + 
                       (end.tv_nsec - start.tv_nsec) / 1e9;

    double final_sum4 = s1 + s2 + s3 + s4;
    printf("Quad Accumulators Time      : %.4f seconds (Sum: %.2f)\n", 
           time_quad, final_sum4);

    printf("\nSpeedup achieved by exposing ILP:\n");
    printf("  Dual vs. Single : %.2fx faster\n", time_single / time_dual);
    printf("  Quad vs. Single : %.2fx faster\n", time_single / time_quad);
    printf("============================================================\n");
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O2 superscalar_bench.c -o superscalar_bench
./superscalar_bench
```

*Expected Output (on an Intel x86-64 superscalar processor):*
```
============================================================
     DEMONSTRATING SUPERSCALAR MULTI-ISSUE PARALLELISM      
============================================================
Single Accumulator Time    : 0.1820 seconds (Sum: 200000020.00)
Dual Accumulators Time      : 0.0915 seconds (Sum: 200000020.00)
Quad Accumulators Time      : 0.0460 seconds (Sum: 200000020.00)

Speedup achieved by exposing ILP:
  Dual vs. Single : 1.99x faster
  Quad vs. Single : 3.96x faster
============================================================
```

#### Key Observation:
By simply unrolling the loop into **4 independent accumulator streams**, the program ran **almost $4\times$ faster on the exact same CPU core**! The 4-way superscalar dispatch engine was able to issue 4 additions per cycle instead of waiting for a single accumulator chain.

---

## 7. Important Comparisons

### Superscalar Execution vs. Simultaneous Multithreading (SMT)

| Feature | Superscalar Architecture | Simultaneous Multithreading (SMT) |
|:---|:---|:---|
| **Primary Goal** | Exploit **Instruction-Level Parallelism (ILP)** within a single thread | Exploit **Thread-Level Parallelism (TLP)** across multiple threads |
| **Number of Threads** | Exactly **1 thread** at any given moment | **2 or more threads** concurrently active in hardware |
| **Replicated State** | None (single architectural register set, 1 PC) | Replicated Architectural State (PC, Register Files per thread) |
| **Execution Units** | Replicated functional units (multiple ALUs/FPUs) | Shared functional units among active threads |
| **Detection of Parallelism** | Dynamic hardware dependency analysis at runtime | Program explicitly created by programmer (e.g., OpenMP/Pthreads) |
| **Failure Mode** | Stalls on data dependencies or memory misses | Cache contention and execution resource starvation |

---

### Physical Core vs. Logical Core (SMT)

| Metric | Physical Core | Logical Core (SMT / Hyper-Thread) |
|:---|:---|:---|
| **Hardware Essence** | A complete, autonomous hardware CPU on silicon | A virtualized software thread context sharing a physical core |
| **ALUs & Caches** | Has its own dedicated private ALUs, FPUs, L1 caches| Shares ALUs, FPUs, L1, L2 with its sibling logical thread |
| **Silicon Area Cost** | $100\%$ full silicon footprint | $\approx 5\%$ additional silicon footprint |
| **Theoretical Speedup**| $2.0\times$ (doubling physical cores doubles capacity) | Typically $1.15\times\text{ to }1.30\times$ (never $2.0\times$) |
| **HPC Best Practice** | Always utilized and pinned to computational tasks | Frequently **disabled** in BIOS for scientific simulation clusters |

---

## 8. Exam-Ready Definitions

### Instruction-Level Parallelism (ILP)
> **Instruction-Level Parallelism (ILP)** is the potential within a sequential instruction stream to overlap or execute multiple independent machine instructions simultaneously during a single clock cycle without altering the final program output.

### Superscalar Processor
> A **Superscalar Processor** is a microprocessor architecture that implements instruction-level parallelism by dispatching and executing multiple instructions concurrently across multiple redundant execution units (such as ALUs, FPUs, and Load/Store units) during each clock cycle.

### Simultaneous Multithreading (SMT)
> **Simultaneous Multithreading (SMT)** is a processor design technique that permits multiple independent execution threads to issue instructions simultaneously to the execution units of a single physical superscalar core during the same clock cycle, sharing execution resources while maintaining separate architectural states.

### Register Renaming
> **Register Renaming** is a dynamic hardware optimization technique used in out-of-order superscalar processors that maps architectural register names to a larger pool of physical internal registers, eliminating false dependencies (Write-After-Read and Write-After-Write hazards).

---

## 9. Solved Problems

### Problem 1: Applying Bernstein’s Conditions
Analyze the following four machine instructions and determine which instructions can execute concurrently according to Bernstein’s conditions:
```assembly
I1: R1 = R2 + R3
I2: R4 = R1 * R5
I3: R6 = R7 + R8
I4: R2 = R6 - R9
```

#### Solution:
1. **Identify the Input and Output sets for each instruction:**
   - $I(I_1) = \{R_2, R_3\}, \quad O(I_1) = \{R_1\}$
   - $I(I_2) = \{R_1, R_5\}, \quad O(I_2) = \{R_4\}$
   - $I(I_3) = \{R_7, R_8\}, \quad O(I_3) = \{R_6\}$
   - $I(I_4) = \{R_6, R_9\}, \quad O(I_4) = \{R_2\}$

2. **Evaluate pairwise dependencies:**
   - **$I_1$ and $I_2$:**
     $O(I_1) \cap I(I_2) = \{R_1\} \cap \{R_1, R_5\} = \{R_1\} \neq \emptyset$ (RAW True Dependency).
     $\implies \mathbf{I_1 \text{ and } I_2 \text{ CANNOT execute in parallel.}}$
   - **$I_1$ and $I_3$:**
     $I(I_1) \cap O(I_3) = \{R_2, R_3\} \cap \{R_6\} = \emptyset$
     $O(I_1) \cap I(I_3) = \{R_1\} \cap \{R_7, R_8\} = \emptyset$
     $O(I_1) \cap O(I_3) = \{R_1\} \cap \{R_6\} = \emptyset$
     $\implies \mathbf{I_1 \text{ and } I_3 \text{ CAN execute in parallel!}}$
   - **$I_1$ and $I_4$:**
     $I(I_1) \cap O(I_4) = \{R_2, R_3\} \cap \{R_2\} = \{R_2\} \neq \emptyset$ (WAR Anti-Dependency on $R_2$).
     $\implies \mathbf{I_1 \text{ and } I_4 \text{ CANNOT execute in parallel}}$ without register renaming.
   - **$I_2$ and $I_3$:**
     Inputs and outputs share no registers.
     $\implies \mathbf{I_2 \text{ and } I_3 \text{ CAN execute in parallel!}}$
   - **$I_3$ and $I_4$:**
     $O(I_3) \cap I(I_4) = \{R_6\} \cap \{R_6, R_9\} = \{R_6\} \neq \emptyset$ (RAW True Dependency).
     $\implies \mathbf{I_3 \text{ and } I_4 \text{ CANNOT execute in parallel.}}$

3. **Conclusion:**
   The pairs that can execute concurrently are **$(I_1, I_3)$** and **$(I_2, I_3)$**.

---

### Problem 2: SMT Throughput and Speedup Modeling
A server has $8$ physical CPU cores.
- When running a compute-intensive scientific benchmark with SMT disabled (1 thread per core), each physical core achieves a sustained throughput of $10\text{ GFLOP/s}$.
- When SMT is enabled (2 logical threads per core = 16 total threads), the two threads running on the same core share functional units and experience cache contention. As a result, each individual thread's throughput drops to $6.25\text{ GFLOP/s}$.

Calculate:
1. The total aggregate throughput of the 8-core server with SMT disabled.
2. The total aggregate throughput of the 8-core server with SMT enabled.
3. The overall speedup delivered by enabling SMT.
4. Explain why the throughput per thread dropped from $10\text{ GFLOP/s}$ to $6.25\text{ GFLOP/s}$.

#### Solution:
1. **Total Throughput with SMT Disabled (8 Threads):**
   \[
   T_{\text{disabled}} = 8\text{ cores} \times 10.0\text{ GFLOP/s} = \mathbf{80.0\text{ GFLOP/s}}
   \]
2. **Total Throughput with SMT Enabled (16 Threads):**
   \[
   T_{\text{enabled}} = 16\text{ threads} \times 6.25\text{ GFLOP/s} = \mathbf{100.0\text{ GFLOP/s}}
   \]
3. **Speedup from Enabling SMT:**
   \[
   \text{Speedup} = \frac{T_{\text{enabled}}}{T_{\text{disabled}}} = \frac{100.0\text{ GFLOP/s}}{80.0\text{ GFLOP/s}} = \mathbf{1.25\times\ (25\%\text{ increase})}
   \]
4. **Why did individual thread throughput drop?**
   Because the two logical threads must share the **same physical execution resources**: the ALUs, FPUs, and L1/L2 caches. Neither thread has dedicated 100% ownership of the core's pipelines. Although total core throughput increased from $10$ to $12.5\text{ GFLOP/s}$ by filling idle stall slots, each individual thread runs slower ($6.25\text{ GFLOP/s}$).

---

## 10. Conceptual Examination Questions & Answers

### Q1: What is the "ILP Wall," and what fundamental hardware problems caused it?
**Model Answer:**
The **ILP Wall** is the limit beyond which computer architects can no longer extract additional performance by making a single processor core wider (issuing more instructions per cycle).

It was caused by three physical and architectural constraints:
1. **Limited Inherent Parallelism:** Typical sequential programs contain an average of only 2 to 4 independent instructions within an execution window before encountering a data dependency or a conditional branch.
2. **Quadratic Hardware Complexity:** The hardware logic required to check dependencies, rename registers, and bypass data across $N$ issue slots scales as $\mathcal{O}(N^2)$. Building issue windows wider than 6 to 8 instructions consumes prohibitive silicon area and electrical power.
3. **Branch Prediction Limitations:** Even with $98\%$ branch prediction accuracy, mispredicted branches deep inside wide out-of-order pipelines require massive pipeline flushes that waste dozens of completed instructions.

---

### Q2: Why is Register Renaming essential for superscalar processors?
**Model Answer:**
In traditional machine code, compilers are constrained to a small set of architectural registers (e.g., 16 registers in x86-64 or 32 in ARM). Consequently, compilers frequently reuse the same register name for unrelated variables, creating artificial **name dependencies**:
- **WAR (Write-After-Read / Anti-dependency)**
- **WAW (Write-After-Write / Output dependency)**

Register Renaming dynamically maps these small architectural register names to a much larger pool of physical hardware registers (typically 128 to 256 physical registers) inside the out-of-order execution engine. This eliminates all false WAR and WAW hazards, allowing instructions that write to the same architectural register name to execute in parallel without corrupting data.

---

## 11. Quick Revision Sheet

- **Bernstein’s Conditions for Parallelism:**
  \[
  \boxed{I_1 \text{ and } I_2 \text{ Parallel} \iff (I_1 \cap O_2 = \emptyset) \land (O_1 \cap I_2 = \emptyset) \land (O_1 \cap O_2 = \emptyset)}
  \]
- **Dependency Summary:**
  - **RAW:** True data dependency $\implies$ Fundamental algorithmic limit; must wait.
  - **WAR:** Anti-dependency $\implies$ False name dependency; eliminated by Register Renaming.
  - **WAW:** Output dependency $\implies$ False name dependency; eliminated by Register Renaming.
- **Superscalar Concept:**
  \[
  \boxed{\text{Multiple redundant execution units} \longrightarrow \text{Fetch/Decode/Execute multiple instructions per cycle}}
  \]
- **Simultaneous Multithreading (SMT):**
  \[
  \boxed{\text{1 Physical Core} = \text{2+ Logical Cores (Replicated Regs/PC + Shared ALUs/Caches)}}
  \]
- **Common Exam Trap:**
  - *Trap:* Thinking Hyper-Threading (SMT) doubles CPU performance like adding physical cores.
  - *Reality:* SMT only yields a $\approx 20\%\text{--}30\%$ gain in general code, and can actually **degrade** performance in memory-intensive HPC simulations due to L1 cache thrashing.

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 02.03, verify that you can independently:
1. State Bernstein's three conditions and evaluate assembly instruction streams for concurrency.
2. Explain how out-of-order superscalar execution uses Reorder Buffers and Register Renaming to resolve false hazards.
3. Explain why SMT was invented and draw the SMT execution slot filling diagram.
4. Contrast a physical core with a logical thread context in terms of silicon area, shared resources, and expected speedup.
5. Explain the ILP Wall and state why supercomputing clusters frequently disable SMT in BIOS.
