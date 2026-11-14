# 02. OpenMP Task and Taskloop Constructs: Distribution, grainsize, and num_tasks

---

## 1. The Basic Idea

In the preceding study file, we explored why task parallelism is essential for irregular, recursive, and pointer-chasing algorithms. While individual `#pragma omp task` directives give the programmer fine-grained control over individual asynchronous blocks, using them to parallelize loops manually requires writing complex chunking wrappers.

To bridge the gap between high-level loop iteration spaces and dynamic work-stealing task schedulers, OpenMP 4.5 introduced the **`#pragma omp taskloop`** construct.

`#pragma omp taskloop` takes a canonical `for` loop and, instead of statically dividing iterations among the fixed thread team (as `#pragma omp for` does), it **slices the iteration space into a collection of independent OpenMP Tasks** and deposits them into the runtime task pool:

```c
#pragma omp taskloop grainsize(100)
for (int i = 0; i < 10000; i++) {
    process(i);
}
```

This combines the clean syntax of loop parallelism with the dynamic load balancing, task-nesting composability, and asynchronous work-stealing capabilities of the OpenMP task runtime.

```
       #pragma omp for                     #pragma omp taskloop grainsize(G)
 (Static/Dynamic Thread Slicing)             (Task Pool Work-Stealing)

   Thread 0: [=== Block 0 ===]             [ Task 0: 100 iters ]   [ Task 2: 100 iters ]
   Thread 1: [=== Block 1 ===]                       \                      /
   Thread 2: [=== Block 2 ===]                        v                    v
   Thread 3: [=== Block 3 ===]                   +-------------------------------+
                                                 |       OPENMP TASK POOL        |
   (Bound to static team threads)                +-------------------------------+
                                                       /         |          \
                                                      v          v           v
                                                  Thread 0    Thread 1    Thread 2
                                                 (Idle threads steal tasks on demand)
```

---

## 2. Why We Need It

Consider an application where a large recursive task (such as evaluating a chess board or solving an irregular octree) contains an internal loop:

```c
void evaluate_subproblem(OctreeNode *node) {
    /* If we use #pragma omp for here, it conflicts with outer tasking! */
    for (int i = 0; i < node->num_elements; i++) {
        compute(node->elements[i]);
    }
}
```

If we attempted to use `#pragma omp for` inside `evaluate_subproblem`:
- `#pragma omp for` is an OpenMP work-sharing construct. Under OpenMP specifications, work-sharing constructs **cannot be bound inside orphan task regions** without spawning nested parallel teams (`#pragma omp parallel`), which introduces massive OS thread creation overhead.
- Threads that are busy waiting at an implicit barrier inside `#pragma omp for` are prohibited from stealing and executing outer tasks.

### The `taskloop` Solution
`#pragma omp taskloop` generates tasks that seamlessly integrate into the **same unified work-stealing scheduler**:
1. If worker threads are busy processing other subtrees, the current thread executes its own loop tasks locally.
2. If other cores finish their subtrees early, they steal loop tasks from this octree node, maintaining $100\%$ CPU utilization.
3. Fine-grained control over task generation overhead is achieved via **`grainsize`** or **`num_tasks`**.

---

## 3. How It Works: Mechanics of `task` and `taskloop`

### 3.1 Advanced Clauses for `#pragma omp task`
An OpenMP task can be tuned using specialized clauses that control its execution lifecycle:

| Clause | Purpose & Behavioral Impact | Silicon / Runtime Effect |
|:---|:---|:---|
| **`untied`** | Allows the task to be suspended at a task scheduling point and **resumed by a different thread**. | Maximizes core utilization during deep I/O or child waits. (Default is `tied`, where the suspending thread must finish the task). |
| **`priority(p)`** | Hints to the runtime that tasks with higher integer priority $p$ should be scheduled before lower priority tasks. | Placed near the head of task deques. |
| **`final(expr)`** | If `expr` evaluates to true, all subsequent descendant tasks created by this task are generated as **undeferred, included tasks** (executed immediately inline). | Essential for recursive cutoff thresholds to prevent task memory explosion. |
| **`mergeable`** | Permits the runtime to avoid generating a full task descriptor, merging the data environment directly into the parent. | Optimizes away task allocation overhead. |

