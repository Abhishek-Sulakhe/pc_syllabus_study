# Module 10: MPI Distributed-Memory Foundations (Lectures 24–25)
# Subtopic 10.01: Distributed-Memory Architecture and the Message Passing Model (SPMD, Interconnects, MPI vs. OpenMP)

---

## 1. Basic Idea

In shared-memory parallel programming (such as OpenMP and POSIX Threads), all execution threads run within a single operating system process and read or write to a **single, unified virtual address space**. While shared-memory programming is intuitive, physical hardware constraints impose a hard ceiling: no single motherboard or server chassis can scale beyond a few dozen or hundreds of CPU cores without facing insurmountable memory bus contention and NUMA latency penalties.

To achieve extreme-scale High-Performance Computing (HPC)—powering weather forecasting, molecular dynamics, climate modeling, and large language model training—computational workloads must run across **thousands of independent server nodes** connected by high-speed networks. This is the **Distributed-Memory Architecture**.

In a distributed-memory system:
- Each compute node contains its own independent CPUs and its own **completely isolated, private physical RAM**.
- Processors on Node $A$ **cannot directly read or write** to the memory addresses of Node $B$.
- Coordination and data exchange must occur through explicit, network-mediated packet transfers. This programming paradigm is called the **Message Passing Model**, standardized globally as the **Message Passing Interface (MPI)**.

```
+-----------------------------------------------------------------------------------------+
|                  Shared-Memory vs. Distributed-Memory Architecture                      |
+-----------------------------------------------------------------------------------------+
| SHARED-MEMORY (OpenMP / Pthreads):                                                      |
|   [ Core 0 ]   [ Core 1 ]   [ Core 2 ]   [ Core 3 ]                                     |
|        \            |            |            /                                         |
|         +-----------+------------+-----------+  (Shared Memory Bus)                     |
|                              |                                                          |
|                    [ GLOBAL SHARED RAM ]  (Single Unified Address Space)                |
|                                                                                         |
| DISTRIBUTED-MEMORY (MPI):                                                               |
|   +-----------------------+               +-----------------------+                     |
|   | NODE 0 (Process 0)    |               | NODE 1 (Process 1)    |                     |
|   |  [CPU] <-> [RAM 0]    |               |  [CPU] <-> [RAM 1]    |                     |
|   +-----------+-----------+               +-----------+-----------+                     |
|               |                                       |                                 |
|               +============ High-Speed Interconnect ==+                                 |
|                             (InfiniBand / RoCE / Slingshot)                             |
|                             Explicit: MPI_Send() -> MPI_Recv()                          |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Breaking the Single-Node Hardware Scaling Wall
No supercomputer on the TOP500 list is a single shared-memory machine. For example, Frontier (the world's first exascale supercomputer) contains over $8{,}699{,}904$ combined CPU and GPU cores distributed across $9{,}472$ nodes. 
- It is physically impossible to route memory traces from 8 million cores to a single shared RAM bank.
- Shared-memory cache coherence protocols (MESI/MOESI) generate an exponential volume of snoop broadcasts, collapsing bus bandwidth beyond $\approx 64\text{–}128$ cores.
- Distributed memory bypasses this limitation: each node operates independently, communicating over a scalable switched fabric.

### 2. High Latency Discrepancy (The Memory-Network Gap)
In shared-memory programming, data access latency between cores is measured in nanoseconds:
- L1 Cache access: $\approx 1\text{ ns}$
- Local DRAM access: $\approx 60\text{ to }80\text{ ns}$
- Remote NUMA DRAM access: $\approx 150\text{ to }250\text{ ns}$

In distributed systems, data must traverse Network Interface Cards (NICs), physical cables, and switches:
- Commodity Gigabit Ethernet: $\approx 15{,}000\text{ to }50{,}000\text{ ns}$ ($15\text{ to }50\,\mu\text{s}$)
- HPC InfiniBand (HDR/NDR) / HPE Slingshot: $\approx 600\text{ to }1{,}500\text{ ns}$ ($0.6\text{ to }1.5\,\mu\text{s}$)

Because network latency is **$10\times\text{ to }1000\times$ slower** than local RAM access, distributed-memory programming requires explicit, deliberate algorithmic design to minimize communication frequency, pack messages into contiguous buffers, and overlap network transmission with local computation.

### 3. Fault Isolation and Operating System Independence
In a shared-memory Pthreads application, if one thread encounters a null-pointer segmentation fault (`SIGSEGV`), the **entire process crashes**, killing all sister threads instantly. In an MPI distributed-memory application, each rank is a completely isolated operating system process running in its own protected virtual memory space. A node or process failure can be trapped, contained, or recovered through checkpointing without corrupting the physical memory of surviving nodes.

---

## 3. How It Works

### The SPMD (Single Program, Multiple Data) Execution Model

MPI applications follow the **SPMD** programming paradigm:
- A single compiled binary executable is distributed and launched simultaneously across $P$ distinct processes.
- Every process runs the **exact same compiled code**.
- However, each process is assigned a unique integer identifier called its **Rank** ($0, 1, 2, \dots, P-1$).
- Processes inspect their own rank to branch into distinct computational roles (e.g., Rank 0 acts as coordinator/master, while Ranks $1 \dots P-1$ act as domain workers).

```
                    Compiled Program: 'mpi_simulation'
                                   |
                  mpirun -np 4 ./mpi_simulation
                                   |
    +------------------------------+------------------------------+
    |                              |                              |
    v                              v                              v
