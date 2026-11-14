# Module 13: Hybrid MPI + OpenMP Programming (Lecture 32)
# Subtopic 13.03: MPI Thread Safety Levels (`MPI_Init_thread`, `SINGLE`, `FUNNELED`, `SERIALIZED`, `MULTIPLE`)

---

## 1. Basic Idea

When combining MPI with multi-threaded runtimes (such as OpenMP or POSIX Threads), multiple execution threads exist simultaneously within the memory space of each MPI process. 

This introduces a critical concurrency question:
**Is the underlying MPI library thread-safe? Can multiple threads invoke MPI communication routines simultaneously without corrupting internal library data structures?**

To establish a strict, standardized contract between the application and the MPI runtime, the MPI standard defines **Four Thread Safety Levels**, arranged in strictly ascending order of concurrency capability:

```
Level 0: MPI_THREAD_SINGLE       (Pure Single-Threaded: Exactly 1 thread exists)
   |
   v
Level 1: MPI_THREAD_FUNNELED     (Multi-threaded: ONLY Master Thread makes MPI calls)
   |
   v
Level 2: MPI_THREAD_SERIALIZED   (Multi-threaded: Any thread can call MPI, but ONE at a time)
   |
   v
Level 3: MPI_THREAD_MULTIPLE     (Fully Concurrent: Multiple threads call MPI simultaneously)
```

An application negotiates thread safety at startup via **`MPI_Init_thread()`**:
- The application specifies the **`required`** level of thread support it needs.
- The MPI library returns the **`provided`** level of support it can physically guarantee on that hardware and library build.

---

## 2. Why We Need It

### 1. The Internal Race Condition Hazard inside MPI Libraries
Inside an MPI library implementation, the runtime manages complex global data structures:
- Network socket buffers and InfiniBand Queue Pairs (QPs).
- Posted Receive Queues (PRQ) and Unexpected Message Queues (UMQ).
- Request handle tracking tables and communicator lookup maps.

If two OpenMP threads simultaneously execute `MPI_Send` on the same network interface without internal synchronization, they race on hardware queue pointers, corrupting network packets and crashing the process with memory errors.

### 2. The Performance Penalty of Thread-Safe Locking
Why doesn't every MPI implementation simply enable `MPI_THREAD_MULTIPLE` by default?
- Enforcing full thread safety requires wrapping every internal MPI function, queue access, and packet transfer in **internal mutexes or atomic locks**.
- These internal locks impose significant latency overhead on every point-to-point and collective call, slowing down communication even when only one thread is active.
- By defining four distinct levels, MPI allows applications that communicate only from the master thread (`MPI_THREAD_FUNNELED`) to bypass all internal locking, achieving **maximum single-threaded network throughput**.

### 3. Graceful Fallback and Portability
Not all supercomputers or specialized network architectures support `MPI_THREAD_MULTIPLE` efficiently. `MPI_Init_thread()` allows portable software to check if the requested level was granted (`provided >= required`), enabling the application to cleanly abort or fall back to a simpler funneled communication model if thread support is missing.

---

## 3. How It Works

### The Four Standardized Thread Safety Levels

```
+-----------------------------------------------------------------------------------------+
|                              The Four Thread Safety Levels                              |
+-----------------------------------------------------------------------------------------+
| LEVEL 0: MPI_THREAD_SINGLE                                                              |
|   - Exactly ONE thread exists in the process.                                           |
|   - OpenMP parallel regions are completely forbidden (or must use num_threads(1)).      |
|   - Equivalent to standard MPI_Init().                                                  |
|                                                                                         |
| LEVEL 1: MPI_THREAD_FUNNELED (Standard for 90% of Hybrid HPC Applications)              |
|   - Process may spawn arbitrary OpenMP threads.                                         |
|   - BUT: ONLY the Master Thread (thread 0) is legally allowed to invoke MPI routines!  |
|   - Worker threads must never call MPI functions.                                       |
|                                                                                         |
| LEVEL 2: MPI_THREAD_SERIALIZED                                                          |
|   - Process may spawn arbitrary OpenMP threads.                                         |
|   - Any thread can make MPI calls, BUT only ONE thread at a time!                       |
|   - The application must manually serialize calls using #pragma omp critical or mutex.  |
|                                                                                         |
| LEVEL 3: MPI_THREAD_MULTIPLE (Highest Concurrency, Highest Overhead)                    |
|   - Completely thread-safe.                                                             |
|   - Any thread may invoke any MPI routine at any time concurrently without locks!       |
|   - MPI library handles all internal locking and thread-safe queue management.          |
+-----------------------------------------------------------------------------------------+
```

