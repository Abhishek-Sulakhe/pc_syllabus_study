# Module 08: POSIX Threads Foundations (Lectures 17–19)
# Subtopic 08.04: Joining, Detaching, and Resource Reclamation (`pthread_join`, `pthread_detach`, Zombie Threads, `pthread_exit`)

---

## 1. Basic Idea

When a POSIX thread terminates its execution (either by returning from its start routine or calling `pthread_exit()`), it does not immediately vanish from operating system memory. Similar to UNIX processes that become zombies if their parent process does not call `wait()` or `waitpid()`, a terminated POSIX thread remains in a **zombie thread state** until another thread reclaims its exit status and metadata.

POSIX provides two fundamentally mutually exclusive lifecycles for managing thread completion and resource reclamation:

1. **Joinable Threads (`PTHREAD_CREATE_JOINABLE` - Default):**
   The thread preserves its exit status code (a `void*` pointer) and internal kernel thread descriptors until another active thread in the same process explicitly synchronizes with it via `pthread_join()`. The calling thread is blocked until the target thread terminates, at which point the kernel frees the terminated thread's thread control block (TCB), execution stack, and registers.

2. **Detached Threads (`PTHREAD_CREATE_DETACHED` or `pthread_detach()`):**
   The thread indicates to the operating system that its return value is of no interest to any other thread. The moment a detached thread finishes execution, all underlying operating system resources—including its virtual memory stack mapping, registers, and kernel task descriptors—are **immediately and automatically reclaimed** by the operating system kernel without waiting for another thread.

```
+-----------------------------------------------------------------------------------------+
|                               Thread Lifecycle Comparison                               |
+-----------------------------------------------------------------------------------------+
| JOINABLE THREAD:                                                                        |
|   Running ---> Terminated ---> [ ZOMBIE STATE ] ---> pthread_join() ---> Reclaimed      |
|                                (Holds Stack & TCB)                       (Fully Cleaned)|
|                                                                                         |
| DETACHED THREAD:                                                                        |
|   Running ---> Terminated ---------------------------------------------> Reclaimed      |
|                                                                          (Immediate)    |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Thread Leak (Zombie Accumulation)
If an application repeatedly spawns joinable threads to handle asynchronous tasks (e.g., in a network connection handler) but fails to call `pthread_join()`, the process suffers from a severe **thread resource leak**:
- Each terminated thread retains its entire allocated virtual memory stack (typically 8 MB on x86_64 Linux).
- The operating system kernel retains a `struct task_struct` and thread ID (TID) in its process table.
- Over time, the process reaches its thread limit (`/proc/sys/kernel/threads-max` or `ulimit -u`), causing subsequent calls to `pthread_create()` to fail with `EAGAIN` or crash with out-of-memory (`ENOMEM`) errors.

### 2. Synchronization of Computational Results
In High-Performance Computing (HPC), parallel algorithms employ the **fork-join model**:
- The main thread creates $N$ worker threads to compute partial matrix blocks, integral slices, or sub-arrays.
- The main thread must guarantee that **all workers have finished computing** before aggregating the partial sums or printing the final result.
- `pthread_join()` provides both the **synchronization barrier** (guaranteeing completion) and the **safe transport mechanism** for passing the worker's dynamically allocated computation result back to the orchestrator.

### 3. "Fire-and-Forget" Background Operations
Certain tasks—such as asynchronous disk logging, cache flushing, or heartbeat polling—operate purely as side effects. Neither the main thread nor any peer thread cares about their exit value or exact termination time. Forcing such workers to be joinable requires maintaining an array of thread handles and orchestrating complex cleanup routines. Detaching them allows the system to clean up autonomously.

---

## 3. How It Works

### The Anatomy of `pthread_join()` Synchronization

When Thread A executes `pthread_join(thread_B, &status)`:

```
  Thread A (Parent / Joiner)                          Thread B (Worker / Target)
  =========================                          ==========================
             |                                                    |
             |                                                    | Computing workload...
  Calls pthread_join(B)                                           |
             |                                                    |
             v                                                    |
    [ BLOCKED in Kernel ]                                         |
    (Waiting on B termination)                                    |
             |                                                    | Finished work
             |                                                    | Calls pthread_exit(res)
             |                                                    v
             |                                            +----------------+
             |                                            |  ZOMBIE STATE  |
             | <--- Kernel wakes Thread A ----------------+----------------+
             v                                            | (Stack, TCB,   |
    Wakes up; copies status                               |  Exit Code)    |
    Unmaps Thread B Stack                                 +----------------+
    Frees Thread B TCB                                            |
             |                                                    v
    Resumes Execution                                     [ Memory Reclaimed ]
