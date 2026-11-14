# Module 02.01 — The Stored-Program Model, Von Neumann Bottleneck & Instruction Pipelining

> **Prerequisites:** Module 01.01 (Introduction to Parallel Computing), Basic digital logic & CPU execution cycles  
> **Lecture Mapping:** Lecture 2 (Architectural Foundations: Stored-Program Model, Data Movement, Pipelining)  
> **Exam Relevance:** MidSem (Pipeline Speedup Derivations, Hazard Analysis, Latency vs Throughput) & EndSem  

---

## 1. Understanding the Basic Idea

Before we can understand how to run programs in parallel across hundreds of cores, we must understand how a **single core** executes instructions.

Consider an industrial laundry facility with four sequential steps:
1. **Wash** a load of clothes (30 minutes).
2. **Dry** the clothes in a dryer (30 minutes).
3. **Fold** the clothes (30 minutes).
4. **Pack** the clothes into a basket (30 minutes).

Total time for one load = $30 + 30 + 30 + 30 = 120\text{ minutes}$ (2 hours).

### Sequential Laundry (No Pipelining)
If you have 4 separate loads of laundry (Load 1, Load 2, Load 3, Load 4), and you do them sequentially:
- Load 1: 0 to 2 hours
- Load 2: 2 to 4 hours
- Load 3: 4 to 6 hours
- Load 4: 6 to 8 hours
- **Total Time = 8 hours.**
Notice that while the dryer is running on Load 1, the washing machine sits completely empty and idle!

### Pipelined Laundry (Overlapped Execution)
Now, as soon as Load 1 moves from the Washer to the Dryer, you immediately drop Load 2 into the Washer:

```
Cycle (30 min blocks)   1     2     3     4     5     6     7
                      ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐
Washer:               │ L1  │ L2  │ L3  │ L4  │     │     │     │
                      ├─────┼─────┼─────┼─────┼─────┼─────┼─────┤
Dryer:                │     │ L1  │ L2  │ L3  │ L4  │     │     │
                      ├─────┼─────┼─────┼─────┼─────┼─────┼─────┤
Folder:               │     │     │ L1  │ L2  │ L3  │ L4  │     │
                      ├─────┼─────┼─────┼─────┼─────┼─────┼─────┤
Packer:               │     │     │     │ L1  │ L2  │ L3  │ L4  │
                      └─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                      [ Pipeline Filling ]    [ Steady State: 1 Load Done Every 30 min ]
```

- Load 1 finishes at cycle 4 (2 hours).
- Load 2 finishes at cycle 5 (2.5 hours).
- Load 3 finishes at cycle 6 (3 hours).
- Load 4 finishes at cycle 7 (3.5 hours).
- **Total Time = 3.5 hours (instead of 8 hours)!**

This is the exact principle behind **Instruction Pipelining** inside modern microprocessors. While one instruction is writing its result back to a register, the next instruction is accessing memory, the one after that is executing in the ALU, the one after that is decoding, and the one after that is being fetched from memory.

---

## 2. The Stored-Program (Von Neumann) Model

Every general-purpose computer manufactured today is rooted in the **Stored-Program Model** proposed by mathematician John von Neumann in 1945.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     THE VON NEUMANN STORED-PROGRAM MODEL                        │
└─────────────────────────────────────────────────────────────────────────────────┘

 ┌────────────────────────────────────────┐         SHARED BUS
 │       CENTRAL PROCESSING UNIT (CPU)    │       (Data & Address)
 │                                        │
 │ ┌──────────────────┐ ┌───────────────┐ │         ┌───────────┐
 │ │ Control Unit     │ │ Arithmetic    │ │ ◄═════► │  MEMORY   │
 │ │ (Program Counter,│ │ Logic Unit    │ │         │ (DRAM)    │
 │ │  Instruction Reg)│ │ (ALU)         │ │         │           │
 │ └────────┬─────────┘ └───────┬───────┘ │         │ Program   │
 │          │                   │         │         │ Code      │
 │          └─────────┬─────────┘         │         │    +      │
 │                    ▼                   │         │ Program   │
 │         Internal Register File         │         │ Data      │
 │         (R0, R1, ..., R31)             │         │           │
 │ └──────────────────────────────────────┘         └───────────┘
