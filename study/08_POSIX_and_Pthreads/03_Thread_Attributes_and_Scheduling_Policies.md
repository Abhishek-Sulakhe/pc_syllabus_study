# Module 08: POSIX Threads Foundations (Lectures 17–19)
# Subtopic 08.03: Thread Attributes and Scheduling Policies (`pthread_attr_t`, `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR`)

---

## 1. Basic Idea

When a thread is created using `pthread_create(&tid, NULL, start_routine, arg)`, passing `NULL` as the second argument directs the POSIX runtime to configure the new thread with system-wide default attributes. While default attributes suffice for introductory programs, high-performance computing (HPC), real-time embedded systems, and massively multithreaded servers require fine-grained control over execution parameters.

A thread attribute object, represented by the opaque type `pthread_attr_t`, encapsulates all configuration parameters that dictate how the operating system kernel allocates resources, schedules execution time, and manages the lifecycle of an individual thread. Rather than passing dozens of parameters to `pthread_create()`, POSIX uses an attribute object pattern:
1. Initialize an attribute object to defaults (`pthread_attr_init()`).
2. Set individual attributes via dedicated setter functions (e.g., stack size, scheduling policy, detach state).
3. Pass the configured attribute object to one or more `pthread_create()` invocations.
4. Destroy the attribute object when thread creation is complete (`pthread_attr_destroy()`).

```
+-----------------------------------------------------------------------------------+
|                        pthread_attr_t Configuration Space                         |
+-----------------------------------------------------------------------------------+
| 1. Detach State       : PTHREAD_CREATE_JOINABLE vs PTHREAD_CREATE_DETACHED        |
| 2. Stack Size         : PTHREAD_STACK_MIN to multi-megabyte allocations           |
| 3. Stack Guard Size   : Page size multiple protecting against silent stack growth |
| 4. Scheduling Policy  : SCHED_OTHER (Normal), SCHED_FIFO (RT), SCHED_RR (RT)      |
| 5. Scheduling Priority: Priority integer within [min_prio, max_prio] range        |
| 6. Sched Inheritance  : PTHREAD_INHERIT_SCHED vs PTHREAD_EXPLICIT_SCHED           |
| 7. CPU Affinity       : Allowed CPU core mask (CPU_SET, pthread_attr_setaffinity) |
+-----------------------------------------------------------------------------------+
```

Modifying an attribute object after a thread has already been created has **no effect** on the existing thread. The object acts solely as a blueprint during `pthread_create()`.

---

## 2. Why We Need It

### The Virtual Memory Wall (Massive Thread Counts)
On 64-bit Linux architectures (such as x86_64), the default stack allocation per thread is typically **8 Megabytes** (governed by the system limit `ulimit -s`). If an application attempts to spawn 10,000 concurrent worker threads to handle discrete network events or lightweight computational tasks:
$$\text{Virtual Memory Required} = 10{,}000 \times 8\text{ MB} = 80{,}000\text{ MB} \approx 78.125\text{ GB}$$

Even though Linux uses demand paging (allocating physical RAM pages only upon access), reserving 8 MB of contiguous virtual address space per thread rapidly fragments and exhausts process address mappings, leading to `EAGAIN` or `ENOMEM` errors during `pthread_create()`. By explicitly reducing the stack size to 64 KB or 128 KB via `pthread_attr_setstacksize()`, 10,000 threads require only:
$$10{,}000 \times 64\text{ KB} = 640\text{ MB}$$
This allows massive concurrency within modest virtual memory bounds.

### Deterministic Latency and Real-Time Deadlines
Standard OS threads operate under `SCHED_OTHER` (governed in Linux by the Completely Fair Scheduler or CFS). CFS prioritizes throughput and fairness across all processes, dynamically penalizing CPU-heavy tasks and introducing non-deterministic scheduling latency (tens of milliseconds). 

For critical HPC tasks (such as ultra-low-latency financial order processing, hardware sensor polling, or deterministic audio signal processing), threads must execute deterministically without being preempted by background daemon processes. Configuring `SCHED_FIFO` or `SCHED_RR` with elevated real-time priorities guarantees immediate CPU preemption over non-real-time tasks.

