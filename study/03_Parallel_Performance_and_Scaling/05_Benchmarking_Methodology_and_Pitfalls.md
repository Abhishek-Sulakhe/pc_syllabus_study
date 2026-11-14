# Module 03.05 — Benchmarking Methodology & Performance-Measurement Pitfalls

> **Prerequisites:** Module 01.04 (Performance Metrics & FLOP/s), Module 02.04 (Memory Hierarchy & Caches), Module 03.01 (Speedup & Efficiency)  
> **Lecture Mapping:** Lectures 3–5 (Benchmarking Principles, Timer Resolution, Cold/Warm Caches, System Jitter, Measurement Pitfalls)  
> **Exam Relevance:** MidSem (Timer Selection, Wall-Clock vs CPU Time, Pitfalls Analysis) & 40% Lab Exam Core  

---

## 1. Understanding the Basic Idea

Suppose you want to time how fast a student can run a 100-meter sprint.
- You use a **sundial** from the town square.
- Before the student runs, you let them sit in the freezing cold for two hours without warming up.
- Right as the student starts running, the school principal walks across the track, forcing the student to dodge.
- The student's running shoes have a loose sole that occasionally trips them.
- You time the run **exactly once** and proclaim: *"This student runs 100 meters in 28 seconds!"*

Is that an accurate, scientific measurement of the student’s true athletic performance? **Of course not!**
1. The timer (sundial) has terrible resolution.
2. The runner wasn't warmed up.
3. Random outside noise (the principal) interfered with the experiment.
4. The equipment was faulty.
5. A single run has zero statistical significance.

Yet, in university computer science laboratories and corporate engineering teams, **programmers make these exact five mistakes every day when benchmarking software!**
- They use coarse timers like `time()` or `clock()` that measure the wrong thing.
- They fail to warm up CPU caches and power states.
- They ignore Intel Turbo Boost fluctuations and OS background daemons.
- They time a parallel code once, see an anomalous number, and draw completely false conclusions about their algorithms.

> **Benchmarking is not just typing `./program` and watching a number appear. Benchmarking is a rigorous physical science that requires controlling experimental variables, minimizing measurement noise, and ensuring reproducible results.**

---

## 2. Wall-Clock Time vs. CPU Time: The Fatal Timing Trap

The most common bug in student parallel programming labs is using the standard C library function `clock()` to measure parallel speedup.

### Why is `clock()` Fatal for Parallel Programs?
The C standard library function `clock()` returns the **total CPU time consumed by the process across all processor cores**.

```
Single-Core Run (1 Thread, 10 seconds of work):
Core 0: [ 10 seconds of CPU time ]
`clock()` reports: 10.0 seconds.

Multi-Core Run (4 Threads, 2.5 seconds of work):
Core 0: [ 2.5 s ] ┐
Core 1: [ 2.5 s ] ┼──► `clock()` SUMS ALL CORES TOGETHER!
Core 2: [ 2.5 s ] │    2.5 + 2.5 + 2.5 + 2.5 = 10.0 seconds!
Core 3: [ 2.5 s ] ┘
`clock()` reports: 10.0 seconds!
```
- If you use `clock()`, the 4-core run will report **10.0 seconds**—the exact same time as the single-core run!
- The student looks at their screen and panics: *"My parallel code has a speedup of $1.0\times$! OpenMP isn't working!"*
- In reality, the code ran in **2.5 wall-clock seconds**, delivering a perfect $4\times$ speedup, but `clock()` summed all the cores together!

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    WALL-CLOCK TIME vs. CPU TIME                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. WALL-CLOCK TIME (Real Time / Elapsed Time):
    The actual time that passes on a physical wristwatch hanging on the wall
    from the start of the program to the end.
    MUST BE USED FOR ALL PARALLEL BENCHMARKING!
    
    Timers: `omp_get_wtime()`, `clock_gettime(CLOCK_MONOTONIC)`, `MPI_Wtime()`.

 2. CPU TIME (Process User + System Time):
    The sum of all active CPU cycles spent executing code across ALL threads.
    Useful only for measuring CPU utilization and thread efficiency, NEVER speedup!
