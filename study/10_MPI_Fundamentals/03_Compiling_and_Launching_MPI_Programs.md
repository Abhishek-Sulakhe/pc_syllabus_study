# Module 10: MPI Distributed-Memory Foundations (Lectures 24–25)
# Subtopic 10.03: Compiling and Launching MPI Programs (`mpicc`, `mpirun`, `mpiexec`, Hostfiles, Process Binding, SLURM)

---

## 1. Basic Idea

Compiling and executing an MPI program involves fundamentally different toolchains and runtime mechanisms than standard single-node C programs:

1. **Compilation via Compiler Wrappers (`mpicc`, `mpicxx`):**
   MPI implementations do not replace your system's C compiler (such as GCC, Clang, or Intel OneAPI). Instead, they provide **compiler wrapper scripts** (e.g., `mpicc`) that transparently invoke the underlying native compiler while automatically injecting the correct header search paths (`-I`), library paths (`-L`), and dynamic link flags (`-lmpi`).

2. **Execution via Process Launchers (`mpirun`, `mpiexec`, `srun`):**
   Because an MPI program consists of multiple autonomous processes running across different network nodes, typing `./my_mpi_prog` in the terminal will only run a single isolated process on the local core. To launch a coordinated parallel team, you must use a **process launcher** (such as `mpirun` or `mpiexec`). The launcher:
   - Connects to remote compute nodes via SSH or cluster daemons.
   - Spawns $P$ process instances.
   - Sets up the communication sockets and environment variables needed to build `MPI_COMM_WORLD`.
   - Binds processes to designated CPU cores.

```
+-----------------------------------------------------------------------------------------+
|                              The MPI Build and Launch Flow                              |
+-----------------------------------------------------------------------------------------+
| 1. COMPILATION:                                                                         |
|    mpicc -O2 simulation.c -o simulation                                                |
|      |                                                                                  |
|      v  (Wraps native gcc: adds -I/opt/openmpi/include -L/opt/openmpi/lib -lmpi)        |
|    Binary: 'simulation'                                                                 |
|                                                                                         |
| 2. LAUNCH:                                                                              |
|    mpirun -np 4 --hostfile my_cluster.txt ./simulation                                  |
|      |                                                                                  |
|      +---> SSH to node1: Spawns Process (Rank 0, Core 0)                                |
|      +---> SSH to node1: Spawns Process (Rank 1, Core 1)                                |
|      +---> SSH to node2: Spawns Process (Rank 2, Core 0)                                |
|      +---> SSH to node2: Spawns Process (Rank 3, Core 1)                                |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Eliminating Header and Library Path Complexity
Manually linking MPI libraries requires cumbersome compiler commands:
```bash
# Complex, error-prone manual command:
gcc -O2 -I/usr/lib/x86_64-linux-gnu/openmpi/include \
    -L/usr/lib/x86_64-linux-gnu/openmpi/lib \
    -Wl,-rpath,/usr/lib/x86_64-linux-gnu/openmpi/lib \
    simulation.c -lmpi -o simulation
```
The wrapper `mpicc` packages all implementation-specific compiler definitions, architecture flags, and linker settings into a single clean command:
```bash
mpicc -O2 simulation.c -o simulation
```

### 2. Hostfiles and Cluster Distribution
When executing across a distributed supercomputer cluster, the process launcher must know **where** to place processes. A **Hostfile** specifies the network hostnames of the compute nodes and the number of available CPU cores (slots) on each.

### 3. CPU Core Binding and NUMA Affinity
In multi-socket servers, if the operating system dynamically migrates an MPI process from Core 0 on Socket 0 to Core 32 on Socket 1, the process's memory accesses become cross-socket remote NUMA accesses, collapsing memory bandwidth. Process launchers provide precise **Process Mapping and Core Binding** directives (`--bind-to core`, `--map-by socket`), locking each MPI process to a dedicated physical CPU core.

### 4. Integration with Supercomputer Schedulers (SLURM)
In production supercomputing centers, compute nodes are shared among thousands of users. Users do not run `mpirun` directly; instead, workload managers like **SLURM** allocate physical nodes, set environment variables (`$SLURM_NODELIST`, `$SLURM_NTASKS`), and launch MPI processes via `srun`.

---

## 3. How It Works

### Inside the Compiler Wrapper (`mpicc -show`)

To inspect what `mpicc` is actually executing under the hood, pass the `-show` (or `-showme` on Open MPI) flag:

```bash
$ mpicc -show
gcc -I/usr/include/x86_64-linux-gnu/mpich -L/usr/lib/x86_64-linux-gnu -lmpich
```
The output reveals that `mpicc` is simply a shell wrapper script or thin executable that prepends and appends necessary include paths and shared libraries to the system's base C compiler.

---

### The Process Launch Sequence (`mpirun` / `mpiexec`)

When you run `mpirun -np 4 --hostfile hosts.txt ./app`:

```
Local Terminal (Head Node)
  |
  v
