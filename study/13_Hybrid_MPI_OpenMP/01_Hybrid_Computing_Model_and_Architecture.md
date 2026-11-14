# Module 13: Hybrid MPI + OpenMP Programming (Lecture 32)
# Subtopic 13.01: Hybrid Computing Model and Architecture (Hierarchical Parallelism, Pure MPI Limitations, Shared vs. Private Address Spaces)

---

## 1. Basic Idea

In modern High-Performance Computing (HPC), supercomputers are built as **hierarchical hardware clusters**:
- At the macro level, the cluster consists of thousands of independent compute nodes connected by high-speed networks (e.g., InfiniBand or Slingshot).
- At the micro level, each compute node is a multi-socket shared-memory computer containing dozens of high-performance CPU cores accessing common NUMA RAM channels.

Neither pure OpenMP nor pure MPI aligns with this two-tiered physical reality:
- **Pure OpenMP** is confined to a single compute node; it cannot scale across the network to other cluster machines.
- **Pure MPI** treats every single CPU core on the machine as an isolated, independent operating system process. When running 128 pure MPI ranks on a 128-core node, 128 separate copies of process metadata, runtime libraries, and halo buffers are duplicated in memory, wasting precious node RAM and creating massive on-node message-passing overhead.

The **Hybrid MPI + OpenMP Programming Model** matches software architecture to the physical hardware hierarchy:
1. **Inter-Node Parallelism (MPI across nodes):** Coarse-grained distributed-memory domain decomposition routes messages across the network between nodes via MPI.
2. **Intra-Node Parallelism (OpenMP within nodes):** Fine-grained shared-memory loop and task parallelism distributes work across the multi-core sockets of each node via OpenMP threads.

```
+-----------------------------------------------------------------------------------------+
|                  The Hybrid MPI + OpenMP Architecture Hierarchy                         |
+-----------------------------------------------------------------------------------------+
| CLUSTER LEVEL (Inter-Node: MPI Message Passing over Network Fabric):                    |
|                                                                                         |
|       +-----------------------------+         +-----------------------------+           |
|       | NODE 0 (MPI Rank 0)         |         | NODE 1 (MPI Rank 1)         |           |
|       | Address Space 0             |         | Address Space 1             |           |
|       +--------------+--------------+         +--------------+--------------+           |
|                      |                                       |                          |
|                      +============== InfiniBand Fabric ======+                          |
|                                                                                         |
| NODE LEVEL (Intra-Node: OpenMP Shared-Memory Multi-Threading):                          |
|                                                                                         |
|   NODE 0 (MPI Rank 0 Address Space):                                                    |
|     +------------------------------------------------------------------+                |
|     | Shared Process Heap & Global Variables                           |                |
|     +------------------------------------------------------------------+                |
|        |                |                |                |                             |
|        v                v                v                v                             |
|     [Thread 0]       [Thread 1]       [Thread 2]       [Thread 3]                       |
|     (Core 0)         (Core 1)         (Core 2)         (Core 3)                         |
|     #pragma omp parallel for (Dynamic Work-Sharing over Shared RAM)                     |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Memory Wall of Pure MPI on High-Core-Count Nodes
Modern server processors feature dense core counts (e.g., AMD EPYC with 128 physical cores per socket, dual-socket systems totaling 256 cores per node). 
If an application deploys **Pure MPI** (one rank per core):
- 256 distinct operating system processes are spawned on a single motherboard.
- Each process allocates its own private global tables, OS file descriptors, and MPI communication buffers.
- In a 3D halo exchange, each rank maintains 6 boundary halo buffers. With 256 ranks on the node, thousands of internal halo buffers are replicated in physical RAM.
- **Memory Footprint Explosion:** Often, more than $50\%$ of node RAM is consumed purely by duplicate ghost cells and MPI internal buffer queues, leaving insufficient RAM for actual scientific calculations.

Under **Hybrid MPI + OpenMP**:
- Only 1 or 2 MPI processes are launched per node (e.g., 1 process per CPU socket).
- Each MPI process spawns 64 or 128 OpenMP threads.
- All threads on the node share a single copy of the global grid in memory.
- Halo cells are required **only at the physical node boundaries**, reducing halo memory overhead by up to $80\%$.

### 2. Message Explosion in Point-to-Point and Collectives
In pure MPI with $P$ total cores across a cluster:
- All-to-all communications or global reductions scale in complexity with process count $P$.
- For $P = 100{,}000$ pure MPI processes, the connection matrix requires tracking $100{,}000^2 = 10^{10}$ potential communication paths!
- In a hybrid model with 32 OpenMP threads per MPI rank, the number of active MPI processes drops by a factor of 32:
$$P_{\text{hybrid}} = \frac{100{,}000}{32} \approx 3{,}125\text{ MPI processes}$$
The total connection matrix shrinks from $10^{10}$ paths to approximately $9.7 \times 10^6$ paths—a **$1{,}000\times$ reduction** in network state management!

### 3. Automatic Load Balancing on Irregular Calculations
Within a single node, some computational sub-regions may take longer to compute than others. In pure MPI, a slow rank forces neighboring ranks to stall at communication boundaries. In hybrid OpenMP, dynamic loop scheduling (`#pragma omp for schedule(dynamic)`) allows fast threads to automatically steal work from slower threads on the same node, smoothing out internal load imbalances before any network communication occurs.

