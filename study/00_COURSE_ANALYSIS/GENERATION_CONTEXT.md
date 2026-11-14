# Parallel Programming & High-Performance Computing (HPC)
## Master Generation Context & Subtopic Briefs

This document contains persistent, self-contained generation briefs for all 62 subtopic study files across the 13 modules of the Parallel Programming curriculum.

Before authoring any individual file, the model MUST read:
1. The subtopic's generation brief below.
2. `study/00_COURSE_ANALYSIS/WRITING_STANDARDS.md`.
3. The corresponding entry in `study/00_COURSE_ANALYSIS/SYLLABUS_COVERAGE_MATRIX.md`.
4. Prerequisite study files where appropriate.

---

# PART A: TILL MIDSEM (Lectures 1–16)

---

## Module 01: HPC Landscape & Foundations (Lecture 1)
*Directory:* `study/01_HPC_Foundations/`

### Brief 01.01: Introduction & HPC Landscape
- **File:** `study/01_HPC_Foundations/01_Introduction_and_HPC_Landscape.md`
- **Lecture Mapping:** Lecture 1
- **Key Concepts:** Course roadmap, evaluation scheme (EndSem 40%, Lab 40%, MidSem 20%), definition of High-Performance Computing, evolution from single-core clock scaling to multi-core/massively parallel systems, scientific/industrial/societal applications.
- **Pedagogical Goal:** Establish why parallelism is no longer optional. Introduce the course layout and grading breakdown.
- **Required Diagrams:** Single-core scaling plateau vs. multi-core era timeline; application domains taxonomy.
- **Required Exam Takeaways:** Definition of HPC, why clock frequency scaling stopped, core application areas.

### Brief 01.02: Seven Dwarfs of Numerical Methods
- **File:** `study/01_HPC_Foundations/02_Seven_Dwarfs_of_Numerical_Methods.md`
- **Lecture Mapping:** Lecture 1
- **Key Concepts:** The Berkeley "Seven Dwarfs" (Phil Colella, Krste Asanović et al.), definition of computational motifs, the 7 motifs: Dense Linear Algebra, Sparse Linear Algebra, Spectral Methods (FFT), N-Body Methods, Structured Grids, Unstructured Grids, Monte Carlo / MapReduce.
- **Pedagogical Goal:** Teach how algorithmic structure dictates hardware requirements and parallel communication patterns.
- **Required Comparisons:** Dense vs. Sparse Linear Algebra, Structured vs. Unstructured Grids.
- **Required Exam Takeaways:** Definitions and characteristics of all seven dwarfs with real-world examples.

### Brief 01.03: Parallel Hardware Hierarchy & TOP500
- **File:** `study/01_HPC_Foundations/03_Parallel_Hardware_Hierarchy_and_TOP500.md`
- **Lecture Mapping:** Lecture 1
- **Key Concepts:** Parallel hardware hierarchy (registers/ILP $\rightarrow$ SIMD vectors $\rightarrow$ multi-core sockets $\rightarrow$ shared-memory SMP/NUMA nodes $\rightarrow$ distributed clusters/supercomputers), the TOP500 list, High-Performance LINPACK (HPL), HPCG benchmark, Green500.
- **Pedagogical Goal:** Connect micro-architecture to macro-architecture. Explain how supercomputers are ranked and measured.
- **Required Diagrams:** Hardware hierarchy pyramid from single vector lane to multi-rack cluster.
- **Required Exam Takeaways:** Layers of hardware hierarchy, LINPACK vs HPCG benchmark, Green500 ranking metric.

### Brief 01.04: Performance Metrics, FLOPS & Power Cost
- **File:** `study/01_HPC_Foundations/04_Performance_Metrics_FLOPS_and_Power_Cost.md`
- **Lecture Mapping:** Lecture 1
- **Key Concepts:** Fundamental performance formula $P = \text{Work} / \text{Time}$, Floating-Point Operations Per Second (FLOP/s): MFLOP/s, GFLOP/s, TFLOP/s, PFLOP/s, EFLOP/s, peak theoretical FLOP/s vs sustained FLOP/s, the Power Wall, energy efficiency (FLOPS/Watt), power-cost case study (megawatt supercomputer operating expense).
- **Pedagogical Goal:** Mathematical formulation of compute throughput and economic/thermal physical limits.
- **Required Formulas:** $P = W/T$, $\text{Peak FLOPS} = \text{Sockets} \times \frac{\text{Cores}}{\text{Socket}} \times \text{Clock Frequency} \times \frac{\text{FLOPs}}{\text{Cycle}}$.
- **Required Solved Problems:** Calculating peak FLOPS of a modern multi-core processor; calculating annual electricity cost of a 15 MW supercomputer.

---

## Module 02: Architectural Foundations for Parallelism (Lecture 2)
*Directory:* `study/02_Architectural_Foundations/`

### Brief 02.01: Stored-Program Model & Pipelining
- **File:** `study/02_Architectural_Foundations/01_Stored_Program_Model_and_Pipelining.md`
- **Lecture Mapping:** Lecture 2
- **Key Concepts:** Von Neumann / Stored-program model, Fetch-Decode-Execute-Memory-Writeback cycle, instruction execution vs. data movement, von Neumann bottleneck, memory wall, instruction pipelining, pipeline stages, throughput vs latency, pipeline hazards (structural, data, control).
- **Pedagogical Goal:** Ground computing in hardware cycles and demonstrate why moving data dominates energy and time.
- **Required Diagrams:** Classical 5-stage RISC pipeline execution diagram showing cycle progression; pipeline hazard stall diagram.
- **Required Formulas:** Pipeline speedup $S_k = \frac{n \cdot k}{k + n - 1}$, ideal speedup $\lim_{n \to \infty} S_k = k$.

### Brief 02.02: ILP, Superscalar & SMT
- **File:** `study/02_Architectural_Foundations/02_ILP_Superscalar_and_SMT.md`
- **Lecture Mapping:** Lecture 2
- **Key Concepts:** Instruction-Level Parallelism (ILP), Bernstein's conditions for instruction independence, superscalar execution (multiple issue units, out-of-order execution, reservation stations, branch prediction), Simultaneous Multithreading (SMT / Hyper-Threading), logical cores vs physical cores.
- **Pedagogical Goal:** Show how hardware automatically extracts parallelism within a single thread, and where its limits lie.
- **Required Diagrams:** Single-issue vs. dual-issue superscalar pipeline; SMT sharing core execution units between two hardware threads.
- **Required Exam Takeaways:** ILP definition, Superscalar vs SMT distinction, when SMT helps vs when it degrades performance due to cache thrashing.

### Brief 02.03: SIMD Registers & Vector Extensions
- **File:** `study/02_Architectural_Foundations/03_SIMD_Registers_and_Vector_Extensions.md`
- **Lecture Mapping:** Lecture 2
- **Key Concepts:** Single Instruction, Multiple Data (SIMD) principle, vector registers, vector packing, x86 SIMD evolution: MMX (64-bit), SSE (128-bit, 4 single-precision floats), AVX/AVX2 (256-bit, 8 floats / 4 doubles), AVX-512 (512-bit, 16 floats / 8 doubles), vector arithmetic lanes.
- **Pedagogical Goal:** Teach data parallelism at the CPU register level before scaling to multi-threading.
- **Required Diagrams:** Scalar addition ($a+b=c$) vs. 4-lane SIMD vector addition ($a[0..3]+b[0..3]=c[0..3]$).
- **Required Table:** Evolution of vector register widths, data types, and peak throughput per cycle.

