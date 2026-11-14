# Module 01.04 — Performance Metrics, FLOP/s & The Power-Cost Case Study

> **Prerequisites:** Module 01.01 (Introduction to Parallel Computing), Module 01.03 (Hardware Hierarchy & TOP500)  
> **Lecture Mapping:** Lecture 1 (Performance Definitions, FLOP/s Formulations, Power Wall & Datacenter Economics)  
> **Exam Relevance:** MidSem (Numerical Calculations, FLOP/s Units, Cost Modeling) & EndSem  

---

## 1. Understanding the Basic Idea

Suppose two different car engines are being compared. One salesperson says, *"My engine has 8 cylinders."* Another says, *"My car travels from City A to City B in 2 hours."* 

Which statement tells you more about true performance? The second one, because it measures **work completed per unit of time**. 

In high-performance computing, students frequently confuse **hardware specifications** (such as clock frequency in Gigahertz, or number of CPU cores) with **actual performance**. A processor clocked at $4.0\text{ GHz}$ is not necessarily twice as fast as a processor clocked at $2.0\text{ GHz}$. If the $4.0\text{ GHz}$ processor is constantly stalled waiting for data from main memory, its execution units sit completely idle.

> **Performance is not what the hardware is theoretically capable of doing at maximum clock speed; performance is the rate at which useful computational work is actually accomplished.**

To measure, compare, and optimize parallel computers, the HPC community established a rigorous mathematical vocabulary grounded in:
1. **Work ($W$)**
2. **Execution Time ($T$)**
3. **Floating-Point Operations Per Second (FLOP/s)**
4. **Energy Efficiency (FLOPS/Watt)**

---

## 2. Fundamental Performance Formulation: Work vs. Time

At the most fundamental level, performance $P$ of any computing system executing a workload is defined as:

\[
\boxed{P = \frac{W}{T}}
\]

Where:
- $W$ is the total amount of useful computational work accomplished.
- $T$ is the wall-clock execution time taken to complete that work.

```
       HIGH PERFORMANCE = MORE WORK IN LESS TIME
   Work (W)
      ▲
      │                 System A (High Performance: Steep Slope P_A)
      │                /
      │               /
      │              /      System B (Lower Performance: Flatter Slope P_B)
      │             /      /
      │            /      /
      │           /      /
      │          /      /
      └─────────┴──────┴────────────────► Execution Time (T)
               T_A    T_B
```

### The Performance Inversion Law
If we run a **fixed workload** ($W$ is constant), comparing two systems (System 1 and System 2) yields an inverse relationship with execution time:

\[
\frac{P_1}{P_2} = \frac{\frac{W}{T_1}}{\frac{W}{T_2}} = \frac{T_2}{T_1}
\]

If System 1 takes $10\text{ seconds}$ and System 2 takes $40\text{ seconds}$:
\[
\frac{P_1}{P_2} = \frac{40\text{ s}}{10\text{ s}} = 4.0
\]
System 1 is **$4\times$ faster (has $4\times$ the performance)** of System 2.

---

## 3. What is a FLOP? Understanding Floating-Point Operations

In scientific and engineering simulations, nearly all numbers are real numbers with fractional components (e.g., $3.14159$, $-0.00284$, $6.022 \times 10^{23}$). 

These are represented in computer hardware using the **IEEE 754 Floating-Point Standard**:
- **Single Precision (`float` in C):** 32 bits total (1 sign bit, 8 exponent bits, 23 mantissa/fraction bits).
- **Double Precision (`double` in C):** 64 bits total (1 sign bit, 11 exponent bits, 52 mantissa/fraction bits).

> A **FLOP** (singular: Floating-Point Operation) is a single basic arithmetic operation performed on floating-point numbers:
> - Floating-point Addition ($a + b$)
> - Floating-point Subtraction ($a - b$)
> - Floating-point Multiplication ($a \times b$)
> - Floating-point Division ($a / b$)

### The Special Case: Fused Multiply-Add (FMA)
Modern CPUs and GPUs contain specialized arithmetic units called **FMA units**. An FMA unit computes:
\[
d = (a \times b) + c
\]
in a **single clock cycle** with a single rounding step. 
Because it performs one multiplication AND one addition in one instruction, **1 FMA instruction counts as 2 FLOPs!**

