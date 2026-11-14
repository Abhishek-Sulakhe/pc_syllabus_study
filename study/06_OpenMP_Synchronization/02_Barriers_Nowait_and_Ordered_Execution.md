# 02. OpenMP Barriers, nowait, and Ordered Execution

---

## 1. The Basic Idea

In parallel programming, threads do not advance through instructions at identical rates. Differences in CPU core frequencies, cache hits and misses, operating system interrupts, and dynamic branch predictions cause threads to drift apart in time.

When an algorithm consists of discrete **sequential phases**—where Phase 2 requires all data from Phase 1 to be completely written before any thread can safely read it—uncoordinated execution produces catastrophic **Read-After-Write (RAW) data races**.

OpenMP provides three critical primitives to coordinate thread progression across parallel phases:
1. **Barriers (`#pragma omp barrier`):** A synchronization checkpoint where every thread in the team must arrive before any thread is allowed to continue past the barrier.
2. **The `nowait` Clause:** An optimization tool that strips away the mandatory implicit barrier at the end of work-sharing constructs (`for`, `sections`, `single`), allowing fast threads to begin subsequent independent computation without idling.
3. **Ordered Execution (`#pragma omp ordered`):** A construct that allows a specific block within a parallel loop to be executed strictly in the **sequential order of loop iterations** ($0, 1, 2, \dots, N-1$), enabling sequential I/O while preserving concurrent parallel computation for the rest of the loop.

```
       PHASE COORDINATION: 4 THREADS AT AN OPENMP BARRIER

   Thread 0: ====== Compute Phase 1 ====== [ Arrived at Barrier ] ... WAITS ...
   Thread 1: === Compute Phase 1 ===       [ Arrived at Barrier ] ... WAITS ...
   Thread 2: ================= Compute 1 = [ Arrived at Barrier ] ... WAITS ...
   Thread 3: ============================ Compute 1 ============ [ ARRIVES! ]
                                                                      |
   ----------------------- ALL THREADS RELEASED TOGETHER ------------+
                                                                      |
   Thread 0: ---- Compute Phase 2 ----
   Thread 1: ---- Compute Phase 2 ----
   Thread 2: ---- Compute Phase 2 ----
   Thread 3: ---- Compute Phase 2 ----
```

---

## 2. Why We Need It

Consider a multiphase stencil computation or fluid simulation:

```c
/* FATAL DEFECT: Missing Barrier between Pipeline Phases */
#pragma omp parallel
{
    /* Phase 1: Each thread computes its slice of shared Grid A */
    #pragma omp for nowait
    for (int i = 0; i < N; i++) {
        GridA[i] = compute_step1(i);
    }

    /* Phase 2: Each thread reads neighbors from Grid A to produce Grid B */
    #pragma omp for
    for (int i = 0; i < N; i++) {
        GridB[i] = (GridA[i-1] + GridA[i] + GridA[i+1]) / 3.0; /* DATA RACE! */
    }
}
```

If the barrier at the end of Phase 1 is removed with `nowait`:
- Thread 0 may rush ahead into Phase 2 and attempt to read `GridA[1]`, which is supposed to be calculated by Thread 1 in Phase 1.
- Because Thread 1 is still busy working on Phase 1, Thread 0 reads stale, uninitialized, or partially updated data from `GridA[1]`.
- The simulation produces completely incorrect physical results.

An **explicit barrier** or preserving the **implicit barrier** guarantees that Phase 1 is globally complete across all processor cores before any core begins Phase 2.

Conversely, if Phase 2 updates an entirely independent array `ArrayC` with no dependencies on `GridA`, forcing threads to wait at an implicit barrier is a pure waste of CPU cycles. Appending `nowait` removes the barrier, achieving optimal pipeline overlap.

---

## 3. How It Works: Step-by-Step Architecture and Mechanics

### 3.1 Taxonomy of OpenMP Barriers
OpenMP strictly distinguishes between **Implicit Barriers** and **Explicit Barriers**:

```
+----------------------------------------------------------------------------------------------------+
|                                    OPENMP BARRIER TAXONOMY                                         |
+----------------------------------------------------------------------------------------------------+
| CONSTRUCT                                  | HAS IMPLICIT BARRIER AT EXIT? | SUPPORTS nowait CLAUSE?|
+--------------------------------------------+-------------------------------+------------------------+
| #pragma omp parallel                       | **YES (MANDATORY)**           | **NO (FORBIDDEN)**     |
| #pragma omp for                            | **YES**                       | **YES**                |
| #pragma omp sections                       | **YES**                       | **YES**                |
| #pragma omp single                         | **YES**                       | **YES**                |
| #pragma omp master / masked                | **NO BARRIER AT ALL**         | N/A                    |
| #pragma omp critical                       | **NO BARRIER AT ALL**         | N/A                    |
| #pragma omp atomic                         | **NO BARRIER AT ALL**         | N/A                    |
| #pragma omp ordered                        | **NO BARRIER AT ALL**         | N/A                    |
+----------------------------------------------------------------------------------------------------+
```

#### Explicit Barrier Syntax:
```c
#pragma omp barrier
```
- When a thread reaches this directive, it halts execution until **all** threads in the current team reach the barrier.
- **Rule of Universal Encounter:** If `#pragma omp barrier` is inside a conditional branch (`if/else`), either **every thread in the team** must enter that branch and encounter the barrier, or **none may encounter it**. If only some threads hit the barrier, the program deadlocks forever!

---

### 3.2 Underlying Hardware Implementation of Barriers
How does the OpenMP runtime implement `#pragma omp barrier` at the silicon level?

#### 1. Centralized Sense-Reversing Barrier ($O(p)$ Contention):
- A shared atomic arrival counter is decremented by each arriving thread using `lock dec`.
- Threads spin-wait on a shared `sense` flag in cache.
- The last arriving thread flips the `sense` flag, releasing all waiting threads simultaneously.
- *Limitation:* On high core counts ($p > 32$), all cores spinning on a single cache line causes memory bus saturation.

#### 2. Dissemination / Tree Barrier ($O(\log p)$ Latency):
- On modern supercomputing runtimes (Intel oneAPI, LLVM), threads form a radix tree or tournament tree.
- Core 0 synchronizes with Core 1; Core 2 synchronizes with Core 3. Their parent nodes then synchronize in an inverted binary or 4-ary tree.
- Arrival and notification require only $\lceil \log_2 p \rceil$ communication rounds, completely eliminating bus saturation.

```
                  TREE-BASED TOURNAMENT BARRIER (p = 8)
  Round 1: [T0 <-> T1]   [T2 <-> T3]   [T4 <-> T5]   [T6 <-> T7]
                 \           /               \           /
  Round 2:       [T0 <-> T2]                   [T4 <-> T6]
                       \                           /
  Round 3:                     [T0 <-> T4]  (Root: All Arrived!)
                                    |
                         (Wakeup Broadcast Down Tree)
```

---

### 3.3 The `nowait` Clause: Overriding Implicit Barriers
When attached to a work-sharing construct:
```c
#pragma omp for nowait
```
The compiler strips away the call to `GOMP_barrier()` or `__kmpc_barrier()` at the closing brace.

#### Contrast: `single` vs. `single nowait`
This is one of the most frequently examined concepts in university midsem papers:

```c
/* ==================== SCENARIO A: #pragma omp single ==================== */
#pragma omp single
{
    read_input_file();
} /* IMPLICIT BARRIER: All other threads wait here until read_input_file() finishes */
do_next_phase();

/* ================== SCENARIO B: #pragma omp single nowait =============== */
#pragma omp single nowait
{
    write_log_message(); /* Thread 0 (or whichever arrives first) writes log */
} /* NO BARRIER: All other threads skip this and IMMEDIATELY do_independent_work()! */
do_independent_work();
```
- In **Scenario A**, `single` acts as a synchronized gateway: no thread can proceed until the file is loaded.
- In **Scenario B**, `single nowait` allows non-executing threads to bypass the block with zero delay, running `do_independent_work()` concurrently while the single thread writes the log.