### Brief 02.04: Memory Hierarchy & Bandwidth
- **File:** `study/02_Architectural_Foundations/04_Memory_Hierarchy_and_Bandwidth.md`
- **Lecture Mapping:** Lecture 2
- **Key Concepts:** Memory hierarchy pyramid (registers, L1, L2, L3, DRAM), access latencies (CPU cycles and nanoseconds), cache lines (64 bytes), spatial locality, temporal locality, cache misses (compulsory, capacity, conflict), memory bandwidth vs. latency, bytes-per-flop ratio.
- **Pedagogical Goal:** Explain why CPU performance is frequently choked by memory bandwidth (the memory wall).
- **Required Diagrams:** Memory hierarchy pyramid annotated with latency, capacity, and bandwidth; cache line loading diagram.
- **Required Exam Takeaways:** Definitions of spatial/temporal locality, cache line structure, memory wall definition.

### Brief 02.05: Flynn's Taxonomy & Memory Architectures
- **File:** `study/02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md`
- **Lecture Mapping:** Lecture 2
- **Key Concepts:** Michael J. Flynn's classification (1972): SISD, SIMD, MISD, MIMD with clear hardware examples; Uniform Memory Access (UMA / SMP) vs. Cache-Coherent Non-Uniform Memory Access (ccNUMA); shared vs. distributed memory; preview of OpenMP as a shared-memory model.
- **Pedagogical Goal:** Provide the grand taxonomic classification of parallel computers.
- **Required Diagrams:** Flynn's 2x2 matrix diagrams; UMA bus-based architecture vs. ccNUMA multi-socket interconnect architecture.
- **Required Comparisons:** UMA vs ccNUMA table; Shared-memory vs Distributed-memory table.

---

## Module 03: Parallel Performance, Limits & Scaling (Lectures 3–5)
*Directory:* `study/03_Parallel_Performance_and_Scaling/`

### Brief 03.01: Speedup, Efficiency & Amdahl's Law
- **File:** `study/03_Parallel_Performance_and_Scaling/01_Speedup_Efficiency_and_Amdahls_Law.md`
- **Lecture Mapping:** Lectures 3–5
- **Key Concepts:** Execution time $T_1$ and $T_N$, Parallel Speedup $S(N) = T_1 / T_N$, Parallel Efficiency $\epsilon(N) = S(N)/N$, linear vs sub-linear vs super-linear speedup (cache effect cause), Gene M. Amdahl's Law (1967), strong scaling (fixed problem size), derivation of $S(N) = \frac{1}{s + \frac{1-s}{N}}$, asymptotic limit $S_{\max} = 1/s$.
- **Pedagogical Goal:** Rigorous mathematical derivation and intuition for why sequential code places a hard ceiling on speedup.
- **Required Formulas:** $S(N)$, $\epsilon(N)$, Amdahl's Law, asymptotic limit $S_{\max}$.
- **Required Solved Problems:** At least 3 detailed numerical problems (calculating speedup, efficiency, and maximum speedup for varying sequential fractions).

### Brief 03.02: Gustafson's Law & Weak Scaling
- **File:** `study/03_Parallel_Performance_and_Scaling/02_Gustafsons_Law_and_Weak_Scaling.md`
- **Lecture Mapping:** Lectures 3–5
- **Key Concepts:** John L. Gustafson's Law (1988), re-evaluating Amdahl's Law, fixed-time vs fixed-size models, scaled speedup derivation: $S(N) = s' + N(1-s') = N - s'(N-1)$, weak scaling (workload expands proportionally with processors).
- **Pedagogical Goal:** Show that massively parallel supercomputing is justified by solving larger problems, not just running small problems faster.
- **Required Comparisons:** Comprehensive table: Amdahl's Law (Strong Scaling) vs. Gustafson's Law (Weak Scaling).
- **Required Solved Problems:** 2 numerical problems comparing Amdahl speedup vs Gustafson scaled speedup on 64 and 1024 processors.

### Brief 03.03: Parallel Overhead & Load Imbalance
- **File:** `study/03_Parallel_Performance_and_Scaling/03_Parallel_Overhead_and_Load_Imbalance.md`
- **Lecture Mapping:** Lectures 3–5
- **Key Concepts:** Parallel overhead function $c(N)$, sources of overhead (thread creation, communication, synchronization, scheduling), modified execution time $T(N) = s T_1 + \frac{(1-s)T_1}{N} + c(N)$, degradation of speedup beyond optimal processor count, load imbalance (definition, causes, quantification, impact on execution time determined by slowest thread).
- **Pedagogical Goal:** Explain why real performance curves peak and then degrade when core count becomes excessive.
- **Required Diagrams:** Real-world speedup curve showing peak and degradation due to overhead $c(N)$; thread execution timeline showing load imbalance.
- **Required Formulas:** Extended Amdahl speedup with overhead $c(N)$.

### Brief 03.04: Resource Bottleneck Models
- **File:** `study/03_Parallel_Performance_and_Scaling/04_Resource_Bottleneck_Models.md`
- **Lecture Mapping:** Lectures 3–5
- **Key Concepts:** The three primary physical resource bottlenecks: Compute bottlenecks (ALU throughput limit), Memory-bandwidth bottlenecks (DRAM bus saturation), Communication bottlenecks (network latency/bandwidth). Introduction to the Roofline Model: Operational / Arithmetic Intensity (FLOPs/byte), compute-bound vs memory-bound regions.
- **Pedagogical Goal:** Enable students to identify which physical subsystem is throttling a parallel application.
- **Required Diagrams:** Roofline model chart showing memory-bandwidth diagonal and peak-compute horizontal ceiling.
- **Required Formulas:** $\text{Arithmetic Intensity} = \frac{\text{Total Floating-Point Operations}}{\text{Total Bytes Transferred from DRAM}}$.

### Brief 03.05: Benchmarking Methodology & Pitfalls
- **File:** `study/03_Parallel_Performance_and_Scaling/05_Benchmarking_Methodology_and_Pitfalls.md`
- **Lecture Mapping:** Lectures 3–5
- **Key Concepts:** Benchmarking principles, wall-clock time vs CPU time, precision timers (`omp_get_wtime()`, `clock_gettime(CLOCK_MONOTONIC)`), timer resolution, warmup iterations, cold caches vs warm caches, system jitter, Intel Turbo Boost and CPU frequency governor effects, statistical significance (mean, median, standard deviation).
- **Pedagogical Goal:** Train students to measure parallel performance scientifically without falling for measurement artifacts.
- **Required Code:** C benchmarking harness measuring execution time over repeated trials with warmup and timing statistics.

---

## Module 04: OpenMP Fundamentals & Data Scoping (Lectures 6–7)
*Directory:* `study/04_OpenMP_Fundamentals/`

