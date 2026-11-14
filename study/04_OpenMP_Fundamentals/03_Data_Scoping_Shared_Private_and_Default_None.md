# 03. Data Scoping: Shared, Private, and default(none)

---

## 1. The Basic Idea

In a multithreaded OpenMP program, all threads belong to a single operating system process and inhabit a common, unified virtual address space. However, having access to the same physical RAM does not mean that every variable should be accessed concurrently by all threads.

**Data Scoping** is the mechanism by which OpenMP defines the accessibility, lifetime, storage location, and visibility of every variable referenced inside a parallel region:

1. **Shared Variables (`shared`):** A single memory location in RAM exists. All threads read and write to this identical physical address via pointers or shared references.
2. **Private Variables (`private`):** Each thread in the team allocates its own separate, independent storage location (typically on its private execution stack). Changes made by thread $k$ are completely invisible to thread $j$.
3. **`default(none)`:** A defensive compiler directive that disables all automatic scoping assumptions, forcing the programmer to explicitly state the scope (`shared`, `private`, `firstprivate`, or `reduction`) for every variable referenced inside the region.

```
+-----------------------------------------------------------------------------------+
|                        PROCESS VIRTUAL ADDRESS SPACE                              |
|                                                                                   |
|  [GLOBAL DATA & HEAP]                                                             |
|    int shared_total = 0; <----+----------+----------+ (All threads access same    |
|                               |          |          |  physical memory location)  |
|                               |          |          |                             |
|  [THREAD STACKS]              |          |          |                             |
|   +-------------------+       |          |          |                             |
|   | Thread 0 Stack    |-------+          |          |                             |
|   |   int local_i;    | (Private to T0)  |          |                             |
|   +-------------------+                  |          |                             |
|   | Thread 1 Stack    |------------------+          |                             |
|   |   int local_i;    | (Private to T1)             |                             |
|   +-------------------+                             |                             |
|   | Thread 2 Stack    |-----------------------------+                             |
|   |   int local_i;    | (Private to T2)                                           |
|   +-------------------+                                                           |
+-----------------------------------------------------------------------------------+
```

Mastering data scoping is the single most critical skill in shared-memory parallel programming. Over $90\%$ of concurrency bugs in student and industrial OpenMP programs are caused by improper variable scoping.

---

## 2. Why We Need It

Consider a simple loop calculating intermediate results:

```c
/* FATAL DEFECTIVE CODE: Implicitly Shared Loop Variable */
int i;
int temp;
#pragma omp parallel
{
    /* All threads read and write to the same 'temp' and 'i' concurrently! */
    temp = heavy_computation(omp_get_thread_num());
    printf("Thread %d computed %d\n", omp_get_thread_num(), temp);
}
```

If multiple threads run this parallel region:
- When Thread 0 writes `temp = 10`, Thread 1 simultaneously writes `temp = 99`.
- Before Thread 0 can print its value, Thread 1 overwrites the memory at `&temp`.
- Thread 0 reads the overwritten value, producing corrupted calculations and non-deterministic logic failures.

This catastrophic concurrency failure is called a **Data Race**.

### Why Not Make Everything Private?
If all variables were private:
- Threads could not communicate intermediate results or coordinate work.
- Giant data arrays (such as an $8\,\text{GB}$ matrix) would be duplicated $T$ times, instantly exhausting system memory ($8\,\text{GB} \times 64\text{ threads} = 512\,\text{GB}$ RAM requirement).
- Aggregating final results back to the master thread would be impossible.

Therefore, parallel programming requires a precise, mathematically sound division: **large read-only or coordinated data structures must be Shared**, while **loop index counters, scratchpad registers, and intermediate buffers must be Private**.

---

## 3. How It Works: Step-by-Step Architecture and Storage Layout

### 3.1 Where Do Variables Actually Live?
In the C memory model on modern operating systems, virtual memory is partitioned into distinct segments:

| Memory Segment | Location & Properties | Default OpenMP Scope |
|:---|:---|:---:|
| **Global / BSS Segment** | File-scope static variables and global variables (`static int x;` or `int g_var;`). Accessible throughout process lifetime. | **Shared** |
| **Heap Segment** | Dynamically allocated memory via `malloc()`, `calloc()`, or C++ `new`. Managed by pointer references. | **Shared** (Points to same heap blocks) |
| **Master Thread Stack** | Variables declared in `main()` or enclosing function before the parallel region. | **Shared** (by default) |
| **Worker Thread Stacks** | Private stack frame created by OS for each spawned thread. Variables declared *inside* `{ ... }` of parallel region. | **Private** (strictly private to thread) |

```
                       PROCESS MEMORY LAYOUT
  Low Address  +---------------------------------------+
               | Text (Executable Code)                |
               +---------------------------------------+
               | Data / BSS (Global & Static Variables)| ---> SHARED
               +---------------------------------------+
               | Heap (malloc / new buffers)           | ---> SHARED
               |   |                                   |
               |   v  (grows downward)                 |
               |                                       |
               |   ^  (grows upward)                   |
               |   |                                   |
               | Thread 0 Stack (Master Stack)         | ---> Holds outer frame vars
               +---------------------------------------+
               | Thread 1 Private Stack                | ---> Holds inner block vars
               +---------------------------------------+
               | Thread 2 Private Stack                | ---> Holds inner block vars
               +---------------------------------------+
               | Thread (T-1) Private Stack            | ---> Holds inner block vars
  High Address +---------------------------------------+
```

---

### 3.2 Mechanism of the `shared` Clause
When a variable is declared `shared(x)`:
1. The master thread retains the original variable $x$ at its original address $\text{Addr}(x)$.
2. When the compiler outlines the parallel block into `.omp_fn`, it passes a pointer `&x` inside the activation context structure.
3. Every thread dereferences `*(&x)` directly.
4. **Hardware Reality:** Reads and writes go directly through each CPU core's L1/L2 cache. If two cores write to $x$ simultaneously without synchronization, a cache line invalidation storm (cache coherence protocol thrashing) and a hardware data race occur.

---

### 3.3 Mechanism of the `private` Clause
When a variable $y$ declared outside the parallel region is marked `private(y)`:
1. The compiler creates a **new, uninitialized local variable of identical type** on each thread's private stack frame:
   $$\forall\, t \in \{0, 1, \dots, T-1\}, \quad y_t \in \text{Stack}_t$$
2. **The Undefined Initialization Trap:**
   The newly allocated $y_t$ contains **random stack garbage**. It does NOT inherit the value that $y$ had immediately prior to entering the parallel region!
3. **The Undefined Termination Trap:**
   When the parallel region terminates at the closing brace `}`, the thread stack frames are unwound. The original variable $y$ outside the construct has an **unspecified / undefined value** (or retains its pre-region value in most implementations). The updates made by threads do NOT write back to the outer $y$.

```
BEFORE REGION:
  int y = 100; (Located at Master Stack: 0x7fff00)

INSIDE REGION with private(y):
  Thread 0: y_0 at 0x7fff00 (Value: GARBAGE, e.g. -142981)
  Thread 1: y_1 at 0x7ffb40 (Value: GARBAGE, e.g. 32767)
  Thread 2: y_2 at 0x7ff780 (Value: GARBAGE, e.g. 0)
  Thread 3: y_3 at 0x7ff3c0 (Value: GARBAGE, e.g. 4198421)

AFTER REGION:
  y at 0x7fff00 (Value: 100 or UNDEFINED, thread changes are LOST!)
```

> [!WARNING]
> If a thread needs the variable to be initialized with the value from before the parallel region, `private` **cannot** be used! You must use `firstprivate` (covered in Lecture 9).

---

### 3.4 The Danger of Default Scoping Rules
If no scoping clause is provided, OpenMP applies implicit default scoping:
1. Variables declared **inside** the parallel construct are **private** to each thread.
2. Variables declared **outside** the parallel construct are **shared** by default.
3. Loop iteration variables for work-sharing loops (`for (int i=0; ...)` or `#pragma omp for`) are automatically made **private**.
4. Const-qualified variables with no mutable members are **shared**.