```

1. **If Thread B is still executing:** Thread A is placed into an uninterruptible sleep state (waiting on the internal termination futex/event associated with Thread B).
2. **If Thread B has already exited:** `pthread_join()` returns immediately without blocking, copies the exit status pointer, and frees Thread B's resources.
3. **Double Join Hazard:** Once `pthread_join()` successfully returns for a target thread, its `pthread_t` handle becomes invalid. Calling `pthread_join()` a second time on the same `pthread_t` produces undefined behavior (often crashing or returning `ESRCH`/`EINVAL`).

### Transitioning to Detached State

A thread can be made detached in two distinct ways:

```
METHOD 1: At Creation Time (via Attributes)
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  pthread_create(&tid, &attr, worker, NULL);
  pthread_attr_destroy(&attr);
  ===> Thread is born detached. Can NEVER be joined.

METHOD 2: Post Creation (Dynamically)
  pthread_create(&tid, NULL, worker, NULL);
  pthread_detach(tid);    <--- Called by parent thread
          -- OR --
  pthread_detach(pthread_self()); <--- Called by worker itself inside worker()
  ===> Thread transitions from joinable to detached. Can NEVER be joined again.
```

---

## 4. Important Mechanisms and Functions

### 1. The `pthread_join` Function

```c
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```

- **`thread`:** The handle of the target joinable thread.
- **`retval`:** A pointer to a `void*` pointer (a double pointer). If non-NULL, `pthread_join()` copies the exit status of the target thread (the value passed to `pthread_exit()` or returned from the start routine) into `*retval`. If the caller does not care about the exit status, `NULL` may be passed.
- **Return Values:**
  - `0`: Success.
  - `EDEADLK`: Deadlock detected (e.g., two threads trying to join each other, or a thread attempting to join itself: `pthread_join(pthread_self(), ...)`).
  - `EINVAL`: Thread is not a joinable thread (it was detached) OR another thread is already waiting to join this thread.
  - `ESRCH`: No thread with the specified ID could be found.

### 2. The `pthread_detach` Function

```c
int pthread_detach(pthread_t thread);
```

- **`thread`:** The handle of the thread to be detached.
- **Return Values:**
  - `0`: Success.
  - `EINVAL`: The thread is already in a detached state.
  - `ESRCH`: No thread with the specified ID could be found.
- Note: Can be called on `pthread_self()` by a worker thread directly.

### 3. Thread Termination: `return` vs `pthread_exit()` vs `exit()`

```c
void pthread_exit(void *retval);
```

| Termination Method | Scope of Termination | Resource Handling | Return Value Passing |
| :--- | :--- | :--- | :--- |
| **`return (void*)val;`** | Exits only the calling thread | Invokes local stack unwinding / C++ destructors in scope | Value returned to joiner via `*retval` |
| **`pthread_exit(val);`** | Exits only the calling thread | Executes cleanup handlers; unwinds stack; keeps process alive | Value passed to joiner via `*retval` |
| **`exit(status);`** | **Terminates ENTIRE Process** | Kills ALL threads in process immediately; flushes stdio | Returns integer exit code to parent shell |

> [!WARNING]
> Calling `exit()`, `_exit()`, or `abort()` from inside ANY worker thread will immediately kill the entire process and all sister threads running within it! Always use `return` or `pthread_exit()` to terminate an individual thread cleanly.

---

## 5. Compilable C Code Examples

### Example 1: Matrix Row Sum with Safe `pthread_join()` and Result Retrieval
This program demonstrates spawning multiple worker threads, computing row-wise sums of a matrix, returning dynamically allocated result structures via `pthread_exit()`, and retrieving them safely in `main()` with proper error handling and deallocation.

```c
/*
 * File: pthreads_join_matrix.c
 * Compilation:
 *   gcc -O2 -pthread pthreads_join_matrix.c -o pthreads_join_matrix
 *
 * Execution:
 *   ./pthreads_join_matrix
 */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NUM_ROWS 4