### Brief 04.01: OpenMP Architecture & Fork-Join Model
- **File:** `study/04_OpenMP_Fundamentals/01_OpenMP_Architecture_and_Fork_Join_Model.md`
- **Lecture Mapping:** Lectures 6–7
- **Key Concepts:** OpenMP definition, shared-memory programming model, three-tier architecture (Compiler directives, Runtime library routines, Environment variables), Fork-Join execution model: master thread, worker thread team, parallel region, barrier join.
- **Pedagogical Goal:** Establish the mental model of how OpenMP interacts with the OS thread subsystem.
- **Required Diagrams:** Fork-join execution model diagram showing master thread splitting into worker threads and rejoining.
- **Required Exam Takeaways:** Precise OpenMP definition, three components of OpenMP, fork-join model mechanics.

### Brief 04.02: Pragma Syntax, Parallel Regions & Compilation
- **File:** `study/04_OpenMP_Fundamentals/02_Pragma_Syntax_Parallel_Regions_and_Compilation.md`
- **Lecture Mapping:** Lectures 6–7
- **Key Concepts:** `#pragma omp directive-name [clauses]`, `#pragma omp parallel`, controlling threads (`num_threads` clause, `omp_set_num_threads()`, `OMP_NUM_THREADS`), core functions: `omp_get_thread_num()`, `omp_get_num_threads()`, compilation with GCC (`-fopenmp`) and Intel (`-qopenmp`), execution non-determinism (why thread print statements interleave unpredictably).
- **Pedagogical Goal:** Get students to write, compile, run, and understand their first parallel OpenMP program.
- **Required Code:** Complete C program creating a parallel region, printing thread IDs, demonstrating non-deterministic output. Include exact compile/run commands.

### Brief 04.03: Data Scoping: Shared, Private & default(none)
- **File:** `study/04_OpenMP_Fundamentals/03_Data_Scoping_Shared_Private_and_Default_None.md`
- **Lecture Mapping:** Lectures 6–7
- **Key Concepts:** The shared-memory data visibility challenge, `shared(var)` clause (common memory address, race risks), `private(var)` clause (uninitialized thread-local copy on thread stack, out-of-scope upon exit), default scoping rules in OpenMP, why `default(none)` is the industry-standard defensive programming best practice.
- **Pedagogical Goal:** Prevent the #1 source of OpenMP bugs: accidental data sharing.
- **Required Diagrams:** Memory layout showing shared variables in heap/data segment vs private variables on individual thread stacks.
- **Required Code:** C program demonstrating the difference between shared and private variables, and compilation failure under `default(none)`.

---

## Module 05: OpenMP Work-Sharing & Scheduling (Lectures 8–10)
*Directory:* `study/05_OpenMP_Work_Sharing_and_Scheduling/`

### Brief 05.01: Work-Sharing: For, Sections & Single
- **File:** `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md`
- **Lecture Mapping:** Lecture 8
- **Key Concepts:** Concept of work-sharing (distributing iterations/code blocks across an existing thread team without forking), `#pragma omp for` (canonical loop requirements), `#pragma omp sections` and `#pragma omp section` (functional task distribution), `#pragma omp single` (single thread execution with implicit barrier) vs `#pragma omp master` (master thread only, no barrier).
- **Pedagogical Goal:** Distinguish creating threads (`parallel`) from assigning work to threads (`for`, `sections`, `single`).
- **Required Diagrams:** Iteration distribution across 4 threads; `sections` task distribution diagram.
- **Required Code:** Complete C programs for `#pragma omp for`, `#pragma omp sections`, and `#pragma omp single`.

### Brief 05.02: Firstprivate Clause & Initialization
- **File:** `study/05_OpenMP_Work_Sharing_and_Scheduling/02_Firstprivate_Clause_and_Initialization.md`
- **Lecture Mapping:** Lectures 9–10
- **Key Concepts:** Semantic limitation of `private` (values start uninitialized, leaving garbage data), `firstprivate` clause: creating thread-local copies initialized with the master thread's value prior to region entry, C++ copy-constructor invocation, data lifecycle.
- **Pedagogical Goal:** Master variable initialization semantics in thread-local storage.
- **Required Comparisons:** Side-by-side table: `shared` vs `private` vs `firstprivate`.
- **Required Code:** C code showing garbage values with `private` vs correct initialization with `firstprivate`.

### Brief 05.03: Loop Scheduling Strategies
- **File:** `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md`
- **Lecture Mapping:** Lectures 9–10
- **Key Concepts:** Why loop scheduling matters, static scheduling (`schedule(static, chunk)`: compile-time deterministic round-robin block mapping), dynamic scheduling (`schedule(dynamic, chunk)`: runtime task queue, FCFS), guided scheduling (`schedule(guided, chunk)`: dynamically decreasing chunks proportional to unassigned iterations / threads), auto scheduling (`schedule(auto)`).
- **Pedagogical Goal:** Mathematical and mechanical understanding of how iterations are assigned to threads under each policy.
- **Required Diagrams:** Visual timeline of iteration chunk assignment across 4 threads for static, dynamic, and guided schedules.
- **Required Math:** Iteration chunk assignment formulas for static and guided policies.

### Brief 05.04: Scheduling Trade-Offs & Performance
- **File:** `study/05_OpenMP_Work_Sharing_and_Scheduling/04_Scheduling_Tradeoffs_and_Performance.md`
- **Lecture Mapping:** Lectures 9–10
- **Key Concepts:** Performance trade-offs: Scheduling overhead vs. Load balance, cache locality and cache thrashing across chunks, impact of chunk size (too small = excessive queue contention, too large = load imbalance), decision framework for selecting scheduling policy based on loop iteration cost profile (uniform, linear, irregular).
- **Pedagogical Goal:** Enable students to select the optimal schedule for any given computational loop.
- **Required Code:** Benchmark program timing a loop with non-uniform iteration costs under static vs dynamic vs guided schedules.

---

## Module 06: OpenMP Synchronization & Worked Examples (Lectures 11–12)
*Directory:* `study/06_OpenMP_Synchronization/`

### Brief 06.01: Race Conditions, Critical & Atomic
- **File:** `study/06_OpenMP_Synchronization/01_Race_Conditions_Critical_and_Atomic.md`
- **Lecture Mapping:** Lectures 11–12
- **Key Concepts:** Anatomy of a race condition at machine-code level (Load $\rightarrow$ Modify $\rightarrow$ Store register interleaving), `#pragma omp critical [name]` (mutual exclusion structured block, lock overhead), `#pragma omp atomic [read|write|update|capture]` (hardware-level atomic operations, bus locking, assembly instructions like `lock add`).
- **Pedagogical Goal:** Master mutual exclusion and understand hardware atomics vs OS/runtime locks.
- **Required Diagrams:** Assembly register interleaving diagram showing lost updates; critical vs atomic execution timeline.
- **Required Comparisons:** Detailed Markdown table: `critical` vs `atomic`.

