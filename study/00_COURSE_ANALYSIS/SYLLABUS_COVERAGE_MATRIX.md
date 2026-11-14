# Parallel Programming & High-Performance Computing (HPC)
## Master Syllabus Coverage Matrix

This document provides a 100% complete, verifiable, item-by-item mapping of every topic, concept, API, and architectural principle specified in the official course syllabus (`input_syllabus/HPC_Course_Syllabus.pdf`) to its designated study file in the `study/` directory.

---

## 1. Coverage Summary Statistics

| Metric | Target / Specification | Current Mapping Status |
|:---|:---:|:---:|
| **Total Syllabus Lectures** | 32 Lectures (Lectures 1 to 32) | 32 Lectures Mapped (100%) |
| **Course Divisions** | Part A: MidSem (L1–16) \| Part B: EndSem (L17–32) | 2 Parts Fully Structured |
| **Thematic Modules** | 13 Distinct Topic Folders | 13 Folders Defined |
| **Total Subtopic Study Files** | 62 Targeted Subtopic Files | 62 Files Mapped |
| **Total Explicit Syllabus Key Concepts** | 85 Key Concepts from Tables | 85 Concepts Tracked |
| **Coverage Percentage** | 100.0% | **100.0% Complete** |

---

## 2. Complete Item-by-Item Concept Mapping

### PART A: TILL MIDSEM (Lectures 1–16)