---

### Code Execution Patterns for Each Level

#### Pattern 1: `MPI_THREAD_FUNNELED` (Funneled to Master)
```c
#pragma omp parallel
{
    compute_local_work();

    /* ONLY Master Thread invokes MPI */
    #pragma omp master
    {
        MPI_Send(halo_buffer, count, MPI_DOUBLE, neighbor, 0, MPI_COMM_WORLD);
    }

    #pragma omp barrier // Ensure master finishes before workers proceed
}
```

#### Pattern 2: `MPI_THREAD_SERIALIZED` (Manual Serialization)
```c
#pragma omp parallel
{
    compute_slice();

    /* Any thread can communicate, but serialized by a critical section */
    #pragma omp critical
    {
        MPI_Send(my_slice, count, MPI_DOUBLE, neighbor, tid, MPI_COMM_WORLD);
    }
}
```

#### Pattern 3: `MPI_THREAD_MULTIPLE` (Concurrent Multi-Threaded)
```c
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    /* Concurrent MPI calls from all threads simultaneously without locks! */
    MPI_Send(thread_data[tid], count, MPI_DOUBLE, neighbor, tid /*unique tag*/, MPI_COMM_WORLD);
}
```

---

## 4. Important Mechanisms and Functions

### 1. Sizing and Negotiation via `MPI_Init_thread`

```c
#include <mpi.h>

int MPI_Init_thread(int *argc, char ***argv, int required, int *provided);
```

- **`required`:** An integer constant representing the requested level:
  - `MPI_THREAD_SINGLE` ($0$)
  - `MPI_THREAD_FUNNELED` ($1$)
  - `MPI_THREAD_SERIALIZED` ($2$)
  - `MPI_THREAD_MULTIPLE` ($3$)
- **`provided`:** An output integer populated by MPI with the maximum level supported by the installed library.
- **Calling Rule:** Replaces `MPI_Init()`. Must be called exactly once before any other MPI call.

---

### 2. Querying the Active Thread Level: `MPI_Query_thread`

To verify the active thread level anywhere in an application or third-party library:

```c
int MPI_Query_thread(int *provided);
```

---

### 3. Verifying the Master Thread: `MPI_Is_thread_main`

Under `MPI_THREAD_FUNNELED`, only the master thread may communicate. The runtime provides a query function to verify whether the calling thread is indeed the primary thread that called `MPI_Init_thread`:

```c
int MPI_Is_thread_main(int *flag);
```
- Sets `*flag = 1` if the calling thread is the main thread; sets `*flag = 0` otherwise.

---

## 5. Compilable C Code Examples

### Example 1: Thread Safety Level Discovery and Negotiation Protocol
This complete program requests `MPI_THREAD_MULTIPLE`, queries the level actually provided by the system, inspects whether degradation occurred, and demonstrates the safe fallback handling pattern required in robust production codes.

