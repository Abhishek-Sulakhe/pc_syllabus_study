# 02. Thread Creation: pthread_create and Thread Handles (pthread_t)

---

## 1. The Basic Idea

In POSIX Threads (Pthreads), multithreading is not handled automatically by compiler directives. Instead, the programmer explicitly commands the operating system kernel to instantiate, schedule, and track individual execution threads.

The central primitive for launching a thread is **`pthread_create()`**:

```c
#include <pthread.h>

int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```

When a process invokes `pthread_create()`:
1. The OS kernel allocates a new **Kernel Task / Execution Context** and a private **Thread Stack** (typically $2\,\text{MB}$ to $8\,\text{MB}$).
2. An opaque, platform-dependent handle of type **`pthread_t`** is written into the location pointed to by `thread`.
3. The newly created thread immediately begins executing concurrently, entering at the function specified by the function pointer `start_routine`, with `arg` passed as its lone argument.
4. The calling thread (e.g., `main()`) returns immediately from `pthread_create()` with status code `0`, continuing its own execution concurrently with the child thread.

```
                  THE PTHREAD_CREATE LIFECYCLE
                  
   Parent Thread (main)
        |
        |---> Calls: pthread_create(&tid, NULL, worker_func, (void*)arg);
        |             |
        |             | [ OS Kernel forks execution context & private stack ]
        |             v
        |        New Thread (tid)
        |             |
        |             v
        |        Enters: worker_func(arg)
        |             |
        v             v
   Both Parent and Child execute concurrently on CPU cores!
```

---

## 2. Why We Need It: Understanding Low-Level Concurrency

In OpenMP, writing `#pragma omp parallel` spawns an entire team of anonymous worker threads managed by an invisible runtime thread pool. While convenient for scientific loop slicing, high-level pragmas fail when designing complex systems software:

### 1. Asynchronous Event-Driven Architectures
In web servers (e.g., NGINX, Apache) or database engines (e.g., PostgreSQL, MySQL), threads must have distinct, persistent roles:
- Thread 1: Listening on a network socket (`accept()`).
- Thread 2: Disk I/O logging to an append-only journal.
- Threads 3..N: Worker pool processing client transactions.
OpenMP's fork-join loop abstraction cannot express persistent, specialized worker threads with independent lifecycles. Pthreads was engineered precisely for this.

### 2. Fine-Grained Kernel Control
Pthreads allows software engineers to configure the exact operating system parameters of each individual thread:
- Setting custom memory stack sizes (reducing stack from $8\,\text{MB}$ to $64\,\text{KB}$ to support $100{,}000$ concurrent threads).
- Assigning real-time scheduling policies (`SCHED_FIFO`, `SCHED_RR`) and deterministic priority levels.
- Direct CPU core pinning (processor affinity via `pthread_setaffinity_np`).

---

## 3. How It Works: Detailed Parameter Anatomy of `pthread_create`

Let us dissect the four arguments of `pthread_create` with rigorous architectural depth:

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

### Argument 1: `pthread_t *thread` (Out-Parameter)
- Points to a memory buffer where the Pthreads library stores the **Thread Handle** (unique identifier) of the newly created thread.
- If creation succeeds, the calling thread uses this `pthread_t` handle to join (`pthread_join`), detach (`pthread_detach`), or signal (`pthread_kill`) the child thread.

#### The Opaque Nature of `pthread_t`:
A universal mistake made by novice programmers is treating `pthread_t` as an integer:
```c
/* DANGEROUS / NON-PORTABLE ASSUMPTION */
printf("Thread ID = %d\n", (int)tid); 
if (tid1 == tid2) ... // WRONG!
```
The POSIX standard strictly defines `pthread_t` as an **Opaque Data Type**:
- On Linux (`glibc`), `pthread_t` is an `unsigned long int` (storing the virtual memory address of the internal `struct pthread`).
- On older Solaris, it was an `unsigned int`.
- On some BSD platforms and Darwin (macOS), it is a pointer to an internal struct (`typedef struct _opaque_pthread_t *pthread_t`).

#### Standard Comparison and Query APIs:
Because `pthread_t` might be a complex struct on certain POSIX systems, you **must never use `==`** to compare thread IDs! Use the standard API:

```c
#include <pthread.h>

/* Compare two thread handles: Returns non-zero (true) if equal, 0 if different */
int pthread_equal(pthread_t t1, pthread_t t2);

/* Obtain the calling thread's own thread handle */
pthread_t pthread_self(void);
```