[ Process 0 ]                 [ Process 1 ]                 [ Process 2 ]
Rank = 0                      Rank = 1                      Rank = 2
if (rank == 0) {              if (rank == 0) {              if (rank == 0) {
  read_input();                 read_input();                 read_input();
  broadcast();                  broadcast();                  broadcast();
} else {                      } else {                      } else {
  compute_slice(); <--- RUNS    compute_slice(); <--- RUNS    compute_slice(); <--- RUNS
}                             }                             }
```

### The Message-Passing Lifecycle

When Rank 0 transmits an array of floating-point numbers to Rank 1:

```
Rank 0 (Sender Address Space)                  Rank 1 (Receiver Address Space)
+-----------------------------+                +-----------------------------+
| User Buffer: double data[N] |                | User Buffer: double recv[N] |
+-----------------------------+                +-----------------------------+
              |                                               ^
              | MPI_Send(...)                                 | MPI_Recv(...)
              v                                               |
+-----------------------------+                +-----------------------------+
| MPI Library / Kernel Buffer |                | MPI Library / Kernel Buffer |
+-----------------------------+                +-----------------------------+
              |                                               ^
              v                                               |
       [ Host NIC 0 ] ======= Physical Network Wire ======> [ Host NIC 1 ]
                             (Packets / InfiniBand FLITs)