---

### 3.4 Ordered Execution: `#pragma omp ordered`
By default, iterations of a parallel loop execute in arbitrary, non-deterministic order. If a loop must perform an action that strictly requires sequential progression—such as writing frames of a video to disk, inserting ordered records into a file, or updating a sequential recurrence—OpenMP provides the **`ordered` construct**.

#### Strict Two-Step Protocol:
1. **Step 1: Declare capability on loop construct:** You **must** append the `ordered` clause to the enclosing `#pragma omp for`:
   ```c
   #pragma omp for ordered
   ```
2. **Step 2: Enclose sequential statement:** Wrap the sequential statement inside `#pragma omp ordered`:
   ```c
   #pragma omp ordered
   {
       printf("Sequential output for index %d\n", i);
   }
   ```

```
       #pragma omp for ordered: PARALLEL COMPUTE + SEQUENTIAL COMMIT

   Core 0 (Iter 0): [=== HEAVY PARALLEL COMPUTE ===] -> [ ORDERED BLOCK 0 ]
   Core 1 (Iter 1): [=== HEAVY PARALLEL COMPUTE ===] -> ... WAITS ... -> [ ORDERED BLOCK 1 ]
   Core 2 (Iter 2): [=== HEAVY PARALLEL COMPUTE ===] -> ... WAITS ...... WAITS ... -> [ ORDERED 2 ]
   Core 3 (Iter 3): [=== HEAVY PARALLEL COMPUTE ===] -> ... WAITS .................. -> [ ORDERED 3 ]
```

All heavy computation is performed in parallel across all cores. Only the lightweight ordered block is serialized, executing strictly in index sequence $0, 1, 2, 3, \dots, N-1$.

---

## 4. Important Mechanisms & Restrictions

### 4.1 Permitted Locations and Clauses Matrix

| Construct | Permits Explicit Barrier? | Permits `nowait`? | Requires `ordered` Clause on Loop? |
|:---|:---:|:---:|:---:|
| `#pragma omp parallel` | Inside `{ ... }` | **FORBIDDEN** | N/A |
| `#pragma omp for` | Inside loop body | **YES** | Yes, if using `#pragma omp ordered` |
| `#pragma omp sections` | Inside section body | **YES** | N/A |
| `#pragma omp single` | Inside block | **YES** | N/A |
| `#pragma omp master` | Inside block | N/A (Has no barrier) | N/A |

### 4.2 Deadlock Pitfalls with Barriers
A barrier forces every thread in the team to arrive. If a barrier is enclosed inside a conditional statement:

```c
/* FATAL DEADLOCK BUG */
#pragma omp parallel num_threads(4)
{
    int tid = omp_get_thread_num();
    if (tid % 2 == 0) {
        /* Only Threads 0 and 2 arrive here! */
        #pragma omp barrier /* DEADLOCK! */
    } else {
        /* Threads 1 and 3 never see the barrier! */
        do_work();
    }
}
```
Threads 0 and 2 arrive at the barrier and wait for Threads 1 and 3. But Threads 1 and 3 take the `else` branch and never arrive. The process hangs forever (**Deadlock**).

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of Barriers, `nowait`, and `ordered`
Save this file as `omp_barriers_ordered_demo.c`:

```c
/* =========================================================================
 * File: omp_barriers_ordered_demo.c
 * Description: Demonstrates:
 *              1. Multiphase coordination using #pragma omp barrier
 *              2. Barrier elimination and pipeline overlap via nowait
 *              3. single vs single nowait mechanics
 *              4. Strictly sequenced output via #pragma omp ordered
 * Compilation:
 *   gcc -O3 -fopenmp omp_barriers_ordered_demo.c -o omp_barriers_ordered_demo
 * Execution:
 *   ./omp_barriers_ordered_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <omp.h>

#define N 8
#define NUM_THREADS 4

int main(void) {
    printf("============================================================\n");
    printf("    OpenMP Synchronization: Barriers, nowait, and ordered   \n");
    printf("============================================================\n\n");

    int shared_buffer[N];

    /* -------------------------------------------------------------
     * DEMONSTRATION 1: Explicit Barrier in Multiphase Algorithm
     * ------------------------------------------------------------- */
    printf("--- 1. Multiphase Coordination using #pragma omp barrier ---\n");

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(shared_buffer, stdout)
    {
        int tid = omp_get_thread_num();

        /* Phase 1: Initialize shared array */
        #pragma omp for
        for (int i = 0; i < N; i++) {
            shared_buffer[i] = (i + 1) * 10;
        } /* Implicit barrier here guarantees all indices are written */

        /* Explicit thread coordination check */
        printf("  [Thread %d] Completed Phase 1 writes.\n", tid);

        #pragma omp barrier /* Explicit barrier: All threads report before Phase 2 */

        if (tid == 0) {
            printf("  -> Master: All threads cleared explicit barrier. Starting Phase 2.\n");
        }

        #pragma omp barrier

        /* Phase 2: Stencil read from neighbor */
        #pragma omp for
        for (int i = 0; i < N; i++) {
            int left_neighbor = (i == 0) ? shared_buffer[N-1] : shared_buffer[i-1];
            int right_neighbor = (i == N-1) ? shared_buffer[0] : shared_buffer[i+1];
            int smooth_val = (left_neighbor + shared_buffer[i] + right_neighbor) / 3;
            printf("  [Phase 2] Iter %d: left=%d, self=%d, right=%d -> Smooth=%d\n",
                   i, left_neighbor, shared_buffer[i], right_neighbor, smooth_val);
        }
    }
    printf("\n");

    /* -------------------------------------------------------------
     * DEMONSTRATION 2: Overriding Barriers with nowait
     * ------------------------------------------------------------- */
    printf("--- 2. Pipeline Overlapping using nowait Clause ---\n");

    #pragma omp parallel num_threads(NUM_THREADS) default(none) shared(stdout)
    {
        int tid = omp_get_thread_num();

        #pragma omp for nowait
        for (int i = 0; i < 4; i++) {
            printf("  [Loop A (nowait)] Thread %d finished task A[%d]\n", tid, i);
        }
        /* NO BARRIER! Threads finish Loop A and immediately jump into Loop B */

        #pragma omp for
        for (int j = 0; j < 4; j++) {
            printf("  [Loop B] Thread %d executing task B[%d]\n", tid, j);
        }
    }
    printf("\n");

    /* -------------------------------------------------------------
     * DEMONSTRATION 3: Strictly Ordered Execution (#pragma omp ordered)
     * ------------------------------------------------------------- */
    printf("--- 3. Ordered Sequential Commit via #pragma omp ordered ---\n");
    printf("Notice: Iterations are computed concurrently, but printed strictly 0 to 7:\n");

    #pragma omp parallel for num_threads(NUM_THREADS) ordered schedule(dynamic, 1)
    for (int i = 0; i < N; i++) {
        int tid = omp_get_thread_num();

        /* 1. Heavy Concurrent Computation (Simulated) */
        int computed_result = i * i * 3;

        /* 2. Sequential Ordered Block: Strictly executed in order 0, 1, 2, ... */
        #pragma omp ordered
        {
            printf("  [ORDERED BLOCK] Iteration %02d (computed by Thread %d) -> Result = %3d\n",
                   i, tid, computed_result);
        }
    }

    printf("\n============================================================\n");
    printf("Demonstrations Complete.\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_barriers_ordered_demo.c -o omp_barriers_ordered_demo
$ ./omp_barriers_ordered_demo
```