---

## 4. The Scale of FLOP/s: From Mega to Exa

The standard unit of throughput in High-Performance Computing is **FLOP/s** (Floating-Point Operations Per Second). 

Notice the casing carefully:
- **FLOP:** A single arithmetic operation (noun).
- **FLOP/s** or **FLOPS:** Operations per second (rate/throughput).

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      THE SCIENTIFIC SCALE OF FLOP/s                             │
└─────────────────────────────────────────────────────────────────────────────────┘

 Prefix    Symbol    Scientific Notation    Standard Decimal Representation
 ─────────────────────────────────────────────────────────────────────────────
 Mega      MFLOP/s   10⁶ FLOP/s             1,000,000 (1 Million)
 Giga      GFLOP/s   10⁹ FLOP/s             1,000,000,000 (1 Billion)
 Tera      TFLOP/s   10¹² FLOP/s            1,000,000,000,000 (1 Trillion)
 Peta      PFLOP/s   10¹⁵ FLOP/s            1,000,000,000,000,000 (1 Quadrillion)
 Exa       EFLOP/s   10¹⁸ FLOP/s            1,000,000,000,000,000,000 (1 Quintillion)
 ─────────────────────────────────────────────────────────────────────────────
 Future Horizons:
 Zetta     ZFLOP/s   10²¹ FLOP/s            1 Sextillion FLOP/s
 Yotta     YFLOP/s   10²⁴ FLOP/s            1 Septillion FLOP/s
```

### Physical Perspective: What Does an ExaFLOP Represent?
An **Exascale** supercomputer operates at $\ge 1.0 \times 10^{18}\text{ FLOP/s}$.
- If every single person on Earth (8 billion humans) each held a pocket calculator and performed **one arithmetic calculation every single second**, non-stop without sleeping...
- It would take the entire human race **4 straight years** to complete the number of calculations that an Exascale supercomputer (like *Frontier*) completes in **just one second**!

---

## 5. Peak Theoretical FLOP/s vs. Sustained FLOP/s

One of the most frequent examination questions concerns the distinction between theoretical peak throughput and sustained throughput.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                   PEAK vs. SUSTAINED PERFORMANCE GAP                            │
└─────────────────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────────────────────┐
 │  THEORETICAL PEAK PERFORMANCE (R_peak)                                      │
 │  Paper calculation based on hardware specs:                                │
 │  R_peak = Sockets × Cores/Socket × Clock Frequency × FLOPs/Cycle/Core       │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
                         LOSSES TO REAL-WORLD REALITIES:
                         ├── Memory bandwidth bottlenecks (waiting for DRAM)
                         ├── Cache misses and memory latency
                         ├── Non-vectorizable scalar code
                         ├── Thread synchronization & lock contention
                         └── Network latency & halo communication
                                        │
                                        ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │  SUSTAINED PERFORMANCE (R_max)                                              │
 │  Actual measured throughput during real program execution:                  │
 │  R_max = Total Arithmetic Operations / Measured Wall-Clock Time             │
 └─────────────────────────────────────────────────────────────────────────────┘
```

### Derivation of Theoretical Peak Performance ($R_{\text{peak}}$)
To calculate $R_{\text{peak}}$ for any modern microprocessor or cluster:

\[
\boxed{R_{\text{peak}} = N_{\text{sockets}} \times \left(\frac{N_{\text{cores}}}{\text{socket}}\right) \times f_{\text{clock}} \times \left(\frac{\text{FLOPs}}{\text{cycle/core}}\right)}
\]

Where:
- $N_{\text{sockets}}$ is the number of physical CPU chips.
- $N_{\text{cores}}$ is the number of physical cores per socket.
- $f_{\text{clock}}$ is the operating clock frequency in Hertz ($\text{Hz}$).
- $\text{FLOPs/cycle/core}$ is determined by the CPU's SIMD vector execution width:
  \[
  \text{FLOPs/cycle/core} = N_{\text{FMA units}} \times \left(\frac{\text{Vector Register Width in Bits}}{\text{Operand Bit-Width}}\right) \times 2\ (\text{for FMA})
  \]