---

### 3.2 Task Synchronization: `taskwait` vs. `taskgroup`
OpenMP provides two distinct mechanisms for synchronizing tasks:

```
                  taskwait                           taskgroup
           (Shallow Synchronization)           (Deep Synchronization)
           
                 Parent Task                        Parent Task
                /           \                      /           \
               v             v                    v             v
          [Child A]       [Child B]          [Child A]       [Child B]
          /       \                          /       \
         v         v                        v         v
     [Grand1]   [Grand2]                [Grand1]   [Grand2]
         |         |                        |         |
     +---+---------+                    +---+---------+---------+
     |                                  |                       |
     v                                  v                       v
#pragma omp taskwait               #pragma omp taskgroup
(Waits ONLY for Child A & B.       (Waits for Child A, Child B,
 Grandchildren may still run!)      Grand1, and Grand2 completely!)
```

1. **`#pragma omp taskwait` (Shallow Wait):**
   Suspends the current task until all **direct (immediate) child tasks** have completed. Descendant tasks spawned by those children are **not** guaranteed to be complete!
2. **`#pragma omp taskgroup` (Deep Wait):**
   Encloses a structured block. The parent thread halts at the closing brace until **all child tasks AND all their recursive descendants** generated within the group have completely finished.

---

### 3.3 Partitioning the Loop: `grainsize` vs. `num_tasks`
When using `#pragma omp taskloop`, how many tasks are created? OpenMP provides two **mutually exclusive** clauses:

#### 1. The `grainsize(G)` Clause:
Specifies the number of loop iterations bundled into each generated task:
```c
#pragma omp taskloop grainsize(G)
```
- OpenMP guarantees that each generated task contains **at least $\lceil G/2 \rceil$ and at most $2 \cdot G$ iterations** (typically between $G$ and $2G-1$).
- The total number of tasks generated scales automatically with the problem size:
  $$\text{Number of Tasks } K \approx \left\lceil \frac{N}{G} \right\rceil$$

#### 2. The `num_tasks(K)` Clause:
Specifies the total number of tasks created to cover the loop:
```c
#pragma omp taskloop num_tasks(K)
```
- The loop iteration space is carved into approximately $K$ tasks.
- Each task contains approximately $\lceil N/K \rceil$ iterations.

> [!WARNING]
> You **cannot** specify both `grainsize` and `num_tasks` on the same `#pragma omp taskloop` directive! They are mutually exclusive. Specifying both causes an immediate compile-time syntax error.

---

### 3.4 Barrier Semantics of `taskloop`: The `nogroup` Clause
By default, `#pragma omp taskloop` implicitly wraps the entire loop in an internal `#pragma omp taskgroup`. 
Therefore, **the thread that encounters `taskloop` waits at the end of the loop until all generated tasks have finished executing**.

If you wish to dispatch the tasks asynchronously and immediately move forward without waiting:
```c
#pragma omp taskloop nogroup
for (int i = 0; i < N; i++) { ... }
/* Thread does NOT wait! It immediately continues executing subsequent code */
```

---

## 4. Architectural Comparison: `#pragma omp for` vs. `#pragma omp taskloop`

| Dimension | `#pragma omp for` | `#pragma omp taskloop` |
|:---|:---|:---|
| **Underlying Entity** | Iterations partitioned among Team Threads | Iterations packaged into **Independent Tasks** |
| **Scheduler Algorithm** | Static chunks / Dynamic queue | **Work-Stealing Deque Pool** |
| **Can be called inside a task?**| **NO** (Orphan work-sharing forbidden) | **YES (Fully composable)** |
| **Can be nested recursively?**| **NO** (Requires complex nested teams) | **YES (Effortless dynamic nesting)** |
| **Tuning Parameter** | `schedule(type, chunk)` | `grainsize(G)` or `num_tasks(K)` |
| **Default Exit Barrier** | Implicit thread barrier | Implicit **taskgroup** barrier (overridden by `nogroup`) |

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of `taskloop`, `grainsize`, `num_tasks`, and `taskgroup`
Save this file as `omp_taskloop_demo.c`:

```c
/* =========================================================================
 * File: omp_taskloop_demo.c
 * Description: Demonstrates:
 *              1. #pragma omp taskloop with grainsize
 *              2. #pragma omp taskloop with num_tasks
 *              3. Asynchronous dispatch via nogroup
 *              4. Deep synchronization via #pragma omp taskgroup
 * Compilation:
 *   gcc -O3 -fopenmp omp_taskloop_demo.c -o omp_taskloop_demo
 * Execution:
 *   ./omp_taskloop_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define N 32
#define NUM_THREADS 4

int main(void) {
    printf("============================================================\n");
    printf("   OpenMP Taskloop: grainsize, num_tasks, and taskgroup     \n");
    printf("============================================================\n\n");

    int results[N];
    for (int i = 0; i < N; i++) results[i] = 0;

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(results, stdout)
    {
        /* ---------------------------------------------------------
         * 1. TASKLOOP WITH grainsize(8)
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("--- 1. taskloop with grainsize(8) (N = %d) ---\n", N);
            printf("  Master Thread %d dispatching loop...\n", omp_get_thread_num());

            /* Iterations partitioned into tasks containing ~8 iterations each */
            #pragma omp taskloop grainsize(8)
            for (int i = 0; i < N; i++) {
                results[i] = i * 2;
                if (i % 8 == 0) {
                    printf("  [Task Chunk] Thread %d executing slice starting at index %02d\n",
                           omp_get_thread_num(), i);
                }
            }
            /* Implicit taskgroup barrier: All tasks guaranteed finished here! */
            printf("  -> Phase 1 complete. All grainsize tasks synchronized.\n\n");
        }

        /* ---------------------------------------------------------
         * 2. TASKLOOP WITH num_tasks(4)
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("--- 2. taskloop with num_tasks(4) (N = %d) ---\n", N);

            /* Slices loop into exactly 4 tasks (N/4 = 8 iterations per task) */
            #pragma omp taskloop num_tasks(4)
            for (int i = 0; i < N; i++) {
                results[i] += 100;
                if (i % 8 == 0) {
                    printf("  [Task Chunk] Thread %d executing Task containing index %02d\n",
                           omp_get_thread_num(), i);
                }
            }
            printf("  -> Phase 2 complete. All num_tasks tasks synchronized.\n\n");
        }

        /* ---------------------------------------------------------
         * 3. TASKGROUP SYNCHRONIZATION WITH nogroup
         * --------------------------------------------------------- */
        #pragma omp single
        {
            printf("--- 3. Enclosing Multiple Asynchronous taskloops in a taskgroup ---\n");

            #pragma omp taskgroup
            {
                /* Loop A dispatched asynchronously */
                #pragma omp taskloop nogroup
                for (int i = 0; i < 16; i++) {
                    results[i] += 500;
                }

                /* Loop B dispatched asynchronously */
                #pragma omp taskloop nogroup
                for (int j = 16; j < 32; j++) {
                    results[j] += 500;
                }

                printf("  Master thread launched both taskloops with nogroup!\n");
                printf("  Waiting at closing brace of taskgroup for all tasks to finish...\n");
            } /* taskgroup barrier ensures BOTH loops are 100% complete */

            printf("  -> Phase 3 complete. Both asynchronous taskloops verified.\n");
        }

    } /* End of parallel region */

    /* Verification */
    printf("\n[Verification] Sample results:\n");
    printf("  results[0]  = %d (Expected: 0*2 + 100 + 500 = 600)\n", results[0]);
    printf("  results[31] = %d (Expected: 31*2 + 100 + 500 = 662)\n", results[31]);
    printf("============================================================\n");

    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_taskloop_demo.c -o omp_taskloop_demo
$ ./omp_taskloop_demo
```