### Brief 06.02: Barriers, Nowait & Ordered Execution
- **File:** `study/06_OpenMP_Synchronization/02_Barriers_Nowait_and_Ordered_Execution.md`
- **Lecture Mapping:** Lecture 11
- **Key Concepts:** Explicit barrier `#pragma omp barrier`, implicit barriers at the end of work-sharing constructs (`parallel`, `for`, `sections`, `single`), the `nowait` clause (removing implicit barriers to allow asynchronous pipelining, data dependency hazard analysis), `#pragma omp ordered` (forcing sequential order within a parallel loop).
- **Pedagogical Goal:** Master thread coordination, barrier removal for performance, and selective ordering.
- **Required Diagrams:** Barrier synchronization diagram showing threads arriving at different times and releasing together; `nowait` pipelining diagram.
- **Required Code:** C code demonstrating `single nowait` and an `#pragma omp ordered` loop.

### Brief 06.03: Low-Level Locks: omp_lock_t
- **File:** `study/06_OpenMP_Synchronization/03_Low_Level_Locks_omp_lock_t.md`
- **Lecture Mapping:** Lecture 11
- **Key Concepts:** Low-level OpenMP lock data type `omp_lock_t`, complete lock API: `omp_init_lock()`, `omp_set_lock()`, `omp_test_lock()`, `omp_unset_lock()`, `omp_destroy_lock()`, nested locks (`omp_nest_lock_t`), when locks are preferred over `#pragma omp critical` (fine-grained locking, dynamic data structures, hash tables).
- **Pedagogical Goal:** Transition from coarse directive-based synchronization to fine-grained programmatic locks.
- **Required Code:** Complete C program implementing a thread-safe concurrent counter / bank account using `omp_lock_t`.

### Brief 06.04: Worked Examples: Manual Local-Sum & False Sharing
- **File:** `study/06_OpenMP_Synchronization/04_Worked_Examples_Manual_Local_Sum_and_False_Sharing.md`
- **Lecture Mapping:** Lecture 12
- **Key Concepts:** Comprehensive worked study: Parallel Sum of 100M elements. Naive sum (race condition) $\rightarrow$ Atomic sum (serialization bottleneck) $\rightarrow$ Critical sum (lock contention) $\rightarrow$ Manual local-sum array `sum[tid]` $\rightarrow$ The False Sharing crisis (multiple cores modifying adjacent elements residing in the same 64-byte cache line) $\rightarrow$ Fixing false sharing with cache-line padding.
- **Pedagogical Goal:** The classic HPC milestone lesson: demonstrate how hardware cache coherence mechanics can destroy parallel speedup.
- **Required Diagrams:** False sharing diagram showing two CPU cores fighting over a single 64-byte L1 cache line.
- **Required Code:** Complete C benchmark comparing all 5 implementations with exact execution times and speedups.

### Brief 06.05: Reduction Clause & Mathematical Properties
- **File:** `study/06_OpenMP_Synchronization/05_Reduction_Clause_and_Mathematical_Properties.md`
- **Lecture Mapping:** Lecture 12
- **Key Concepts:** The OpenMP `reduction(operator : variable_list)` clause, syntax, automatic allocation of private accumulators initialized to operator identities (e.g. 0 for `+`, 1 for `*`), automatic $O(\log P)$ tree reduction at barrier, valid operators, mathematical requirements: Associativity and Commutativity, floating-point non-associativity and numerical drift in parallel sums.
- **Pedagogical Goal:** Teach the optimal, idiomatic solution for parallel reductions and its mathematical foundations.
- **Required Diagrams:** Logarithmic tree reduction across 8 threads ($O(\log_2 P)$ steps).
- **Required Code:** Complete, clean C program for array reduction and numerical $\pi$ calculation using OpenMP reduction.

---

## Module 07: OpenMP Tasks & SIMD Vectorization (Lectures 13–16)
*Directory:* `study/07_OpenMP_Tasks_and_SIMD/`

### Brief 07.01: Task Parallelism vs. Data Parallelism
- **File:** `study/07_OpenMP_Tasks_and_SIMD/01_Task_Parallelism_vs_Data_Parallelism.md`
- **Lecture Mapping:** Lectures 13–14
- **Key Concepts:** Data (loop) parallelism vs. Task parallelism, limitations of `#pragma omp for` (requires known loop trip count, cannot handle pointer chasing, linked lists, recursive tree traversal, or dynamic graphs), the OpenMP task model: explicit task generation, task pools, work stealing runtime.
- **Pedagogical Goal:** Broaden student thinking beyond loops into irregular and dynamic parallel algorithms.
- **Required Comparisons:** Loop Parallelism vs. Task Parallelism comparison table.
- **Required Diagrams:** Task pool / queue and worker threads stealing tasks.

### Brief 07.02: OpenMP Task & Taskloop Constructs
- **File:** `study/07_OpenMP_Tasks_and_SIMD/02_OpenMP_Task_and_Taskloop_Constructs.md`
- **Lecture Mapping:** Lectures 13–14
- **Key Concepts:** `#pragma omp task`, task synchronization with `#pragma omp taskwait`, recursive Fibonacci example (and task cutoff optimization to prevent task creation overhead explosion), `#pragma omp taskloop`, controlling granularity: `grainsize(val)` vs `num_tasks(val)`, task distribution across thread team.
- **Pedagogical Goal:** Hands-on task authoring with recursion and loop chunking.
- **Required Code:** Complete C recursive Fibonacci using `#pragma omp task` with base-case cutoff threshold.
- **Required Code:** Complete C program demonstrating `#pragma omp taskloop grainsize(100)`.

### Brief 07.03: SIMD Vectorization Basics & Pragma simd
- **File:** `study/07_OpenMP_Tasks_and_SIMD/03_SIMD_Vectorization_Basics_and_Pragma_simd.md`
- **Lecture Mapping:** Lecture 15
- **Key Concepts:** OpenMP 4.0+ SIMD execution model, difference between multi-threading (multiple cores) and vectorization (single core, wide registers), `#pragma omp simd`, strip-mining loop iterations into vector chunks, combined directive `#pragma omp parallel for simd`.
- **Pedagogical Goal:** Bridge high-level OpenMP with the CPU's hardware SIMD vector units.
- **Required Diagrams:** Loop execution: Scalar sequential vs. OpenMP multi-threaded vs. OpenMP SIMD vs. Combined Parallel SIMD.
- **Required Code:** C vector addition program with `#pragma omp simd`.

### Brief 07.04: Compiler Vectorization Reports & Flags
- **File:** `study/07_OpenMP_Tasks_and_SIMD/04_Compiler_Vectorization_Reports_and_Flags.md`
- **Lecture Mapping:** Lecture 15
- **Key Concepts:** Compiler autovectorization vs directive-driven vectorization, GCC flags: `-O3`, `-march=native`, `-fopt-info-vec`, `-fopt-info-vec-missed`, `-fopt-info-vec-all`, Intel compiler flags (`-qopt-report`), vectorization inhibitors: loop-carried dependencies, pointer aliasing (`restrict` keyword), complex branching, function calls inside loops.
- **Pedagogical Goal:** Teach students how to read compiler optimization diagnostics to verify that vectorization actually occurred.
- **Required Output:** Real GCC vectorization report outputs showing vectorized loops vs loops not vectorized due to aliasing/dependencies.

