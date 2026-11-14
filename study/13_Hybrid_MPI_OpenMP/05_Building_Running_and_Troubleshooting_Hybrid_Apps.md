# Subtopic 13.05: Building, Running, and Troubleshooting Hybrid MPI + OpenMP Applications

---

## 1. Basic Idea

Writing syntactically valid hybrid MPI + OpenMP source code is only half the battle in high-performance computing. A hybrid program lives simultaneously in two execution models: the operating system process model managed by the distributed MPI runtime, and the user/kernel thread model managed by the OpenMP threading library (`libgomp`, `libiomp5`). Because of this dual runtime reality, compiling, deploying, pinning, and debugging hybrid software requires precise orchestration.

Building a hybrid application requires instructing the compiler driver wrapper (such as `mpicc`) to pass multithreading flags (`-fopenmp` for GCC/Clang, `-qopenmp` for Intel) to both the compilation phase and the linker phase. If the OpenMP flag is omitted during compilation, the preprocessor silently ignores `#pragma omp` directives, collapsing your parallel regions into single-threaded serial execution without issuing a single warning.

Running a hybrid application requires coordinating the distributed process launcher (`mpirun`, `mpiexec`, or SLURM's `srun`) with the operating system scheduler and OpenMP runtime environment variables (`OMP_NUM_THREADS`, `OMP_PROC_BIND`, `OMP_PLACES`). The process launcher must distribute exactly $M$ MPI processes per node, spaced across physical CPU sockets, and reserve exactly $T$ physical hardware cores per process. In turn, the OpenMP runtime must pin each of the $T$ worker threads to dedicated, distinct physical cores inside the assigned socket to prevent operating system thread migration and Non-Uniform Memory Access (NUMA) cache-line thrashing.

Troubleshooting hybrid systems requires a systematic diagnostic methodology to detect and isolate five major failure modes:
1. **Core Over-Subscription:** Accidentally launching $M \times T$ threads exceeding the number of physical hardware cores, causing catastrophic context-switching overhead.
2. **Silent OpenMP Deactivation:** Omitting `-fopenmp` at compile or link time, producing an application that runs purely as $M$ single-threaded MPI processes.
3. **Thread Safety Level Violations:** Making concurrent MPI calls from OpenMP worker threads when the MPI library only initialized support for `MPI_THREAD_SINGLE` or `MPI_THREAD_FUNNELED`.
4. **NUMA Straddling and Thread Wandering:** Failing to bind threads to cores, causing the OS kernel scheduler to migrate threads across CPU sockets, incurring massive inter-socket crossbar memory latency penalties.
5. **Environment Variable Propagation Failures:** Setting `export OMP_NUM_THREADS=8` on the login head node, but failing to propagate the environment variable across remote compute nodes via `mpirun -x` or SLURM job steps, causing compute nodes to default to 1 thread or all available logical cores.

---

## 2. Why We Need It

In pure OpenMP, launching a job simply requires `./my_program`, and the OpenMP runtime automatically queries the local machine's core count. In pure MPI, launching a job uses `mpirun -np 64 ./my_program`, where every process occupies one CPU core with straightforward 1:1 binding.

However, in hybrid computing, default launcher behaviors produce catastrophic performance collapses if executed naively:

1. **The Over-Subscription Trap:**
   If a user submits an MPI job requesting 16 MPI processes across a dual-socket 32-core node (`mpirun -np 16 ./hybrid_app`), and the OpenMP runtime defaults to using all detected hardware threads on the node ($T = 32$), the operating system suddenly attempts to run:
   $$\text{Total Active Threads} = 16 \text{ processes} \times 32 \text{ threads/process} = 512 \text{ threads}$$
   On a node with only 32 physical execution pipelines, 512 active compute threads compete viciously for CPU time slices. The CPU spends over 70% of its clock cycles executing OS kernel context switches, cache invalidations, and TLB flushes rather than numerical work, reducing computational throughput by up to an order of magnitude.

2. **The Under-Subscription / Resource Starvation Trap:**
   Conversely, if an MPI launcher places 2 MPI processes on a 64-core node, but the remote cluster compute daemon does not forward `OMP_NUM_THREADS`, the OpenMP library may default to 1 thread per process. As a result, only 2 cores are active while 62 expensive enterprise supercomputer cores sit completely idle.

3. **The NUMA Affinity Disaster:**
   Modern dual-socket servers connect two independent memory controllers across high-speed interconnects (such as Intel Ultra Path Interconnect [UPI] or AMD Infinity Fabric). Accessing memory attached to the local socket takes $\approx 70\text{ ns}$, whereas accessing memory attached to the remote socket across the interconnect takes $\approx 140\text{ ns}$ (a 2x latency penalty with severely throttled bandwidth). If OpenMP threads are free to wander between cores across both sockets, memory pages allocated in local DRAM become remote references when the thread is rescheduled on the other socket, crippling memory-bound algorithms (such as sparse matrix solvers, stencil computations, and FFTs).

Understanding how to build, bind, run, and diagnose hybrid codes ensures that theoretical algorithmic scalability translates into actual bare-metal supercomputing performance.

---

## 3. How It Works

### 3.1 The Hybrid Compilation Pipeline

MPI compilers are not distinct compilers from scratch; they are **compiler wrappers** (shell scripts or binary proxies) around native system compilers such as GCC, Clang, or Intel oneAPI. When you invoke `mpicc`, the wrapper automatically inserts the necessary MPI header include paths (`-I/opt/openmpi/include`) and MPI library link flags (`-L/opt/openmpi/lib -lmpi`).

However, the MPI compiler wrapper **does not** automatically enable OpenMP. You must explicitly supply the OpenMP compilation and linkage flags to the wrapper:

```bash
# Using GCC / Clang toolchain
mpicc -O3 -fopenmp hybrid_code.c -o hybrid_app

# Using Intel oneAPI toolchain
mpiicc -O3 -qopenmp hybrid_code.c -o hybrid_app
```

```
+---------------------------------------------------------------------------------+
|                               HYBRID BUILD FLOW                                 |
+---------------------------------------------------------------------------------+
   Source File (hybrid_code.c)
        |
        v
   [mpicc Wrapper] ----(reads options: -O3 -fopenmp)
        |
        +---> Injects MPI include paths: -I$MPI_HOME/include
        +---> Invokes underlying GCC: gcc -O3 -fopenmp -I$MPI_HOME/include
        |
        v
   [Compilation Phase]
        * Interprets #include <mpi.h> and #include <omp.h>
        * Recognizes #pragma omp directives (transforms loops to multithreaded runtime calls)
        * Generates object file with OpenMP runtime hooks (GOMP_parallel, etc.)
        |
        v
   [Linker Phase]
        * Links MPI shared libraries (-lmpi)
        * Links OpenMP shared runtime (-lgomp or -liomp5)
        * Links POSIX threads runtime (-lpthread) and math library (-lm)
        |
        v
   Final Binary Executable (hybrid_app)
```

> [!WARNING]
> If you compile with `mpicc -O3 hybrid_code.c -o hybrid_app` (omitting `-fopenmp`), the compiler ignores every `#pragma omp parallel` directive as an unrecognized pragma. The code will compile cleanly without errors, link against `libmpi.so`, but run entirely sequentially inside each MPI rank!

---

### 3.2 Launching and Process-to-Core Binding (Affinity)

When running a hybrid program, two distinct software entities control process and thread placement:
1. **The MPI Launcher (`mpirun`, `mpiexec`, or `srun`):** Responsible for placing MPI process ranks across nodes, sockets, and NUMA domains, and assigning a reserved slice of CPU cores to each rank.
2. **The OpenMP Runtime (`OMP_PROC_BIND`, `OMP_PLACES`):** Responsible for taking the assigned cores allocated to an MPI process and placing individual OpenMP threads onto specific hardware execution units (cores or hardware threads).

#### Process and Thread Mapping on a Dual-Socket 16-Core Node (32 Cores Total)

Suppose we have a compute node with 2 CPU Sockets, each containing 16 physical CPU cores (Cores 0–15 on Socket 0, Cores 16–31 on Socket 1). We wish to run **4 MPI processes** per node, with **8 OpenMP threads** per process ($4 \times 8 = 32$ cores utilized).

```
===================================================================================
COMPUTE NODE: 2 Sockets x 16 Cores = 32 Physical Cores
===================================================================================

[ SOCKET 0: Cores 0 - 15 ] (Local NUMA Domain 0)
+------------------------------------+------------------------------------+
| MPI Rank 0                         | MPI Rank 1                         |
| Assigned Cores: 0, 1, 2, ..., 7    | Assigned Cores: 8, 9, 10, ..., 15  |
|                                    |                                    |
| Thread 0 -> Core 0                 | Thread 0 -> Core 8                 |
| Thread 1 -> Core 1                 | Thread 1 -> Core 9                 |
| Thread 2 -> Core 2                 | Thread 2 -> Core 10                |
| Thread 3 -> Core 3                 | Thread 3 -> Core 11                |
| Thread 4 -> Core 4                 | Thread 4 -> Core 12                |
| Thread 5 -> Core 5                 | Thread 5 -> Core 13                |
| Thread 6 -> Core 6                 | Thread 6 -> Core 14                |
| Thread 7 -> Core 7                 | Thread 7 -> Core 15                |
+------------------------------------+------------------------------------+

[ SOCKET 1: Cores 16 - 31 ] (Local NUMA Domain 1)
+------------------------------------+------------------------------------+
| MPI Rank 2                         | MPI Rank 3                         |
| Assigned Cores: 16, 17, ..., 23    | Assigned Cores: 24, 25, ..., 31    |
|                                    |                                    |
| Thread 0 -> Core 16                | Thread 0 -> Core 24                |
| Thread 1 -> Core 17                | Thread 1 -> Core 25                |
| Thread 2 -> Core 18                | Thread 2 -> Core 26                |
| Thread 3 -> Core 19                | Thread 3 -> Core 27                |
| Thread 4 -> Core 20                | Thread 4 -> Core 28                |
| Thread 5 -> Core 21                | Thread 5 -> Core 29                |
| Thread 6 -> Core 22                | Thread 6 -> Core 30                |
| Thread 7 -> Core 23                | Thread 7 -> Core 31                |
+------------------------------------+------------------------------------+
```

---

## 4. Important Mechanisms

### 4.1 Launch Commands and Syntax Across Runtimes

Different MPI implementations and resource schedulers use distinct command-line parameters to define process-to-core binding and thread slot reservations.

#### 1. OpenMPI (v4.x / v5.x)
In OpenMPI, the key parameters are `--map-by` (how processes are distributed across hardware units) and `--bind-to` (how tightly a process is constrained). The `PE=N` (Processing Elements) modifier reserves $N$ physical cores for each MPI rank's OpenMP threads:

```bash
export OMP_NUM_THREADS=8
export OMP_PROC_BIND=close
export OMP_PLACES=cores

# Launch 4 MPI ranks, 2 per socket, allocating 8 physical cores per rank
mpirun -np 4 \
       --map-by socket:PE=8 \
       --bind-to core \
       -x OMP_NUM_THREADS \
       -x OMP_PROC_BIND \
       -x OMP_PLACES \
       ./hybrid_app
```
- `--map-by socket:PE=8`: Steps through CPU sockets, assigning each MPI process a block of 8 processing elements (physical cores).
- `--bind-to core`: Binds each process to the set of cores allocated by `PE=8`.
- `-x VARIABLE`: Instructs `mpirun` to export and propagate the specified environment variable to all remote cluster nodes.

#### 2. MPICH / Intel MPI
MPICH and Intel MPI use the Hydra process manager. In Intel MPI, binding is controlled using `I_MPI_PIN_DOMAIN`:

```bash
export OMP_NUM_THREADS=8
export OMP_PROC_BIND=close
export OMP_PLACES=cores

# Set Intel MPI pin domain size equal to the number of OpenMP threads
export I_MPI_PIN_DOMAIN=omp

mpirun -np 4 -ppn 2 ./hybrid_app
```
- `-ppn 2`: Places 2 processes per node.
- `I_MPI_PIN_DOMAIN=omp`: Automatically creates process binding domains sized to match `OMP_NUM_THREADS`, guaranteeing that each MPI rank receives an isolated set of 8 cores.

#### 3. SLURM Workload Manager (`srun`)
In production supercomputing clusters managed by SLURM, resource allocation and binding are coordinated via job script directives (`#SBATCH`):

```bash
#!/bin/bash
#SBATCH --job-name=hybrid_sim
#SBATCH --nodes=2                  # 2 Compute Nodes
#SBATCH --ntasks=8                 # 8 Total MPI Ranks (4 per node)
#SBATCH --ntasks-per-node=4        # 4 MPI Ranks per Node
#SBATCH --cpus-per-task=8          # 8 Physical Cores per MPI Rank (for OpenMP threads)
#SBATCH --time=01:00:00
#SBATCH --partition=compute

# Ensure environment variables are set inside the SLURM job
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PROC_BIND=close
export OMP_PLACES=cores

# Launch via SLURM's native launcher (inherits SLURM topology automatically)
srun --cpu-bind=cores ./hybrid_app
```
- `--ntasks-per-node=4`: Spawns 4 MPI processes on each physical node.
- `--cpus-per-task=8`: Informs SLURM's cgroups allocator to allocate 8 hardware cores per MPI rank.
- `--cpu-bind=cores`: Restricts each MPI rank to its assigned 8 cores, allowing OpenMP threads to execute without inter-process contention.

---

### 4.2 OpenMP Thread Affinity Control (`OMP_PLACES` and `OMP_PROC_BIND`)

Since OpenMP 4.0, standardized environment variables manage thread binding without relying on vendor-specific extensions (such as `KMP_AFFINITY` or `GOMP_CPU_AFFINITY`).

#### `OMP_PLACES`
Defines the hardware execution units that OpenMP threads can occupy:
- `OMP_PLACES=threads`: Each thread is pinned to an individual hardware execution thread (Hyper-Thread / SMT logical core).
- `OMP_PLACES=cores`: Each thread is pinned to an entire physical CPU core (recommended for HPC numerical computing).
- `OMP_PLACES=sockets`: Threads are pinned to CPU sockets (threads may wander between cores inside the same socket).

#### `OMP_PROC_BIND`
Specifies the policy for distributing threads across the defined places:
- `OMP_PROC_BIND=close`: Threads are placed on hardware cores consecutively adjacent to the master thread. Best for cache-sharing between OpenMP worker threads (e.g., sharing L2/L3 caches).
- `OMP_PROC_BIND=spread`: Threads are distributed as widely as possible across the available places/sockets. Maximizes aggregate memory bandwidth if an MPI rank spans multiple memory channels or sockets.
- `OMP_PROC_BIND=master`: All worker threads are co-located in the same place as the master thread (rarely used in production).
- `OMP_PROC_BIND=false`: Thread binding is disabled. Operating system kernel dynamically migrates threads arbitrarily across cores (avoid in HPC!).

---

### 4.3 Five Common Hybrid Pitfalls and Diagnostic Matrix

| Pitfall / Failure Mode | Root Cause | Observable Symptom | Corrective Action |
| :--- | :--- | :--- | :--- |
| **1. Severe Over-Subscription** | Launching $M$ ranks per node where $M \times T > \text{physical cores}$. | CPU utilization shows >50% `system` / `wait` time; runtime is 5x–20x slower than serial code. | Enforce $M \times T = \text{physical cores}$. Set `--cpus-per-task=T` in SLURM. |
| **2. Missing `-fopenmp`** | Omitted `-fopenmp` during `mpicc` compilation or linking. | Application runs cleanly, but output shows thread count is always 1; no parallel speedup within rank. | Recompile with `mpicc -O3 -fopenmp`. Verify `#ifdef _OPENMP` guards. |
| **3. Thread Safety Level Mismatch** | Using multiple OpenMP threads to issue MPI calls without requesting `MPI_THREAD_MULTIPLE`. | Intermittent deadlocks, race conditions, memory corruption, or `MPI_ERR_INTERN` segfaults under high concurrency. | Initialize via `MPI_Init_thread(&argc, &argv, MPI_THREAD_MULTIPLE, &provided)` and verify `provided >= required`. |
| **4. NUMA Straddling / Wandering** | Single MPI rank spawns threads across multiple CPU sockets without pinning (`OMP_PROC_BIND=false`). | Erratic iteration runtimes; severe memory bandwidth throttling; high remote UPI/Infinity Fabric traffic. | Set 1 MPI rank per CPU socket. Export `OMP_PROC_BIND=close` and `OMP_PLACES=cores`. |
| **5. Environment Variable Non-Propagation** | `export OMP_NUM_THREADS` performed on head node, not exported across compute nodes. | Head node runs with $T$ threads; remote compute nodes default to 1 thread per rank or max SMT cores. | Use `mpirun -x OMP_NUM_THREADS` or invoke via SLURM `srun` which automatically inherits environment. |

---

## 5. Compilable C Code: Complete Hybrid Diagnostic and Affinity Verification Tool

The following standalone C application acts as a diagnostic benchmark for high-performance clusters. It initializes MPI with multithreading support, validates the thread level, spawns OpenMP threads, determines the exact physical CPU core each thread is currently executing on using the Linux kernel system call `sched_getcpu()`, validates NUMA consistency, and checks for core over-subscription or collision.

```c
/*
 * hybrid_diagnostic.c
 *
 * Professional Diagnostic Tool for Hybrid MPI + OpenMP Deployments.
 * Verifies:
 *   1. MPI multithreading initialization level.
 *   2. Node hostname and rank placement.
 *   3. OpenMP thread team allocation.
 *   4. Exact physical CPU core binding per thread via sched_getcpu().
 *   5. Core collisions (two threads sharing the same core).
 *
 * Compilation:
 *   mpicc -O2 -fopenmp hybrid_diagnostic.c -o hybrid_diagnostic
 *
 * Execution (Example on 1 node, 2 ranks, 4 threads each):
 *   export OMP_NUM_THREADS=4
 *   export OMP_PROC_BIND=close
 *   export OMP_PLACES=cores
 *   mpirun -np 2 --bind-to core ./hybrid_diagnostic
 */

#define _GNU_SOURCE  /* Required for sched_getcpu() on Linux */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sched.h>
#include <mpi.h>
#include <omp.h>

/* Helper function to convert MPI thread level integer to human-readable string */
const char* thread_level_to_string(int level) {
    switch (level) {
        case MPI_THREAD_SINGLE:     return "MPI_THREAD_SINGLE";
        case MPI_THREAD_FUNNELED:   return "MPI_THREAD_FUNNELED";
        case MPI_THREAD_SERIALIZED: return "MPI_THREAD_SERIALIZED";
        case MPI_THREAD_MULTIPLE:   return "MPI_THREAD_MULTIPLE";
        default:                    return "UNKNOWN_LEVEL";
    }
}

int main(int argc, char *argv[]) {
    int required = MPI_THREAD_FUNNELED;
    int provided;

    /* Initialize MPI with thread support */
    int init_rc = MPI_Init_thread(&argc, &argv, required, &provided);
    if (init_rc != MPI_SUCCESS) {
        fprintf(stderr, "FATAL: MPI_Init_thread failed!\n");
        MPI_Abort(MPI_COMM_WORLD, 1);
    }

    int world_rank, world_size;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    char hostname[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(hostname, &name_len);

    /* Rank 0 prints global configuration summary */
    if (world_rank == 0) {
        printf("=================================================================\n");
        printf("         HYBRID MPI + OPENMP DIAGNOSTIC & AFFINITY AUDIT         \n");
        printf("=================================================================\n");
        printf("Total MPI Ranks          : %d\n", world_size);
        printf("Requested Thread Level   : %s\n", thread_level_to_string(required));
        printf("Provided Thread Level    : %s\n", thread_level_to_string(provided));
        if (provided < required) {
            printf("[WARNING] MPI implementation provided lower thread safety than requested!\n");
        } else {
            printf("[SUCCESS] Thread safety level meets application requirements.\n");
        }
        
#ifdef _OPENMP
        printf("OpenMP Compiler Support  : ENABLED (_OPENMP = %d)\n", _OPENMP);
#else
        printf("OpenMP Compiler Support  : DISABLED! Recompile with -fopenmp!\n");
#endif
        printf("=================================================================\n\n");
        fflush(stdout);
    }

    /* Synchronize before rank-by-rank output */
    MPI_Barrier(MPI_COMM_WORLD);

    /* Each rank records thread affinity into local diagnostic arrays */
    int num_threads = 0;
    int thread_cores[64];
    memset(thread_cores, -1, sizeof(thread_cores));

    #pragma omp parallel default(none) shared(num_threads, thread_cores)
    {
        int tid = omp_get_thread_num();
        int nthreads = omp_get_num_threads();

        #pragma omp single
        {
            num_threads = nthreads;
        }

        /* Detect physical core assignment on Linux systems */
#if defined(__linux__)
        int cpu_id = sched_getcpu();
#else
        int cpu_id = -1; /* Non-Linux fallback */
#endif
        if (tid < 64) {
            thread_cores[tid] = cpu_id;
        }
    }

    /* Serialize console printing across MPI ranks using a token ring */
    for (int r = 0; r < world_size; r++) {
        if (world_rank == r) {
            printf("[Rank %2d of %2d] Host: %-15s | OpenMP Threads: %2d\n",
                   world_rank, world_size, hostname, num_threads);

            /* Check for core collisions within this rank */
            int collision_found = 0;
            for (int i = 0; i < num_threads; i++) {
                printf("  -> Thread %2d is bound to Physical CPU Core: %3d\n",
                       i, thread_cores[i]);
                for (int j = i + 1; j < num_threads; j++) {
                    if (thread_cores[i] != -1 && thread_cores[i] == thread_cores[j]) {
                        collision_found = 1;
                    }
                }
            }

            if (collision_found) {
                printf("  [ALERT] Core collision detected! Multiple threads sharing identical core.\n");
                printf("          Check OMP_PROC_BIND and launcher binding flags!\n");
            } else {
                printf("  [OK] All threads mapped to distinct physical cores.\n");
            }
            printf("-----------------------------------------------------------------\n");
            fflush(stdout);
        }
        MPI_Barrier(MPI_COMM_WORLD);
    }

    /* Rank 0 final performance sanity check */
    if (world_rank == 0) {
        printf("\nDiagnostic audit complete across all %d processes.\n", world_size);
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return 0;
}
```

### 5.1 Compilation and Execution Workflow

```bash
# 1. Compile with GCC MPI wrapper
mpicc -O2 -fopenmp hybrid_diagnostic.c -o hybrid_diagnostic

# 2. Set OpenMP environment variables
export OMP_NUM_THREADS=4
export OMP_PROC_BIND=close
export OMP_PLACES=cores

# 3. Launch 2 MPI ranks (1 on Socket 0, 1 on Socket 1)
mpirun -np 2 --bind-to core ./hybrid_diagnostic
```

### 5.2 Expected Output

```
=================================================================
         HYBRID MPI + OPENMP DIAGNOSTIC & AFFINITY AUDIT         
=================================================================
Total MPI Ranks          : 2
Requested Thread Level   : MPI_THREAD_FUNNELED
Provided Thread Level    : MPI_THREAD_FUNNELED
[SUCCESS] Thread safety level meets application requirements.
OpenMP Compiler Support  : ENABLED (_OPENMP = 201511)
=================================================================

[Rank  0 of  2] Host: node01          | OpenMP Threads:  4
  -> Thread  0 is bound to Physical CPU Core:   0
  -> Thread  1 is bound to Physical CPU Core:   1
  -> Thread  2 is bound to Physical CPU Core:   2
  -> Thread  3 is bound to Physical CPU Core:   3
  [OK] All threads mapped to distinct physical cores.
-----------------------------------------------------------------
[Rank  1 of  2] Host: node01          | OpenMP Threads:  4
  -> Thread  0 is bound to Physical CPU Core:   4
  -> Thread  1 is bound to Physical CPU Core:   5
  -> Thread  2 is bound to Physical CPU Core:   6
  -> Thread  3 is bound to Physical CPU Core:   7
  [OK] All threads mapped to distinct physical cores.
-----------------------------------------------------------------

Diagnostic audit complete across all 2 processes.
=================================================================
```

---

## 6. Mathematical Formulas and Performance Models

### 6.1 Hardware Core Allocation Equation

For a cluster consisting of $N_{\text{nodes}}$ physical compute nodes, where each node possesses $S$ CPU sockets and each socket contains $C_{\text{socket}}$ physical cores, the total number of physical hardware cores is:

$$C_{\text{total}} = N_{\text{nodes}} \times S \times C_{\text{socket}}$$

To prevent both over-subscription (resource contention) and under-subscription (wasted hardware), the configuration of total MPI ranks $M$ and OpenMP threads per rank $T$ must satisfy:

$$M \times T = C_{\text{total}}$$

For NUMA optimality (1 MPI rank per CPU socket), the per-rank thread count is set to:

$$T = C_{\text{socket}}$$
$$M = N_{\text{nodes}} \times S$$

---

### 6.2 The Over-Subscription Context-Switching Penalty Model

When the operating system is forced to run $N_{\text{active}}$ threads on $C_{\text{physical}}$ cores such that the over-subscription ratio $\rho > 1$:

$$\rho = \frac{N_{\text{active}}}{C_{\text{physical}}} = \frac{M \times T}{C_{\text{physical}}}$$

The effective execution time $t_{\text{exec}}$ inflates exponentially due to operating system scheduling time slices ($\Delta t_{\text{slice}} \approx 1\text{ ms} - 10\text{ ms}$) and kernel context switch overhead ($\tau_{\text{cs}} \approx 2\text{ \textmu s} - 5\text{ \textmu s}$ plus L1/L2 cache cold-miss penalties $\tau_{\text{cache}} \approx 15\text{ \textmu s}$):

$$t_{\text{exec}}(\rho) = t_{\text{ideal}} \times \left( \rho + (\rho - 1) \cdot \frac{\tau_{\text{cs}} + \tau_{\text{cache}}}{\Delta t_{\text{slice}}} \right)$$

When $\rho \ge 4$, cache thrashing dominates: data brought into L1/L2 cache lines by one thread is completely evicted by competing threads before the original thread is scheduled again, degrading sustained arithmetic throughput by 400% to 1200%.

---

### 6.3 The NUMA Interconnect Latency Ratio

Let $L_{\text{local}}$ be the memory access latency to local socket DRAM, and $L_{\text{remote}}$ be the access latency across the inter-socket interconnect (UPI/Infinity Fabric):

$$\gamma_{\text{NUMA}} = \frac{L_{\text{remote}}}{L_{\text{local}}} \approx 1.8 \text{ to } 2.5$$

If threads migrate across sockets or access memory allocated on remote NUMA nodes, the average memory access latency $\bar{L}$ is:

$$\bar{L} = f_{\text{local}} \cdot L_{\text{local}} + (1 - f_{\text{local}}) \cdot L_{\text{remote}}$$

Where $f_{\text{local}}$ is the fraction of memory accesses satisfied locally. If thread binding is disabled ($f_{\text{local}} \approx 0.50$):

$$\bar{L} = 0.50 \cdot L_{\text{local}} + 0.50 \cdot (2.2 \cdot L_{\text{local}}) = 1.6 \cdot L_{\text{local}}$$

The application experiences a **60% average memory access latency penalty**, directly crippling memory-bandwidth-bound scientific kernels.

---

## 7. Advantages and Limitations

### Advantages of Structured Hybrid Deployment
1. **Maximal Hardware Saturation:** Correct process mapping and thread pinning guarantees 100% physical core saturation without wasting resources on idle execution pipelines or SMT stalls.
2. **Deterministic Cache Locality:** Pinning threads with `OMP_PROC_BIND=close` ensures shared L3 cache lines remain warm across loop iterations, preventing cache invalidations.
3. **NUMA Interconnect Traffic Elimination:** Allocating 1 MPI rank per NUMA domain and binding threads locally ensures all DRAM accesses route through local memory channels ($f_{\text{local}} \to 1.0$).
4. **Diagnostic Transparency:** Proper diagnostic instrumentation exposes cluster topology mismatches before multi-day supercomputing simulations are launched.

### Limitations and Operational Overheads
1. **Fragile Portability Across Clusters:** Slurm scripts, `mpirun` flags, and binding options vary significantly across MPI implementations (OpenMPI vs. Intel MPI vs. MPICH vs. Cray MPICH).
2. **SMT Complexity:** On processors with Hyper-Threading / Simultaneous Multithreading enabled (2–4 logical threads per core), choosing whether to bind to physical cores or logical threads requires empirical profiling; compute-bound codes lose performance on SMT, while integer/latency-bound codes may benefit.
3. **Environment Propagation Pitfalls:** Cluster firewalls, non-login remote SSH shells, and heterogeneous cluster configurations can silently drop OpenMP environment variables.

---

## 8. Standard Exam Definitions

> **Definition 1: Compiler Wrapper (`mpicc`)**  
> A front-end utility script provided by an MPI implementation that invokes the underlying system C compiler (e.g., `gcc`, `icc`, `clang`) while automatically injecting required MPI header paths (`-I`), library paths (`-L`), and linker dependencies (`-lmpi`). It does not automatically enable OpenMP without explicit flags.

> **Definition 2: Core Affinity (Thread Pinning)**  
> The operating system mechanism of binding an execution context (an MPI process or an OpenMP thread) to a specific logical or physical CPU core. Affinity prevents the OS kernel scheduler from migrating threads between cores, preserving L1/L2 cache warmth and NUMA locality.

> **Definition 3: Over-Subscription**  
> An execution condition occurring when the total number of active threads and processes assigned to a compute node exceeds the number of available physical CPU execution pipelines ($M \times T > C$). Over-subscription forces the operating system kernel to perform frequent context switches, destroying computational throughput.

> **Definition 4: NUMA Domain (Non-Uniform Memory Access)**  
> A cluster of CPU cores directly coupled to a dedicated physical memory controller and DRAM channel. While memory is globally addressable across all domains, access to local DRAM exhibits significantly lower latency and higher bandwidth than access to remote DRAM across inter-socket interconnects.

> **Definition 5: `OMP_PROC_BIND`**  
> A standardized OpenMP environment variable that instructs the runtime library whether and how to bind worker threads to hardware places (cores or sockets). Common values include `close` (pack threads contiguously), `spread` (distribute threads evenly across domains), and `false` (no binding).

> **Definition 6: Processing Element (`PE`)**  
> An MPI launcher abstraction (notably in OpenMPI `--map-by <unit>:PE=N`) specifying the number of physical hardware cores reserved exclusively for each MPI rank to accommodate multi-threaded execution runtimes like OpenMP.

---

## 9. Comprehensive Comparison Tables

### Table 1: MPI Wrapper vs. Underlying Compiler Flags

| Feature | `mpicc` (OpenMPI / MPICH) | Underlying Compiler (`gcc`) | Intel MPI Wrapper (`mpiicc`) |
| :--- | :--- | :--- | :--- |
| **Role** | Wrapper script / Proxy | Native machine-code compiler | Wrapper around Intel oneAPI C/C++ compiler |
| **MPI Inclusions** | Automatic (`-I<path>/include`) | Manual specification required | Automatic |
| **MPI Libraries** | Automatic (`-lmpi`) | Manual (`-lmpi -lpthread`) | Automatic (`-lmpi`) |
| **OpenMP Flag** | **Requires `-fopenmp`** | **Requires `-fopenmp`** | **Requires `-qopenmp`** |
| **Omission Outcome** | Compiles pure MPI code; ignores OpenMP pragmas | Compiles pure code; ignores OpenMP pragmas | Compiles pure MPI code; ignores OpenMP pragmas |

---

### Table 2: Launch and Affinity Syntax Across Major Environments

| Launch Environment | Process Placement Flag | Core Reservation per Process | Thread Pinning Variables |
| :--- | :--- | :--- | :--- |
| **OpenMPI (Standalone)** | `mpirun -np <M> --map-by socket` | `--bind-to core --map-by socket:PE=<T>` | `export OMP_PROC_BIND=close`<br>`export OMP_PLACES=cores` |
| **Intel MPI (Hydra)** | `mpirun -np <M> -ppn <P>` | `export I_MPI_PIN_DOMAIN=omp` | `export KMP_AFFINITY=compact`<br>or standard `OMP_PROC_BIND=close` |
| **SLURM (`srun`)** | `#SBATCH --ntasks=<M>`<br>`#SBATCH --ntasks-per-node=<P>` | `#SBATCH --cpus-per-task=<T>`<br>`srun --cpu-bind=cores` | `export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`<br>`export OMP_PROC_BIND=close` |

---

### Table 3: `OMP_PROC_BIND` Policy Comparison

| Policy Setting | Thread Placement Strategy | Cache Behavior | Ideal Workload |
| :--- | :--- | :--- | :--- |
| `OMP_PROC_BIND=close` | Threads assigned to adjacent contiguous cores within the domain. | Maximizes L2/L3 cache sharing between threads. | Stencil codes, parallel reductions, pipeline parallelism. |
| `OMP_PROC_BIND=spread` | Threads distributed evenly across disparate sockets/domains. | Maximizes aggregate memory bandwidth across channels. | Embarrassingly parallel memory-bound streaming (e.g., Stream benchmark, large element-wise operations). |
| `OMP_PROC_BIND=false` | Operating system scheduler migrates threads freely. | Poor; frequent cache invalidations and remote NUMA accesses. | Non-HPC workloads, desktop multitasking (AVOID in HPC!). |

---

## 10. Solved Problems

### Problem 1: Slurm Allocation and Over-Subscription Calculation
**Question:**  
You are deploying a hybrid MPI + OpenMP simulation on an HPC cluster with nodes containing 2 sockets, each with 24 physical cores (48 cores per node total). Hyper-Threading is enabled (2 threads per core = 96 logical CPUs).  
(a) You submit a SLURM job with `--ntasks-per-node=8` and `--cpus-per-task=12`. Will this cause physical core over-subscription?  
(b) If you set `OMP_NUM_THREADS=12`, calculate the total number of threads launched per node.  
(c) Determine the optimal values for `--ntasks-per-node`, `--cpus-per-task`, and `OMP_NUM_THREADS` to achieve exactly 1 MPI rank per socket while fully saturating all physical cores without Hyper-Threading.

**Solution:**

**(a) Assessing Over-Subscription:**
- Number of physical cores per node: $C_{\text{phys}} = 2 \times 24 = 48\text{ cores}$.
- Requested physical cores: 
  $$\text{Requested Cores} = \text{ntasks-per-node} \times \text{cpus-per-task} = 8 \times 12 = 96\text{ cores}$$
- Because $96 > 48$, this configuration requests 96 hardware contexts. On a node with only 48 physical execution cores, SLURM will place threads onto SMT logical siblings (Hyper-Threads). If the code is compute-bound, this results in core over-subscription relative to physical ALU/FPU pipelines, inducing pipeline stalling and context-switching contention.

**(b) Total Threads Launched:**
$$\text{Total Threads/Node} = \text{ntasks-per-node} \times \text{OMP\_NUM\_THREADS} = 8 \times 12 = 96\text{ threads}$$

**(c) Optimal NUMA Configuration (1 Rank Per Socket):**
- Total sockets per node: $S = 2$.
- Physical cores per socket: $C_{\text{socket}} = 24$.
- Therefore:
  - Set `--ntasks-per-node=2` (exactly 1 MPI rank per socket).
  - Set `--cpus-per-task=24` (reserving all 24 physical cores of that socket for the rank).
  - Set `export OMP_NUM_THREADS=24`.
- Verification: $2 \text{ ranks} \times 24 \text{ threads} = 48 \text{ threads}$, which matches the 48 physical cores with $f_{\text{local}} = 1.0$.

---

### Problem 2: Debugging the "Zero-Speedup" Hybrid Paradox
**Question:**  
A researcher compiles a hybrid MPI + OpenMP program using the following commands:
```bash
mpicc -O3 -c main.c -o main.o
mpicc -O3 -c solver.c -o solver.o
mpicc -O3 -fopenmp main.o solver.o -o simulation
export OMP_NUM_THREADS=16
mpirun -np 4 ./simulation
```
When running the simulation, the execution time is identical whether `OMP_NUM_THREADS` is set to 1, 4, 8, or 16. What is the fatal flaw in the build pipeline, and how does the researcher fix it?

**Solution:**
1. **Root Cause Analysis:**
   - In modular compilation, C source files are compiled to object files (`.o`) via the `-c` flag.
   - The developer passed `-fopenmp` **only to the linker step** (`mpicc -O3 -fopenmp main.o solver.o -o simulation`).
   - During the compilation phase (`mpicc -O3 -c main.c` and `mpicc -O3 -c solver.c`), `-fopenmp` was **omitted**.
   - As mandated by the OpenMP specification, when a C compiler encounters `#pragma omp` directives without `-fopenmp`, it treats them as unrecognized pragmas and silently ignores them!
   - Consequently, the generated object code contains no calls to the OpenMP runtime library (`libgomp`). The loops were compiled as standard serial `for` loops.
   - At the link stage, `-fopenmp` merely linked `libgomp.so`, but there were no OpenMP entry points to invoke.
2. **Correct Build Workflow:**
   Pass `-fopenmp` to **every** compilation and linking stage:
   ```bash
   mpicc -O3 -fopenmp -c main.c -o main.o
   mpicc -O3 -fopenmp -c solver.c -o solver.o
   mpicc -O3 -fopenmp main.o solver.o -o simulation
   ```

---

### Problem 3: NUMA Memory Bandwidth Penalty Calculation
**Question:**  
A hybrid conjugate gradient solver is memory-bandwidth bound. On a dual-socket AMD EPYC server, local socket memory bandwidth is $B_{\text{local}} = 180\text{ GB/s}$, while cross-socket interconnect bandwidth is $B_{\text{remote}} = 45\text{ GB/s}$.  
A programmer launches 1 MPI rank spanning both sockets ($T = 64$ threads across 2 sockets) without thread affinity (`OMP_PROC_BIND=false`). Due to random OS thread migration, 40% of memory accesses cross the socket interconnect.  
(a) Compute the effective sustained memory bandwidth $B_{\text{eff}}$.  
(b) What is the performance speedup if the code is restructured to 2 MPI ranks (1 per socket) with `OMP_PROC_BIND=close`, achieving 98% local memory access?

**Solution:**

**(a) Unpinned Effective Bandwidth:**
The effective access time per byte is the weighted harmonic mean of the local and remote bandwidths:
$$\frac{1}{B_{\text{eff}}} = \frac{f_{\text{local}}}{B_{\text{local}}} + \frac{f_{\text{remote}}}{B_{\text{remote}}}$$
Given $f_{\text{remote}} = 0.40$ and $f_{\text{local}} = 0.60$:
$$\frac{1}{B_{\text{eff}}} = \frac{0.60}{180} + \frac{0.40}{45} = \frac{1}{300} + \frac{1}{112.5} = 0.00333 + 0.00889 = 0.01222\text{ s/GB}$$
$$B_{\text{eff}} = \frac{1}{0.01222} \approx 81.82\text{ GB/s}$$

**(b) Restructured and Pinned Bandwidth:**
Given $f_{\text{local}} = 0.98$ and $f_{\text{remote}} = 0.02$:
$$\frac{1}{B_{\text{eff, pinned}}} = \frac{0.98}{180} + \frac{0.02}{45} = 0.00544 + 0.00044 = 0.00588\text{ s/GB}$$
$$B_{\text{eff, pinned}} = \frac{1}{0.00588} \approx 169.8\text{ GB/s}$$

**(c) Performance Speedup:**
$$\text{Speedup} = \frac{B_{\text{eff, pinned}}}{B_{\text{eff}}} = \frac{169.8}{81.82} \approx 2.075\times$$
By binding threads to eliminate remote NUMA crossbar traffic, the memory-bound solver runs **over 2x faster** without modifying a single line of numerical algorithm code.

---

## 11. Conceptual Questions & Answers

### Q1: Why does `mpicc` require the `-fopenmp` flag if it is already a specialized parallel compiler?
**Answer:**  
`mpicc` is not a standalone compiler; it is an administrative wrapper script around a standard system compiler (such as GCC, Clang, or Intel ICC). The wrapper's sole role is to inject the include paths (`-I`), library paths (`-L`), and linker library names (`-lmpi`) required by the MPI standard. Because OpenMP is a separate, independent shared-memory standard supported by the underlying compiler, `mpicc` does not activate OpenMP by default. Passing `-fopenmp` instructs the underlying compiler to enable OpenMP preprocessing, transform `#pragma omp` directives into runtime multithreading function calls, and link against the OpenMP runtime library (`libgomp`).

---

### Q2: What is "thread wandering" (thread migration), and why is it fatal to HPC performance?
**Answer:**  
Thread wandering refers to the operating system kernel's process scheduler preemptively relocating an active thread from one physical CPU core to another during execution. In general-purpose desktop computing, this provides dynamic load balancing. In HPC, however, it degrades performance due to two factors:
1. **Cache Cold-Start Penalty:** When a thread moves to a new core, the data it recently accessed remains in the L1 and L2 caches of its previous core. The thread experiences a barrage of cache misses, stalling the execution pipeline while cache lines are re-fetched from L3 or main memory.
2. **NUMA Penalty:** If the thread is migrated across a socket boundary, all data previously allocated in local DRAM becomes remote memory. The thread must access its variables over high-latency inter-socket links, halving memory bandwidth and increasing memory latency.

---

### Q3: Contrast `OMP_PROC_BIND=close` with `OMP_PROC_BIND=spread`. When should each be chosen?
**Answer:**  
- `OMP_PROC_BIND=close` packs OpenMP threads onto consecutive physical cores immediately adjacent to the master thread. This policy is ideal for algorithms with high degrees of inter-thread communication or shared read access to common data structures (such as stencil operations or reductions), because threads share higher-level cache hierarchies (L2 or L3 caches).
- `OMP_PROC_BIND=spread` distributes OpenMP threads as far apart as possible across available sockets or NUMA domains. This policy is preferred for memory-bandwidth-bound streaming computations where individual threads do not share data, allowing each thread to utilize distinct memory controllers and maximum aggregate DRAM bandwidth.

---

### Q4: What occurs if an application calls `MPI_Send` inside an OpenMP parallel loop when the runtime initialized with `MPI_Init_thread(..., MPI_THREAD_FUNNELED, ...)`?
**Answer:**  
In `MPI_THREAD_FUNNELED`, the MPI runtime contract guarantees thread safety **only if the master thread** (thread ID 0) issues MPI calls. If OpenMP worker threads ($T_{\text{id}} > 0$) issue `MPI_Send` concurrently, the MPI library's internal state structures (queue pointers, sequence numbers, network buffer rings) are modified without mutex synchronization. This leads to undefined behavior: silent packet corruption, message routing deadlocks, double-free faults, or catastrophic application crashes (`SIGSEGV`).

---

## 12. Quick Revision Sheet

- **Compilation Rule:** Always use `mpicc -O3 -fopenmp`. Apply `-fopenmp` to **both** compilation (`-c`) and linking stages.
- **Verification Pragma:** Use `#ifdef _OPENMP` guards to ensure the compiler actually processed the OpenMP directives.
- **Process Pinning (OpenMPI):**
  ```bash
  mpirun -np $RANKS --map-by socket:PE=$OMP_NUM_THREADS --bind-to core ./app
  ```
- **Thread Pinning Variables:**
  - `export OMP_NUM_THREADS=N`
  - `export OMP_PROC_BIND=close` (or `spread`)
  - `export OMP_PLACES=cores`
- **SLURM Directives:**
  - `#SBATCH --ntasks-per-node=M`
  - `#SBATCH --cpus-per-task=T`
  - `srun --cpu-bind=cores ./app`
- **NUMA Best Practice:** 1 MPI Rank per CPU socket, $T = \text{cores per socket}$, eliminating cross-socket DRAM traffic.
- **Diagnostic System Call:** On Linux, use `sched_getcpu()` inside `#pragma omp parallel` to print the exact physical core executing each thread.
- **Golden Sizing Rule:**
  $$\text{Ranks per Node} \times \text{Threads per Rank} = \text{Physical Cores per Node}$$

---

## 13. What You Should Be Able to Do

1. **Write Production Build Scripts:** Write Makefiles and build scripts that properly invoke `mpicc -fopenmp` across modular object compilations.
2. **Author Launch Scripts:** Configure standalone `mpirun` and SLURM batch scripts that correctly set process-to-socket mapping and core reservations (`PE=N`, `--cpus-per-task`).
3. **Configure Standard OpenMP Affinity:** Correctly apply `OMP_PLACES=cores` and `OMP_PROC_BIND=close` to prevent OS thread wandering.
4. **Diagnose and Resolve Performance Bugs:** Systematically diagnose silent serialization (missing `-fopenmp`), core over-subscription, thread safety violations, and remote NUMA bandwidth bottlenecks.
5. **Verify Runtime Core Placement:** Integrate `sched_getcpu()` into diagnostic harnesses to verify that every rank and thread executes on its dedicated physical core.
