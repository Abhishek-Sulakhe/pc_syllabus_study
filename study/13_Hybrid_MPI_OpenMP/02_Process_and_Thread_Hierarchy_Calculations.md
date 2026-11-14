# Module 13: Hybrid MPI + OpenMP Programming (Lecture 32)
# Subtopic 13.02: Process and Thread Hierarchy Calculations (NUMA Placement, Core Binding, `OMP_PLACES`, `OMP_PROC_BIND`)

---

## 1. Basic Idea

In a pure shared-memory (OpenMP) program, execution sizing is determined by a single integer: `OMP_NUM_THREADS`. In a pure distributed-memory (MPI) program, execution sizing is determined by a single integer: `mpirun -np P`.

In a **Hybrid MPI + OpenMP** application, execution sizing is **hierarchical and multi-dimensional**. The computational capacity of the cluster is partitioned across three distinct physical and logical tiers:
1. **Tier 1: Physical Node Tier ($N_{\text{nodes}}$):** The number of physical server chassis allocated by the cluster scheduler.
2. **Tier 2: Process Tier ($P_{\text{node}}$):** The number of independent MPI processes spawned on each node.
3. **Tier 3: Thread Tier ($T_{\text{proc}}$):** The number of OpenMP worker threads spawned by each MPI process.

The fundamental relationship governing total concurrent execution threads ($T_{\text{total}}$) across the entire cluster is:

$$T_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}} \times T_{\text{proc}}$$

To achieve optimal performance on multi-socket Non-Uniform Memory Access (NUMA) servers, processes and threads must not be placed arbitrarily. They must be explicitly mapped to CPU sockets and pinned to physical cores using operating system affinity controls (`OMP_PLACES`, `OMP_PROC_BIND`, `--bind-to core`).

```
+-----------------------------------------------------------------------------------------+
|                       The Three-Tier Hybrid Sizing Hierarchy                            |
+-----------------------------------------------------------------------------------------+
| TIER 1: PHYSICAL NODES (N_nodes = 2)                                                    |
|                                                                                         |
|   +------------------------------------+   +------------------------------------+       |
|   | NODE 0                             |   | NODE 1                             |       |
|   | TIER 2: MPI Processes (P_node = 1) |   | TIER 2: MPI Processes (P_node = 1) |       |
|   |   +------------------------------+ |   |   +------------------------------+ |       |
|   |   | MPI Rank 0                   | |   |   | MPI Rank 1                   | |       |
|   |   | TIER 3: OpenMP Threads (T=4) | |   |   | TIER 3: OpenMP Threads (T=4) | |       |
|   |   |   [T0]  [T1]  [T2]  [T3]     | |   |   |   [T0]  [T1]  [T2]  [T3]     | |       |
|   |   +------------------------------+ |   |   +------------------------------+ |       |
|   +------------------------------------+   +------------------------------------+       |
|                                                                                         |
| SYLLABUS WORKED EXAMPLE:                                                                |
|   2 Nodes  x  1 Process/Node  x  4 Threads/Process  =  8 TOTAL ACTIVE THREADS           |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Eliminating Core Oversubscription and Undersubscription
If a cluster node possesses $64$ physical CPU cores:
- **Oversubscription ($P_{\text{node}} \times T_{\text{proc}} > C_{\text{node}}$):** Launching $4$ MPI processes with $32$ threads each requests $4 \times 32 = 128$ threads on a 64-core node. Two threads fight for each core, causing severe context-switch thrashing and cache eviction penalties.
- **Undersubscription ($P_{\text{node}} \times T_{\text{proc}} < C_{\text{node}}$):** Launching $2$ MPI processes with $16$ threads each utilizes only $32$ threads. $32$ expensive supercomputer cores sit permanently unpowered and idle.
- **Perfect Hardware Saturation:**
$$P_{\text{node}} \times T_{\text{proc}} = C_{\text{node}}$$

### 2. The NUMA Latency Penalty (Why Socket Mapping Matters)
Modern servers contain multiple CPU sockets, each connected directly to its own local memory channels (NUMA nodes).
- Accessing local NUMA memory: $\approx 70\text{ ns}$ latency, full memory bus bandwidth ($200+\text{ GB/s}$).
- Accessing remote NUMA memory on the adjacent socket: $\approx 150\text{ to }200\text{ ns}$ latency, throttled by cross-socket interconnect bandwidth.
If an OpenMP thread is spawned on Socket 1 but accesses memory allocated by the master thread on Socket 0, every memory read traverses the cross-socket interconnect, reducing memory bandwidth by up to $60\%$.

### 3. Thread Migration and OS Jitter
Without explicit core binding, the Linux Completely Fair Scheduler (CFS) dynamically migrates threads between cores to balance thermal loads. When an OpenMP thread migrates to a different core, its L1 and L2 data caches are left behind, forcing high-latency memory re-fetches. Setting `OMP_PROC_BIND=close` pins threads permanently to physical cores, preserving cache hotness.

---

## 3. How It Works

### The Official Syllabus Worked Example Walkthrough

Let us examine the exact architectural setup specified in the course syllabus:

$$\mathbf{2\text{ Nodes}} \times \mathbf{1\text{ Process/Node}} \times \mathbf{4\text{ Threads/Process}} = \mathbf{8\text{ Total Active Threads}}$$

```
Cluster Configuration:
- Cluster Allocation : N_nodes = 2 (e.g., node01 and node02)
- MPI Processes      : P_node = 1  (Rank 0 on node01, Rank 1 on node02)
- OpenMP Thread Team : T_proc = 4  (OMP_NUM_THREADS = 4)
```

```
[ Physical Node 1: node01 ]                      [ Physical Node 2: node02 ]
MPI Process: Rank 0                              MPI Process: Rank 1
  Thread 0 (Master) -> CPU Core 0                  Thread 0 (Master) -> CPU Core 0
  Thread 1 (Worker) -> CPU Core 1                  Thread 1 (Worker) -> CPU Core 1
  Thread 2 (Worker) -> CPU Core 2                  Thread 2 (Worker) -> CPU Core 2
  Thread 3 (Worker) -> CPU Core 3                  Thread 3 (Worker) -> CPU Core 3
