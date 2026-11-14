# 01. Task Parallelism vs. Data Parallelism in OpenMP

---

## 1. The Basic Idea

In shared-memory parallel programming, two distinct paradigms govern how computation is divided across processor cores:

1. **Data Parallelism (Loop Parallelism):** The identical operation or algorithm is applied concurrently across multiple elements of a uniform data structure (such as an array or matrix). OpenMP addresses this primarily via `#pragma omp for`.
2. **Task Parallelism (Functional / Asynchronous Parallelism):** The application is decomposed into discrete, independent, potentially heterogeneous units of work called **Tasks**. Each task packages a block of executable code together with its private data environment. Tasks are scheduled dynamically and executed asynchronously by whichever thread in the team becomes available.

```
       DATA PARALLELISM: Uniform Iteration Space
       All threads execute the same loop body on different slices of array A[0..N-1].
       Core 0: [=== A[0..24] ===]
       Core 1: [=== A[25..49] ===]
       Core 2: [=== A[50..74] ===]
       Core 3: [=== A[75..99] ===]

       TASK PARALLELISM: Irregular Dynamic Task Pool
       Tasks are created dynamically and placed into a shared task queue.
       [ Task: Process Node A ]   [ Task: Recurse Left Subtree ]   [ Task: Parse File ]
                  \                          |                         /
                   v                         v                        v
            +-------------------------------------------------------------+
            |               SHARED OPENMP TASK POOL / QUEUE               |
            +-------------------------------------------------------------+
                     |                       |                     |
                     v (Pulls Task)          v (Pulls Task)        v (Pulls Task)
                 [ Thread 0 ]            [ Thread 1 ]          [ Thread 2 ]
```

Prior to OpenMP 3.0, OpenMP was heavily dominated by data parallelism (`#pragma omp for` and `#pragma omp sections`). However, many fundamental algorithms in computer science—such as graph traversals, pointer-chasing linked lists, divide-and-conquer recursion (e.g., QuickSort), and dynamic tree searches—cannot be expressed in the canonical loop form. **Task Parallelism** was introduced to conquer these irregular, dynamic problem domains.

---

## 2. Why We Need It: The Limitations of `#pragma omp for`

Why is `#pragma omp for` insufficient for general parallel computing? Because loop work-sharing imposes four strict, unavoidable architectural constraints:

### Constraint 1: Canonical Loop Form Requires a Pre-Computable Trip Count
As established in Lecture 8, `#pragma omp for` requires that the number of iterations can be calculated *before the loop starts executing*. 
What if you are traversing a singly linked list?

```c
/* IMPOSSIBLE TO PARALLELIZE WITH #pragma omp for */
Node *curr = head;
while (curr != NULL) {
    process_node(curr);
    curr = curr->next; /* Pointer chasing: next address unknown until current is read! */
}
```
You cannot compute the trip count of a linked list or tree without traversing it sequentially first. `#pragma omp for` completely fails here.

### Constraint 2: Unbalanced Divide-and-Conquer Recursion
Consider the classic recursive Fibonacci or QuickSort algorithm:
```c
void quicksort(int *arr, int low, int high) {
    if (low < high) {
        int pivot = partition(arr, low, high);
        quicksort(arr, low, pivot - 1);  // Subproblem 1
        quicksort(arr, pivot + 1, high); // Subproblem 2
    }
}
```
Here, parallelism is generated dynamically in a recursive tree. The number of subproblems and their sizes are completely unpredictable at compile-time. There is no `for` loop to parallelize!

### Constraint 3: Rigid Thread-Team Binding
In `#pragma omp for`, loop slices are tightly coupled to the static thread team. If a thread finishes early or stalls on a memory fetch, work cannot easily be dynamically spawned and redistributed on the fly.

### How Tasking Solves All Three Limitations
With OpenMP Tasks:
- Any thread can package an arbitrary block of work as a task (`#pragma omp task`).
- Tasks can be spawned recursively from within other tasks.
- Pointer-chasing traversals can have one thread walk the list and generate tasks, while a team of worker threads consumes tasks from a runtime work pool.

---

## 3. How It Works: The OpenMP Tasking Architecture