```c
/*
 * File: mpi_thread_safety_check.c
 * Compilation:
 *   mpicc -O2 -fopenmp mpi_thread_safety_check.c -o mpi_thread_safety_check
 *
 * Execution:
 *   export OMP_NUM_THREADS=4
 *   mpirun -np 2 ./mpi_thread_safety_check
 */

#include <mpi.h>
#include <omp.h>
#include <stdio.h>
#include <stdlib.h>

const char* thread_level_to_string(int level) {
    switch (level) {
        case MPI_THREAD_SINGLE:     return "MPI_THREAD_SINGLE (No multi-threading)";
        case MPI_THREAD_FUNNELED:   return "MPI_THREAD_FUNNELED (Master thread only)";
        case MPI_THREAD_SERIALIZED: return "MPI_THREAD_SERIALIZED (One thread at a time)";
        case MPI_THREAD_MULTIPLE:   return "MPI_THREAD_MULTIPLE (Fully concurrent)";
        default:                    return "UNKNOWN LEVEL";
    }
}

int main(int argc, char** argv) {
    int rank, size;
    int required = MPI_THREAD_MULTIPLE;
    int provided;

    /* Step 1: Request MPI_THREAD_MULTIPLE */
    int rc = MPI_Init_thread(&argc, &argv, required, &provided);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "Fatal: MPI_Init_thread failed!\n");
        exit(EXIT_FAILURE);
    }

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  MPI THREAD SAFETY NEGOTIATION & DISCOVERY REPORT               \n");
        printf("=================================================================\n");
        printf("  Requested Level : %d -> %s\n", required, thread_level_to_string(required));
        printf("  Provided Level  : %d -> %s\n", provided, thread_level_to_string(provided));
        printf("-----------------------------------------------------------------\n");

        if (provided < required) {
            printf("  [NOTICE] Thread level degraded by MPI implementation!\n");
            printf("  Application requested MULTIPLE, but library granted: %s\n",
                   thread_level_to_string(provided));
            if (provided < MPI_THREAD_FUNNELED) {
                fprintf(stderr, "  Fatal: Application requires at least MPI_THREAD_FUNNELED!\n");
                MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
            }
            printf("  [FALLBACK] Adapting algorithm to use MPI_THREAD_FUNNELED...\n");
        } else {
            printf("  [SUCCESS] Full MPI_THREAD_MULTIPLE capability granted!\n");
        }
        printf("=================================================================\n\n");
    }

    /* Synchronize before multi-threaded test */
    MPI_Barrier(MPI_COMM_WORLD);

    /* Step 2: Test Master Thread Identification */
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        int is_main = 0;
        MPI_Is_thread_main(&is_main);

        #pragma omp critical
        {
            printf("[Rank %d | Thread %d] MPI_Is_thread_main() returned: %s\n",
                   rank, tid, is_main ? "TRUE (I am the Master)" : "FALSE (Worker)");
        }
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running on modern Open MPI / MPICH build) ---
=================================================================
  MPI THREAD SAFETY NEGOTIATION & DISCOVERY REPORT               
=================================================================
  Requested Level : 3 -> MPI_THREAD_MULTIPLE (Fully concurrent)
  Provided Level  : 3 -> MPI_THREAD_MULTIPLE (Fully concurrent)
-----------------------------------------------------------------
  [SUCCESS] Full MPI_THREAD_MULTIPLE capability granted!
=================================================================

[Rank 0 | Thread 0] MPI_Is_thread_main() returned: TRUE (I am the Master)
[Rank 0 | Thread 1] MPI_Is_thread_main() returned: FALSE (Worker)
[Rank 0 | Thread 2] MPI_Is_thread_main() returned: FALSE (Worker)
[Rank 0 | Thread 3] MPI_Is_thread_main() returned: FALSE (Worker)
[Rank 1 | Thread 0] MPI_Is_thread_main() returned: TRUE (I am the Master)
[Rank 1 | Thread 1] MPI_Is_thread_main() returned: FALSE (Worker)
[Rank 1 | Thread 2] MPI_Is_thread_main() returned: FALSE (Worker)
[Rank 1 | Thread 3] MPI_Is_thread_main() returned: FALSE (Worker)
```

---

## 6. Mathematical Ordering and Degradation Invariants

### The Monotonic Inequality of Thread Safety Levels

The four thread safety constants are defined by the MPI standard such that their integer values form a strictly monotonically increasing sequence:

$$\text{MPI\_THREAD\_SINGLE} < \text{MPI\_THREAD\_FUNNELED} < \text{MPI\_THREAD\_SERIALIZED} < \text{MPI\_THREAD\_MULTIPLE}$$

Mathematically:
$$0 < 1 < 2 < 3$$

#### Sizing and Verification Invariant:
A parallel hybrid algorithm requiring capability level $L_{\text{req}}$ can execute safely if and only if the level provided by the MPI library $L_{\text{prov}}$ satisfies:

$$L_{\text{prov}} \ge L_{\text{req}}$$

#### The Degradation Condition:
If:
$$L_{\text{prov}} < L_{\text{req}}$$
The MPI implementation does **not** support the requested level of concurrency. The application **must not** attempt to invoke multi-threaded MPI operations beyond $L_{\text{prov}}$, or undefined memory corruption and crashes will occur.

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Comparison Matrix of Thread Levels

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Thread Safety Level       | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_THREAD_SINGLE`       | - Absolute highest network throughput   | - Multi-threaded execution completely   |
|                           | - Zero internal locking overhead        |   prohibited                            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_THREAD_FUNNELED`     | - Zero internal locking in MPI runtime  | - Worker threads sit idle during master |
| (HPC Workhorse)           | - Highly portable across all clusters   |   network communication calls           |
|                           | - Clean separation of concerns          |                                         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_THREAD_SERIALIZED`   | - Allows workers to send their own data | - High developer burden: must protect   |
|                           |   without buffering on master           |   all calls with explicit locks         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_THREAD_MULTIPLE`     | - Ultimate asynchronous flexibility     | - Heavy internal locking penalty        |
|                           | - Overlaps independent thread transfers | - Slower single-thread latency          |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **`MPI_Init_thread`:** An advanced initialization function that negotiates multi-threaded capability between the application and the MPI library, accepting a requested safety level and returning the actual provided level.
>
> **`MPI_THREAD_SINGLE`:** The base thread safety level signifying that the application executes as a strictly single-threaded process with zero concurrent threads.
>
> **`MPI_THREAD_FUNNELED`:** A thread safety level permitting multi-threaded execution, with the strict operational constraint that **only the primary master thread** that initialized MPI is permitted to make MPI communication calls.
>
> **`MPI_THREAD_SERIALIZED`:** A thread safety level permitting any thread in a process to invoke MPI calls, provided that the application manually ensures that no two threads invoke MPI routines simultaneously.
>
> **`MPI_THREAD_MULTIPLE`:** The highest thread safety level where the MPI library is fully reentrant and thread-safe, allowing multiple concurrent threads to invoke MPI calls simultaneously without application-level locking.
>
> **`MPI_Is_thread_main`:** A query function that returns a boolean flag indicating whether the calling thread is the primary thread that initialized the MPI environment.

---

## 9. Comparison Tables

### Summary of What is Permitted at Each Level

| Operation / Capability | `SINGLE` | `FUNNELED` | `SERIALIZED` | `MULTIPLE` |
| :--- | :---: | :---: | :---: | :---: |
| Multiple OpenMP Threads Allowed? | **No** | **Yes** | **Yes** | **Yes** |
| Master Thread Calls MPI? | **Yes** | **Yes** | **Yes** | **Yes** |
| Worker Threads Call MPI? | **No** | **No** | **Yes** (Sequentially) | **Yes** (Concurrently) |
| Manual Locking Required by Dev? | None | None | **Yes** (`#pragma omp critical`)| None (MPI handles locks) |
| Internal Library Locking Overhead? | Zero | Zero | Minimal | **High** |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Diagnosing the `FUNNELED` Race Violation
**Question:** An application initializes MPI with `MPI_THREAD_FUNNELED`. The developer writes:

```c
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    /* Worker threads send their own slice */
    MPI_Send(local_data, 100, MPI_DOUBLE, 1, tid, MPI_COMM_WORLD);
}
```
Explain what violates the MPI standard and what runtime failure will occur.

**Solution:**
- **The Violation:** Under `MPI_THREAD_FUNNELED`, the application entered into a binding contract with the MPI library promising that **only the master thread** would ever invoke MPI routines.
- In this code, all OpenMP worker threads (`tid = 1, 2, ...`) concurrently invoke `MPI_Send()`.
- Because the library was initialized with `FUNNELED`, internal locking is disabled to maximize speed.
- The concurrent worker threads simultaneously access shared network device descriptors and message queues, triggering race conditions.
- **Result:** Undefined behavior—typically packet corruption, socket disconnection, or an immediate `SIGSEGV` crash inside the MPI transport layer (`libmpi.so`).
- **The Fix:** Either enclose the send inside `#pragma omp master`, or request `MPI_THREAD_MULTIPLE` during initialization.