### Brief 07.05: SIMD Scoping: Private & Latent Bugs
- **File:** `study/07_OpenMP_Tasks_and_SIMD/05_SIMD_Scoping_Private_and_Latent_Bugs.md`
- **Lecture Mapping:** Lecture 16
- **Key Concepts:** Data scoping in SIMD loops: `private`, `firstprivate`, `linear`, why missing scoping in `#pragma omp simd` creates subtle, silent calculation corruptions (multiple vector lanes overwriting temporary scalar variables), the `linear(var : step)` clause for induction variables.
- **Pedagogical Goal:** Master data scoping within SIMD loops and debug latent vectorization bugs.
- **Required Diagrams:** Vector lanes overwriting a shared temporary scalar register.
- **Required Code:** Example showing buggy un-scoped SIMD code vs correctly scoped SIMD code with `private()` and `linear()`.

### Brief 07.06: SIMD Reduction, Alignment & Warmup
- **File:** `study/07_OpenMP_Tasks_and_SIMD/06_SIMD_Reduction_Alignment_and_Warmup.md`
- **Lecture Mapping:** Lecture 16
- **Key Concepts:** SIMD reduction (`reduction(+:sum)` in `#pragma omp simd`), memory alignment boundaries (16-byte SSE, 32-byte AVX, 64-byte AVX-512), cache-line alignment effects (unaligned memory access penalty, cache line splits), the `aligned(ptr : bytes)` clause, `posix_memalign()` / `aligned_alloc()`, benchmark warmup for vector code (AVX frequency downclocking, power states).
- **Pedagogical Goal:** Attain peak vector performance through aligned memory allocation and proper benchmarking protocol.
- **Required Code:** Complete C program allocating 64-byte aligned memory and executing an aligned SIMD reduction loop.

---

# PART B: AFTER MIDSEM (Lectures 17–32)

---

## Module 08: POSIX & Pthreads Foundations (Lectures 17–19)
*Directory:* `study/08_POSIX_and_Pthreads/`

### Brief 08.01: POSIX Standard & Pthreads Overview
- **File:** `study/08_POSIX_and_Pthreads/01_POSIX_Standard_and_Pthreads_Overview.md`
- **Lecture Mapping:** Lecture 17
- **Key Concepts:** What POSIX is (Portable Operating System Interface, IEEE 1003), why portability matters across Unix variants, certified systems (macOS, AIX, HP-UX, Solaris, QNX) vs mostly compliant systems (Linux, FreeBSD, Android), Pthreads vs OpenMP (manual low-level thread control vs pragma directives), programmer responsibilities (thread lifecycle, stack management, synchronization, race prevention).
- **Pedagogical Goal:** Establish why Pthreads is the foundational threading API of modern computing.
- **Required Comparisons:** Comprehensive Markdown table: Pthreads vs OpenMP.
- **Required Exam Takeaways:** Definition of POSIX, certified vs compliant systems, manual vs declarative threading models.

### Brief 08.02: Thread Creation: pthread_create & Handles
- **File:** `study/08_POSIX_and_Pthreads/02_Thread_Creation_pthread_create_and_Handles.md`
- **Lecture Mapping:** Lecture 18
- **Key Concepts:** `pthread_t` opaque platform-dependent handle, complete signature:
  `int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);`
  detailed parameter semantics, return code checking (0 on success, error number on failure), the `void*` universal pointer pattern.
- **Pedagogical Goal:** Write and understand the fundamental Pthreads creation function with proper error handling.
- **Required Code:** Complete C program creating 4 threads printing "Hello World", with compilation command (`gcc -pthread`).

### Brief 08.03: Thread Attributes & Scheduling Policies
- **File:** `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md`
- **Lecture Mapping:** Lecture 18
- **Key Concepts:** `pthread_attr_t` lifecycle (`pthread_attr_init`, `pthread_attr_destroy`), configuring thread stack size (`pthread_attr_setstacksize`), detach state (`PTHREAD_CREATE_JOINABLE` vs `PTHREAD_CREATE_DETACHED`), scheduling policies (`SCHED_OTHER` normal time-sharing, `SCHED_FIFO` real-time FIFO, `SCHED_RR` real-time round-robin), real-time thread priority (`struct sched_param`).
- **Pedagogical Goal:** Master low-level OS thread configuration and scheduling mechanisms.
- **Required Code:** C program configuring custom 2MB stack size and detached state using `pthread_attr_t`.

### Brief 08.04: Joining, Detaching & Resource Reclamation
- **File:** `study/08_POSIX_and_Pthreads/04_Joining_Detaching_and_Resource_Reclamation.md`
- **Lecture Mapping:** Lecture 19
- **Key Concepts:** `pthread_join(pthread_t thread, void **retval)`: blocking until target thread terminates, extracting exit status, `pthread_exit(void *retval)` vs returning from start routine, joinable vs detached threads (`pthread_detach`), resource reclamation: what happens to thread stack and kernel descriptors upon exit, thread resource leaks (zombie threads in long-running servers).
- **Pedagogical Goal:** Understand thread termination, return value extraction, and OS resource cleanup.
- **Required Diagrams:** Timeline showing main thread blocking in `pthread_join` until worker thread completes.
- **Required Code:** Complete C program passing and retrieving dynamically allocated return values via `pthread_join`.

### Brief 08.05: Argument Passing Patterns & Pitfalls
- **File:** `study/08_POSIX_and_Pthreads/05_Argument_Passing_Patterns_and_Pitfalls.md`
- **Lecture Mapping:** Lecture 19
- **Key Concepts:** Passing distinct arguments to $N$ threads, packing multiple arguments into structs, heap vs stack struct allocation, The Fatal Loop Index Reference Bug (passing `&i` to `pthread_create`), memory lifecycle safety.
- **Pedagogical Goal:** Prevent the most frequent Pthreads bug in student and industry code.
- **Required Code:** Side-by-side comparison: Buggy code passing `&i` (with visual race explanation) vs Correct code passing per-thread allocated structs or arrays of structs.

---

## Module 09: Pthreads Synchronization Primitives (Lectures 20–23)
*Directory:* `study/09_Pthreads_Synchronization/`

### Brief 09.01: Race Conditions & Mutexes
- **File:** `study/09_Pthreads_Synchronization/01_Race_Conditions_and_Mutexes.md`
- **Lecture Mapping:** Lecture 20
- **Key Concepts:** Synchronization toolbox overview, race conditions on shared memory, Mutex (`pthread_mutex_t`), initialization (`PTHREAD_MUTEX_INITIALIZER` vs `pthread_mutex_init`), locking (`pthread_mutex_lock`), non-blocking lock (`pthread_mutex_trylock`), unlocking (`pthread_mutex_unlock`), destruction (`pthread_mutex_destroy`), protecting a shared counter.
- **Pedagogical Goal:** Master mutual exclusion under Pthreads.
- **Required Code:** Complete, robust C program running $N$ threads incrementing a shared counter with mutex protection.

### Brief 09.02: Condition Variables & Coordination
- **File:** `study/09_Pthreads_Synchronization/02_Condition_Variables_and_Coordination.md`
- **Lecture Mapping:** Lecture 21
- **Key Concepts:** Condition Variables (`pthread_cond_t`), `pthread_cond_wait`, `pthread_cond_signal`, `pthread_cond_broadcast`, `pthread_cond_destroy`, mechanics of `pthread_cond_wait`: atomically releases associated mutex and blocks, re-acquires mutex before returning, why CVs MUST always be paired with a mutex, The mandatory `while (!condition)` loop pattern: why `if` is a fatal bug (spurious wakeups, thread race upon wakeup).
- **Pedagogical Goal:** Master state-change coordination and signaling between threads without busy waiting.
- **Required Diagrams:** Step-by-step state transition of `pthread_cond_wait` (Mutex unlock $\rightarrow$ Sleep on CV $\rightarrow$ Woken by signal $\rightarrow$ Mutex re-lock $\rightarrow$ Resume).
- **Required Code:** Complete C program implementing thread coordination using mutex + condition variable.