### 3.1 What is an OpenMP Task?
An OpenMP task is an explicit, self-contained data structure created by the runtime that encapsulates:
1. **The Code to Execute:** A pointer to the outlined function representing the structured block.
2. **The Data Environment:** A snapshot of variables captured at task creation time (private copies of `firstprivate` data, and pointers to `shared` memory).
3. **Internal Control Variables (ICVs):** Metadata tracking task dependencies, priority, execution status (deferred, undeferred, suspended), and parent-child hierarchy.

```
+-------------------------------------------------------------------------------+
|                       OPENMP EXPLICIT TASK STRUCTURE                          |
|                                                                               |
|  [ Function Pointer ] -------> Points to .omp_task_entry()                    |
|  [ Data Environment ]                                                         |
|      - Shared Pointers: &shared_arr, &global_counter                          |
|      - Firstprivate Copies: node_val = 42, depth = 3                          |
|  [ Task Metadata ]                                                            |
|      - Parent Task Pointer: Task #104                                         |
|      - State: [ YIELDED | SUSPENDED | RUNNING | COMPLETED ]                   |
+-------------------------------------------------------------------------------+
```

---

### 3.2 The Task Producer-Consumer Pattern
The standard idiom for launching tasks in OpenMP involves **one thread generating tasks** while the **entire team executes them**:

```c
#pragma omp parallel
{
    #pragma omp single
    {
        /* Exactly ONE thread traverses the list and creates tasks */
        Node *p = head;
        while (p != NULL) {
            #pragma omp task firstprivate(p)
            {
                process_node(p); /* Executed by ANY idle thread in the team! */
            }
            p = p->next;
        }
    } /* Implicit barrier: Waits until ALL generated tasks finish execution */
}
```

```
                   TASK PRODUCER-CONSUMER MECHANISM
                   
    [ MASTER / SINGLE THREAD ]
    Walks Linked List: Node 0 -> Node 1 -> Node 2 -> Node 3 ...
         |              |          |          |
         +-(Spawns)-----+----------+----------+
         |
         v
    +------------------------------------------------------------------+
    |                     RUNTIME TASK POOL                            |
    |   [Task: Node 0]  [Task: Node 1]  [Task: Node 2]  [Task: Node 3] |
    +------------------------------------------------------------------+
              |                 |                 |
              v                 v                 v
        [ Thread 0 ]      [ Thread 1 ]      [ Thread 2 ]
       (Executes N0)     (Executes N1)     (Executes N2)
```

1. `#pragma omp parallel` creates the team of $T$ threads.
2. `#pragma omp single` ensures that only **one thread** walks the data structure and generates tasks. If `single` were omitted, all $T$ threads would redundantly traverse the linked list, generating $T \times N$ duplicate tasks!
3. Worker threads that are not executing the `single` block do **not** sit idle. The OpenMP runtime automatically instructs idle threads waiting at the barrier to pull and execute tasks from the task pool!

---

### 3.3 Work-Stealing Runtime Scheduling
Modern OpenMP runtimes (such as LLVM `libomp` and GCC `libgomp`) implement dynamic task scheduling via **Work-Stealing**:

```
      Thread 0 Deque                Thread 1 Deque                Thread 2 Deque
  +--------------------+        +--------------------+        +--------------------+
  | Task 3 (Top)       |        | Task 7 (Top)       |        | [ EMPTY ]          |
  | Task 2             |        | Task 6             |        +--------------------+
  | Task 1             |        +--------------------+                   ^
  | Task 0 (Bottom)    |                  ^ (Work Steal: Core 2          |
  +--------------------+                  |  steals Task 7 from Top!)    |
            ^                             +------------------------------+
            | (Owner pushes/pops from Bottom)
```

- Each thread maintains its own private double-ended queue (**Deque**) of tasks.
- A thread pushes new tasks and pops tasks to execute from the **bottom** (LIFO order: Last-In, First-Out, maximizing cache locality).
- When a thread exhausts its own deque and becomes idle, it becomes a **thief**: it targets a randomly selected peer thread and **steals** a task from the **top** of the peer's deque (FIFO order).
- This guarantees near-optimal dynamic load balancing across arbitrary irregular workloads with minimal lock contention.