| Lecture(s) | Official Syllabus Topic | Explicit Key Concepts Listed in Syllabus | Target Study File Path | Coverage Status |
|:---:|:---|:---|:---|:---:|
| **L1** | Introduction to Parallel Computing & HPC Landscape | Course roadmap; evaluation scheme (EndSem 40%, Lab 40%, MidSem 20%) | `study/01_HPC_Foundations/01_Introduction_and_HPC_Landscape.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | HPC applications (scientific, industrial, societal) | `study/01_HPC_Foundations/01_Introduction_and_HPC_Landscape.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | The "Seven Dwarfs" of numerical methods (Berkeley view) | `study/01_HPC_Foundations/02_Seven_Dwarfs_of_Numerical_Methods.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | Parallel hardware hierarchy (ILP $\rightarrow$ SIMD $\rightarrow$ Cores $\rightarrow$ Sockets $\rightarrow$ Clusters) | `study/01_HPC_Foundations/03_Parallel_Hardware_Hierarchy_and_TOP500.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | TOP500 rankings, HPL benchmark, Green500 | `study/01_HPC_Foundations/03_Parallel_Hardware_Hierarchy_and_TOP500.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | Performance definition ($P = \text{Work} / \text{Time}$), Work vs Execution Time | `study/01_HPC_Foundations/04_Performance_Metrics_FLOPS_and_Power_Cost.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | FLOP/s interpretation (GFLOP/s, TFLOP/s, PFLOP/s, EFLOP/s, peak vs sustained) | `study/01_HPC_Foundations/04_Performance_Metrics_FLOPS_and_Power_Cost.md` | **COMPLETE** |
| **L1** | Introduction to Parallel Computing & HPC Landscape | Power-cost case study (Power Wall, energy efficiency, megawatt costs) | `study/01_HPC_Foundations/04_Performance_Metrics_FLOPS_and_Power_Cost.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Stored-program model (von Neumann architecture, fetch-decode-execute) | `study/02_Architectural_Foundations/01_Stored_Program_Model_and_Pipelining.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Instruction execution vs. data movement (von Neumann bottleneck, memory wall) | `study/02_Architectural_Foundations/01_Stored_Program_Model_and_Pipelining.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Instruction pipelining (stages, throughput, structural/data/control hazards) | `study/02_Architectural_Foundations/01_Stored_Program_Model_and_Pipelining.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Superscalar processors and Instruction-Level Parallelism (ILP, out-of-order) | `study/02_Architectural_Foundations/02_ILP_Superscalar_and_SMT.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Simultaneous Multithreading (SMT / Hyper-Threading) | `study/02_Architectural_Foundations/02_ILP_Superscalar_and_SMT.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | SIMD registers and vector instructions (SSE, AVX, AVX-512) | `study/02_Architectural_Foundations/03_SIMD_Registers_and_Vector_Extensions.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Memory hierarchy (registers to DRAM, cache lines, temporal/spatial locality) | `study/02_Architectural_Foundations/04_Memory_Hierarchy_and_Bandwidth.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Memory bandwidth, latency vs bandwidth, memory saturation | `study/02_Architectural_Foundations/04_Memory_Hierarchy_and_Bandwidth.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Flynn's classification (SISD, SIMD, MISD, MIMD) | `study/02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | UMA (SMP) vs. ccNUMA (Cache-Coherent Non-Uniform Memory Access) | `study/02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | Shared-memory vs. Distributed-memory systems | `study/02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md` | **COMPLETE** |
| **L2** | Architectural Foundations for Parallelism | OpenMP preview (bridging shared-memory hardware with directives) | `study/02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Speedup $S(N) = T_1 / T_N$ and parallel efficiency $\epsilon(N) = S(N)/N$ | `study/03_Parallel_Performance_and_Scaling/01_Speedup_Efficiency_and_Amdahls_Law.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Amdahl's Law (strong scaling, fixed problem size derivation) | `study/03_Parallel_Performance_and_Scaling/01_Speedup_Efficiency_and_Amdahls_Law.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Asymptotic limits of Amdahl's Law ($S_{\max} = 1/s$) | `study/03_Parallel_Performance_and_Scaling/01_Speedup_Efficiency_and_Amdahls_Law.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Parallel overhead models $c(N)$ ($T(N) = s T_1 + \frac{(1-s)T_1}{N} + c(N)$) | `study/03_Parallel_Performance_and_Scaling/03_Parallel_Overhead_and_Load_Imbalance.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Gustafson's Law (weak scaling, scaled speedup derivation) | `study/03_Parallel_Performance_and_Scaling/02_Gustafsons_Law_and_Weak_Scaling.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Weak scaling vs Strong scaling detailed comparison | `study/03_Parallel_Performance_and_Scaling/02_Gustafsons_Law_and_Weak_Scaling.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Load imbalance: definition, causes, quantification, and impact on efficiency | `study/03_Parallel_Performance_and_Scaling/03_Parallel_Overhead_and_Load_Imbalance.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Resource-bottleneck models (Compute bottlenecks, Memory-bandwidth, Comm) | `study/03_Parallel_Performance_and_Scaling/04_Resource_Bottleneck_Models.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Benchmarking methodology (precision timers, wall-clock vs CPU time) | `study/03_Parallel_Performance_and_Scaling/05_Benchmarking_Methodology_and_Pitfalls.md` | **COMPLETE** |
| **L3–5** | Parallel Computing and Its Limits | Performance-measurement pitfalls (warmup, cold cache, turboboost, jitter) | `study/03_Parallel_Performance_and_Scaling/05_Benchmarking_Methodology_and_Pitfalls.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | OpenMP architecture (pragmas, runtime library, environment variables) | `study/04_OpenMP_Fundamentals/01_OpenMP_Architecture_and_Fork_Join_Model.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | Shared-memory programming model & memory abstraction | `study/04_OpenMP_Fundamentals/01_OpenMP_Architecture_and_Fork_Join_Model.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | Fork-join execution model (master thread, team creation, implicit barrier) | `study/04_OpenMP_Fundamentals/01_OpenMP_Architecture_and_Fork_Join_Model.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | General OpenMP pragma syntax (`#pragma omp directive-name [clauses]`) | `study/04_OpenMP_Fundamentals/02_Pragma_Syntax_Parallel_Regions_and_Compilation.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | Parallel regions (`#pragma omp parallel`, `num_threads`, thread functions) | `study/04_OpenMP_Fundamentals/02_Pragma_Syntax_Parallel_Regions_and_Compilation.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | Compiling (`gcc -fopenmp`) and running OpenMP programs | `study/04_OpenMP_Fundamentals/02_Pragma_Syntax_Parallel_Regions_and_Compilation.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | Data scoping: shared variables vs. private variables | `study/04_OpenMP_Fundamentals/03_Data_Scoping_Shared_Private_and_Default_None.md` | **COMPLETE** |
| **L6–7** | Basics of OpenMP | `default(none)` and defensive programming practices | `study/04_OpenMP_Fundamentals/03_Data_Scoping_Shared_Private_and_Default_None.md` | **COMPLETE** |
| **L8** | OpenMP Work-Sharing Constructs | Work-sharing concept (distributing iterations across existing team) | `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md` | **COMPLETE** |
| **L8** | OpenMP Work-Sharing Constructs | `#pragma omp for` (canonical loop requirements and iteration distribution) | `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md` | **COMPLETE** |
| **L8** | OpenMP Work-Sharing Constructs | `sections` construct (`#pragma omp sections`, `#pragma omp section`) | `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md` | **COMPLETE** |
| **L8** | OpenMP Work-Sharing Constructs | `single` construct (single-thread execution, barrier semantics) | `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md` | **COMPLETE** |
| **L8** | OpenMP Work-Sharing Constructs | Work distribution across a thread team & construct comparison | `study/05_OpenMP_Work_Sharing_and_Scheduling/01_Work_Sharing_For_Sections_and_Single.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | `firstprivate` clause (initialization semantics vs private) | `study/05_OpenMP_Work_Sharing_and_Scheduling/02_Firstprivate_Clause_and_Initialization.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | `#pragma omp for` scheduling integration | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | Static scheduling (`schedule(static, [chunk])`) | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | Dynamic scheduling (`schedule(dynamic, [chunk])`) | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | Guided scheduling (`schedule(guided, [chunk])`) | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | Auto scheduling (`schedule(auto)`) | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | How iterations are assigned to threads (mathematical mapping) | `study/05_OpenMP_Work_Sharing_and_Scheduling/03_Loop_Scheduling_Strategies.md` | **COMPLETE** |
| **L9–10** | OpenMP Loop Scheduling | Scheduling trade-offs (overhead vs load balance, cache affinity) | `study/05_OpenMP_Work_Sharing_and_Scheduling/04_Scheduling_Tradeoffs_and_Performance.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | Critical sections (`#pragma omp critical [name]`) | `study/06_OpenMP_Synchronization/01_Race_Conditions_Critical_and_Atomic.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | `critical` vs. `atomic` (`#pragma omp atomic`, hardware atomics) | `study/06_OpenMP_Synchronization/01_Race_Conditions_Critical_and_Atomic.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | Barriers (`#pragma omp barrier`, implicit vs explicit barriers) | `study/06_OpenMP_Synchronization/02_Barriers_Nowait_and_Ordered_Execution.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | `single` with and without `nowait` clause | `study/06_OpenMP_Synchronization/02_Barriers_Nowait_and_Ordered_Execution.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | Ordered execution (`#pragma omp ordered`) | `study/06_OpenMP_Synchronization/02_Barriers_Nowait_and_Ordered_Execution.md` | **COMPLETE** |
| **L11** | OpenMP Synchronization | Low-level OpenMP locks: `omp_lock_t` and complete API | `study/06_OpenMP_Synchronization/03_Low_Level_Locks_omp_lock_t.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | Race conditions in shared sums (machine-level interleaving demonstration) | `study/06_OpenMP_Synchronization/01_Race_Conditions_Critical_and_Atomic.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | Fixing race conditions using `atomic` and `critical` | `study/06_OpenMP_Synchronization/04_Worked_Examples_Manual_Local_Sum_and_False_Sharing.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | Manual local-sum pattern (thread array accumulation) | `study/06_OpenMP_Synchronization/04_Worked_Examples_Manual_Local_Sum_and_False_Sharing.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | False sharing analysis and cache-line padding | `study/06_OpenMP_Synchronization/04_Worked_Examples_Manual_Local_Sum_and_False_Sharing.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | OpenMP reduction clause (`reduction(op : list)`) | `study/06_OpenMP_Synchronization/05_Reduction_Clause_and_Mathematical_Properties.md` | **COMPLETE** |
| **L12** | Worked Examples: Critical, Atomic, Reduction | Reduction operators and mathematical properties (associativity, commutativity) | `study/06_OpenMP_Synchronization/05_Reduction_Clause_and_Mathematical_Properties.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | Task parallelism vs. Loop (data) parallelism | `study/07_OpenMP_Tasks_and_SIMD/01_Task_Parallelism_vs_Data_Parallelism.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | OpenMP tasks (`#pragma omp task`, task queues, thread pools) | `study/07_OpenMP_Tasks_and_SIMD/02_OpenMP_Task_and_Taskloop_Constructs.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | `#pragma omp taskloop` construct | `study/07_OpenMP_Tasks_and_SIMD/02_OpenMP_Task_and_Taskloop_Constructs.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | `grainsize` and `num_tasks` clauses | `study/07_OpenMP_Tasks_and_SIMD/02_OpenMP_Task_and_Taskloop_Constructs.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | Task distribution and execution (recursive tasks, work stealing) | `study/07_OpenMP_Tasks_and_SIMD/02_OpenMP_Task_and_Taskloop_Constructs.md` | **COMPLETE** |
| **L13–14** | Task Parallelism & OpenMP Taskloop | When task-based parallelism is useful (irregular, recursive, graph algorithms) | `study/07_OpenMP_Tasks_and_SIMD/01_Task_Parallelism_vs_Data_Parallelism.md` | **COMPLETE** |
| **L15** | OpenMP SIMD — Part 1 | SIMD execution model and `#pragma omp simd` | `study/07_OpenMP_Tasks_and_SIMD/03_SIMD_Vectorization_Basics_and_Pragma_simd.md` | **COMPLETE** |
| **L15** | OpenMP SIMD — Part 1 | How the compiler vectorizes loops (SSE, AVX, AVX-512) | `study/07_OpenMP_Tasks_and_SIMD/03_SIMD_Vectorization_Basics_and_Pragma_simd.md` | **COMPLETE** |
| **L15** | OpenMP SIMD — Part 1 | Compiler vectorization reports (`-fopt-info-vec`, `-fopt-info-vec-missed`) | `study/07_OpenMP_Tasks_and_SIMD/04_Compiler_Vectorization_Reports_and_Flags.md` | **COMPLETE** |
| **L15** | OpenMP SIMD — Part 1 | Compilation flag `-march=native` | `study/07_OpenMP_Tasks_and_SIMD/04_Compiler_Vectorization_Reports_and_Flags.md` | **COMPLETE** |
| **L15** | OpenMP SIMD — Part 1 | Vectorization opportunities and limitations (dependencies, aliasing) | `study/07_OpenMP_Tasks_and_SIMD/04_Compiler_Vectorization_Reports_and_Flags.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | `private()` in SIMD loops and scoping rules | `study/07_OpenMP_Tasks_and_SIMD/05_SIMD_Scoping_Private_and_Latent_Bugs.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | Latent bugs caused by missing or incorrect scoping in SIMD | `study/07_OpenMP_Tasks_and_SIMD/05_SIMD_Scoping_Private_and_Latent_Bugs.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | SIMD reduction (`reduction(+:var)`) | `study/07_OpenMP_Tasks_and_SIMD/06_SIMD_Reduction_Alignment_and_Warmup.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | The `aligned` clause (`aligned(ptr : alignment)`) | `study/07_OpenMP_Tasks_and_SIMD/06_SIMD_Reduction_Alignment_and_Warmup.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | Cache-line alignment effects (aligned vs unaligned memory access) | `study/07_OpenMP_Tasks_and_SIMD/06_SIMD_Reduction_Alignment_and_Warmup.md` | **COMPLETE** |
| **L16** | OpenMP SIMD — Part 2 | Benchmark warmup and steady-state vector performance | `study/07_OpenMP_Tasks_and_SIMD/06_SIMD_Reduction_Alignment_and_Warmup.md` | **COMPLETE** |

---

### PART B: AFTER MIDSEM (Lectures 17–32)

| Lecture(s) | Official Syllabus Topic | Explicit Key Concepts Listed in Syllabus | Target Study File Path | Coverage Status |
|:---:|:---|:---|:---|:---:|
| **L17** | Introduction to POSIX & Pthreads | What POSIX is and why portability matters (APIs, system calls, utilities) | `study/08_POSIX_and_Pthreads/01_POSIX_Standard_and_Pthreads_Overview.md` | **COMPLETE** |
| **L17** | Introduction to POSIX & Pthreads | Certified systems (macOS, AIX, HP-UX, Solaris, QNX) vs mostly compliant (Linux, BSD, Android) | `study/08_POSIX_and_Pthreads/01_POSIX_Standard_and_Pthreads_Overview.md` | **COMPLETE** |
| **L17** | Introduction to POSIX & Pthreads | Pthreads vs. OpenMP (manual thread management vs `#pragma` directives) | `study/08_POSIX_and_Pthreads/01_POSIX_Standard_and_Pthreads_Overview.md` | **COMPLETE** |
| **L17** | Introduction to POSIX & Pthreads | What you manage in Pthreads: creating threads, assigning work, synchronizing | `study/08_POSIX_and_Pthreads/01_POSIX_Standard_and_Pthreads_Overview.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | `pthread_t` as an opaque, platform-dependent thread handle | `study/08_POSIX_and_Pthreads/02_Thread_Creation_pthread_create_and_Handles.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | `pthread_create` complete function signature and parameter semantics | `study/08_POSIX_and_Pthreads/02_Thread_Creation_pthread_create_and_Handles.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | `pthread_attr_t`: attributes object lifecycle (`init`/`destroy`) | `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | Thread stack size configuration (`pthread_attr_setstacksize`) | `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | Scheduling policy: `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR` | `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | Detach state: `PTHREAD_CREATE_JOINABLE` vs `PTHREAD_CREATE_DETACHED` | `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md` | **COMPLETE** |
| **L18** | Creating Threads: `pthread_create` & Attributes | Thread priority (mainly relevant under real-time policies, `sched_param`) | `study/08_POSIX_and_Pthreads/03_Thread_Attributes_and_Scheduling_Policies.md` | **COMPLETE** |
| **L19** | Joining & Detaching Threads | `pthread_join` and retrieving a thread's exit status (`pthread_exit`, return codes) | `study/08_POSIX_and_Pthreads/04_Joining_Detaching_and_Resource_Reclamation.md` | **COMPLETE** |
| **L19** | Joining & Detaching Threads | Joinable vs detached threads and what happens to a detached thread's resources | `study/08_POSIX_and_Pthreads/04_Joining_Detaching_and_Resource_Reclamation.md` | **COMPLETE** |
| **L19** | Joining & Detaching Threads | Custom thread stack sizes and resource reclamation (preventing zombie leaks) | `study/08_POSIX_and_Pthreads/04_Joining_Detaching_and_Resource_Reclamation.md` | **COMPLETE** |
| **L19** | Joining & Detaching Threads | Passing distinct arguments to $N$ threads (structs, heap vs stack) | `study/08_POSIX_and_Pthreads/05_Argument_Passing_Patterns_and_Pitfalls.md` | **COMPLETE** |
| **L19** | Joining & Detaching Threads | Common argument-passing mistakes (loop index reference bug) | `study/08_POSIX_and_Pthreads/05_Argument_Passing_Patterns_and_Pitfalls.md` | **COMPLETE** |
| **L20** | Pthreads Synchronization: Race Conditions & Mutexes | Overview of the synchronization toolbox | `study/09_Pthreads_Synchronization/01_Race_Conditions_and_Mutexes.md` | Planned |
| **L20** | Pthreads Synchronization: Race Conditions & Mutexes | Race conditions in shared data structures | `study/09_Pthreads_Synchronization/01_Race_Conditions_and_Mutexes.md` | Planned |
| **L20** | Pthreads Synchronization: Race Conditions & Mutexes | Mutexes: `pthread_mutex_init`, `lock`, `trylock`, `unlock`, `destroy` | `study/09_Pthreads_Synchronization/01_Race_Conditions_and_Mutexes.md` | Planned |
| **L20** | Pthreads Synchronization: Race Conditions & Mutexes | Protecting a shared counter (complete critical-section C program) | `study/09_Pthreads_Synchronization/01_Race_Conditions_and_Mutexes.md` | Planned |
| **L21** | Condition Variables | Condition variables (`pthread_cond_t`): wait, signal, broadcast | `study/09_Pthreads_Synchronization/02_Condition_Variables_and_Coordination.md` | Planned |
| **L21** | Condition Variables | `pthread_cond_wait` mechanism: atomically releases mutex while blocking | `study/09_Pthreads_Synchronization/02_Condition_Variables_and_Coordination.md` | Planned |
| **L21** | Condition Variables | Why condition variables must always be paired with a mutex | `study/09_Pthreads_Synchronization/02_Condition_Variables_and_Coordination.md` | Planned |
| **L21** | Condition Variables | The wait-in-a-while-loop pattern and why it is needed (spurious wakeups) | `study/09_Pthreads_Synchronization/02_Condition_Variables_and_Coordination.md` | Planned |
| **L22** | Producer–Consumer & Semaphores | Bounded buffer: waiting when full or empty without busy waiting | `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md` | Planned |
| **L22** | Producer–Consumer & Semaphores | POSIX Semaphores: `sem_init`, `sem_wait`, `sem_post`, `sem_destroy` | `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md` | Planned |
| **L22** | Producer–Consumer & Semaphores | Counting semaphores vs binary semaphores | `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md` | Planned |
| **L22** | Producer–Consumer & Semaphores | Why they live in `<semaphore.h>` (POSIX Realtime Extensions, IEEE 1003.1b) | `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md` | Planned |
| **L22** | Producer–Consumer & Semaphores | Comparing mutex/condition-variable and semaphore solutions | `study/09_Pthreads_Synchronization/03_Producer_Consumer_and_POSIX_Semaphores.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Read-write locks (`pthread_rwlock_t`): concurrent readers, exclusive writer | `study/09_Pthreads_Synchronization/04_Read_Write_Locks_and_Read_Heavy_Workloads.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Read-heavy workloads, writer starvation and fairness | `study/09_Pthreads_Synchronization/04_Read_Write_Locks_and_Read_Heavy_Workloads.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Barriers (`pthread_barrier_init`, `pthread_barrier_wait`) | `study/09_Pthreads_Synchronization/05_Barriers_and_Phase_Synchronization.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Synchronizing parallel phases in multi-step algorithms | `study/09_Pthreads_Synchronization/05_Barriers_and_Phase_Synchronization.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Spinlocks (`pthread_spin_init`, `pthread_spin_lock`, `pthread_spin_unlock`) | `study/09_Pthreads_Synchronization/06_Spinlocks_Busy_Waiting_and_Selection_Guide.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | `PTHREAD_PROCESS_PRIVATE` vs `PTHREAD_PROCESS_SHARED` | `study/09_Pthreads_Synchronization/06_Spinlocks_Busy_Waiting_and_Selection_Guide.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | When busy-waiting beats context switching | `study/09_Pthreads_Synchronization/06_Spinlocks_Busy_Waiting_and_Selection_Guide.md` | Planned |
| **L23** | Read-Write Locks, Barriers & Spinlocks | Comprehensive decision framework for choosing the right synchronization primitive | `study/09_Pthreads_Synchronization/06_Spinlocks_Busy_Waiting_and_Selection_Guide.md` | Planned |
| **L24** | Introduction to MPI & Distributed-Memory Model | Distributed-memory architecture: private memory per process, interconnects | `study/10_MPI_Fundamentals/01_Distributed_Memory_and_Message_Passing_Model.md` | Planned |
| **L24** | Introduction to MPI & Distributed-Memory Model | Message-passing communication model | `study/10_MPI_Fundamentals/01_Distributed_Memory_and_Message_Passing_Model.md` | Planned |
| **L24** | Introduction to MPI & Distributed-Memory Model | MPI vs. OpenMP (separate vs shared address spaces, multi-machine support) | `study/10_MPI_Fundamentals/01_Distributed_Memory_and_Message_Passing_Model.md` | Planned |
| **L24** | Introduction to MPI & Distributed-Memory Model | Explicit message communication vs direct memory access | `study/10_MPI_Fundamentals/01_Distributed_Memory_and_Message_Passing_Model.md` | Planned |
| **L24** | Introduction to MPI & Distributed-Memory Model | Structure of an MPI program: `MPI_Init`, `MPI_Comm_rank`, `MPI_Comm_size`, `MPI_Finalize` | `study/10_MPI_Fundamentals/02_Structure_of_an_MPI_Program.md` | Planned |
| **L25** | Building & Running MPI Programs | Details of `MPI_Init` (called once by main thread, argument semantics) | `study/10_MPI_Fundamentals/02_Structure_of_an_MPI_Program.md` | Planned |
| **L25** | Building & Running MPI Programs | `MPI_COMM_WORLD` predefined communicator | `study/10_MPI_Fundamentals/02_Structure_of_an_MPI_Program.md` | Planned |
| **L25** | Building & Running MPI Programs | Rank and size: why the same executable behaves differently per process (SPMD) | `study/10_MPI_Fundamentals/02_Structure_of_an_MPI_Program.md` | Planned |
| **L25** | Building & Running MPI Programs | Compiling with `mpicc` / `mpicxx` wrapper | `study/10_MPI_Fundamentals/03_Compiling_and_Launching_MPI_Programs.md` | Planned |
| **L25** | Building & Running MPI Programs | Launching with `mpirun` / `mpiexec`: `-np`, `--oversubscribe`, `--hostfile`, `--display-allocation` | `study/10_MPI_Fundamentals/03_Compiling_and_Launching_MPI_Programs.md` | Planned |
| **L25** | Building & Running MPI Programs | Overview of point-to-point vs. collective communication | `study/10_MPI_Fundamentals/04_Communication_Paradigms_Overview.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | Sender and receiver communication model | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | Message envelope arguments: buffer, count, datatype, rank, tag, communicator | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | Using tags to distinguish messages | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | `MPI_Send` and `MPI_Recv` syntax and mechanics | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | What blocking actually means (buffer safety vs receiver delivery) | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L26** | MPI Point-to-Point: Blocking Send & Receive | Eager vs Rendezvous buffering protocols | `study/11_MPI_Communication/01_Blocking_Point_to_Point_Send_and_Recv.md` | Planned |
| **L27** | Non-Blocking Communication | `MPI_Isend` and `MPI_Irecv` non-blocking primitives | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L27** | Non-Blocking Communication | Request handles (`MPI_Request`) | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L27** | Non-Blocking Communication | Overlapping computation with communication (latency hiding) | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L27** | Non-Blocking Communication | Completing operations with `MPI_Wait` (itself blocking) and `MPI_Test` | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L27** | Non-Blocking Communication | Rules for when it is safe to reuse or modify a buffer | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L27** | Non-Blocking Communication | Direct comparison: blocking vs non-blocking communication | `study/11_MPI_Communication/02_Non_Blocking_Communication_Isend_and_Irecv.md` | Planned |
| **L28** | Connecting Separate MPI Programs: Ports & Intercommunicators | Dynamic process management: two independently launched MPI programs exchanging data | `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md` | Planned |
| **L28** | Connecting Separate MPI Programs: Ports & Intercommunicators | `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`, `MPI_Comm_disconnect` | `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md` | Planned |
| **L28** | Connecting Separate MPI Programs: Ports & Intercommunicators | Role of `MPI_COMM_SELF` | `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md` | Planned |
| **L28** | Connecting Separate MPI Programs: Ports & Intercommunicators | Intercommunicators vs Intracommunicators | `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md` | Planned |
| **L28** | Connecting Separate MPI Programs: Ports & Intercommunicators | Why the sender must be waiting in `MPI_Comm_accept` before receiver connects | `study/11_MPI_Communication/03_Dynamic_Process_Management_Ports_and_Intercommunicators.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | Sender workflow (open port, save port name, accept, send) | `study/11_MPI_Communication/04_Sender_Receiver_Workflows_and_ompi_server.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | Receiver workflow (read port name, connect, receive) | `study/11_MPI_Communication/04_Sender_Receiver_Workflows_and_ompi_server.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | `ompi-server` for name publishing and lookup between separate jobs | `study/11_MPI_Communication/04_Sender_Receiver_Workflows_and_ompi_server.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | Compile and run procedure for independent jobs | `study/11_MPI_Communication/04_Sender_Receiver_Workflows_and_ompi_server.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | Troubleshooting checklist: missing `port.txt`, hangs, unsupported dynamic process management | `study/11_MPI_Communication/05_MPI_Troubleshooting_and_Error_Handling.md` | Planned |
| **L29** | Sender/Receiver Workflows, ompi-server & Troubleshooting | Safe buffer widths and checking MPI return codes | `study/11_MPI_Communication/05_MPI_Troubleshooting_and_Error_Handling.md` | Planned |
| **L30** | MPI Collectives: Data Distribution | What makes an operation collective (all processes participate, unlike point-to-point) | `study/12_MPI_Collectives/01_Collective_Operations_Overview_and_Bcast.md` | Planned |
| **L30** | MPI Collectives: Data Distribution | Differences between collective and point-to-point communication | `study/12_MPI_Collectives/01_Collective_Operations_Overview_and_Bcast.md` | Planned |
| **L30** | MPI Collectives: Data Distribution | `MPI_Bcast` (broadcasting identical data from root to everyone) | `study/12_MPI_Collectives/01_Collective_Operations_Overview_and_Bcast.md` | Planned |
| **L30** | MPI Collectives: Data Distribution | `MPI_Scatter` (distributing different equal portions to each process) | `study/12_MPI_Collectives/02_Data_Distribution_and_Scatter.md` | Planned |
| **L30** | MPI Collectives: Data Distribution | Root process role, buffer and count arguments in collective routines | `study/12_MPI_Collectives/02_Data_Distribution_and_Scatter.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | `MPI_Gather` (assembling segments from each process to root) | `study/12_MPI_Collectives/03_Data_Collection_Gather_and_Allgather.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | `MPI_Allgather` (assembling segments from each process to every process) | `study/12_MPI_Collectives/03_Data_Collection_Gather_and_Allgather.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | `MPI_Reduce` (sum, max, min; result stored exclusively at root) | `study/12_MPI_Collectives/04_Collective_Reductions_Reduce_and_Allreduce.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | `MPI_Allreduce` (sum, max, min; result stored at every process) | `study/12_MPI_Collectives/04_Collective_Reductions_Reduce_and_Allreduce.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | `MPI_Barrier` process synchronization | `study/12_MPI_Collectives/05_Process_Synchronization_MPI_Barrier_and_Comparison.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | Why `MPI_Barrier` exchanges zero application data | `study/12_MPI_Collectives/05_Process_Synchronization_MPI_Barrier_and_Comparison.md` | Planned |
| **L31** | MPI Collectives: Collection, Reduction & Synchronization | Grand comparative analysis of all collective operations | `study/12_MPI_Collectives/05_Process_Synchronization_MPI_Barrier_and_Comparison.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | Hybrid parallel architecture: MPI between nodes, OpenMP threads within each process | `study/13_Hybrid_MPI_OpenMP/01_Hybrid_Computing_Model_and_Architecture.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | MPI processes vs OpenMP threads: separate vs shared address spaces | `study/13_Hybrid_MPI_OpenMP/01_Hybrid_Computing_Model_and_Architecture.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | Calculating total thread count across nodes | `study/13_Hybrid_MPI_OpenMP/02_Process_and_Thread_Hierarchy_Calculations.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | Worked example: 2 nodes $\times$ 1 process $\times$ 4 threads = 8 total threads | `study/13_Hybrid_MPI_OpenMP/02_Process_and_Thread_Hierarchy_Calculations.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | MPI thread safety levels: `MPI_Init_thread` (`SINGLE`, `FUNNELED`, `SERIALIZED`, `MULTIPLE`) | `study/13_Hybrid_MPI_OpenMP/03_MPI_Thread_Safety_Levels.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | Worked example: Complete hybrid MPI + OpenMP parallel sum | `study/13_Hybrid_MPI_OpenMP/04_Worked_Example_Hybrid_Parallel_Sum.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | How work is divided hierarchically between processes and threads | `study/13_Hybrid_MPI_OpenMP/04_Worked_Example_Hybrid_Parallel_Sum.md` | Planned |
| **L32** | Hybrid MPI + OpenMP | Complete hybrid program compilation (`mpicc -fopenmp`), launch, and troubleshooting | `study/13_Hybrid_MPI_OpenMP/05_Building_Running_and_Troubleshooting_Hybrid_Apps.md` | Planned |