---

### Argument 2: `const pthread_attr_t *attr` (Attributes Pointer)
- Points to a thread attributes object that defines properties such as stack size, scheduling policy, guard size, and detach state.
- **Passing `NULL`:** Instructs the runtime to use the operating system's **Default Attributes** (Joinable thread, standard default stack size of $2\text{--}8\,\text{MB}$, default time-sharing scheduler `SCHED_OTHER`).

---

### Argument 3: `void *(*start_routine)(void *)` (Thread Entry Function)
This parameter is a **C Function Pointer**. The function that the new thread begins executing must conform strictly to this mathematical type signature:

$$\text{Function Signature: } \text{void*} \quad \text{FunctionName}(\text{void}* \quad \text{arg})$$

- **Input:** It must accept exactly **one argument** of generic pointer type `void*`.
- **Output:** It must return a generic pointer of type `void*` (which can be retrieved later by `pthread_join()`).

---

### Argument 4: `void *arg` (Argument Payload)
- A single generic pointer passed directly to `start_routine` as its argument.
- To pass a primitive integer: cast it to `void*` (or pass its address).
- To pass multiple arguments: package them into a dynamically allocated heap `struct` and pass the pointer to the struct.
- If the thread requires no arguments, pass `NULL`.

---

## 4. Return Value Semantics & Error Handling

Unlike standard Unix system calls (`fork()`, `read()`, `write()`) which return `-1` on error and set the global variable `errno`, **Pthreads functions do NOT set `errno`!**

### Why Pthreads Avoids the Global `errno`:
Setting a global variable in a multi-threaded process creates an immediate data race: if Thread A suffers an error and sets `errno = 2`, and Thread B simultaneously suffers an error and sets `errno = 12`, Thread A will read Thread B's error!

### The Pthreads Error Protocol:
- If `pthread_create()` succeeds, it returns **`0`**.
- If `pthread_create()` fails, **it returns the positive error code directly as its integer return value**:

```c
int rc = pthread_create(&tid, NULL, worker, NULL);
if (rc != 0) {
    /* Handle error directly using rc */
    fprintf(stderr, "Error creating thread: %s\n", strerror(rc));
}
```

| Return Code | Symbolic Macro | Meaning / Cause |
|:---:|:---|:---|
| **`0`** | `SUCCESS` | Thread created and launched successfully. |
| **`EAGAIN`** | *Resource Temporarily Unavailable* | System lacked resources to create another thread, or the system-imposed process thread limit (`/proc/sys/kernel/threads-max` or `ulimit -u`) was exceeded. |
| **`EINVAL`** | *Invalid Argument* | The attributes settings passed in `attr` are invalid or contradictory. |
| **`EPERM`** | *Operation Not Permitted* | The caller lacks permission to set the scheduling policy or priority specified in `attr`. |

---

## 5. Compilable C Code Examples

### 5.1 Program: Complete Demonstration of `pthread_create`, Handles, and Error Handling
Save this file as `pthreads_creation_demo.c`:

```c
/* =========================================================================
 * File: pthreads_creation_demo.c
 * Description: Demonstrates:
 *              1. Launching multiple POSIX threads via pthread_create
 *              2. Inspecting thread IDs with pthread_self()
 *              3. Portable thread comparison using pthread_equal()
 *              4. Proper Pthreads error-return handling
 *              5. Passing and casting generic void* arguments
 * Compilation:
 *   gcc -O2 -pthread pthreads_creation_demo.c -o pthreads_demo
 * Execution:
 *   ./pthreads_demo
 * ========================================================================= */

#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <unistd.h>

#define NUM_THREADS 4

/* Structure to pass multiple arguments to thread */
typedef struct {
    int thread_idx;
    int work_units;
    char task_name[32];
} ThreadTask;

/* Thread Entry Function matching signature: void* func(void*) */
void* worker_routine(void *arg) {
    /* 1. Cast generic void* back to concrete struct pointer */
    ThreadTask *task = (ThreadTask*)arg;

    /* 2. Query calling thread's own handle */
    pthread_t my_tid = pthread_self();

    printf("  [Child Thread] Index: %d | Task: '%s' | Internal TID Handle: %lu\n",
           task->thread_idx, task->task_name, (unsigned long)my_tid);

    /* Simulate work */
    usleep(50000); // 50 milliseconds

    /* Return exit status (NULL in this basic demo) */
    return NULL;
}

int main(void) {
    printf("============================================================\n");
    printf("    POSIX Thread Creation (pthread_create) Demonstration    \n");
    printf("============================================================\n\n");

    pthread_t thread_handles[NUM_THREADS];
    ThreadTask task_payloads[NUM_THREADS];

    pthread_t main_thread_id = pthread_self();
    printf("[Main Thread] Main Thread ID Handle = %lu\n\n", (unsigned long)main_thread_id);

    printf("--- Phase 1: Launching %d Worker Threads ---\n", NUM_THREADS);

    for (int i = 0; i < NUM_THREADS; i++) {
        /* Initialize unique task payload per thread */
        task_payloads[i].thread_idx = i;
        task_payloads[i].work_units = (i + 1) * 100;
        snprintf(task_payloads[i].task_name, sizeof(task_payloads[i].task_name),
                 "ComputeKernel_%c", 'A' + i);

        /* Launch Thread */
        int rc = pthread_create(&thread_handles[i],   /* Pointer to handle */
                                NULL,                 /* Default attributes */
                                worker_routine,       /* Entry routine */
                                (void*)&task_payloads[i]); /* Argument pointer */

        if (rc != 0) {
            fprintf(stderr, "FATAL: pthread_create failed for thread %d: %s\n",
                    i, strerror(rc));
            exit(EXIT_FAILURE);
        }

        printf("  -> Successfully created Thread %d (Handle: %lu)\n",
               i, (unsigned long)thread_handles[i]);
    }

    printf("\n--- Phase 2: Demonstrating pthread_equal() ---\n");
    if (pthread_equal(thread_handles[0], thread_handles[1])) {
        printf("  Handles 0 and 1 are identical (Unexpected!).\n");
    } else {
        printf("  [PASS] Handles 0 and 1 are confirmed DISTINCT via pthread_equal().\n");
    }

    if (pthread_equal(thread_handles[0], thread_handles[0])) {
        printf("  [PASS] Handle 0 equals itself via pthread_equal().\n\n");
    }

    /* Wait for all child threads to finish */
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(thread_handles[i], NULL);
    }

    printf("All worker threads joined successfully. Program exiting cleanly.\n");
    printf("============================================================\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O2 -pthread pthreads_creation_demo.c -o pthreads_demo
$ ./pthreads_demo
```

```text
============================================================
    POSIX Thread Creation (pthread_create) Demonstration    
============================================================

[Main Thread] Main Thread ID Handle = 140348700288832

--- Phase 1: Launching 4 Worker Threads ---
  -> Successfully created Thread 0 (Handle: 140348683527936)
  -> Successfully created Thread 1 (Handle: 140348675135232)
  [Child Thread] Index: 0 | Task: 'ComputeKernel_A' | Internal TID Handle: 140348683527936
  -> Successfully created Thread 2 (Handle: 140348666742528)
  [Child Thread] Index: 1 | Task: 'ComputeKernel_B' | Internal TID Handle: 140348675135232
  -> Successfully created Thread 3 (Handle: 140348658349824)
  [Child Thread] Index: 2 | Task: 'ComputeKernel_C' | Internal TID Handle: 140348666742528
  [Child Thread] Index: 3 | Task: 'ComputeKernel_D' | Internal TID Handle: 140348658349824

--- Phase 2: Demonstrating pthread_equal() ---
  [PASS] Handles 0 and 1 are confirmed DISTINCT via pthread_equal().
  [PASS] Handle 0 equals itself via pthread_equal().

All worker threads joined successfully. Program exiting cleanly.
============================================================
```

Notice how thread handles are large 64-bit addresses ($140348\dots$), and child threads execute concurrently and asynchronously with main.

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 OS Kernel Thread Creation Cost Model
Calling `pthread_create()` is an explicit OS system call (`sys_clone` on Linux). It is fundamentally heavier than OpenMP's reuse of sleeping threads from a runtime thread pool:

$$T_{\text{pthread\_create}} = \tau_{\text{sys\_call}} + \tau_{\text{stack\_mmap}} + \tau_{\text{task\_struct}} + \tau_{\text{sched\_enqueue}}$$