---

## 4. Architectural Comparison: Task vs. Data Parallelism

| Feature | Data Parallelism (`#pragma omp for`) | Task Parallelism (`#pragma omp task`) |
|:---|:---|:---|
| **Primary Target** | Arrays, matrices, dense linear algebra | Linked lists, trees, graphs, recursive algorithms |
| **Work Granularity** | Loop iteration chunks | Arbitrary structured code blocks |
| **Scheduling Decision** | Static, Dynamic, or Guided chunking | Dynamic Work-Stealing task scheduler |
| **Creation Timing** | Fixed at loop entry | Dynamically spawned on the fly at runtime |
| **Memory Overhead** | Minimal (Loop bounds per thread) | High (Task descriptor heap allocation) |
| **Cache Locality** | **Superb** (Contiguous spatial prefetching) | Variable (Depends on data structure traversal) |
| **Synchronization** | Implicit barrier at loop exit | `#pragma omp taskwait` or enclosing barrier |

---

## 5. Compilable C Code Examples

### 5.1 Program: Parallel Linked List Traversal and Recursive Fibonacci
This program demonstrates both fundamental tasking paradigms:
1. Traversal of an irregular, pointer-chasing singly linked list.
2. Divide-and-conquer recursive computation (Fibonacci) with task synchronization.

Save this file as `omp_task_vs_data.c`:

```c
/* =========================================================================
 * File: omp_task_vs_data.c
 * Description: Demonstrates:
 *              1. Pointer-chasing Linked List traversal using OpenMP Tasks
 *              2. Divide-and-conquer recursion (Fibonacci) using Tasks
 *              3. Task synchronization with #pragma omp taskwait
 * Compilation:
 *   gcc -O3 -fopenmp omp_task_vs_data.c -o omp_task_vs_data
 * Execution:
 *   ./omp_task_vs_data
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <omp.h>

#define NUM_NODES 10
#define NUM_THREADS 4

/* Singly linked list definition */
typedef struct Node {
    int id;
    int payload;
    struct Node *next;
} Node;

/* Helper to append node */
Node* create_node(int id, int payload, Node *next) {
    Node *n = (Node*)malloc(sizeof(Node));
    n->id = id;
    n->payload = payload;
    n->next = next;
    return n;
}

/* Simulated compute-intensive task */
void process_node(Node *node) {
    int tid = omp_get_thread_num();
    /* Artificial micro-delay to simulate work */
    for (volatile int d = 0; d < 2000000; d++);
    printf("  [Task Process] Node %d (Payload=%d) processed by Thread %d\n",
           node->id, node->payload, tid);
}

/* -------------------------------------------------------------
 * Recursive Divide-and-Conquer Task Parallelism: Fibonacci
 * ------------------------------------------------------------- */
long long fib_task(int n) {
    if (n < 2) return n;

    /* Cutoff threshold: Sequential fallback for small subproblems
     * to avoid task creation overhead dominating computation! */
    if (n < 20) {
        long long a = 0, b = 1;
        for (int i = 2; i <= n; i++) {
            long long temp = a + b;
            a = b;
            b = temp;
        }
        return b;
    }

    long long x, y;

    /* Spawn subproblem 1 as an asynchronous task */
    #pragma omp task shared(x) firstprivate(n)
    {
        x = fib_task(n - 1);
    }

    /* Spawn subproblem 2 as an asynchronous task */
    #pragma omp task shared(y) firstprivate(n)
    {
        y = fib_task(n - 2);
    }

    /* CRITICAL SYNCHRONIZATION:
     * Wait until both child tasks finish before adding x + y! */
    #pragma omp taskwait

    return x + y;
}

int main(void) {
    printf("============================================================\n");
    printf("     OpenMP Task Parallelism vs. Data Parallelism Demo      \n");
    printf("============================================================\n\n");

    /* -------------------------------------------------------------
     * DEMO 1: Linked List Traversal
     * ------------------------------------------------------------- */
    printf("--- 1. Irregular Pointer-Chasing Linked List Traversal ---\n");
    Node *head = NULL;
    for (int i = NUM_NODES - 1; i >= 0; i--) {
        head = create_node(i, (i + 1) * 100, head);
    }

    double t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(head, stdout)
    {
        #pragma omp single
        {
            printf("  Master Thread %d walking linked list and dispatching tasks...\n",
                   omp_get_thread_num());

            Node *curr = head;
            while (curr != NULL) {
                /* Capture the pointer 'curr' by value (firstprivate) */
                #pragma omp task firstprivate(curr)
                {
                    process_node(curr);
                }
                curr = curr->next;
            }
        } /* Implicit barrier: Waits for all linked-list tasks to complete! */
    }

    double t1 = omp_get_wtime();
    printf("Linked list processed in %8.4f ms\n\n", (t1 - t0) * 1000.0);

    /* -------------------------------------------------------------
     * DEMO 2: Recursive Divide-and-Conquer Task Parallelism
     * ------------------------------------------------------------- */
    int fib_n = 30;
    printf("--- 2. Recursive Tasking: Fibonacci(%d) ---\n", fib_n);
    long long fib_result = 0;

    t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(fib_n, fib_result)
    {
        #pragma omp single
        {
            fib_result = fib_task(fib_n);
        }
    }

    t1 = omp_get_wtime();
    printf("Fibonacci(%d) = %lld (Computed in %8.4f ms)\n",
           fib_n, fib_result, (t1 - t0) * 1000.0);

    /* Free linked list */
    while (head != NULL) {
        Node *temp = head;
        head = head->next;
        free(temp);
    }

    printf("============================================================\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_task_vs_data.c -o omp_task_vs_data
$ ./omp_task_vs_data
```