```

1. **Sender Packaging:** Rank 0 specifies the memory buffer, data count, MPI datatype, destination rank, unique tag, and communicator.
2. **Network Protocol Transmission:** The underlying MPI transport layer (e.g., UCX, Libfabric) breaks the data into packets and routes them across switches using protocols such as Eager (for small messages) or Rendezvous (for large messages).
3. **Receiver Matching:** Rank 1 posts an `MPI_Recv()` specifying matching sender rank, tag, and communicator.
4. **Buffer Extraction:** The incoming data is copied from network packets directly into Rank 1's local memory buffer.

---

## 4. Architectural Comparison: MPI vs. OpenMP vs. Pthreads

The three dominant parallel programming paradigms serve distinct architectural tiers:

```
+---------------------------+-----------------------+-----------------------+-----------------------+
| Architectural Dimension   | OpenMP                | POSIX Threads (Pthreads)| MPI (Message Passing) |
+---------------------------+-----------------------+-----------------------+-----------------------+
| Hardware Target           | Single Multi-Core Node| Single Multi-Core Node| Distributed Clusters  |
| Memory Model              | Shared Virtual Memory | Shared Virtual Memory | Private Memory/Node   |
| Address Space             | Single Process        | Single Process        | Multiple Processes    |
| Inter-Process Scaling     | Single Machine Only   | Single Machine Only   | 100,000+ Nodes        |
| Data Exchange Mechanism   | Direct Memory Read/Wr | Direct Memory Read/Wr | Explicit Send/Receive |
| Syntax / Approach         | Compiler Pragma Direct| Explicit C API Library| Explicit C/C++ API    |
| Race Conditions           | Severe Risk           | Severe Risk           | Impossible (No shared)|
| Communication Bottleneck  | Memory Bus Bandwidth  | Memory Bus Bandwidth  | Network Fabric Latency|
+---------------------------+-----------------------+-----------------------+-----------------------+
```

---

## 5. Compilable C Code Examples

### Example 1: Foundational MPI SPMD Rank and Size Discovery
This complete program initializes the MPI runtime environment, queries the total process team size (`MPI_Comm_size`), queries each process's unique identifier (`MPI_Comm_rank`), retrieves the physical host name of the node where each process is executing, and cleanly finalizes the environment.

```c
/*
 * File: mpi_spmd_foundations.c
 * Compilation:
 *   mpicc -O2 mpi_spmd_foundations.c -o mpi_spmd_foundations
 *
 * Execution (Local multi-process launch):
 *   mpirun -np 4 ./mpi_spmd_foundations
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    int rank;
    int world_size;
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    /* Step 1: Initialize the MPI execution environment */
    int rc = MPI_Init(&argc, &argv);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "Fatal: MPI_Init failed with error code %d\n", rc);
        MPI_Abort(MPI_COMM_WORLD, rc);
    }

    /* Step 2: Determine total number of cooperating processes in the communicator */
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    /* Step 3: Determine the unique rank (0 .. world_size-1) of this process */
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    /* Step 4: Query the physical machine / hostname running this specific rank */
    MPI_Get_processor_name(processor_name, &name_len);

    /* SPMD Branching: Execution differs based on rank value */
    if (rank == 0) {
        printf("=================================================================\n");
        printf("  MPI DISTRIBUTED-MEMORY FOUNDATION: SPMD EXECUTION ENGINE       \n");
        printf("  Total Processes Launched (MPI_COMM_WORLD Size): %d             \n");
        printf("=================================================================\n", world_size);
    }

    /* Every rank executes this diagnostic printout independently */
    printf("[MPI Process %d of %d] Executing on Host: %s | Private PID: %d\n",
           rank, world_size, processor_name, (int)getpid());

    /* Collective synchronization: Ensure all processes reach this point */
    MPI_Barrier(MPI_COMM_WORLD);

    if (rank == 0) {
        printf("-----------------------------------------------------------------\n");
        printf("[Master Rank 0] All %d distributed ranks reached synchronization barrier.\n", world_size);
        printf("Shutting down MPI environment cleanly.\n");
    }

    /* Step 5: Finalize MPI execution environment */
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
=================================================================
  MPI DISTRIBUTED-MEMORY FOUNDATION: SPMD EXECUTION ENGINE       
  Total Processes Launched (MPI_COMM_WORLD Size): 4             
=================================================================
[MPI Process 0 of 4] Executing on Host: compute-node-01 | Private PID: 40912
[MPI Process 1 of 4] Executing on Host: compute-node-01 | Private PID: 40913
[MPI Process 2 of 4] Executing on Host: compute-node-02 | Private PID: 18451
[MPI Process 3 of 4] Executing on Host: compute-node-02 | Private PID: 18452
-----------------------------------------------------------------
[Master Rank 0] All 4 distributed ranks reached synchronization barrier.
Shutting down MPI environment cleanly.
```

---

## 6. Mathematical Formulations & Network Communication Models

### 1. The Hockney Network Communication Model

In distributed-memory computing, the time $T_{\text{comm}}(m)$ required to transfer a message of size $m$ bytes between two nodes over an interconnect is modeled by the classic **Hockney Model**:

$$T_{\text{comm}}(m) = \alpha + \beta \cdot m = \alpha + \frac{m}{B}$$

Where:
- $\alpha$ ($\text{seconds}$): **Network Latency** (Startup time). The constant overhead to initiate the message, negotiate protocols, traverse software stack layers, and enter the physical wire.
- $\beta$ ($\text{seconds/byte}$): **Transfer Time per Byte**, equal to the reciprocal of asymptotic bandwidth ($\beta = \frac{1}{B}$).
- $B$ ($\text{bytes/second}$): **Network Bandwidth**.
- $m$ ($\text{bytes}$): **Payload Size**.

```
    Transmission Time
         ^
         |                                       / (Slope = 1 / B = beta)
         |                                      /
         |                                     /
         |                                    /
         |                                   /
         |                                  /
         |                                 /
  alpha  +--------------------------------+
         |
         +--------------------------------------------> Message Size (m bytes)
         0