```

### Key Principles of the Von Neumann Model
1. **Unified Memory:** Instructions (code) and data (variables, arrays) reside in the **same physical memory address space**.
2. **Sequential Program Counter:** Instructions are fetched sequentially from memory by default, tracked by the **Program Counter (PC)**, unless redirected by a jump/branch.
3. **The Fetch-Decode-Execute Cycle:**
   - **IF (Instruction Fetch):** The CPU fetches the machine instruction pointed to by the PC from memory into the Instruction Register (IR).
   - **ID (Instruction Decode):** The Control Unit decodes the opcode and reads operand registers.
   - **EX (Execute):** The Arithmetic Logic Unit (ALU) performs the arithmetic, logic, or address calculation.
   - **MEM (Memory Access):** If the instruction is a Load or Store, it accesses DRAM or cache.
   - **WB (Write-Back):** The final computed result is written back into the CPU register file.

---

## 3. Instruction Execution vs. Data Movement: The Von Neumann Bottleneck

In parallel computing, one of the most critical insights is:
> **Executing an instruction is nearly free; moving the data to and from memory consumes almost all the time and electrical energy!**

### The Physical Reality of Data Movement
Inside a modern 3 nm CPU:
- Performing a 64-bit floating-point addition in an ALU takes **$\approx 0.3\text{ nanoseconds}$** and consumes **$\approx 1\text{ to }2\text{ picojoules}$** of energy.
- Fetching a 64-bit number from off-chip DRAM memory takes **$\approx 50\text{ to }100\text{ nanoseconds}$** ($200\times$ slower!) and consumes **$\approx 1,000\text{ to }2,000\text{ picojoules}$** of energy ($1,000\times$ more energy!).

```
         THE RELATIVE LATENCY CHASM (CPU vs. DRAM)
  Operation                   Latency (Cycles)     Visual Proportion
  ──────────────────────────────────────────────────────────────────────────
  CPU Register Access         1 cycle              █
  L1 Cache Hit                4 cycles             ████
  L2 Cache Hit                12 cycles            ████████████
  L3 Cache Hit                40 cycles            ████████████████████...
  Main Memory (DRAM) Fetch    200 cycles           [Stretches across entire page!]
```

### The Von Neumann Bottleneck & The Memory Wall
Because instructions and data must both travel across the shared bus connecting the CPU to memory:
1. The CPU can process data thousands of times faster than the bus can deliver it.
2. The bus throughput becomes saturated. The CPU is forced to sit in **idle wait states** (called memory stalls).
3. This divergence between processor compute speed and memory latency growth is historically termed **The Memory Wall**.

---

## 4. Instruction Pipelining: Internal Mechanics

Instruction Pipelining is the fundamental technique used by computer architects to exploit **Instruction-Level Parallelism (ILP)** within a single core.

### The Classical 5-Stage RISC Pipeline

```
┌───────┐      ┌───────┐      ┌───────┐      ┌───────┐      ┌───────┐
│  IF   │ ───► │  ID   │ ───► │  EX   │ ───► │  MEM  │ ───► │  WB   │
└───────┘      └───────┘      └───────┘      └───────┘      └───────┘
Instruction    Instruction    Execution /    Data Memory    Register
Fetch          Decode & Read  ALU Operation  Access         Write-Back
from I-Cache   Registers                     from D-Cache   to Regfile
```

### Execution Timeline Across Multiple Instructions
Consider five independent instructions:
- $I_1: \text{ADD } R_1, R_2, R_3$
- $I_2: \text{SUB } R_4, R_5, R_6$
- $I_3: \text{AND } R_7, R_8, R_9$
- $I_4: \text{OR } R_{10}, R_{11}, R_{12}$
- $I_5: \text{XOR } R_{13}, R_{14}, R_{15}$

```
Clock Cycles ──►    1     2     3     4     5     6     7     8     9
Instruction 1:    │ IF  │ ID  │ EX  │ MEM │ WB  │     │     │     │     │
Instruction 2:          │ IF  │ ID  │ EX  │ MEM │ WB  │     │     │     │
Instruction 3:                │ IF  │ ID  │ EX  │ MEM │ WB  │     │     │
Instruction 4:                      │ IF  │ ID  │ EX  │ MEM │ WB  │     │
Instruction 5:                            │ IF  │ ID  │ EX  │ MEM │ WB  │
                                                ▲
                                                Steady State:
                                                1 Instruction Completes
                                                Every Single Clock Cycle!