### Lifecycle Decoupling
If a parent thread spawns an independent worker to log diagnostics or flush disk caches and has no need to inspect its return code or synchronize its termination, configuring the thread as `PTHREAD_CREATE_DETACHED` ensures that its OS resources and kernel thread structures are automatically reclaimed by the kernel upon termination, preventing memory leakage without requiring the parent to call `pthread_join()`.

---

## 3. How It Works

### The Lifecycle of `pthread_attr_t`

The attribute object must follow a strict lifecycle protocol:

```
           +--------------------------+
           | pthread_attr_t attr;     |  (Uninitialized storage)
           +--------------------------+
                        |
                        | pthread_attr_init(&attr)
                        v
           +--------------------------+
           | Default Attributes Set   |  (Default stack, joinable, SCHED_OTHER)
           +--------------------------+
                        |
                        | pthread_attr_setstacksize(&attr, size)
                        | pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)
                        | pthread_attr_setschedpolicy(&attr, SCHED_FIFO)
                        v
           +--------------------------+
           | Custom Attributes Ready  |
           +--------------------------+
                   /          \
  pthread_create(&t1, &attr..) pthread_create(&t2, &attr..)  (Can reuse for multiple threads)
                   \          /
                        v
           +--------------------------+
           | pthread_attr_destroy()   |  (Releases internal dynamic resources)
           +--------------------------+
```

### Memory Architecture of a Thread Stack and Guard Page

Each thread's stack is allocated independently in user space (typically via `mmap()` by the threading runtime library `libpthread` / `libc`). Above (or below, depending on hardware growth direction) the allocated stack pages sits an unmapped **guard page**:

```
High Memory Address
+------------------------------------------+
| Thread Stack Base (Initial RSP)          |  <--- Stack pointer grows downward
| [Local variables, frame pointers, etc.]  |       towards lower addresses
|                                          |
|                                          |
|                                          |
+------------------------------------------+  <--- Base + guardsize
| GUARD PAGE (Size = pthread_attr_setguard)|  PROT_NONE (No Read, No Write)
+------------------------------------------+
Low Memory Address

[Attempted write into Guard Page -> Hardware Memory Management Unit (MMU) Fault]
                                  |
                                  v
                         Operating System sends
                              SIGSEGV
```

If a thread enters deep recursion or allocates an oversized local array exceeding its stack size, the stack pointer enters the Guard Page. Because the OS kernel maps this page with permissions `PROT_NONE`, the CPU's MMU triggers a page fault that the kernel converts into a `SIGSEGV` (segmentation fault), instantly stopping the process and preventing silent corruption of neighboring memory regions.

### Linux Scheduling Classes and Policies

The Linux kernel dispatches threads according to scheduling classes ordered by priority:

```
Priority 99 (Highest)
    ^
    |  +---------------------------------------------------------------+
    |  | Real-Time Class: SCHED_FIFO                                    |
    |  | - Run-to-completion or until blocked/yielded                  |
    |  | - Never preempted by lower-priority threads                   |
    |  +---------------------------------------------------------------+
    |  | Real-Time Class: SCHED_RR                                      |
    |  | - Preempted only by higher-priority threads                   |
    |  | - Round-robin among same-priority threads after time quantum   |
    |  +---------------------------------------------------------------+
    |                                                                   
    |  ====================== Real-Time / Normal Boundary =============
    |                                                                   
    |  +---------------------------------------------------------------+
    |  | Fair / Batch Class: SCHED_OTHER (CFS)                         |
    |  | - Completely Fair Scheduler (vruntime tracking)               |
    |  | - Dynamic priority adjustments based on nice values (-20 to 19)|
    |  +---------------------------------------------------------------+
Priority 0 (Lowest)
```

1. **`SCHED_OTHER` (Default Time-Sharing):**
   - The standard scheduling policy for Linux processes and threads.
   - Operates with static priority `0`.
   - Driven by the Completely Fair Scheduler (CFS), which balances CPU execution time using a red-black tree indexed by virtual runtime (`vruntime`).
   - Threads yielding or waiting on I/O receive latency bonuses; CPU-bound threads get time-sliced fairly.

2. **`SCHED_FIFO` (First-In, First-Out Real-Time):**
   - Fixed priority ranging from `1` (lowest RT priority) to `99` (highest RT priority).
   - Once a `SCHED_FIFO` thread gains a CPU core, it runs **indefinitely** until:
     1. It voluntarily blocks on an I/O operation, mutex, semaphore, or condition variable.
     2. It voluntarily yields the processor by calling `sched_yield()`.
     3. A higher-priority real-time thread becomes runnable.
   - **No time-slicing:** Threads of equal `SCHED_FIFO` priority will not interrupt each other.