*Example:* An Intel Xeon core with **two 512-bit AVX-512 FMA units** executing 64-bit double precision:
\[
\text{FLOPs/cycle/core} = 2\text{ FMA units} \times \left(\frac{512\text{ bits}}{64\text{ bits}}\right) \times 2\text{ (multiply + add)} = 2 \times 8 \times 2 = \mathbf{32\text{ FLOPs/cycle/core}}
\]

---

## 6. The Power-Cost Case Study: Supercomputing Economics

High-Performance Computing is fundamentally constrained by electricity bills and cooling capacity. Building a supercomputer is not merely a capital purchase; it is a long-term commitment to massive utility costs.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│               TOTAL COST OF OWNERSHIP (TCO) FOR AN HPC FACILITY                 │
└─────────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────┐
                    │      HPC TOTAL COST OF OWNERSHIP        │
                    └────────────────────┬────────────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
   CAPITAL EXPENDITURE (CapEx)                     OPERATIONAL EXPENDITURE (OpEx)
   [ One-time upfront cost ]                       [ Recurring continuous cost ]
   ├── Server nodes & processors                   ├── Electrical power for compute
   ├── Memory (DRAM & HBM)                         ├── Electrical power for cooling (Chillers)
   ├── High-speed network switches & cables        ├── Datacenter facility maintenance
   └── High-speed parallel storage (Lustre)        └── System administration staff
```

### Power Usage Effectiveness (PUE)
Datacenters do not just power processors; they must also power giant water chillers, air handlers, and pumps to remove the heat.
The metric used worldwide is **Power Usage Effectiveness (PUE)**:

\[
\boxed{\text{PUE} = \frac{\text{Total Facility Power}}{\text{IT Equipment Power}}}
\]

- An ideal datacenter has a $\text{PUE} = 1.0$ (100% of electricity goes to compute chips).
- Typical commodity datacenters operate at $\text{PUE} \approx 1.5\text{ to }1.8$ (for every $1\text{ MW}$ of compute, an extra $0.5\text{ to }0.8\text{ MW}$ is spent on cooling!).
- Modern state-of-the-art HPC facilities with direct warm-water liquid cooling achieve $\text{PUE} \approx 1.05\text{ to }1.15$.

### Total Electricity Cost Formula
The monthly cost to power an HPC facility is formulated as:

\[
\boxed{\text{Monthly Electricity Cost} = P_{\text{IT}} \times \text{PUE} \times 24\frac{\text{hours}}{\text{day}} \times 30\text{ days} \times \text{Tariff Rate}\ (\$/\text{kWh})}
\]

Where $P_{\text{IT}}$ is the average electrical power of the supercomputer in Kilowatts ($\text{kW}$).

---

## 7. Practical Implementation: Benchmarking Sustained FLOP/s in C

Here is a complete, compilable C benchmark that measures the **actual sustained GFLOP/s** of your machine's CPU by performing a known number of floating-point operations.

### Sustained FLOPS Benchmark (`flops_benchmark.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define VECTOR_LEN 10000000
#define NUM_ITERATIONS 50