```text
============================================================
     OpenMP Task Parallelism vs. Data Parallelism Demo      
============================================================

--- 1. Irregular Pointer-Chasing Linked List Traversal ---
  Master Thread 0 walking linked list and dispatching tasks...
  [Task Process] Node 0 (Payload=100) processed by Thread 0
  [Task Process] Node 2 (Payload=300) processed by Thread 1
  [Task Process] Node 1 (Payload=200) processed by Thread 2
  [Task Process] Node 3 (Payload=400) processed by Thread 3
  [Task Process] Node 4 (Payload=500) processed by Thread 0
  [Task Process] Node 5 (Payload=600) processed by Thread 1
  [Task Process] Node 6 (Payload=700) processed by Thread 2
  [Task Process] Node 7 (Payload=800) processed by Thread 3
  [Task Process] Node 8 (Payload=900) processed by Thread 0
  [Task Process] Node 9 (Payload=1000) processed by Thread 1
Linked list processed in  12.4120 ms

--- 2. Recursive Tasking: Fibonacci(30) ---
Fibonacci(30) = 832040 (Computed in   4.1200 ms)
============================================================
```

Notice:
1. Thread 0 walked the list and posted tasks into the runtime pool.
2. Threads 1, 2, and 3 simultaneously grabbed tasks from the pool and processed nodes concurrently!
3. In Fibonacci, tasks synchronized their results using `#pragma omp taskwait` before performing addition.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Task Creation Overhead vs. Cutoff Optimization
Creating a task is not free. When `#pragma omp task` executes, the runtime must:
1. Allocate a Task Control Block (TCB) on the heap or thread task buffer ($\approx 64\text{--}128$ bytes).
2. Copy `firstprivate` arguments.
3. Push the task descriptor pointer onto the local deque.

$$\text{Task Creation Latency } \tau_{\text{task}} \approx 100\text{--}300 \text{ CPU clock cycles} \quad (30\text{--}100\,\text{ns})$$

#### The Amortization Condition:
A function or block of work $W$ should be packaged as a task if and only if its execution time significantly exceeds task creation overhead:

$$W_{\text{compute}} \gg \tau_{\text{task}}$$

If a subproblem takes only $5\,\text{ns}$ (e.g., `Fibonacci(3)`), turning it into a task will make the parallel program **$50\times$ slower than sequential execution**!
This is why recursive task algorithms **must always implement a Sequential Cutoff Threshold** (as shown in our code for $N < 20$).

---

## 7. Advantages & Limitations