### Brief 09.03: Producer–Consumer & POSIX Semaphores
- **File:** `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md`
- **Lecture Mapping:** Lecture 22
- **Key Concepts:** Bounded-Buffer / Producer-Consumer problem, buffer full and buffer empty conditions, blocking without busy waiting, POSIX Semaphores: `sem_t`, `<semaphore.h>`, POSIX Realtime Extensions (IEEE 1003.1b), `sem_init`, `sem_wait`, `sem_post`, `sem_destroy`, counting semaphores vs binary semaphores, comparing Mutex+CV solution vs Semaphore solution.
- **Pedagogical Goal:** Solve the classical concurrency synchronization problem using semaphores.
- **Required Diagrams:** Circular bounded buffer with producer pointer, consumer pointer, and full/empty slots.
- **Required Code:** Complete C implementation of circular bounded buffer with multiple producers and consumers using `sem_t`.

### Brief 09.04: Read-Write Locks & Read-Heavy Workloads
- **File:** `study/09_Pthreads_Synchronization/04_Read_Write_Locks_and_Read_Heavy_Workloads.md`
- **Lecture Mapping:** Lecture 23
- **Key Concepts:** Read-Write Locks (`pthread_rwlock_t`), concurrency semantics: multiple concurrent readers OR single exclusive writer, API: `pthread_rwlock_init`, `pthread_rwlock_rdlock`, `pthread_rwlock_wrlock`, `pthread_rwlock_unlock`, `pthread_rwlock_destroy`, performance in read-heavy vs write-heavy workloads, writer starvation and fairness.
- **Pedagogical Goal:** Apply specialized locks for asymmetric read/write access patterns.
- **Required Code:** Complete C benchmark comparing throughput of mutex vs read-write lock under a 95% read / 5% write workload.

### Brief 09.05: Barriers & Phase Synchronization
- **File:** `study/09_Pthreads_Synchronization/05_Barriers_and_Phase_Synchronization.md`
- **Lecture Mapping:** Lecture 23
- **Key Concepts:** POSIX Barriers: `pthread_barrier_t`, API: `pthread_barrier_init`, `pthread_barrier_wait`, `pthread_barrier_destroy`, synchronizing parallel phases, `PTHREAD_BARRIER_SERIAL_THREAD` return code for executing master serial tasks between phases.
- **Pedagogical Goal:** Coordinate multi-phase iterative algorithms across threads.
- **Required Code:** Complete C program implementing a 2-phase iterative stencil calculation synchronized with `pthread_barrier_t`.

### Brief 09.06: Spinlocks, Busy-Waiting & Selection Guide
- **File:** `study/09_Pthreads_Synchronization/06_Spinlocks_Busy_Waiting_and_Selection_Guide.md`
- **Lecture Mapping:** Lecture 23
- **Key Concepts:** Spinlocks (`pthread_spinlock_t`), API: `pthread_spin_init`, `pthread_spin_lock`, `pthread_spin_trylock`, `pthread_spin_unlock`, `pthread_spin_destroy`, scoping: `PTHREAD_PROCESS_PRIVATE` vs `PTHREAD_PROCESS_SHARED`, busy waiting vs OS blocking/context switching (~1–5 $\mu$s cost), when spinlocks outperform mutexes, Comprehensive Decision Framework for choosing the right primitive.
- **Pedagogical Goal:** Deep understanding of hardware spin vs OS sleeping, and mastering primitive selection.
- **Required Decision Flowchart / Table:** Comprehensive selection guide across Mutex, CondVar, Semaphore, RW-Lock, Barrier, and Spinlock.

---

## Module 10: MPI Fundamentals & Environment (Lectures 24–25)
*Directory:* `study/10_MPI_Fundamentals/`

### Brief 10.01: Distributed Memory & Message Passing Model
- **File:** `study/10_MPI_Fundamentals/01_Distributed_Memory_and_Message_Passing_Model.md`
- **Lecture Mapping:** Lecture 24
- **Key Concepts:** Distributed-memory computer systems, compute nodes, network interconnects, private address space per process (no shared pointers, no direct memory access), the Message-Passing paradigm, MPI standard history and scope, MPI vs OpenMP (separate vs shared address spaces, multi-machine scalability, communication cost vs memory contention).
- **Pedagogical Goal:** Transition students from shared-memory multi-threading to distributed-memory multi-processing.
- **Required Comparisons:** Master comparison table: OpenMP vs Pthreads vs MPI.
- **Required Diagrams:** Distributed memory cluster showing private memories and message-passing network.

### Brief 10.02: Structure of an MPI Program
- **File:** `study/10_MPI_Fundamentals/02_Structure_of_an_MPI_Program.md`
- **Lecture Mapping:** Lectures 24–25
- **Key Concepts:** Single Program, Multiple Data (SPMD) model, MPI program lifecycle: `MPI_Init(&argc, &argv)` (details: called once by main thread, argument passing), `MPI_COMM_WORLD` communicator, `MPI_Comm_rank` (process ID), `MPI_Comm_size` (total process count), `MPI_Finalize`, rank-based conditional branching (`if (rank == 0)`).
- **Pedagogical Goal:** Master the foundational structure and execution model of an MPI program.
- **Required Code:** Complete, standard MPI C program with proper rank and size querying and graceful exit.

### Brief 10.03: Compiling & Launching MPI Programs
- **File:** `study/10_MPI_Fundamentals/03_Compiling_and_Launching_MPI_Programs.md`
- **Lecture Mapping:** Lecture 25
- **Key Concepts:** The compiler wrapper: `mpicc`, `mpicxx` (flags and libraries linked behind the scenes), process launcher: `mpirun` / `mpiexec`, launch arguments: `-np <N>` (number of processes), `--oversubscribe` (running more processes than physical CPU cores), `--hostfile <file>` (multi-node execution), `--display-allocation` (process-to-core mapping).
- **Pedagogical Goal:** Practical lab mastery of compiling, launching, and controlling MPI processes on local and clustered systems.
- **Required Commands:** Step-by-step terminal commands for local compilation, single-machine launch, oversubscribed launch, and hostfile cluster launch.

### Brief 10.04: Communication Paradigms Overview
- **File:** `study/10_MPI_Fundamentals/04_Communication_Paradigms_Overview.md`
- **Lecture Mapping:** Lecture 25
- **Key Concepts:** Overview of point-to-point vs collective communication, point-to-point: one-to-one communication between two specific ranks, collective: group communication across all processes in a communicator, communication channels, envelopes, buffers, network protocol overview.
- **Pedagogical Goal:** Provide the high-level roadmap of MPI communication modes before diving into APIs.
- **Required Diagrams:** High-level schematic contrasting Point-to-Point vs Collective communication patterns.

