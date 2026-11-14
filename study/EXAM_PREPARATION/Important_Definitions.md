# Master Examination Glossary: Important Definitions in Parallel Programming & HPC

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 1–32)  
**Applicability:** University Theory Examinations (MidSem & EndSem) and Lab Viva Voce  
**Standard:** Every definition is formulated to meet the highest university grading rubrics, providing exact formal terminology, mathematical representations, API signatures, and critical grading nuances.

---

## Table of Contents
1. [HPC Landscape, Architecture & Performance Models (Lectures 1–5)](#1-hpc-landscape-architecture--performance-models-lectures-15)
2. [OpenMP Architecture, Pragmas, Scoping & Work-Sharing (Lectures 6–10)](#2-openmp-architecture-pragmas-scoping--work-sharing-lectures-610)
3. [OpenMP Synchronization, Tasks & SIMD Vectorization (Lectures 11–16)](#3-openmp-synchronization-tasks--simd-vectorization-lectures-1116)
4. [POSIX Threads & Concurrency Primitives (Lectures 17–23)](#4-posix-threads--concurrency-primitives-lectures-1723)
5. [Distributed Memory, MPI & Hybrid Programming (Lectures 24–32)](#5-distributed-memory-mpi--hybrid-programming-lectures-2432)

---

## 1. HPC Landscape, Architecture & Performance Models (Lectures 1–5)

### High-Performance Computing (HPC)
> **Formal Definition:**  
> The practice of aggregating computing power across multiple physical processors, execution cores, or discrete distributed compute nodes to execute complex computational simulations, numerical models, or data-intensive workloads at speeds substantially higher than achievable on a typical general-purpose workstation.  
> **Key Nuance:** HPC systems emphasize sustained arithmetic throughput ($\text{FLOP/s}$), low-latency inter-node communication, and high aggregate memory bandwidth rather than high multi-tasking desktop responsiveness.

### FLOP/s (Floating-Point Operations per Second)
> **Formal Definition:**  
> The standard quantitative metric measuring the arithmetic compute rate of a processor or parallel computer, defined as the total number of floating-point arithmetic operations (additions, multiplications, fused multiply-adds) completed divided by execution time in seconds:
> $$\text{FLOP/s} = \frac{\text{Floating-Point Operations}}{\text{Execution Time (seconds)}}$$
> **Metric Multiples:** GFLOP/s ($10^9$), TFLOP/s ($10^{12}$), PFLOP/s ($10^{15}$), EFLOP/s ($10^{18}$).  
> **Key Nuance:** Distinguish between *Peak Theoretical FLOP/s* ($R_{\text{peak}} = \text{Cores} \times \text{Clock Frequency} \times \text{FLOPs/cycle}$) and *Sustained/Linpack FLOP/s* ($R_{\text{max}}$).

### TOP500 Supercomputer Ranking
> **Formal Definition:**  
> A bi-annual ranking (published in June and November) that ranks the 500 most powerful commercially active supercomputers globally based exclusively on their sustained performance ($R_{\text{max}}$) achieved while solving a dense system of linear equations using the High-Performance Linpack (HPL) benchmark.

### The Seven Dwarfs of Numerical Methods
> **Formal Definition:**  
> A taxonomic classification formulated by researchers at UC Berkeley identifying seven fundamental algorithmic patterns that capture the computational and communication behaviors of scientific and engineering applications across hardware architectures.  
> **The 7 Dwarfs:** (1) Dense Linear Algebra, (2) Sparse Linear Algebra, (3) Spectral Methods (FFT), (4) N-Body Methods, (5) Structured Grids, (6) Unstructured Grids, and (7) MapReduce / Monte Carlo.

### Instruction Pipelining
> **Formal Definition:**  
> An architectural implementation technique whereby multiple processor instructions are overlapped in execution across a multi-stage hardware pipeline (e.g., Fetch, Decode, Execute, Memory Access, Writeback), achieving an ideal throughput of one completed instruction per clock cycle under steady-state conditions ($CPI \to 1$).

### Superscalar Execution
> **Formal Definition:**  
> A microarchitectural technique enabling a CPU core to issue, execute, and retire multiple independent instructions simultaneously during a single clock cycle through the provision of redundant, parallel execution units (multiple ALUs, FPUs, load/store units).

### Instruction-Level Parallelism (ILP)
> **Formal Definition:**  
> A measure of how many operations in a computer program can be executed simultaneously without violating control flow dependencies, data flow dependencies (RAW, WAR, WAW), or structural hardware resource constraints.

### Simultaneous Multithreading (SMT / Hyper-Threading)
> **Formal Definition:**  
> A hardware architectural feature enabling a single physical CPU core with superscalar execution units to maintain the architectural state of multiple independent threads of execution (multiple register sets, program counters, and control registers) simultaneously, dynamically interleaving instructions to maximize pipeline utilization.

### SIMD (Single Instruction, Multiple Data)
> **Formal Definition:**  
> A class of parallel hardware architectures in Flynn's taxonomy wherein a single control unit broadcasts an identical machine instruction to multiple parallel arithmetic processing elements, which simultaneously execute that operation across distinct data elements stored in wide vector registers.

### Memory Bandwidth
> **Formal Definition:**  
> The maximum theoretical or sustained rate at which data can be transferred between physical dynamic random-access memory (DRAM) and the processor cache hierarchy, measured in Gigabytes per second ($\text{GB/s}$):
> $$\text{Bandwidth} = \text{Bus Width (bytes)} \times \text{Memory Clock Frequency (Hz)} \times \text{Transfers per Clock}$$

### Flynn's Taxonomy
> **Formal Definition:**  
> A 1972 classification system proposed by Michael J. Flynn categorizing all computer architectures into four structural classes based on the multiplicity of hardware instruction streams and data streams:
> 1. **SISD (Single Instruction, Single Data):** Classic sequential von Neumann machine.
> 2. **SIMD (Single Instruction, Multiple Data):** Vector processors, GPU streaming multiprocessors.
> 3. **MISD (Multiple Instruction, Single Data):** Systolic arrays, space flight fault-tolerant voting systems.
> 4. **MIMD (Multiple Instruction, Multiple Data):** Multicore processors, HPC clusters, distributed-memory supercomputers.

### UMA vs. ccNUMA Architectures
> **Uniform Memory Access (UMA):**  
> A shared-memory parallel hardware architecture where all physical processors connect to a centralized main memory via a shared system bus or crossbar switch, exhibiting identical access latencies and bandwidths for all memory addresses regardless of which core initiates the read or write.
> 
> **Cache-Coherent Non-Uniform Memory Access (ccNUMA):**  
> A shared-memory parallel architecture where memory is physically partitioned among distinct processor sockets (NUMA nodes). While any core can globally address all memory addresses through hardware-maintained cache coherency, access to memory physically wired to the local socket is significantly faster than access to remote socket memory across inter-socket interconnects (UPI, Infinity Fabric).

### Speedup ($S(N)$)
> **Formal Definition:**  
> The dimensionless ratio quantifying the performance improvement achieved by executing a program using $N$ parallel processing elements compared to the execution time of the best known sequential algorithm on a single processing element:
> $$S(N) = \frac{T(1)}{T(N)}$$
> Where $T(1)$ is the sequential execution time, and $T(N)$ is the execution time on $N$ processing units.

### Parallel Efficiency ($\varepsilon(N)$)
> **Formal Definition:**  
> The fraction of total processing power utilized effectively by a parallel program, defined as the speedup achieved per processing unit:
> $$\varepsilon(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$$
> **Ideal Target:** $\varepsilon(N) = 1.0$ (100% linear efficiency). In real systems, $\varepsilon(N) < 1.0$ due to communication overhead, synchronization latency, and serial bottlenecks.

### Amdahl's Law (Strong Scaling)
> **Formal Definition:**  
> A mathematical law formulating the maximum theoretical speedup limit of a parallel program running on $N$ processors when the total problem workload size remains strictly fixed:
> $$S_{\text{Amdahl}}(N) = \frac{1}{s + \frac{1 - s}{N}} \quad \xrightarrow{N \to \infty} \quad S_{\text{max}} = \frac{1}{s}$$
> Where $s$ is the inherently sequential fraction of the execution time ($0 \le s \le 1$), and $p = 1 - s$ is the parallelizable fraction.  
> **Key Nuance:** Highlights that the serial fraction $s$ acts as an insurmountable performance brick wall under strong scaling as $N$ grows indefinitely.

### Gustafson's Law (Weak Scaling)
> **Formal Definition:**  
> A mathematical law formulating the scaled speedup achievable on $N$ processors when the problem workload size is scaled proportionally with the number of processors to maintain a constant per-processor execution time:
> $$S_{\text{Gustafson}}(N) = s + (1 - s)N = N + s(1 - N)$$
> Where $s$ is the sequential fraction of execution time measured on the parallel system.  
> **Key Nuance:** Demonstrates that massive parallel architectures are computationally viable because scaling problem dimensions dilutes the impact of fixed serial code.

### Strong Scaling vs. Weak Scaling
> **Strong Scaling:**  
> The scaling regime where the **total problem size is held constant** while the number of processors $N$ increases. The objective is minimizing time-to-solution for a fixed workload. Governed by Amdahl's Law.
> 
> **Weak Scaling:**  
> The scaling regime where the **workload assigned per processor is held constant**, meaning the total problem size grows linearly with the processor count $N$ ($W_{\text{total}} = N \times W_0$). The objective is solving larger scientific simulations within a fixed wall-clock budget. Governed by Gustafson's Law.

### Load Imbalance
> **Formal Definition:**  
> The non-uniform distribution of computational work across parallel threads or processes, causing some execution units to finish early and idle at barriers while waiting for the slowest execution unit (the straggler thread) to complete.
> $$\text{Imbalance Factor} = \frac{T_{\text{max}} - T_{\text{avg}}}{T_{\text{max}}}$$

---

## 2. OpenMP Architecture, Pragmas, Scoping & Work-Sharing (Lectures 6–10)

### Shared-Memory Programming Model
> **Formal Definition:**  
> A parallel execution paradigm in which multiple concurrently executing threads run within a single operating system process and share a common global virtual address space, permitting direct inter-thread communication via standard read and write memory instructions.

### Fork-Join Execution Model
> **Formal Definition:**  
> The foundational execution paradigm of OpenMP in which an application begins as a single sequential thread (the master thread / initial thread). Upon encountering an OpenMP parallel pragma, the master thread **forks** a team of worker threads. At the end of the parallel block, an implicit barrier occurs, the team of worker threads **joins**, and the master thread resumes single-threaded sequential execution.

### Pragma Directive (`#pragma omp ...`)
> **Formal Definition:**  
> A standardized C/C++ preprocessor directive that conveys parallelization commands, work-sharing patterns, synchronization constraints, and scoping rules to an OpenMP-compliant compiler without altering the semantic validity of the code on compilers that do not support OpenMP (which safely ignore unhandled pragmas).

### Data Scoping
> **Formal Definition:**  
> The explicit classification of variables within an OpenMP parallel region to determine whether an allocated variable is globally shared across all threads in the team or instantiated independently in thread-private memory (thread stack).

### Shared Variable (`shared(var)`)
> **Formal Definition:**  
> A data scoping attribute specifying that a single memory storage location is allocated for the variable, and that all threads in the OpenMP team access, modify, and reference that exact identical physical memory address throughout the parallel region.

### Private Variable (`private(var)`)
> **Formal Definition:**  
> A data scoping attribute specifying that each thread in the OpenMP team allocates an uninitialized local copy of the variable on its private execution stack.  
> **Key Nuance:** The variable is **uninitialized** upon entry into the parallel region, and its value is **undefined** upon exit from the parallel construct; the original variable outside the construct retains its pre-parallel value.

### `firstprivate(var)`
> **Formal Definition:**  
> An OpenMP data scoping clause that allocates a private instance of the variable on each thread's stack and automatically initializes each private copy with the current value of the master thread's corresponding variable immediately prior to entering the parallel construct.

### `default(none)`
> **Formal Definition:**  
> A defensive OpenMP clause that disables compiler default scoping heuristics, forcing the programmer to explicitly specify the scoping rule (`shared`, `private`, `firstprivate`, `reduction`) for every variable referenced within the lexical extent of the parallel region, eliminating accidental data race conditions during compilation.

### Work-Sharing Construct
> **Formal Definition:**  
> An OpenMP construct (`for`, `sections`, `single`) enclosed within an active parallel region that divides the execution of an enclosed block of code or loop iteration space among the threads of the existing team, rather than replicating the entire block across every thread.

### Loop Canonical Form
> **Formal Definition:**  
> The standardized syntactic structure required for a `for` loop to be parallelizable by `#pragma omp for`, characterized by an integer/pointer index, an invariant loop bound, an increment expression of monotonic form (`i++`, `i += step`), and no premature loop termination (`break`, `goto`, `return`). Canonical form allows the compiler to determine the exact iteration count prior to loop execution.

### `sections` and `section` Constructs
> **Formal Definition:**  
> An OpenMP work-sharing construct that partitions non-iterative, independent functional blocks of code among the threads of a team:
> ```c
> #pragma omp sections
> {
>     #pragma omp section { taskA(); }
>     #pragma omp section { taskB(); }
> }
> ```
> Each enclosed `#pragma omp section` block is executed exactly once by an individual thread.

### `single` Construct
> **Formal Definition:**  
> An OpenMP work-sharing construct that specifies that an enclosed code block must be executed by **only one arbitrary thread** in the team (not necessarily the master thread), while all other threads in the team wait at an implicit barrier at the end of the construct (unless suppressed with `nowait`).

### `nowait` Clause
> **Formal Definition:**  
> A performance optimization clause appended to OpenMP work-sharing constructs (`for`, `sections`, `single`) that suppresses the mandatory implicit barrier synchronization at the closing brace of the construct, allowing threads that finish early to proceed immediately to subsequent instructions without idling.

### Loop Scheduling Strategies
> **Static Scheduling (`schedule(static, chunk)`):**  
> An iteration distribution strategy where the loop iteration space is divided into chunks of size `chunk` and assigned to threads in round-robin fashion **at compile-time/launch-time**. Offers minimal runtime dispatch overhead; optimal for perfectly balanced, uniform workloads.
> 
> **Dynamic Scheduling (`schedule(dynamic, chunk)`):**  
> A runtime work-stealing strategy where iterations are divided into chunks and placed in a shared task queue. Threads request a chunk, process it, and dynamically request subsequent chunks upon completion. Introduces locking overhead; optimal for irregular, unpredictable workloads.
> 
> **Guided Scheduling (`schedule(guided, min_chunk)`):**  
> A dynamic scheduling strategy where chunk sizes decrease exponentially over time ($Chunk \approx \frac{\text{Remaining Iterations}}{\text{Number of Threads}}$) down to a specified minimum threshold (`min_chunk`). Combines low initial overhead with fine-grained dynamic load balancing during the final iterations.

---

## 3. OpenMP Synchronization, Tasks & SIMD Vectorization (Lectures 11–16)

### Race Condition (Data Race)
> **Formal Definition:**  
> An anomalous concurrent execution flaw occurring when two or more concurrent execution threads access the same memory location simultaneously without synchronization, where at least one of the accesses is a write, resulting in non-deterministic and program-corrupting execution behavior.

### Critical Section (`#pragma omp critical`)
> **Formal Definition:**  
> A mutual exclusion synchronization construct that restricts the execution of an enclosed block of code to **exactly one thread at a time**, forcing all other threads to block and wait until the active thread exits the critical section.

### Atomic Operation (`#pragma omp atomic`)
> **Formal Definition:**  
> A low-overhead synchronization construct that guarantees that a single, specific memory read-modify-write operation (e.g., `x += val`, `counter++`) is performed as an indivisible hardware transaction without using general software mutex locks, leveraging native CPU atomic instructions (`LOCK XADD`, `CMPXCHG`).

### OpenMP Barrier (`#pragma omp barrier`)
> **Formal Definition:**  
> An explicit synchronization point within a parallel region at which every thread in the team must arrive and halt execution; no thread may advance beyond the barrier until all threads in the team have reached the barrier.

### OpenMP Lock (`omp_lock_t`)
> **Formal Definition:**  
> A low-level, programmer-controlled synchronization object providing mutual exclusion via an explicit procedural API: `omp_init_lock()`, `omp_set_lock()`, `omp_unset_lock()`, `omp_test_lock()`, and `omp_destroy_lock()`.

### False Sharing
> **Formal Definition:**  
> A severe hardware performance degradation occurring in multiprocessor shared-memory architectures when distinct threads on different cores update independent variables that reside within the **same physical cache line** (typically 64 bytes). The cache coherency protocol (MESI/MOESI) forces the cache line to bounce between core caches in an invalid state, stalling memory pipelines despite no logical data dependency existing.

### Parallel Reduction (`reduction(op:var)`)
> **Formal Definition:**  
> An OpenMP clause that calculates an aggregate scalar result across all iterations or threads using a specified associative and commutative binary operator (`+`, `*`, `min`, `max`). The OpenMP runtime allocates a private accumulator initialized to the operator's mathematical identity, accumulates thread-local sums independently without synchronization overhead, and combines the local accumulators into the original shared variable at the conclusion of the construct.

### Mathematical Preconditions for Parallel Reduction
> **Associativity:** $(a \oplus b) \oplus c = a \oplus (b \oplus c)$ (guarantees that grouping sub-problems hierarchically preserves numerical correctness).  
> **Commutativity:** $a \oplus b = b \oplus a$ (guarantees that non-deterministic thread completion order does not alter the mathematical result).  
> *Note:* Floating-point addition is algebraically associative but strictly non-associative in IEEE-754 finite-precision arithmetic due to rounding errors ($ (a+b)+c \ne a+(b+c) $).

### Task Parallelism vs. Data Parallelism
> **Data Parallelism:**  
> A parallel execution paradigm where identical computational operations are applied concurrently to distinct subsets of a large, structured dataset or array (typified by loop-based `#pragma omp for`).
> 
> **Task Parallelism:**  
> A flexible parallel execution paradigm where distinct, potentially heterogeneous units of work (tasks) comprising arbitrary functions, varying execution times, and complex recursive dependencies are packaged and queued dynamically for execution by an available pool of worker threads.

### OpenMP Task Construct (`#pragma omp task`)
> **Formal Definition:**  
> A construct that packages an enclosed block of code, together with its current data environment, into an independent execution unit (a task) that is added to the OpenMP runtime task pool to be executed asynchronously by any idle thread in the team.

### `taskloop` Construct (`#pragma omp taskloop`)
> **Formal Definition:**  
> An OpenMP 4.5 construct that divides the iterations of an enclosed loop into discrete task chunks, submitting each chunk as an independent OpenMP task to the runtime scheduler. Controlled via `grainsize(g)` (specifies iterations per task) or `num_tasks(n)` (specifies total tasks generated).

### SIMD Loop (`#pragma omp simd`)
> **Formal Definition:**  
> An OpenMP directive that instructs the compiler to transform a sequential loop into vector machine instructions (e.g., Intel AVX-512, ARM SVE) executed by the CPU's hardware vector execution pipelines, asserting that loop iterations have no cross-iteration dependencies that prevent vector execution.

### Cache-Line Alignment (`aligned(ptr:byte_boundary)`)
> **Formal Definition:**  
> An OpenMP SIMD clause informing the compiler that a data pointer is stored at a memory address that is an exact multiple of the specified byte boundary (e.g., `aligned(A:64)` for 64-byte AVX-512 vector register boundaries), allowing the code generator to emit high-speed aligned vector load/store instructions (`_mm512_load_pd`) rather than slow unaligned memory instructions.

---

## 4. POSIX Threads & Concurrency Primitives (Lectures 17–23)

### POSIX Standard (IEEE 1003.1)
> **Formal Definition:**  
> The Portable Operating System Interface standard defined by the IEEE Computer Society that specifies vendor-neutral application programming interfaces (APIs), system call signatures, shell behaviors, and utility tools across Unix-like operating systems to guarantee software source-code portability.

### POSIX Threads (Pthreads / `<pthread.h>`)
> **Formal Definition:**  
> A standardized C-language application programming interface (POSIX standard 1003.1c) that provides low-level operating system thread management, thread attribute configuration, mutual exclusion locks, condition variables, barriers, and read-write synchronization primitives.

### Thread Handle (`pthread_t`)
> **Formal Definition:**  
> An opaque, platform-dependent data type defined in `<pthread.h>` that uniquely identifies a POSIX thread within an operating system process. Used by thread management functions (`pthread_create`, `pthread_join`, `pthread_cancel`) to direct actions to that specific thread.

### `pthread_create`
> **Formal Definition:**  
> The fundamental POSIX system call that instantiates a new concurrent thread of execution within the calling process:
> ```c
> int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
>                    void *(*start_routine)(void *), void *arg);
> ```
> Returns `0` on success; returns a positive system error code (`EAGAIN`, `EINVAL`, `EPERM`) on failure.

### Joinable vs. Detached Threads
> **Joinable Thread (Default):**  
> A thread whose termination status, return value pointer, and thread ID are retained by the operating system kernel after its execution terminates, until another thread explicitly calls `pthread_join()` to read the return status and reclaim its kernel storage (prevents thread resource leaks / zombie threads).
> 
> **Detached Thread:**  
> A thread whose resources are automatically and immediately reclaimed by the operating system kernel upon its termination, without requiring another thread to join it. Created via `pthread_detach(tid)` or setting `PTHREAD_CREATE_DETACHED` in thread attributes.

### Mutex (`pthread_mutex_t`)
> **Formal Definition:**  
> A low-level mutual exclusion primitive providing binary sleep-lock synchronization. Exactly one thread may own the mutex at any given time. Any other thread attempting to lock an owned mutex is suspended by the operating system kernel into a blocking queue, yielding the CPU until the mutex is released.

### Priority Inversion
> **Formal Definition:**  
> An anomalous scheduling condition where a high-priority thread is blocked waiting for a shared mutex held by a low-priority thread, while intermediate-priority threads preempt the low-priority thread, effectively causing the high-priority thread to be indirectly blocked by intermediate-priority threads. Resolved via Priority Inheritance Protocols.

### Condition Variable (`pthread_cond_t`)
> **Formal Definition:**  
> An explicit thread coordination synchronization primitive that enables threads to atomically suspend execution and sleep without consuming CPU resources until a specific application-defined state condition or predicate becomes true, signaled by another executing thread.

### The Mandatory Atomicity Rule of `pthread_cond_wait`
> **Formal Definition:**  
> The fundamental architectural contract of `pthread_cond_wait(&cond, &mutex)`:
> 1. It **atomically releases the associated mutex** and suspends the calling thread into the condition variable's wait queue in a single indivisible operation.
> 2. Upon being awakened via `pthread_cond_signal()` or `pthread_cond_broadcast()`, it **re-acquires the associated mutex** before returning to the caller.  
> **Key Nuance:** Atomicity prevents the fatal "Lost Wakeup Problem," where a signaling thread signals between the condition check and the wait call.

### Spurious Wakeup
> **Formal Definition:**  
> An event in which a thread waiting on a condition variable is awakened by the operating system kernel without any other thread having issued `pthread_cond_signal()` or `pthread_cond_broadcast()`. Mandates that condition variables **must always be evaluated inside a `while` loop**, never an `if` statement:
> ```c
> pthread_mutex_lock(&mutex);
> while (!condition_predicate) {
>     pthread_cond_wait(&cond, &mutex);
> }
> /* State is guaranteed true here */
> pthread_mutex_unlock(&mutex);
> ```

### POSIX Semaphore (`sem_t` / `<semaphore.h>`)
> **Formal Definition:**  
> A thread and process synchronization primitive defined by the POSIX Realtime Extensions (IEEE 1003.1b) consisting of an integer counter managed exclusively through atomic operations:
> - `sem_init(sem_t *sem, int pshared, unsigned int value)`: Initializes counter to `value`.
> - `sem_wait(sem_t *sem)`: Decrements counter. If counter $\le 0$, blocks calling thread.
> - `sem_post(sem_t *sem)`: Increments counter atomically, awakening one blocked waiter.
> - `sem_destroy(sem_t *sem)`: Destroys semaphore.

### Counting Semaphore vs. Binary Semaphore
> **Binary Semaphore:** A semaphore whose integer counter is restricted strictly to values `0` and `1`, functionally equivalent to a mutual exclusion lock (except it lacks ownership constraints).  
> **Counting Semaphore:** A semaphore whose integer counter represents a pool of available identical resources (e.g., slots in a bounded buffer, database connection pools), permitting multiple concurrent threads up to the integer limit.

### Read-Write Lock (`pthread_rwlock_t`)
> **Formal Definition:**  
> A concurrency control primitive that differentiates between read-only shared access and write-exclusive access. Permits multiple reader threads to hold the lock simultaneously (`pthread_rwlock_rdlock`), but restricts write access to exactly one exclusive writer thread (`pthread_rwlock_wrlock`), blocking all other readers and writers.

### POSIX Barrier (`pthread_barrier_t`)
> **Formal Definition:**  
> A multi-thread phase synchronization primitive initialized with a participant count $C$ (`pthread_barrier_init(&barrier, NULL, C)`). Any thread calling `pthread_barrier_wait(&barrier)` blocks until exactly $C$ distinct threads have called the wait function, at which instant all $C$ threads are released concurrently. Exactly one thread receives the return status `PTHREAD_BARRIER_SERIAL_THREAD`.

### Spinlock (`pthread_spinlock_t`)
> **Formal Definition:**  
> A mutual exclusion lock that continuously tests a shared memory flag in an active, tight execution loop (busy-waiting / polling) until the lock becomes available, rather than placing the waiting thread to sleep via an OS kernel context switch. Optimal exclusively for ultra-short critical sections on dedicated multiprocessor systems where lock wait time is less than the OS context-switching overhead ($\approx 2\text{ \textmu s}$).

---

## 5. Distributed Memory, MPI & Hybrid Programming (Lectures 24–32)

### Distributed-Memory Architecture
> **Formal Definition:**  
> A parallel computing architecture consisting of multiple independent autonomous compute nodes, where each node possesses its own dedicated private physical memory address space. No hardware shared memory exists across node boundaries; all inter-node communication and coordination must occur through explicit message-passing over a network interconnect.

### Message Passing Interface (MPI)
> **Formal Definition:**  
> A standardized, portable, library-based message-passing specification formulated by the MPI Forum that defines the syntax and semantics of point-to-point communication, collective communication, process topologies, and dynamic process management across distributed-memory systems.

### Communicator (`MPI_Comm`)
> **Formal Definition:**  
> An opaque MPI handle that encapsulates an ordered group of distinct processes combined with a secure communication context, ensuring that message traffic within one communicator cannot be intercepted or corrupted by message traffic in another communicator.

### `MPI_COMM_WORLD`
> **Formal Definition:**  
> The default predefined global communicator created automatically during `MPI_Init()`, encompassing all processes spawned by the MPI job launcher (`mpirun`) across the allocated nodes.

### Process Rank
> **Formal Definition:**  
> A contiguous zero-indexed unique integer ($0, 1, \dots, N-1$) identifying an individual MPI process within the context of a specific communicator of total size $N$. Determined via `MPI_Comm_rank(comm, &rank)`.

### Blocking Communication (`MPI_Send` / `MPI_Recv`)
> **Formal Definition:**  
> A communication mode where the calling function does not return control to the application until the communication operation completes and the user's buffer can be safely accessed and modified without data corruption:
> - `MPI_Send` returns when the application buffer can be safely overwritten (data copied to system buffer or delivered).
> - `MPI_Recv` returns when the incoming message has been completely written into the receiving application buffer.

### Non-Blocking Communication (`MPI_Isend` / `MPI_Irecv`)
> **Formal Definition:**  
> A communication mode that initiates a message transfer and immediately returns control to the calling program before the transfer completes, returning an opaque request handle (`MPI_Request`). Allows computation and network communication to proceed concurrently in the background (latency hiding). Requires explicit completion verification via `MPI_Wait()` or `MPI_Test()`.

### Request Handle (`MPI_Request`)
> **Formal Definition:**  
> An opaque tracking object returned by non-blocking communication routines (`MPI_Isend`, `MPI_Irecv`) representing an in-flight background communication transaction.

### `MPI_Wait` vs. `MPI_Test`
> **`MPI_Wait(&request, &status)`:**  
> A blocking synchronization call that halts the calling process until the non-blocking operation identified by `request` has completed. Deallocates the request handle.
> 
> **`MPI_Test(&request, &flag, &status)`:**  
> A non-blocking inspection call that queries whether the operation identified by `request` has finished. Returns immediately with `flag = 1` (completed) or `flag = 0` (still in-flight).

### Collective Communication
> **Formal Definition:**  
> A communication pattern that requires the coordinated participation of **every process** belonging to a communicator group. Failure of even a single process to invoke the collective call results in a global cluster deadlock. All collective operations in standard MPI are blocking (unless using non-blocking variants prefixed with `MPI_I...`).

### Broadcast (`MPI_Bcast`)
> **Formal Definition:**  
> A one-to-all collective communication operation in which a single designated root process replicates and transmits an identical buffer of data to every process in the communicator.

### Scatter (`MPI_Scatter`)
> **Formal Definition:**  
> A one-to-all collective data distribution operation where a root process takes a contiguous array, divides it into $N$ equal non-overlapping segments, and transmits segment $i$ exclusively to process rank $i$.

### Gather (`MPI_Gather`)
> **Formal Definition:**  
> An all-to-one collective data collection operation where each process in a communicator transmits an equal-sized local data buffer to a single root process, which concatenates the incoming segments in rank order into a contiguous destination array.

### Allgather (`MPI_Allgather`)
> **Formal Definition:**  
> An all-to-all collective operation functionally equivalent to an `MPI_Gather` followed immediately by an `MPI_Bcast`, whereby every process in the communicator contributes an equal chunk of local data, and every process receives the completely assembled global dataset.

### Collective Reduction (`MPI_Reduce`)
> **Formal Definition:**  
> An all-to-one collective operation that collects data elements from all processes in the communicator, combines them mathematically using an associative reduction operator (`MPI_SUM`, `MPI_PROD`, `MPI_MAX`, `MPI_MIN`), and deposits the final aggregate scalar result exclusively into the receive buffer of the designated root process.

### Allreduce (`MPI_Allreduce`)
> **Formal Definition:**  
> An all-to-all collective reduction operation where data from all processes is mathematically combined using an associative reduction operator, and the final reduced result is deposited into the receive buffers of **every process** in the communicator.

### `MPI_Barrier`
> **Formal Definition:**  
> A collective process synchronization primitive that blocks all calling processes in a communicator until every process in that communicator has invoked the routine. Exchanges zero bytes of user data; purely enforces temporal execution alignment.

### Dynamic Process Management (MPI Ports & Name Service)
> **Formal Definition:**  
> An MPI-2 feature allowing independent, separately launched MPI programs to establish network connections dynamically at runtime. The server opens an address port (`MPI_Open_port`), publishes its service name (`MPI_Publish_name`) to a shared daemon (`ompi-server`), and accepts connections (`MPI_Comm_accept`), while the client discovers the port (`MPI_Lookup_name`) and connects (`MPI_Comm_connect`), establishing an intercommunicator.

### Intercommunicator
> **Formal Definition:**  
> A specialized MPI communicator that binds two disjoint, non-overlapping process groups (e.g., a Client group and a Server group) to enable cross-group point-to-point and collective message passing, where ranks in communication calls reference the rank index in the remote group.

### Hybrid MPI + OpenMP Computing Model
> **Formal Definition:**  
> A hierarchical parallel programming paradigm that combines distributed-memory MPI across cluster nodes with shared-memory OpenMP multithreading across CPU cores within each node. Matches modern supercomputer hardware topology (clusters of multicore SMP/NUMA nodes).

### MPI Thread Safety Levels
> The four standardized levels of multithreading support requested via `MPI_Init_thread(argc, argv, required, &provided)`:
> 1. **`MPI_THREAD_SINGLE`:** The application executes strictly as a single thread. No multithreading permitted.
> 2. **`MPI_THREAD_FUNNELED`:** The application may spawn multiple OpenMP threads, but **only the master thread** (thread ID 0) is permitted to make MPI calls.
> 3. **`MPI_THREAD_SERIALIZED`:** Multiple OpenMP threads may make MPI calls, but only **one thread at a time** (calls must be serialized using critical sections or mutexes).
> 4. **`MPI_THREAD_MULTIPLE`:** Any OpenMP thread may make concurrent MPI calls simultaneously without application-level synchronization; the MPI runtime library internally guarantees full thread safety.

### Core Affinity (Thread Pinning)
> **Formal Definition:**  
> The operating system mechanism of binding an execution thread or process to a specific physical hardware CPU core or socket, preventing the OS scheduler from migrating threads between cores, preserving L1/L2 cache warmth and eliminating cross-socket NUMA memory latency penalties.

### `OMP_PROC_BIND`
> **Formal Definition:**  
> An OpenMP environment variable specifying the binding policy for worker threads to hardware places:
> - `close`: Binds threads to consecutive cores adjacent to the master thread (maximizes cache sharing).
> - `spread`: Distributes threads evenly across sockets/domains (maximizes memory bandwidth).
> - `false`: Disables binding, allowing the OS kernel to migrate threads freely.

### Core Over-Subscription
> **Formal Definition:**  
> An execution state occurring when the total number of active threads/processes launched on a compute node exceeds the number of physical hardware cores ($M \times T > C_{\text{phys}}$), forcing the operating system to perform frequent hardware context switches, leading to severe pipeline stalls and cache thrashing.

---
*End of Master Examination Glossary. All definitions meet university and competitive examination standards.*