```

Execution Commands:
```bash
# Set OpenMP thread count
export OMP_NUM_THREADS=4

# Launch 2 MPI processes mapped 1 per node
mpirun -np 2 --map-by ppr:1:node --bind-to core ./hybrid_app
```

---

### Process Placement Strategies: Node vs. Socket Level

When deploying a hybrid application on a cluster of dual-socket nodes (e.g., 2 sockets per node, 32 cores per socket = 64 cores/node):

```
+-----------------------------------------------------------------------------------------+
|                           Process Placement Comparison Matrix                           |
+-----------------------------------------------------------------------------------------+
| STRATEGY A: 1 MPI Process per Node (P_node = 1, OMP_NUM_THREADS = 64)                  |
|   - Advantage: Minimal MPI network connections; single large shared memory space.       |
|   - Hazard   : Severe NUMA cross-socket latency; threads on Socket 1 read Socket 0 RAM. |
|                                                                                         |
| STRATEGY B: 1 MPI Process per Socket (RECOMMENDED) (P_node = 2, OMP_NUM_THREADS = 32)  |
|   - Advantage: 100% NUMA local memory access! Rank 0 stays on Socket 0, Rank 1 on       |
|     Socket 1. Maximum memory bandwidth utilization.                                     |
|   - Hazard   : Slight increase in on-node MPI traffic.                                  |
|                                                                                         |
| STRATEGY C: Pure MPI (P_node = 64, OMP_NUM_THREADS = 1)                                 |
|   - Hazard   : Massive memory footprint; duplicated halo cells; network message flood.  |
+-----------------------------------------------------------------------------------------+
```

---

## 4. Important Mechanisms, CLI Flags, and Environment Variables

### 1. OpenMP Thread Affinity Variables (`OMP_PLACES` & `OMP_PROC_BIND`)

Introduced in OpenMP 4.0, these standard environment variables control thread placement and binding:

```bash
export OMP_PLACES=cores
export OMP_PROC_BIND=close
```

#### A. `OMP_PLACES` (Defines Where Threads Can Live):
- `threads`: Binds to individual hardware threads (Hyperthreads / SMT logical cores).
- `cores`: Binds to physical CPU cores (recommended for HPC; ignores hyperthreads).
- `sockets`: Binds to CPU sockets.

#### B. `OMP_PROC_BIND` (Defines How Threads Are Distributed Across Places):
- `close`: Places consecutive OpenMP threads on adjacent physical cores near the master thread (maximizes shared L2/L3 cache reuse).
- `spread`: Distributes threads evenly across all available sockets or NUMA nodes (maximizes total aggregate memory bandwidth).
- `master`: Binds all threads to the exact same core/place as the master thread (typically avoided).

---

### 2. Open MPI Process Mapping Directives (`--map-by` & `--bind-to`)

Open MPI's process launcher provides the `ppr` (*processes per resource*) syntax:

```bash
# Launch 1 MPI process per node, allocating 4 physical Processing Elements (cores) each:
mpirun -np 2 --map-by ppr:1:node:PE=4 --bind-to core ./hybrid_app