3. **`SCHED_RR` (Round-Robin Real-Time):**
   - Identical to `SCHED_FIFO` in priority range (`1` to `99`) and preemption of lower-priority tasks.
   - Threads of **equal priority** execute in round-robin fashion, each receiving a fixed time quantum (typically 100 ms on Linux, queryable via `sched_rr_get_interval()`).
   - When the time quantum expires, the thread is moved to the tail of its priority queue, allowing peer threads of identical priority to execute.

---

## 4. Important Mechanisms and Functions

### 1. Attribute Initialization and Destruction

```c
#include <pthread.h>

int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
```
- `pthread_attr_init()` initializes the structure with system defaults. Returns `0` on success, or an error number (`ENOMEM`).
- `pthread_attr_destroy()` invalidates the object, freeing any internal dynamic memory. Attempting to use a destroyed attribute object results in `EINVAL`.

### 2. Stack Size and Guard Configuration

```c
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_getstacksize(const pthread_attr_t *attr, size_t *stacksize);

int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
int pthread_attr_getguardsize(const pthread_attr_t *attr, size_t *guardsize);
```
- `stacksize`: Must be at least `PTHREAD_STACK_MIN` (typically 16,384 bytes / 16 KB on x86_64 Linux). It should be an integer multiple of the system page size (typically 4096 bytes).
- Returns `EINVAL` if `stacksize < PTHREAD_STACK_MIN` or is improperly aligned on certain architectures.
- `guardsize`: Defaults to one system page (4 KB). Passing `0` disables the guard page, saving virtual memory at the expense of stack overflow safety.

### 3. Detach State Configuration

```c
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
```
- `detachstate` options:
  - `PTHREAD_CREATE_JOINABLE` (Default): The thread's termination code and resources remain preserved in the kernel until another thread calls `pthread_join()`.
  - `PTHREAD_CREATE_DETACHED`: The thread's resources are automatically reclaimed by the kernel immediately upon exit.

### 4. Scheduling Inheritance, Policy, and Priority

```c
#include <sched.h>

int pthread_attr_setinheritsched(pthread_attr_t *attr, int inheritsched);
int pthread_attr_getinheritsched(const pthread_attr_t *attr, int *inheritsched);

int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy);
int pthread_attr_getschedpolicy(const pthread_attr_t *attr, int *policy);

int pthread_attr_setschedparam(pthread_attr_t *attr, const struct sched_param *param);
int pthread_attr_getschedparam(const pthread_attr_t *attr, struct sched_param *param);
```

#### The Inheritance Trap (`PTHREAD_EXPLICIT_SCHED`)
By default on Linux, attribute objects have their inheritance attribute set to:
$$\text{inheritsched} = \text{PTHREAD\_INHERIT\_SCHED}$$
Under `PTHREAD_INHERIT_SCHED`, any scheduling policy (`setschedpolicy`) or priority (`setschedparam`) defined in the attribute object is **completely ignored**! The child thread automatically inherits the policy and priority of the calling (parent) thread.

To enforce custom scheduling policies, you **must** explicitly call:
```c
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
```

#### Querying Priority Ranges
Real-time priorities must fall within valid POSIX boundaries:
```c
int sched_get_priority_min(int policy);
int sched_get_priority_max(int policy);
```
- For `SCHED_OTHER`: Min = `0`, Max = `0`.
- For `SCHED_FIFO` / `SCHED_RR`: Min = `1`, Max = `99` (on Linux).

---

## 5. Compilable C Code Examples

### Example 1: Custom Stack Size and Detached Thread Management
This program demonstrates how to safely configure a 128 KB custom stack, set the detach state to detached, inspect default vs. custom stack sizes, and safely clean up the attribute object.

