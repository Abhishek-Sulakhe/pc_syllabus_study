# Module 03.01 — Parallel Speedup, Efficiency & Amdahl’s Law (Strong Scaling)

> **Prerequisites:** Module 01.01 (Introduction to Parallel Computing), Module 02.04 (Memory Hierarchy & Caches)  
> **Lecture Mapping:** Lectures 3–5 (Parallel Computing Limits, Amdahl's Law, Strong Scaling, Asymptotic Limits)  
> **Exam Relevance:** MidSem (MANDATORY: Derivations, Proof of $S_{\max} = 1/s$, High-Frequency Numerical Problems) & EndSem  

---

## 1. Understanding the Basic Idea

Suppose you are leading a team tasked with building a house. 
- Laying the concrete foundation takes **10 days**. No matter how many workers you hire, concrete takes 10 days to chemically cure and harden. You cannot speed this up with more people.
- Framing the walls, roofing, and painting takes **90 days** if 1 worker does it alone. But if you hire 9 workers, they can frame and paint together in only **10 days**.

Now, how long does the entire house take with 1 worker?
\[
\text{Total Time} = 10\text{ days (foundation)} + 90\text{ days (framing)} = 100\text{ days}
\]
How long does the house take with 9 workers?
\[
\text{Total Time} = 10\text{ days (foundation)} + \frac{90\text{ days}}{9\text{ workers}} = 10 + 10 = \mathbf{20\text{ days}}
\]
You hired 9 times as many workers, but the job only finished $5\times$ faster ($100 / 20 = 5$).

Now suppose you hire **1,000,000 workers**. 
The framing and painting will finish in a fraction of a second. But the concrete foundation **still takes 10 full days to dry**! 
Therefore, even with an infinite workforce, the total construction time can **never drop below 10 days**. The maximum speedup you can ever achieve is:
\[
\text{Maximum Speedup} = \frac{100\text{ days}}{10\text{ days}} = \mathbf{10\times}
\]

```
Total Problem: [ Sequential Part: 10% ] [       Parallelizable Part: 90%       ]
Execution:     ├───────────────────────┼──────────────────────────────────────┤
1 Processor:   │ 10 seconds            │ 90 seconds                           │ = 100 s
               ├───────────────────────┼──────────┬──────────┬──────────┬─────┤
10 Processors: │ 10 seconds (Unchanged)│ 9 seconds│ 9 seconds│ 9 seconds│ ... │ =  19 s
               ├───────────────────────┼──────────┴──────────┴──────────┴─────┘
∞ Processors:  │ 10 seconds (BOTTLE-   │ 0 seconds (Approaches Zero)          │ =  10 s (Hard Limit!)
               │ NECK!)                │                                      │
               └───────────────────────┴──────────────────────────────────────┘
```

This is the historic insight formulated by computer pioneer **Gene M. Amdahl** in 1967. 
No matter how many millions of processors or supercomputer nodes you throw at a problem, **the sequential portion of the program places an impenetrable speed limit on parallel performance.**

---

## 2. Fundamental Metrics: Parallel Speedup and Parallel Efficiency

Before diving into Amdahl's Law, we must establish the formal mathematical definitions of speedup and efficiency.

### 1. Parallel Speedup ($S(N)$)
> **Parallel Speedup ($S(N)$)** is the ratio of the execution time of a sequential program running on a single processor to the execution time of the parallelized program running on $N$ identical processors.

Let:
- $T_1$ = Execution time of the program on **1 processor** (or baseline sequential time).
- $T_N$ = Execution time of the parallel program on **$N$ processors**.

\[
\boxed{S(N) = \frac{T_1}{T_N}}
\]

#### Interpretation of Speedup Values:
- **Ideal Linear Speedup ($S(N) = N$):** Doubling processors cuts execution time in half ($S(2) = 2, S(4) = 4, S(8) = 8$).
- **Sub-Linear Speedup ($S(N) < N$):** The normal real-world case. Overhead, sequential bottlenecks, and communication prevent perfect scaling.
- **Super-Linear Speedup ($S(N) > N$):** A fascinating phenomenon where running on $N$ processors yields more than $N\times$ speedup! (Explored below).

```
       SPEEDUP PROFILES
   Speedup S(N)
      ▲
   16 ┼                                      / [Ideal Linear: S(N) = N]
      │                                   /
   12 ┼                                /     ..- [Super-Linear (Cache Effect)]
      │                             /     .-'
    8 ┼                          /    _.-' [Sub-Linear: Real World]
      │                       /  _.-'
    4 ┼                    / _.-'
      │                 /_.-'
      │              _.-'
    0 ┴─────────────┴───────────┴───────────┴───────────► Processors (N)
      1             4           8           12          16
```

---

### The Mystery of Super-Linear Speedup: Why Does $S(N) > N$ Occur?
Students are often taught that $S(N)$ can never exceed $N$. In theoretical computer science with ideal abstract machines, this is true. 

However, in **real physical computer hardware**, super-linear speedup regularly occurs due to the **Aggregated Cache Effect**:

```
Single Core Execution (Problem Size = 128 MB):
Core 0 ──► [ L3 Cache: 32 MB ] ◄── MISS! Data too big!
             │
             ▼
         [ Main Memory DRAM: 128 MB ] (Slow off-chip memory access: ~80 ns)
         Execution Time T₁ is high due to constant cache misses!

4 Cores Parallel Execution (Each core gets 128 MB / 4 = 32 MB):
Core 0 ──► [ L3 Cache: 32 MB ] ◄── HIT! Entire local data fits in cache! (~10 ns)
Core 1 ──► [ L3 Cache: 32 MB ] ◄── HIT!
Core 2 ──► [ L3 Cache: 32 MB ] ◄── HIT!
Core 3 ──► [ L3 Cache: 32 MB ] ◄── HIT!
Total aggregated cache = 4 × 32 MB = 128 MB!
DRAM is NEVER touched during computation!
```
Because the aggregate cache across $N$ cores is $N$ times larger than the cache of a single core, the working set suddenly fits completely in high-speed on-chip SRAM, eliminating DRAM latency and causing $T_N$ to drop faster than $1/N$, yielding $S(N) > N$!

---

### 2. Parallel Efficiency ($\epsilon(N)$)
Speedup alone does not tell you if computational resources are being used economically. 
If 100 processors deliver a speedup of $10\times$, is that good? You are paying for 100 processors, but 90 of them are effectively wasted!

> **Parallel Efficiency ($\epsilon(N)$)** measures the fraction of time for which the processors are effectively utilized relative to ideal linear speedup.

\[
\boxed{\epsilon(N) = \frac{S(N)}{N} = \frac{T_1}{N \cdot T_N}}
\]

- In an **ideal system**: $S(N) = N \implies \epsilon(N) = \frac{N}{N} = 1.0 = \mathbf{100\%}$.
- In real-world systems: As $N$ increases, $\epsilon(N)$ monotonically decreases toward $0\%$ due to communication overhead, synchronization, and sequential fractions.

---

## 3. Mathematical Derivation of Amdahl’s Law

In 1967, Gene Amdahl published his seminal paper: *"Validity of the single processor approach to achieving large scale computing capabilities."* 

Let us derive Amdahl's Law rigorously from first principles.

### Step 1: Program Work Partitioning
Assume a program has a fixed computational workload. We divide the total sequential execution time $T_1$ into two mutually exclusive fractions:
1. **$s$ = The Sequential Fraction ($0 \le s \le 1$):**
   The portion of work that, by its algorithmic nature, must be executed sequentially by a single processor (e.g., file I/O, memory allocation, initialization, final data reduction).
2. **$p = (1 - s)$ = The Parallelizable Fraction:**
   The portion of work that can be divided evenly and executed simultaneously across $N$ processors with zero parallel overhead.

Notice that:
\[
s + p = 1 \implies p = 1 - s
\]

### Step 2: Formulating Execution Time on $N$ Processors ($T_N$)
Under **Strong Scaling** (where total problem size is kept strictly fixed):
- The sequential portion cannot be divided and takes $s \cdot T_1$ seconds on 1 processor.
- The parallelizable portion is divided equally among $N$ processors, taking $\frac{(1 - s) \cdot T_1}{N}$ seconds.

Therefore, total parallel execution time $T_N$ is:
\[
T_N = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{N}
\]
Factor out $T_1$:
\[
\boxed{T_N = T_1 \left( s + \frac{1 - s}{N} \right)}
\]

### Step 3: Deriving Speedup $S(N)$
Substitute $T_N$ into the speedup definition $S(N) = \frac{T_1}{T_N}$:
\[
S(N) = \frac{T_1}{T_1 \left( s + \frac{1 - s}{N} \right)}
\]
Cancel $T_1$ from numerator and denominator:

\[
\boxed{S(N) = \frac{1}{s + \frac{1 - s}{N}}}
\]

This is **Amdahl’s Law**.

---

## 4. The Asymptotic Limit of Amdahl’s Law ($S_{\max}$)

What happens if an organization buys an infinitely large supercomputer with an infinite number of processors ($N \to \infty$)?

Let us evaluate the mathematical limit of Amdahl’s equation as $N \to \infty$:
\[
S_{\max} = \lim_{N \to \infty} S(N) = \lim_{N \to \infty} \frac{1}{s + \frac{1 - s}{N}}
\]
As $N \to \infty$, the term $\frac{1 - s}{N} \to 0$:
\[
S_{\max} = \frac{1}{s + 0}
\]
\[
\boxed{S_{\max} = \frac{1}{s}}
\]

### The Devastating Implication: The Table of Despair
The asymptotic limit proves that **the maximum possible speedup is strictly bounded by the reciprocal of the sequential fraction**, regardless of whether you have 100 cores, 100,000 cores, or 10 billion cores!

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    THE ASYMPTOTIC SPEEDUP CEILING TABLE                         │
└─────────────────────────────────────────────────────────────────────────────────┘

 Sequential Fraction (s)   Parallel Fraction (1 - s)   Maximum Possible Speedup (S_max = 1/s)
 ───────────────────────────────────────────────────────────────────────────────────
 50% (0.50)                50% (0.50)                  1 / 0.50 = 2.0x max
 25% (0.25)                75% (0.75)                  1 / 0.25 = 4.0x max
 10% (0.10)                90% (0.90)                  1 / 0.10 = 10.0x max
  5% (0.05)                95% (0.95)                  1 / 0.05 = 20.0x max
  1% (0.01)                99% (0.99)                  1 / 0.01 = 100.0x max
  0.1% (0.001)             99.9% (0.999)               1 / 0.001 = 1,000.0x max
 ───────────────────────────────────────────────────────────────────────────────────
```

### The Iron Law of Parallel Optimization:
If just **$5\%$** of your simulation code must run sequentially ($s = 0.05$):
- Even if you run on the world's fastest supercomputer with $1,000,000$ cores...
- Your maximum possible speedup can **NEVER exceed $20\times$**!
- The remaining 999,980 cores sit completely wasted!

```
       AMDAHL'S LAW SPEEDUP SATURATION CURVES
   Speedup S(N)
      ▲
   20 ┼───────────────────────────────────────────── S_max = 20 (s = 0.05)
      │                                   . - - - -
   15 ┼                             . - '
      │                       . - '
   10 ┼──────────────────.─'──────────────────────── S_max = 10 (s = 0.10)
      │             . - '
    5 ┼───────. ─ '
      │  . - ' ◄── Rapid initial gains, then hard saturation!
    0 ┴───────┴───────────┴───────────┴───────────┴──► Processors (N)
      1       16          64          256         1024
```

---

## 5. What is Strong Scaling?

Amdahl's Law is the mathematical embodiment of **Strong Scaling**.

> **Strong Scaling** is a parallel scaling model in which the **total problem size remains strictly fixed**, while the number of processors $N$ is systematically increased.

- **The Core Question:** *"How much faster can we solve this specific problem if we throw more processors at it?"*
- **Goal:** Minimize the wall-clock time $T_N$ to completion.
- **Limitation:** As $N$ grows large, the amount of parallel work per processor shrinks ($\frac{\text{Work}}{N} \to 0$). Eventually, the fixed sequential time $s \cdot T_1$ dominates the execution time, causing the speedup curve to flatten completely.

---

## 6. Practical Implementation: Observing Amdahl's Law in C

To witness Amdahl's Law in action, consider a C program that has an intentional sequential setup phase followed by a parallel computational loop.

### Amdahl's Law Demonstration (`amdahl_demo.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <math.h>

#define N 50000000

// A computationally intensive mathematical function
double heavy_math(double x) {
    return sin(x) * cos(x) + sqrt(fabs(x));
}

int main(void) {
    printf("============================================================\n");
    printf("         EXPERIMENTAL DEMONSTRATION OF AMDAHL'S LAW         \n");
    printf("============================================================\n");

    double *data = (double *)malloc(N * sizeof(double));
    if (!data) return 1;

    // Measure single-thread baseline (T1)
    omp_set_num_threads(1);
    double t_start = omp_get_wtime();

    // 1. Sequential Fraction (s = ~10% of runtime): Serial initialization
    for (int i = 0; i < N / 10; i++) {
        data[i] = heavy_math((double)i);
    }

    // 2. Parallelizable Fraction (1 - s = ~90% of runtime): Heavy computation
    #pragma omp parallel for
    for (int i = N / 10; i < N; i++) {
        data[i] = heavy_math((double)i);
    }

    double t_end = omp_get_wtime();
    double T1 = t_end - t_start;
    printf("Baseline 1-Thread Execution Time (T1) : %.4f seconds\n", T1);
    printf("------------------------------------------------------------\n");
    printf("%-10s %-15s %-15s %-15s\n", "Threads", "Time (s)", "Speedup S(N)", "Efficiency");
    printf("------------------------------------------------------------\n");

    int thread_counts[] = {1, 2, 4, 8, 16};
    int count = sizeof(thread_counts) / sizeof(thread_counts[0]);

    for (int t = 0; t < count; t++) {
        int threads = thread_counts[t];
        omp_set_num_threads(threads);

        double start = omp_get_wtime();

        // Sequential work (always executed by 1 thread!)
        for (int i = 0; i < N / 10; i++) {
            data[i] = heavy_math((double)i);
        }

        // Parallel work distributed across 'threads'
        #pragma omp parallel for
        for (int i = N / 10; i < N; i++) {
            data[i] = heavy_math((double)i);
        }

        double end = omp_get_wtime();
        double TN = end - start;
        double speedup = T1 / TN;
        double efficiency = (speedup / threads) * 100.0;

        printf("%-10d %-15.4f %-15.2fx %-14.1f%%\n", 
               threads, TN, speedup, efficiency);
    }

    printf("============================================================\n");
    free(data);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp -lm amdahl_demo.c -o amdahl_demo
./amdahl_demo
```

*Expected Output (on a multi-core machine):*
```
============================================================
         EXPERIMENTAL DEMONSTRATION OF AMDAHL'S LAW         
============================================================
Baseline 1-Thread Execution Time (T1) : 2.5000 seconds
------------------------------------------------------------
Threads    Time (s)        Speedup S(N)    Efficiency     
------------------------------------------------------------
1          2.5000          1.00x           100.0%        
2          1.3750          1.82x           90.9%         
4          0.8125          3.08x           76.9%         
8          0.5312          4.71x           58.8%         
16         0.3906          6.40x           40.0%         
============================================================
```

#### Analytical Validation:
Notice that at 16 threads, the speedup is only **$6.40\times$** (efficiency dropped to $40\%$), and no matter how many threads you add, the speedup will asymptote toward:
\[
S_{\max} = \frac{1}{s} = \frac{1}{0.10} = \mathbf{10.0\times}
\]

---

## 7. Important Comparisons

### Speedup vs. Efficiency

| Dimension | Parallel Speedup ($S(N)$) | Parallel Efficiency ($\epsilon(N)$) |
|:---|:---|:---|
| **Fundamental Question** | *"How many times faster is the program?"* | *"How effectively are the processors utilized?"* |
| **Mathematical Formula** | $S(N) = \frac{T_1}{T_N}$ | $\epsilon(N) = \frac{S(N)}{N} = \frac{T_1}{N \cdot T_N}$ |
| **Ideal Value** | $S(N) = N$ (Linear scaling) | $\epsilon(N) = 1.0$ ($100\%$ utilization) |
| **Normal Behavior** | Monotonically increases, then saturates | Monotonically decreases as $N$ increases |
| **Economic Meaning** | Measures raw completion speed | Measures return on hardware investment / power cost |

---

## 8. Exam-Ready Definitions

### Parallel Speedup ($S(N)$)
> **Parallel Speedup ($S(N)$)** is the dimensionless metric of parallel performance defined as the ratio of the execution time of a sequential program on a single processor ($T_1$) to the execution time of the parallel program on $N$ identical processors ($T_N$).

### Parallel Efficiency ($\epsilon(N)$)
> **Parallel Efficiency ($\epsilon(N)$)** is the measure of the fractional utilization of computational resources, defined as parallel speedup divided by the number of processors ($\epsilon(N) = \frac{S(N)}{N}$), representing how close a parallel system operates to ideal linear speedup.

### Amdahl’s Law
> **Amdahl’s Law** is a mathematical law of parallel computing stating that the maximum speedup achievable by a parallel program on $N$ processors is constrained by the sequential fraction of the algorithm ($s$), formulated as $S(N) = \frac{1}{s + \frac{1 - s}{N}}$, with an asymptotic upper bound of $S_{\max} = \frac{1}{s}$ as $N \to \infty$.

### Strong Scaling
> **Strong Scaling** is a parallel scaling paradigm where the total workload (problem size) remains fixed while the number of processors is increased, measuring the program's ability to reduce time-to-solution for a given problem.

---

## 9. Solved Problems

### Problem 1: Basic Amdahl's Law Speedup Calculation
A weather forecasting program runs in $100\text{ seconds}$ on a single processor.
Profiling reveals that $80\text{ seconds}$ are spent in parallelizable atmospheric fluid loops, while $20\text{ seconds}$ are spent reading initial satellite radar files sequentially from disk.

Calculate:
1. The sequential fraction $s$ and the parallel fraction $p$.
2. The speedup $S(4)$ and efficiency $\epsilon(4)$ on $4$ processors.
3. The speedup $S(16)$ and efficiency $\epsilon(16)$ on $16$ processors.
4. The maximum theoretical speedup $S_{\max}$ with an infinite number of processors.

#### Solution:
#### 1. Identify Fractions
\[
s = \frac{20\text{ s}}{100\text{ s}} = \mathbf{0.20\ (20\%)}, \quad p = 1 - s = \mathbf{0.80\ (80\%)}
\]

#### 2. Speedup and Efficiency on $N = 4$ Processors
\[
S(4) = \frac{1}{s + \frac{1 - s}{N}} = \frac{1}{0.20 + \frac{0.80}{4}} = \frac{1}{0.20 + 0.20} = \frac{1}{0.40} = \mathbf{2.50\times}
\]
\[
\epsilon(4) = \frac{S(4)}{4} = \frac{2.50}{4} = 0.625 = \mathbf{62.5\%}
\]

#### 3. Speedup and Efficiency on $N = 16$ Processors
\[
S(16) = \frac{1}{0.20 + \frac{0.80}{16}} = \frac{1}{0.20 + 0.05} = \frac{1}{0.25} = \mathbf{4.00\times}
\]
\[
\epsilon(16) = \frac{S(16)}{16} = \frac{4.00}{16} = 0.25 = \mathbf{25.0\%}
\]

#### 4. Maximum Theoretical Speedup ($N \to \infty$)
\[
S_{\max} = \frac{1}{s} = \frac{1}{0.20} = \mathbf{5.00\times}
\]
*Conclusion:* Even if you purchase a 10,000-core supercomputer, the code will never run faster than $5\times$ baseline speed!

---

### Problem 2: Reverse Engineering Sequential Fraction from Empirical Speedup
A software engineer parallelizes an image processing program. 
On a dual-core CPU ($N = 2$), the measured speedup is $S(2) = 1.80\times$.
Assuming the program obeys Amdahl’s Law:
1. Find the exact sequential fraction $s$ of the program.
2. Predict the speedup achievable on an 8-core CPU ($N = 8$).
3. What is the absolute ceiling on speedup for this program?

#### Solution:
#### 1. Solve for $s$ using $S(2) = 1.80$
\[
S(2) = \frac{1}{s + \frac{1 - s}{2}} = 1.80
\]
Invert both sides:
\[
s + \frac{1 - s}{2} = \frac{1}{1.80} = \frac{5}{9} \approx 0.5556
\]
Multiply by 2 to clear the fraction:
\[
2s + (1 - s) = 2 \times \frac{5}{9} = \frac{10}{9}
\]
\[
s + 1 = \frac{10}{9} \implies s = \frac{10}{9} - 1 = \frac{1}{9} \approx \mathbf{0.1111\ (11.11\%)}
\]
The sequential fraction is **$11.11\%$**.

#### 2. Predict Speedup on $N = 8$
\[
S(8) = \frac{1}{s + \frac{1 - s}{8}} = \frac{1}{\frac{1}{9} + \frac{8/9}{8}} = \frac{1}{\frac{1}{9} + \frac{1}{9}} = \frac{1}{\frac{2}{9}} = \frac{9}{2} = \mathbf{4.50\times}
\]

#### 3. Absolute Ceiling ($S_{\max}$)
\[
S_{\max} = \frac{1}{s} = \frac{1}{1/9} = \mathbf{9.00\times}
\]

---

### Problem 3: Determining Required Parallel Fraction for a Target Speedup
A research lab buys a 64-core compute node ($N = 64$).
They want their parallel simulation to achieve a speedup of at least **$50\times$** ($S(64) \ge 50$).

What is the **maximum allowable sequential fraction ($s$)** the programmers can leave in the code to meet this goal?

#### Solution:
1. **Set up the inequality:**
   \[
   S(64) = \frac{1}{s + \frac{1 - s}{64}} \ge 50
   \]
2. **Invert both sides (flips inequality sign):**
   \[
   s + \frac{1 - s}{64} \le \frac{1}{50} = 0.020
   \]
3. **Multiply by 64:**
   \[
   64s + 1 - s \le 64 \times 0.020 = 1.28
   \]
   \[
   63s + 1 \le 1.28
   \]
   \[
   63s \le 0.28 \implies s \le \frac{0.28}{63} \approx \mathbf{0.00444\ (0.444\%)}
   \]
4. **Conclusion:**
   To achieve a $50\times$ speedup on 64 cores, **more than $99.56\%$ of the code must be completely parallelized!** Less than $0.44\%$ sequential work can be tolerated.

---

## 10. Conceptual Examination Questions & Answers

### Q1: State Amdahl’s Law and derive its asymptotic limit as the number of processors approaches infinity.
**Model Answer:**
Amdahl’s Law states that the speedup of a program on $N$ processors is bounded by its sequential fraction $s$:
\[
S(N) = \frac{1}{s + \frac{1 - s}{N}}
\]
Where $s$ is the fraction of sequential execution and $(1 - s)$ is the parallelizable fraction.

**Derivation of Asymptotic Limit:**
Taking the limit as $N \to \infty$:
\[
S_{\max} = \lim_{N \to \infty} \frac{1}{s + \frac{1 - s}{N}}
\]
Because $\lim_{N \to \infty} \frac{1 - s}{N} = 0$:
\[
S_{\max} = \frac{1}{s + 0} = \frac{1}{s}
\]
Thus, the maximum speedup achievable by any parallel computer running a fixed-size problem is strictly limited to the reciprocal of its sequential fraction.

---

### Q2: Can a parallel program ever achieve super-linear speedup ($S(N) > N$)? Explain the physical mechanism.
**Model Answer:**
**Yes.** While impossible on idealized mathematical models, super-linear speedup can and does occur on physical hardware primarily due to the **Aggregated Cache Effect**.

When a large problem executes on a single processor ($N = 1$), the working dataset often exceeds the capacity of the on-chip CPU caches (L1, L2, L3). The single core suffers frequent cache misses, forcing thousands of slow accesses to off-chip DRAM. 

When the exact same problem is distributed across $N$ processors, each processor receives only $\frac{1}{N}\text{th}$ of the total dataset. This smaller working chunk fits entirely inside the local on-chip cache of each core. 

Because cache access is orders of magnitude faster than DRAM ($\approx 1\text{ ns}$ vs $\approx 80\text{ ns}$), memory stall cycles drop to near zero. The execution time $T_N$ decreases by more than a factor of $N$, resulting in an empirical speedup $S(N) > N$ and efficiency $\epsilon(N) > 100\%$.

---

## 11. Quick Revision Sheet

- **Parallel Speedup:**
  \[
  \boxed{S(N) = \frac{T_1}{T_N}}
  \]
- **Parallel Efficiency:**
  \[
  \boxed{\epsilon(N) = \frac{S(N)}{N} \times 100\%}
  \]
- **Amdahl’s Law (Strong Scaling):**
  \[
  \boxed{S(N) = \frac{1}{s + \frac{1 - s}{N}}}
  \]
- **Asymptotic Limit:**
  \[
  \boxed{S_{\max} = \frac{1}{s}}
  \]
- **Strong Scaling Definition:**
  \[
  \boxed{\text{Fixed Problem Size} \longrightarrow \text{Increase Processors } N \longrightarrow \text{Speedup bounded by } \frac{1}{s}}
  \]
- **Common Exam Trap:**
  - *Trap:* Thinking that $s$ in Amdahl's Law is the fraction of sequential code lines in the source file.
  - *Correction:* $s$ is the fraction of **execution TIME** spent running sequentially on 1 processor, not line count!

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 03.02, verify that you can independently:
1. Define Speedup $S(N)$ and Efficiency $\epsilon(N)$ with physical units and interpretations.
2. Derive Amdahl’s Law mathematically from $T_1$ and $T_N$.
3. Prove that $\lim_{N \to \infty} S(N) = 1/s$ and explain why sequential bottlenecks limit supercomputing scaling.
4. Calculate speedup, efficiency, and sequential fraction $s$ from empirical benchmark timing numbers.
5. Explain the physical mechanism behind super-linear speedup ($S(N) > N$).
