# 01. OpenMP Work-Sharing Constructs: for, sections, and single

---

## 1. The Basic Idea

In OpenMP, the fundamental construct `#pragma omp parallel` solely creates a team of threads. By default, every thread in that team redundantly executes **every single line of code** inside the parallel region. If you have an array of $1{,}000{,}000$ elements and place a standard loop inside `#pragma omp parallel`, four threads will not divide the work—each thread will compute all $1{,}000{,}000$ iterations independently, resulting in $4{,}000{,}000$ total iterations and zero parallel speedup!

To actually split work among threads, OpenMP provides **Work-Sharing Constructs**. A work-sharing construct does **not** create new threads; rather, it intercepts an existing team of threads and distributes distinct portions of work across them:

1. **`#pragma omp for` (Loop Work-Sharing):** Divides the iterations of an independent `for` loop across the threads in the team.
2. **`#pragma omp sections` (Functional Work-Sharing):** Divides a fixed set of distinct, independent structured blocks (subroutines or tasks) among the threads.
3. **`#pragma omp single` (Single-Thread Execution):** Specifies that an enclosed structured block must be executed by **exactly one** thread from the team (whichever thread reaches it first), while all other threads wait at an implicit barrier.

```
                  PARALLEL REGION: TEAM OF 4 THREADS
                                   |
    +------------------------------+------------------------------+
    |                              |                              |
    v                              v                              v
[ #pragma omp for ]    [ #pragma omp sections ]      [ #pragma omp single ]
Loop Iterations [0..99]    Distinct Code Blocks         Single Thread Entry
 Thread 0: [ 0..24]       Thread 0: Section A (I/O)    Thread 1: Read Config File
 Thread 1: [25..49]       Thread 1: Section B (Calc)   Threads 0, 2, 3:
 Thread 2: [50..74]       Thread 2: Section C (Log)      WAIT AT BARRIER
 Thread 3: [75..99]       Thread 3: Idle (if done)
    |                              |                              |
    v                              v                              v
[ Implicit Barrier ]       [ Implicit Barrier ]          [ Implicit Barrier ]
```

---

## 2. Why We Need It

Without work-sharing directives, the programmer is forced to manually calculate loop bounds using thread IDs:

```c
/* MANUAL WORK-SHARING (Tedious, error-prone, non-portable) */
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    int nthreads = omp_get_num_threads();
    int chunk = N / nthreads;
    int start = tid * chunk;
    int end = (tid == nthreads - 1) ? N : start + chunk;

    for (int i = start; i < end; i++) {
        process(i);
    }
}
```

Manual partitioning has severe disadvantages:
1. **Handling Remainder Iterations:** If $N$ is not cleanly divisible by the thread count $T$ ($N \pmod T \neq 0$), the programmer must manually balance edge cases.
2. **Zero Scheduling Flexibility:** What if iteration $0$ takes $1\,\mu\text{s}$ while iteration $99$ takes $10\,\text{ms}$? Static manual slicing causes massive load imbalance.
3. **Boilerplate Pollution:** Writing manual index arithmetic clutters core scientific algorithms.

With OpenMP work-sharing, the compiler and runtime handle the indexing, remainder distribution, and scheduling automatically:

```c
/* OPENMP WORK-SHARING (Clean, robust, highly optimized) */
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    process(i);
}
```

---

## 3. How It Works: Step-by-Step Architecture and Mechanics

### 3.1 The Three Essential Rules of Work-Sharing Constructs
1. **No New Threads Created:** A work-sharing construct must be dynamically enclosed inside an active parallel region. It does not spawn threads; it borrows the existing team.
2. **Must Be Encountered by All Threads:** If a work-sharing construct is placed inside a branch (`if/else`), either **all** threads in the team must execute the construct or **none** may execute it. Threads cannot take divergent paths where some enter `#pragma omp for` and others do not (this causes deadlocks).
3. **Implicit Barrier at the End:** By default, every work-sharing construct (`for`, `sections`, `single`) terminates with an **implicit barrier**. No thread can proceed to subsequent statements until every thread in the team has completed its assigned slice of work (unless the `nowait` clause is specified).