mpirun / mpiexec Daemon
  |
  +---> Reads 'hosts.txt':
  |       nodeA slots=2
  |       nodeB slots=2
  |
  +---> Launches SSH / Daemon connection to nodeA:
  |       - Spawns PID 1024 (Rank 0) -> Binds to CPU Core 0
  |       - Spawns PID 1025 (Rank 1) -> Binds to CPU Core 1
  |
  +---> Launches SSH / Daemon connection to nodeB:
  |       - Spawns PID 8410 (Rank 2) -> Binds to CPU Core 0
  |       - Spawns PID 8411 (Rank 3) -> Binds to CPU Core 1
  |
  +---> Coordinates Socket Exchange:
          All 4 processes exchange IP/port endpoints and initialize MPI_COMM_WORLD.
```

---

### Hostfile Syntax and Slots

A hostfile is a plain-text configuration file listing cluster nodes:

```text
# Format: <hostname_or_ip> slots=<number_of_cores> [max_slots=<limit>]
node01 slots=4
node02 slots=4
node03 slots=8
192.168.1.50 slots=2
```

- **`slots`:** Indicates how many CPU cores or processes this node can comfortably execute without oversubscription.
- If you request `-np 8` with the hostfile above:
  - Ranks 0, 1, 2, 3 run on `node01`.
  - Ranks 4, 5, 6, 7 run on `node02`.

---

## 4. Important Mechanisms and CLI Flags

### 1. Essential `mpirun` / `mpiexec` Flags

| Flag | Purpose | Example |
| :--- | :--- | :--- |
| **`-n <P>` / `-np <P>`** | Specifies total number of MPI processes to spawn | `mpirun -np 16 ./app` |
| **`--hostfile <file>` / `-f`** | Specifies file listing target node hostnames and core slots | `mpirun -np 32 --hostfile cluster.hosts ./app` |
| **`-H <hosts>` / `--host`** | Comma-separated list of hostnames directly on CLI | `mpirun -H node1,node2,node3 -np 6 ./app` |
| **`--oversubscribe`** | Permits launching more processes than physical CPU cores | `mpirun -np 128 --oversubscribe ./test` |
| **`--display-map`** | Displays process-to-node and process-to-core mapping in terminal | `mpirun -np 4 --display-map ./app` |
| **`--display-allocation`** | Prints discovered cluster nodes and slots | `mpirun -np 4 --display-allocation ./app` |

---

### 2. Process Mapping and Core Binding Policies

In Open MPI, fine-grained process placement is governed by two orthogonal options:

#### A. Mapping (`--map-by <unit>`)
Dictates how processes are distributed across physical hardware hierarchy:
- `--map-by core` (Default): Round-robin assignment by individual core.
- `--map-by socket`: Alternates assignment across CPU sockets to balance memory bandwidth.
- `--map-by node`: Round-robin distribution across separate physical nodes (Rank 0 on Node 1, Rank 1 on Node 2, Rank 2 on Node 1...).

#### B. Binding (`--bind-to <unit>`)
Locks each process to specific hardware resources so the OS scheduler cannot migrate it:
- `--bind-to core`: Binds process strictly to one physical core.
- `--bind-to socket`: Binds process to a CPU socket (process can migrate between cores of that socket).
- `--bind-to none`: Disables binding (OS can migrate processes anywhere).

```bash
# Production HPC Command for maximum memory bandwidth:
mpirun -np 8 --map-by socket --bind-to core ./simulation
```

---

### 3. Running under SLURM Workload Manager

In supercomputing clusters, SLURM automates resource allocation:

```bash
#!/bin/bash
#SBATCH --job-name=mpi_sim
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=16
#SBATCH --time=01:00:00
#SBATCH --partition=compute