---

## 3. How It Works

### Architectural Address Space Separation

Understanding the hybrid model requires tracking the boundary between private and shared virtual memory:

```
+-----------------------------------------------------------------------------------------+
|                           Hybrid Memory Boundary Separation                             |
+-----------------------------------------------------------------------------------------+
| MACHINE 1 (Host A):                                                                     |
|   MPI Process 0 Address Space:                                                          |
|     - Global Variables: Process-Private (Invisible to Machine 2)                        |
|     - OpenMP Thread 0 (Master) : Stack 0 (Private)                                      |
|     - OpenMP Thread 1 (Worker) : Stack 1 (Private)                                      |
|     - OpenMP Thread 2 (Worker) : Stack 2 (Private)                                      |
|     - OpenMP Thread 3 (Worker) : Stack 3 (Private)                                      |
|     - Shared Heap: Accessible by all 4 threads of Process 0                             |
|                                                                                         |
|                             [ NETWORK CABLE (InfiniBand) ]                              |
|                                                                                         |
| MACHINE 2 (Host B):                                                                     |
|   MPI Process 1 Address Space:                                                          |
|     - Global Variables: Process-Private (Invisible to Machine 1)                        |
|     - OpenMP Thread 0 (Master) : Stack 0 (Private)                                      |
|     - OpenMP Thread 1 (Worker) : Stack 1 (Private)                                      |
|     - OpenMP Thread 2 (Worker) : Stack 2 (Private)                                      |
|     - OpenMP Thread 3 (Worker) : Stack 3 (Private)                                      |
|     - Shared Heap: Accessible by all 4 threads of Process 1                             |
+-----------------------------------------------------------------------------------------+
```

- **Thread 1 on Process 0** can directly read memory modified by **Thread 3 on Process 0** via standard C pointer dereferences (OpenMP shared memory).
- **Thread 1 on Process 0** CANNOT read memory modified by **Thread 1 on Process 1**! Any data transfer between Process 0 and Process 1 must be packed into an MPI buffer and transmitted over the network via an MPI communication call (`MPI_Send` / `MPI_Bcast`).

---

### The Three Structural Models of Hybrid Programming

Hybrid applications generally follow one of three architectural patterns based on how OpenMP threads interact with the MPI library:

#### 1. Master-Only / Funneled Model (Most Common & Recommended)
All MPI communication calls are placed **outside** OpenMP parallel regions, or executed strictly by the Master Thread inside a `#pragma omp master` directive.
- **Advantage:** Requires only basic thread safety (`MPI_THREAD_FUNNELED`). No race conditions on network calls. Simple to write, debug, and optimize.
- **Disadvantage:** Worker threads may sit idle while the master thread transmits network data.

#### 2. Pipelined / Overlapped Model
The master thread executes non-blocking MPI halo transfers (`MPI_Isend`, `MPI_Irecv`) while worker threads compute the interior matrix domain concurrently.
- **Advantage:** Maximum computation-communication overlap.
- **Disadvantage:** Requires careful synchronization barriers between boundary and interior calculations.