### When to Use Task Parallelism vs. Data Parallelism

```
                              DECISION FLOWCHART
                              
                   Is data structure an array/matrix
                  with computable iteration count N?
                                  |
                   +--------------+--------------+
                   | YES                         | NO
                   v                             v
           Use Loop Parallelism         Are subproblems recursive,
           (#pragma omp for)            pointer-chasing, or irregular?
                                                 |
                                  +--------------+--------------+
                                  | YES                         | NO
                                  v                             v
                          Use Tasking             Use Sections or Pthreads
                          (#pragma omp task)
```

### Advantages of Tasking
1. **Unbounded Irregularity:** Seamlessly handles linked lists, trees, graphs, and dynamic recursion.
2. **Work-Stealing Efficiency:** Idle cores actively steal work, maximizing hardware utilization.
3. **Decoupled Architecture:** Work generation is completely decoupled from thread execution.

### Limitations & Traps
1. **High Memory Overhead:** Creating millions of microscopic tasks can exhaust RAM and trash CPU caches.
2. **Firstprivate Scoping Trap:** In a loop or pointer walk, forgetting `firstprivate(p)` causes all tasks to share a single pointer variable, leading to data races on the pointer itself!

---

## 8. Formal Definitions for Exam

- **Task Parallelism:** A parallel execution model where an application is decomposed into discrete, concurrent computational units (tasks) that can execute asynchronously on any available thread.
- **Data Parallelism:** A parallel execution model where the same computational operation is applied simultaneously across disjoint partitions of a uniform dataset or array.
- **OpenMP Task:** An independent unit of executable work consisting of code to execute, a private data environment, and internal control variables, managed by the OpenMP task scheduler.
- **Work-Stealing:** A dynamic scheduling algorithm where processor cores that have exhausted their private task deques steal tasks from the deques of other busy cores to maintain load balance.
- **`#pragma omp taskwait`:** A task synchronization directive that suspends the current task until all immediate child tasks generated by it have completed execution.
- **Sequential Cutoff:** An algorithmic optimization threshold in recursive task parallelism where subproblems below a certain depth or size are executed sequentially to avoid task allocation overhead.

---

## 9. Comparison Table

### Summary Comparison: Data Parallelism vs. Task Parallelism

| Dimension | Data Parallelism (`for`) | Task Parallelism (`task`) |
|:---|:---|:---|
| **Underlying Metaphor** | Assembly line slicing | Shared job board / work queue |
| **Control Structure** | Counted `for` loop | Arbitrary functions, blocks, while-loops, recursion |
| **Dynamic Work Creation?**| **NO** (Fixed at loop boundary) | **YES** (Spawned dynamically anywhere) |
| **Data Scoping Default** | Variables default to `shared` | Outer variables default to `firstprivate` (in explicit tasks!) |
| **Overhead per Work Unit** | Extremely low ($< 1\,\text{ns}$) | Moderate ($30\text{--}100\,\text{ns}$) |
| **Hardware Prefetching** | **Optimal** (Linear memory streaming) | Poor to Moderate (Pointer chasing) |

---

## 10. Solved Analytical Problems

### Problem 1: The Pointer Scoping Disaster in While-Loops
**Question:** Examine the following tasking code. Identify the catastrophic concurrency bug and explain how to fix it.

```c
Node *curr = head;
#pragma omp parallel
{
    #pragma omp single
    {
        while (curr != NULL) {
            #pragma omp task
            {
                process(curr); /* Line X: What is the scope of curr? */
            }
            curr = curr->next;
        }
    }
}
```

**Solution:**
1. **The Bug:** In OpenMP, variables declared outside an explicit task construct that are shared in the enclosing parallel region are **shared by default** inside the task!
2. **The Race Condition:**
   - The master thread walks the while-loop and modifies `curr = curr->next`.
   - The tasks spawned inside the loop do not capture their own snapshot of `curr`; they read the shared pointer `curr`.
   - By the time a worker thread actually executes `process(curr)`, the master thread has already advanced `curr` multiple steps ahead, or even reached `NULL`!
   - Result: Multiple tasks process the same tail nodes, earlier nodes are completely skipped, and passing `NULL` causes an immediate **Segmentation Fault**.