---

### 3.2 Loop Work-Sharing: `#pragma omp for`
The `#pragma omp for` directive instructs the compiler to carve up loop iterations among the active threads.

#### Canonical Loop Form Requirements
In C/C++, a `for` loop parallelized by `#pragma omp for` must conform strictly to the **Canonical Loop Form** so the compiler can compute the total trip count before entering the loop:

```c
for (init-expr; test-expr; incr-expr) {
    /* Loop Body */
}
```

| Component | Permissible C Syntax | Invalid / Forbidden Syntax |
|:---|:---|:---|
| **`init-expr`** | `var = lb` or `type var = lb` (e.g., `int i = 0;`) | Multiple induction variables (`i = 0, j = 10;`) |
| **`test-expr`** | `i < b`, `i <= b`, `i > b`, `i >= b` | Relational inequality (`i != b`), non-linear checks |
| **`incr-expr`** | `i++`, `++i`, `i--`, `--i`, `i += step`, `i -= step`, `i = i + step` | Dynamic increments, pointer walking (`p = p->next`), multiplication (`i *= 2`) |
| **Loop Body** | Single structured block. Must not modify loop index `i`! | Modifying loop counter (`i = i + 5;` inside body) |
| **Termination** | Runs until completion. | `break` or `return` inside loop body (violates single-exit rule) |

```
   LEGAL CANONICAL LOOP:              ILLEGAL OPENMP LOOPS:
   for (int i = 0; i < N; i++)        for (int i = 0; i != N; i++)      // != not allowed
   for (int i = 100; i >= 0; i -= 2)  for (node* p = head; p; p = p->next) // pointer chasing
   for (size_t i = 0; i < count; ++i) for (int i = 0; i < N; i++) {
                                          if (arr[i] < 0) break;         // break forbidden!
                                      }
```

#### Loop Index Scoping Rule
By definition, the loop control variable (`i`) is automatically **private** to each thread. Even if `i` was declared in the enclosing outer scope before the parallel region, the OpenMP compiler forces `i` into thread-private storage so that each thread can manipulate its iteration counter without race conditions.

---

### 3.3 Functional Work-Sharing: `#pragma omp sections`
When parallel tasks are heterogeneous (different functions or algorithms that can execute concurrently, such as rendering graphics while updating physics and streaming audio), loop parallelism does not apply. Here, **sections** are used.

- `#pragma omp sections` marks the container region.
- `#pragma omp section` marks individual, independent work blocks.

```c
#pragma omp parallel
{
    #pragma omp sections
    {
        #pragma omp section
        {
            compute_physics_engine();   /* Executed by Thread X */
        }

        #pragma omp section
        {
            render_graphics_pipeline(); /* Executed by Thread Y */
        }

        #pragma omp section
        {
            process_audio_stream();     /* Executed by Thread Z */
        }
    } /* Implicit barrier: All threads wait here until all sections complete */
}
```

#### Execution Mechanics of Sections:
- If there are $S$ sections and $T$ threads:
  - If $S \le T$: Each section is assigned to a distinct thread. Remaining $T - S$ threads sit idle at the ending barrier.
  - If $S > T$: The first $T$ sections are dispatched to the $T$ threads. As soon as a thread finishes its section, it grabs the next available unprocessed section until all $S$ sections are finished.
- The assignment of sections to thread IDs is non-deterministic.

---

### 3.4 Single-Thread Execution: `#pragma omp single`
Often within a parallel region, an operation must be performed **exactly once**, rather than redundantly by all threads (e.g., reading an input file, allocating a shared array, printing a progress timestamp).

```c
#pragma omp parallel
{
    do_parallel_work_phase1();

    #pragma omp single
    {
        /* Executed by ONE thread only (whichever arrives first) */
        printf("Phase 1 complete. Initializing Phase 2 buffers...\n");
        allocate_shared_buffer();
    } /* IMPLICIT BARRIER: All other threads wait here until single block finishes! */

    do_parallel_work_phase2();
}
```

#### Vital Contrast: `single` vs. `master` (`masked`)
A universal midsem exam question is the architectural difference between `single` and `master`:

```
                 #pragma omp single                 #pragma omp master / masked
      Thread 0   Thread 1   Thread 2   Thread 3   Thread 0   Thread 1   Thread 2   Thread 3
         |          |          |          |          |          |          |          |
         |      Arrives 1st    |          |          |          |          |          |
         |      [Executes]     |          |      [Executes]     |          |          |
         |          |          |          |          |          |          |          |
      [BARRIER]  [Finishes] [BARRIER]  [BARRIER]  [NO BARRIER]  |          |          |
         |          |          |          |          |     Bypasses   Bypasses   Bypasses
         +----------+----------+----------+          |    Immediately Immediately Immediately
                    v                                v          v          v          v
```

1. **Who Executes?**
   - `single`: **Any** single thread (the first thread to arrive at runtime).
   - `master`: Strictly **Thread 0** (the master thread).
2. **Barrier Semantics:**
   - `single`: Contains an **implicit barrier** at the closing brace. Worker threads wait until the executing thread finishes.
   - `master`: Has **NO barrier**. Non-master threads skip the block and immediately continue executing subsequent code. If non-master threads depend on data modified by Thread 0 inside `master`, a severe data race occurs unless an explicit `#pragma omp barrier` is added!

---

## 4. Important Mechanisms & Combined Constructs

### 4.1 The Combined Construct: `#pragma omp parallel for`
Rather than nesting `#pragma omp for` inside an outer `#pragma omp parallel` block:

```c
/* Two-step syntax */
#pragma omp parallel
{
    #pragma omp for
    for (int i = 0; i < N; i++) { ... }
}

/* Equivalent Combined Construct (Shorthand) */
#pragma omp parallel for
for (int i = 0; i < N; i++) { ... }
```
The combined construct forks the thread team and immediately binds loop work-sharing to the team, eliminating one level of lexical indentation. It also automatically applies an implicit barrier at the end of the loop.

---

### 4.2 The `nowait` Clause
If the code following a work-sharing construct does not depend on the results produced by that construct, waiting at the implicit barrier is a waste of CPU cycles. Appending `nowait` strips away the barrier:

```c
#pragma omp parallel
{
    #pragma omp for nowait
    for (int i = 0; i < N; i++) {
        compute_independent_A(i);
    } /* Threads finish their slice and IMMEDIATELY move forward! */

    #pragma omp for
    for (int j = 0; j < N; j++) {
        compute_independent_B(j);
    } /* Has implicit barrier: all threads synchronize here */
}
```

> [!WARNING]
> You cannot attach `nowait` to a combined construct like `#pragma omp parallel for`. The reason: the barrier at the end of a parallel region is mandatory to ensure all worker threads join before the master thread resumes sequential execution. `nowait` can only be attached to `#pragma omp for`, `#pragma omp sections`, or `#pragma omp single` inside an enclosing parallel region!

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of `for`, `sections`, and `single`
Save this file as `omp_worksharing_demo.c`:

```c
/* =========================================================================
 * File: omp_worksharing_demo.c
 * Description: Demonstrates OpenMP Work-Sharing Constructs:
 *              1. #pragma omp for (Loop iteration partitioning)
 *              2. #pragma omp sections (Functional task distribution)
 *              3. #pragma omp single (Coordinated single-thread execution)
 *              4. Demonstration of implicit barrier behavior
 * Compilation:
 *   gcc -O3 -fopenmp omp_worksharing_demo.c -o omp_worksharing_demo
 * Execution:
 *   ./omp_worksharing_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define N 16
#define NUM_THREADS 4

void task_alpha(int tid) {
    printf("  [Sections] Task Alpha executed by Thread %d\n", tid);
}

void task_beta(int tid) {
    printf("  [Sections] Task Beta  executed by Thread %d\n", tid);
}

void task_gamma(int tid) {
    printf("  [Sections] Task Gamma executed by Thread %d\n", tid);
}

int main(void) {
    int data[N];
    int shared_config_val = 0;

    printf("============================================================\n");
    printf("        OpenMP Work-Sharing Constructs Demonstration        \n");
    printf("============================================================\n\n");

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(data, shared_config_val, stdout)
    {
        int tid = omp_get_thread_num();

        /* ---------------------------------------------------------
         * 1. SINGLE CONSTRUCT: Configuration / Setup
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("--- Phase 1: #pragma omp single ---\n");
            printf("  [Single] Executed exclusively by Thread %d (first to arrive)\n", tid);
            shared_config_val = 42; /* Initializing shared variable */
            printf("  [Single] Initialized shared_config_val = %d\n", shared_config_val);
            printf("  [Single] All other threads are waiting at the implicit barrier...\n\n");
        } /* IMPLICIT BARRIER: Guarantees shared_config_val is visible to all threads */

        /* ---------------------------------------------------------
         * 2. FOR CONSTRUCT: Partitioning 16 iterations across 4 threads
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("--- Phase 2: #pragma omp for (Loop Work-Sharing) ---\n");
        }

        #pragma omp for
        for (int i = 0; i < N; i++) {
            data[i] = i * shared_config_val;
            printf("  [For Loop] Thread %d processed iteration %02d (data[%02d]=%d)\n",
                   tid, i, i, data[i]);
        } /* IMPLICIT BARRIER: Guarantees data[] is fully populated before Phase 3 */

        /* ---------------------------------------------------------
         * 3. SECTIONS CONSTRUCT: Functional task distribution
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("\n--- Phase 3: #pragma omp sections (Functional Work-Sharing) ---\n");
        }

        #pragma omp sections
        {
            #pragma omp section
            {
                task_alpha(tid);
            }

            #pragma omp section
            {
                task_beta(tid);
            }

            #pragma omp section
            {
                task_gamma(tid);
            }
        } /* IMPLICIT BARRIER: All 3 tasks must finish before leaving sections */

    } /* End of Parallel Region */

    printf("\n--- All Work-Sharing Phases Complete Successfully! ---\n");
    printf("Verification: data[0]=%d, data[15]=%d\n", data[0], data[15]);
    printf("============================================================\n");

    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_worksharing_demo.c -o omp_worksharing_demo
$ ./omp_worksharing_demo
```

```text
============================================================
        OpenMP Work-Sharing Constructs Demonstration        
============================================================

--- Phase 1: #pragma omp single ---
  [Single] Executed exclusively by Thread 1 (first to arrive)
  [Single] Initialized shared_config_val = 42
  [Single] All other threads are waiting at the implicit barrier...

--- Phase 2: #pragma omp for (Loop Work-Sharing) ---
  [For Loop] Thread 0 processed iteration 00 (data[00]=0)
  [For Loop] Thread 0 processed iteration 01 (data[01]=42)
  [For Loop] Thread 0 processed iteration 02 (data[02]=84)
  [For Loop] Thread 0 processed iteration 03 (data[03]=126)
  [For Loop] Thread 1 processed iteration 04 (data[04]=168)
  [For Loop] Thread 1 processed iteration 05 (data[05]=210)
  [For Loop] Thread 1 processed iteration 06 (data[06]=252)
  [For Loop] Thread 1 processed iteration 07 (data[07]=294)
  [For Loop] Thread 2 processed iteration 08 (data[08]=336)
  [For Loop] Thread 2 processed iteration 09 (data[09]=378)
  [For Loop] Thread 2 processed iteration 10 (data[10]=420)
  [For Loop] Thread 2 processed iteration 11 (data[11]=462)
  [For Loop] Thread 3 processed iteration 12 (data[12]=504)
  [For Loop] Thread 3 processed iteration 13 (data[13]=546)
  [For Loop] Thread 3 processed iteration 14 (data[14]=588)
  [For Loop] Thread 3 processed iteration 15 (data[15]=630)

--- Phase 3: #pragma omp sections (Functional Work-Sharing) ---
  [Sections] Task Alpha executed by Thread 0
  [Sections] Task Beta  executed by Thread 2
  [Sections] Task Gamma executed by Thread 1

--- All Work-Sharing Phases Complete Successfully! ---
Verification: data[0]=0, data[15]=630
============================================================
```