```

---

## 3. High-Resolution Precision Timers in C

To measure parallel execution scientifically, you must use timers that:
1. Measure **monotonic wall-clock time** (cannot jump backwards if network time syncs or daylight saving shifts!).
2. Have **nanosecond or microsecond resolution**.
3. Have extremely **low calling overhead** (so the act of measuring does not slow down the program).

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      STANDARD PARALLEL TIMING APIS                              │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. OpenMP Native Timer:
    double t_start = omp_get_wtime();
    // ... parallel computation ...
    double t_end = omp_get_wtime();
    double elapsed_seconds = t_end - t_start;
    [ Resolution: ~10 to 100 nanoseconds | Portability: Any OpenMP compiler ]

 2. POSIX Monotonic High-Resolution Timer:
    struct timespec ts_start, ts_end;
    clock_gettime(CLOCK_MONOTONIC, &ts_start);
    // ... parallel computation ...
    clock_gettime(CLOCK_MONOTONIC, &ts_end);
    double elapsed_seconds = (ts_end.tv_sec - ts_start.tv_sec) +
                             (ts_end.tv_nsec - ts_start.tv_nsec) / 1e9;
    [ Resolution: 1 nanosecond | Portability: Linux, macOS, Unix ]

 3. MPI Native Timer:
    double t_start = MPI_Wtime();
    // ... distributed computation ...
    double t_end = MPI_Wtime();
    double elapsed_seconds = t_end - t_start;
    [ Resolution: High precision | Portability: Any MPI implementation ]
```

---

## 4. The Six Deadly Pitfalls of Parallel Performance Measurement

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                  THE SIX DEADLY BENCHMARKING PITFALLS                           │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
     ┌──────────────────┬──────────────┴──────────────────┬─────────────────┐
     ▼                  ▼                                 ▼                 ▼
 1. COLD CACHES    2. TURBO BOOST JITTER             3. SYSTEM NOISE   4. COMPILER DEAD
    First run is      CPU downclocks as more            OS daemons,       CODE REMOVAL
    slow due to       cores get hot;                    interrupts,       Compiler deletes
    cold DRAM misses. single core boosts!               thread migration. unread results!
```

---

### Pitfall 1: Cold Caches & Page Faults (The Warmup Mandate)
When a C program starts and allocates memory via `malloc()`, physical memory pages are not yet mapped into the CPU’s page tables (due to Linux lazy page allocation). 
Furthermore, the CPU hardware caches (L1, L2, L3) are empty ("cold").
- **Run 1:** Suffers thousands of TLB page faults, compulsory cache misses, and DRAM loading stalls.
- **Run 2:** Memory pages are already mapped, cache lines are populated ("warm"), and branch predictors are trained.
- **The Solution (The Warmup Run):** Always execute a dummy "warmup" iteration before starting the measurement timer!

---

### Pitfall 2: Dynamic Voltage and Frequency Scaling (DVFS & Intel Turbo Boost)
Modern CPUs dynamically adjust their clock frequencies based on temperature and active core count:
- **Single-Core Boost:** When only 1 core is active, the CPU boosts its clock speed to **$5.0\text{ GHz}$** because the thermal budget is low.
- **All-Core Throttle:** When 16 cores are active, the CPU downclocks to **$3.2\text{ GHz}$** to prevent thermal melting!
- **The Measurement Distortion:**
  Your single-core baseline ran at $5.0\text{ GHz}$, but your 16-core parallel test ran at $3.2\text{ GHz}$! 
  You are comparing two completely different hardware frequencies, artificially making parallel speedup look worse than it really is.
- **The Solution:** For scientific benchmarking, lock CPU frequency using OS performance governors (e.g., `cpupower frequency-set -g performance`) or record frequencies during the test.

---

### Pitfall 3: System Noise and Jitter
In a multi-tasking operating system, your program is not alone.
- Background daemons, antivirus scanners, network packet interrupts, and OS kernel thread schedulers periodically wake up.
- If an OS interrupt preempts Worker Thread 3 for just 5 milliseconds, all other 15 worker threads sit frozen at the next OpenMP barrier for 5 milliseconds!
- **The Solution:** Run the benchmark multiple times (e.g., 5 to 10 trials) and take the **minimum** or **median** execution time, never just a single run.

---

### Pitfall 4: Aggressive Compiler Dead-Code Elimination
Modern optimizing compilers (GCC `-O3`) are extremely smart:
```c
double start = omp_get_wtime();

double sum = 0.0;
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; i++) {
    sum += a[i];
}