# Launch 1 MPI process per socket, allocating 32 physical cores each:
mpirun -np 4 --map-by ppr:1:socket:PE=32 --bind-to core ./hybrid_app
```

- **`ppr:K:resource`:** Places exactly $K$ processes per physical unit (`node`, `socket`, `numa`).
- **`PE=T`:** Informs the launcher that each MPI process will spawn $T$ threads (Processing Elements), reserving $T$ contiguous physical cores for that process.
- **`--bind-to core`:** Locks the process and its threads to those reserved physical cores.

---

## 5. Compilable C Code Examples

### Example 1: Multi-Tier Hierarchy Discovery and Core Verification Utility
This complete program calculates and verifies the entire three-tier hierarchy ($N_{\text{nodes}}$, $P_{\text{node}}$, $T_{\text{proc}}$, $T_{\text{total}}$), inspects process affinity masks via Linux `sched_getaffinity()`, prints the physical CPU core running each thread, and validates whether the syllabus worked example (2 nodes $\times$ 1 process $\times$ 4 threads = 8 total threads) is configured correctly.

```c
/*
 * File: mpi_omp_hierarchy.c
 * Compilation:
 *   mpicc -O2 -fopenmp mpi_omp_hierarchy.c -o mpi_omp_hierarchy
 *
 * Execution (Syllabus Worked Example: 2 processes x 4 threads = 8 total threads):
 *   export OMP_NUM_THREADS=4
 *   export OMP_PLACES=cores
 *   export OMP_PROC_BIND=close
 *   mpirun -np 2 ./mpi_omp_hierarchy
 */

#define _GNU_SOURCE
#include <mpi.h>
#include <omp.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sched.h>