**Why Implicit Scoping is Dangerous:**
If a developer forgets to declare an intermediate scratchpad variable inside the block or forgets to mark it private, the compiler silently treats it as `shared`. The program will compile without a single warning, pass quick single-thread tests, and then randomly produce wrong numerical answers or crash under production multi-threaded loads.

---

### 3.5 The Gold Standard: `default(none)`
To eliminate implicit scoping errors at compile time, OpenMP provides the `default(none)` clause:

```c
#pragma omp parallel default(none) shared(A, B, N) private(tid, temp)
{
    /* If ANY variable used here is not explicitly listed, 
       the compiler aborts with a compilation ERROR! */
}
```

With `default(none)`:
- The compiler acts as an automated static analyzer.
- If you forget to categorize even a single variable (e.g., a counter `k`), the compiler halts immediately:
  ```
  error: 'k' not specified in enclosing 'parallel'
  error: enclosing 'parallel'
  ```
- You are forced to consciously decide whether every variable is `shared` or `private`.
- **Exam Rule:** In university examinations and professional HPC projects, omitting `default(none)` is considered poor software engineering practice.

---

## 4. Important Mechanisms & Variants

### 4.1 C/C++ Supported `default` Clauses
Unlike Fortran (which supports `default(private)`, `default(firstprivate)`, etc.), the C/C++ OpenMP specification explicitly permits only two variants for `default`:

```c
/* Variant 1: Strict Defensive Scoping (Standard HPC Best Practice) */
#pragma omp parallel default(none) ...

/* Variant 2: Implicit Shared Scoping (OpenMP Default Behavior) */
#pragma omp parallel default(shared) ...
```

In OpenMP 5.0+, `default(firstprivate)` was added to C/C++, but for university exams and foundational HPC, `default(none)` and `default(shared)` are the standard syllabus targets.

---

### 4.2 Lifetime and Scope Rules Matrix

| Variable Declaration Point | In Enclosing Function (Before Region) | Inside Parallel Region `{ ... }` | Static / File Scope Outside Functions | Dynamically Allocated (`malloc`) |
|:---|:---:|:---:|:---:|:---:|
| **Physical Memory** | Master Stack Frame | Calling Thread Stack Frame | BSS / Data Segment | Process Heap |
| **Implicit Scope** | **Shared** | **Private** | **Shared** | **Shared** (Heap blocks) |
| **Can be made `shared`?** | Yes | No (Local to stack) | Yes | Yes |
| **Can be made `private`?** | Yes | Already Private | Yes | Pointer can; heap block cannot |
| **Initial Value inside region** | Retained if `shared`; **Garbage** if `private` | **Garbage** until assigned | Retained | Retained |

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of Data Race, Private Scoping, and `default(none)`
This comprehensive program illustrates three scenarios:
1. **Flawed Region:** A data race caused by an inadvertently shared variable.
2. **Corrected Region:** Clean isolation using explicit `private` scoping.
3. **Defensive Region:** Using `default(none)` to guarantee zero un-scoped variables.

Save this file as `omp_data_scoping.c`:

```c
/* =========================================================================
 * File: omp_data_scoping.c
 * Description: In-depth demonstration of OpenMP data scoping:
 *              - Shared variable data races
 *              - Private variable allocation and stack isolation
 *              - Uninitialized private variable hazards
 *              - Strict compile-time safety using default(none)
 * Compilation:
 *   gcc -O2 -fopenmp omp_data_scoping.c -o omp_data_scoping
 * Execution:
 *   ./omp_data_scoping
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define NUM_THREADS 4

int main(void) {
    printf("============================================================\n");
    printf("       OpenMP Data Scoping: Shared vs. Private Analysis     \n");
    printf("============================================================\n\n");

    /* -------------------------------------------------------------
     * SCENARIO 1: The Catastrophic Implicitly Shared Variable
     * ------------------------------------------------------------- */
    printf("--- SCENARIO 1: Race Condition on Inadvertently Shared Variable ---\n");
    int shared_scratchpad = 0;

    #pragma omp parallel num_threads(NUM_THREADS) default(shared)
    {
        int tid = omp_get_thread_num();
        
        /* Each thread attempts to store its own thread ID */
        shared_scratchpad = tid * 100;

        /* Artificial micro-delay to encourage thread pre-emption and race */
        for (volatile int delay = 0; delay < 1000000; delay++);

        /* Read back value: Will it still be (tid * 100)? Rarely! */
        printf("  [Thread %d] Wrote %d, but read back: %d %s\n",
               tid, tid * 100, shared_scratchpad,
               (shared_scratchpad == tid * 100) ? "(MATCH)" : "--> [CORRUPTED by Race!]");
    }

    printf("\n--- SCENARIO 1 Complete ---\n\n");

    /* -------------------------------------------------------------
     * SCENARIO 2: Correct Isolation with private() & The Uninitialized Trap
     * ------------------------------------------------------------- */
    printf("--- SCENARIO 2: Explicit private() and Uninitialized Storage ---\n");
    int isolated_var = 777; /* Outer value */

    printf("[Outer Scope] Before parallel region: isolated_var = %d (Addr: %p)\n",
           isolated_var, (void*)&isolated_var);

    #pragma omp parallel num_threads(NUM_THREADS) private(isolated_var)
    {
        int tid = omp_get_thread_num();

        /* ATTEMPTING TO READ isolated_var HERE WITHOUT ASSIGNING IS UNDEFINED!
         * We print its raw garbage value to demonstrate uninitialized allocation. */
        int raw_stack_val = isolated_var;

        /* Explicitly initialize on private thread stack */
        isolated_var = (tid + 1) * 10;

        printf("  [Thread %d] Raw garbage was: %11d | Assigned: %2d | Private Addr: %p\n",
               tid, raw_stack_val, isolated_var, (void*)&isolated_var);
    }

    /* Inspect value after the parallel region terminates */
    printf("[Outer Scope] After parallel region: isolated_var = %d (Addr: %p)\n",
           isolated_var, (void*)&isolated_var);
    printf("  -> Notice: Outer variable retains 777. Private updates were NOT written back!\n\n");

    /* -------------------------------------------------------------
     * SCENARIO 3: Professional Best Practice: default(none)
     * ------------------------------------------------------------- */
    printf("--- SCENARIO 3: Compile-Time Verification with default(none) ---\n");
    
    const int array_size = 8;
    int shared_array[8] = {1, 2, 3, 4, 5, 6, 7, 8};
    int total_processed = 0;

    /* Every single variable (shared_array, total_processed, array_size) 
     * MUST be categorized explicitly, or the compiler will halt compilation. */
    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(shared_array, total_processed, stdout) \
                         firstprivate(array_size)
    {
        /* Variables declared locally inside structured block are automatically private */
        int tid = omp_get_thread_num();
        int local_chunk = array_size / omp_get_num_threads();
        int start = tid * local_chunk;
        int end = start + local_chunk;

        int local_sum = 0;
        for (int i = start; i < end; i++) {
            local_sum += shared_array[i];
        }

        #pragma omp critical
        {
            total_processed += local_sum;
            printf("  [Thread %d] Processed slice [%d..%d], local_sum = %2d\n",
                   tid, start, end - 1, local_sum);
        }
    }

    printf("[Result] Total processed sum across all threads = %d (Expected: 36)\n",
           total_processed);
    printf("============================================================\n");

    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O2 -fopenmp omp_data_scoping.c -o omp_data_scoping
$ ./omp_data_scoping
```