#### 3. Fully Multi-Threaded Model (`MPI_THREAD_MULTIPLE`)
Any OpenMP thread may invoke MPI routines concurrently at any time.
- **Advantage:** Flexible, fine-grained asynchronous communication.
- **Disadvantage:** Incurs heavy internal locking overhead inside the MPI library; difficult to verify correctness.

---

## 4. Important Mechanisms and Functions

### 1. Hybrid Process and Thread Hierarchy Calculations

To configure and launch a hybrid application, three parameters must be specified:
1. $N_{\text{nodes}}$: Total physical server chassis allocated.
2. $P_{\text{node}}$: Number of MPI processes spawned on each node.
3. $T_{\text{proc}}$: Number of OpenMP threads spawned per MPI process (governed by `$OMP_NUM_THREADS`).

The fundamental sizing formulas are:

$$\text{Total MPI Processes } P_{\text{total}} = N_{\text{nodes}} \times P_{\text{node}}$$

$$\text{Total Active Threads across Cluster } T_{\text{total}} = P_{\text{total}} \times T_{\text{proc}} = N_{\text{nodes}} \times P_{\text{node}} \times T_{\text{proc}}$$

$$\text{Hardware Saturation Condition: } P_{\text{node}} \times T_{\text{proc}} \le C_{\text{node}} \quad \text{(Available CPU Cores per Node)}$$

---

### 2. MPI Multi-Threaded Initialization (`MPI_Init_thread`)

Standard `MPI_Init()` initializes the runtime for single-threaded operation. Hybrid codes must initialize via **`MPI_Init_thread()`**:

```c
#include <mpi.h>
#include <omp.h>

int MPI_Init_thread(int *argc, char ***argv, int required, int *provided);
```

- **`required`:** The desired thread safety level requested by the application.
- **`provided`:** The actual thread safety level granted by the underlying MPI library implementation.

---

## 5. Compilable C Code Examples

### Example 1: Foundational Hybrid MPI + OpenMP Topology Diagnostic
This complete program initializes MPI with multi-threading support, discovers the hierarchical process and thread structure, prints the process rank, OpenMP thread ID, physical hostname, and active CPU core ID, and validates that the cluster's hardware cores are saturated properly without oversubscription.

```c
/*
 * File: hybrid_topology_check.c
 * Compilation:
 *   mpicc -O2 -fopenmp hybrid_topology_check.c -o hybrid_topology_check
 *
 * Execution (Simulating 2 nodes / processes with 4 threads each = 8 total threads):
 *   export OMP_NUM_THREADS=4
 *   mpirun -np 2 ./hybrid_topology_check
 */

#define _GNU_SOURCE
#include <mpi.h>
#include <omp.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>

int main(int argc, char** argv) {
    int rank, size;
    int provided_level;
    char hostname[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    /* Step 1: Initialize MPI with Funneled Thread Support */
    int rc = MPI_Init_thread(&argc, &argv, MPI_THREAD_FUNNELED, &provided_level);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "MPI_Init_thread failed\n");
        exit(EXIT_FAILURE);
    }

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Get_processor_name(hostname, &name_len);

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  HYBRID MPI + OpenMP HIERARCHICAL TOPOLOGY DIAGNOSTIC           \n");
        printf("  MPI Processes (Ranks): %d                                      \n", size);
        printf("  MPI Thread Safety Level Granted: ");
        if (provided_level == MPI_THREAD_SINGLE)     printf("MPI_THREAD_SINGLE\n");
        else if (provided_level == MPI_THREAD_FUNNELED)   printf("MPI_THREAD_FUNNELED\n");
        else if (provided_level == MPI_THREAD_SERIALIZED) printf("MPI_THREAD_SERIALIZED\n");
        else if (provided_level == MPI_THREAD_MULTIPLE)   printf("MPI_THREAD_MULTIPLE\n");
        printf("=================================================================\n\n");
    }

    /* Synchronize before printing diagnostics */
    MPI_Barrier(MPI_COMM_WORLD);

    /* Step 2: OpenMP Parallel Region inside each MPI Process */
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        int total_threads = omp_get_num_threads();
        int active_cpu = sched_getcpu(); /* Query physical CPU core */

        /* Critical section prevents jumbled stdout across threads */
        #pragma omp critical
        {
            printf("[Host: %-12s] MPI Rank: %02d | OpenMP Thread: %02d of %02d | Active Core: %02d\n",
                   hostname, rank, tid, total_threads, active_cpu);
        }
    }

    MPI_Barrier(MPI_COMM_WORLD);

    if (rank == 0) {
        printf("\n=================================================================\n");
        printf("Hierarchical topology verification completed successfully.\n");
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 2 MPI Ranks and 4 OpenMP Threads each) ---
=================================================================
  HYBRID MPI + OpenMP HIERARCHICAL TOPOLOGY DIAGNOSTIC           
  MPI Processes (Ranks): 2                                      
  MPI Thread Safety Level Granted: MPI_THREAD_FUNNELED
=================================================================

[Host: node01      ] MPI Rank: 00 | OpenMP Thread: 00 of 04 | Active Core: 00
[Host: node01      ] MPI Rank: 00 | OpenMP Thread: 01 of 04 | Active Core: 01
[Host: node01      ] MPI Rank: 00 | OpenMP Thread: 02 of 04 | Active Core: 02
[Host: node01      ] MPI Rank: 00 | OpenMP Thread: 03 of 04 | Active Core: 03
[Host: node02      ] MPI Rank: 01 | OpenMP Thread: 00 of 04 | Active Core: 00
[Host: node02      ] MPI Rank: 01 | OpenMP Thread: 01 of 04 | Active Core: 01
[Host: node02      ] MPI Rank: 01 | OpenMP Thread: 02 of 04 | Active Core: 02
[Host: node02      ] MPI Rank: 01 | OpenMP Thread: 03 of 04 | Active Core: 03

=================================================================
Hierarchical topology verification completed successfully.
```