```text
============================================================
   OpenMP Taskloop: grainsize, num_tasks, and taskgroup     
============================================================

--- 1. taskloop with grainsize(8) (N = 32) ---
  Master Thread 0 dispatching loop...
  [Task Chunk] Thread 0 executing slice starting at index 00
  [Task Chunk] Thread 2 executing slice starting at index 16
  [Task Chunk] Thread 1 executing slice starting at index 08
  [Task Chunk] Thread 3 executing slice starting at index 24
  -> Phase 1 complete. All grainsize tasks synchronized.

--- 2. taskloop with num_tasks(4) (N = 32) ---
  [Task Chunk] Thread 1 executing Task containing index 08
  [Task Chunk] Thread 0 executing Task containing index 00
  [Task Chunk] Thread 3 executing Task containing index 24
  [Task Chunk] Thread 2 executing Task containing index 16
  -> Phase 2 complete. All num_tasks tasks synchronized.

--- 3. Enclosing Multiple Asynchronous taskloops in a taskgroup ---
  Master thread launched both taskloops with nogroup!
  Waiting at closing brace of taskgroup for all tasks to finish...
  -> Phase 3 complete. Both asynchronous taskloops verified.

[Verification] Sample results:
  results[0]  = 600 (Expected: 0*2 + 100 + 500 = 600)
  results[31] = 662 (Expected: 31*2 + 100 + 500 = 662)
============================================================
```

Notice:
1. In Phase 1, `grainsize(8)` divided 32 iterations into 4 chunks of 8 iterations each.
2. In Phase 2, `num_tasks(4)` explicitly demanded 4 tasks.
3. In Phase 3, two independent `taskloop` constructs with `nogroup` ran concurrently, safely joined by the enclosing `#pragma omp taskgroup`.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Mathematical Partitioning for `grainsize`
Let $N$ be the loop trip count, and $G$ be the argument passed to `grainsize(G)`.
The OpenMP standard mandates the following bounds on the number of generated tasks $K$ and actual chunk size $g$:

$$\lceil G / 2 \rceil \le g < 2 \cdot G$$

$$K = \max\left(1, \left\lfloor \frac{N}{G} \right\rfloor\right)$$

- If $N < G$, exactly **$1$ task** of size $N$ is generated.
- If $N \ge G$, the runtime divides $N$ into $K$ chunks, ensuring no chunk is smaller than $\lceil G/2 \rceil$ or larger than $2G-1$.

---

### 6.2 Mathematical Partitioning for `num_tasks`
Let $N$ be the trip count, and $M$ be the argument passed to `num_tasks(M)`.
- If $N < M$, OpenMP creates exactly **$N$ tasks**, each containing **$1$ iteration** (it will not create empty tasks).
- If $N \ge M$, OpenMP creates exactly **$M$ tasks**:
  - $R = N \pmod M$ tasks receive $\lfloor N/M \rfloor + 1$ iterations.
  - $M - R$ tasks receive $\lfloor N/M \rfloor$ iterations.

---

## 7. Advantages & Limitations

### Advantages
1. **Dynamic Load Balancing:** Tasks are distributed via work-stealing rather than rigid thread slicing.
2. **Recursive & Nested Composability:** Can be called safely from within recursive task functions where `#pragma omp for` is illegal.
3. **Overhead Control:** `grainsize` allows the developer to guarantee that no task is too small, amortizing runtime deque scheduling costs.

### Limitations & Traps
1. **Clause Exclusivity:** Using both `grainsize` and `num_tasks` in the same directive triggers a compilation error.
2. **Implicit Barrier Overhead:** Forgetting `nogroup` when chaining multiple independent taskloops forces threads to wait unnecessarily between loops.
3. **Memory Allocations:** `taskloop` creates real heap/pool task descriptors; for simple, perfectly balanced uniform array loops, `#pragma omp for` has lower memory overhead.