```c
/*
 * File: thread_stack_detach.c
 * Compilation:
 *   gcc -O2 -pthread thread_stack_detach.c -o thread_stack_detach
 *
 * Execution:
 *   ./thread_stack_detach
 */

#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <limits.h>

void* detached_worker(void* arg) {
    long thread_id = (long)arg;
    pthread_attr_t current_attr;
    size_t stack_size = 0;
    size_t guard_size = 0;

    /* Initialize and query running attributes */
    pthread_getattr_np(pthread_self(), &current_attr);
    pthread_attr_getstacksize(&current_attr, &stack_size);
    pthread_attr_getguardsize(&current_attr, &guard_size);
    pthread_attr_destroy(&current_attr);

    printf("[Worker %ld] Running: Active Stack Size = %zu KB, Guard Size = %zu bytes\n",
           thread_id, stack_size / 1024, guard_size);

    /* Allocate local data on stack to verify headroom */
    char buffer[16 * 1024]; /* 16 KB local frame */
    memset(buffer, 0xAA, sizeof(buffer));

    printf("[Worker %ld] Successfully touched 16 KB stack buffer. Terminating.\n", thread_id);
    return NULL;
}

int main(void) {
    pthread_t thread;
    pthread_attr_t attr;
    int status;
    size_t default_stack;
    const size_t custom_stack = 128 * 1024; /* 128 KB stack */

    printf("=== POSIX Thread Attributes: Custom Stack & Detached State ===\n");

    /* Step 1: Initialize attribute object */
    status = pthread_attr_init(&attr);
    if (status != 0) {
        fprintf(stderr, "Error: pthread_attr_init failed: %s\n", strerror(status));
        return EXIT_FAILURE;
    }

    /* Query default stack size */
    pthread_attr_getstacksize(&attr, &default_stack);
    printf("[Main] Default thread stack size: %zu MB (%zu bytes)\n",
           default_stack / (1024 * 1024), default_stack);

    /* Step 2: Set custom stack size */
    if (custom_stack < PTHREAD_STACK_MIN) {
        fprintf(stderr, "Error: Requested stack smaller than PTHREAD_STACK_MIN (%d)\n",
                PTHREAD_STACK_MIN);
        pthread_attr_destroy(&attr);
        return EXIT_FAILURE;
    }

    status = pthread_attr_setstacksize(&attr, custom_stack);
    if (status != 0) {
        fprintf(stderr, "Error: pthread_attr_setstacksize failed: %s\n", strerror(status));
        pthread_attr_destroy(&attr);
        return EXIT_FAILURE;
    }

    /* Step 3: Set detach state */
    status = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    if (status != 0) {
        fprintf(stderr, "Error: pthread_attr_setdetachstate failed: %s\n", strerror(status));
        pthread_attr_destroy(&attr);
        return EXIT_FAILURE;
    }

    /* Step 4: Create detached thread using configured attribute */
    printf("[Main] Spawning detached worker with 128 KB stack...\n");
    status = pthread_create(&thread, &attr, detached_worker, (void*)1L);
    if (status != 0) {
        fprintf(stderr, "Error: pthread_create failed: %s\n", strerror(status));
        pthread_attr_destroy(&attr);
        return EXIT_FAILURE;
    }

    /* Step 5: Destroy attribute object (no longer needed) */
    pthread_attr_destroy(&attr);

    /* Since thread is detached, pthread_join(thread, NULL) would fail with EINVAL.
       Sleep briefly to allow detached worker to finish before main exits. */
    usleep(200000); /* 200 milliseconds */
    printf("[Main] Completed execution successfully.\n");

    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== POSIX Thread Attributes: Custom Stack & Detached State ===
[Main] Default thread stack size: 8 MB (8388608 bytes)
[Main] Spawning detached worker with 128 KB stack...
[Worker 1] Running: Active Stack Size = 128 KB, Guard Size = 4096 bytes
[Worker 1] Successfully touched 16 KB stack buffer. Terminating.
[Main] Completed execution successfully.
```

---

### Example 2: Configuring Real-Time Scheduling Policies (`SCHED_FIFO` / `SCHED_RR`)
This program illustrates how to query scheduling boundaries, set `PTHREAD_EXPLICIT_SCHED`, configure `SCHED_FIFO` with real-time priority, and handle permission checks (`EPERM`).