double end = omp_get_wtime();
// Notice: We never print or use 'sum'!
```
- If the compiler notices that the variable `sum` is never printed or saved to a file, **it completely deletes the entire parallel loop from the binary!**
- The execution time drops to $0.0000001\text{ seconds}$. The programmer proudly reports: *"My parallel code achieved an infinite speedup!"*
- **The Solution:** Always print a checksum or use the result in a visible `printf` or volatile variable.

---

### Pitfall 5: Timing Code That Includes File I/O
Hard drives and SSDs operate on millisecond timescales, while CPU computing operates on nanosecond timescales.
- If you wrap your timer around both disk file reading and the parallel calculation:
  \[
  T_{\text{measured}} = T_{\text{disk I/O}} + T_{\text{parallel math}}
  \]
- If reading the input file from disk takes $10\text{ seconds}$ and parallel math takes $0.5\text{ seconds}$, adding 100 processors will only reduce the math to $0.05\text{ seconds}$. Total time drops from $10.5\text{ s}$ to $10.05\text{ s}$—a microscopic speedup!
- **The Solution:** Separate setup/disk I/O from computation. Time only the computational kernel.

---

### Pitfall 6: Insufficient Problem Size (Timer Resolution & Overhead Domination)
If your computational loop runs in $50\text{ microseconds}$ ($0.00005\text{ s}$):
- Spawning an OpenMP thread team takes $10\text{ to }50\text{ microseconds}$.
- The parallelization overhead completely overwhelms the work!
- Furthermore, the timer itself may only have microsecond precision.
- **The Solution:** Ensure the benchmark workload runs for at least **$0.1\text{ to }2.0\text{ seconds}$**, or repeat the kernel inside an outer timing loop.

---

## 5. Statistical Rigor in Benchmarking: Mean, Median & Min

When you run a benchmark $K$ times, you obtain a vector of execution times: $[T_1, T_2, T_3, \dots, T_K]$. Which number should you report in an academic paper or examination?

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    CHOOSING THE RIGHT STATISTICAL METRIC                        │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. THE MINIMUM EXECUTION TIME: min(T)
    Advantage: Represents the true capability of the hardware when OS noise,
               cache warming, and background daemon interference are at their lowest.
    Standard convention in many HPC publications!

 2. THE MEDIAN EXECUTION TIME: median(T)
    Advantage: Immune to extreme outlier spikes (e.g., an antivirus scan that
               tripled one run). Best robust indicator of typical performance.

 3. THE ARITHMETIC MEAN: mean(T) with Standard Deviation (σ)
    Required when reporting real-world service level agreements (SLAs).
    Must ALWAYS be accompanied by error bars: Mean ± σ.
```

---

## 6. Practical Implementation: A Production-Grade C Benchmarking Harness

Here is a complete, production-grade C benchmarking harness implementing warmups, monotonic POSIX timing, statistical analysis (mean, min, max, standard deviation), and dead-code prevention.

### Rigorous Benchmark Harness (`benchmark_harness.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>
#include <math.h>
#include <float.h>

#define N 20000000 // 20 Million elements (~160 MB)
#define NUM_TRIALS 5

// The computational kernel to benchmark
double compute_kernel(double *a, double *b, int n) {
    double total = 0.0;
    #pragma omp parallel for reduction(+:total)
    for (int i = 0; i < n; i++) {
        b[i] = sqrt(fabs(a[i])) * 1.00001 + sin(a[i]);
        total += b[i];
    }
    return total; // Return checksum to prevent dead-code elimination!
}