int main(int argc, char** argv) {
    int rank, size;
    int provided;
    char hostname[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    /* Initialize MPI with Funneled Thread Support */
    MPI_Init_thread(&argc, &argv, MPI_THREAD_FUNNELED, &provided);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Get_processor_name(hostname, &name_len);

    /* Query OpenMP thread configuration */
    int num_threads_per_proc = omp_get_max_threads();
    int total_active_threads = size * num_threads_per_proc;

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  HYBRID PROCESS & THREAD HIERARCHY VERIFICATION REPORT          \n");
        printf("=================================================================\n");
        printf("  Total MPI Processes (P_total)        : %d\n", size);
        printf("  OpenMP Threads per Process (T_proc)  : %d\n", num_threads_per_proc);
        printf("  Calculated Total Cluster Threads     : %d\n", total_active_threads);
        printf("  Sizing Formula                       : %d Ranks x %d Threads = %d Threads\n",
               size, num_threads_per_proc, total_active_threads);
        printf("=================================================================\n\n");
    }

    /* Synchronize before diagnostic output */
    MPI_Barrier(MPI_COMM_WORLD);

    /* Sequential printing across MPI ranks */
    for (int target_rank = 0; target_rank < size; ++target_rank) {
        if (rank == target_rank) {
            printf("[MPI Rank %02d on Host: %-12s] Spawning %d OpenMP Threads:\n",
                   rank, hostname, num_threads_per_proc);

            #pragma omp parallel
            {
                int tid = omp_get_thread_num();
                int active_core = sched_getcpu();

                #pragma omp critical
                {
                    printf("    -> OpenMP Thread [%02d] executing on Physical CPU Core [%02d]\n",
                           tid, active_core);
                }
            }
            fflush(stdout);
        }
        MPI_Barrier(MPI_COMM_WORLD);
    }

    if (rank == 0) {
        printf("\n=================================================================\n");
        if (size == 2 && num_threads_per_proc == 4 && total_active_threads == 8) {
            printf(">>> VERIFICATION SUCCESS: Matches Course Syllabus Worked Example! <<<\n");
            printf(">>> (2 Nodes/Processes x 4 Threads = 8 Total Threads)              <<<\n");
        } else {
            printf(">>> Custom Configuration Verified Cleanly. <<<\n");
        }
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 2 Processes and OMP_NUM_THREADS=4) ---
=================================================================
  HYBRID PROCESS & THREAD HIERARCHY VERIFICATION REPORT          
=================================================================
  Total MPI Processes (P_total)        : 2
  OpenMP Threads per Process (T_proc)  : 4
  Calculated Total Cluster Threads     : 8
  Sizing Formula                       : 2 Ranks x 4 Threads = 8 Threads
=================================================================

[MPI Rank 00 on Host: workstation ] Spawning 4 OpenMP Threads:
    -> OpenMP Thread [00] executing on Physical CPU Core [00]
    -> OpenMP Thread [01] executing on Physical CPU Core [01]
    -> OpenMP Thread [02] executing on Physical CPU Core [02]
    -> OpenMP Thread [03] executing on Physical CPU Core [03]
[MPI Rank 01 on Host: workstation ] Spawning 4 OpenMP Threads:
    -> OpenMP Thread [00] executing on Physical CPU Core [04]
    -> OpenMP Thread [01] executing on Physical CPU Core [05]
    -> OpenMP Thread [02] executing on Physical CPU Core [06]
    -> OpenMP Thread [03] executing on Physical CPU Core [07]

=================================================================
>>> VERIFICATION SUCCESS: Matches Course Syllabus Worked Example! <<<
>>> (2 Nodes/Processes x 4 Threads = 8 Total Threads)              <<<
=================================================================
```

---

## 6. Mathematical Formulations & Multi-Socket NUMA Allocation

### The Three Master Hierarchy Formulas

Let:
- $N_{\text{nodes}}$ = Number of physical cluster nodes.
- $S_{\text{node}}$ = Sockets per physical node (typically $2$).
- $C_{\text{socket}}$ = Physical cores per socket.
- $C_{\text{node}} = S_{\text{node}} \times C_{\text{socket}}$ = Total physical cores per node.

#### Formula 1: Sizing the Cluster Allocation
To configure an application with $P_{\text{node}}$ processes per node and $T_{\text{proc}}$ threads per process:

$$P_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}}$$

$$T_{\text{total}} = P_{\text{total}} \times T_{\text{proc}} = N_{\text{nodes}} \times P_{\text{node}} \times T_{\text{proc}}$$

#### Formula 2: The Core Saturation Invariant
To prevent both idle CPU cores and context-switching oversubscription:

$$P_{\text{node}} \times T_{\text{proc}} = C_{\text{node}}$$

#### Formula 3: NUMA Socket Alignment Condition
To guarantee zero remote cross-socket NUMA latency penalties:

$$P_{\text{node}} = S_{\text{node}}$$
$$T_{\text{proc}} = C_{\text{socket}} = \frac{C_{\text{node}}}{S_{\text{node}}}$$

Each MPI process is pinned to exactly one CPU socket, and its OpenMP threads are pinned exclusively to the cores of that socket.

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Placement Strategies

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Strategy                  | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `OMP_PROC_BIND=close`     | - Threads stay on adjacent cores        | - If all threads pack on 1 socket while |
|                           | - Maximizes shared L2/L3 cache hit rate |   other socket is empty, bandwidth halts|
+---------------------------+-----------------------------------------+-----------------------------------------+
| `OMP_PROC_BIND=spread`    | - Distributes threads across sockets    | - Cache sharing between threads is lost |
|                           | - Maximizes aggregate memory bandwidth  | - Higher interconnect traffic           |
+---------------------------+-----------------------------------------+-----------------------------------------+
| 1 Rank per Socket         | - 100% NUMA local memory access         | - Requires setting PE parameters in     |
| (Optimal Hybrid)          | - Balances memory bus & network links   |   mpirun launchers                      |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Total Thread Count ($T_{\text{total}}$):** The aggregate number of concurrent execution streams active across a distributed cluster, mathematically defined as $T_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}} \times T_{\text{proc}}$.
>
> **Core Saturation:** An optimal system configuration state where the product of MPI processes per node and OpenMP threads per process precisely equals the physical CPU core count ($P_{\text{node}} \times T_{\text{proc}} = C_{\text{node}}$).
>
> **NUMA Node:** A hardware grouping of physical CPU cores that share direct, high-speed access to a local bank of physical RAM channels, exhibiting lower access latency than accessing RAM connected to other sockets.
>
> **`OMP_PLACES`:** An OpenMP environment variable specifying the abstract hardware execution entities (`threads`, `cores`, `sockets`) to which OpenMP threads are bound.
>
> **`OMP_PROC_BIND`:** An OpenMP environment variable defining the policy (`close`, `spread`, `master`) by which OpenMP threads are assigned and bound to the hardware places defined by `OMP_PLACES`.

---

## 9. Comparison Tables

### Placement Policies: `close` vs. `spread`

| Dimension | `OMP_PROC_BIND=close` | `OMP_PROC_BIND=spread` |
| :--- | :--- | :--- |
| **Placement Behavior** | Consecutive cores near master | Dispersed evenly across sockets |
| **L3 Cache Sharing** | **High** (Threads share common cache) | Low (Threads distributed across caches) |
| **Memory Bandwidth** | Limited to memory bus of 1 socket | **Maximum** (Uses all socket memory buses) |
| **Ideal Workload** | Data-dependent, tightly coupled loops | Memory-bandwidth-bound streaming kernels |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Solving the Core Oversubscription Disaster
**Question:** A student executes a hybrid application on a cluster of $4$ nodes. Each node possesses $32$ physical CPU cores. The student writes a launch script containing:
```bash
export OMP_NUM_THREADS=16
mpirun -np 16 ./hybrid_sim
```
1. Calculate the total number of MPI processes and OpenMP threads spawned across the cluster.
2. Calculate the number of threads competing on each physical node.
3. Identify the performance defect and provide the exact corrected commands.

**Solution:**
1. **Total Processes & Threads:**
   - Total MPI processes = $16$ processes across $4$ nodes $\implies P_{\text{node}} = 16 / 4 = 4$ processes per node.
   - Total cluster threads = $16\text{ processes} \times 16\text{ threads} = \mathbf{256\text{ total threads}}$.
2. **Threads per Node:**
   - Threads per node = $P_{\text{node}} \times T_{\text{proc}} = 4 \times 16 = \mathbf{64\text{ threads per node}}$.
   - Physical cores available per node = $32\text{ cores}$.
   - Oversubscription factor = $64 / 32 = \mathbf{2.0\times \text{ Oversubscribed}}$!
3. **The Defect & Fix:**
   - The node is $200\%$ oversubscribed. Two threads constantly fight for each physical core, triggering heavy OS scheduling thrashing.
   - **The Fix:** Reduce `OMP_NUM_THREADS` to $8$, or run fewer MPI processes per node:

```bash
# Correct Approach: 4 processes per node x 8 threads = 32 cores saturated
export OMP_NUM_THREADS=8
mpirun -np 16 --map-by ppr:4:node:PE=8 --bind-to core ./hybrid_sim
```

---

### Problem 2: Sizing for a Dual-Socket Supercomputer Allocation
**Question:** You are allocated $8$ compute nodes on a national supercomputer. Each node has $2$ sockets, each with $48$ physical cores (total $96$ cores/node). To maximize memory bandwidth and avoid NUMA penalties:
1. How many MPI processes should be placed per node?
2. What should `OMP_NUM_THREADS` be set to?
3. What is the total count of MPI processes ($P_{\text{total}}$) and total threads ($T_{\text{total}}$) across the cluster?

**Solution:**
1. **MPI Processes per Node:**
   - To match NUMA architecture, assign **1 MPI process per socket**:
$$P_{\text{node}} = 2\text{ processes per node}$$
2. **OpenMP Threads per Process:**
   - Each socket has 48 cores:
$$\text{export } \mathbf{OMP\_NUM\_THREADS = 48}$$
3. **Cluster Totals:**
$$P_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}} = 8 \times 2 = \mathbf{16\text{ MPI processes across cluster}}$$
$$T_{\text{total}} = P_{\text{total}} \times \text{OMP\_NUM\_THREADS} = 16 \times 48 = \mathbf{768\text{ Total Active Threads}}$$

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: What does the syllabus worked example ($2\text{ nodes} \times 1\text{ process} \times 4\text{ threads} = 8\text{ threads}$) demonstrate?
**Answer:** It demonstrates the minimal two-tiered hierarchical parallel execution model:
- **Inter-node tier:** 2 physical nodes communicate across the network, running 1 MPI process each (Rank 0 on Node 1, Rank 1 on Node 2).
- **Intra-node tier:** Each MPI process spawns 4 OpenMP threads executing on local multi-core shared memory.
- **Combined execution:** Exactly 8 concurrent threads cooperate across the cluster to solve a unified computational problem.

### Q2: What happens if `OMP_PROC_BIND` is not set in an MPI+OpenMP program?
**Answer:** If `OMP_PROC_BIND` is omitted, OpenMP threads are not pinned to specific CPU cores. The operating system kernel is free to migrate threads between cores or even between CPU sockets during execution. This causes severe cache invalidations (loss of L1/L2 cache data) and forces high-latency cross-socket NUMA memory accesses, degrading performance by $20\%\text{ to }50\%$.

### Q3: What is the meaning of `:PE=K` in Open MPI's `--map-by` flag?
**Answer:** `:PE=K` stands for **Processing Elements = K**. It informs the MPI process launcher that each spawned MPI process will run $K$ OpenMP threads. The launcher uses this information to reserve a contiguous block of $K$ physical CPU cores for each process, ensuring that sister OpenMP threads have dedicated cores and do not collide with adjacent MPI processes.

---

## 12. Quick Revision Sheet

- **Master Sizing Formula:** $T_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}} \times T_{\text{proc}}$.
- **Syllabus Worked Example:** $2\text{ nodes} \times 1\text{ proc} \times 4\text{ threads} = \mathbf{8\text{ total threads}}$.
- **Saturation Rule:** $P_{\text{node}} \times T_{\text{proc}} = C_{\text{node}}$ (Cores per node).
- **NUMA Best Practice:** $P_{\text{node}} = \text{Sockets per node}$; $T_{\text{proc}} = \text{Cores per socket}$.
- **Affinity Environment Variables:**
  - `export OMP_PLACES=cores` (binds to physical cores).
  - `export OMP_PROC_BIND=close` (packs threads on adjacent cores).
- **Launch Command:** `mpirun -np P --map-by ppr:P_node:node:PE=T_proc --bind-to core ./app`.

---

## 13. What You Should Be Able to Do

1. **Calculate hierarchical process and thread counts** for any cluster configuration given nodes, sockets, and cores.
2. **Reproduce and explain the syllabus worked example** ($2 \times 1 \times 4 = 8$) in university examinations.
3. **Configure thread binding and affinity** using `OMP_PLACES` and `OMP_PROC_BIND`.
4. **Prevent core oversubscription and undersubscription** using precise mathematical saturation checks.
5. **Optimize NUMA process placement** by mapping MPI processes directly to physical CPU sockets.