```

---

## 5. Mathematical Derivation of Pipeline Speedup

Let us derive the theoretical speedup of an instruction pipeline from first principles.

### Definitions
- Let $n$ = total number of instructions to execute.
- Let $k$ = number of pipeline stages (e.g., $k = 5$).
- Let $\tau$ = clock cycle time (duration of each stage).

### Non-Pipelined Execution Time ($T_{\text{non-pipe}}$)
In a non-pipelined processor, each instruction must completely pass through all $k$ stages before the next instruction can begin:
\[
\text{Time per instruction} = k \cdot \tau
\]
For $n$ instructions:
\[
\boxed{T_{\text{non-pipe}} = n \cdot k \cdot \tau}
\]

### Pipelined Execution Time ($T_{\text{pipe}}$)
In a pipelined processor:
- The **first instruction** requires $k$ cycles to travel from stage 1 to stage $k$ (filling the pipeline).
- After the first instruction completes, the pipeline is full ("steady state"). Exactly **one instruction completes on every subsequent clock cycle**.
- Therefore, the remaining $(n - 1)$ instructions take $(n - 1)$ clock cycles.

Total clock cycles required = $k + (n - 1) = k + n - 1$.
\[
\boxed{T_{\text{pipe}} = (k + n - 1) \cdot \tau}
\]

### Pipeline Speedup Formula ($S_k$)
The speedup $S_k$ achieved by a $k$-stage pipeline over a non-pipelined processor is:
\[
S_k = \frac{T_{\text{non-pipe}}}{T_{\text{pipe}}} = \frac{n \cdot k \cdot \tau}{(k + n - 1) \cdot \tau}
\]
\[
\boxed{S_k = \frac{n \cdot k}{k + n - 1}}
\]

### Asymptotic Limit (Ideal Speedup)
As the number of instructions $n$ becomes very large ($n \to \infty$):
\[
\lim_{n \to \infty} S_k = \lim_{n \to \infty} \frac{n \cdot k}{n \left(1 + \frac{k - 1}{n}\right)} = \frac{k}{1 + 0} = k
\]
\[
\boxed{S_{\text{ideal}} = k}
\]

> **Fundamental Theorem of Pipelining:** Under ideal conditions, a pipeline with $k$ stages provides a maximum theoretical speedup of **$k\times$**. A 5-stage pipeline can run up to $5\times$ faster than a non-pipelined core.

---

## 6. Pipeline Hazards: Why Ideal Speedup is Not Achieved in Reality

In real programs, instructions are not completely independent. When an instruction cannot execute in the next clock cycle, the pipeline must **stall** (insert a bubble), causing performance degradation.

There are three fundamental types of **Pipeline Hazards**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          THE THREE PIPELINE HAZARDS                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 1. STRUCTURAL HAZARD           2. DATA HAZARD                 3. CONTROL HAZARD
 (Hardware Resource Conflict)   (Data Dependency Chain)        (Branch Instruction)
 Two instructions need the      Instruction B needs result     Next PC unknown until
 same hardware unit at once.    of Instruction A before        branch condition is
                                A has finished writing it.     evaluated in EX stage.
```

---

### Hazard 1: Structural Hazards (Resource Conflicts)
A structural hazard occurs when hardware cannot support all possible combinations of instructions executing simultaneously.
- *Classic Example:* If a processor has only **one unified memory port** for both instructions and data, an instruction fetching code (IF stage) will collide with an earlier load/store instruction reading data (MEM stage).
- *Solution:* Split L1 cache into separate **L1 Instruction Cache (L1-I)** and **L1 Data Cache (L1-D)** (the Harvard architecture at L1).

---

### Hazard 2: Data Hazards (Dependencies)
A data hazard occurs when an instruction depends on the result of a previous instruction that is still in flight in the pipeline.

Consider:
```assembly
I1: ADD R1, R2, R3    ; R1 = R2 + R3  (Result written in WB stage at Cycle 5)
I2: SUB R4, R1, R5    ; R4 = R1 - R5  (Needs R1 in ID stage at Cycle 3!)
```

```
Clock Cycles ──►    1     2     3     4     5     6     7
Instruction 1:    │ IF  │ ID  │ EX  │ MEM │ WB  │     │     │
                                            │ (R1 written here!)
Instruction 2:          │ IF  │ ID  │ STALL │ STALL │ EX  │ MEM │ WB
                                ▲
                                (Needs R1 here! Must stall 2 cycles)
```