```c
/*
 * File: thread_realtime_sched.c
 * Compilation:
 *   gcc -O2 -pthread thread_realtime_sched.c -o thread_realtime_sched
 *
 * Execution:
 *   ./thread_realtime_sched          (Standard run: may fall back if unprivileged)
 *   sudo ./thread_realtime_sched     (Elevated run: activates true SCHED_FIFO)
 */

#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sched.h>
#include <unistd.h>
#include <errno.h>

void* rt_worker(void* arg) {
    long id = (long)arg;
    int policy;
    struct sched_param param;

    pthread_getschedparam(pthread_self(), &policy, &param);

    const char* policy_str = (policy == SCHED_FIFO) ? "SCHED_FIFO" :
                             (policy == SCHED_RR)   ? "SCHED_RR"   :
                             (policy == SCHED_OTHER)? "SCHED_OTHER": "UNKNOWN";

    printf("[Worker %ld] Active Policy: %s | Real-Time Priority: %d\n",
           id, policy_str, param.sched_priority);

    return NULL;
}

int main(void) {
    pthread_t thread;
    pthread_attr_t attr;
    struct sched_param param;
    int status;
    int min_prio, max_prio;

    printf("=== POSIX Real-Time Thread Scheduling Configuration ===\n");

    /* Step 1: Query priority boundaries for SCHED_FIFO */
    min_prio = sched_get_priority_min(SCHED_FIFO);
    max_prio = sched_get_priority_max(SCHED_FIFO);
    printf("[Main] System SCHED_FIFO Priority Range: [%d to %d]\n", min_prio, max_prio);

    /* Step 2: Initialize thread attribute */
    pthread_attr_init(&attr);

    /* CRITICAL STEP: Set explicit scheduling inheritance */
    status = pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
    if (status != 0) {
        fprintf(stderr, "Error: setinheritsched failed: %s\n", strerror(status));
    }

    /* Step 3: Configure scheduling policy to SCHED_FIFO */
    status = pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
    if (status != 0) {
        fprintf(stderr, "Error: setschedpolicy failed: %s\n", strerror(status));
    }

    /* Step 4: Configure real-time priority (e.g., midpoint priority) */
    param.sched_priority = min_prio + (max_prio - min_prio) / 2;
    status = pthread_attr_setschedparam(&attr, &param);
    if (status != 0) {
        fprintf(stderr, "Error: setschedparam failed: %s\n", strerror(status));
    }

    printf("[Main] Requesting SCHED_FIFO with Priority = %d...\n", param.sched_priority);

    /* Step 5: Attempt thread creation */
    status = pthread_create(&thread, &attr, rt_worker, (void*)1L);
    if (status == EPERM) {
        printf("[Notice] pthread_create returned EPERM (Operation not permitted).\n");
        printf("[Notice] Real-time policies require superuser privileges (CAP_SYS_NICE).\n");
        printf("[Main] Falling back to standard SCHED_OTHER...\n");

        /* Fallback to standard scheduling */
        pthread_attr_setschedpolicy(&attr, SCHED_OTHER);
        param.sched_priority = 0;
        pthread_attr_setschedparam(&attr, &param);

        status = pthread_create(&thread, &attr, rt_worker, (void*)1L);
        if (status != 0) {
            fprintf(stderr, "Fatal: Fallback creation failed: %s\n", strerror(status));
            pthread_attr_destroy(&attr);
            return EXIT_FAILURE;
        }
    } else if (status != 0) {
        fprintf(stderr, "Fatal: pthread_create failed: %s\n", strerror(status));
        pthread_attr_destroy(&attr);
        return EXIT_FAILURE;
    }

    pthread_attr_destroy(&attr);

    pthread_join(thread, NULL);
    printf("[Main] Real-time test completed successfully.\n");
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Non-Root User) ---
=== POSIX Real-Time Thread Scheduling Configuration ===
[Main] System SCHED_FIFO Priority Range: [1 to 99]
[Main] Requesting SCHED_FIFO with Priority = 50...
[Notice] pthread_create returned EPERM (Operation not permitted).
[Notice] Real-time policies require superuser privileges (CAP_SYS_NICE).
[Main] Falling back to standard SCHED_OTHER...
[Worker 1] Active Policy: SCHED_OTHER | Real-Time Priority: 0
[Main] Real-time test completed successfully.

--- Expected Output (Running as root / sudo) ---
=== POSIX Real-Time Thread Scheduling Configuration ===
[Main] System SCHED_FIFO Priority Range: [1 to 99]
[Main] Requesting SCHED_FIFO with Priority = 50...
[Worker 1] Active Policy: SCHED_FIFO | Real-Time Priority: 50
[Main] Real-time test completed successfully.
```