Where:
- $\tau_{\text{sys\_call}}$ is the user-to-kernel mode switch latency ($\approx 100\text{--}300\,\text{ns}$).
- $\tau_{\text{stack\_mmap}}$ is the time required by the kernel virtual memory manager to allocate the thread stack using `mmap(MAP_ANONYMOUS | MAP_STACK)` ($\approx 2\text{--}10\,\mu\text{s}$).
- $\tau_{\text{task\_struct}}$ is the allocation and initialization of the kernel `task_struct` and Thread Local Storage (TLS).
- $\tau_{\text{sched\_enqueue}}$ is the insertion of the thread into the OS runqueue (e.g., Linux Completely Fair Scheduler - CFS).

#### Practical Performance Rule:
$$\text{Average Latency of } \text{pthread\_create}() \approx \mathbf{5\,\mu\text{s} \text{ to } 25\,\mu\text{s}}$$

> [!CAUTION]
> Because creating a raw pthread takes $\approx 10\,\mu\text{s}$ ($30{,}000$ CPU clock cycles), you should **never spawn pthreads inside an inner loop**! Pthreads should be created once at program startup as persistent workers, or managed via an application-level **Thread Pool**.

---

## 7. Advantages & Limitations

### Advantages of Explicit Thread Creation
1. **Surgical Control:** Full control over which thread runs what code, how memory is allocated, and what attributes are assigned.
2. **Dynamic Spawning:** Threads can be spawned asynchronously in response to network packets, user GUI clicks, or queue events.
3. **Zero Language Extensions:** Pure C code; requires no special compiler pragmas.

### Limitations & Traps
1. **High Overhead per Creation:** Spawning threads continuously degrades performance compared to pre-warmed thread pools.
2. **Error-Prone Parameter Passing:** Passing addresses of stack variables from a loop index causes race conditions (covered in detail in Subtopic 08.05).
3. **No Automatic Joining:** Forgetting to join or detach threads leaks kernel memory.

---

## 8. Formal Definitions for Exam

- **`pthread_t`:** An opaque, implementation-defined data type defined in `<pthread.h>` that uniquely identifies a POSIX thread within a process.
- **`pthread_create()`:** The POSIX system API that creates and launches a new thread of execution within the calling process, running the specified start routine.
- **Start Routine:** The function pointer passed to `pthread_create` matching the signature `void* (*)(void*)` that serves as the entry point for the new thread.
- **`pthread_self()`:** A Pthreads library function that returns the `pthread_t` handle of the calling thread.
- **`pthread_equal()`:** A portable Pthreads library macro or function that evaluates whether two `pthread_t` handles refer to the exact same thread.
- **Opaque Data Type:** A data structure whose physical representation (whether integer, pointer, or struct) is deliberately hidden from the programmer to guarantee platform portability and encapsulation.

---

## 9. Comparison Table

### Thread Creation across Parallel Programming Models

| Feature | POSIX Threads (`pthread_create`) | OpenMP (`#pragma omp parallel`) | Windows API (`CreateThread`) |
|:---|:---|:---|:---|
| **Programming Style** | Explicit Library API | Implicit Compiler Directive | Explicit Win32 API |
| **Creation Trigger** | Per-thread function call | Region entry (Team fork) | Per-thread function call |
| **Thread Handle** | `pthread_t` (Opaque) | None exposed directly (Integer TID) | `HANDLE` (Win32 kernel handle)|
| **Start Routine** | `void* func(void*)` | Outlined static `.omp_fn` | `DWORD WINAPI func(LPVOID)` |
| **Error Reporting** | Function return value (`errno` code)| Runtime abort or sequential fallback | Returns `NULL`, sets `GetLastError()` |
| **Compilation Flag** | `-pthread` | `-fopenmp` | None (Default in MSVC CRT) |

---

## 10. Solved Analytical Problems

### Problem 1: Diagnosing the Non-Portable `printf("%d", tid)`
**Question:** Explain why the following code line fails on macOS or 64-bit Linux with strict compiler warning flags (`-Wall -Werror`), and provide the portable alternative.

```c
pthread_t tid = pthread_self();
printf("Thread ID: %d\n", tid);
```

**Solution:**
1. **The Bug:**
   - On 64-bit Linux, `pthread_t` is `typedef unsigned long int pthread_t;`. Passing it to `%d` triggers a format specifier mismatch warning (`format '%d' expects argument of type 'int', but argument has type 'long unsigned int'`).
   - On macOS, `pthread_t` is a pointer (`struct _opaque_pthread_t*`). Passing a pointer to `%d` is an illegal type conversion.
   - Under `-Werror`, this warning halts compilation immediately.
