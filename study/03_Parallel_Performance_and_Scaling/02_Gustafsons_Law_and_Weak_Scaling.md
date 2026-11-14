# Module 03.02 — Gustafson’s Law, Scaled Speedup & Weak Scaling

> **Prerequisites:** Module 03.01 (Parallel Speedup, Efficiency & Amdahl's Law)  
> **Lecture Mapping:** Lectures 3–5 (Gustafson's Law, Re-evaluating Amdahl's Law, Weak Scaling)  
> **Exam Relevance:** MidSem (MANDATORY: Derivation of Scaled Speedup, Amdahl vs. Gustafson Comparisons, Numerical Calculations) & EndSem  

---

## 1. Understanding the Basic Idea

In 1967, Gene Amdahl cast a dark shadow over the future of parallel computing. As we saw in Module 03.01, if a program has even $5\%$ sequential execution time ($s = 0.05$), the maximum speedup can never exceed $20\times$, even on a billion processors. 

For two decades, critics cited Amdahl's Law to argue that building massive supercomputers with thousands of processors was a colossal waste of taxpayer money. 

Then, in 1988, at Sandia National Laboratories, computer scientist **John L. Gustafson** published a groundbreaking paper in the *Communications of the ACM* titled:
> **"Reevaluating Amdahl’s Law"**

Gustafson made an observation so profound yet so simple that it revolutionized high-performance computing:
> **When scientists buy a supercomputer with 1,000 times more processors, they DO NOT run the exact same tiny problem they ran on a desktop!**
> **They scale the problem UP to fit the new machine!**

```
Amdahl's World (Fixed Problem Size / Strong Scaling):
You have a 1-acre garden. 
You hire 100 gardeners. 
They bump into each other, trip over tools, and wait for one wheelbarrow.
Result: Terrible efficiency. Diminishing returns.

Gustafson's World (Fixed Time / Weak Scaling):
You have 100 gardeners. 
You give them a 100-ACRE farm! 
Each gardener works on their own 1-acre plot simultaneously.
Result: 100 times more crops harvested in the exact same amount of time!
```

Amdahl asked: *"How much faster can I solve a fixed-size problem?"*  
Gustafson asked: *"How much larger a problem can I solve in the same amount of time?"*

---

## 2. Fixed-Size vs. Fixed-Time Paradigms

To understand the mathematical divergence between Amdahl's Law and Gustafson's Law, we must contrast their fundamental operational assumptions.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    STRONG SCALING vs. WEAK SCALING PARADIGMS                    │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. STRONG SCALING (Amdahl's Assumption - Fixed Problem Size):
    Problem Size = CONSTANT
    
    1 CPU:  [████████████████████████████████] (Work = 100 MB, Time = 100 s)
    
    2 CPUs: [████████████████] [████████████████] (Work = 50 MB each, Time = 50 s)
    
    4 CPUs: [████████] [████████] [████████] [████████] (Work = 25 MB each, Time = 25 s)
    
    Question: How fast can we finish this fixed 100 MB job?

 2. WEAK SCALING (Gustafson's Assumption - Fixed Time, Scaled Problem Size):
    Work per Processor = CONSTANT
    
    1 CPU:  [████] (Work = 100 MB, Time = 60 s)
    
    2 CPUs: [████] [████] (Work = 200 MB, Time = 60 s)
    
    4 CPUs: [████] [████] [████] [████] (Work = 400 MB, Time = 60 s)
    
    Question: How much more data can we process in 60 seconds?
```

### Why Problem Size Expands in the Real World
Consider weather forecasting:
- If a meteorologist is given a faster supercomputer, they don't want tomorrow's weather forecast calculated in 2 seconds instead of 1 hour.
- **They want a vastly more accurate forecast!**
- Instead of dividing the Earth's atmosphere into coarse $50\text{ km}$ grid blocks, they refine the grid to $1\text{ km}$ blocks, simulate atmospheric chemistry, and add cloud microphysics. 
- The amount of parallelizable arithmetic expands by factors of thousands, while the serial overhead (reading initial satellite files) remains roughly constant!

---

## 3. Mathematical Derivation of Gustafson’s Law (Scaled Speedup)

Let us derive Gustafson's Law step by step, using the exact framing from John Gustafson’s 1988 paper.

### Step 1: Normalizing the Parallel Execution Time
Gustafson noticed that in real experiments, scientists measure the time spent on the **parallel machine** ($N$ processors), not the single-core baseline.

Let the total execution time of the scaled problem on **$N$ processors** be normalized to 1 unit of time:
\[
T_N = 1.0
\]
On the parallel machine, this unit of time is composed of:
1. **$s'$ = Fraction of time spent on sequential execution** ($0 \le s' \le 1$).
2. **$(1 - s')$ = Fraction of time spent on parallel execution** across the $N$ processors.

Notice:
\[
s' + (1 - s') = 1.0
\]

### Step 2: How Long Would This Scaled Problem Take on a Single Processor ($T_1$)?
Now, imagine taking this massive, scaled-up problem and forcing **one single processor** to execute all of it sequentially:
- The sequential portion took $s'$ time on 1 core. On 1 core, it still takes:
  \[
  \text{Time}_{\text{sequential}} = s'
  \]
- The parallel portion took $(1 - s')$ time when shared across $N$ processors.
  Since 1 processor must now do all the work of all $N$ processors alone, that parallel work takes $N$ times as long:
  \[
  \text{Time}_{\text{parallel}} = N \cdot (1 - s')
  \]

Therefore, the total time for a single processor to execute the scaled workload is:
\[
\boxed{T_1 = s' + N \cdot (1 - s')}
\]

### Step 3: Deriving Scaled Speedup ($S_s(N)$)
By definition, speedup is $S = \frac{T_1}{T_N}$.
Substitute $T_1$ and $T_N = 1$:
\[
S_s(N) = \frac{s' + N \cdot (1 - s')}{1}
\]
\[
\boxed{S_s(N) = s' + N(1 - s')}
\]

We can also express this algebraically by expanding:
\[
S_s(N) = s' + N - N \cdot s' = N - s'(N - 1)
\]
\[
\boxed{S_s(N) = N - (N - 1)s'}
\]

This is **Gustafson’s Law** (often called **Gustafson-Barsis’s Law** or **Scaled Speedup**).

---

## 4. Analyzing Gustafson’s Law: The Asymptotic Behavior

Look closely at the formula:
\[
S_s(N) = N - (N - 1)s'
\]

What happens as the number of processors $N$ grows toward infinity ($N \to \infty$)?
\[
\lim_{N \to \infty} S_s(N) = \lim_{N \to \infty} [s' + N(1 - s')]
\]
As long as the parallel fraction $(1 - s') > 0$, **the speedup scales linearly with $N$!**
\[
\boxed{S_s(N) \propto N}
\]

There is **NO horizontal asymptote**! 
There is **NO speedup ceiling** like the $1/s$ barrier in Amdahl’s Law!

```
       AMDAHL'S LAW vs. GUSTAFSON'S LAW SCALING CURVES
   Speedup
      ▲                                         / [Gustafson's Law: Linear Scaling!]
  1000┼                                      /     S_s(N) = s' + N(1 - s')
      │                                   /
   800┼                                /
      │                             /
   600┼                          /
      │                       /
   400┼                    /
      │                 /
   200┼              /
      │          /
    20┼─────────┴──────────────────────────────── [Amdahl's Law: Hard Ceiling 1/s = 20]
      └─────────┴───────────┴───────────┴────────► Processors (N)
      0        200         400         600       1000
```

### The Difference in Meaning of $s$ vs. $s'$
This is the single most common conceptual trap in parallel computing examinations:
- In **Amdahl's Law**, $s$ is the sequential fraction of a **small, fixed-size problem** measured on **1 processor**.
- In **Gustafson's Law**, $s'$ is the sequential fraction of a **massive, scaled-up problem** measured on **$N$ processors**.

As problem size expands, the total number of grid cells or matrix entries grows as $\mathcal{O}(N^2)$ or $\mathcal{O}(N^3)$, while serial overhead (such as program launch and I/O) grows slowly as $\mathcal{O}(1)$ or $\mathcal{O}(N)$. 
Consequently, **in real-world large-scale problems, $s'$ becomes vanishingly small** ($s' \to 0$), enabling near-ideal linear speedup on thousands of supercomputer nodes!

---

## 5. Master Comparison: Amdahl’s Law vs. Gustafson’s Law

| Architectural Dimension | Amdahl’s Law (1967) | Gustafson’s Law (1988) |
|:---|:---|:---|
| **Scaling Model** | **Strong Scaling** | **Weak Scaling** |
| **Problem Workload** | **Fixed Problem Size** (Constant total work $W$) | **Scaled Problem Size** (Work scales linearly with $N$) |
| **Execution Time Goal** | Minimize wall-clock time $T_N \to 0$ | Maintain constant wall-clock time $T_N \approx \text{const}$ |
| **Core Research Question**| *"How fast can we finish this fixed job?"* | *"How large a problem can we solve in fixed time?"*|
| **Mathematical Formula**| $S(N) = \frac{1}{s + \frac{1 - s}{N}}$ | $S_s(N) = s' + N(1 - s') = N - (N - 1)s'$ |
| **Asymptotic Limit ($N \to \infty$)**| $S_{\max} = \frac{1}{s}$ (Strict horizontal ceiling) | No ceiling; scales linearly ($S_s(N) \to \infty$) |
| **Perspective on HPC** | Pessimistic; suggests supercomputing hits a wall | Optimistic; justifies building massively parallel clusters|
| **Physical Analogy** | Adding more engines to a car on a fixed 10-mile track | Adding more trucks to deliver 100 times more cargo |

---

## 6. Practical Implementation: Strong Scaling vs. Weak Scaling in C

To observe the difference empirically, examine the C program below. It runs both a **Strong Scaling test** (fixed $N$) and a **Weak Scaling test** (work scaled proportionally with thread count).

### Scaling Benchmark (`scaling_comparison.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <math.h>

#define BASE_WORK_PER_THREAD 10000000 // 10 Million elements per thread

// Intensive arithmetic function
double compute_kernel(double x) {
    return sqrt(fabs(sin(x) * cos(x) + 1.5));
}

int main(void) {
    printf("============================================================\n");
    printf("       STRONG SCALING vs. WEAK SCALING EXPERIMENT           \n");
    printf("============================================================\n");

    int thread_counts[] = {1, 2, 4, 8};
    int num_tests = sizeof(thread_counts) / sizeof(thread_counts[0]);

    // -------------------------------------------------------------
    // PART 1: STRONG SCALING (Problem size is FIXED at 40 Million)
    // -------------------------------------------------------------
    long long fixed_total_work = 40000000;
    printf("\n--- PART 1: STRONG SCALING (Fixed Problem Size: %lld items) ---\n", 
           fixed_total_work);
    printf("%-10s %-15s %-15s %-15s\n", "Threads", "Time (s)", "Speedup", "Efficiency");

    double strong_T1 = 0.0;

    for (int t = 0; t < num_tests; t++) {
        int threads = thread_counts[t];
        omp_set_num_threads(threads);

        double start = omp_get_wtime();

        #pragma omp parallel for schedule(static)
        for (long long i = 0; i < fixed_total_work; i++) {
            compute_kernel((double)i);
        }

        double end = omp_get_wtime();
        double elapsed = end - start;

        if (threads == 1) strong_T1 = elapsed;

        double speedup = strong_T1 / elapsed;
        double efficiency = (speedup / threads) * 100.0;

        printf("%-10d %-15.4f %-15.2fx %-14.1f%%\n", 
               threads, elapsed, speedup, efficiency);
    }

    // -------------------------------------------------------------
    // PART 2: WEAK SCALING (Problem size scales with thread count!)
    // -------------------------------------------------------------
    printf("\n--- PART 2: WEAK SCALING (Work per thread: %d items) ---\n", 
           BASE_WORK_PER_THREAD);
    printf("%-10s %-20s %-15s %-15s\n", "Threads", "Total Elements", "Time (s)", "Efficiency");

    double weak_T1 = 0.0;

    for (int t = 0; t < num_tests; t++) {
        int threads = thread_counts[t];
        long long current_total_work = (long long)threads * BASE_WORK_PER_THREAD;
        omp_set_num_threads(threads);

        double start = omp_get_wtime();

        #pragma omp parallel for schedule(static)
        for (long long i = 0; i < current_total_work; i++) {
            compute_kernel((double)i);
        }

        double end = omp_get_wtime();
        double elapsed = end - start;

        if (threads == 1) weak_T1 = elapsed;

        // Weak scaling efficiency = T1 / TN (Ideally exactly 100%)
        double weak_efficiency = (weak_T1 / elapsed) * 100.0;

        printf("%-10d %-20lld %-15.4f %-14.1f%%\n", 
               threads, current_total_work, elapsed, weak_efficiency);
    }

    printf("============================================================\n");
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp -lm scaling_comparison.c -o scaling_comparison
./scaling_comparison
```

*Expected Output (on an 8-core CPU):*
```
============================================================
       STRONG SCALING vs. WEAK SCALING EXPERIMENT           
============================================================

--- PART 1: STRONG SCALING (Fixed Problem Size: 40000000 items) ---
Threads    Time (s)        Speedup         Efficiency    
1          1.8200          1.00x           100.0%        
2          0.9350          1.95x           97.3%         
4          0.4850          3.75x           93.8%         
8          0.2800          6.50x           81.2%         

--- PART 2: WEAK SCALING (Work per thread: 10000000 items) ---
Threads    Total Elements       Time (s)        Efficiency    
1          10000000             0.4550          100.0%        
2          20000000             0.4600          98.9%         
4          40000000             0.4720          96.4%         
8          80000000             0.4890          93.0%         
============================================================
```

#### The Profound Takeaway:
- In **Strong Scaling**, as threads increase, execution time drops, but efficiency degrades noticeably ($81.2\%$ at 8 threads) as overhead and fixed costs bite.
- In **Weak Scaling**, the execution time remains **almost perfectly flat ($\approx 0.46\text{ to }0.48\text{ s}$)** even though we processed **$8\times$ more data**! Efficiency remained above **$93\%$**!

---

## 7. Exam-Ready Definitions

### Gustafson’s Law (Scaled Speedup)
> **Gustafson’s Law** is a parallel scaling law stating that for a problem where the workload scales proportionally with the number of processors to maintain constant execution time, the achievable speedup is given by $S_s(N) = s' + N(1 - s') = N - (N - 1)s'$, where $s'$ is the sequential fraction of execution time on the parallel system.

### Weak Scaling
> **Weak Scaling** is a parallel scaling paradigm in which the workload assigned to each individual processor remains constant as the total number of processors increases, measuring the system’s ability to solve proportionally larger problems in constant wall-clock time.

### Scaled Speedup
> **Scaled Speedup** is the speedup metric defined under the assumption of weak scaling, representing the ratio of the time required by a single processor to execute a scaled-up workload to the time taken by $N$ processors executing that same scaled workload.

---

## 8. Solved Problems

### Problem 1: Direct Comparison of Amdahl vs. Gustafson Speedup
A computational fluid dynamics application is run on a supercomputer with $N = 64$ processors.
Suppose profiling indicates a sequential fraction of $5\%$ ($0.05$).

Calculate:
1. The speedup predicted by **Amdahl's Law** (assuming a fixed problem size).
2. The scaled speedup predicted by **Gustafson's Law** (assuming the problem size scales with processor count, where $s' = 0.05$).
3. Now recalculate both speedups for a supercomputer with $N = 1024$ processors.
4. Comment on the dramatic contrast between the two results.

#### Solution:
#### 1. On $N = 64$ Processors:
- **Amdahl’s Law (Strong Scaling):**
  \[
  S_{\text{Amdahl}}(64) = \frac{1}{s + \frac{1 - s}{N}} = \frac{1}{0.05 + \frac{0.95}{64}} = \frac{1}{0.05 + 0.01484} = \frac{1}{0.06484} \approx \mathbf{15.42\times}
  \]
- **Gustafson’s Law (Weak Scaling):**
  \[
  S_{\text{Gustafson}}(64) = s' + N(1 - s') = 0.05 + 64(1 - 0.05) = 0.05 + 64(0.95) = 0.05 + 60.80 = \mathbf{60.85\times}
  \]

#### 2. On $N = 1024$ Processors:
- **Amdahl’s Law (Strong Scaling):**
  \[
  S_{\text{Amdahl}}(1024) = \frac{1}{0.05 + \frac{0.95}{1024}} = \frac{1}{0.05 + 0.000928} = \frac{1}{0.050928} \approx \mathbf{19.64\times}
  \]
  *(Note: It is already trapped hard against the theoretical ceiling of $S_{\max} = 1/0.05 = 20\times$!)*
- **Gustafson’s Law (Weak Scaling):**
  \[
  S_{\text{Gustafson}}(1024) = 0.05 + 1024(0.95) = 0.05 + 972.80 = \mathbf{972.85\times}
  \]

#### 3. Contrast & Analysis:
- On 1024 processors, Amdahl’s Law predicts a meager **$19.64\times$** speedup (an abysmal efficiency of $\epsilon = 19.64 / 1024 \approx 1.9\%$).
- Gustafson’s Law predicts an impressive **$972.85\times$** scaled speedup (an outstanding efficiency of $\epsilon = 972.85 / 1024 \approx 95.0\%$).
- This mathematical contrast proves that massively parallel computing is economically justified **only when problem size expands with processor count**.

---

### Problem 2: Finding Single-Processor Equivalent Execution Time
A weather simulation runs on an HPC cluster with $N = 512$ nodes. 
Under weak scaling, the simulation takes exactly **$1.0\text{ hour}$** to complete. 
Profiling shows that during this $1\text{-hour}$ parallel run, $2\%$ of the time ($s' = 0.02$) was spent in serial coordination (0.02 hours), while $98\%$ of the time was spent in parallel computation.

Calculate:
1. The Gustafson scaled speedup $S_s(512)$.
2. The total wall-clock time (in days) that would be required if a single processor were forced to run this entire scaled simulation alone.

#### Solution:
#### 1. Gustafson Scaled Speedup
\[
S_s(512) = s' + N(1 - s') = 0.02 + 512(1 - 0.02) = 0.02 + 512(0.98) = 0.02 + 501.76 = \mathbf{501.78\times}
\]

#### 2. Single-Processor Equivalent Time ($T_1$)
Recall the derivation of $T_1$ from Gustafson’s Law:
\[
T_1 = s' \cdot T_N + N(1 - s') \cdot T_N
\]
Given $T_N = 1.0\text{ hour}$:
\[
T_1 = 0.02\text{ hours} + 512(0.98)\text{ hours} = 0.02 + 501.76 = \mathbf{501.78\text{ hours}}
\]
Convert hours to days:
\[
\text{Time in Days} = \frac{501.78\text{ hours}}{24\text{ hours/day}} \approx \mathbf{20.91\text{ days}}
\]
*Insight:* The parallel cluster completed in **1 hour** a scientific computation that would have taken a single processor **nearly three full weeks** of continuous execution!

---

## 9. Conceptual Examination Questions & Answers

### Q1: How did John Gustafson "re-evaluate" Amdahl’s Law? What was the fundamental flaw in Amdahl’s assumption?
**Model Answer:**
John Gustafson demonstrated that Amdahl’s Law relies on an unrealistic assumption: that the **problem size remains fixed** when more processors are added (**Strong Scaling**). 

Gustafson pointed out that in scientific computing, researchers do not use massive supercomputers merely to run the same small problems faster. Instead, they exploit increased computing power to run **larger, higher-resolution problems** that were previously intractable. 

By modeling execution under the assumption that problem size scales with processor count to maintain constant execution time (**Weak Scaling**), Gustafson derived the scaled speedup formula $S_s(N) = s' + N(1 - s')$. This proved that speedup scales almost linearly with $N$ because the parallel computational work expands much faster than the fixed sequential overhead.

---

### Q2: Why does Gustafson’s Law predict linear speedup without a horizontal asymptote, unlike Amdahl’s Law?
**Model Answer:**
In Amdahl’s Law, the total workload is fixed. As $N \to \infty$, the parallel portion shrinks toward zero ($\frac{1-s}{N} \to 0$), leaving the fixed sequential time $s$ as an immovable bottleneck that imposes an upper bound of $S_{\max} = 1/s$.

In Gustafson’s Law, the parallel workload **scales up proportionally with $N$**. 
When $N$ processors are used, the total parallel work performed is $N \times (1 - s')$. 
Even though the sequential portion remains, the parallel work grows linearly with $N$. 
Therefore:
\[
\lim_{N \to \infty} S_s(N) = \lim_{N \to \infty} [s' + N(1 - s')] = \infty
\]
Because the volume of parallel work expands at the same rate as the hardware resources, the speedup curve continues to rise linearly with slope $(1 - s')$ rather than leveling off.

---

## 10. Quick Revision Sheet

- **Gustafson’s Law Formula (Scaled Speedup):**
  \[
  \boxed{S_s(N) = s' + N(1 - s') = N - (N - 1)s'}
  \]
- **Weak Scaling Definition:**
  \[
  \boxed{\text{Constant Work per Processor} \longrightarrow \text{Constant Execution Time} \longrightarrow \text{Linear Speedup } \mathcal{O}(N)}
  \]
- **Summary of Terms:**
  - $s$ (Amdahl): Sequential fraction of a *fixed problem* on *1 processor*.
  - $s'$ (Gustafson): Sequential fraction of a *scaled problem* on *N processors*.
- **Amdahl vs. Gustafson Core Contrast:**
  - **Amdahl:** Fixed Problem Size (Strong Scaling) $\implies S_{\max} = \frac{1}{s}$.
  - **Gustafson:** Scaled Problem Size (Weak Scaling) $\implies S_s(N) \propto N$.
- **Common Exam Trap:**
  - *Trap:* Confusing Weak Scaling with Strong Scaling in problem statements.
  - *Rule of Thumb:* If the problem mentions "same problem size" $\implies$ use **Amdahl's Law**. If it mentions "work scaled with processors" or "fixed execution time" $\implies$ use **Gustafson's Law**.

---

## 11. What You Should Be Able to Do After This Topic

Before proceeding to Module 03.03, verify that you can independently:
1. State the fundamental philosophical difference between Amdahl's Law and Gustafson's Law.
2. Derive Gustafson's Scaled Speedup formula $S_s(N) = s' + N(1 - s')$ from first principles.
3. Contrast Strong Scaling and Weak Scaling with execution diagrams and real-world examples.
4. Solve numerical exam problems comparing Amdahl and Gustafson speedup predictions for large $N$.
5. Explain why real-world supercomputing clusters are evaluated primarily using weak scaling benchmarks.