---

### Problem 2: Handling Thread Level Degradation
**Question:** Write a robust initialization sequence in C that requests `MPI_THREAD_MULTIPLE`, checks if it was granted, and if only `MPI_THREAD_FUNNELED` was provided, prints a warning and sets a global flag `use_funneled_fallback = 1`. If neither was granted, abort the job.

**Solution:**

```c
int required = MPI_THREAD_MULTIPLE;
int provided;
int use_funneled_fallback = 0;

MPI_Init_thread(&argc, &argv, required, &provided);

if (provided >= MPI_THREAD_MULTIPLE) {
    /* Full concurrent support granted */
    use_funneled_fallback = 0;
} else if (provided >= MPI_THREAD_FUNNELED) {
    /* Degraded to funneled mode */
    use_funneled_fallback = 1;
    printf("[WARNING] MPI_THREAD_MULTIPLE not supported. Falling back to FUNNELED mode.\n");
} else {
    /* Insufficient thread support */
    fprintf(stderr, "[FATAL] MPI library does not support multi-threading!\n");
    MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
}
```

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why is `MPI_THREAD_FUNNELED` the most popular level in High-Performance Computing?
**Answer:** Because it combines the best of both worlds:
1. It allows full multi-threaded OpenMP parallelism across all multi-core CPU sockets for compute loops.
2. Because only the master thread communicates over MPI, the MPI library does not need internal mutexes, avoiding the heavy locking latency penalties of `MPI_THREAD_MULTIPLE`.
3. It maps naturally to standard halo-exchange patterns where data is gathered to the master thread before inter-node transmission.

### Q2: What is the difference between `MPI_THREAD_SERIALIZED` and `MPI_THREAD_MULTIPLE`?
**Answer:** In `MPI_THREAD_SERIALIZED`, multiple threads are legally allowed to invoke MPI functions, but the **programmer must ensure** that no two threads call MPI simultaneously (e.g., by wrapping calls inside `#pragma omp critical` or a mutex). In `MPI_THREAD_MULTIPLE`, the **MPI library itself** provides internal thread safety, allowing multiple threads to call MPI routines concurrently without any application-level synchronization.

### Q3: Can different processes in the same `MPI_COMM_WORLD` negotiate different thread levels?
**Answer:** Yes. The MPI standard specifies that `MPI_Init_thread` is an operation local to each process. For example, Rank 0 could run with `MPI_THREAD_MULTIPLE` (acting as a multi-threaded asynchronous network router), while worker Ranks $1 \dots P-1$ run with `MPI_THREAD_FUNNELED`. However, processes must adhere to their own locally negotiated contracts.

---

## 12. Quick Revision Sheet

- **Four Levels:** `SINGLE` ($0$) $\to$ `FUNNELED` ($1$) $\to$ `SERIALIZED` ($2$) $\to$ `MULTIPLE` ($3$).
- **`SINGLE`:** Exactly 1 thread exists in process.
- **`FUNNELED`:** OpenMP threads exist, but **only Master Thread** calls MPI. (HPC default).
- **`SERIALIZED`:** Multiple threads can call MPI, but **only one at a time** (manual locks required).
- **`MULTIPLE`:** Fully thread-safe. Multiple threads can call MPI concurrently.
- **Negotiation:** `MPI_Init_thread(&argc, &argv, required, &provided)`. Always check `provided >= required`!
- **Master Query:** `MPI_Is_thread_main(&flag)` returns 1 on master thread.

---

## 13. What You Should Be Able to Do

1. **Initialize hybrid applications safely** using `MPI_Init_thread()` and handle level degradation.
2. **Select the optimal thread safety level** for any hybrid software architecture.
3. **Verify master thread identity** in funneled applications using `MPI_Is_thread_main()`.
4. **Identify and correct race conditions** caused by calling MPI from worker threads under `MPI_THREAD_FUNNELED`.
5. **Contrast the performance trade-offs** between `MPI_THREAD_FUNNELED` and `MPI_THREAD_MULTIPLE`.