2. **Portable Approach:**
   POSIX does not guarantee any printable scalar representation for `pthread_t`. If numerical printing is needed for debugging, cast it explicitly to `(unsigned long)` on Linux/POSIX:
   ```c
   printf("Thread ID: %lu\n", (unsigned long)tid);
   ```
   Or on Linux-specific systems, use the non-portable kernel Thread ID (TID) via `gettid()`:
   ```c
   #define _GNU_SOURCE
   #include <unistd.h>
   printf("Kernel TID: %d\n", gettid());
   ```

---

### Problem 2: Error Handling Anti-Pattern
**Question:** A student writes:
```c
pthread_t tid;
if (pthread_create(&tid, NULL, worker, NULL) < 0) {
    perror("pthread_create failed");
}
```
Explain the two major bugs in this error-checking code.

**Solution:**
1. **Bug 1 (Checking `< 0`):**
   - `pthread_create()` **never returns negative numbers**.
   - On failure, it returns a **positive integer error code** (such as `EAGAIN = 11` or `EINVAL = 22`).
   - Checking `< 0` will evaluate to false even when `pthread_create()` fails catastrophically! The failure will be silently ignored.
2. **Bug 2 (Calling `perror`):**
   - `perror()` reads the global C variable `errno`.
   - `pthread_create()` **does not set `errno`**!
   - `perror()` will print whatever unrelated error occurred previously in the program (or *"Success"*), misleading the engineer.
3. **The Correct Implementation:**
   ```c
   int rc = pthread_create(&tid, NULL, worker, NULL);
   if (rc != 0) {
       fprintf(stderr, "pthread_create failed: %s\n", strerror(rc));
       exit(EXIT_FAILURE);
   }
   ```

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Can a thread start executing before `pthread_create()` returns in the parent thread?
**Answer:** **YES.** As soon as the kernel sets up the child thread's context and places it on the runnable queue, the OS scheduler is free to grant it a CPU time slice immediately. It is entirely possible that the child thread starts, executes its entire function, and terminates *before* the parent thread even returns from `pthread_create()`! Code must never rely on execution order between parent and child.

### Q2: Why does `pthread_create` take a pointer to `pthread_t` (`pthread_t *thread`) instead of returning the handle directly?
**Answer:** Because in POSIX, the integer return value of a function is strictly reserved for the **error code** (`0` for success, positive `errno` for error). Therefore, any output data produced by the function must be returned via pointer arguments (out-parameters).

### Q3: What happens if `pthread_create` runs out of system memory?
**Answer:** It fails and immediately returns **`EAGAIN`** (Resource temporarily unavailable). This indicates either that the OS cannot allocate the requested thread stack memory, or that the process has reached its system thread ceiling (`/proc/sys/kernel/threads-max`). The process does not crash; the parent can handle the error gracefully.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    PTHREAD_CREATE CHEAT SHEET                                      |
+----------------------------------------------------------------------------------------------------+
| Signature              | int pthread_create(pthread_t *tid, const pthread_attr_t *attr,            |
|                        |                    void *(*routine)(void *), void *arg);                  |
| Return Value           | 0 on SUCCESS; Positive errno code on failure (Does NOT set errno!).       |
| Common Errors          | EAGAIN (Thread limit/memory exhausted), EINVAL (Bad attributes).          |
+----------------------------------------------------------------------------------------------------+
|                                      THREAD HANDLE RULES                                           |
+----------------------------------------------------------------------------------------------------+
| Type                   | pthread_t (OPAQUE data type - never assume integer!).                     |
| Comparison             | pthread_equal(t1, t2) (Never use ==!).                                    |
| Query Self             | pthread_t my_id = pthread_self();                                         |
+----------------------------------------------------------------------------------------------------+
|                                  THE CRITICAL ANTI-PATTERNS                                        |
+----------------------------------------------------------------------------------------------------+
| NEVER check: if (pthread_create(...) < 0)  ---> Always check: if (rc != 0)                         |
| NEVER call: perror(...)                    ---> Always call: strerror(rc)                          |
| NEVER spawn raw pthreads in an inner loop  ---> Creation takes ~10 microseconds! Use thread pools. |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the four arguments and return value protocol of `pthread_create()`.
2. Treat `pthread_t` portably as an opaque type and compare handles using `pthread_equal()`.
3. Correctly write thread entry functions matching the `void* (*)(void*)` signature.
4. Implement rigorous Pthreads error handling without relying on global `errno` or `perror()`.
5. Quantify the OS system call and memory latency overhead of raw thread creation.