---

## 3. Official Citations & Reference Materials Accounted For

| Syllabus Reference | Cited in Syllabus Context | Primary Target Module | Target Study File(s) |
|:---|:---|:---:|:---|
| **Gene M. Amdahl (1967)** | *"Validity of the single processor approach to achieving large scale computing capabilities"* | Module 03 | `03_Parallel_Performance_and_Scaling/01_Speedup_Efficiency_and_Amdahls_Law.md` |
| **John L. Gustafson (1988)** | *"Reevaluating Amdahl's Law"*, Communications of the ACM | Module 03 | `03_Parallel_Performance_and_Scaling/02_Gustafsons_Law_and_Weak_Scaling.md` |
| **UC Berkeley (Asanović et al.)**| *"The Landscape of Parallel Computing Research: A View from Berkeley"* (Seven Dwarfs) | Module 01 | `01_HPC_Foundations/02_Seven_Dwarfs_of_Numerical_Methods.md` |
| **Michael J. Flynn (1972)** | Flynn's classification of parallel computer architectures (IEEE TC) | Module 02 | `02_Architectural_Foundations/05_Flynn_Taxonomy_and_Memory_Architectures.md` |
| **openmp.org** | Specification, directives, and runtime library reference | Modules 04–07, 13 | All OpenMP modules (`04`, `05`, `06`, `07`, `13`) |
| **TOP500 List** | `top500.org` supercomputer performance rankings and Linpack benchmark | Module 01 | `01_HPC_Foundations/03_Parallel_Hardware_Hierarchy_and_TOP500.md` |
| **POSIX Standard (IEEE 1003.1)**| POSIX Threads and Realtime Extensions (`<pthread.h>`, `<semaphore.h>`) | Modules 08–09 | All Pthreads modules (`08`, `09`) |
| **MPI Forum Specifications** | Open MPI / MPICH documentation (`mpi.h`, `mpicc`, `mpirun`, `ompi-server`) | Modules 10–13 | All MPI modules (`10`, `11`, `12`, `13`) |