```text
============================================================
       OpenMP Data Scoping: Shared vs. Private Analysis     
============================================================

--- SCENARIO 1: Race Condition on Inadvertently Shared Variable ---
  [Thread 0] Wrote 0, but read back: 300 --> [CORRUPTED by Race!]
  [Thread 2] Wrote 200, but read back: 300 --> [CORRUPTED by Race!]
  [Thread 1] Wrote 100, but read back: 300 --> [CORRUPTED by Race!]
  [Thread 3] Wrote 300, but read back: 300 (MATCH)

--- SCENARIO 1 Complete ---

--- SCENARIO 2: Explicit private() and Uninitialized Storage ---
[Outer Scope] Before parallel region: isolated_var = 777 (Addr: 0x7ffd5bf784b0)
  [Thread 0] Raw garbage was:       32767 | Assigned: 10 | Private Addr: 0x7ffd5bf784b0
  [Thread 1] Raw garbage was:           0 | Assigned: 20 | Private Addr: 0x7f4e917fd6f0
  [Thread 3] Raw garbage was:  1428918272 | Assigned: 40 | Private Addr: 0x7f4e907fb6f0
  [Thread 2] Raw garbage was:           0 | Assigned: 30 | Private Addr: 0x7f4e90ffc6f0
[Outer Scope] After parallel region: isolated_var = 777 (Addr: 0x7ffd5bf784b0)
  -> Notice: Outer variable retains 777. Private updates were NOT written back!

--- SCENARIO 3: Compile-Time Verification with default(none) ---
  [Thread 0] Processed slice [0..1], local_sum =  3
  [Thread 1] Processed slice [2..3], local_sum =  7
  [Thread 2] Processed slice [4..5], local_sum = 11
  [Thread 3] Processed slice [6..7], local_sum = 15
[Result] Total processed sum across all threads = 36 (Expected: 36)
============================================================
```

#### What happens if you miss a variable under `default(none)`?
Suppose you remove `shared_array` from the clause list in Scenario 3:
```bash
$ gcc -O2 -fopenmp test_error.c
test_error.c: In function 'main':
test_error.c:85:26: error: 'shared_array' not specified in enclosing 'parallel'
   85 |             local_sum += shared_array[i];
      |                          ^~~~~~~~~~~~
test_error.c:75:13: note: enclosed here
   75 |     #pragma omp parallel num_threads(NUM_THREADS) default(none) ...
      |             ^~~
```
The compiler catches the bug during compilation before the executable is ever built!

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Memory Footprint Scaling: Shared vs. Private
Consider a problem with data arrays of size $M$ elements, each requiring $S$ bytes (e.g., $M = 10^7$ doubles $\implies 80\,\text{MB}$):

$$\text{Memory}_{\text{shared}} = S \cdot M + O(p \cdot \text{sizeof(pointer)})$$

For shared scoping, regardless of whether $p = 1$ or $p = 1024$ threads participate, memory consumption remains constant at $80\,\text{MB}$ (plus tiny 8-byte pointer offsets per thread).

$$\text{Memory}_{\text{private}} = p \cdot (S \cdot M)$$

If the data array is declared `private`, total memory scales linearly with thread count $p$:
- For $p = 1$: $80\,\text{MB}$
- For $p = 64$: $5.12\,\text{GB}$
- For $p = 256$: $20.48\,\text{GB}$

> [!CAUTION]
> Declaring large arrays or buffers as `private` can quickly cause **Stack Overflow** crashes (`Segmentation Fault`), because default thread stack sizes on Linux are often constrained to $2\,\text{MB}$ or $8\,\text{MB}$ (`ulimit -s`).

---

## 7. Advantages & Limitations

### Comparison of Scoping Clauses

| Feature | `shared(x)` | `private(x)` | `default(none)` |
|:---|:---|:---|:---|
| **Memory Allocation** | Single address in RAM / Heap. | $p$ distinct allocations on thread stacks. | N/A (Compile-time meta-policy). |
| **Initial Value** | Retains value from sequential code. | **Undefined (Stack Garbage).** | N/A |
| **Final Value after Region** | Preserves latest written value. | **Outer variable value unchanged/undefined.** | N/A |
| **Risk of Data Race** | **Very High** (Requires locks/atomic). | **Zero** (Completely isolated). | Prevents un-scoped races. |
| **Memory Consumption** | $O(1)$ memory growth with $p$. | $O(p)$ memory growth with $p$. | Zero runtime memory cost. |
| **Compiler Overhead** | Negligible. | Stack allocation prologue/epilogue. | Minor compile-time AST check. |

---

## 8. Formal Definitions for Exam