```text
============================================================
    OpenMP Synchronization: Barriers, nowait, and ordered   
============================================================

--- 1. Multiphase Coordination using #pragma omp barrier ---
  [Thread 0] Completed Phase 1 writes.
  [Thread 2] Completed Phase 1 writes.
  [Thread 1] Completed Phase 1 writes.
  [Thread 3] Completed Phase 1 writes.
  -> Master: All threads cleared explicit barrier. Starting Phase 2.
  [Phase 2] Iter 0: left=80, self=10, right=20 -> Smooth=36
  [Phase 2] Iter 1: left=10, self=20, right=30 -> Smooth=20
  [Phase 2] Iter 2: left=20, self=30, right=40 -> Smooth=30
  [Phase 2] Iter 3: left=30, self=40, right=50 -> Smooth=40
  [Phase 2] Iter 4: left=40, self=50, right=60 -> Smooth=50
  [Phase 2] Iter 5: left=50, self=60, right=70 -> Smooth=60
  [Phase 2] Iter 6: left=60, self=70, right=80 -> Smooth=70
  [Phase 2] Iter 7: left=70, self=80, right=10 -> Smooth=53

--- 2. Pipeline Overlapping using nowait Clause ---
  [Loop A (nowait)] Thread 0 finished task A[0]
  [Loop B] Thread 0 executing task B[0]  <-- Thread 0 moved into Loop B without waiting!
  [Loop A (nowait)] Thread 1 finished task A[1]
  [Loop A (nowait)] Thread 2 finished task A[2]
  [Loop B] Thread 1 executing task B[1]
  [Loop A (nowait)] Thread 3 finished task A[3]
  [Loop B] Thread 2 executing task B[2]
  [Loop B] Thread 3 executing task B[3]

--- 3. Ordered Sequential Commit via #pragma omp ordered ---
Notice: Iterations are computed concurrently, but printed strictly 0 to 7:
  [ORDERED BLOCK] Iteration 00 (computed by Thread 0) -> Result =   0
  [ORDERED BLOCK] Iteration 01 (computed by Thread 2) -> Result =   3
  [ORDERED BLOCK] Iteration 02 (computed by Thread 1) -> Result =  12
  [ORDERED BLOCK] Iteration 03 (computed by Thread 3) -> Result =  27
  [ORDERED BLOCK] Iteration 04 (computed by Thread 0) -> Result =  48
  [ORDERED BLOCK] Iteration 05 (computed by Thread 2) -> Result =  75
  [ORDERED BLOCK] Iteration 06 (computed by Thread 1) -> Result = 108
  [ORDERED BLOCK] Iteration 07 (computed by Thread 3) -> Result = 147

============================================================
Demonstrations Complete.
```

### Critical Observations:
1. In Section 1, the explicit `#pragma omp barrier` prevented any thread from entering Phase 2 until all threads reported that Phase 1 was complete.
2. In Section 2 with `nowait`, Thread 0 finished `A[0]` and immediately jumped into `B[0]` before other threads had even finished Loop A.
3. In Section 3, despite using `schedule(dynamic, 1)` where threads finished out of order, the outputs were emitted in **strictly monotonic sequential order ($00 \to 07$)**.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Barrier Synchronization Latency Model
The time spent at a barrier by a team of $p$ threads can be divided into **Wait Time** (due to load imbalance) and **Synchronization Latency** (due to barrier communication):

$$T_{\text{barrier\_total}} = \Delta T_{\text{wait}} + T_{\text{sync}}(p)$$

Where:
- $\Delta T_{\text{wait}} = \max_{t} (T_{\text{work}}(t)) - \overline{T_{\text{work}}}$
- $T_{\text{sync}}(p)$ depends on the barrier implementation:
  - **Sense-Reversing Linear Barrier:**
    $$T_{\text{sync}}(p) = O(p) \cdot \tau_{\text{cache\_bounce}}$$
  - **Tournament Tree Barrier:**
    $$T_{\text{sync}}(p) = O(\log_2 p) \cdot \tau_{\text{hop}}$$

---

### 6.2 The Cost of `ordered` on Amdahl's Law
If an `ordered` section consumes time $T_{\text{ordered}}$ per iteration, the total time of the loop across $N$ iterations cannot be less than:

$$T_{\text{loop}}(p) \ge N \cdot T_{\text{ordered}}$$

