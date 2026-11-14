# Module 04.01 — OpenMP Architecture, The Shared-Memory Model & The Fork-Join Paradigm

> **Prerequisites:** Module 02.05 (Flynn's Taxonomy & UMA/ccNUMA Shared Memory), Module 03.01 (Speedup & Amdahl's Law)  
> **Lecture Mapping:** Lectures 6–7 (Basics of OpenMP: Architecture, Shared Memory Abstraction, Fork-Join Model)  
> **Exam Relevance:** MidSem (MANDATORY: Architecture Tiers, Fork-Join Diagrams, Core Definitions) & 40% Lab Exam Core  

---

## 1. Understanding the Basic Idea

Suppose you run a busy restaurant kitchen. You are the **Head Chef**.
- When the kitchen is slow, you work alone: prepping vegetables, cooking sauces, and plating dishes (**Sequential Execution**).
- During the dinner rush, 50 orders arrive simultaneously. You cannot keep up alone.
- You blow a whistle and summon a team of **three Assistant Chefs**.
- You assign specific tasks: Assistant 1 chops onions, Assistant 2 grills steaks, Assistant 3 plates appetizers (**Parallel Execution**).
- Notice something crucial: all four of you are standing inside the **exact same kitchen**, sharing the **same spice rack**, the **same refrigerator**, and the **same cutting boards** (**Shared-Memory Abstraction**).
- Once all 50 orders are plated and served, the dinner rush ends. The assistant chefs clean up and step aside. You return to working alone (**The Fork-Join Model**).

```
                            THE KITCHEN ANALOGY OF FORK-JOIN
                        
                             Master Thread (Head Chef)
                                        │
                                  [ Serial Code ]
                                        │
                                  FORK (Whistle!)
                         ┌──────────────┼──────────────┐
                         ▼              ▼              ▼
                      Thread 0       Thread 1       Thread 2 ... (Worker Team)
                      (Master)       (Assistant)    (Assistant)
                         │              │              │
                     [Steaks]       [Chop Salad]    [Plating]    (Parallel Region)
                         │              │              │
                         └──────────────┼──────────────┘
                                        ▼
                                   JOIN (Barrier)
                                        │
                                  [ Serial Code ]
                             Master Thread (Head Chef)
```

This is the exact programming model of **OpenMP (Open Multi-Processing)**:
- You do not write low-level assembly or complex operating system thread management code.
- You simply tell the compiler: *"When you reach this section of code, fork a team of threads, execute concurrently within our shared memory space, and join back together when finished."*

---

## 2. What Exactly is OpenMP? (And What It Is NOT)

One of the most frequent examination traps is misunderstanding what OpenMP actually is.

### Precise University Examination Definition:
> **OpenMP (Open Multi-Processing)** is an industry-standard Application Programming Interface (API) for **shared-memory parallel programming** in C, C++, and Fortran that provides portable, scalable multi-threaded execution across multi-core systems through compiler directives, runtime library routines, and environment variables.

### What OpenMP is NOT:
1. ❌ **OpenMP is NOT a new programming language.** It does not have its own compiler from scratch. It is an extension layered onto standard C, C++, and Fortran.
2. ❌ **OpenMP is NOT for distributed clusters.** OpenMP requires a **shared physical memory address space**. A pure OpenMP program cannot run across multiple independent servers connected via network cables (that requires MPI, covered in Part B).
3. ❌ **OpenMP does NOT guarantee automatic parallelization.** The compiler will not magically parallelize your sequential algorithms on its own. You, the human engineer, must explicitly tell the compiler *where* and *how* to parallelize.

---

## 3. The Three-Tier Architecture of OpenMP