#define NUM_COLS 5

/* Input matrix */
static int matrix[NUM_ROWS][NUM_COLS] = {
    { 1,  2,  3,  4,  5},  /* Sum = 15  */
    {10, 20, 30, 40, 50},  /* Sum = 150 */
    { 2,  4,  6,  8, 10},  /* Sum = 30  */
    { 7,  7,  7,  7,  7}   /* Sum = 35  */
};

/* Result container allocated on heap by each worker */
typedef struct {
    int row_index;
    long long row_sum;
} WorkerResult;

/* Thread worker routine */
void* compute_row_sum(void* arg) {
    long row = (long)arg;

    WorkerResult* result = (WorkerResult*)malloc(sizeof(WorkerResult));
    if (result == NULL) {
        perror("malloc failed in worker");
        pthread_exit(NULL);
    }

    result->row_index = (int)row;
    result->row_sum = 0;

    for (int col = 0; col < NUM_COLS; ++col) {
        result->row_sum += matrix[row][col];
    }

    printf("[Worker %ld] Computed sum of row %ld = %lld\n", row, row, result->row_sum);

    /* Return heap-allocated structure to joining thread */
    pthread_exit((void*)result);
}

int main(void) {
    pthread_t threads[NUM_ROWS];
    int status;
    long long grand_total = 0;

    printf("=== Pthreads Fork-Join: Parallel Row Summation ===\n");

    /* Fork: Spawn worker threads */
    for (long i = 0; i < NUM_ROWS; ++i) {
        status = pthread_create(&threads[i], NULL, compute_row_sum, (void*)i);
        if (status != 0) {
            fprintf(stderr, "Error: pthread_create failed for row %ld: %s\n",
                    i, strerror(status));
            exit(EXIT_FAILURE);
        }
    }

    /* Join: Wait for all workers and aggregate results */
    for (int i = 0; i < NUM_ROWS; ++i) {
        void* raw_result = NULL;
        status = pthread_join(threads[i], &raw_result);
        if (status != 0) {
            fprintf(stderr, "Error: pthread_join failed for thread %d: %s\n",
                    i, strerror(status));
            continue;
        }

        if (raw_result != NULL) {
            WorkerResult* res = (WorkerResult*)raw_result;
            printf("[Main] Collected result from Row %d: %lld\n",
                   res->row_index, res->row_sum);
            grand_total += res->row_sum;
            free(res); /* Crucial: Free worker's heap allocation */
        }
    }

    printf("=== Final Aggregated Grand Total = %lld ===\n", grand_total);
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== Pthreads Fork-Join: Parallel Row Summation ===
[Worker 0] Computed sum of row 0 = 15
[Worker 1] Computed sum of row 1 = 150
[Worker 2] Computed sum of row 2 = 30
[Worker 3] Computed sum of row 3 = 35
[Main] Collected result from Row 0: 15
[Main] Collected result from Row 1: 150
[Main] Collected result from Row 2: 30
[Main] Collected result from Row 3: 35
=== Final Aggregated Grand Total = 230 ===
```

---

### Example 2: Self-Detaching Threads and Zombie Prevention Demonstration
This example demonstrates how an asynchronous logger detaches itself using `pthread_detach(pthread_self())`, proving that detached threads prevent zombie leaks and that calling `pthread_join()` on them returns `EINVAL`.

```c
/*
 * File: pthreads_detach_demo.c
 * Compilation:
 *   gcc -O2 -pthread pthreads_detach_demo.c -o pthreads_detach_demo
 *
 * Execution:
 *   ./pthreads_detach_demo
 */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>

void* detached_logger(void* arg) {
    long task_id = (long)arg;

    /* Self-detach: Informs OS to reclaim all resources immediately upon return */
    int detach_res = pthread_detach(pthread_self());
    if (detach_res != 0) {
        fprintf(stderr, "Worker failed to self-detach: %s\n", strerror(detach_res));
    } else {
        printf("[Logger %ld] Successfully self-detached. Thread ID = %lu\n",
               task_id, (unsigned long)pthread_self());
    }

    /* Simulate periodic logging work */
    printf("[Logger %ld] Writing audit records to simulated persistent log...\n", task_id);
    usleep(100000); /* 100 ms */

    printf("[Logger %ld] Work finished. Terminating via pthread_exit().\n", task_id);
    pthread_exit(NULL);
}

int main(void) {
    pthread_t log_thread;
    int status;

    printf("=== POSIX Thread Detachment and Error Verification ===\n");

    /* Create thread in default joinable state */
    status = pthread_create(&log_thread, NULL, detached_logger, (void*)42L);
    if (status != 0) {
        fprintf(stderr, "pthread_create failed: %s\n", strerror(status));
        return EXIT_FAILURE;
    }

    /* Give worker time to execute its self-detach */
    usleep(20000); /* 20 ms */

    /* Attempt to join the detached thread: Must fail with EINVAL */
    printf("[Main] Attempting pthread_join() on self-detached thread...\n");
    status = pthread_join(log_thread, NULL);

    if (status == EINVAL) {
        printf("[Main] Caught expected error EINVAL (%d): Thread is detached!\n", status);
    } else if (status == 0) {
        printf("[Main] ERROR: Thread was unexpectedly joined!\n");
    } else {
        printf("[Main] pthread_join returned error: %s (%d)\n", strerror(status), status);
    }

    /* Allow the detached worker to complete cleanly before process terminates */
    usleep(150000); /* 150 ms */
    printf("[Main] Process exiting cleanly.\n");

    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== POSIX Thread Detachment and Error Verification ===
[Logger 42] Successfully self-detached. Thread ID = 140456123456784
[Logger 42] Writing audit records to simulated persistent log...
[Main] Attempting pthread_join() on self-detached thread...
[Main] Caught expected error EINVAL (22): Thread is detached!
[Logger 42] Work finished. Terminating via pthread_exit().
[Main] Process exiting cleanly.
```

---

## 6. Mathematical Formulations & Overhead Analysis

### Memory Accumulation of Unjoined Zombie Threads

Suppose an application creates threads at a continuous rate of $\lambda$ threads per second. Each thread executes for an average duration $\bar{t}_{\text{exec}}$ seconds and terminates. If the main program neglects to call `pthread_join()` or `pthread_detach()`:

Let:
- $S_{\text{stack}}$ = Stack memory reserved per thread (typically 8 MB default).
- $S_{\text{TCB}}$ = Kernel thread control block and book-keeping metadata ($\approx 2\text{ KB}$).
- $t$ = Elapsed time in seconds.

The cumulative virtual memory held in uncollected zombie state $M_{\text{leak}}(t)$ grows monotonically:

$$M_{\text{leak}}(t) = \lambda \cdot t \cdot (S_{\text{stack}} + S_{\text{TCB}})$$

For $\lambda = 10\text{ threads/sec}$, $S_{\text{stack}} = 8\text{ MB}$:
$$\frac{dM}{dt} = 10 \times 8.002\text{ MB/s} \approx 80.02\text{ MB/sec}$$

After just 60 seconds of operation:
$$M_{\text{leak}}(60) = 60 \times 80.02\text{ MB} \approx 4{,}801.2\text{ MB} \approx 4.69\text{ GB}$$

The system will completely exhaust its virtual memory mappings within minutes, terminating with out-of-memory errors.

### The Fork-Join Execution Barrier Overhead

In a parallel divide-and-conquer algorithm with $P$ threads, the execution time $T(P)$ is bounded by the slowest executing thread (the straggler effect):

$$T_{\text{parallel}} = \max_{1 \le i \le P} \left( T_i \right) + T_{\text{spawn}}(P) + T_{\text{join}}(P)$$

Where:
- $T_{\text{spawn}}(P) = \sum_{i=1}^P t_{\text{create}}$: Time taken to sequentially invoke `pthread_create()`.
- $T_{\text{join}}(P) = \sum_{i=1}^P t_{\text{join}}$: Time to resolve joins and context-switch.

If work is imbalanced such that thread $k$ takes significantly longer than others:
$$\text{Synchronization Idle Wait} = \sum_{i=1}^P \left( \max_j(T_j) - T_i \right)$$

This idle wait represents wasted core cycles during `pthread_join()` synchronization.

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Comparison

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Mechanism                 | Key Advantages                          | Critical Pitfalls / Limitations         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_join()`          | - Deterministic termination barrier     | - Calling thread is blocked             |
|                           | - Retrieves dynamically computed values | - Double-join causes undefined behavior |
|                           | - Completely prevents zombie leaks      | - Can cause deadlocks (joining self)    |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_detach()`        | - Zero-overhead automatic cleanup       | - Cannot retrieve return code/value     |
|                           | - No blocked joiner thread required     | - Parent cannot easily determine when   |
|                           | - Prevents thread descriptor exhaustion |   the detached thread completes         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_exit()`          | - Exits calling thread without killing  | - Passing pointers to local variables   |
|                           |   the entire process                    |   causes dangling pointer crashes       |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **`pthread_join()`:** A blocking synchronization function that suspends the execution of the calling thread until the specified target joinable thread terminates, optionally retrieving the target thread's exit code and freeing its system resources.
>
> **`pthread_detach()`:** A lifecycle management function that marks a thread as detached, indicating to the operating system that its resources should be reclaimed immediately upon termination, disabling any subsequent joins.
>
> **Zombie Thread:** A terminated joinable thread whose execution has finished, but whose stack and kernel descriptors remain allocated in system memory because no thread has called `pthread_join()` to collect its exit status.
>
> **Thread Leak:** A resource exhaustion condition occurring when joinable threads terminate without being joined, resulting in unbounded accumulation of zombie thread stacks and kernel thread table entries.
>
> **`pthread_exit()`:** A POSIX library function that terminates only the calling thread, returning a status pointer to any thread awaiting its completion via `pthread_join()`, without affecting other threads in the process.

---

## 9. Comparison Tables

### `pthread_join()` vs `pthread_detach()`

| Dimension | `pthread_join(tid, &status)` | `pthread_detach(tid)` |
| :--- | :--- | :--- |
| **Blocking Behavior** | **Yes** (Blocks caller until target finishes) | **No** (Returns immediately without blocking) |
| **Return Value Access** | Can inspect and retrieve `void*` return value | **Cannot** retrieve return value (discarded) |
| **Resource Reclamation** | Occurs when `pthread_join()` returns | Occurs automatically upon thread termination |
| **Subsequent Join Attempt**| Undefined / crashes (`EINVAL` / `ESRCH`) | Fails immediately with `EINVAL` |
| **Ideal Use Case** | Worker threads computing partial results | Background loggers, monitor daemons, garbage collectors |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: The Stack Variable Return Anti-Pattern
**Question:** Analyze the following C code snippet. Explain what catastrophic runtime error occurs and provide the correct fix.

```c
void* worker_sum(void* arg) {
    long n = (long)arg;
    long long total = 0;
    for (long i = 1; i <= n; ++i) total += i;
    
    /* Returning pointer to local variable */
    return (void*)&total; 
}

int main() {
    pthread_t tid;
    void* res;
    pthread_create(&tid, NULL, worker_sum, (void*)1000L);
    pthread_join(tid, &res);
    printf("Total = %lld\n", *(long long*)res);
    return 0;
}
```

**Analysis & Solution:**
- **The Bug:** `total` is allocated on `worker_sum`'s local thread stack. When `worker_sum` returns, its stack frame is marked invalid and is deallocated by the kernel during `pthread_join()`. The pointer `res` in `main()` points to an address in deallocated virtual memory (a **dangling pointer**). Attempting to dereference `*(long long*)res` results in reading garbage values or triggering an immediate `SIGSEGV` (segmentation fault).
- **The Fix:** Allocate memory on the **heap** using `malloc()`, or cast integer return values directly into the `void*` pointer itself (if size permits):

```c
/* Correct Approach 1: Heap Allocation */
void* worker_sum(void* arg) {
    long n = (long)arg;
    long long* total = (long long*)malloc(sizeof(long long));
    *total = 0;
    for (long i = 1; i <= n; ++i) *total += i;
    return (void*)total; /* Heap memory survives thread exit */
}

/* In main(): */
pthread_join(tid, &res);
printf("Total = %lld\n", *(long long*)res);
free(res); /* Always free heap memory */
```

---

### Problem 2: Deadlock by Self-Joining
**Question:** Explain what happens when a thread executes:
```c
int res = pthread_join(pthread_self(), NULL);
```

**Solution:**
Under the POSIX standard, an implementation must detect if a thread attempts to join itself. 
- The function will fail immediately and return the error code **`EDEADLK`** (Resource deadlock avoided).
- If an implementation does not support deadlock detection for this case, the thread would enter an infinite sleep waiting for itself to exit, freezing permanently.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: What happens if `main()` calls `pthread_exit(NULL)` instead of `return 0`?
**Answer:** In standard C, if `main()` executes `return 0;` or `exit(0);`, the entire process terminates, instantly killing all running worker threads even if they have not finished. However, if `main()` calls `pthread_exit(NULL);`, **only the main thread terminates**. The process remains alive in the operating system until the last active worker thread finishes execution.

### Q2: Can multiple threads call `pthread_join()` on the same target thread simultaneously?
**Answer:** No. POSIX explicitly states that the behavior of multiple simultaneous calls to `pthread_join()` specifying the same target thread is **undefined**. In practice, one thread may succeed while others return `EINVAL` or crash the application.

### Q3: What is a "zombie thread" and how is it eliminated?
**Answer:** A zombie thread is a terminated joinable thread whose stack and kernel descriptors remain allocated because no other thread has called `pthread_join()` on it. It is eliminated when:
1. Another thread calls `pthread_join()` specifying its handle.
2. The entire process terminates (at which point all process memory is freed by the OS).

### Q4: Can a thread that was created as `PTHREAD_CREATE_DETACHED` be converted back to `joinable`?
**Answer:** No. Thread detachment is a one-way transition. Once a thread is detached (either at creation time via attributes or later via `pthread_detach()`), it can never be made joinable again.

---

## 12. Quick Revision Sheet

- **Joinable Threads:** Must be joined with `pthread_join()`. Preserves exit code. Cleans up stack on join.
- **Detached Threads:** Cannot be joined. Discards exit code. Cleans up stack automatically upon termination.
- **`pthread_join(tid, &retval)`:**
  - Blocks until `tid` terminates.
  - Copies exit value into `*retval`.
  - Returns `0` on success, `EDEADLK` on self-join, `EINVAL` if detached.
- **`pthread_detach(tid)`:**
  - Makes `tid` detached; returns immediately.
  - Safe for a thread to detach itself: `pthread_detach(pthread_self())`.
- **Termination Return Values:**
  - Never return pointers to local stack variables!
  - Always use `malloc()` for return structures, or cast scalars: `(void*)(intptr_t)val`.
- **Main Thread Exit:**
  - `exit(0)` / `return 0` kills all threads in the process.
  - `pthread_exit(NULL)` in `main()` allows background threads to complete while main thread exits.

---

## 13. What You Should Be Able to Do

1. **Implement parallel algorithms** that synchronize worker completion using `pthread_join()`.
2. **Safely pass and retrieve computational results** from worker threads using heap-allocated memory.
3. **Prevent zombie thread leaks** in server loops using `pthread_detach()`.
4. **Distinguish clearly** between `return`, `pthread_exit()`, and `exit()` in multi-threaded programs.
5. **Diagnose and resolve errors** such as `EINVAL`, `ESRCH`, and `EDEADLK`.
6. **Calculate process memory growth** resulting from neglected joins in continuous thread pools.
