# Module 03.03 — Parallel Overhead Models $c(N)$ & Load Imbalance

> **Prerequisites:** Module 03.01 (Amdahl's Law & Strong Scaling), Module 03.02 (Gustafson's Law & Weak Scaling)  
> **Lecture Mapping:** Lectures 3–5 (Parallel Computing Limits: Overhead Models $c(N)$, Load Imbalance)  
> **Exam Relevance:** MidSem (Overhead Derivations, Peak Optimal Processor Calculations, Load Imbalance Quantification) & EndSem  

---

## 1. Understanding the Basic Idea

In elementary parallel computing theory (like pure Amdahl's Law), we make an overly optimistic assumption:
> *"If 1 worker takes 100 seconds to dig a ditch, then 100 workers will take exactly 1 second."*

In the real physical world, this never happens. Why?
1. **The Overhead Problem:** Before the 100 workers can dig, a manager must gather them, explain the plan, hand out shovels, and blow a whistle (**Thread Creation & Scheduling Overhead**). While digging, workers must communicate to avoid hitting each other (**Synchronization & Contention Overhead**).
2. **The Load Imbalance Problem:** Some parts of the ground are soft dirt, while other parts are solid rock. Worker 1 finishes their dirt patch in 10 seconds and sits idle waiting. Worker 100 hits a giant granite boulder and takes 50 seconds. The entire project is not done until Worker 100 finishes!

```
Idealized Theory (Zero Overhead, Perfect Balance):
Worker 0: [████████] (Time: 25s)
Worker 1: [████████] (Time: 25s)
Worker 2: [████████] (Time: 25s)
Worker 3: [████████] (Time: 25s)
Total Time = 25s

Real-World Reality (Overhead + Load Imbalance):
Worker 0: [Overhead][██████████████████████████████] (Heavy work: 45s)
Worker 1: [Overhead][████████████]────────────────── (Light work: 20s, IDLE STALL 25s!)
Worker 2: [Overhead][████████████████████]────────── (Medium work: 30s, IDLE STALL 15s!)
Worker 3: [Overhead][████████████████████████]────── (Work: 35s, IDLE STALL 10s!)
                                                   ▲
                                          Barrier Synchronization
Total Time = 45s (Determined entirely by the slowest thread!)
```

This module explores the two most destructive performance killers in real-world parallel computing:
- **Parallel Overhead ($c(N)$):** The penalty you pay just to manage parallelism.
- **Load Imbalance:** The penalty you pay when work is distributed unevenly.

---

## 2. The Parallel Overhead Function $c(N)$

In real hardware, parallel execution time does not simply drop toward zero. Every parallel system incurs extra work that would never exist in a sequential program.

We formalize this by defining the **Parallel Overhead Function $c(N)$**:
> **Parallel Overhead $c(N)$** is the total extra time spent by a parallel system performing non-computational tasks associated with managing, coordinating, synchronizing, and communicating across $N$ processors.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    THE SOURCES OF PARALLEL OVERHEAD c(N)                        │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. THREAD / PROCESS CREATION OVERHEAD (t_spawn)
    OS system calls, allocating thread stacks, initializing process control blocks.

 2. SYNCHRONIZATION OVERHEAD (t_sync)
    Waiting at barriers, acquiring mutex locks, resolving atomic memory contention.

 3. COMMUNICATION OVERHEAD (t_comm)
    Packing network buffers, TCP/IP or InfiniBand packet flight latency, routing.

 4. SCHEDULING & ARBITRATION OVERHEAD (t_sched)
    Task queue contention, dynamic load-balancing work-stealing checks.

 5. HARDWARE CACHE COHERENCE TRAFFIC (t_coherence)
    Cache line invalidations and cross-socket bus snooping (False Sharing).
```

---

## 3. The Extended Execution Time & Speedup Model

Let us incorporate the overhead function $c(N)$ into Amdahl's formulation:

### 1. Extended Execution Time $T(N)$
\[
\boxed{T(N) = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{N} + c(N)}
\]
Where:
- $s \cdot T_1$ is the fixed sequential time.
- $\frac{(1 - s) \cdot T_1}{N}$ is the parallelizable computation time (shrinks as $N$ grows).
- $c(N)$ is the parallel overhead time (**GROWS as $N$ grows!**).

### 2. How Does Overhead Scale with $N$?
Overhead is almost never constant; it increases as more processors join the computation:
- **Constant Overhead ($\mathcal{O}(1)$):** Fixed setup costs (rare in practice).
- **Logarithmic Overhead ($\mathcal{O}(\log N)$):** Tree-based synchronization barriers and reduction trees.
- **Linear Overhead ($\mathcal{O}(N)$):** Master-worker serialized distribution, broadcast loops.
- **Quadratic Overhead ($\mathcal{O}(N^2)$):** All-to-all communication where every processor communicates with every other processor!

### 3. The Phenomenon of Speedup Roll-off (Degradation)
Because $\frac{(1 - s)T_1}{N}$ decreases while $c(N)$ increases, there exists an **optimal number of processors ($N_{\text{opt}}$)**:
- Up to $N_{\text{opt}}$, adding processors decreases execution time.
- Beyond $N_{\text{opt}}$, **overhead dominates computation** ($c(N) > \frac{(1-s)T_1}{N}$), causing execution time to **increase** and speedup to **collapse**!

```
       REAL-WORLD SPEEDUP CURVE WITH OVERHEAD c(N)
   Speedup S(N)
      ▲
      │                              Peak Optimal Speedup S_max
   12 ┼                                      /▼\
      │                                    /     \  [Overhead Dominates!]
   10 ┼                                  /         \
      │                                /             \
    8 ┼                              /                 \
      │                            /                     \
    6 ┼                          /                         \
      │                        /                             \
    4 ┼                      /                                 \
      │                    /                                     \
    2 ┼                  /                                         \
      │              _.-'
    0 ┴─────────────┴─────────────────────────┴─────────────────────► Processors (N)
      1             N_opt                     High N
                    [Sweet Spot]              [Severe Slowdown!]
```

---

## 4. Mathematical Derivation: Finding the Optimal Processor Count ($N_{\text{opt}}$)

Suppose overhead scales linearly with processor count:
\[
c(N) = \alpha \cdot N
\]
Where $\alpha$ is the per-processor overhead coefficient in seconds.

The total parallel execution time is:
\[
T(N) = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{N} + \alpha \cdot N
\]

To find the number of processors $N$ that minimizes execution time $T(N)$, take the derivative with respect to $N$ and set it to zero:
\[
\frac{d T(N)}{d N} = 0
\]
\[
\frac{d}{d N} \left[ s \cdot T_1 + (1 - s)T_1 \cdot N^{-1} + \alpha \cdot N \right] = 0
\]
\[
0 - (1 - s)T_1 \cdot N^{-2} + \alpha = 0
\]
\[
\frac{(1 - s)T_1}{N^2} = \alpha
\]
Multiply by $N^2$ and divide by $\alpha$:
\[
N^2 = \frac{(1 - s)T_1}{\alpha}
\]
Take the square root:
\[
\boxed{N_{\text{opt}} = \sqrt{\frac{(1 - s)T_1}{\alpha}}}
\]

### Engineering Significance:
1. If parallel work $(1 - s)T_1$ is massive, $N_{\text{opt}}$ is large (you can use many cores effectively).
2. If overhead $\alpha$ is high (slow network, slow lock contention), $N_{\text{opt}}$ shrinks dramatically.
3. Adding more processors beyond $N_{\text{opt}}$ is actively harmful—it makes the code run slower while burning more electricity!

---

## 5. Load Imbalance: The Silent Speedup Killer

Even if overhead $c(N) = 0$, parallel scaling can be ruined by **Load Imbalance**.

> **The Fundamental Law of Synchronous Parallelism:**
> In any parallel phase ending with a barrier or synchronization point, **the execution time of the entire team is determined strictly by the slowest thread!**

```
       SYNCHRONIZATION BARRIER PENALTY
                 Time ──►
 Core 0: ├───────────────────┤ [Idle Waiting...] ════════════════════════════► █ BARRIER
 Core 1: ├───────────────────────────────┤ [Idle Waiting...] ════════════════► █ BARRIER
 Core 2: ├──────────┤ [Long Idle Stall...] ══════════════════════════════════► █ BARRIER
 Core 3: ├───────────────────────────────────────────────────────────────────► █ BARRIER
                                                                              ▲
                                                                     Slowest Thread
                                                                     Determines Time!
```

### Causes of Load Imbalance in Real Programs
1. **Non-Uniform Loop Workloads:**
   ```c
   // Triangular Loop: Iteration i does 'i' units of work!
   for (int i = 0; i < N; i++) {
       for (int j = 0; j < i; j++) { // Early iterations do almost nothing;
           compute(i, j);            // Late iterations do massive work!
       }
   }
   ```
2. **Data-Dependent Branch Divergence:**
   In Mandelbrot fractals or ray tracing, rays hitting empty space finish in 2 iterations, while rays hitting reflective glass bounce 500 times.
3. **Irregular Meshes & Graph Traversal:**
   In social networks, high-degree hub nodes (millions of connections) take $10,000\times$ longer to process than normal nodes (10 connections).
4. **Hardware Heterogeneity / System Noise:**
   One core suffers OS kernel interrupts, thermal throttling, or background daemon execution, slowing down while other cores wait.

---

## 6. Quantifying Load Imbalance Mathematically

To measure and optimize load balance, HPC engineers use two standard metrics:

### 1. Load Imbalance Factor ($\lambda$)
Let $t_i$ be the computation time spent by processor $i$ ($i = 0, 1, \dots, N-1$).
- Maximum execution time across all processors:
  \[
  t_{\max} = \max_{0 \le i < N} (t_i)
  \]
- Average execution time across all processors:
  \[
  t_{\text{avg}} = \frac{1}{N} \sum_{i=0}^{N-1} t_i
  \]

The **Load Imbalance Factor ($\lambda$)** is defined as:
\[
\boxed{\lambda = \frac{t_{\max} - t_{\text{avg}}}{t_{\max}} = 1 - \frac{t_{\text{avg}}}{t_{\max}}}
\]

#### Interpretation of $\lambda$:
- **$\lambda = 0.0$ ($0\%$):** Perfect load balance ($t_{\max} = t_{\text{avg}}$). Every processor does identical work.
- **$\lambda \to 1.0$ ($100\%$):** Catastrophic imbalance. One single processor does all the work while $N-1$ processors sit idle.

### 2. Imbalance Ratio ($\beta$)
The ratio of maximum work to average work:
\[
\boxed{\beta = \frac{t_{\max}}{t_{\text{avg}}}} \quad (\beta \ge 1.0)
\]
- $\beta = 1.0$ indicates perfect balance.
- $\beta = 2.0$ indicates that the slowest thread did twice the average work, cutting parallel efficiency in half!

---

## 7. Practical Implementation: Observing Load Imbalance in C

Here is a C program demonstrating non-uniform loop workloads. It compares a **static chunk distribution** (which suffers severe load imbalance on triangular loops) against a **dynamic guided work distribution**.

### Load Imbalance Benchmark (`load_imbalance_demo.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <math.h>

#define N 10000

// Simulated non-uniform computational workload
void irregular_work(int i) {
    // Workload increases quadratically with loop index i!
    int iterations = i * 50; 
    double dummy = 1.0;
    for (int k = 0; k < iterations; k++) {
        dummy += sin((double)k);
    }
}

int main(void) {
    printf("============================================================\n");
    printf("      DEMONSTRATING LOAD IMBALANCE IN PARALLEL LOOPS        \n");
    printf("============================================================\n");

    int num_threads = 4;
    omp_set_num_threads(num_threads);

    double thread_times[4];

    // -------------------------------------------------------------
    // Case 1: Naive Static Scheduling (Severe Load Imbalance!)
    // Thread 0 gets iterations 0..2499 (Light work)
    // Thread 3 gets iterations 7500..9999 (Massive work!)
    // -------------------------------------------------------------
    double t_start = omp_get_wtime();

    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        double my_start = omp_get_wtime();

        #pragma omp for schedule(static)
        for (int i = 0; i < N; i++) {
            irregular_work(i);
        }

        double my_end = omp_get_wtime();
        thread_times[tid] = my_end - my_start;
    }

    double t_end = omp_get_wtime();
    double total_static_time = t_end - t_start;

    printf("\nCase 1: Static Scheduling Execution Times:\n");
    double max_time = 0.0, sum_time = 0.0;
    for (int t = 0; t < num_threads; t++) {
        printf("  Thread %d Execution Time : %.4f s\n", t, thread_times[t]);
        if (thread_times[t] > max_time) max_time = thread_times[t];
        sum_time += thread_times[t];
    }

    double avg_time = sum_time / num_threads;
    double lambda = (max_time - avg_time) / max_time;

    printf("  Total Wall-Clock Time   : %.4f s (Determined by Thread 3!)\n", total_static_time);
    printf("  Load Imbalance Factor   : %.2f%% (Massive waste!)\n", lambda * 100.0);

    // -------------------------------------------------------------
    // Case 2: Dynamic Scheduling (Fixing Load Imbalance!)
    // Iterations handed out dynamically as threads become free
    // -------------------------------------------------------------
    t_start = omp_get_wtime();

    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        double my_start = omp_get_wtime();

        #pragma omp for schedule(dynamic, 64)
        for (int i = 0; i < N; i++) {
            irregular_work(i);
        }

        double my_end = omp_get_wtime();
        thread_times[tid] = my_end - my_start;
    }

    t_end = omp_get_wtime();
    double total_dynamic_time = t_end - t_start;

    printf("\nCase 2: Dynamic Scheduling Execution Times:\n");
    max_time = 0.0; sum_time = 0.0;
    for (int t = 0; t < num_threads; t++) {
        printf("  Thread %d Execution Time : %.4f s\n", t, thread_times[t]);
        if (thread_times[t] > max_time) max_time = thread_times[t];
        sum_time += thread_times[t];
    }

    avg_time = sum_time / num_threads;
    lambda = (max_time - avg_time) / max_time;

    printf("  Total Wall-Clock Time   : %.4f s\n", total_dynamic_time);
    printf("  Load Imbalance Factor   : %.2f%% (Near-perfect balance!)\n", lambda * 100.0);
    printf("\nSpeedup from Balancing Load: %.2fx faster!\n", 
           total_static_time / total_dynamic_time);
    printf("============================================================\n");
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O2 -fopenmp -lm load_imbalance_demo.c -o load_imbalance_demo
./load_imbalance_demo
```

*Expected Output:*
```
============================================================
      DEMONSTRATING LOAD IMBALANCE IN PARALLEL LOOPS        