Notice:
1. In Phase 1, Thread 1 arrived first and executed `single`. The other threads waited until Thread 1 finished.
2. In Phase 2, the 16 iterations were statically partitioned into blocks of 4 iterations per thread ($16 / 4 = 4$).
3. In Phase 3, the 3 sections were claimed dynamically by Threads 0, 2, and 1. Thread 3 remained idle at the barrier because there were only 3 sections.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Iteration Distribution Equations for `#pragma omp for`
When dividing $N$ iterations across $T$ threads using default static partitioning:

$$\text{Base Chunk Size } C = \left\lfloor \frac{N}{T} \right\rfloor, \qquad \text{Remainder } R = N \pmod T$$

Each thread $t \in \{0, 1, \dots, T-1\}$ is assigned an iteration range $[\text{Start}(t), \text{End}(t))$ calculated as:

$$\text{Start}(t) = t \cdot C + \min(t, R)$$

$$\text{Count}(t) = C + (1 \text{ if } t < R \text{ else } 0)$$

$$\text{End}(t) = \text{Start}(t) + \text{Count}(t)$$

#### Concrete Numerical Example:
If $N = 10$ and $T = 4$:
- $C = \lfloor 10 / 4 \rfloor = 2$
- $R = 10 \pmod 4 = 2$
- **Thread 0:** $\text{Start} = 0 \cdot 2 + \min(0,2) = 0$, $\text{Count} = 2 + 1 = 3 \implies [0, 1, 2]$
- **Thread 1:** $\text{Start} = 1 \cdot 2 + \min(1,2) = 3$, $\text{Count} = 2 + 1 = 3 \implies [3, 4, 5]$
- **Thread 2:** $\text{Start} = 2 \cdot 2 + \min(2,2) = 6$, $\text{Count} = 2 + 0 = 2 \implies [6, 7]$
- **Thread 3:** $\text{Start} = 3 \cdot 2 + \min(3,2) = 8$, $\text{Count} = 2 + 0 = 2 \implies [8, 9]$

Total iterations covered $= 3 + 3 + 2 + 2 = 10$. The workload is balanced with at most a difference of $1$ iteration among threads.

---

## 7. Advantages & Limitations

### Advantages
1. **Clean Abstraction:** Developers parallelize complex loops and pipelines in single lines of declarative code.
2. **Deterministic Partitioning:** The canonical loop form guarantees exact trip counts and deterministic iteration slicing.
3. **Automatic Synchronization:** Built-in implicit barriers prevent data races across pipeline phases without requiring explicit manual locks.

### Limitations & Traps
1. **No Dynamic Work Partitioning in Sections:** `#pragma omp sections` has static code blocks known at compile-time. You cannot dynamically scale the number of sections at runtime based on input size.
2. **Idle Thread Inefficiency in Sections:** If you have $64$ threads and only $4$ sections, $60$ threads sit completely idle, yielding disastrous parallel efficiency ($\varepsilon(64) \approx 4/64 = 6.25\%$).
3. **Loop Canonical Restrictions:** You cannot parallelize pointer-chasing while-loops (`while (curr != NULL)`) using `#pragma omp for`. (Such data structures require OpenMP Tasks, covered in Lecture 13).
4. **Deadlock from Unmatched Constructs:** Putting a work-sharing construct inside an `if` branch that only some threads take causes the remaining threads to hang forever waiting at the implicit barrier.

---

## 8. Formal Definitions for Exam

- **Work-Sharing Construct:** An OpenMP directive that distributes the execution of associated code or loop iterations among the existing members of the thread team encountering it, without spawning new threads.
- **Canonical Loop Form:** A standardized format for `for` loops in C/C++ mandated by OpenMP, characterized by a single induction variable, invariant bounds, computable trip count, and monotonic increments, enabling the compiler to partition iterations prior to loop entry.
- **Trip Count:** The total number of iterations executed by a loop, computable prior to entering the loop as $\left\lfloor \frac{\text{upper} - \text{lower} + \text{step} - 1}{\text{step}} \right\rfloor$.
- **`#pragma omp single`:** A work-sharing construct specifying that the enclosed structured block is executed by exactly one thread of the team, with all non-executing threads blocking at an implicit barrier at the end of the construct.
- **`#pragma omp sections`:** A work-sharing construct that encloses one or more `#pragma omp section` directives, distributing separate, independent blocks of code among the threads in the team.
- **`nowait` Clause:** A clause that can be appended to certain work-sharing directives (`for`, `sections`, `single`) to eliminate the mandatory implicit barrier at the end of the construct.