- **Data Scoping:** The set of rules and OpenMP clauses that govern the accessibility, memory allocation location, and lifetime of variables across threads during the execution of a parallel region.
- **Shared Variable:** A variable whose memory location is shared among all threads in a thread team, such that any thread can read or modify the single physical address.
- **Private Variable:** A variable for which each thread in the team allocates an independent, private storage instance on its local stack, inaccessible to any other thread.
- **Data Race:** A concurrency flaw that occurs when two or more concurrent threads access the same memory location without synchronization, and at least one of the accesses is a write.
- **`default(none)` Clause:** A compiler directive that disables default scoping rules, enforcing that every variable referenced within the lexical or dynamic extent of a parallel region must be explicitly scoped by the programmer.
- **Variable Shadowing:** In OpenMP, the condition where a private variable declared in a parallel region masks or shadows an outer variable of the same name in the enclosing scope.

---

## 9. Comparison Tables

### Default Scoping Rules in C/C++ OpenMP

| Variable Category | Scope Default | Notes / Justification |
|:---|:---:|:---|
| Global / Static Variables | **Shared** | Reside in BSS/Data segments; shared across process. |
| Heap memory (`malloc`) | **Shared** | Allocated in process heap; pointers can be private, data is shared. |
| Automatic variables declared **outside** parallel region | **Shared** | Master stack frame; shared unless placed in `private` clause. |
| Automatic variables declared **inside** parallel region | **Private** | Created on individual thread's stack frame upon execution. |
| Loop index of `#pragma omp for` | **Private** | Required so threads can increment counter independently. |
| Const-qualified variables | **Shared** | Read-only; cannot cause data races. |

---

## 10. Solved Analytical & Code-Tracing Problems

### Problem 1: Code Trace with `private` Scoping
**Question:** Predict the output printed by the following C program when executed with 2 threads. Assume thread 0 and thread 1 run to completion.

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    int a = 10;
    int b = 20;

    #pragma omp parallel num_threads(2) private(a) shared(b)
    {
        int tid = omp_get_thread_num();
        a = tid + 1;
        #pragma omp atomic
        b += a;
    }

    printf("a = %d, b = %d\n", a, b);
    return 0;
}
```

**Step-by-Step Analysis:**
1. **Variable `a`:**
   - Declared `private(a)`.
   - Each thread receives a private copy of `a` on its local stack.
   - The outer variable `a` in `main()` retains its original value of $10$.
   - Thread 0 sets its private $a_0 = 0 + 1 = 1$.
   - Thread 1 sets its private $a_1 = 1 + 1 = 2$.
   - At the end of the region, the private copies $a_0$ and $a_1$ are destroyed.
   - In `main()`, the outer `a` is still **$10$**.
2. **Variable `b`:**
   - Declared `shared(b)`.
   - Initial value $b = 20$.
   - Thread 0 performs `b += 1` atomically $\implies b = 21$.
   - Thread 1 performs `b += 2` atomically $\implies b = 23$.
   - Total accumulation into shared $b$: $20 + 1 + 2 = 23$.
3. **Final Printed Output:**
   ```text
   a = 10, b = 23
   ```

---

### Problem 2: Identify the Hidden Bug
**Question:** Explain what is wrong with the following loop and describe two distinct ways to fix it.

```c
int i, n = 1000;
double factor = 2.5;
double result[1000];

#pragma omp parallel for
for (i = 0; i < n; i++) {
    double temp = i * factor;
    result[i] = temp * temp;
}
```

**Solution:**
- **Analysis:**
  - `i` is the loop index of `#pragma omp for`, so OpenMP automatically scopes `i` as **private**.
  - `n` and `factor` are read-only, so their default **shared** scoping is safe.
  - `temp` is declared *inside* the loop body `{ ... }`. Therefore, each thread allocates `temp` on its own private stack.
  - `result` is an array; each iteration writes to a unique index `result[i]`. There is no overlap among iterations.
- **Verdict:** This code is actually **CORRECT and thread-safe**!
- **Contrast with Buggy Pattern:** If `temp` had been declared *outside* the loop:
  ```c
  double temp; /* Declared outside! */
  #pragma omp parallel for
  for (i = 0; i < n; i++) {
      temp = i * factor;        /* RACE CONDITION ON temp! */
      result[i] = temp * temp;
  }
  ```
  Here `temp` is implicitly shared, causing a severe data race.