```

### 2. Network Latency vs Bandwidth Dominance (Half-Performance Length)
The message size $m_{1/2}$ at which half the asymptotic bandwidth is achieved (the **half-performance length**) is:

$$m_{1/2} = \frac{\alpha}{\beta} = \alpha \cdot B$$

#### Analytical Comparison: Ethernet vs. InfiniBand
Suppose an application transmits small telemetry packets ($m = 64\text{ bytes}$):

| Metric | 10 Gigabit Ethernet (TCP/IP) | InfiniBand HDR (200 Gbps RDMA) |
| :--- | :--- | :--- |
| **Latency ($\alpha$)** | $25\,\mu\text{s} = 25{,}000\text{ ns}$ | $0.8\,\mu\text{s} = 800\text{ ns}$ |
| **Bandwidth ($B$)** | $1.25\text{ GB/s}$ | $25.0\text{ GB/s}$ |
| **Time for 64 Bytes** | $25{,}000\text{ ns} + \frac{64}{1.25} \approx \mathbf{25{,}051\text{ ns}}$ | $800\text{ ns} + \frac{64}{25.0} \approx \mathbf{802\text{ ns}}$ |
| **Dominant Factor** | **$99.8\%$ Latency ($\alpha$)** | **$99.7\%$ Latency ($\alpha$)** |

*Crucial HPC Principle:* For small messages, bandwidth ($B$) is completely irrelevant; transmission time is governed entirely by startup latency ($\alpha$). High-performance MPI programming demands **message aggregation**—bundling millions of small messages into large contiguous arrays to overcome $\alpha$.

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Distributed-Memory Trade-offs

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Distributed Memory        | - Unlimited hardware scalability        | - Network latency is 100x slower than   |
| (MPI Architecture)        | - Node failure isolation                |   local RAM                             |
|                           | - Zero shared-memory race conditions    | - High algorithmic restructuring cost   |
+---------------------------+-----------------------------------------+-----------------------------------------+
| SPMD Programming Model    | - Single codebase to maintain & compile | - Requires explicit rank branching logic|
|                           | - Highly predictable execution flow     | - Straggler processes waste global time |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Explicit Message Passing  | - Absolute developer control over data  | - Complex memory packing & buffering    |
|                           |   movement and network transmission     | - Vulnerable to network deadlocks       |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Distributed-Memory Architecture:** A parallel computing architecture consisting of multiple independent processing nodes, where each node possesses its own private physical memory space inaccessible to other nodes, requiring communication via a network interconnect.
>
> **Message Passing Interface (MPI):** A standardized, portable message-passing system specification designed by the MPI Forum to function on a wide range of parallel computing architectures.
>
> **SPMD (Single Program, Multiple Data):** A parallel programming execution model where identical copies of the same compiled executable run concurrently across multiple processes, each operating on distinct subsets of data determined by its unique process rank.
>
> **Communicator (`MPI_Comm`):** An opaque MPI object representing a designated group of communicating processes accompanied by an isolated communication context, ensuring that messages sent within one communicator cannot collide with messages in another.
>
> **Rank:** An integer assigned uniquely to each process within a given communicator, ranging from `0` to `size - 1`, used to specify message sources, destinations, and computational data partitions.

---

## 9. Comparison Tables

### Shared Memory vs Distributed Memory

| Dimension | Shared Memory Architecture | Distributed Memory Architecture |
| :--- | :--- | :--- |
| **Physical Memory** | Single shared RAM space | Disjoint private RAM per node |
| **Interconnect** | On-chip bus / Crossbar / QPI | InfiniBand, Ethernet, Slingshot |
| **Process Scale** | 1 to $\approx 128$ cores typically | $1$ to $1{,}000{,}000+$ cores |
| **Primary Programming API**| OpenMP, POSIX Threads (Pthreads)| MPI (Message Passing Interface) |
| **Data Access Method** | Direct pointer dereferencing | Explicit network send / receive |
| **Data Races on Variables**| **Constant hazard** (shared memory) | **Impossible** (variables are process-private) |
| **Fault Isolation** | None (one thread crash kills all) | High (isolated process memory spaces) |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Hockney Model Calculation for Array Distribution
**Question:** A cluster node sends an array of $1{,}000{,}000$ double-precision floats ($8\text{ bytes}$ each) to a neighbor node over an InfiniBand interconnect with latency $\alpha = 1.2\,\mu\text{s}$ and bandwidth $B = 12.5\text{ GB/s}$.
1. Calculate the total transmission time.
2. If the application instead sends $1{,}000{,}000$ individual floats as separate messages, calculate the total transmission time.
3. Determine the speedup achieved by bundling the data into a single message.

**Solution:**

**Part 1: Single Bundled Message**
- Payload size $m = 1{,}000{,}000 \times 8\text{ bytes} = 8{,}000{,}000\text{ bytes} = 8\text{ MB}$.
- Bandwidth $B = 12.5 \times 10^9\text{ bytes/sec}$.
$$T_{\text{single}} = \alpha + \frac{m}{B} = 1.2 \times 10^{-6} + \frac{8 \times 10^6}{12.5 \times 10^9} = 1.2 \times 10^{-6} + 6.4 \times 10^{-4} = 0.0006412\text{ seconds} \approx \mathbf{0.641\text{ ms}}$$

**Part 2: Million Individual Messages**
- For each message, $m_i = 8\text{ bytes}$.
$$T_{\text{indiv}} = \alpha + \frac{m_i}{B} \approx 1.2 \times 10^{-6}\text{ seconds}$$
- Total time for $1{,}000{,}000$ messages:
$$T_{\text{total\_indiv}} = 10^6 \times 1.2 \times 10^{-6} = \mathbf{1.200\text{ seconds}}$$

**Part 3: Speedup Factor**
$$\text{Speedup} = \frac{T_{\text{total\_indiv}}}{T_{\text{single}}} = \frac{1.200\text{ s}}{0.0006412\text{ s}} \approx \mathbf{1{,}871.5\times}$$

*Conclusion:* Sending one aggregated array is almost **$2{,}000$ times faster** than sending individual elements, directly demonstrating why message consolidation is mandatory in MPI programming.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why do MPI programs not require mutexes to protect variables from data races?
**Answer:** Because in MPI, every process runs in its own **completely isolated virtual address space**. Process 0 has its own private global and heap variables; Process 1 has entirely separate physical RAM. When Process 0 modifies a variable, it is physically impossible for Process 1's CPU to access or corrupt that memory directly. Data races can only occur when memory is shared. In MPI, data exchange is always explicit and mediated by network buffers.

### Q2: What is the meaning of SPMD and how does it allow different processes to do different tasks?
**Answer:** SPMD stands for **Single Program, Multiple Data**. All processes run identical binary machine instructions. However, each process queries its unique integer ID (its **rank**) via `MPI_Comm_rank()`. Programmers write conditional branches based on rank (`if (rank == 0) { ... } else { ... }`), enabling processes to read different data slices, execute distinct computational subroutines, or assume master/worker roles while executing the exact same binary.

### Q3: Why is MPI considered a specification rather than an implementation?
**Answer:** MPI is a standardized document defined by the **MPI Forum** (an international consortium of vendors, researchers, and supercomputing centers) specifying function names, parameter syntax, and semantic behavioral guarantees. It is not code. Concrete software implementations of this standard are developed independently—including **Open MPI**, **MPICH**, **MVAPICH**, and vendor-optimized distributions like **Intel MPI** and **Cray MPICH**.

---

## 12. Quick Revision Sheet

- **Distributed Memory:** Multiple nodes, private RAM per node, network interconnect. Scales to millions of cores.
- **Message Passing Model:** No shared memory. All data movement is explicit via send/receive network calls.
- **SPMD:** Single Program, Multiple Data. Same executable launched everywhere; behavior branches via `rank`.
- **Hockney Model:** $T_{\text{comm}} = \alpha + \frac{m}{B}$.
  - $\alpha$: Latency (startup time). Dominates small messages.
  - $B$: Bandwidth. Dominates large messages.
- **Key MPI Constants & Communicator:**
  - `MPI_COMM_WORLD`: Default communicator containing all launched processes.
  - `MPI_Init(&argc, &argv)`: Initializes MPI.
  - `MPI_Finalize()`: Shuts down MPI.
  - `MPI_Comm_size()`: Total number of processes.
  - `MPI_Comm_rank()`: Process rank identifier ($0 \dots \text{size}-1$).

---

## 13. What You Should Be Able to Do

1. **Articulate the architectural differences** between shared-memory (OpenMP/Pthreads) and distributed-memory (MPI) systems.
2. **Calculate communication times** and identify latency-dominated regimes using the Hockney network model.
3. **Write, compile, and run** a foundational SPMD MPI program querying rank, size, and hostname.
4. **Explain why data races on shared memory variables cannot occur** in pure distributed-memory MPI code.
5. **Justify why message aggregation** is critical for HPC performance over high-latency interconnects.