---

## 6. Mathematical Formulations & Overhead Analysis

### Virtual Memory Footprint Model

Let $N$ be the number of concurrent threads spawned in a process. Let $S_{\text{stack}}$ be the stack size per thread, and $S_{\text{guard}}$ be the guard page size. The total virtual address space reserved strictly for thread stacks is:

$$V_{\text{stacks}}(N) = N \cdot \left( S_{\text{stack}} + S_{\text{guard}} \right)$$

For 64-bit Linux with default settings:
- $S_{\text{stack}} = 8\text{ MB} = 8{,}388{,}608\text{ bytes}$
- $S_{\text{guard}} = 4\text{ KB} = 4{,}096\text{ bytes}$

$$V_{\text{default}}(N) \approx N \times 8.0039\text{ MB}$$

For $N = 20{,}000$ threads:
$$V_{\text{default}}(20{,}000) \approx 20{,}000 \times 8.0039\text{ MB} \approx 160{,}078\text{ MB} \approx 156.32\text{ GB}$$

If configured with a customized lightweight stack ($S_{\text{stack}} = 64\text{ KB}$):
$$V_{\text{tuned}}(20{,}000) = 20{,}000 \times (65{,}536 + 4{,}096)\text{ bytes} = 20{,}000 \times 69{,}632\text{ bytes} \approx 1.297\text{ GB}$$

$$\text{Memory Footprint Reduction Factor} = \frac{V_{\text{default}}}{V_{\text{tuned}}} = \frac{8{,}392{,}704}{69{,}632} \approx 120.5\times$$

### Real-Time Priority Inversion Latency

In mixed-priority systems, if a lower-priority thread $T_L$ acquires a shared lock $M$, and a higher-priority thread $T_H$ preempts $T_L$ and then requests $M$, $T_H$ blocks. If an intermediate-priority thread $T_M$ (which does not need $M$) runs, it preempts $T_L$, indirectly blocking $T_H$ for unbounded time:

```
Priority
  ^
  |  T_H (High)      [Blocked on M]......................[Acquires M, Runs]
  |
  |  T_M (Medium)                  [Preempts T_L, Runs]
  |
  |  T_L (Low)       [Locks M].....[Preempted]..........[Releases M]
  +------------------------------------------------------------------------> Time
                     ^             ^
                     |             +--- Unbounded Priority Inversion window
                     +-- T_H requests M, forced to sleep
```

To bound worst-case latency $T_{\text{latency}}$ in real-time Pthreads, mutexes must use the priority inheritance protocol (`PTHREAD_PRIO_INHERIT`), ensuring:

$$T_{\text{blocked}}(T_H) \le \max_{i \in \text{Critical Sections}} (t_{\text{cs}}(T_L)) + t_{\text{context\_switch}}$$

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Trade-offs

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Custom Stack Size         | - Enables 10,000+ thread concurrency   | - Risk of Stack Overflow / SIGSEGV      |
|                           | - Drastically reduces virtual VM use    | - Deep recursion or large buffers crash |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Detached State            | - Zero memory leaks on termination     | - Cannot retrieve thread exit status    |
|                           | - No need for parent pthread_join()     | - Cannot synchronize termination easily |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Real-Time (SCHED_FIFO)    | - Deterministic execution latency       | - Requires root/CAP_SYS_NICE privilege  |
|                           | - Strict preemption guarantee           | - CPU starvation if loop doesn't yield  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Explicit Inheritance      | - Full developer control over policy    | - Easy to omit: silently reverts to     |
| (PTHREAD_EXPLICIT_SCHED)  |   and real-time priority levels         |   PTHREAD_INHERIT_SCHED                 |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Thread Attribute Object (`pthread_attr_t`):** An opaque data structure defined in `<pthread.h>` that encapsulates the creation attributes of a thread—including stack address, stack size, guard size, detach state, scheduling policy, scheduling parameter, and inheritance—used to configure threads at creation time.
>
> **Detached Thread (`PTHREAD_CREATE_DETACHED`):** A thread whose execution state and system resources are automatically released by the operating system kernel immediately upon termination, without requiring or permitting another thread to join with it.
>
> **`SCHED_OTHER`:** The standard time-sharing scheduling policy under POSIX/Linux, governed by dynamic priority calculations (e.g., Linux CFS) where no thread possesses real-time preemption guarantees.
>
> **`SCHED_FIFO`:** A static-priority, preemptive, real-time scheduling policy where the executing thread runs to completion, blocks, or yields voluntarily, never preempted by threads of equal or lower priority.
>
> **`SCHED_RR`:** A real-time round-robin scheduling policy identical to `SCHED_FIFO`, with the addition of a fixed time quantum for threads of equal priority, preventing monopolization among peer real-time tasks.
>
> **Scheduling Inheritance (`PTHREAD_EXPLICIT_SCHED` vs `PTHREAD_INHERIT_SCHED`):** An attribute specifying whether a newly created thread inherits scheduling policy and priorities from its parent thread (`INHERIT`) or takes its parameters explicitly from the provided attribute object (`EXPLICIT`).