---

## 9. Comparison Table

### Comparison of OpenMP Work-Sharing Constructs

| Feature | `#pragma omp for` | `#pragma omp sections` | `#pragma omp single` | `#pragma omp master` (Note) |
|:---|:---:|:---:|:---:|:---:|
| **Type of Parallelism** | Data / Loop Parallelism | Functional / Task Parallelism | Coordination / Serial Block | Serial Master Block |
| **Work Unit** | Individual Loop Iteration | Discrete `#pragma omp section` | Single Structured Block | Single Structured Block |
| **Executing Entity** | Entire Thread Team | Up to $S$ threads in team | **Exactly One Thread** (any) | **Thread 0 Only** |
| **Implicit Barrier at Exit?** | **YES** | **YES** | **YES** | **NO** |
| **Accepts `nowait` clause?** | **YES** | **YES** | **YES** | **NO** (No barrier exists) |
| **Scalability with Thread Count** | Excellent (scales with $N$) | Poor (capped at section count $S$) | None (single-threaded) | None (single-threaded) |

*(Note: `master` is a synchronization/master construct, included here for contrast with `single`).*

---

## 10. Solved Analytical & Code-Tracing Problems

### Problem 1: Remainder Iteration Calculation
**Question:** A student parallelizes a loop with trip count $N = 50$ across $T = 6$ threads using `#pragma omp for` with default static chunking. Calculate:
1. The base chunk size $C$.
2. The number of threads receiving an extra iteration.
3. The exact iteration ranges assigned to Thread 0 and Thread 5.

**Solution:**
1. **Base Chunk:**
   $$C = \left\lfloor \frac{50}{6} \right\rfloor = 8$$
2. **Remainder:**
   $$R = 50 \pmod 6 = 2$$
   Therefore, exactly $R = 2$ threads (Thread 0 and Thread 1) receive $C + 1 = 9$ iterations. Threads 2, 3, 4, and 5 receive $C = 8$ iterations.
3. **Iteration Ranges:**
   - **Thread 0:** Assigned $9$ iterations $\implies \mathbf{[0..8]}$.
   - **Thread 1:** Assigned $9$ iterations $\implies \mathbf{[9..17]}$.
   - **Thread 2:** Assigned $8$ iterations $\implies [18..25]$.
   - **Thread 3:** Assigned $8$ iterations $\implies [26..33]$.
   - **Thread 4:** Assigned $8$ iterations $\implies [34..41]$.
   - **Thread 5:** Assigned $8$ iterations $\implies \mathbf{[42..49]}$.

---

### Problem 2: The Missing Barrier Bug
**Question:** What is the critical concurrency bug in the following code, and how does changing `master` to `single` resolve it?

```c
int *buffer;

#pragma omp parallel
{
    #pragma omp master
    {
        buffer = (int*)malloc(1024 * sizeof(int));
    }

    /* All threads immediately initialize buffer */
    #pragma omp for
    for (int i = 0; i < 1024; i++) {
        buffer[i] = 0; /* CRASH or SEGFAULT! */
    }
}
```

**Solution:**
1. **Root Cause:** `#pragma omp master` has **NO implicit barrier** at its closing brace.
2. **The Race Condition:** Only Thread 0 executes the `malloc()`. Threads 1, 2, and 3 skip the `master` block and immediately encounter the `#pragma omp for` loop.
3. **The Crash:** Because Thread 0 has not yet finished allocating memory, `buffer` is still uninitialized (a dangling pointer or `NULL`). Threads 1, 2, and 3 dereference `buffer[i]`, causing an immediate **Segmentation Fault** or undefined memory corruption.
4. **The Resolution:** Replace `#pragma omp master` with `#pragma omp single`.
   `#pragma omp single` contains a **mandatory implicit barrier** at the closing brace. Threads 1, 2, and 3 are blocked at the barrier until the single thread completes `malloc()`, ensuring that `buffer` is completely allocated and visible to all threads before the loop begins.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Can work-sharing constructs be nested inside one another?