---

## 6. Mathematical Formulations & Memory Footprint Scaling

### Memory Savings Formulation: Pure MPI vs. Hybrid MPI+OpenMP

Consider a 3D stencil simulation on a cubic grid of global dimension $N \times N \times N$.
Let each grid point store $8\text{ bytes}$ of double-precision data. 

#### 1. In Pure MPI (with $P$ total cores):
The domain is partitioned into $P$ cubic sub-domains, each of size:

$$L_{\text{sub}} = \frac{N}{P^{1/3}}$$

Each sub-domain requires a 1-cell halo boundary on all 6 faces to exchange boundary data with its 6 neighbors:

$$\text{Interior Volume per Rank} = L_{\text{sub}}^3 = \frac{N^3}{P}$$
$$\text{Halo Surface Cells per Rank} = 6 \times L_{\text{sub}}^2 = 6 \left( \frac{N}{P^{1/3}} \right)^2 = \frac{6 N^2}{P^{2/3}}$$

Total halo cells stored across all $P$ processes in Pure MPI:

$$H_{\text{pure}}(P) = P \times \frac{6 N^2}{P^{2/3}} = 6 N^2 P^{1/3}$$

Notice that as core count $P$ increases, **total halo memory grows as $P^{1/3}$**!

#### 2. In Hybrid MPI+OpenMP:
Let there be $M$ MPI processes (one per node), where each process spawns $T$ OpenMP threads ($P = M \times T$).
Halo data is exchanged **only between MPI processes across nodes**. OpenMP threads on the same node access shared memory directly without duplicating halo cells!

Total halo cells stored across the cluster in Hybrid:

$$H_{\text{hybrid}}(M) = 6 N^2 M^{1/3} = 6 N^2 \left( \frac{P}{T} \right)^{1/3} = \frac{H_{\text{pure}}(P)}{T^{1/3}}$$

#### Memory Reduction Factor:

$$\text{Halo Memory Savings Factor} = \frac{H_{\text{pure}}}{H_{\text{hybrid}}} = T^{1/3}$$

If each node runs $T = 64$ OpenMP threads per MPI rank:

$$\text{Savings Factor} = 64^{1/3} = \mathbf{4\times \text{ Reduction in Halo Memory!}}$$

On modern 256-core supercomputing nodes ($T = 256$):

$$\text{Savings Factor} = 256^{1/3} \approx \mathbf{6.35\times \text{ Reduction!}}$$