No matter how many threads $p$ are added, the ordered section forms an irreducible serial bottleneck of length $N \cdot T_{\text{ordered}}$. Therefore:
> [!IMPORTANT]
> Keep code inside `#pragma omp ordered` blocks as small as possible (e.g., a simple buffer store or pointer increment). Perform all heavy math, formatting, and calculations outside the ordered block.

---

## 7. Advantages & Limitations

### Barriers
- **Advantages:** Essential for pipeline phase correctness; guarantees memory consistency across all cores.
- **Limitations:** Wastes CPU cycles if threads are imbalanced; potential for deadlock if placed in divergent branches.

### `nowait` Clause
- **Advantages:** Eliminates idle stall time; enables full pipeline saturation across independent loops.
- **Limitations:** High risk of data races if the programmer miscalculates data dependencies across loops.

### Ordered Execution
- **Advantages:** Preserves sequential output/logging/commit order in parallel loops without converting the entire loop to serial.
- **Limitations:** Serializes the ordered portion across all threads; destroys parallel scalability if the ordered block is large.

---

## 8. Formal Definitions for Exam

- **Barrier:** A synchronization primitive that halts the execution of every thread in a thread team until all threads in that team have reached the barrier directive.
- **Implicit Barrier:** A barrier automatically placed by the OpenMP compiler and runtime at the conclusion of certain directives (`parallel`, `for`, `sections`, `single`).
- **Explicit Barrier:** A barrier manually inserted into program code by the developer using the `#pragma omp barrier` directive.
- **Rule of Universal Encounter:** The OpenMP requirement stating that if a barrier directive is encountered by any thread in a team, it must be encountered by all threads in that team; otherwise deadlock ensues.
- **`nowait` Clause:** An OpenMP clause applicable to `#pragma omp for`, `#pragma omp sections`, and `#pragma omp single` that removes the default implicit barrier from the end of the construct.
- **Ordered Construct:** An OpenMP construct (`#pragma omp ordered`) used inside an ordered loop that enforces execution of its enclosed structured block in the exact sequential order of loop iterations.

---

## 9. Comparison Table

### Comparison: Explicit Barrier vs `nowait` vs Ordered

| Feature | `#pragma omp barrier` | `nowait` Clause | `#pragma omp ordered` |
|:---|:---:|:---:|:---:|
| **Primary Purpose** | Global team phase synchronization | Eliminates synchronization stall | Enforces sequential execution order |
| **Placement** | Standalone executable directive | Clause on `for`, `sections`, `single` | Block inside loop annotated with `ordered` |
| **Execution by Threads** | **All threads in team** | Modifies construct exit behavior | Executed one thread at a time in loop order |
| **Risk of Deadlock** | High if placed inside divergent `if` | None (Risk is Data Race instead) | None (Runtime maintains ordered queue) |
| **Impact on Performance** | Increases idle wait time | **Improves throughput & overlap** | Introduces serial execution bottleneck |

---

## 10. Solved Analytical Problems

### Problem 1: Determining Legality of `nowait`
**Question:** A student writes the following OpenMP code. Will it compile? If not, identify the error.

```c
#pragma omp parallel for nowait
for (int i = 0; i < N; i++) {
    process(i);
}
```

**Solution:**
- **Verdict: COMPILATION ERROR.**
- **Reason:** In OpenMP, `nowait` is **forbidden** on combined constructs like `#pragma omp parallel for`.
- **Architectural Rationale:** The closing brace of `#pragma omp parallel` marks the end of the parallel region where worker threads must join or be reclaimed by the runtime pool, and the master thread alone must resume sequential execution. Bypassing the barrier of a parallel region would allow the master thread to execute post-parallel sequential code while worker threads are still running, violating the fundamental Fork-Join execution model.
- **Fix:** If `nowait` is desired, the parallel region and loop work-sharing construct must be separated:
  ```c
  #pragma omp parallel
  {
      #pragma omp for nowait
      for (int i = 0; i < N; i++) {
          process(i);
      }
      do_other_work();
  }
  ```