OpenMP is structured as a three-component cooperative software architecture:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     THE THREE-TIER OPENMP ARCHITECTURE                          │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. COMPILER DIRECTIVES (#pragma omp ...)
    High-level pragma statements placed directly in C/C++ source code instructing
    the compiler to generate multi-threaded code, distribute loop iterations,
    and insert synchronization barriers.
    Examples: `#pragma omp parallel`, `#pragma omp for`, `#pragma omp critical`
                                   │
                                   ▼
 2. RUNTIME LIBRARY ROUTINES (omp.h)
    Dynamic API functions called at runtime to query the thread environment,
    adjust team sizing, inspect timers, and manipulate low-level locks.
    Examples: `omp_get_thread_num()`, `omp_get_num_threads()`, `omp_get_wtime()`
                                   │
                                   ▼
 3. ENVIRONMENT VARIABLES (OMP_*)
    Operating system shell variables that control runtime execution parameters
    WITHOUT requiring code recompilation.
    Examples: `OMP_NUM_THREADS=8`, `OMP_SCHEDULE="dynamic,16"`, `OMP_PROC_BIND=true`
```

---

### Detailed Analysis of the Three Tiers

#### Tier 1: Compiler Pragmas (`#pragma omp`)
In the C and C++ standards, the `#pragma` mechanism provides implementation-specific information to the compiler.
- If a compiler supports OpenMP and the OpenMP flag is enabled (`gcc -fopenmp`), the compiler recognizes the `#pragma omp` directives, transforms the code into multi-threaded functions, and links the runtime library.
- **Graceful Degradation (Portability):** If you compile the exact same source code on an ancient or embedded compiler that does not support OpenMP (or if you forget `-fopenmp`), the compiler simply **ignores the pragmas as unrecognized warnings** and produces a valid sequential binary!

#### Tier 2: Runtime API (`<omp.h>`)
Header file defining prototypes for approximately 40 runtime library functions:
- Execution environment queries (`omp_get_thread_num()`, `omp_get_num_threads()`, `omp_in_parallel()`).
- Resource configuration (`omp_set_num_threads()`, `omp_set_dynamic()`).
- Lock management (`omp_init_lock()`, `omp_set_lock()`, `omp_unset_lock()`).
- Portable high-resolution wall-clock timing (`omp_get_wtime()`, `omp_get_wtick()`).

#### Tier 3: Environment Variables
Shell environment variables evaluated when the program is launched by the OS loader:
```bash
# Set default thread team to 16 threads
export OMP_NUM_THREADS=16

# Configure default loop scheduling to guided with chunk size 32
export OMP_SCHEDULE="guided,32"

# Bind threads to physical cores to prevent OS migration across NUMA nodes
export OMP_PROC_BIND=close
```

---

## 4. The Shared-Memory Programming Abstraction

In an OpenMP program, all threads belong to a **single operating system process**. 
Consequently, they share the **same virtual address space**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                   OPENMP SHARED-MEMORY PROCESS ADDRESS SPACE                    │
└─────────────────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ COMMON SHARED ADDRESS SPACE (Global Scope)                                 │
 │                                                                             │
 │  ├── TEXT SEGMENT: Executable Machine Code Instructions                     │
 │  ├── DATA / BSS SEGMENT: Global variables, static variables                 │
 │  └── HEAP: Dynamically allocated memory via malloc() / calloc()             │
 │                                                                             │
 │      int shared_counter = 0;                                                │
 │      double *large_array = (double *)malloc(N * sizeof(double));            │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
 ┌───────────────┐               ┌───────────────┐               ┌───────────────┐
 │ Thread 0      │               │ Thread 1      │               │ Thread 2      │
 │ (Master)      │               │ (Worker)      │               │ (Worker)      │
 │               │               │               │               │               │
 │ PRIVATE STACK │               │ PRIVATE STACK │               │ PRIVATE STACK │
 │ ├── Local vars│               │ ├── Local vars│               │ ├── Local vars│
 │ ├── Loop 'i'  │               │ ├── Loop 'i'  │               │ ├── Loop 'i'  │
 │ └── Stack frame│              │ └── Stack frame│              │ └── Stack frame│
 └───────────────┘               └───────────────┘               └───────────────┘
```

### The Two Data Scopes:
1. **Shared Memory (Heap / Data Segment):**
   - By default, variables declared outside a parallel region reside in shared memory.
   - Any thread can read or write to these variables at any time.
   - Communication between threads occurs effortlessly through shared variables—no network packets!
   - **The Danger:** Unsynchronized concurrent writes cause **Race Conditions** (covered in Module 06).
2. **Private Memory (Individual Thread Stacks):**
   - Each thread possesses its own private, isolated stack allocated by the OS runtime.
   - Variables declared *inside* a parallel region are private to that specific thread.
   - Thread 0 cannot read or corrupt Thread 1's private stack variables.

---

## 5. The Fork-Join Execution Model in Deep Detail

The core paradigm governing all OpenMP execution is the **Fork-Join Model**.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        THE FORK-JOIN TIMELINE OF EXECUTION                      │
└─────────────────────────────────────────────────────────────────────────────────┘

 Master Thread
       │
       ▼
 [ SERIAL CODE ] ──► Executes on a single core (Thread 0)
       │
   ─── ▼ ───────────────────────────────────────────────────────── FORK POINT
       │ Spawns / wakes up worker threads
       ├─────────────────┬─────────────────┬─────────────────┐
       ▼                 ▼                 ▼                 ▼
   Thread 0          Thread 1          Thread 2          Thread 3
   (Master)          (Worker)          (Worker)          (Worker)
       │                 │                 │                 │
 [ PARALLEL WORK ] [ PARALLEL WORK ] [ PARALLEL WORK ] [ PARALLEL WORK ]
       │                 │                 │                 │
   ─── ▼ ────────────────┴─────────────────┴─────────────────┴──── IMPLICIT BARRIER
       │ All threads synchronize; workers wait or hibernate
       ▼
 [ SERIAL CODE ] ──► Resumes single-threaded execution on Master (Thread 0)
       │
   ─── ▼ ───────────────────────────────────────────────────────── FORK POINT (Loop 2)
       │ Spawns / wakes up worker threads again!
       ├─────────────────┬─────────────────┬─────────────────┐
       ▼                 ▼                 ▼                 ▼
   Thread 0          Thread 1          Thread 2          Thread 3
       │                 │                 │                 │
 [ PARALLEL WORK ] [ PARALLEL WORK ] [ PARALLEL WORK ] [ PARALLEL WORK ]
       │                 │                 │                 │
   ─── ▼ ────────────────┴─────────────────┴─────────────────┴──── JOIN POINT
       ▼
 [ PROGRAM TERMINATES ]
```

---

### Step-by-Step Mechanics of Fork-Join

#### 1. The Initial Thread
When an OpenMP program begins execution, the operating system creates a standard single process with exactly **one thread**. This initial thread is designated as the **Master Thread (Thread 0)**.

#### 2. The Fork
When the master thread encounters a `#pragma omp parallel` directive:
- It creates a **team of threads** consisting of the master thread and zero or more worker threads.
- In modern high-performance runtimes (such as GNU `libgomp` or LLVM `libomp`), worker threads are not constantly created and destroyed from scratch (which would incur massive OS syscall overhead). Instead, a **Thread Pool** is initialized at startup. The fork operation simply wakes sleeping worker threads from the pool!

#### 3. Concurrent Execution (The Structured Block)
All threads in the team execute the structured block of code enclosed by the parallel region concurrently. 
- Each thread is assigned a unique integer identifier called its **Thread Number (Thread ID)**:
  \[
  \text{Thread ID} \in \{0, 1, 2, \dots, p - 1\}
  \]
  where $p$ is the total number of threads in the active team.
- The master thread's ID is **always 0**.

#### 4. The Join (Implicit Barrier)
At the end of the structured block enclosed by the parallel construct:
- There is an **implicit synchronization barrier**.
- Faster threads that finish their portion of the parallel region must halt and wait at the barrier.
- No thread can proceed to the sequential code until **every single thread in the team has arrived at the barrier**.
- Once all threads arrive, the worker threads return to sleep in the thread pool, and the master thread alone resumes sequential execution.

---

## 6. Practical Implementation: The First OpenMP Program in C

Here is a complete, compilable C program demonstrating the fork-join model, thread ID querying, and execution non-determinism.

### OpenMP Hello World (`omp_hello.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <omp.h> // Essential header for OpenMP API prototypes

int main(void) {
    printf("============================================================\n");
    printf("         OPENMP FORK-JOIN DEMONSTRATION PROGRAM             \n");
    printf("============================================================\n");

    // 1. Serial Execution (Master Thread Only)
    printf("1. SERIAL REGION: Master thread starting execution.\n");
    printf("   Initial active threads = %d\n", omp_get_num_threads());

    int requested_threads = 4;
    printf("\n2. FORKING: Spawning a team of %d threads...\n", requested_threads);

    // 2. Parallel Region: Fork Point
    #pragma omp parallel num_threads(requested_threads)
    {
        // Executed CONCURRENTLY by all 4 threads!
        int tid = omp_get_thread_num();       // Query my unique thread ID
        int total = omp_get_num_threads();     // Query total team size

        printf("   [Thread %d of %d] Hello from the parallel region! (Core: %d)\n", 
               tid, total, omp_get_thread_num());
    } // 3. Implicit Barrier & Join Point!

    // 4. Serial Execution Resumed
    printf("\n3. JOIN COMPLETED: Back to serial execution on Master Thread.\n");
    printf("   Active threads now = %d\n", omp_get_num_threads());
    printf("============================================================\n");

    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O2 -fopenmp omp_hello.c -o omp_hello
./omp_hello
```

*Possible Terminal Output (Run 1):*
```
============================================================
         OPENMP FORK-JOIN DEMONSTRATION PROGRAM             
============================================================
1. SERIAL REGION: Master thread starting execution.
   Initial active threads = 1

2. FORKING: Spawning a team of 4 threads...
   [Thread 0 of 4] Hello from the parallel region! (Core: 0)
   [Thread 2 of 4] Hello from the parallel region! (Core: 2)
   [Thread 1 of 4] Hello from the parallel region! (Core: 1)
   [Thread 3 of 4] Hello from the parallel region! (Core: 3)

3. JOIN COMPLETED: Back to serial execution on Master Thread.
   Active threads now = 1
============================================================
```

*Possible Terminal Output (Run 2):*
```
...
2. FORKING: Spawning a team of 4 threads...
   [Thread 3 of 4] Hello from the parallel region! (Core: 3)
   [Thread 1 of 4] Hello from the parallel region! (Core: 1)
   [Thread 0 of 4] Hello from the parallel region! (Core: 0)
   [Thread 2 of 4] Hello from the parallel region! (Core: 2)
...
```

#### The Vital Pedagogical Observation:
Notice that between Run 1 and Run 2, **the lines printed by the threads came out in completely different orders!**
- Run 1 printed: Thread 0, then 2, then 1, then 3.
- Run 2 printed: Thread 3, then 1, then 0, then 2.
- **Why?** Because parallel threads execute concurrently. The operating system thread scheduler decides which thread gets CPU cycles at any given microsecond.
- **Exam Rule:** In parallel programming, **concurrent execution does NOT imply ordered execution!** Never write code that assumes Thread 0 finishes before Thread 1.

---

## 7. Important Comparisons

### OpenMP vs. POSIX Threads (Pthreads)

| Feature | OpenMP (Directive-Based) | POSIX Threads (Pthreads - API-Based) |
|:---|:---|:---|
| **Paradigm** | High-level declarative compiler pragmas | Low-level procedural C library API |
| **Thread Management** | Automatic: handled by OpenMP runtime thread pool | Manual: explicit `pthread_create()` and `pthread_join()` |
| **Work Distribution** | Automatic: `#pragma omp for` slices loops | Manual: programmer must calculate index bounds per thread |
| **Portability** | High across C, C++, and Fortran | Unix/Linux standard (Windows requires emulation libraries) |
| **Lines of Code** | Minimal (typically 1 to 2 lines per loop) | High (requires dozens of lines of scaffolding per thread) |
| **Control Granularity**| Coarse to medium | Fine-grained (direct control over stack, priority, attributes)|
| **Compiler Support** | Requires OpenMP-enabled compiler (`-fopenmp`) | Works with any standard ANSI C compiler |

---

## 8. Exam-Ready Definitions

### OpenMP (Open Multi-Processing)
> **OpenMP** is an open-standard Application Programming Interface (API) for shared-memory parallel programming that provides compiler directives, runtime library routines, and environment variables to create and manage multi-threaded programs in C, C++, and Fortran.

### Fork-Join Model
> The **Fork-Join Model** is an execution paradigm in which an initial master thread executes sequential portions of a program and dynamically forks a team of parallel worker threads upon encountering a parallel construct, synchronizing and joining back into a single thread at the end of the parallel region.

### Master Thread
> The **Master Thread** is the initial thread of execution created when an OpenMP process launches, possessing Thread ID 0, responsible for executing serial regions and orchestrating thread teams during parallel regions.

### Implicit Barrier
> An **Implicit Barrier** is an automatic synchronization point inserted by the OpenMP compiler at the end of structured parallel and work-sharing blocks, halting arriving threads until every thread in the active team arrives before allowing any thread to proceed.

---

## 9. Solved Problems

### Problem 1: Tracing Thread Counts and State Queries
A programmer writes the following code:
```c
int a = omp_get_num_threads();
#pragma omp parallel num_threads(6)
{
    int b = omp_get_num_threads();
    int c = omp_get_thread_num();
}
int d = omp_get_num_threads();
```
1. What value is assigned to variable `a`?
2. What value is assigned to variable `b` inside the parallel region?
3. What range of values can variable `c` take?
4. What value is assigned to variable `d` after the parallel region?
5. Which thread executes the statement `int a = ...` and `int d = ...`?

#### Solution:
1. **Value of `a`:** `a = 1`. Outside of any parallel construct, only the master thread exists, so the active team size is exactly 1.
2. **Value of `b`:** `b = 6`. Inside the parallel region, the active thread team size requested by `num_threads(6)` is 6.
3. **Range of `c`:** `c` represents the thread ID, which takes integer values in the range:
   \[
   c \in \{0, 1, 2, 3, 4, 5\}
   \]
4. **Value of `d`:** `d = 1`. After the join barrier, the worker threads hibernate and only the master thread continues, resetting active threads to 1.
5. **Executing Thread:** The Master Thread (**Thread 0**).

---

### Problem 2: Analyzing Execution Overhead in Fork-Join Loops
A program has a loop that executes 1,000 times:
```c
for (int step = 0; step < 1000; step++) {
    #pragma omp parallel num_threads(8)
    {
        do_work(step);
    }
}
```
1. How many fork and join operations are executed during this program run?
2. Why is this code architecturally inefficient compared to placing the `#pragma omp parallel` directive *outside* the outer loop?
3. Rewrite the code using the optimized OpenMP pattern.

#### Solution:
1. **Number of Fork/Join Operations:**
   The `#pragma omp parallel` construct is entered and exited on **every single iteration** of the `step` loop:
   \[
   \text{Forks} = 1,000, \quad \text{Joins} = 1,000
   \]
2. **Architectural Inefficiency:**
   Entering and exiting a parallel region requires waking up threads, synchronizing thread state, and executing an implicit barrier. Repeating this 1,000 times introduces significant **parallelization overhead $c(N)$**.
3. **Optimized OpenMP Pattern:**
   Fork the thread team **once** outside the loop, and use work-sharing inside:
   ```c
   #pragma omp parallel num_threads(8)
   {
       for (int step = 0; step < 1000; step++) {
           do_work(step);
           #pragma omp barrier // Synchronize steps if required
       }
   }
   ```
   *Benefit:* Exactly **1 fork** and **1 join** operation total!

---

## 10. Conceptual Examination Questions & Answers

### Q1: What happens if an OpenMP program is compiled with a standard C compiler without the `-fopenmp` flag?
**Model Answer:**
If an OpenMP program is compiled without the `-fopenmp` compiler flag:
1. The compiler treats all `#pragma omp ...` directives as **unrecognized pragmas**, emits a harmless compiler warning, and ignores them completely.
2. The code within the structured blocks is compiled as standard, single-threaded **sequential C code**.
3. If the code calls OpenMP runtime library functions (such as `omp_get_thread_num()`) and includes `<omp.h>`, the compilation will fail during linking unless stub functions are provided.
4. This feature illustrates the **graceful degradation and portability** of OpenMP: parallel annotations do not break sequential semantics on non-OpenMP systems.

---

### Q2: Explain why OpenMP programs can only execute on shared-memory computer systems and cannot natively run across a distributed cluster.
**Model Answer:**
OpenMP’s architectural execution model relies on the assumption of a **unified, shared virtual address space**. 

In OpenMP:
- Threads communicate by reading and writing directly to shared memory addresses in RAM via standard pointer and load/store machine instructions.
- Synchronization primitives (such as barriers, critical sections, and atomic operations) rely on hardware cache-coherence protocols and atomic bus transactions (e.g., `test-and-set` or `compare-and-swap`).

In a distributed-memory cluster, distinct physical compute nodes have disjoint, private physical DRAMs with no shared physical addresses or hardware cache coherence across the network. A pointer valid on Node 0 is completely meaningless on Node 1. Because OpenMP possesses no native message-passing networking layer to serialize and transmit memory packets over Ethernet or InfiniBand, it cannot run across separate cluster nodes without a distributed-memory layer like MPI.

---

## 11. Quick Revision Sheet

- **OpenMP Definition:**
  \[
  \boxed{\text{OpenMP} = \text{Compiler Pragmas} + \text{Runtime Library } (omp.h) + \text{Environment Variables}}
  \]
- **Fork-Join Paradigm:**
  \[
  \boxed{\text{Master Thread 0 (Serial)} \xrightarrow{\text{Fork}} \text{Thread Team (Parallel)} \xrightarrow{\text{Join/Barrier}} \text{Master Thread 0 (Serial)}}
  \]
- **Key Query Functions:**
  - `omp_get_thread_num()` $\implies$ Returns my Thread ID ($0 \le \text{ID} < p$).
  - `omp_get_num_threads()` $\implies$ Returns total threads in the current team ($p$).
- **Core Memory Abstraction:**
  - **Shared Memory:** Heap, global, and static variables; visible to all threads.
  - **Private Memory:** Local variables on individual thread stacks; isolated to each thread.
- **Common Exam Trap:**
  - *Trap:* Thinking `omp_get_num_threads()` tells the CPU how many threads to create.
  - *Correction:* `omp_get_num_threads()` is a **query** (getter) that reports current team size. To specify thread count, use the clause `num_threads(N)` or the setter `omp_set_num_threads(N)`.

---

## 12. What You Should Be Able to Do After This Topic

Before proceeding to Module 04.02, verify that you can independently:
1. State the three architectural components of OpenMP and give examples of each.
2. Draw and explain the complete lifecycle diagram of the Fork-Join execution model.
3. Write, compile (`gcc -fopenmp`), and run a basic OpenMP multi-threaded program in C.
4. Explain why thread print outputs appear in non-deterministic order across runs.
5. Contrast OpenMP and Pthreads across abstraction level, lines of code, and thread management.