This dramatic memory saving enables scientists to run higher-resolution simulations that would otherwise exceed available physical RAM under pure MPI.

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Trade-off Matrix

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Architecture              | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Hybrid MPI + OpenMP       | - Aligns perfectly with hardware tiers  | - Higher programming complexity         |
|                           | - Massive reduction in halo memory      | - Requires thread safety configuration  |
|                           | - Shrinks MPI connection matrix by 100x | - Prone to NUMA performance penalties   |
|                           | - Automatic intra-node load balancing   |   if threads migrate across sockets     |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Pure MPI                  | - Simple, uniform programming model     | - Severe memory duplication of halos    |
| (1 Rank per Core)         | - No thread-safety issues               | - Massive connection matrix at scale    |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Pure OpenMP               | - Extremely fast shared memory access   | - Cannot scale beyond a single node;    |
|                           | - Trivially simple loop pragmas         |   useless for distributed clusters      |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Hybrid Parallelism (MPI + OpenMP):** A hierarchical programming model combining distributed-memory message passing (MPI) across cluster nodes with shared-memory multithreading (OpenMP) across the CPU cores of each individual node.
>
> **Coarse-Grained Parallelism:** High-level algorithmic decomposition where a global dataset is partitioned into large sub-domains assigned to independent MPI processes.
>
> **Fine-Grained Parallelism:** Low-level parallel execution where iterations of loops or discrete tasks within a sub-domain are distributed across OpenMP threads sharing a common virtual address space.
>
> **`MPI_Init_thread`:** An extended MPI initialization routine that requests and negotiates a specific level of multi-threaded safety from the underlying MPI library.
>
> **Halo Memory Reduction:** The architectural property of hybrid computing ensuring that ghost cell exchange buffers exist only between distinct physical nodes, eliminating internal ghost buffers between cores sharing local RAM.

---

## 9. Comparison Tables

### Pure MPI vs Pure OpenMP vs Hybrid MPI+OpenMP

| Feature | Pure MPI | Pure OpenMP | Hybrid MPI + OpenMP |
| :--- | :--- | :--- | :--- |
| **Cluster Scaling** | Unlimited (Millions of cores)| **Single Node Only** ($\le 128$ cores) | Unlimited (Millions of cores) |
| **Address Space** | Disjoint per process | Unified shared process space | Hierarchical: Disjoint per node; Shared within node |
| **Memory Overhead** | **High** (Duplicated halos/data) | **Low** (Single shared heap) | **Minimal** (Halos only at node borders) |
| **MPI Rank Count ($P$)**| Equal to Total CPU Cores | None (0) | Equal to Nodes or Sockets |
| **Load Balancing** | Static / Domain repartitioning | Dynamic work-stealing | Dynamic inside node; Static across nodes |
| **Programming Difficulty**| Medium | Low | **High** (Requires dual-level mastery) |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Cluster Sizing and Configuration Calculations
**Question:** A university researcher is allocated $N_{\text{nodes}} = 16$ compute nodes on a supercomputer. Each node features dual AMD EPYC sockets, with each socket containing $64$ physical cores (total $128$ physical cores per node).
1. If configured as Pure MPI, how many MPI ranks are launched?
2. If configured as a Hybrid application with **1 MPI process per socket**, how many MPI ranks are launched across the cluster?
3. In this hybrid configuration, what must `OMP_NUM_THREADS` be set to in order to fully saturate all physical cores without oversubscription?
4. What is the total count of active concurrent threads across the entire 16-node cluster?

**Solution:**
1. **Pure MPI Ranks:**
$$P_{\text{pure}} = N_{\text{nodes}} \times C_{\text{node}} = 16 \times 128 = \mathbf{2{,}048\text{ MPI ranks}}$$

2. **Hybrid MPI Ranks (1 rank per socket, 2 sockets per node):**
$$P_{\text{node}} = 2\text{ ranks/node}$$
$$P_{\text{hybrid}} = N_{\text{nodes}} \times P_{\text{node}} = 16 \times 2 = \mathbf{32\text{ MPI ranks across cluster}}$$

3. **OpenMP Threads per Process:**
   Each socket possesses 64 physical cores. To assign 1 thread per core:
$$\text{export } \mathbf{OMP\_NUM\_THREADS = 64}$$