---

### Problem 2: Missing `ordered` Clause on Loop
**Question:** Explain what happens when the compiler compiles the following code:

```c
#pragma omp parallel for
for (int i = 0; i < 10; i++) {
    #pragma omp ordered
    {
        printf("%d\n", i);
    }
}
```

**Solution:**
- **Verdict: COMPILATION ERROR.**
- **Reason:** An OpenMP `#pragma omp ordered` block can only be placed inside a loop construct that has been explicitly declared with the `ordered` clause on the loop directive.
- **Fix:** Change the loop header to:
  ```c
  #pragma omp parallel for ordered
  ```
- **Architectural Justification:** The OpenMP runtime must allocate internal tracking structures (such as a sequential iteration token or ticket lock) to manage ordered commits. Adding `ordered` to the loop header instructs the compiler to generate this tracking framework. Without it, the loop generator omits the tracking mechanism, making `#pragma omp ordered` undefined.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why doesn't `#pragma omp master` have an implicit barrier, while `#pragma omp single` does?
**Answer:**
- `#pragma omp master` is designed for quick administrative tasks strictly by Thread 0 (such as updating a counter or printing a status line) where other threads are assumed to have independent work they can perform immediately without waiting.
- `#pragma omp single` is designed for cooperative work-sharing where **any** thread can execute the block, but subsequent code typically relies on the result of that single execution (e.g., reading a shared config file). Hence, OpenMP mandates an implicit barrier on `single` for safety, while allowing developers to opt-out via `nowait`.

### Q2: What happens if two threads in a team call `#pragma omp barrier` in different order?
**Answer:** Threads do not have an "order" at a barrier. A barrier is a collective rendezvous point. The first $T-1$ threads to arrive block and wait; the $T$-th thread to arrive triggers the release of all $T$ threads. Order of arrival is irrelevant as long as **all** $T$ threads arrive.

### Q3: When is using `nowait` on a `#pragma omp for` loop $100\%$ safe?
**Answer:** It is safe if and only if **no variable written in Loop A is read or written in subsequent code before the next barrier**. In compiler terminology: there must be no **RAW (Read-After-Write)**, **WAR (Write-After-Read)**, or **WAW (Write-After-Write)** dependencies between the iterations of Loop A and any subsequent operations executed by other threads.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    BARRIER & NOWAIT CHEAT SHEET                                     |
+----------------------------------------------------------------------------------------------------+
| Explicit Barrier       | #pragma omp barrier (All threads in team MUST arrive; no divergence!)     |
| Implicit Barriers      | At end of: parallel, for, sections, single.                               |
| NO Implicit Barriers   | At end of: master, atomic, critical, ordered.                             |
| nowait Clause          | Removes implicit barrier from: for, sections, single.                      |
| nowait Forbidden On    | #pragma omp parallel AND #pragma omp parallel for                          |
| single vs single nowait| single = all threads wait at exit. single nowait = other threads run ahead|
+----------------------------------------------------------------------------------------------------+
|                                     ORDERED EXECUTION RULES                                        |
+----------------------------------------------------------------------------------------------------+
| Loop Requirement       | Loop MUST declare ordered clause: #pragma omp for ordered                 |
| Block Syntax           | #pragma omp ordered { ... }                                               |
| Execution Guarantee    | Executes in strictly sequential iteration index order: 0, 1, 2, ..., N-1.  |
| Best Practice          | Keep ordered blocks MINIMAL (I/O, commits only). Compute outside!         |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the operational difference between implicit and explicit barriers.
2. Identify code regions where implicit barriers naturally exist versus where they are absent (`single` vs `master`).
3. Diagnose and fix deadlocks caused by placing `#pragma omp barrier` inside divergent conditional branches.
4. Safely apply the `nowait` clause to overlap parallel loops without introducing data races.
5. Implement sequenced parallel output using the two-step protocol of `#pragma omp for ordered` and `#pragma omp ordered`.
6. Quantify the synchronization latency of linear vs tree-based barrier implementations.