---

## 9. Comparison Tables

### Comparison of POSIX Scheduling Policies

| Characteristic | `SCHED_OTHER` | `SCHED_FIFO` | `SCHED_RR` |
| :--- | :--- | :--- | :--- |
| **Class** | Normal / Time-Sharing | Real-Time (RT) | Real-Time (RT) |
| **Priority Range** | Fixed at `0` | `1` to `99` (Linux) | `1` to `99` (Linux) |
| **Preemption** | Dynamic (CFS / Time slices) | Only by higher RT priority | Only by higher RT priority |
| **Peer Behavior** | Proportional CPU sharing | First-to-run executes until block/yield | Round-robin execution per time slice |
| **Starvation Risk** | Zero (guaranteed fair share) | **High** (can starve normal processes) | **High** (can starve normal processes) |
| **Privileges Needed** | None (standard user) | `CAP_SYS_NICE` / Superuser (`root`) | `CAP_SYS_NICE` / Superuser (`root`) |
| **Typical Use Case** | Web servers, batch compute, CLI tools | Motor control, sensor polling, low-latency audio | Equal-priority audio mixing, real-time streaming |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Maximum Thread Limit Under Address Space Constraints
**Question:** A high-throughput telemetry server runs on a 32-bit Linux system with a maximum user address space limit of 3 GB ($3 \times 2^{30}$ bytes). The code and heap consume 500 MB. 
1. If the default stack size is 8 MB, calculate the theoretical maximum number of threads that can be spawned before virtual memory is exhausted.
2. If the developer tunes the stack size to 256 KB with a 4 KB guard page, how many threads can theoretically be supported?

**Solution:**

**Part 1:**
- Available Virtual Address Space for Stacks = $3000\text{ MB} - 500\text{ MB} = 2500\text{ MB}$.
- Default stack per thread = $8\text{ MB} + 4\text{ KB} \approx 8.0039\text{ MB}$.
$$N_{\text{default}} = \left\lfloor \frac{2500\text{ MB}}{8.0039\text{ MB}} \right\rfloor = \left\lfloor 312.34 \right\rfloor = 312\text{ threads}$$

**Part 2:**
- Tuned stack size per thread = $256\text{ KB} + 4\text{ KB} = 260\text{ KB}$.
- Available memory in KB = $2500 \times 1024\text{ KB} = 2{,}560{,}000\text{ KB}$.
$$N_{\text{tuned}} = \left\lfloor \frac{2{,}560{,}000\text{ KB}}{260\text{ KB}} \right\rfloor = \left\lfloor 9846.15 \right\rfloor = 9{,}846\text{ threads}$$

*Conclusion:* Configuring thread stack size increases concurrency limits by over **31-fold** in virtual memory-constrained architectures.

---

### Problem 2: Detecting the Scheduling Inheritance Bug
**Question:** The following code snippet attempts to create a high-priority real-time thread running `SCHED_FIFO`. When inspected inside the thread, the policy is still `SCHED_OTHER` with priority 0. Identify the bug and write the exact fix.

```c
pthread_t tid;
pthread_attr_t attr;
struct sched_param param;

pthread_attr_init(&attr);
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
param.sched_priority = 80;
pthread_attr_setschedparam(&attr, &param);

pthread_create(&tid, &attr, worker_fn, NULL);
pthread_attr_destroy(&attr);
```