- **Two Fixes for the Buggy Pattern:**
  1. *Fix 1 (Preferred):* Declare `double temp` inside the loop body (limiting scope to iteration).
  2. *Fix 2:* Add the clause `private(temp)` to the directive: `#pragma omp parallel for private(temp)`.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why does a private variable contain garbage instead of copying the value from before the parallel region?
**Answer:** **Performance.** In high-performance computing, copying memory incurs CPU cycles and memory bandwidth. If OpenMP mandated copying outer values into every thread's private stack by default, parallelizing a loop with a large struct or array would suffer massive latency penalties even if the thread only intended to overwrite the variable. If you truly require the initial value to be copied, OpenMP explicitly provides the `firstprivate` clause.

### Q2: What happens to a private variable's value after the parallel region finishes?
**Answer:** The value of the original variable outside the parallel region is **unspecified or unchanged**. When the threads reach the closing brace of the parallel region, worker thread stack frames are destroyed. The master thread's stack variable was shadowed during the parallel region and was never written to. Therefore, you cannot use a `private` clause to extract results from a parallel region.

### Q3: Why should I always use `default(none)` in university exams and HPC assignments?
**Answer:** Because it provides **automated verification against silent bugs**:
1. Without `default(none)`, any variable you accidentally omit defaults to `shared`, silently creating data races that are impossible to detect with simple test cases.
2. With `default(none)`, the compiler refuses to build the binary until you have explicitly verified the intended concurrency semantics of every single variable.
3. Exam evaluators specifically grade whether a student understands shared vs. private memory partitioning; using `default(none)` demonstrates professional competence.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    OPENMP DATA SCOPING RULES                                       |
+----------------------------------------------------------------------------------------------------+
| Clause                 | Storage Location      | Initial Value         | Value After Region        |
+------------------------+-----------------------+-----------------------+---------------------------+
| shared(var)            | Shared RAM / Master   | Retained from outer   | Retains latest write      |
| private(var)           | Each Thread's Stack   | UNDEFINED (Garbage)   | Outer var unchanged       |
| firstprivate(var)      | Each Thread's Stack   | Copied from outer     | Outer var unchanged       |
| default(none)          | Compiler Meta-Directive: Forces explicit declaration of all variables     |
| default(shared)        | Default OpenMP C/C++ behavior: Unlisted outer variables become shared     |
+----------------------------------------------------------------------------------------------------+
|                                   AUTOMATIC DEFAULT SCOPES                                         |
+----------------------------------------------------------------------------------------------------+
| Declared INSIDE { }    | Automatically PRIVATE to each thread stack                                |
| Declared OUTSIDE { }   | Automatically SHARED across all threads                                   |
| Loop index in omp for  | Automatically PRIVATE                                                     |
| Static / Global vars   | Always SHARED (Reside in Data/BSS segment)                                 |
| Heap data (malloc)     | Always SHARED across process                                              |
+----------------------------------------------------------------------------------------------------+
|                                     CRITICAL WARNING                                               |
+----------------------------------------------------------------------------------------------------+
| Stack Overflow Risk    | Large private arrays can exceed default thread stack limits (ulimit -s).  |
| Data Race Hazard       | Any shared variable written by >= 1 thread without locks causes data race! |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic, you should be able to:
1. Explain the architectural and memory differences between `shared` and `private` variables down to stack frames and heap layout.
2. Predict the exact output and variable values before, during, and after parallel regions containing `shared` and `private` clauses.
3. Diagnose and fix data races caused by improperly scoped intermediate scratchpad variables.
4. Apply `default(none)` defensively in any OpenMP parallel construct and satisfy compiler requirements by scoping every variable correctly.
5. Explain why `private` variables contain undefined garbage upon entry and why their updates do not persist after the region ends.
6. Identify the memory scaling implications ($O(1)$ vs $O(p)$) of shared versus private data structures.