---

## Module 11: MPI Point-to-Point & Dynamic Processes (Lectures 26–29)
*Directory:* `study/11_MPI_Communication/`

### Brief 11.01: Blocking Point-to-Point: Send & Recv
- **File:** `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md`
- **Lecture Mapping:** Lecture 26
- **Key Concepts:** Sender and receiver coordination, message arguments / envelope: Buffer, Count, Datatype, Rank (destination/source), Tag, Communicator; standard MPI datatypes (`MPI_INT`, `MPI_DOUBLE`, etc.); using tags to distinguish message semantics; `MPI_Send` and `MPI_Recv`; what "blocking" actually means (buffer safety vs receiver delivery); Eager Protocol (internal buffering) vs Rendezvous Protocol (handshake synchronization); deadlocks in blocking communication (opposite send/receive order); fixing deadlocks with `MPI_Sendrecv`.
- **Pedagogical Goal:** Master blocking message passing, protocols, and deadlock avoidance.
- **Required Diagrams:** Message envelope diagram; Eager protocol vs Rendezvous protocol timeline; Deadlock cycle diagram.
- **Required Code:** Complete C program exchanging messages between rank 0 and rank 1 using `MPI_Send` and `MPI_Recv`.

### Brief 11.02: Non-Blocking Communication: Isend & Irecv
- **File:** `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md`
- **Lecture Mapping:** Lecture 27
- **Key Concepts:** Non-blocking communication motivation: latency hiding and deadlock prevention; `MPI_Isend`, `MPI_Irecv`; request handles (`MPI_Request`); overlapping computation with communication; completion functions: `MPI_Wait` (blocking wait for completion) and `MPI_Test` (non-blocking status check); `MPI_Waitall`; safe buffer reuse rules (when is it safe to read/modify send and receive buffers).
- **Pedagogical Goal:** Implement high-performance communication-computation overlap.
- **Required Diagrams:** Timeline contrasting blocking communication (idle CPU during transfer) vs non-blocking communication (computation overlapped with transfer).
- **Required Code:** Complete C program initiating non-blocking halo/data exchange with `MPI_Isend`/`MPI_Irecv`, performing local computation, and synchronizing with `MPI_Waitall`.

### Brief 11.03: Dynamic Process Management: Ports & Intercommunicators
- **File:** `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md`
- **Lecture Mapping:** Lecture 28
- **Key Concepts:** Dynamic process management (MPI-2 feature): connecting two independently launched MPI programs exchanging data, client-server paradigm in MPI, `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`, `MPI_Comm_disconnect`, `MPI_COMM_SELF`, Intracommunicators vs Intercommunicators, synchronization rule: why the sender MUST be waiting in `MPI_Comm_accept` before the receiver connects.
- **Pedagogical Goal:** Connect completely separate MPI jobs dynamically across network ports.
- **Required Diagrams:** Intercommunicator linking two separate `MPI_COMM_WORLD` universes.
- **Required API Signatures:** Full breakdown of `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`.

### Brief 11.04: Sender/Receiver Workflows & ompi-server
- **File:** `study/11_MPI_Communication/04_Sender_Receiver_Workflows_and_ompi_server.md`
- **Lecture Mapping:** Lecture 29
- **Key Concepts:** Sender workflow: open port, persist port string to `port.txt`, accept connection, send data; Receiver workflow: read port string from `port.txt`, connect, receive data; Open MPI's `ompi-server` for enterprise name publishing and lookup (`MPI_Publish_name`, `MPI_Lookup_name`); complete compilation, launch, and execution procedure for two separate binaries.
- **Pedagogical Goal:** Complete practical laboratory implementation of independent sender and receiver programs.
- **Required Code:** Two complete C programs: `server.c` (sender) and `client.c` (receiver) exchanging data via port file. Include step-by-step terminal execution instructions.

### Brief 11.05: MPI Troubleshooting & Error Handling
- **File:** `study/11_MPI_Communication/05_MPI_Troubleshooting_and_Error_Handling.md`
- **Lecture Mapping:** Lecture 29
- **Key Concepts:** Troubleshooting checklist: missing `port.txt` race conditions, hanging MPI jobs (deadlocks, mismatched tags/ranks), unsupported dynamic process management in minimal MPI builds, buffer overflow/truncation (`MPI_ERR_TRUNCATE`), checking MPI return codes, error handlers (`MPI_Errhandler_set`), debugging tools (Valgrind, MPI debuggers).
- **Pedagogical Goal:** Equip students to diagnose and fix real-world distributed MPI crashes and hangs.
- **Required Checklist:** Comprehensive 7-point MPI diagnostic and debugging checklist.

---

## Module 12: MPI Collective Operations (Lectures 30–31)
*Directory:* `study/12_MPI_Collectives/`

### Brief 12.01: Collective Operations Overview & Bcast
- **File:** `study/12_MPI_Collectives/01_Collective_Operations_Overview_and_Bcast.md`
- **Lecture Mapping:** Lecture 30
- **Key Concepts:** What makes an operation collective (all processes in a communicator participate), comparison with point-to-point (no tags, collective coordination, algorithmic optimizations), `MPI_Bcast`: syntax, parameter analysis, broadcasting identical data from root to all processes, spanning tree broadcast algorithms ($O(\log N)$ scaling vs $O(N)$ sequential sends).
- **Pedagogical Goal:** Understand the collective paradigm and master data broadcasting.
- **Required Diagrams:** Point-to-point loop vs. collective spanning-tree broadcast tree.
- **Required Code:** Complete C program broadcasting configuration data from rank 0 to all ranks with `MPI_Bcast`.

### Brief 12.02: Data Distribution & Scatter
- **File:** `study/12_MPI_Collectives/02_Data_Distribution_and_Scatter.md`
- **Lecture Mapping:** Lecture 30
- **Key Concepts:** Data distribution principle, `MPI_Scatter`: distributing distinct, equal-sized contiguous portions of an array from root to all processes, parameter distinction: `sendcount` is the count of elements sent to EACH process, not the total array size; receive buffer setup; root vs non-root argument requirements.
- **Pedagogical Goal:** Master partitioning large global datasets across distributed ranks.
- **Required Diagrams:** Visual memory diagram showing array on root split into equal chunks across processes.
- **Required Code:** Complete C program scattering a large array from rank 0, computing local values, and verifying results.

### Brief 12.03: Data Collection: Gather & Allgather
- **File:** `study/12_MPI_Collectives/03_Data_Collection_Gather_and_Allgather.md`
- **Lecture Mapping:** Lecture 31
- **Key Concepts:** `MPI_Gather`: collecting distinct segments from all processes back to the root's contiguous buffer (inverse of `MPI_Scatter`), `MPI_Allgather`: gathering distributed segments such that EVERY process receives the complete combined dataset (no root argument), memory buffer sizing for gather operations.
- **Pedagogical Goal:** Master aggregating distributed results back into unified arrays.
- **Required Diagrams:** Memory diagram showing `MPI_Gather` (data flows to root only) vs `MPI_Allgather` (data flows to all ranks).
- **Required Code:** Complete C program where ranks compute local data, gather them to root with `MPI_Gather`, and also demonstrate `MPI_Allgather`.