**Analysis & Fix:**
- **The Bug:** By default, POSIX attribute objects have their inheritance property set to `PTHREAD_INHERIT_SCHED`. Under this default, `pthread_attr_setschedpolicy` and `pthread_attr_setschedparam` are silently ignored during `pthread_create()`, and the new thread inherits its parent's policy (`SCHED_OTHER`).
- **The Fix:** Insert `pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);` before `pthread_create()`:

```c
pthread_attr_init(&attr);
/* FIX: Force POSIX runtime to use explicit settings from this attribute object */
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
param.sched_priority = 80;
pthread_attr_setschedparam(&attr, &param);

pthread_create(&tid, &attr, worker_fn, NULL);
pthread_attr_destroy(&attr);
```

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: What happens if a thread created with `PTHREAD_CREATE_DETACHED` is passed to `pthread_join()`?
**Answer:** `pthread_join()` immediately fails and returns the error code `EINVAL` (Invalid Argument). Once a thread is detached, the runtime disconnects its synchronization handle; its termination state cannot be waited upon or queried.

### Q2: Why does `pthread_attr_destroy()` not terminate the threads created with that attribute object?
**Answer:** The attribute object `pthread_attr_t` is purely a **configuration blueprint** read during the execution of `pthread_create()`. Once `pthread_create()` completes, the operating system kernel allocates the thread with its own independent task structures (`task_struct` in Linux). The attribute object plays no role in the active lifecycle of the running thread; destroying it simply deallocates internal metadata associated with the blueprint.

### Q3: Why can an unconstrained `SCHED_FIFO` thread lock up an entire single-core computer?
**Answer:** Because `SCHED_FIFO` is a non-time-sliced real-time policy. A `SCHED_FIFO` thread with priority $> 0$ will run indefinitely unless it voluntarily yields, performs blocking I/O, or sleeps. If a `SCHED_FIFO` thread enters an infinite computation loop (`while(1) {}`), the OS scheduler will **never** preempt it to allow lower-priority processes—including system shells, desktop GUIs, or SSH daemons—to run.

### Q4: What is the purpose of `pthread_attr_setguardsize()`?
**Answer:** It configures the size of the unmapped memory region (Guard Page) immediately following the thread's stack. When a stack overflow occurs, execution hits this read/write-protected guard page, causing a hardware memory management unit (MMU) fault that generates a `SIGSEGV`. This alerts the developer immediately rather than allowing the runaway stack to silently corrupt adjacent memory.

---

## 12. Quick Revision Sheet

- **Attribute Lifecycle:**
  `init(&attr)` $\to$ `set_*(&attr, val)` $\to$ `create(&t, &attr, ...)` $\to$ `destroy(&attr)`.
- **Default Stack Size:** 8 MB on Linux x86_64 (`ulimit -s`).
- **Minimum Stack Size:** `PTHREAD_STACK_MIN` (typically 16 KB on Linux).
- **Detach State Clause:**
  - `PTHREAD_CREATE_JOINABLE`: Default. Preserves exit status. Must be joined with `pthread_join()` to avoid resource leakage.
  - `PTHREAD_CREATE_DETACHED`: Self-reclaiming. Cannot be joined.
- **Scheduling Inheritance Rule:**
  Must call `pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED)` or all `setschedpolicy` and `setschedparam` calls are ignored!
- **Policies:**
  - `SCHED_OTHER`: Fair-share CFS, priority fixed at 0.
  - `SCHED_FIFO`: Real-time, priority 1–99, no time quantum, runs until block/yield.
  - `SCHED_RR`: Real-time, priority 1–99, fixed round-robin time slice among equal priority peers.
- **Required Capability:** Real-time policies (`SCHED_FIFO`, `SCHED_RR`) require administrative privileges (`CAP_SYS_NICE` or `root`), returning `EPERM` otherwise.

---

## 13. What You Should Be Able to Do

1. **Write complete C code** to initialize, configure, and destroy a `pthread_attr_t` object.
2. **Configure custom stack sizes** and calculate the reduction in virtual memory usage for high-density thread architectures.
3. **Set a thread to detached mode** at creation time and explain why detached threads cannot be joined.
4. **Implement real-time scheduling** using `SCHED_FIFO` and `SCHED_RR` with explicit priority settings (`sched_param`).
5. **Debug scheduling bugs** involving missing `PTHREAD_EXPLICIT_SCHED` directives.
6. **Formulate error-handling code** checking for `EPERM`, `EINVAL`, and `ENOMEM` during thread attribute operations.