int main(void) {
    printf("============================================================\n");
    printf("       MEASURING SUSTAINED FLOATING-POINT THROUGHPUT        \n");
    printf("============================================================\n");

    // Allocate memory for three double-precision arrays (~240 MB)
    double *a = (double *)malloc(VECTOR_LEN * sizeof(double));
    double *b = (double *)malloc(VECTOR_LEN * sizeof(double));
    double *c = (double *)malloc(VECTOR_LEN * sizeof(double));

    if (!a || !b || !c) {
        fprintf(stderr, "Error: Memory allocation failed!\n");
        return 1;
    }

    // Initialize arrays
    #pragma omp parallel for
    for (int i = 0; i < VECTOR_LEN; i++) {
        a[i] = 1.0001;
        b[i] = 2.0002;
        c[i] = 0.5003;
    }

    double scalar = 1.00005;

    // Warmup run (prime the caches and wake up CPU power states)
    #pragma omp parallel for
    for (int i = 0; i < VECTOR_LEN; i++) {
        a[i] = a[i] * scalar + b[i];
    }

    printf("Executing benchmark: %d iterations over %d elements...\n", 
           NUM_ITERATIONS, VECTOR_LEN);

    double start_time = omp_get_wtime();

    // Benchmark loop: Triad operation a[i] = b[i] + scalar * c[i]
    // Each iteration performs 1 multiply and 1 add = 2 FLOPs per element
    for (int iter = 0; iter < NUM_ITERATIONS; iter++) {
        #pragma omp parallel for
        for (int i = 0; i < VECTOR_LEN; i++) {
            a[i] = b[i] + scalar * c[i];
        }
    }

    double end_time = omp_get_wtime();
    double elapsed_time = end_time - start_time;

    // Calculate total operations
    // 2 FLOPs per inner iteration * VECTOR_LEN * NUM_ITERATIONS
    double total_flops = 2.0 * (double)VECTOR_LEN * (double)NUM_ITERATIONS;
    double sustained_gflops = (total_flops / elapsed_time) / 1e9;

    printf("\nExecution Summary:\n");
    printf("  Elapsed Wall Time    : %.4f seconds\n", elapsed_time);
    printf("  Total FLOPs Executed : %.2e\n", total_flops);
    printf("  Sustained Throughput : %.2f GFLOP/s\n", sustained_gflops);
    printf("============================================================\n");

    free(a);
    free(b);
    free(c);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp -march=native flops_benchmark.c -o flops_benchmark
./flops_benchmark
```

*Expected Output (on a 6-core Intel i7 workstation):*
```
============================================================
       MEASURING SUSTAINED FLOATING-POINT THROUGHPUT        
============================================================
Executing benchmark: 50 iterations over 10000000 elements...

Execution Summary:
  Elapsed Wall Time    : 0.1245 seconds
  Total FLOPs Executed : 1.00e+09
  Sustained Throughput : 8.03 GFLOP/s
============================================================
```

---

## 8. Important Comparisons

### Peak Performance ($R_{\text{peak}}$) vs. Sustained Performance ($R_{\max}$)

| Aspect | Theoretical Peak Performance ($R_{\text{peak}}$) | Sustained Performance ($R_{\max}$) |
|:---|:---|:---|
| **Nature of Metric** | Theoretical paper calculation | Empirical, experimentally measured value |
| **How It Is Obtained**| Formula: $\text{Cores} \times \text{Clock} \times \text{FLOPs/Cycle}$ | Benchmarking application run: $\text{Work} / \text{Time}$ |
| **Memory Bottlenecks**| Completely ignored (assumes 100% register hit) | Fully exposed (reflects DRAM & cache latency stalls)|
| **Communication Impact**| Completely ignored | Fully exposed (reflects network synchronization delays)|
| **Standard Role** | Marketing specification by chip vendors | Honest measure of real-world application throughput |
| **Typical Value** | High ($100\%$ by definition) | $60\%\text{--}80\%$ for LINPACK, $<5\%$ for Sparse codes|

---

## 9. Exam-Ready Definitions

### FLOP/s (Floating-Point Operations Per Second)
> **FLOP/s** is the standard international metric of computational throughput in high-performance computing, representing the total number of IEEE 754 floating-point arithmetic operations (additions, subtractions, multiplications, divisions) successfully completed per second of wall-clock time.

### Fused Multiply-Add (FMA)
> A **Fused Multiply-Add (FMA)** is a hardware execution unit capability that computes the operation $(a \times b) + c$ in a single clock cycle with a single rounding step, counting as two distinct floating-point operations (one multiply, one add).

### Power Usage Effectiveness (PUE)
> **Power Usage Effectiveness (PUE)** is an efficiency metric for datacenters defined as the ratio of total electrical energy entering the datacenter facility to the electrical energy consumed directly by the computing IT equipment.

### Energy Efficiency (in HPC)
> In supercomputing, **Energy Efficiency** is the ratio of sustained computational throughput to electrical power consumed, typically quantified as $\text{GFLOP/s per Watt}$ or $\text{MFLOP/s per Watt}$.

---

## 10. Solved Problems

### Problem 1: Full-System $R_{\text{peak}}$ Calculation
A university installs a small research cluster with the following specifications:
- **Total Compute Nodes:** $32$ nodes.
- **CPUs per Node:** $2$ sockets.
- **Cores per CPU:** $64$ physical cores.
- **Base Clock Frequency:** $2.0\text{ GHz}$.
- **SIMD Capability:** Each core has two 256-bit AVX2 FMA execution units.
  (Note: 256-bit registers hold four 64-bit double-precision numbers. Each FMA unit performs 4 multiplies and 4 adds per cycle = 8 FLOPs/cycle. Two units yield $16\text{ FLOPs/cycle/core}$).

Calculate:
1. The theoretical peak performance of one core in GFLOP/s.
2. The theoretical peak performance of one compute node in TFLOP/s.
3. The total cluster peak performance in TFLOP/s and PFLOP/s.

#### Solution:
#### 1. Peak Performance per Core
\[
R_{\text{core}} = f_{\text{clock}} \times \left(\frac{\text{FLOPs}}{\text{cycle/core}}\right) = 2.0 \times 10^9\text{ cycles/s} \times 16\text{ FLOPs/cycle}
\]
\[
R_{\text{core}} = 32.0 \times 10^9\text{ FLOP/s} = \mathbf{32.0\text{ GFLOP/s}}
\]

#### 2. Peak Performance per Node
- Cores per node:
  \[
  \text{Cores/Node} = 2\text{ sockets} \times 64\text{ cores/socket} = 128\text{ cores}
  \]
- Node peak throughput:
  \[
  R_{\text{node}} = 128\text{ cores} \times 32.0\text{ GFLOP/s} = 4,096\text{ GFLOP/s} = \mathbf{4.096\text{ TFLOP/s}}
  \]

#### 3. Total Cluster Peak Performance
- Cluster throughput:
  \[
  R_{\text{cluster}} = 32\text{ nodes} \times 4.096\text{ TFLOP/s} = \mathbf{131.072\text{ TFLOP/s}}
  \]
- Convert to PFLOP/s:
  \[
  131.072\text{ TFLOP/s} = \frac{131.072}{1000} = \mathbf{0.1311\text{ PFLOP/s}}
  \]

---

### Problem 2: Complete Datacenter Power & Utility Cost Case Study
A supercomputing center operates a $10\text{ PFLOP/s}$ cluster that draws $2.5\text{ Megawatts}$ of electrical power for its IT compute racks.
The datacenter facility has a **Power Usage Effectiveness (PUE) of $1.4$**.
The local electrical utility charges **$\$0.10\text{ per kWh}$**.

Calculate:
1. The total power drawn by the entire facility (including cooling and power distribution).
2. The total electrical energy consumed by the entire facility over a full 365-day year (in $\text{kWh}$).
3. The annual electricity bill to operate the facility.
4. The supercomputer's energy efficiency in GFLOP/s per Watt.

#### Solution:
#### 1. Total Facility Power Drawn
\[
\text{Total Power} = P_{\text{IT}} \times \text{PUE} = 2.5\text{ MW} \times 1.4 = \mathbf{3.5\text{ Megawatts}} = 3,500\text{ kW}
\]

#### 2. Annual Energy Consumed
- Hours in a year:
  \[
  \text{Hours} = 365\text{ days} \times 24\frac{\text{hours}}{\text{day}} = 8,760\text{ hours}
  \]
- Energy in $\text{kWh}$:
  \[
  \text{Energy} = 3,500\text{ kW} \times 8,760\text{ hours} = \mathbf{30,660,000\text{ kWh}}
  \]

#### 3. Annual Electricity Cost
\[
\text{Annual Cost} = 30,660,000\text{ kWh} \times \$0.10/\text{kWh} = \mathbf{\$3,066,000}
\]
\[
\boxed{\text{Annual Utility Cost} = \$3.066\text{ Million USD}}
\]

#### 4. Energy Efficiency
- Convert sustained throughput to GFLOP/s:
  \[
  R_{\max} = 10\text{ PFLOP/s} = 10 \times 10^6\text{ GFLOP/s} = 10,000,000\text{ GFLOP/s}
  \]
- Compute IT power in Watts:
  \[
  P_{\text{IT}} = 2.5\text{ MW} = 2,500,000\text{ Watts}
  \]
- Energy Efficiency:
  \[
  \text{Efficiency} = \frac{10,000,000\text{ GFLOP/s}}{2,500,000\text{ Watts}} = \mathbf{4.0\text{ GFLOP/s per Watt}}
  \]

---

## 11. Conceptual Examination Questions & Answers

### Q1: Why is an FMA instruction counted as two FLOPs rather than one?
**Model Answer:**
A Fused Multiply-Add (FMA) instruction performs the combined operation $d = (a \times b) + c$. 
Mathematically, this evaluates two distinct arithmetic operations: first, a floating-point multiplication between $a$ and $b$; second, a floating-point addition of $c$ to the product. 

Although modern hardware executes this sequence in a single instruction cycle using unified hardware circuitry without intermediate rounding, it accomplishes the exact computational work of two separate scalar instructions. Therefore, standard benchmark accounting conventions (such as LINPACK) count each completed FMA instruction as **2 FLOPs**.

---

### Q2: A company claims its new processor operates at $5.0\text{ GHz}$ and is therefore twice as fast as a competitor's $2.5\text{ GHz}$ processor. Why is this claim technically invalid?
**Model Answer:**
Clock frequency measures only the rate of processor clock cycles ($f$), not the total work completed per second ($P = W/T$). 

Execution time depends on the **Iron Law of Processor Performance**:
\[
T = \frac{\text{Instructions}}{\text{Program}} \times \text{CPI} \times \frac{1}{f}
\]
Where $\text{CPI}$ is Cycles Per Instruction.

The claim is invalid because:
1. **Instruction Set & IPC:** The $2.5\text{ GHz}$ processor might have a wider superscalar pipeline or wider SIMD vector registers (e.g., 512-bit vs. 128-bit), allowing it to complete significantly more instructions or FLOPs per clock cycle (higher IPC) than the $5.0\text{ GHz}$ chip.
2. **The Memory Wall:** If the workload is memory-bound (e.g., Sparse Linear Algebra), the $5.0\text{ GHz}$ core will spend most of its clock cycles stalled waiting for DRAM transfers. A higher clock frequency merely increases idle wait cycles without speeding up memory bus transfers.

---

## 12. Quick Revision Sheet

- **Core Performance Equation:**
  \[
  \boxed{P = \frac{\text{Work}}{\text{Time}}} \implies \frac{P_1}{P_2} = \frac{T_2}{T_1}
  \]
- **Theoretical Peak Performance Equation:**
  \[
  \boxed{R_{\text{peak}} = \text{Sockets} \times \text{Cores/Socket} \times f_{\text{clock}} \times \left(\frac{\text{FLOPs}}{\text{cycle}}\right)}
  \]
- **FMA Arithmetic Accounting:**
  \[
  \boxed{1\text{ Fused Multiply-Add (FMA) Instruction} = 2\text{ FLOPs}}
  \]
- **Scale Reference:**
  - Mega ($10^6$) $\longrightarrow$ Giga ($10^9$) $\longrightarrow$ Tera ($10^{12}$) $\longrightarrow$ Peta ($10^{15}$) $\longrightarrow$ Exa ($10^{18}$)
- **PUE Equation:**
  \[
  \boxed{\text{PUE} = \frac{\text{Total Facility Power}}{\text{IT Equipment Power}}} \quad (\text{Ideal } = 1.0)
  \]
- **Common Exam Trap:**
  - *Trap:* Writing that higher PUE means a greener datacenter.
  - *Reality:* **Lower PUE is better.** A PUE of $1.1$ is vastly more efficient than a PUE of $1.8$.

---

## 13. What You Should Be Able to Do After This Topic

Before proceeding to Module 02.01, verify that you can independently:
1. Define performance as Work / Time and calculate relative speedup from execution times.
2. Explain the IEEE 754 precision formats (32-bit float vs. 64-bit double) and explain why FMA counts as 2 FLOPs.
3. Convert fluidly between MFLOP/s, GFLOP/s, TFLOP/s, PFLOP/s, and EFLOP/s.
4. Calculate $R_{\text{peak}}$ for any given processor or cluster architecture from fundamental clock and vector parameters.
5. Solve datacenter energy bills using power, PUE, and utility tariff rates.