int main(void) {
    printf("============================================================\n");
    printf("     SCIENTIFIC PARALLEL BENCHMARKING HARNESS (POSIX/OMP)   \n");
    printf("============================================================\n");

    // 1. Allocate memory
    double *a = (double *)malloc(N * sizeof(double));
    double *b = (double *)malloc(N * sizeof(double));
    if (!a || !b) {
        fprintf(stderr, "Allocation failed!\n");
        return 1;
    }

    // Initialize data
    for (int i = 0; i < N; i++) a[i] = (double)i * 0.001;

    printf("Problem Size         : %d elements (%.2f MB)\n", N, (N * sizeof(double) * 2.0) / (1024 * 1024));
    printf("OpenMP Active Cores  : %d\n", omp_get_max_threads());
    printf("Number of Trials     : %d\n", NUM_TRIALS);

    // -------------------------------------------------------------
    // STEP 1: WARMUP RUN (Crucial: Primes caches and locks CPU clock)
    // -------------------------------------------------------------
    printf("\nExecuting Warmup Run (Untimed)... ");
    fflush(stdout);
    volatile double warmup_checksum = compute_kernel(a, b, N);
    printf("Done. (Warmup Checksum: %.2f)\n\n", warmup_checksum);

    // -------------------------------------------------------------
    // STEP 2: REPEATED BENCHMARK TRIALS
    // -------------------------------------------------------------
    double times[NUM_TRIALS];
    double sum_times = 0.0;
    double min_time = DBL_MAX;
    double max_time = 0.0;
    volatile double final_checksum = 0.0;

    printf("Executing Timed Trials:\n");
    for (int trial = 0; trial < NUM_TRIALS; trial++) {
        double start = omp_get_wtime();

        final_checksum = compute_kernel(a, b, N);

        double end = omp_get_wtime();
        double elapsed = end - start;

        times[trial] = elapsed;
        sum_times += elapsed;

        if (elapsed < min_time) min_time = elapsed;
        if (elapsed > max_time) max_time = elapsed;

        printf("  Trial %d: %.5f seconds\n", trial + 1, elapsed);
    }

    // -------------------------------------------------------------
    // STEP 3: STATISTICAL ANALYSIS
    // -------------------------------------------------------------
    double mean_time = sum_times / NUM_TRIALS;

    // Calculate standard deviation
    double variance = 0.0;
    for (int trial = 0; trial < NUM_TRIALS; trial++) {
        variance += (times[trial] - mean_time) * (times[trial] - mean_time);
    }
    double std_dev = sqrt(variance / NUM_TRIALS);

    printf("\n------------------------------------------------------------\n");
    printf("BENCHMARK STATISTICAL RESULTS:\n");
    printf("  Minimum Time (Best)  : %.5f seconds (Preferred in HPC)\n", min_time);
    printf("  Maximum Time (Worst) : %.5f seconds\n", max_time);
    printf("  Mean Time            : %.5f seconds\n", mean_time);
    printf("  Standard Deviation   : ±%.5f seconds (Jitter: %.2f%%)\n", 
           std_dev, (std_dev / mean_time) * 100.0);
    printf("  Valid Checksum       : %.2f (Dead code successfully averted!)\n", final_checksum);
    printf("============================================================\n");

    free(a);
    free(b);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O3 -fopenmp -lm benchmark_harness.c -o benchmark_harness
./benchmark_harness
```

*Expected Output:*
```
============================================================
     SCIENTIFIC PARALLEL BENCHMARKING HARNESS (POSIX/OMP)   
============================================================
Problem Size         : 20000000 elements (305.18 MB)
OpenMP Active Cores  : 8
Number of Trials     : 5

Executing Warmup Run (Untimed)... Done. (Warmup Checksum: 1729831.42)

Executing Timed Trials:
  Trial 1: 0.04820 seconds
  Trial 2: 0.04610 seconds
  Trial 3: 0.04590 seconds
  Trial 4: 0.04600 seconds
  Trial 5: 0.04630 seconds

------------------------------------------------------------
BENCHMARK STATISTICAL RESULTS:
  Minimum Time (Best)  : 0.04590 seconds (Preferred in HPC)
  Maximum Time (Worst) : 0.04820 seconds
  Mean Time            : 0.04650 seconds
  Standard Deviation   : ±0.00086 seconds (Jitter: 1.85%)
  Valid Checksum       : 1729831.42 (Dead code successfully averted!)
============================================================
```

---

## 7. Important Comparisons

### `clock()` vs. `omp_get_wtime()` vs. `clock_gettime(CLOCK_MONOTONIC)`

| Feature | `clock()` (C Standard) | `omp_get_wtime()` (OpenMP) | `clock_gettime(CLOCK_MONOTONIC)` (POSIX) |
|:---|:---|:---|:---|
| **What It Measures** | **CPU Time** (Sum across all cores!) | **Wall-Clock Time** (Real elapsed time) | **Wall-Clock Time** (Monotonic hardware clock) |
| **Usable for Parallel Speedup?**| **FATAL ERROR! NEVER USE!** | **YES (Industry Standard)** | **YES (Industry Standard)** |
| **Typical Resolution** | Coarse ($\approx 1\text{ to }10\text{ ms}$) | High ($\approx 10\text{ to }100\text{ ns}$) | Ultra-High ($1\text{ nanosecond}$) |
| **Required Header** | `<time.h>` | `<omp.h>` | `<time.h>` |
| **Return Type** | `clock_t` (Integer ticks) | `double` (Seconds directly) | `struct timespec` (Seconds + Nanoseconds) |
| **Affected by NTP Time Syncs?**| No | No (Monotonic) | No (`CLOCK_MONOTONIC` never jumps backwards)|

---

## 8. Exam-Ready Definitions

### Wall-Clock Time (Elapsed Real Time)
> **Wall-Clock Time** is the physical duration that elapses from the start of an application or routine to its completion as measured by an independent, continuous external clock, representing the true time experienced by the user.

### CPU Time (Process Execution Time)
> **CPU Time** is the total accumulated duration spent by all processor cores actively executing instructions on behalf of a specific process, encompassing both user-mode execution and kernel system-call execution.

### Cold Cache
> A **Cold Cache** refers to the state of a microprocessor cache hierarchy prior to execution when cache lines contain no instructions or data relevant to the active application, resulting in compulsory cache misses on initial memory references.

### System Jitter (Noise)
> **System Jitter** refers to the non-deterministic variability in execution time caused by background operating system tasks, periodic hardware interrupts, context switches, and thermal throttling that interfere with parallel threads.

---

## 9. Solved Problems

### Problem 1: Detecting the `clock()` Timing Disaster
A student parallelizes an array processing algorithm on an 8-core CPU.
- When running sequentially on 1 thread, the run completes in **$8.0\text{ seconds}$** of real wall-clock time.
- When running in parallel on 8 threads, the run completes in **$1.1\text{ seconds}$** of real wall-clock time.

1. Calculate the true parallel speedup achieved using wall-clock time.
2. If the student incorrectly used `clock()` to measure execution time, predict what values of `clock()` (in seconds) will be reported for the 1-thread run and the 8-thread run (assuming near 100% core utilization and negligible overhead).
3. What incorrect speedup would `clock()` report to the student?

#### Solution:
1. **True Parallel Speedup (Wall-Clock Time):**
   \[
   S(8) = \frac{T_1}{T_8} = \frac{8.0\text{ s}}{1.1\text{ s}} \approx \mathbf{7.27\times\ (Excellent\ speedup!)}
   \]
2. **Predicted `clock()` Output:**
   Recall that `clock()` accumulates total CPU time across all threads:
   - For 1 thread running for $8.0\text{ seconds}$:
     \[
     \text{CPU Time}_1 = 1\text{ thread} \times 8.0\text{ s} = \mathbf{8.0\text{ seconds}}
     \]
   - For 8 threads running concurrently for $1.1\text{ seconds}$:
     \[
     \text{CPU Time}_8 = 8\text{ threads} \times 1.1\text{ s} = \mathbf{8.8\text{ seconds}}
     \]
     *(Notice: CPU time actually increased due to thread synchronization!)*
3. **Incorrect Speedup Reported by `clock()`:**
   \[
   S_{\text{faulty}} = \frac{\text{CPU Time}_1}{\text{CPU Time}_8} = \frac{8.0\text{ s}}{8.8\text{ s}} = \mathbf{0.91\times\ (A FALSE SLOWDOWN!)}
   \]
   *The student mistakenly believes the parallel program ran slower, when in reality it finished over $7\times$ faster!*

---

### Problem 2: Statistical Significance and Outlier Rejection
A benchmark for a parallel molecular dynamics simulation is executed for 5 consecutive trials. 
The measured wall-clock times are:
- Trial 1 (Cold run): $1.420\text{ s}$
- Trial 2: $0.810\text{ s}$
- Trial 3: $0.805\text{ s}$
- Trial 4 (OS Antivirus scan spike): $2.650\text{ s}$
- Trial 5: $0.812\text{ s}$

1. Identify the two anomalous trials and explain their physical causes.
2. Calculate the arithmetic mean of all 5 trials.
3. Calculate the median of all 5 trials.
4. Calculate the minimum execution time among the valid steady-state runs.
5. Explain why reporting the median or minimum is scientifically superior to reporting the raw arithmetic mean in this scenario.

#### Solution:
1. **Anomalous Trials:**
   - **Trial 1 ($1.420\text{ s}$):** Suffered from **Cold Cache / Page Allocation penalty**. The data was loaded into DRAM for the first time.
   - **Trial 4 ($2.650\text{ s}$):** Suffered from **System Jitter / OS preemption**. An external background process stole CPU cycles.
2. **Raw Arithmetic Mean:**
   \[
   \text{Mean} = \frac{1.420 + 0.810 + 0.805 + 2.650 + 0.812}{5} = \frac{6.497}{5} = \mathbf{1.2994\text{ seconds}}
   \]
3. **Median Calculation:**
   Sort the trials in ascending order:
   \[
   [0.805, \quad 0.810, \quad \mathbf{0.812}, \quad 1.420, \quad 2.650]
   \]
   The middle value (3rd element) is:
   \[
   \text{Median} = \mathbf{0.812\text{ seconds}}
   \]
4. **Minimum Valid Steady-State Time:**
   \[
   \text{Minimum} = \mathbf{0.805\text{ seconds}}
   \]
5. **Scientific Superiority:**
   The raw arithmetic mean ($1.30\text{ s}$) is severely distorted by the single $2.65\text{ s}$ operating system spike, overestimating the algorithm's runtime by more than **$60\%$**! The median ($0.812\text{ s}$) and minimum ($0.805\text{ s}$) accurately capture the true, repeatable algorithmic performance of the parallel code.

---

## 10. Conceptual Examination Questions & Answers

### Q1: Why must the standard C library function `clock()` NEVER be used to measure parallel speedup?
**Model Answer:**
The standard C library function `clock()` measures **CPU time**, not wall-clock elapsed time. 

CPU time is calculated as the sum of all execution cycles consumed by all active hardware threads associated with the process. In a multi-threaded parallel application running across $N$ cores, if each core executes for $T$ seconds, `clock()` accumulates the time from all $N$ cores and reports approximately $N \times T$ seconds. 

Consequently, `clock()` completely masks the time savings achieved through parallel concurrency. To measure real-world parallel speedup ($S = T_1 / T_N$), developers must exclusively use monotonic **wall-clock timers** (such as `omp_get_wtime()` or `clock_gettime(CLOCK_MONOTONIC)`), which record the true physical time elapsed from the beginning of the parallel region to its end.

---

### Q2: Explain the purpose of a "Warmup Run" in parallel benchmarking and identify two architectural phenomena it addresses.
**Model Answer:**
A **Warmup Run** is an unmeasured execution of the computational kernel performed immediately prior to starting official benchmark timing. 

It addresses two major physical and architectural phenomena:
1. **Cold Caches and Translation Lookaside Buffers (TLB):** On the initial execution pass, memory addresses have not yet been translated by the OS page tables (causing soft page faults), and CPU hardware caches (L1, L2, L3) are empty ("cold"). The initial pass forces memory allocation and populates cache lines, ensuring subsequent measured trials record pure compute throughput rather than one-time setup latency.
2. **CPU Power State Transitions (C-States / P-States):** Idle processors enter low-power sleep states (C-states) with lower clock frequencies. It takes several milliseconds for the hardware frequency governor to detect heavy load, ramp voltages, and lock clock speeds into high-performance execution states. The warmup run wakes up the hardware so that measurement occurs at stable operating frequencies.

---

## 11. Quick Revision Sheet

- **Golden Rule of Parallel Timing:**
  \[
  \boxed{\text{ALWAYS use Wall-Clock Time } (omp\_get\_wtime) \quad \mid \quad \text{NEVER use CPU Time } (clock())}
  \]
- **Recommended Timing APIs:**
  - OpenMP: `omp_get_wtime()`
  - POSIX C: `clock_gettime(CLOCK_MONOTONIC, &ts)`
  - MPI: `MPI_Wtime()`
- **The Benchmarking Checklist:**
  1. [ ] Separate disk file I/O from compute kernels.
  2. [ ] Execute an untimed **warmup run** to prime caches and CPU frequencies.
  3. [ ] Run at least **5 to 10 trials** to quantify statistical variance.
  4. [ ] Report **Minimum** or **Median** time to eliminate outlier OS jitter.
  5. [ ] Consume/print final results to prevent **compiler dead-code elimination** (`-O3`).
  6. [ ] Lock CPU frequency or verify that Intel Turbo Boost isn't skewing single-core comparisons.
- **Common Exam Trap:**
  - *Trap:* Thinking standard deviation should be zero on a computer because it is digital.
  - *Reality:* Multi-tasking operating systems introduce hardware interrupts, context switches, and thermal throttling, making computer benchmarking inherently statistical.

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 04 (Lectures 6–7: Basics of OpenMP), verify that you can independently:
1. Explain the fatal flaw of using `clock()` for multi-threaded speedup calculations.
2. Write complete C code using `omp_get_wtime()` and `clock_gettime(CLOCK_MONOTONIC)`.
3. List and explain the six major performance measurement pitfalls.
4. Explain how aggressive compiler optimization (`-O3`) can delete benchmarks and how to prevent it.
5. Calculate statistical mean, median, minimum, and standard deviation from benchmark trial results.