============================================================

Case 1: Static Scheduling Execution Times:
  Thread 0 Execution Time : 0.0450 s
  Thread 1 Execution Time : 0.1380 s
  Thread 2 Execution Time : 0.2310 s
  Thread 3 Execution Time : 0.3240 s
  Total Wall-Clock Time   : 0.3245 s (Determined by Thread 3!)
  Load Imbalance Factor   : 43.05% (Massive waste!)

Case 2: Dynamic Scheduling Execution Times:
  Thread 0 Execution Time : 0.1850 s
  Thread 1 Execution Time : 0.1845 s
  Thread 2 Execution Time : 0.1860 s
  Thread 3 Execution Time : 0.1855 s
  Total Wall-Clock Time   : 0.1865 s
  Load Imbalance Factor   : 0.67% (Near-perfect balance!)

Speedup from Balancing Load: 1.74x faster!
============================================================
```

#### Analytical Insight:
In Case 1, Thread 0 finished in **$0.045\text{ s}$** and sat completely idle for nearly **$0.28\text{ seconds}$** waiting for Thread 3 at the loop barrier! 
By switching to dynamic scheduling, faster threads automatically consumed more work chunks, balancing execution times across all cores and speeding up the program by **$1.74\times$** without changing any algorithms or hardware!

---

## 8. Exam-Ready Definitions

### Parallel Overhead ($c(N)$)
> **Parallel Overhead ($c(N)$)** is the aggregate execution time spent by a parallel computing system performing non-computational tasks necessitated by parallelism—including thread management, process synchronization, inter-processor communication, and memory contention—which would not exist in a sequential implementation.

### Load Imbalance
> **Load Imbalance** is the uneven distribution of computational work across parallel processing elements, causing faster or lightly loaded processors to sit idle at synchronization barriers while waiting for the most heavily loaded processor to finish.

### Load Imbalance Factor ($\lambda$)
> The **Load Imbalance Factor ($\lambda$)** is a normalized metric quantifying the severity of uneven workload distribution, defined as $\lambda = \frac{t_{\max} - t_{\text{avg}}}{t_{\max}}$, where $t_{\max}$ is the execution time of the slowest processor and $t_{\text{avg}}$ is the mean execution time across all processors.

### Optimal Processor Count ($N_{\text{opt}}$)
> The **Optimal Processor Count ($N_{\text{opt}}$)** is the specific number of parallel processing elements that minimizes total execution time for a given workload, beyond which the marginal increase in parallel overhead exceeds the marginal reduction in computational time.

---

## 9. Solved Problems

### Problem 1: Calculating $N_{\text{opt}}$ and Maximum Achievable Speedup
A parallel weather simulation has a baseline sequential execution time of $T_1 = 1,000\text{ seconds}$. 
- The sequential fraction is $s = 0.04$ ($4\%$).
- Profiling indicates that parallel communication and barrier overhead scales linearly as $c(N) = 0.10 \times N\text{ seconds}$.

Calculate:
1. The execution time on $N = 10$ processors.
2. The optimal number of processors $N_{\text{opt}}$ that minimizes total execution time.
3. The minimum achievable execution time $T(N_{\text{opt}})$.
4. The maximum achievable speedup $S(N_{\text{opt}})$.
5. Compare this real maximum speedup with the theoretical Amdahl ceiling ($1/s$).

#### Solution:
#### 1. Execution Time on $N = 10$ Processors
\[
T(10) = s \cdot T_1 + \frac{(1 - s)T_1}{10} + c(10)
\]
- $s \cdot T_1 = 0.04 \times 1,000 = 40\text{ s}$
- $(1 - s)T_1 = 0.96 \times 1,000 = 960\text{ s}$
- $c(10) = 0.10 \times 10 = 1.0\text{ s}$
\[
T(10) = 40 + \frac{960}{10} + 1.0 = 40 + 96 + 1.0 = \mathbf{137.0\text{ seconds}}
\]

#### 2. Calculate $N_{\text{opt}}$
Using the derived optimization formula:
\[
N_{\text{opt}} = \sqrt{\frac{(1 - s)T_1}{\alpha}}
\]
Where $\alpha = 0.10$:
\[
N_{\text{opt}} = \sqrt{\frac{960}{0.10}} = \sqrt{9,600} \approx \mathbf{97.98 \implies 98\text{ processors}}
\]

#### 3. Minimum Execution Time $T(98)$
\[
T(98) = 40 + \frac{960}{98} + (0.10 \times 98) = 40 + 9.796 + 9.800 = \mathbf{59.596\text{ seconds}}
\]

#### 4. Maximum Achievable Speedup
\[
S(N_{\text{opt}}) = \frac{T_1}{T(N_{\text{opt}})} = \frac{1,000\text{ s}}{59.596\text{ s}} \approx \mathbf{16.78\times}
\]

#### 5. Comparison with Theoretical Amdahl Ceiling:
- Pure Amdahl's Law (ignoring overhead) predicts:
  \[
  S_{\max} = \frac{1}{s} = \frac{1}{0.04} = \mathbf{25.00\times}
  \]
- In reality, because of the overhead function $c(N)$, the maximum speedup peaks at **$16.78\times$ on 98 processors** and then degrades! If you run on 500 processors:
  \[
  T(500) = 40 + \frac{960}{500} + (0.10 \times 500) = 40 + 1.92 + 50.0 = \mathbf{91.92\text{ s}} \implies S(500) = 10.88\times
  \]
  Adding 400 more processors made the code run **almost twice as slow**!

---

### Problem 2: Load Imbalance Factor Calculation
A 4-threaded OpenMP program finishes a computational phase with the following individual thread execution times:
- Thread 0: $t_0 = 12.0\text{ seconds}$
- Thread 1: $t_1 = 18.0\text{ seconds}$
- Thread 2: $t_2 = 14.0\text{ seconds}$
- Thread 3: $t_3 = 24.0\text{ seconds}$

At the end of this phase, an implicit OpenMP barrier exists.

Calculate:
1. The total wall-clock time required to complete the phase.
2. The total wasted CPU idle time summed across all threads.
3. The Average Execution Time $t_{\text{avg}}$.
4. The Load Imbalance Factor $\lambda$.
5. The Imbalance Ratio $\beta$.

#### Solution:
1. **Total Wall-Clock Time:**
   Determined strictly by the slowest thread ($t_{\max}$):
   \[
   T_{\text{phase}} = t_{\max} = \max(12, 18, 14, 24) = \mathbf{24.0\text{ seconds}}
   \]
2. **Total Wasted CPU Idle Time:**
   Each thread waits at the barrier until Thread 3 finishes:
   - Thread 0 idle stall: $24 - 12 = 12\text{ s}$
   - Thread 1 idle stall: $24 - 18 = 6\text{ s}$
   - Thread 2 idle stall: $24 - 14 = 10\text{ s}$
   - Thread 3 idle stall: $24 - 24 = 0\text{ s}$
   \[
   \text{Total Wasted Time} = 12 + 6 + 10 + 0 = \mathbf{28.0\text{ CPU-seconds}}
   \]
3. **Average Execution Time ($t_{\text{avg}}$):**
   \[
   t_{\text{avg}} = \frac{12 + 18 + 14 + 24}{4} = \frac{68}{4} = \mathbf{17.0\text{ seconds}}
   \]
4. **Load Imbalance Factor ($\lambda$):**
   \[
   \lambda = \frac{t_{\max} - t_{\text{avg}}}{t_{\max}} = \frac{24.0 - 17.0}{24.0} = \frac{7.0}{24.0} \approx 0.2917 = \mathbf{29.17\%}
   \]
5. **Imbalance Ratio ($\beta$):**
   \[
   \beta = \frac{t_{\max}}{t_{\text{avg}}} = \frac{24.0}{17.0} \approx \mathbf{1.41}
   \]

---

## 10. Conceptual Examination Questions & Answers

### Q1: Why does a real-world speedup curve typically exhibit a peak and then roll downward, unlike the monotonically increasing curve of Amdahl's Law?
**Model Answer:**
Amdahl’s Law models execution time as $T_N = s \cdot T_1 + \frac{(1-s)T_1}{N}$, which assumes zero parallel overhead ($c(N) = 0$). Under this assumption, adding processors always reduces or maintains execution time.

In physical systems, parallel execution includes an overhead function $c(N)$ representing thread synchronization, network message latency, barrier waits, and cache coherence invalidations:
\[
T(N) = s \cdot T_1 + \frac{(1-s)T_1}{N} + c(N)
\]
While the computational term $\frac{(1-s)T_1}{N}$ decreases with $N$, the overhead term $c(N)$ monotonically increases with $N$ (often as $\mathcal{O}(N)$ or $\mathcal{O}(N^2)$). 

At lower processor counts, the reduction in computational time outweighs the overhead. However, beyond an optimal processor count $N_{\text{opt}}$, the marginal overhead added by an extra processor exceeds the computational work assigned to it. Consequently, total execution time begins increasing, causing the speedup curve to roll over and degrade.

---

### Q2: How does dynamic loop scheduling help mitigate load imbalance, and what is its associated trade-off?
**Model Answer:**
In static loop scheduling, iterations are divided into fixed chunks at compile time and assigned deterministically to threads. If iterations have non-uniform computational complexity (e.g., triangular loops), threads assigned lighter iterations finish early and sit idle at the terminal barrier, producing severe load imbalance.

**Dynamic loop scheduling** mitigates this by maintaining a centralized, shared task queue at runtime. Iterations are grouped into small chunks (e.g., 16 or 64 iterations). As soon as any thread finishes its chunk, it dynamically requests the next available chunk from the queue. Fast threads automatically process more chunks, while slow threads process fewer chunks, equalizing thread completion times and driving $\lambda \to 0$.

**The Trade-off:** Dynamic scheduling introduces **runtime lock contention overhead**. Every time a thread requests a chunk, it must lock the shared work queue. If chunk sizes are too small, the synchronization overhead of accessing the queue outweighs the load-balancing benefit.

---

## 11. Quick Revision Sheet

- **Extended Execution Time Equation:**
  \[
  \boxed{T(N) = s \cdot T_1 + \frac{(1 - s) \cdot T_1}{N} + c(N)}
  \]
- **Optimal Processor Count (Linear Overhead $c(N) = \alpha N$):**
  \[
  \boxed{N_{\text{opt}} = \sqrt{\frac{(1 - s)T_1}{\alpha}}}
  \]
- **Load Imbalance Factor ($\lambda$):**
  \[
  \boxed{\lambda = \frac{t_{\max} - t_{\text{avg}}}{t_{\max}} = 1 - \frac{t_{\text{avg}}}{t_{\max}}} \quad (0 \le \lambda < 1)
  \]
- **Imbalance Ratio ($\beta$):**
  \[
  \boxed{\beta = \frac{t_{\max}}{t_{\text{avg}}}} \quad (\beta \ge 1.0)
  \]
- **The Core Rule of Barriers:**
  \[
  \boxed{\text{Total Barrier Time} = \max(t_0, t_1, \dots, t_{N-1})}
  \]
- **Common Exam Trap:**
  - *Trap:* Thinking you can eliminate load imbalance by simply adding more threads.
  - *Correction:* Adding more threads to an imbalanced static problem **worsens** efficiency because more cores spend time stalled at the barrier!

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 03.04, verify that you can independently:
1. Explain the 5 major physical sources of parallel overhead $c(N)$.
2. Derive the optimal processor count formula $N_{\text{opt}} = \sqrt{\frac{(1 - s)T_1}{\alpha}}$ using differential calculus.
3. Draw and explain the speedup roll-off curve demonstrating why performance degrades at excessive core counts.
4. Calculate the Load Imbalance Factor $\lambda$ and wasted CPU idle time from thread execution timings.
5. Explain how dynamic scheduling resolves non-uniform loop load imbalance and state its trade-offs.