**Answer:** **NO.** By default in OpenMP, you cannot nest a work-sharing construct (`#pragma omp for`, `#pragma omp sections`, or `#pragma omp single`) inside another work-sharing construct in the same parallel region. For example, placing `#pragma omp for` inside an outer `#pragma omp for` (for parallelizing nested loops) is illegal and causes a compiler error or runtime abort. To parallelize nested loops, use the `collapse` clause: `#pragma omp for collapse(2)`.

### Q2: Why does OpenMP forbid the use of `break` statements inside a `#pragma omp for` loop?
**Answer:** Because an OpenMP loop must be a structured block with a predetermined, computable trip count. When the compiler distributes iterations among threads, each thread is statically or dynamically given an assigned slice of the iteration space (e.g., Thread 1 gets iterations $25..49$). If Thread 1 executes a `break`, it would violate the single-exit contract of the structured block, leave other threads hanging or out of sync, and break the runtime's internal loop state tracking.

### Q3: When should I choose `sections` instead of `for`?
**Answer:** Use `for` when you have **homogeneous data parallelism** (the same algorithm applied across thousands or millions of array elements). Use `sections` when you have **heterogeneous task parallelism** (a small number of completely different functions that can run concurrently, such as executing a Fast Fourier Transform on Audio while concurrently decompression an image frame in Video).

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                OPENMP WORK-SHARING CONSTRUCTS                                      |
+----------------------------------------------------------------------------------------------------+
| Directive               | Purpose                         | Who Executes?    | Implicit Barrier?   |
+-------------------------+---------------------------------+------------------+---------------------+
| #pragma omp for         | Distributes loop iterations     | Team of Threads  | YES (Overridable)   |
| #pragma omp sections    | Encloses distinct task sections | Team of Threads  | YES (Overridable)   |
| #pragma omp section     | Individual functional block     | Exactly 1 Thread | NO (Enclosing does) |
| #pragma omp single      | Executes block once only        | Exactly 1 Thread | YES (Overridable)   |
| #pragma omp master      | Executes block on Thread 0 only | Thread 0 strictly| NO BARRIER!         |
+----------------------------------------------------------------------------------------------------+
|                               CANONICAL LOOP FORM RULES                                            |
+----------------------------------------------------------------------------------------------------+
| Loop Index Variable     | Automatically PRIVATE. Must be signed/unsigned integer or pointer.        |
| Trip Count Calculation  | Must be computable BEFORE loop entry: Floor((upper - lower + step - 1)/step)|
| Forbidden Statements    | break, return, goto into/out of loop body are strictly ILLEGAL.            |
| Increments Permitted    | i++, ++i, i--, --i, i += step, i -= step (Constant/invariant step).        |
+----------------------------------------------------------------------------------------------------+
|                                    CLAUSES & COMBINATIONS                                          |
+----------------------------------------------------------------------------------------------------+
| Combined Construct      | #pragma omp parallel for (Forks team + distributes loop in 1 line)        |
| nowait Clause           | Removes implicit barrier at end of for, sections, or single constructs.    |
|                         | (ILLEGAL on combined '#pragma omp parallel for').                          |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain why `#pragma omp parallel` alone does not distribute work and how work-sharing constructs solve redundant execution.
2. Verify whether a C `for` loop complies with the OpenMP Canonical Loop Form requirements.
3. Calculate the exact iteration slices assigned to each thread given $N$ iterations and $T$ threads with remainder handling.
4. Implement heterogeneous task parallelism using `#pragma omp sections` and `#pragma omp section`.
5. Clearly articulate the behavioral and barrier differences between `#pragma omp single` and `#pragma omp master`.
6. Use the `nowait` clause safely to eliminate unnecessary pipeline barriers without causing data races.