---

## 8. Formal Definitions for Exam

- **`#pragma omp taskloop`:** An OpenMP work-sharing construct that partitions the iterations of an enclosed canonical `for` loop into a set of discrete tasks, dispatching them to the runtime work-stealing scheduler.
- **`grainsize` Clause:** A taskloop clause specifying the target number of logical loop iterations assigned to each generated task, bounded between $\lceil G/2 \rceil$ and $2G$.
- **`num_tasks` Clause:** A taskloop clause specifying the total number of discrete tasks to be created from the loop's iteration space.
- **`nogroup` Clause:** A clause that suppresses the default implicit taskgroup enclosure around a `#pragma omp taskloop`, enabling asynchronous dispatch.
- **`#pragma omp taskgroup`:** A synchronization construct specifying that the encountering thread will wait at the end of the structured block until all child tasks and all their recursive descendants created within the block have finished execution.
- **Tied Task:** An OpenMP task that, if suspended at a scheduling point, must strictly be resumed and completed by the exact same thread that initially began its execution.

---

## 9. Comparison Table

### Comparison of Loop Partitioning Mechanisms

| Feature | `#pragma omp for schedule(static)` | `#pragma omp for schedule(dynamic)` | `#pragma omp taskloop grainsize(G)` |
|:---|:---:|:---:|:---:|
| **Underlying Mechanism** | Thread range math | Shared atomic loop index | **Runtime Task Deque Pool** |
| **Scheduler Entity** | Fixed OS Thread | Central Atomic Queue | **Work-Stealing Deque** |
| **Can execute inside a Task?**| **NO (Illegal)** | **NO (Illegal)** | **YES (Fully Supported)** |
| **Task Allocation Cost** | Zero | Zero | $\approx 64$ bytes per task descriptor |
| **Composability** | Poor (Rigid Team) | Poor (Rigid Team) | **Superb (Dynamic Tree Integration)** |

---

## 10. Solved Analytical Problems

### Problem 1: Resolving `grainsize` vs `num_tasks` Conflict
**Question:** A student writes the following OpenMP code. Explain why it fails to compile and how to fix it depending on whether the student wants to prioritize chunk duration or core saturation.

```c
#pragma omp taskloop grainsize(64) num_tasks(8)
for (int i = 0; i < 10000; i++) {
    process(i);
}
```

**Solution:**
1. **Compilation Error:** The OpenMP specification strictly states: *"At most one of `grainsize` and `num_tasks` clauses can appear on the `taskloop` directive."* Specifying both is an illegal syntactic conflict.
2. **Fix 1 (Prioritizing Chunk Duration):**
   If the goal is to guarantee that each task computes at least $64$ iterations to amortize scheduling overhead:
   ```c
   #pragma omp taskloop grainsize(64)
   ```
   Total tasks generated $\approx 10000 / 64 \approx 156$ tasks.
3. **Fix 2 (Prioritizing Core Saturation):**
   If the machine has 8 cores and the goal is to generate exactly 1 task per core:
   ```c
   #pragma omp taskloop num_tasks(8)
   ```
   Total tasks generated $= 8$, each containing $1250$ iterations.

---

### Problem 2: Deep vs Shallow Task Synchronization
**Question:** Analyze the task tree below. If Task 0 calls `#pragma omp taskwait`, which tasks are guaranteed to be finished before Task 0 resumes? What if Task 0 had enclosed the code in `#pragma omp taskgroup`?

```
                    Task 0 (Parent)
                   /               \
            Task 1 (Child)     Task 2 (Child)
               /                     \
       Task 3 (Grandchild)    Task 4 (Grandchild)
```