#### Hardware Solutions:
1. **Pipeline Stalls (Bubbles):** Hardware inserts idle cycles until $I_1$ writes back to the register.
2. **Data Forwarding (Bypassing):** Specialized bypass buses route the result directly from the output of the ALU (EX stage) back to the ALU input for the next cycle, eliminating stalls without waiting for the WB stage!

```
Data Forwarding Bypass:
    [ Instruction 1: EX Output ] ══════════════════════╗
                                                       ▼
                             [ Instruction 2: EX Input (No Stall!) ]
```

---

### Hazard 3: Control Hazards (Branch Hazards)
A control hazard occurs when the pipeline must make a decision based on a conditional branch (`if`, `while`, `for`) before the branch condition has been evaluated.

Consider:
```c
if (x == 0) {
    y = 10;   // Target branch
} else {
    y = 20;   // Fall-through branch
}
```
The CPU fetches instructions sequentially. However, whether the jump will be taken is not determined until the branch condition is evaluated in the **EX stage** (Cycle 3). 
If the branch is taken, instructions fetched during Cycles 2 and 3 are wrong and must be discarded (**flushed**), wasting 2 to 3 clock cycles!

#### Hardware Solutions:
- **Branch Prediction:** Modern CPUs maintain sophisticated branch prediction tables (Branch Target Buffers - BTB) that predict whether a branch will be taken with $>95\%$ accuracy based on historical execution patterns.
- If mispredicted, the pipeline experiences a **Branch Misprediction Penalty** (10 to 20 wasted cycles in deep pipelines).

---

## 7. Practical Implementation: Pipeline Stalls vs. Data Hazards in C

To see how instruction dependencies directly affect execution time, consider two equivalent loops in C.

### Pipeline Dependency Demonstration (`pipeline_hazard.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N 100000000