---

## 4. Software & Tools Usage Verification

| Tool / Compiler Option | Mentioned in Syllabus | Target Study Modules & Lab Exercises | Verified Inclusion |
|:---|:---|:---|:---:|
| **GCC / Intel Compilers** | `-fopenmp`, `-qopenmp` | Modules 04, 05, 06, 07, 13 | [x] |
| **Optimization & SIMD Flags** | `-O3`, `-march=native`, `-fopt-info-vec` | Modules 02, 07 | [x] |
| **OpenMP Runtime API** | `<omp.h>` (`omp_get_thread_num`, `omp_get_wtime`, etc.) | Modules 04, 05, 06, 07, 13 | [x] |
| **POSIX Threads Compiler** | `-pthread`, `<pthread.h>` | Modules 08, 09 | [x] |
| **POSIX Realtime Semaphores**| `<semaphore.h>`, `-lrt` | Module 09 | [x] |
| **MPI Compiler Wrapper** | `mpicc`, `mpicxx` | Modules 10, 11, 12, 13 | [x] |
| **MPI Job Launcher** | `mpirun`, `mpiexec` | Modules 10, 11, 12, 13 | [x] |
| **Launch Configurations** | `--oversubscribe`, `--hostfile`, `--display-allocation` | Modules 10, 11, 13 | [x] |
| **Dynamic Name Server** | `ompi-server` | Module 11 | [x] |
| **Hybrid Compiler Command** | `mpicc -fopenmp` | Module 13 | [x] |