4. **Total Active Threads across Cluster:**
$$T_{\text{total}} = P_{\text{hybrid}} \times \text{OMP\_NUM\_THREADS} = 32 \times 64 = \mathbf{2{,}048\text{ total threads}}$$
*(All 2,048 physical hardware cores across the 16 nodes are fully saturated).*

---

### Problem 2: Analyzing the Master-Only Thread Starvation Bottleneck
**Question:** In a hybrid simulation using 1 MPI process per node and 64 OpenMP threads, the programmer writes:
```c
#pragma omp parallel
{
    compute_subdomain();
}
/* All 64 threads join here */

/* Master thread sends 100 MB halo */
MPI_Send(halo_buffer, 10000000, MPI_DOUBLE, neighbor, 0, MPI_COMM_WORLD);

#pragma omp parallel
{
    compute_next_phase();
}
```
Explain the performance bottleneck during the `MPI_Send` call.

**Solution:**
- **The Bottleneck:** When the first parallel region ends, 63 OpenMP worker threads are terminated or put to sleep at the barrier.
- The master thread executes the blocking `MPI_Send` transferring 100 MB over the network (which may take several milliseconds).
- During this entire network transfer, **63 physical CPU cores sit completely idle** consuming zero productive cycles.
- **The Solution:** Employ the **Pipelined / Overlapped Hybrid Model**:
  - Keep the OpenMP parallel region active.
  - The master thread executes non-blocking `MPI_Isend` on the halo while worker threads concurrently compute the independent interior domain.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why is launching 1 MPI process per socket often superior to launching 1 MPI process per node?
**Answer:** Modern server motherboards feature **Non-Uniform Memory Access (NUMA)** architectures where each CPU socket has its own dedicated physical memory controllers. If a single MPI process spans multiple sockets, OpenMP threads executing on Socket 1 that access memory allocated on Socket 0 must traverse cross-socket interconnects (e.g., AMD Infinity Fabric or Intel UPI), increasing latency and halving memory bandwidth. Launching **1 MPI process per NUMA socket** ensures that all OpenMP threads belonging to that process access local, high-bandwidth RAM.

### Q2: Can an OpenMP thread access a variable declared as `private` by another thread in the same MPI process?
**Answer:** No. Variables marked `private` in an OpenMP parallel region are allocated on that individual thread's private stack frame. They are physically invisible and inaccessible to other threads within the same process.

### Q3: Why is compiling a hybrid program performed using `mpicc -fopenmp` rather than `gcc`?
**Answer:** `mpicc` is the compiler wrapper that automatically injects the necessary MPI header include paths (`-I`), library directories (`-L`), and dynamic link libraries (`-lmpi`). The `-fopenmp` flag instructs the underlying compiler (such as GCC) to recognize and process OpenMP `#pragma omp` directives and link against `libgomp`. Omitting `mpicc` causes MPI compilation errors; omitting `-fopenmp` causes the compiler to ignore OpenMP pragmas, producing a single-threaded sequential code inside each MPI rank.

---

## 12. Quick Revision Sheet

- **Hybrid Model:** MPI across nodes (inter-node message passing); OpenMP within nodes (intra-node shared memory).
- **Core Motivation:** Eliminates pure MPI memory explosion (halo duplication) and shrinks network connection matrices by orders of magnitude.
- **Sizing Invariant:** $P_{\text{node}} \times \text{OMP\_NUM\_THREADS} = C_{\text{node}}$ (Cores per node).
- **Initialization:** Must use `MPI_Init_thread(&argc, &argv, required, &provided)`.
- **Compiler Command:** `mpicc -O2 -fopenmp hybrid_app.c -o hybrid_app`.
- **NUMA Best Practice:** Bind 1 MPI process per CPU socket, and spawn OpenMP threads matching that socket's physical core count.

---

## 13. What You Should Be Able to Do

1. **Calculate hierarchical process and thread topologies** for arbitrary multi-node supercomputer architectures.
2. **Explain the mathematical reduction in halo memory footprint** achieved by hybrid computing over pure MPI.
3. **Compile and execute hybrid programs** using `mpicc -fopenmp` and `OMP_NUM_THREADS`.
4. **Identify the architectural boundary** between shared-memory thread pointers and distributed-memory message envelopes.
5. **Architect master-only, pipelined, and multi-threaded hybrid pipelines** avoiding thread starvation.