3. **The Fix:** Explicitly capture `curr` by value using `firstprivate(curr)`:
   ```c
   #pragma omp task firstprivate(curr)
   {
       process(curr); // Each task receives a private copy of the pointer address!
   }
   ```

---

### Problem 2: Missing `taskwait` in Recursive Algorithms
**Question:** Explain what happens if `#pragma omp taskwait` is omitted from the recursive Fibonacci function.

**Solution:**
- If `taskwait` is omitted:
  ```c
  #pragma omp task shared(x)
  x = fib(n - 1);
  #pragma omp task shared(y)
  y = fib(n - 2);

  return x + y; // OMITTED taskwait!
  ```
- The function will immediately evaluate `x + y` and return **before** the asynchronous child tasks have finished computing!
- Variables `x` and `y` will contain whatever random garbage was in memory or uncompleted calculations.
- **Rule:** A parent task must always execute `#pragma omp taskwait` before reading the outputs of its asynchronous child tasks.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Why must task generation loops be enclosed in `#pragma omp single`?
**Answer:** Because inside a `#pragma omp parallel` block, every thread in the team executes the structured block. If you omit `#pragma omp single`, **all $T$ threads will execute the while-loop simultaneously**, each traversing the linked list and each generating $N$ tasks. If $T = 8$ and $N = 1000$, the system will create $8{,}000$ duplicate tasks, wasting memory and executing every node 8 times! `#pragma omp single` guarantees that exactly one thread generates the tasks while the remaining $T-1$ threads immediately begin executing tasks from the queue.

### Q2: What is the difference between `#pragma omp barrier` and `#pragma omp taskwait`?
**Answer:**
- `#pragma omp barrier` synchronizes **threads in a team**: no thread can pass until *all threads* arrive.
- `#pragma omp taskwait` synchronizes **tasks in a hierarchy**: the calling task suspends until all of its *immediate child tasks* complete execution. Other sibling tasks and unrelated threads continue running uninterrupted.

### Q3: When is Data Parallelism strictly superior to Task Parallelism?
**Answer:** For dense array and matrix operations (e.g., image filters, matrix multiplication, vector addition). In these domains, `#pragma omp for` has near-zero overhead, perfect spatial cache locality, and enables compiler vectorization (SIMD). Using tasks here would add unnecessary queue allocation overhead without any load-balancing benefit.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                TASK vs. DATA PARALLELISM CHEAT SHEET                                |
+----------------------------------------------------------------------------------------------------+
| Model                  | Data Parallelism (#pragma omp for)  | Task Parallelism (#pragma omp task)  |
| Primary Domain         | Predictable arrays, counted loops   | Linked lists, trees, graphs, recurse |
| Scheduling             | Static, Dynamic, Guided chunks      | Work-stealing task deques            |
| Canonical Idiom        | #pragma omp parallel for            | #pragma omp parallel                 |
|                        |                                     |   #pragma omp single                 |
|                        |                                     |     #pragma omp task firstprivate(p) |
+----------------------------------------------------------------------------------------------------+
|                                    CRITICAL TASKING RULES                                          |
+----------------------------------------------------------------------------------------------------+
| The Pointer Rule       | ALWAYS use firstprivate(p) when traversing pointer data structures!       |
| The Single Rule        | ALWAYS enclose task generation inside #pragma omp single!                 |
| The Synchronization    | Use #pragma omp taskwait to wait for child tasks before reading results.  |
| The Cutoff Rule        | Always use a sequential cutoff threshold to avoid micro-task overhead!    |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the architectural limitations of `#pragma omp for` that necessitate task parallelism.
2. Implement the standard OpenMP producer-consumer pattern using `#pragma omp single` and `#pragma omp task`.
3. Avoid the classic pointer-scoping trap by applying `firstprivate` to captured node references.
4. Implement recursive divide-and-conquer algorithms using `#pragma omp task` and `#pragma omp taskwait`.
5. Explain how work-stealing task deques provide automatic dynamic load balancing.
6. Design sequential cutoff thresholds to amortize task allocation overhead.