**Solution:**
1. **Under `#pragma omp taskwait`:**
   - `taskwait` synchronizes **only immediate child tasks**.
   - Tasks 1 and 2 are guaranteed to be finished.
   - However, if Task 1 spawned Task 3 and did not wait for it, **Task 3 and Task 4 may still be actively running in the background** when Task 0 resumes!
2. **Under `#pragma omp taskgroup`:**
   - `taskgroup` synchronizes **all descendants deeply**.
   - Tasks 1, 2, 3, and 4 are **all $100\%$ guaranteed** to have completed before execution exits the taskgroup block.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why should I use `taskloop` instead of `#pragma omp for schedule(dynamic)`?
**Answer:** While both provide dynamic load balancing, `schedule(dynamic)` uses a single centralized atomic index variable that suffers from severe memory bus contention and cache line bouncing on high core counts. In contrast, `taskloop` uses **distributed work-stealing deques**: each core interacts primarily with its own local L1/L2 cache deque, stealing from peers only when idle. Furthermore, `taskloop` can be nested inside irregular recursive functions where `#pragma omp for` is strictly forbidden.

### Q2: What is the purpose of the `untied` clause on tasks?
**Answer:** By default, OpenMP tasks are **tied**: if Thread 1 starts Task A, and Task A yields or suspends (e.g. at a `taskwait`), only Thread 1 is permitted to resume Task A later. If Thread 1 is blocked on another task, Task A sits stalled even if Threads 2, 3, and 4 are idle! Marking the task `untied` allows **any available thread** to resume Task A, maximizing core throughput (though caution is required if the task uses Thread-Local Storage or thread IDs).

### Q3: What happens if I forget `nogroup` when chaining multiple independent loops?
**Answer:** If you have three consecutive `#pragma omp taskloop` constructs without `nogroup`, each loop acts as a synchronous checkpoint: the thread waits for Loop 1 to finish completely before launching Loop 2, and waits for Loop 2 before launching Loop 3. Adding `nogroup` allows all three loops to dispatch their tasks asynchronously into the pool simultaneously, enabling cross-loop pipelining and overlap.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    TASKLOOP & TASKING CHEAT SHEET                                  |
+----------------------------------------------------------------------------------------------------+
| Directive              | #pragma omp taskloop [clauses]                                            |
| Core Purpose           | Slices canonical for-loop into dynamic work-stealing tasks.                |
| grainsize(G)           | Guarantees each task has ~G iterations (Between G/2 and 2G).               |
| num_tasks(K)           | Creates exactly K tasks across the iteration space.                       |
| Clause Conflict        | grainsize and num_tasks CANNOT appear on the same directive!               |
| nogroup Clause         | Removes default implicit taskgroup barrier; dispatches asynchronously.    |
+----------------------------------------------------------------------------------------------------+
|                                 TASK SYNCHRONIZATION SUMMARY                                       |
+----------------------------------------------------------------------------------------------------+
| #pragma omp taskwait   | Shallow wait: Suspends parent until all direct child tasks finish.        |
| #pragma omp taskgroup  | Deep wait: Suspends until all child tasks AND all descendants finish.     |
| #pragma omp barrier    | Thread barrier: Waits for all threads AND all outstanding tasks in team.  |
+----------------------------------------------------------------------------------------------------+
|                                   TASK TUNING CLAUSES                                              |
+----------------------------------------------------------------------------------------------------+
| untied                 | Permits task to be resumed by a different thread after suspension.        |
| final(expr)            | Inlines/undeferrs all child tasks when threshold expression is met.       |
| priority(val)          | Hints priority in work-stealing deque.                                    |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain how `#pragma omp taskloop` bridges the gap between data-parallel iteration spaces and dynamic task work-stealing.
2. Correctly choose and apply `grainsize` versus `num_tasks`, explaining why they are mutually exclusive.
3. Differentiate between shallow synchronization (`taskwait`) and deep recursive synchronization (`taskgroup`).
4. Apply the `nogroup` clause to overlap multiple independent taskloop pipelines.
5. Contrast tied versus untied task execution semantics and explain their scheduling implications.