### Brief 12.04: Collective Reductions: Reduce & Allreduce
- **File:** `study/12_MPI_Collectives/04_Collective_Reductions_Reduce_and_Allreduce.md`
- **Lecture Mapping:** Lecture 31
- **Key Concepts:** `MPI_Reduce`: combining data across all ranks with an associative mathematical reduction operator, storing result exclusively at root; Predefined reduction operators: `MPI_SUM`, `MPI_MAX`, `MPI_MIN`, `MPI_PROD`, `MPI_LAND`, etc.; `MPI_Allreduce`: combining data and distributing the reduced result to ALL processes; vector/array reductions.
- **Pedagogical Goal:** Perform distributed parallel mathematical reductions efficiently.
- **Required Diagrams:** Logarithmic tree reduction across processes in `MPI_Reduce` vs `MPI_Allreduce`.
- **Required Code:** Complete C program computing the global minimum, maximum, and sum of distributed arrays using `MPI_Reduce` and `MPI_Allreduce`.

### Brief 12.05: Process Synchronization: MPI_Barrier & Comparison
- **File:** `study/12_MPI_Collectives/05_Process_Synchronization_MPI_Barrier_and_Comparison.md`
- **Lecture Mapping:** Lecture 31
- **Key Concepts:** `MPI_Barrier`: synchronizing all processes in a communicator, why `MPI_Barrier` exchanges zero application data (pure control synchronization), performance costs and why unnecessary barriers harm scaling, Grand Collective Comparison Matrix (Broadcast, Scatter, Gather, Allgather, Reduce, Allreduce, Barrier).
- **Pedagogical Goal:** Master process synchronization and solidify the collective API landscape.
- **Required Table:** Master Collective Comparison Table with columns: Operation, Sender(s), Receiver(s), Data Transferred, Root Argument, Common Use Case.

---

## Module 13: Hybrid MPI + OpenMP Programming (Lecture 32)
*Directory:* `study/13_Hybrid_MPI_OpenMP/`

### Brief 13.01: Hybrid Computing Model & Architecture
- **File:** `study/13_Hybrid_MPI_OpenMP/01_Hybrid_Computing_Model_and_Architecture.md`
- **Lecture Mapping:** Lecture 32
- **Key Concepts:** The modern supercomputer architecture: clusters of multi-socket, multi-core SMP nodes; why pure MPI has drawbacks (duplicated data, large memory footprint, network protocol overhead within same node); why pure OpenMP cannot scale beyond one node; the Hybrid Model: MPI across nodes (distributed memory), OpenMP within nodes (shared memory).
- **Pedagogical Goal:** Motivate the state-of-the-art programming paradigm for modern HPC clusters.
- **Required Diagrams:** Architectural hierarchy: Supercomputer Cluster $\rightarrow$ Nodes $\rightarrow$ Sockets $\rightarrow$ Multi-Core CPUs $\rightarrow$ Shared DRAM $\rightarrow$ Network Interconnect.
- **Required Comparisons:** Pure MPI vs Pure OpenMP vs Hybrid MPI+OpenMP.

### Brief 13.02: Process & Thread Hierarchy Calculations
- **File:** `study/13_Hybrid_MPI_OpenMP/02_Process_and_Thread_Hierarchy_Calculations.md`
- **Lecture Mapping:** Lecture 32
- **Key Concepts:** Thread counting mathematics:
  $$\text{Total Execution Units} = N_{\text{nodes}} \times P_{\text{procs/node}} \times T_{\text{threads/proc}}$$
  Detailed syllabus worked example: 2 nodes $\times$ 1 process $\times$ 4 threads = 8 total threads; exploring alternative configurations for fixed core counts (e.g., 16 cores: 16x1 vs 1x16 vs 4x4 vs 2x8); impact of process-to-thread ratio on memory usage, NUMA domain affinity, and communication performance.
- **Pedagogical Goal:** Master the mathematical modeling and configuration of hybrid execution topologies.
- **Required Solved Problems:** At least 3 solved numerical problems on process/thread mapping and core resource allocation.

### Brief 13.03: MPI Thread Safety Levels
- **File:** `study/13_Hybrid_MPI_OpenMP/03_MPI_Thread_Safety_Levels.md`
- **Lecture Mapping:** Lecture 32
- **Key Concepts:** Multi-threaded MPI initialization with `MPI_Init_thread(&argc, &argv, required, &provided)`, the four standard thread support levels:
  1. `MPI_THREAD_SINGLE`: Strictly single-threaded.
  2. `MPI_THREAD_FUNNELED`: Multiple OpenMP threads exist, but only the master thread makes MPI calls.
  3. `MPI_THREAD_SERIALIZED`: Multiple threads make MPI calls, but serialized (one thread at a time).
  4. `MPI_THREAD_MULTIPLE`: Any thread can make MPI calls concurrently without restriction.
  Hardware/runtime overhead and availability of `MPI_THREAD_MULTIPLE`.
- **Pedagogical Goal:** Ensure thread safety and avoid race conditions when mixing MPI calls with OpenMP threads.
- **Required Table:** Thread safety levels, required vs provided semantics, and performance implications.
- **Required Code:** C snippet querying and verifying MPI thread support levels.

### Brief 13.04: Worked Example: Hybrid Parallel Sum
- **File:** `study/13_Hybrid_MPI_OpenMP/04_Worked_Example_Hybrid_Parallel_Sum.md`
- **Lecture Mapping:** Lecture 32
- **Key Concepts:** The Complete Hybrid Masterpiece: Calculating the parallel sum of a massive dataset across multiple nodes and multiple threads per node.
  - Step 1: Work partitioned across MPI processes (domain decomposition).
  - Step 2: Each process spawns OpenMP threads to compute local partial sum with `#pragma omp parallel for reduction(+:local_sum)`.
  - Step 3: MPI processes combine local sums into global sum via `MPI_Reduce` or `MPI_Allreduce`.
- **Pedagogical Goal:** Deliver a complete, elegant, production-grade hybrid C program demonstrating the full synthesis of Part A and Part B.
- **Required Code:** Complete, fully commented C program ready for compilation with `mpicc -fopenmp`.
- **Required Trace:** ASCII execution trace showing data partitioning from cluster level down to OpenMP thread registers and back up through MPI reduction.

### Brief 13.05: Building, Running & Troubleshooting Hybrid Apps
- **File:** `study/13_Hybrid_MPI_OpenMP/05_Building_Running_and_Troubleshooting_Hybrid_Apps.md`
- **Lecture Mapping:** Lecture 32
- **Key Concepts:** Compilation flags: `mpicc -fopenmp program.c -o program`, execution launch commands: `export OMP_NUM_THREADS=4`, `mpirun -np 2 ./program`, process and thread binding (CPU affinity, core pinning with `--bind-to core`, `--map-by socket`), avoiding thread migration across NUMA nodes, hybrid troubleshooting checklist: thread safety mismatches, oversubscription of physical cores, stack overflow in OpenMP threads.
- **Pedagogical Goal:** Provide complete operational mastery for building and running hybrid applications in university lab and cluster environments.
- **Required Commands:** Exact terminal commands for compiling and launching hybrid jobs on 1, 2, and multi-node configurations.
