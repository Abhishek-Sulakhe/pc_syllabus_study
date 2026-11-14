# 02. OpenMP Pragma Syntax, Parallel Regions, and Compilation

---

## 1. The Basic Idea

In standard C and C++, the compiler is strictly bound to execute sequential semantics: instructions are evaluated according to the abstract machine model defined by the language standards (ISO C / ISO C++). To introduce multithreading without breaking language standard compliance or requiring custom dialect compilers, OpenMP relies on **compiler directives** known as **pragmas** (`#pragma`).

A directive instructs the compiler to perform special preprocessing and code transformations if OpenMP support is activated. If OpenMP support is deactivated, standard-compliant compilers simply ignore unrecognized `#pragma` directives with a harmless warning (or none at all), leaving the underlying code to compile as an ordinary single-threaded program.

The foundational construct in OpenMP is the **parallel region**, declared with:
```c
#pragma omp parallel
{
    /* Structured Block executed redundantly by every thread in the team */
}
```

When execution encounters this directive, the runtime transforms the enclosed block of code into an outlined function (often called an *outlined worker function* or `.omp_fn`). The master thread activates a team of threads, passes the shared execution context, and each thread independently executes the instructions within the structured block.

```
       Sequential Flow (Master Thread Only)
                    |
                    v
          #pragma omp parallel
                   /|\
                  / | \
                 /  |  \   [Fork: Team Created / Awakened]
                v   v   v
            Thread Thread Thread ... Thread
              0      1      2         T-1
            |------ Structured Block ------|
            (Each thread executes full code)
                \   |   /
                 \  |  /   [Join: Implicit Barrier at Closing Brace]
                  \ | /
                    v
       Sequential Flow Resumes (Master Thread Only)
```

Understanding how to write correct pragma syntax, configure thread team counts, conditionally include runtime APIs, and invoke the proper compiler flags forms the prerequisite foundation for all shared-memory programming.

---

## 2. Why We Need It

Before OpenMP introduced pragma-based parallelism, parallel programming in C required raw POSIX Threads (`pthreads`) or platform-specific system APIs (such as Win32 Threads). Writing raw thread code required:
1. Manually allocating and managing thread handles (`pthread_t threads[N]`).
2. Packaging function arguments into dynamically allocated heap structs.
3. Defining explicit function pointers for thread entry routines (`void* (*start_routine)(void*)`).
4. Writing explicit loop logic to launch threads (`pthread_create`) and synchronize thread completion (`pthread_join`).
5. Completely restructuring the source code, obliterating readability and making sequential debugging nearly impossible.

### The OpenMP Pragma Advantage
OpenMP solves these pain points through directive-based abstraction:

```c
/* ======================== PTHREADS APPROACH ======================== */
/* 25+ lines of boilerplate, struct packing, error checking, joining */
typedef struct { int tid; int num_threads; } ThreadArg;
void* worker(void* arg) {
    ThreadArg* t = (ThreadArg*)arg;
    printf("Thread %d of %d\n", t->tid, t->num_threads);
    return NULL;
}
// In main(): pthread_t th[4]; ThreadArg a[4]; for(...) pthread_create(...); for(...) pthread_join(...);

/* ========================= OPENMP APPROACH ========================= */
/* 4 lines of clean, non-intrusive annotations */
#pragma omp parallel
{
    printf("Thread %d of %d\n", omp_get_thread_num(), omp_get_num_threads());
}
```

Furthermore, by encapsulating parallel directives in `#pragma omp`, OpenMP guarantees **incremental parallelism**: a developer can profile a large production codebase, identify the critical hotspots, and parallelize them one loop or region at a time without breaking the sequential baseline.

---

## 3. How It Works: Step-by-Step Architecture and Mechanics

### 3.1 The OpenMP Pragma Grammar
The OpenMP specification defines a strict grammar for directives in C and C++:

$$\text{\#pragma omp } \langle \text{directive-name} \rangle \quad [\langle \text{clause} \rangle [[\text{,}] \langle \text{clause} \rangle] \dots] \quad \backslash\text{n}$$

Key syntactical rules:
1. **Case Sensitivity:** All OpenMP directives and clause keywords are strictly lowercase (`#pragma omp parallel`, not `#pragma OMP PARALLEL`).
2. **Sentinel:** `#pragma omp` acts as the directive sentinel. Compilers recognize `omp` following `#pragma` as reserved for the OpenMP API.
3. **Line Termination:** Directives end with a newline character (`\n`). To span a directive across multiple physical lines, you must escape the newline using a backslash (`\`):
   ```c
   #pragma omp parallel default(none) \
                        shared(A, B, C) \
                        private(tid) \
                        num_threads(4)
   ```
4. **Structured Block Requirement:** A directive applies to the immediately following *structured block*. A structured block is a single executable statement or a compound statement enclosed in `{ ... }` that has **exactly one entry point at the top and exactly one exit point at the bottom**.
   - You cannot jump into a structured block using `goto`, `longjmp`, or `switch`.
   - You cannot jump out of a structured block using `goto`, `break`, `return`, or `longjmp` (except `exit()` which terminates the entire process).

```
   VALID STRUCTURED BLOCK:           INVALID (ILLEGAL) JUMP:
   #pragma omp parallel              #pragma omp parallel
   {                                 {
       step1();                          step1();
       step2();                          if (error) goto cleanup; /* COMPILER ERROR! */
       step3();                          step2();
   }                                 }
                                     cleanup: ...
```

---

### 3.2 The Master-Thread Outlining Mechanism
When the compiler encounters `#pragma omp parallel`, it performs an internal transformation called **function outlining**:

1. **Extraction:** The compiler cuts out the body of the structured block and generates an internal static function:
   ```c
   /* Conceptual internal compiler generation */
   static void .omp_outlined_fn(void *shared_context) {
       /* user structured block code */
   }
   ```
2. **Context Packaging:** Variables declared outside the parallel region that are shared among threads are bundled into an activation record (a struct of pointers).
3. **Runtime Invocation:** The compiler replaces the `#pragma omp parallel` block with a call to the OpenMP runtime library (e.g., `GOMP_parallel` in GCC/libgomp, or `__kmpc_fork_call` in Clang/LLVM/libomp):
   ```c
   /* Compiler-generated master code */
   __kmpc_fork_call(&loc, num_args, .omp_outlined_fn, &shared_context);
   ```
4. **Thread Dispatch & Barrier:** The runtime wakes up dormant threads in the runtime thread pool, assigns thread IDs ($0$ through $T-1$, where Thread 0 is the master thread), and instructs all $T$ threads to execute `.omp_outlined_fn`. At the closing brace `}`, an implicit synchronization barrier forces all threads to wait until the slowest thread finishes before the master thread resumes sequential execution.

```
+-------------------------------------------------------------------------------+
| User C Source:                                                                |
|                                                                               |
|   int x = 42;                                                                 |
|   #pragma omp parallel num_threads(2)                                         |
|   {                                                                           |
|       printf("T%d: x=%d\n", omp_get_thread_num(), x);                         |
|   }                                                                           |
+-------------------------------------------------------------------------------+
                                    |
                                    v (Compiler Outlining)
+-------------------------------------------------------------------------------+
| Outlined Function:                                                            |
|   void .omp_fn.0(struct .omp_data_s *data) {                                  |
|       int *x_ptr = data->x_ptr;                                               |
|       printf("T%d: x=%d\n", omp_get_thread_num(), *x_ptr);                    |
|   }                                                                           |
|                                                                               |
| Master Transformed Function:                                                  |
|   int x = 42;                                                                 |
|   struct .omp_data_s data = { &x };                                           |
|   GOMP_parallel(.omp_fn.0, &data, 2, 0); // fork 2 threads, run, implicit join|
+-------------------------------------------------------------------------------+
```

---

## 4. Important Mechanisms & Variants

### 4.1 Determining Thread Team Size: The Hierarchy of Precedence
How many threads will execute a parallel region? OpenMP defines a rigid, deterministic **4-level precedence hierarchy** to resolve the team size $T$:

```
HIGHEST PRECEDENCE:
[ Level 1 ] ---> num_threads(N) clause on the #pragma omp parallel directive
                     | (If absent, fall back to)
                     v
[ Level 2 ] ---> omp_set_num_threads(N) runtime API call
                     | (If not called, fall back to)
                     v
[ Level 3 ] ---> OMP_NUM_THREADS=N environment variable
                     | (If unset, fall back to)
                     v
[ Level 4 ] ---> System Default (typically number of online logical CPU cores)
LOWEST PRECEDENCE
```

1. **Clause Level (`num_threads(N)`):**
   Explicitly passed directly on the directive:
   ```c
   #pragma omp parallel num_threads(8)
   ```
   Binds strictly to this specific parallel region. It overrides all environment variables and preceding API calls.
2. **API Level (`omp_set_num_threads(N)`):**
   Called in sequential code before entering a parallel region:
   ```c
   omp_set_num_threads(4);
   ```
   Alters the default thread count for all subsequent parallel regions in the current thread's scope, unless overridden by a `num_threads` clause.
3. **Environment Variable Level (`export OMP_NUM_THREADS=N`):**
   Set from the operating system shell prior to launching the binary:
   ```bash
   $ export OMP_NUM_THREADS=16
   $ ./my_simulation
   ```
   Applies globally across the execution of the application if neither `num_threads()` nor `omp_set_num_threads()` is specified.
4. **Implementation Default:**
   If none of the above are set, the runtime queries the operating system for the number of online hardware threads (logical cores returned by `sysconf(_SC_NPROCESSORS_ONLN)` or `GetActiveProcessorCount`).

---

### 4.2 Core OpenMP Runtime APIs
OpenMP provides a runtime library accessible via `#include <omp.h>`. The most essential query and control functions are:

| Function Signature | Description | Scope / Behavior |
|:---|:---|:---|
| `int omp_get_thread_num(void);` | Returns the integer ID of the calling thread. | Master is always $0$; worker threads are $1, 2, \dots, T-1$. If called outside a parallel region, returns $0$. |
| `int omp_get_num_threads(void);` | Returns the number of threads currently in the active team. | Returns $T$ inside a parallel region; returns $1$ in sequential regions. |
| `int omp_get_max_threads(void);` | Returns the maximum number of threads that would be used if a parallel region without `num_threads` were entered now. | Safe to call outside parallel regions to allocate memory buffers proportional to team size. |
| `int omp_get_num_procs(void);` | Returns the number of logical processors available to the device at runtime. | Constant hardware query. |
| `int omp_in_parallel(void);` | Returns non-zero (true) if called within the dynamic extent of an active parallel region; otherwise returns $0$ (false). | Useful for writing libraries that must guard against nested oversubscription. |
| `double omp_get_wtime(void);` | Returns high-resolution elapsed wall-clock time in seconds (floating-point). | Monotonically increasing per-thread clock; ideal for benchmarking speedup. |
| `void omp_set_num_threads(int);` | Sets the default number of threads for subsequent parallel regions. | Must be passed a positive integer ($N \ge 1$). |

---

### 4.3 Compiling OpenMP Programs Across Vendors
To compile an OpenMP program, specific flags must be passed to activate the compiler's OpenMP transformation passes and link against the platform's OpenMP runtime library (`libgomp`, `libomp`, or `libiomp5`):

```bash
# 1. GNU Compiler Collection (GCC)
gcc -O3 -fopenmp main.c -o main_omp

# 2. Clang / LLVM
clang -O3 -fopenmp main.c -o main_omp

# 3. Intel oneAPI C/C++ Compiler (icx)
icx -O3 -qopenmp main.c -o main_omp

# 4. Microsoft Visual C++ (MSVC - Windows Developer Command Prompt)
cl /O2 /openmp main.c
```

#### What happens if compiled WITHOUT `-fopenmp`?
If you compile an OpenMP program without the flag:
```bash
gcc -O3 main.c -o main_serial
```
1. **Directives are ignored:** The compiler encounters `#pragma omp parallel`. Because `omp` is not recognized without the flag, standard C rules mandate that unknown pragmas are silently skipped or flagged with a minor warning:
   ```
   warning: ignoring '#pragma omp parallel' [-Wunknown-pragmas]
   ```
2. **Missing Symbols:** If your code explicitly calls runtime functions like `omp_get_thread_num()`, the compiler/linker will abort with:
   ```
   undefined reference to 'omp_get_thread_num'
   ```
3. **The Solution — Conditional Compilation (`_OPENMP`):**
   The OpenMP standard mandates that whenever an OpenMP-compliant compiler compiles with OpenMP enabled, it predefines the integer preprocessor macro `_OPENMP` with the value `yyyymm` (representing the year and month of the supported OpenMP specification).

   We can guard OpenMP headers and API calls to write code that compiles seamlessly both in parallel and serially:

   ```c
   #ifdef _OPENMP
       #include <omp.h>
   #else
       /* Stub replacements for serial compilation */
       #define omp_get_thread_num()  0
       #define omp_get_num_threads() 1
       #define omp_get_wtime()       0.0
   #endif
   ```

---

### 4.4 Execution Non-Determinism and Output Interleaving
When multiple threads execute concurrently, the operating system kernel's thread scheduler decides when each thread receives a CPU time slice. Therefore:
1. **Thread Arrival Order is Arbitrary:** In a team of 4 threads, thread 2 may execute its `printf` statement before thread 0 or thread 1.
2. **Output Stream Contention:** The standard output buffer (`stdout`) is a shared resource. Although standard C library `printf` functions typically acquire an internal lock on `stdout` during individual character stream emission, multiple threads calling `printf` concurrently cause their output lines to interleave unpredictably.

```
Run 1 Output (4 Threads):        Run 2 Output (Same Binary, Same Machine):
  Hello from thread 0 of 4         Hello from thread 2 of 4
  Hello from thread 2 of 4         Hello from thread 0 of 4
  Hello from thread 1 of 4         Hello from thread 3 of 4
  Hello from thread 3 of 4         Hello from thread 1 of 4
```
This behavior is completely normal in parallel computing and proves that the threads operate asynchronously and concurrently.

---

## 5. Compilable C Code Examples

### 5.1 Program 1: The Canonical OpenMP Parallel Region & Runtime Queries
This complete program demonstrates conditional compilation, runtime queries, thread team sizing, and execution non-determinism.

Save this file as `omp_parallel_basics.c`:

```c
/* =========================================================================
 * File: omp_parallel_basics.c
 * Description: Demonstrates OpenMP pragma syntax, parallel regions,
 *              runtime query APIs, and conditional compilation.
 * Compilation:
 *   Parallel:  gcc -O3 -fopenmp omp_parallel_basics.c -o omp_parallel_basics
 *   Serial:    gcc -O3 omp_parallel_basics.c -o omp_parallel_basics
 * Execution:
 *   ./omp_parallel_basics
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>

#ifdef _OPENMP
    #include <omp.h>
#else
    /* Fallback stubs for serial compilation without -fopenmp */
    int omp_get_thread_num(void)  { return 0; }
    int omp_get_num_threads(void) { return 1; }
    int omp_get_max_threads(void) { return 1; }
    int omp_get_num_procs(void)   { return 1; }
    int omp_in_parallel(void)     { return 0; }
#endif

int main(void) {
    printf("============================================================\n");
    printf("   OpenMP Pragma Syntax & Parallel Region Demonstration    \n");
    printf("============================================================\n\n");

    /* 1. Inspect OpenMP specification version */
    #ifdef _OPENMP
        printf("[INFO] OpenMP is ENABLED. Macro _OPENMP = %ld\n", (long)_OPENMP);
    #else
        printf("[INFO] OpenMP is DISABLED. Compiling as plain sequential C.\n");
    #endif

    printf("[INFO] Hardware Logical Processors Available : %d\n", omp_get_num_procs());
    printf("[INFO] Maximum Threads Configured (Default)  : %d\n", omp_get_max_threads());
    printf("[INFO] Currently in parallel region?          : %s\n\n",
           omp_in_parallel() ? "YES" : "NO");

    /* 2. Basic Parallel Region: Uses system default or OMP_NUM_THREADS */
    printf("--- Phase 1: Default Parallel Region ---\n");
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        int total_threads = omp_get_num_threads();

        /* Thread 0 performs administrative reporting */
        if (tid == 0) {
            printf("  -> Master Thread %d initialized team of %d threads.\n",
                   tid, total_threads);
        }

        printf("  [Thread %02d/%02d] Executing redundant parallel block (in_parallel=%d)\n",
               tid, total_threads, omp_in_parallel());
    } /* Implicit barrier here: all threads synchronize before exiting */

    printf("\n--- Phase 1 Complete. Sequential flow resumed. ---\n\n");

    /* 3. Overriding Thread Count via num_threads Clause */
    int custom_threads = 3;
    printf("--- Phase 2: Explicit num_threads(%d) Clause ---\n", custom_threads);

    #pragma omp parallel num_threads(custom_threads)
    {
        int tid = omp_get_thread_num();
        int total = omp_get_num_threads();
        printf("  [Custom Team] Hello from Thread %d of %d\n", tid, total);
    } /* Implicit barrier */

    printf("\n--- Phase 2 Complete. Sequential flow resumed. ---\n\n");

    /* 4. Overriding Thread Count via Runtime API omp_set_num_threads */
    #ifdef _OPENMP
        printf("--- Phase 3: Runtime API omp_set_num_threads(2) ---\n");
        omp_set_num_threads(2);

        #pragma omp parallel
        {
            int tid = omp_get_thread_num();
            int total = omp_get_num_threads();
            printf("  [API Team] Hello from Thread %d of %d\n", tid, total);
        } /* Implicit barrier */
        printf("\n--- Phase 3 Complete. Program exiting cleanly. ---\n");
    #endif

    return EXIT_SUCCESS;
}
```

### 5.2 Build Commands and Sample Outputs

#### Parallel Build (OpenMP Active):
```bash
$ gcc -O3 -fopenmp omp_parallel_basics.c -o omp_parallel_basics
$ ./omp_parallel_basics
```
```text
============================================================
   OpenMP Pragma Syntax & Parallel Region Demonstration    
============================================================

[INFO] OpenMP is ENABLED. Macro _OPENMP = 201511
[INFO] Hardware Logical Processors Available : 8
[INFO] Maximum Threads Configured (Default)  : 8
[INFO] Currently in parallel region?          : NO

--- Phase 1: Default Parallel Region ---
  -> Master Thread 0 initialized team of 8 threads.
  [Thread 00/08] Executing redundant parallel block (in_parallel=1)
  [Thread 04/08] Executing redundant parallel block (in_parallel=1)
  [Thread 02/08] Executing redundant parallel block (in_parallel=1)
  [Thread 01/08] Executing redundant parallel block (in_parallel=1)
  [Thread 06/08] Executing redundant parallel block (in_parallel=1)
  [Thread 07/08] Executing redundant parallel block (in_parallel=1)
  [Thread 03/08] Executing redundant parallel block (in_parallel=1)
  [Thread 05/08] Executing redundant parallel block (in_parallel=1)

--- Phase 1 Complete. Sequential flow resumed. ---

--- Phase 2: Explicit num_threads(3) Clause ---
  [Custom Team] Hello from Thread 0 of 3
  [Custom Team] Hello from Thread 2 of 3
  [Custom Team] Hello from Thread 1 of 3

--- Phase 2 Complete. Sequential flow resumed. ---

--- Phase 3: Runtime API omp_set_num_threads(2) ---
  [API Team] Hello from Thread 0 of 2
  [API Team] Hello from Thread 1 of 2

--- Phase 3 Complete. Program exiting cleanly. ---
```

#### Serial Build (Compiled Without `-fopenmp`):
```bash
$ gcc -O3 omp_parallel_basics.c -o omp_parallel_basics
$ ./omp_parallel_basics
```
```text
============================================================
   OpenMP Pragma Syntax & Parallel Region Demonstration    
============================================================

[INFO] OpenMP is DISABLED. Compiling as plain sequential C.
[INFO] Hardware Logical Processors Available : 1
[INFO] Maximum Threads Configured (Default)  : 1
[INFO] Currently in parallel region?          : NO

--- Phase 1: Default Parallel Region ---
  -> Master Thread 0 initialized team of 1 threads.
  [Thread 00/01] Executing redundant parallel block (in_parallel=0)

--- Phase 1 Complete. Sequential flow resumed. ---

--- Phase 2: Explicit num_threads(3) Clause ---
  [Custom Team] Hello from Thread 0 of 1

--- Phase 2 Complete. Sequential flow resumed. ---
```

---

## 6. Mathematical Model & Complexity Analysis

### 6.1 Parallel Region Fork-Join Overhead Model
Entering and exiting an OpenMP parallel region is not mathematically free. Even when threads are pre-spawned in an internal runtime thread pool, coordinating a team introduces runtime overhead:

$$T_{\text{total}} = T_{\text{comp}}(p) + T_{\text{fork}}(p) + T_{\text{barrier}}(p)$$

Where:
- $p$ is the number of threads participating in the team.
- $T_{\text{comp}}(p) = \frac{W}{p}$ is the ideal parallel computation time for total work $W$.
- $T_{\text{fork}}(p)$ is the latency required by the master thread to signal dormant worker threads in the pool, push the stack activation frame, and awaken the workers.
- $T_{\text{barrier}}(p)$ is the cost of the implicit barrier at the end of the region, where threads synchronize before the master thread resumes sequential execution.

### 6.2 Amortization and Minimum Granularity
Because thread pool wake-up and barrier synchronization typically cost between **$0.5\,\mu\text{s}$ and $5\,\mu\text{s}$** ($1{,}500$ to $15{,}000$ CPU cycles on modern x86-64 hardware depending on core counts and NUMA hops):

$$\text{Break-Even Condition: } W_{\text{sequential}} - \left(\frac{W_{\text{sequential}}}{p} + T_{\text{overhead}}(p)\right) > 0$$

$$\implies W_{\text{sequential}} \left(1 - \frac{1}{p}\right) > T_{\text{overhead}}(p)$$

$$\implies W_{\text{sequential}} > \left(\frac{p}{p - 1}\right) T_{\text{overhead}}(p)$$

> [!IMPORTANT]
> If a computation takes less than $\approx 10\,\mu\text{s}$, wrapping it inside a `#pragma omp parallel` region will **slow it down** compared to serial execution because $T_{\text{overhead}}$ will dominate the computation!

---

## 7. Advantages & Limitations

### Advantages
1. **Incremental Adoption:** Existing serial applications can be parallelized step-by-step without rewriting the system architecture.
2. **Unified Codebase:** With `#ifdef _OPENMP` and standard pragma rules, a single source file compiles cleanly on high-end parallel supercomputers and minimal embedded systems lacking OpenMP libraries.
3. **Low Programmer Boilerplate:** Replaces hundreds of lines of error-prone Pthreads thread management with single-line declarative directives.
4. **Optimized Runtime Thread Pooling:** Modern OpenMP runtimes (`libgomp`, `libomp`) keep worker threads sleeping in user-space or kernel wait states, eliminating repeated OS-level kernel context allocation overhead.

### Limitations & Traps
1. **Redundant Execution Hazard:** Writing `#pragma omp parallel` alone does **not** split loop iterations across threads. Every thread runs the entire enclosed code redundantly unless a work-sharing construct (`#pragma omp for`) is attached.
2. **Implicit Barrier Overhead:** Forgetting that `{ ... }` has an implicit barrier at the closing brace can introduce severe performance stalls if thread workloads are unbalanced.
3. **Compiler Invisibility:** If the `-fopenmp` flag is accidentally omitted during compilation, no compiler error is generated by default; the code silently runs serially, baffling novice developers.
4. **Standard Output Chaos:** Debugging with `printf` inside parallel regions causes garbled output due to lack of console stream synchronization.

---

## 8. Formal Definitions for Exam

- **OpenMP Directive:** A C/C++ preprocessor directive beginning with the sentinel `#pragma omp`, followed by a directive name and optional clauses, instructing an OpenMP-compliant compiler to generate multithreaded runtime routines for the associated structured block.
- **Structured Block:** An executable statement or compound block (`{ ... }`) in C/C++ with exactly one point of entry at the top and one point of exit at the bottom, strictly prohibiting branches (`goto`, `longjmp`) into or out of the block.
- **Master Thread:** The original OS thread executing the application that encounters an OpenMP parallel directive and serves as Thread ID $0$ within the newly forked team.
- **Thread Team:** The collection of threads consisting of the master thread and one or more worker threads created or allocated by the runtime to execute a parallel region.
- **Implicit Barrier:** A synchronization barrier automatically placed by the OpenMP specification at the end of specific constructs (such as `#pragma omp parallel`), requiring all team threads to arrive before any thread can proceed.
- **`_OPENMP` Macro:** A standard predefined preprocessor macro whose integer value represents the specification edition supported by the compiler in `yyyymm` format (e.g., `201511` for OpenMP 4.5).

---

## 9. Comparison Tables

### Table 1: Thread Team Sizing Mechanisms Comparison

| Mechanism | Syntax / Command | Precedence Rank | Lifetime / Scope | Overridable by Lower Ranks? |
|:---|:---|:---:|:---|:---:|
| **Directive Clause** | `#pragma omp parallel num_threads(N)` | **1 (Highest)** | Single parallel region only | No |
| **Runtime API** | `omp_set_num_threads(N);` | **2** | All subsequent parallel regions until next call | Overridden by `num_threads` clause |
| **Environment Variable** | `export OMP_NUM_THREADS=N` | **3** | Entire process execution | Overridden by Clause and API |
| **System Default** | Implementation defined | **4 (Lowest)** | Default fallback | Overridden by all above |

---

### Table 2: OpenMP Compilation Across Major Compilers

| Compiler Suite | C/C++ OpenMP Flag | Default Thread Library Linked | OpenMP Macro (`_OPENMP`) Example |
|:---|:---:|:---:|:---:|
| **GNU GCC** | `-fopenmp` | `libgomp.so` / `libgomp.a` | `201511` (v4.5 on GCC 9+) |
| **LLVM Clang** | `-fopenmp` | `libomp.so` | `201811` (v5.0 on Clang 12+) |
| **Intel oneAPI (`icx`)** | `-qopenmp` | `libiomp5.so` | `202011` (v5.1/5.2) |
| **MSVC (Windows)** | `/openmp` | `vcomp.lib` | `200203` (v2.0 legacy default) |

---

## 10. Solved Analytical & Code-Tracing Problems

### Problem 1: Precedence Resolution Trace
**Question:** Analyze the following C code snippet. Assuming the machine has 16 logical cores, and the user executed the program from the bash terminal after running `export OMP_NUM_THREADS=8`, state the exact thread count $T$ for Regions A, B, and C.

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    /* Region A */
    #pragma omp parallel
    {
        /* Code */
    }

    omp_set_num_threads(4);

    /* Region B */
    #pragma omp parallel
    {
        /* Code */
    }

    /* Region C */
    #pragma omp parallel num_threads(2)
    {
        /* Code */
    }

    return 0;
}
```

**Step-by-Step Solution:**
1. **Region A:**
   - Clause `num_threads` present? **No.**
   - API `omp_set_num_threads()` called prior? **No.**
   - Environment variable `OMP_NUM_THREADS` set? **Yes ($8$).**
   - **Resolution:** Region A executes with **$T = 8$ threads**.
2. **Region B:**
   - Prior to Region B, `omp_set_num_threads(4)` is executed.
   - Clause `num_threads` present on Region B? **No.**
   - Runtime API precedence (Level 2) beats environment variable (Level 3).
   - **Resolution:** Region B executes with **$T = 4$ threads**.
3. **Region C:**
   - Clause `num_threads(2)` is explicitly specified on the directive.
   - Directive clause precedence (Level 1) overrides all API settings and environment variables.
   - **Resolution:** Region C executes with **$T = 2$ threads**.

---

### Problem 2: Structured Block Violation Identification
**Question:** Identify the compilation error in the following OpenMP code segment and explain why the OpenMP specification forbids it.

```c
void search_value(int *arr, int n, int target) {
    #pragma omp parallel num_threads(4)
    {
        int tid = omp_get_thread_num();
        for (int i = tid; i < n; i += 4) {
            if (arr[i] == target) {
                printf("Target found at index %d by thread %d\n", i, tid);
                goto target_found; /* Line X */
            }
        }
    }
target_found:
    printf("Search complete.\n");
}
```

**Step-by-Step Solution:**
1. **Rule Violation:** Line X contains `goto target_found;`, which attempts to jump directly out of the parallel region block to a label in the enclosing function.
2. **Specification Constraint:** OpenMP mandates that the code following `#pragma omp parallel` must be a **structured block** with a single entry point and a single exit point. Jumping out of the block via `goto`, `break`, or `return` bypasses the function outlining boundary and the runtime's mandatory **implicit barrier**.
3. **Silicon/Runtime Consequence:** If one thread jumps out, the internal thread team state machine becomes corrupt: the remaining 3 threads will reach the implicit barrier and block forever (deadlock), waiting for the departed thread that will never arrive.
4. **Fix:** Use a shared boolean flag or atomic condition variable so threads exit the block normally through the closing brace `}`.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Does `#pragma omp parallel` automatically split up a loop across multiple threads?
**Answer:** **NO.** This is the most common beginner misconception in OpenMP. 
`#pragma omp parallel` solely creates a team of threads. Every thread in that team will execute **every single instruction** inside the enclosed compound block `{ ... }`. 
If you place an unannotated `for` loop inside a `#pragma omp parallel` block with 4 threads, all 4 threads will redundantly run the loop from $0$ to $N$, performing $4\times$ the total work. To partition loop iterations across threads, you must bind a **work-sharing construct** such as `#pragma omp for` or use the combined construct `#pragma omp parallel for`.

### Q2: What is the difference between `omp_get_num_threads()` and `omp_get_max_threads()`?
**Answer:**
- `omp_get_num_threads()` queries the **active** team size. If called inside a sequential region (outside `#pragma omp parallel`), it always returns **1**, because only the master thread is active.
- `omp_get_max_threads()` queries the **configured thread limit** that will be used if a new parallel region without an explicit `num_threads` clause is encountered. It can be safely called in serial code to pre-allocate thread-local arrays:
  ```c
  int max_t = omp_get_max_threads();
  double *local_sums = (double*)malloc(sizeof(double) * max_t);
  ```

### Q3: Why is standard output (`printf`) output often shuffled or garbled when printing from multiple threads?
**Answer:** Even though individual standard C `printf` calls are thread-safe (libc locks `stdout` during a single call), the OS thread scheduler grants execution quanta independently to CPU cores. Thread $0$ may print its first line, get pre-empted by an OS timer interrupt, and Thread $3$ may be scheduled on another core and print its line. Because there is no ordering guarantee among threads in a parallel region, lines appear in arbitrary, non-deterministic order.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    OPENMP PRAGMA CHEAT SHEET                                       |
+----------------------------------------------------------------------------------------------------+
| Basic Directive        | #pragma omp parallel                                                      |
| Line Continuation      | Use backslash: \ at the end of physical line                              |
| Team Size Precedence   | num_threads() clause > omp_set_num_threads() > OMP_NUM_THREADS > Default  |
| Outlining Mechanism    | Compiler extracts block to internal static function, invokes runtime fork  |
| Mandatory Exit Rule    | Structured block: strictly ONE entry, strictly ONE exit (no goto/return)  |
| Barrier Behavior       | Implicit synchronization barrier at the closing brace: }                  |
| Predefined Macro       | _OPENMP (Integer: yyyymm, e.g., 201511 for v4.5)                          |
+----------------------------------------------------------------------------------------------------+
|                                 CORE RUNTIME APIS (<omp.h>)                                        |
+----------------------------------------------------------------------------------------------------+
| omp_get_thread_num()   | Returns calling thread ID: 0 (master) to T - 1                            |
| omp_get_num_threads()  | Returns active thread team size T (returns 1 in serial code)               |
| omp_get_max_threads()  | Returns capacity limit for next parallel region (callable in serial)      |
| omp_set_num_threads(N) | Sets team size for subsequent parallel regions                            |
| omp_get_wtime()        | Returns double precision wall-clock timestamp in seconds                  |
+----------------------------------------------------------------------------------------------------+
|                                 COMMON COMPILER FLAGS                                              |
+----------------------------------------------------------------------------------------------------+
| GCC / Clang            | gcc -O3 -fopenmp source.c -o binary                                       |
| Intel oneAPI           | icx -O3 -qopenmp source.c -o binary                                       |
| MSVC (Windows)         | cl /O2 /openmp source.c                                                   |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Write syntactically valid OpenMP pragmas, handling multi-line directives with line continuations (`\`).
2. Explain the function outlining transformation performed by the compiler when encountering `#pragma omp parallel`.
3. Resolve thread counts deterministically using the 4-level precedence hierarchy (`num_threads`, `omp_set_num_threads`, `OMP_NUM_THREADS`, system defaults).
4. Utilize `_OPENMP` conditional compilation macros to write portable code that compiles both with and without `-fopenmp`.
5. Identify and rectify illegal jumps (`goto`, `break`, `return`) that violate OpenMP structured block rules.
6. Use runtime APIs (`omp_get_thread_num`, `omp_get_num_threads`, `omp_get_max_threads`) correctly in both serial and parallel scopes.