# Under SLURM, srun replaces mpirun directly (inherits SLURM variables):
srun ./simulation
```

---

## 5. Compilable C Code Examples

### Example 1: Cluster Topology and Core Affinity Diagnostic Utility
This complete program queries process rank, physical hostname, process PID, and (on Linux) the exact CPU core ID to which each MPI process is bound, providing concrete proof of process distribution.

```c
/*
 * File: mpi_cluster_topology.c
 * Compilation:
 *   mpicc -O2 mpi_cluster_topology.c -o mpi_cluster_topology
 *
 * Execution (Test 1: Local run):
 *   mpirun -np 4 ./mpi_cluster_topology
 *
 * Execution (Test 2: Round-robin node mapping):
 *   mpirun -np 4 --map-by node ./mpi_cluster_topology
 */

#define _GNU_SOURCE
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>

int main(int argc, char** argv) {
    int rank, size;
    char hostname[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Get_processor_name(hostname, &name_len);

    /* Query physical CPU core running this thread on Linux */
    int active_cpu = sched_getcpu();

    /* Ensure clean, sequential output across ranks */
    for (int target_rank = 0; target_rank < size; ++target_rank) {
        if (rank == target_rank) {
            if (rank == 0) {
                printf("=================================================================\n");
                printf("  MPI CLUSTER TOPOLOGY AND CORE BINDING REPORT                   \n");
                printf("  Total Processes: %d                                            \n");
                printf("=================================================================\n", size);
            }

            printf("[Rank %02d] Host: %-15s | PID: %-7d | Active CPU Core: %02d\n",
                   rank, hostname, (int)getpid(), active_cpu);
            fflush(stdout);
        }
        /* Synchronize across ranks so output prints without mangling */
        MPI_Barrier(MPI_COMM_WORLD);
    }

    if (rank == 0) {
        printf("=================================================================\n");
        printf("Topology diagnostic completed successfully.\n");
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running locally on an 8-core CPU) ---
=================================================================
  MPI CLUSTER TOPOLOGY AND CORE BINDING REPORT                   
  Total Processes: 4                                            
=================================================================
[Rank 00] Host: workstation     | PID: 24901   | Active CPU Core: 00
[Rank 01] Host: workstation     | PID: 24902   | Active CPU Core: 01
[Rank 02] Host: workstation     | PID: 24903   | Active CPU Core: 02
[Rank 03] Host: workstation     | PID: 24904   | Active CPU Core: 03
=================================================================
Topology diagnostic completed successfully.
```

---

## 6. Mathematical Formulations & Launch Topology Models

### Multi-Node Process Mapping Formulations

Let $N_{\text{nodes}}$ be the number of physical nodes, and $C_{\text{node}}$ be the number of CPU cores per node. The total available hardware cores across the cluster is:

$$C_{\text{total}} = N_{\text{nodes}} \cdot C_{\text{node}}$$

Let $P$ be the number of MPI processes requested via `-np <P>`.

#### 1. Block Mapping (`--map-by core` or `--map-by socket`)
In block mapping, processes fill all cores of Node 0 before placing any processes on Node 1:

$$\text{NodeID}(r) = \left\lfloor \frac{r}{C_{\text{node}}} \right\rfloor$$
$$\text{LocalCore}(r) = r \pmod{C_{\text{node}}}$$

- **Advantage:** Minimizes inter-node network traffic for adjacent ranks ($r$ and $r+1$ share high-speed on-node memory).
- **Disadvantage:** Memory bus contention on early nodes if $P < C_{\text{total}}$.

#### 2. Cyclic / Round-Robin Mapping (`--map-by node`)
In cyclic mapping, consecutive ranks are distributed across different nodes:

$$\text{NodeID}(r) = r \pmod{N_{\text{nodes}}}$$
$$\text{LocalCore}(r) = \left\lfloor \frac{r}{N_{\text{nodes}}} \right\rfloor$$

- **Advantage:** Maximizes total aggregate memory bandwidth when running with fewer processes than total cores (each rank gets an entire node's memory bus to itself).
- **Disadvantage:** Every neighbor communication requires traversing the physical network interconnect.

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Launch Mechanisms

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Compiler Wrapper (`mpicc`)| - Zero manual flag management           | - Hides underlying compiler; errors may |
|                           | - Automatically matches installed MPI   |   be confusing if wrapper is misaligned |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `--oversubscribe`         | - Permits testing 16-rank codes on a    | - Severe performance collapse in real   |
|                           |   4-core laptop                         |   compute; massive context-switch waste |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Core Binding              | - Eliminates OS thread migration        | - If combined with OpenMP improperly,   |
| (`--bind-to core`)        | - Guarantees NUMA cache locality        |   all OpenMP threads squeeze on 1 core! |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Compiler Wrapper (`mpicc` / `mpicxx`):** A shell or binary tool provided by MPI implementations that invokes the system C/C++ compiler while automatically injecting implementation-specific header search paths, library directories, and link flags for the MPI runtime.
>
> **Process Launcher (`mpirun` / `mpiexec`):** A runtime management utility that initializes parallel execution across local or remote cluster nodes, spawns the designated number of processes, establishes communication daemons, and configures `MPI_COMM_WORLD`.
>
> **Hostfile (Machinefile):** A configuration text file supplied to the process launcher specifying the network hostnames or IP addresses of cluster nodes and the number of processor slots available on each node.
>
> **Oversubscription:** A runtime execution state where the number of active MPI processes assigned to a node exceeds the physical number of CPU cores available, forcing the operating system to time-slice processes and degrading performance.
>
> **Core Binding (Processor Affinity):** An operating system mechanism configured by the MPI launcher that binds an MPI process to a specific physical CPU core, preventing thread migration across sockets and preserving CPU cache locality.

---

## 9. Comparison Tables

### `mpirun` vs `mpiexec` vs `srun`

| Tool | Standardization | Primary Environment | Process Management |
| :--- | :--- | :--- | :--- |
| **`mpiexec`** | Standardized by MPI Forum (MPI-2) | Cross-platform, portable scripts | Driven by implementation daemon (e.g., Hydra) |
| **`mpirun`** | Legacy convention (de-facto standard) | Local workstations, Open MPI | Often a symlink to `mpiexec` or `orterun` |
| **`srun`** | Proprietary to SLURM Workload Mgr | Production Supercomputers | Direct kernel task launch via SLURM `slurmd` |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Analyzing the Oversubscription Performance Cliff
**Question:** A student benchmarks an MPI matrix multiplication code on a 4-core laptop.
- Running with `-np 4` takes **$2.5\text{ seconds}$**.
- Running with `-np 16 --oversubscribe` takes **$18.4\text{ seconds}$** ($7.3\times$ slower!).
Explain the physical and operating system architectural reasons for this performance collapse.

**Solution:**
1. **CPU Core Saturation:** The physical hardware possesses only 4 physical cores. Running 16 processes cannot increase compute throughput because the ALUs and FPUs are already $100\%$ saturated at 4 processes.
2. **Context-Switching Thrashing:** The operating system kernel must constantly preempt each of the 16 processes to maintain scheduling fairness. Each context switch flushes CPU register states and invalidates L1/L2 instruction and data caches.
3. **Lock / Polling Starvation:** When an MPI process calls a blocking receive (`MPI_Recv`), modern MPI implementations aggressively **spin-poll** the network card for low latency. When 4 processes spin on a 4-core machine, they starve the other 12 processes of CPU time, preventing the sender from ever running to send the data!

---

### Problem 2: The OpenMP Core-Binding Trap in Hybrid MPI+OpenMP
**Question:** An engineer compiles a hybrid MPI+OpenMP program. She runs:
```bash
export OMP_NUM_THREADS=4
mpirun -np 2 --bind-to core ./hybrid_app
```
She notices that CPU utilization never exceeds $100\%$ on each node, and parallel speedup is nonexistent. Identify the mistake.

**Solution:**
- **The Defect:** `--bind-to core` instructs the MPI launcher to bind the **entire MPI process** to exactly **one single physical CPU core**.
- When OpenMP initializes within that process and spawns 4 OpenMP threads, the operating system kernel is legally restricted by the affinity mask to schedule all 4 OpenMP threads on that **single core**!
- All 4 OpenMP threads are forced to time-slice on 1 core, destroying speedup.
- **The Fix:** Bind to socket (`--bind-to socket`) or bind to a range of cores (`--bind-to none` or `--map-by socket:PE=4`):
```bash
mpirun -np 2 --map-by socket:PE=4 --bind-to core ./hybrid_app
```

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Is `mpicc` an actual compiler like `gcc` or `clang`?
**Answer:** No. `mpicc` is not a standalone compiler. It is a **compiler wrapper**. It parses the command-line arguments, injects the necessary MPI header include directories (`-I`), library paths (`-L`), and dynamic link libraries (`-lmpi`), and then forwards the augmented command directly to the system's underlying native compiler (such as `gcc`, `clang`, or `icc`). You can verify this by executing `mpicc -show`.

### Q2: Why does executing `./my_mpi_prog` directly in bash (without `mpirun`) produce unexpected behavior?
**Answer:** Running `./my_mpi_prog` directly launches a single normal operating system process. Because no process launcher was invoked, no network daemons or peer processes exist. `MPI_COMM_WORLD` is formed with a size of exactly $1$ (`rank = 0`, `size = 1`). If the code contains point-to-point communication expecting Rank 1 to send data, the program will hang indefinitely in a deadlock.

### Q3: What is the purpose of the `--oversubscribe` flag?
**Answer:** By default, Open MPI prevents you from launching more processes than physical CPU cores detected on the machine to protect system performance. The `--oversubscribe` flag overrides this safety check, forcing the launcher to spawn the requested number of processes regardless of core count. This is useful for functional debugging of multi-rank algorithms on personal laptops with limited cores.

---

## 12. Quick Revision Sheet

- **Compiler Wrapper:** `mpicc` wraps native compiler (`gcc`), automatically injecting `-I`, `-L`, `-lmpi`.
- **Process Launcher:** `mpirun` / `mpiexec` launches $P$ processes across nodes and sets up `MPI_COMM_WORLD`.
- **Key Flags:**
  - `-np <P>`: Number of processes.
  - `--hostfile <file>`: Specifies node list and available slots.
  - `--oversubscribe`: Allows processes $>$ CPU cores (for debugging only).
  - `--bind-to core`: Binds processes to dedicated physical cores (prevents OS migration).
- **Inspecting Wrappers:** Use `mpicc -show` to see the underlying compiler commands.
- **SLURM Integration:** In HPC clusters, `srun` replaces `mpirun` and automatically reads cluster allocations.

---

## 13. What You Should Be Able to Do

1. **Compile MPI C applications** using `mpicc` with optimization and debug flags.
2. **Launch multi-process applications** across local and multi-node clusters using `mpirun` and `mpiexec`.
3. **Construct hostfiles** specifying node names and core slot capacities.
4. **Inspect process-to-core binding and NUMA mapping** using `--display-map` and `sched_getcpu()`.
5. **Debug performance issues** arising from core-oversubscription and incorrect hybrid thread binding.