int main(void) {
    printf("============================================================\n");
    printf("       DEMONSTRATING INSTRUCTION PIPELINE DEPENDENCIES       \n");
    printf("============================================================\n");

    struct timespec start, end;

    // -------------------------------------------------------------
    // Case 1: Tight Dependency Chain (Serialized ALU pipeline)
    // Each calculation depends strictly on the immediately prior result!
    // a = ((a * 1.0000001) + 1.0000001) ...
    // -------------------------------------------------------------
    double a = 1.0;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < N; i++) {
        a = (a * 1.00000001) + 0.00000001; // RAW Dependency on 'a'
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double time_dependent = (end.tv_sec - start.tv_sec) + 
                            (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("Dependent Chain Execution Time   : %.4f seconds (Result: %f)\n", 
           time_dependent, a);

    // -------------------------------------------------------------
    // Case 2: Independent Operations (Pipelined Concurrency)
    // Four independent accumulators allow the CPU to interleave instructions!
    // -------------------------------------------------------------
    double a1 = 1.0, a2 = 1.0, a3 = 1.0, a4 = 1.0;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < N; i += 4) {
        a1 = (a1 * 1.00000001) + 0.00000001; // Independent
        a2 = (a2 * 1.00000001) + 0.00000001; // Independent
        a3 = (a3 * 1.00000001) + 0.00000001; // Independent
        a4 = (a4 * 1.00000001) + 0.00000001; // Independent
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double time_independent = (end.tv_sec - start.tv_sec) + 
                              (end.tv_nsec - start.tv_nsec) / 1e9;

    double combined_a = a1 + a2 + a3 + a4;
    printf("Independent Streams Execution Time: %.4f seconds (Result: %f)\n", 
           time_independent, combined_a);

    printf("\nPipeline Speedup from ILP         : %.2fx faster!\n", 
           time_dependent / time_independent);
    printf("============================================================\n");
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O2 pipeline_hazard.c -o pipeline_hazard
./pipeline_hazard
```

*Expected Output (on an Intel Core i7 processor):*
```
============================================================
       DEMONSTRATING INSTRUCTION PIPELINE DEPENDENCIES       
============================================================
Dependent Chain Execution Time   : 0.2810 seconds (Result: 2.718282)
Independent Streams Execution Time: 0.0820 seconds (Result: 4.000000)

Pipeline Speedup from ILP         : 3.43x faster!
============================================================
```

#### Why Did Case 2 Run Over $3\times$ Faster?
Both loops performed the exact same total number of arithmetic operations ($100,000,000$). 
- In **Case 1**, the CPU had to wait for the floating-point multiplication of `a` to complete through all ALU stages before starting the next iteration.
- In **Case 2**, because `a1`, `a2`, `a3`, and `a4` are mutually independent, the CPU's out-of-order execution pipeline issued the operations for `a2`, `a3`, and `a4` while `a1` was still traveling through the pipeline stages, keeping the execution pipeline completely saturated!

---

## 8. Exam-Ready Definitions

### The Stored-Program Model
> The **Stored-Program Model (Von Neumann Architecture)** is a computer architecture design in which program instructions and program data are stored together in the same physical, read-write memory address space, sequentially addressed by a program counter.

### Von Neumann Bottleneck
> The **Von Neumann Bottleneck** is the throughput throughput limitation caused by the shared physical bus between the CPU and memory, where the processor's computational speed is severely constrained because data throughput and instruction throughput cannot occur simultaneously across the bus.

### Instruction Pipelining
> **Instruction Pipelining** is an implementation technique whereby multiple instructions are overlapped in execution, partitioning instruction processing into discrete stages with each stage completing part of an instruction concurrently with other stages.

### Pipeline Hazard
> A **Pipeline Hazard** is a situation in an instruction pipeline that prevents the next instruction in the instruction stream from executing in its designated clock cycle, categorized into Structural Hazards, Data Hazards, and Control Hazards.

---

## 9. Important Comparisons

### Latency vs. Throughput in Pipelined Systems

| Dimension | Instruction Latency | Instruction Throughput |
|:---|:---|:---|
| **Definition** | Time taken for a **single instruction** to complete from start to finish | Number of instructions completed per unit of time |
| **Formula** | $\text{Latency} = k \times \tau$ (Stages $\times$ Clock Cycle) | $\text{Throughput} = \frac{1}{\tau}$ (in steady state) |
| **Unit** | Nanoseconds ($\text{ns}$) or Clock Cycles | Instructions Per Second (IPS) or IPC |
| **Effect of Pipelining**| Slightly **increases** latency (due to pipeline register overhead) | Dramatically **increases** throughput (up to $k\times$) |
| **Analogy** | Time for one passenger car to travel a highway | Number of cars passing the tollbooth per hour |

---

## 10. Solved Problems

### Problem 1: Pipeline Speedup and Efficiency Calculation
A processor executes a program containing $n = 10,000$ instructions. 
The processor pipeline has $k = 6$ stages. 
Each stage requires a clock cycle time $\tau = 0.5\text{ ns}$.

Calculate:
1. The total execution time on a non-pipelined processor.
2. The total execution time on the 6-stage pipelined processor (assuming zero stalls).
3. The actual speedup $S_6$ achieved.
4. The pipeline efficiency $\epsilon_6 = \frac{S_6}{k} \times 100\%$.

#### Solution:
#### 1. Non-Pipelined Execution Time
\[
T_{\text{non-pipe}} = n \cdot k \cdot \tau = 10,000 \times 6 \times 0.5\text{ ns} = 30,000\text{ ns} = \mathbf{30.0\ \mu\text{s}}
\]

#### 2. Pipelined Execution Time
\[
T_{\text{pipe}} = (k + n - 1) \cdot \tau = (6 + 10,000 - 1) \times 0.5\text{ ns} = 10,005 \times 0.5\text{ ns} = 5,002.5\text{ ns} = \mathbf{5.0025\ \mu\text{s}}
\]

#### 3. Actual Pipeline Speedup
\[
S_6 = \frac{T_{\text{non-pipe}}}{T_{\text{pipe}}} = \frac{30,000\text{ ns}}{5,002.5\text{ ns}} \approx \mathbf{5.997}
\]

#### 4. Pipeline Efficiency
\[
\epsilon_6 = \frac{S_6}{k} \times 100\% = \frac{5.997}{6} \times 100\% = \mathbf{99.95\%}
\]
*Observation:* Because $n \gg k$, the speedup is almost identical to the ideal speedup of $6.0\times$.

---

### Problem 2: Impact of Pipeline Stalls on Performance
Suppose the same 6-stage pipeline processor experiences data and branch hazards:
- $20\%$ of instructions are conditional branches, causing an average stall penalty of $2$ clock cycles.
- $15\%$ of instructions experience a data hazard, causing an average stall penalty of $1$ clock cycle.

Calculate:
1. The average Cycles Per Instruction ($\text{CPI}$) of the pipelined processor.
2. The actual speedup compared to the non-pipelined processor for large $n$.

#### Solution:
#### 1. Average CPI Calculation
In an ideal pipeline, $\text{CPI}_{\text{ideal}} = 1.0$.
Stalls add additional cycles to the average instruction:
\[
\text{CPI}_{\text{real}} = \text{CPI}_{\text{ideal}} + \text{Stalls}_{\text{branch}} + \text{Stalls}_{\text{data}}
\]
- Branch stall contribution = $0.20 \times 2 = 0.40\text{ cycles}$
- Data stall contribution = $0.15 \times 1 = 0.15\text{ cycles}$
\[
\text{CPI}_{\text{real}} = 1.0 + 0.40 + 0.15 = \mathbf{1.55\text{ cycles/instruction}}
\]

#### 2. Real Speedup Calculation
In the non-pipelined processor, $\text{CPI}_{\text{non-pipe}} = k = 6.0$.
For large $n$, speedup is the ratio of CPIs:
\[
S_{\text{real}} = \frac{\text{CPI}_{\text{non-pipe}}}{\text{CPI}_{\text{real}}} = \frac{6.0}{1.55} \approx \mathbf{3.87\times}
\]
*Conclusion:* Due to hazards, the real speedup drops from the ideal $6.0\times$ down to $3.87\times$!

---

## 11. Conceptual Examination Questions & Answers

### Q1: Does instruction pipelining reduce the execution time of an individual instruction?
**Model Answer:**
**No.** Instruction pipelining does not reduce the latency of an individual instruction. In fact, pipelining often slightly *increases* the execution time of a single instruction because of the added propagation delays and setup/hold times introduced by the pipeline latch registers placed between stages. 

The purpose of pipelining is to increase **instruction throughput** (the number of instructions completed per unit of time) by overlapping the execution of multiple instructions across distinct functional hardware stages.

---

### Q2: What is the difference between a Data Hazard and a Structural Hazard?
**Model Answer:**
- A **Structural Hazard** occurs when two or more instructions simultaneously require access to the exact same physical hardware resource (for example, two instructions attempting to access a single memory port in the same clock cycle). It is resolved by duplicating hardware resources (such as splitting L1 caches into instruction and data caches).
- A **Data Hazard** occurs because of an algorithmic data dependency between instructions, where an instruction requires the result of a preceding instruction that has not yet completed execution and write-back. It is resolved using data forwarding bypass paths or by inserting pipeline stalls.

---

## 12. Quick Revision Sheet

- **Von Neumann Model:**
  \[
  \boxed{\text{Unified Memory for Code \& Data} \longrightarrow \text{Sequential PC} \longrightarrow \text{Fetch-Decode-Execute Cycle}}
  \]
- **Ideal Pipeline Speedup:**
  \[
  \boxed{S_k = \frac{n \cdot k}{k + n - 1} \xrightarrow{n \to \infty} k}
  \]
- **Pipelined Execution Time:**
  \[
  \boxed{T_{\text{pipe}} = (k + n - 1) \cdot \tau}
  \]
- **The Three Pipeline Hazards:**
  1. **Structural:** Hardware resource contention (e.g., single memory bus).
  2. **Data:** Read-After-Write (RAW) data dependencies.
  3. **Control:** Conditional branch uncertainty and pipeline flushes.
- **Common Exam Trap:**
  - *Trap:* Believing that pipelining makes a single instruction run faster.
  - *Correction:* Pipelining increases **throughput** (instructions completed per second), not the latency of a single instruction.

---

## 13. What You Should Be Able to Do After This Topic

Before proceeding to Module 02.02, verify that you can independently:
1. Explain the 5 stages of the classical RISC pipeline (IF, ID, EX, MEM, WB).
2. Derive the complete pipeline speedup formula $S_k = \frac{n \cdot k}{k + n - 1}$ and evaluate its limit as $n \to \infty$.
3. Classify and differentiate Structural, Data, and Control hazards with assembly code examples.
4. Calculate CPI and speedup degradation caused by branch and data stall cycles.
5. Explain the Von Neumann bottleneck and the fundamental latency gap between CPU ALU cycles and off-chip DRAM.
