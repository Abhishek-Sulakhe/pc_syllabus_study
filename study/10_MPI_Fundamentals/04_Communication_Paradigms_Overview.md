# Module 10: MPI Distributed-Memory Foundations (Lectures 24–25)
# Subtopic 10.04: MPI Communication Paradigms Overview (Point-to-Point, Collectives, One-Sided RMA, Dynamic Ports)

---

## 1. Basic Idea

In the Message Passing Interface (MPI), distributed processes operate in disjoint physical address spaces. Coordination, state synchronization, and data movement must occur through well-defined **communication paradigms**.

The MPI standard structures its communication capabilities into four distinct architectural tiers:

```
+-----------------------------------------------------------------------------------------+
|                              The Four MPI Communication Paradigms                       |
+-----------------------------------------------------------------------------------------+
| 1. POINT-TO-POINT COMMUNICATION (Lectures 26–27):                                       |
|    - Two-party exchange between exactly ONE sender process and ONE receiver process.    |
|    - Driven by explicit message envelopes: [Buffer, Count, Type, Rank, Tag, Comm].      |
|    - Modes: Blocking (MPI_Send/Recv) vs Non-Blocking (MPI_Isend/Irecv).                 |
|                                                                                         |
| 2. COLLECTIVE COMMUNICATION (Lectures 28–30):                                          |
|    - Multi-party exchange involving EVERY process in an entire communicator.            |
|    - Patterns: One-to-All (Bcast, Scatter), All-to-One (Gather, Reduce),               |
|      All-to-All (Allgather, Alltoall), Synchronization (MPI_Barrier).                   |
|                                                                                         |
| 3. ONE-SIDED COMMUNICATION / RMA (Lecture 31):                                          |
|    - Remote Memory Access (RMA) where one process accesses remote memory directly       |
|      without explicit participation of the remote process.                              |
|    - Operations: MPI_Put (Remote write), MPI_Get (Remote read), MPI_Accumulate.         |
|    - Synchronization: Epoch fences (MPI_Win_fence) and access/exposure epochs.          |
|                                                                                         |
| 4. DYNAMIC PROCESS & CLIENT-SERVER ARCHITECTURE (Lecture 28):                           |
|    - Spawning new processes dynamically at runtime (MPI_Comm_spawn).                    |
|    - Independent programs connecting via network ports (MPI_Open_port, MPI_Comm_connect)|
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Expressiveness Across Algorithmic Topologies
Different scientific algorithms exhibit vastly different communication topologies:
- **Neighbor Exchanges (Stencils / PDEs):** In a 2D weather simulation, each process computes a grid tile and needs data only from its North, South, East, and West neighbors. This is fundamentally a **Point-to-Point** communication pattern.
- **Global Parameter Distribution & Reductions:** Broadcasting simulation hyperparameters or finding the global minimum error across 10,000 ranks is fundamentally a **Collective Operation**. Implementing a global reduction with manual point-to-point sends requires complex logarithmic tree algorithms; MPI collectives provide this with optimized hardware offload in a single function call.
- **Irregular / Graph Traversal:** In sparse graph analytics or distributed hash tables, a process does not know in advance which peer will access its data. **One-Sided RMA** decouples sender from receiver, allowing a thread to read or write remote memory directly.
- **Client-Server Service Coupling:** Autonomous services (e.g., a real-time visualization client connecting to an active supercomputer simulation) require **Dynamic Ports** to establish communication dynamically without sharing an initial `MPI_COMM_WORLD`.

---

## 3. How It Works

### Taxonomy of MPI Communication Paradigms

```
                                  MPI COMMUNICATION
                                          |
         +--------------------------------+--------------------------------+
         |                                                                 |
   Two-Sided Communication                                       One-Sided Communication
   (Sender & Receiver both call MPI)                             (Only initiator calls MPI)
         |                                                                 |
   +-----+-----+                                                           v
   |           |                                                  Remote Memory Access (RMA)
   v           v                                                  - Memory Windows (MPI_Win)
Point-to-Point Collectives                                        - MPI_Put / MPI_Get
(1-to-1)       (All-in-Comm)                                      - Epoch Fences
   |           |
   |           +---> Synchronization : MPI_Barrier
   |           +---> Data Movement   : Broadcast, Scatter, Gather, Alltoall
   |           +---> Reductions      : MPI_Reduce, MPI_Allreduce, Scan
   |
   +---> Blocking     : MPI_Send, MPI_Recv
   +---> Non-Blocking : MPI_Isend, MPI_Irecv (Overlaps compute & network)
```

---

### Point-to-Point vs. Collective Comparison

```
POINT-TO-POINT (1-to-1):
  Rank 0:  MPI_Send(data, count, MPI_INT, 1, tag, comm)
  Rank 1:  MPI_Recv(data, count, MPI_INT, 0, tag, comm, &status)
  ===> Only Ranks 0 and 1 participate. Other ranks in comm are completely uninvolved.

COLLECTIVE (All-to-All / One-to-All):
  Rank 0:  MPI_Bcast(data, count, MPI_INT, 0, comm)
  Rank 1:  MPI_Bcast(data, count, MPI_INT, 0, comm)
  Rank 2:  MPI_Bcast(data, count, MPI_INT, 0, comm)
  Rank 3:  MPI_Bcast(data, count, MPI_INT, 0, comm)
  ===> EVERY rank in comm MUST call MPI_Bcast() with identical arguments!
       If even ONE rank omits the call, the entire cluster hangs in a permanent deadlock!
```

---

## 4. Important Mechanisms and Functions

### 1. Point-to-Point Primitives (Preview)

```c
int MPI_Send(const void *buf, int count, MPI_Datatype datatype, int dest,
             int tag, MPI_Comm comm);
int MPI_Recv(void *buf, int count, MPI_Datatype datatype, int source,
             int tag, MPI_Comm comm, MPI_Status *status);
int MPI_Isend(const void *buf, int count, MPI_Datatype datatype, int dest,
              int tag, MPI_Comm comm, MPI_Request *request);
int MPI_Irecv(void *buf, int count, MPI_Datatype datatype, int source,
              int tag, MPI_Comm comm, MPI_Request *request);
int MPI_Wait(MPI_Request *request, MPI_Status *status);
```

### 2. Collective Primitives (Preview)

```c
/* Synchronization */
int MPI_Barrier(MPI_Comm comm);

/* Data Movement */
int MPI_Bcast(void *buffer, int count, MPI_Datatype datatype, int root, MPI_Comm comm);
int MPI_Scatter(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
                void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm);
int MPI_Gather(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
               void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm);

/* Global Reductions */
int MPI_Reduce(const void *sendbuf, void *recvbuf, int count, MPI_Datatype datatype,
               MPI_Op op, int root, MPI_Comm comm);
int MPI_Allreduce(const void *sendbuf, void *recvbuf, int count, MPI_Datatype datatype,
                  MPI_Op op, MPI_Comm comm);
```

### 3. One-Sided RMA Primitives (Preview)

```c
int MPI_Win_create(void *base, MPI_Aint size, int disp_unit, MPI_Info info,
                   MPI_Comm comm, MPI_Win *win);
int MPI_Put(const void *origin_addr, int origin_count, MPI_Datatype origin_datatype,
            int target_rank, MPI_Aint target_disp, int target_count,
            MPI_Datatype target_datatype, MPI_Win win);
int MPI_Get(void *origin_addr, int origin_count, MPI_Datatype origin_datatype,
            int target_rank, MPI_Aint target_disp, int target_count,
            MPI_Datatype target_datatype, MPI_Win win);
int MPI_Win_fence(int assert, MPI_Win win);
int MPI_Win_free(MPI_Win *win);
```

---

## 5. Compilable C Code Examples

### Example 1: Contrasting Point-to-Point Manual Ring vs. Collective Reduction
This complete program calculates the global sum of process ranks across all nodes using two contrasting approaches:
1. **Approach A (Point-to-Point Ring):** Manual transmission where each process passes an accumulator to its neighbor ($0 \to 1 \to 2 \to \dots \to P-1 \to 0$).
2. **Approach B (Collective `MPI_Allreduce`):** A single collective call computing the identical sum using vendor-optimized logarithmic reduction trees.

```c
/*
 * File: mpi_paradigms_contrast.c
 * Compilation:
 *   mpicc -O2 mpi_paradigms_contrast.c -o mpi_paradigms_contrast
 *
 * Execution:
 *   mpirun -np 4 ./mpi_paradigms_contrast
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    int rank, size;
    int local_value;
    int ring_sum = 0;
    int collective_sum = 0;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    local_value = rank + 1; /* Ranks 0..3 have values 1, 2, 3, 4 -> Sum = 10 */

    /* =========================================================================
     * PARADIGM 1: POINT-TO-POINT MANUAL RING SUMMATION
     * Each rank adds its value and passes to (rank + 1) % size
     * ========================================================================= */
    if (rank == 0) {
        /* Rank 0 starts the ring */
        ring_sum = local_value;
        printf("[P2P Ring] Rank 0 starting token with value: %d\n", ring_sum);
        MPI_Send(&ring_sum, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);

        /* Rank 0 waits for the completed token from the last rank */
        MPI_Recv(&ring_sum, 1, MPI_INT, size - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("[P2P Ring] Rank 0 received final token: %d\n", ring_sum);
    } else {
        /* Worker ranks receive from left neighbor, add local value, pass right */
        MPI_Recv(&ring_sum, 1, MPI_INT, rank - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        ring_sum += local_value;
        int next_rank = (rank + 1) % size;
        MPI_Send(&ring_sum, 1, MPI_INT, next_rank, 0, MPI_COMM_WORLD);
    }

    /* Synchronize before demonstrating collective */
    MPI_Barrier(MPI_COMM_WORLD);

    /* =========================================================================
     * PARADIGM 2: COLLECTIVE ALLREDUCE
     * A single, hardware-accelerated call executed by ALL ranks simultaneously
     * ========================================================================= */
    MPI_Allreduce(&local_value, &collective_sum, 1, MPI_INT, MPI_SUM, MPI_COMM_WORLD);

    printf("  [Rank %d] Collective MPI_Allreduce Result = %d (Expected = 10)\n",
           rank, collective_sum);

    /* Verify consistency */
    if (rank == 0) {
        printf("-----------------------------------------------------------------\n");
        if (ring_sum == collective_sum) {
            printf("SUCCESS: Both Point-to-Point and Collective produced Sum = %d!\n", collective_sum);
        } else {
            printf("ERROR: Mismatch between paradigms!\n");
        }
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[P2P Ring] Rank 0 starting token with value: 1
[P2P Ring] Rank 0 received final token: 10
  [Rank 0] Collective MPI_Allreduce Result = 10 (Expected = 10)
  [Rank 1] Collective MPI_Allreduce Result = 10 (Expected = 10)
  [Rank 2] Collective MPI_Allreduce Result = 10 (Expected = 10)
  [Rank 3] Collective MPI_Allreduce Result = 10 (Expected = 10)
-----------------------------------------------------------------
SUCCESS: Both Point-to-Point and Collective produced Sum = 10!
```

---

## 6. Mathematical Formulations & Complexity Scaling

### Algorithmic Complexity: Point-to-Point Ring vs. Tree Collective

Let $P$ be the number of processes in the communicator, and $m$ be the message payload size.

#### 1. Point-to-Point Ring Complexity
In a ring algorithm, the token must visit each process sequentially:

$$\text{Number of Network Hops} = P$$
$$T_{\text{ring}}(P) = P \cdot \left( \alpha + \beta \cdot m \right) = \mathcal{O}(P)$$

- **Scalability:** Linear complexity $\mathcal{O}(P)$. 
- On Frontier ($P = 100{,}000$ processes), a ring traversal takes $100{,}000$ sequential hops ($\approx 1\text{ to }2\text{ seconds}$), completely destroying scaling!

#### 2. Collective Tree Reduction Complexity (`MPI_Reduce` / `MPI_Allreduce`)
Modern MPI implementations implement collectives using binomial trees, recursive doubling, or ring-pipelining:

$$\text{Binomial Tree Hops} = \lceil \log_2(P) \rceil$$
$$T_{\text{tree}}(P) = \lceil \log_2(P) \rceil \cdot (\alpha + \beta \cdot m) = \mathcal{O}(\log P)$$

```
     Rank 0 <============== Binomial Tree (log2 P Steps) ==============>
     /    \                                                          /    \
  Rank 1  Rank 2                                                  Rank 4  Rank 6
            |                                                       |       |
          Rank 3                                                  Rank 5  Rank 7
```

#### Analytical Scaling Comparison across Core Counts ($P$):

| Process Count ($P$) | Ring Hops ($\mathcal{O}(P)$) | Tree Collective Hops ($\mathcal{O}(\log_2 P)$) | Collective Advantage |
| :--- | :--- | :--- | :--- |
| **$P = 4$** | $4$ hops | $2$ hops | $2\times$ faster |
| **$P = 64$** | $64$ hops | $6$ hops | $10.6\times$ faster |
| **$P = 1{,}024$** | $1{,}024$ hops | $10$ hops | **$102.4\times$ faster** |
| **$P = 65{,}536$** | $65{,}536$ hops | $16$ hops | **$4{,}096\times$ faster** |

*Lesson:* Whenever an algorithmic operation matches an MPI collective pattern, **always use the collective**. Writing manual point-to-point loops scales as $\mathcal{O}(P)$, whereas vendor MPI collectives scale logarithmically as $\mathcal{O}(\log P)$.

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Paradigm Comparison

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Paradigm                  | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Point-to-Point            | - Fine-grained neighbor communication   | - Vulnerable to network deadlocks       |
| (1-to-1)                  | - Non-blocking hides network latency    | - Linear scaling O(P) for global patterns|
+---------------------------+-----------------------------------------+-----------------------------------------+
| Collective Operations     | - Logarithmic scaling O(log P)          | - Every process in comm MUST call it!   |
| (All-in-Comm)             | - Optimized hardware multicast support  | - Difficult to use with irregular data  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| One-Sided RMA             | - Decoupled: target process does not call| - Complex memory consistency models    |
| (Put / Get)               | - Ideal for irregular graph algorithms  | - High synchronization overhead (fence) |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Point-to-Point Communication:** A two-party communication mechanism where exactly one designated sender process transmits data to exactly one designated receiver process using matching message envelopes.
>
> **Collective Communication:** A coordinated communication pattern involving every process belonging to a specified communicator, categorized into synchronization (barrier), data movement (broadcast, scatter, gather), and global reduction operations.
>
> **One-Sided Communication (Remote Memory Access - RMA):** A communication paradigm introduced in MPI-2 where a process can directly read from (`MPI_Get`) or write to (`MPI_Put`) memory belonging to a remote target process without that remote process participating in the function call.
>
> **Collective Participation Invariant:** The strict semantic rule requiring that every process belonging to a communicator must invoke the same collective function with compatible arguments; omission of the call by even a single process produces a permanent distributed deadlock.
>
> **Logarithmic Scaling ($\mathcal{O}(\log P)$):** The property of tree-structured collective algorithms ensuring that communication latency grows proportionally to $\log_2(P)$ rather than linearly with process count $P$.

---

## 9. Comparison Tables

### Point-to-Point vs Collective Operations

| Feature | Point-to-Point (`MPI_Send` / `Recv`) | Collective Operations (`Bcast`, `Reduce`) |
| :--- | :--- | :--- |
| **Participants** | Exactly 2 processes (Sender & Receiver) | **All** processes in `MPI_Comm` |
| **Message Tags** | **Yes** (Tags distinguish messages) | **No** (Collectives do not use tags) |
| **Non-Blocking Options** | Fully supported (`MPI_Isend`, `Irecv`) | Supported in MPI-3 (`MPI_Ibcast`, etc.) |
| **Complexity Scaling** | $\mathcal{O}(P)$ when chaining | $\mathcal{O}(\log P)$ via hardware trees |
| **Deadlock Vulnerability**| Reversing send/receive orders | Omitting call in a conditional `if` branch |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Diagnosing the Conditional Collective Deadlock
**Question:** Explain what will happen when the following code is executed across 4 processes:

```c
int rank;
MPI_Comm_rank(MPI_COMM_WORLD, &rank);

if (rank != 0) {
    int data = rank * 10;
    MPI_Bcast(&data, 1, MPI_INT, 0, MPI_COMM_WORLD); /* HANGS! */
}
```

**Solution:**
- **The Defect:** The developer placed `MPI_Bcast()` inside a conditional block that excludes Rank 0 (`if (rank != 0)`).
- **The Deadlock:** Under the **Collective Participation Invariant**, *every* process in `MPI_COMM_WORLD`—including the root process (Rank 0)—**must call `MPI_Bcast()`**.
- When Ranks 1, 2, and 3 invoke `MPI_Bcast()`, they wait for Rank 0 to broadcast data.
- Rank 0 skips the `if` block entirely and continues executing sequential code or terminates.
- **Result:** Ranks 1, 2, and 3 remain permanently blocked waiting for Rank 0. The job **deadlocks indefinitely**.
- **The Fix:** Move `MPI_Bcast()` outside the `if` block so that Rank 0 participates as root.

---

### Problem 2: Bandwidth Comparison of Reduction Implementations
**Question:** To sum an 8 MB vector across 64 nodes over a network where $\alpha = 2\,\mu\text{s}$ and bandwidth $B = 10\text{ GB/s}$:
- Implementation 1 uses a sequential Point-to-Point gather to Rank 0 ($63$ sends).
- Implementation 2 uses a Binomial Tree Collective Reduction ($\lceil \log_2(64) \rceil = 6$ steps).
Calculate the network transmission time for both implementations.

**Solution:**
- Payload $m = 8\text{ MB} = 8 \times 10^6\text{ bytes}$.
- Transfer time per step:
$$t_{\text{step}} = \alpha + \frac{m}{B} = 2 \times 10^{-6} + \frac{8 \times 10^6}{10 \times 10^9} = 2\,\mu\text{s} + 800\,\mu\text{s} = 802\,\mu\text{s} = 0.000802\text{ s}$$

- **Implementation 1 (P2P Serial Gather):**
$$T_1 = 63 \times 0.000802\text{ s} \approx \mathbf{0.0505\text{ seconds}} = 50.5\text{ ms}$$

- **Implementation 2 (Tree Collective):**
$$T_2 = 6 \times 0.000802\text{ s} \approx \mathbf{0.00481\text{ seconds}} = 4.81\text{ ms}$$

- **Advantage:** The collective reduction is over **$10.5\times$ faster**!

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why do MPI collective functions not accept a `tag` argument?
**Answer:** In point-to-point communication, tags are required to differentiate between multiple concurrent messages arriving from the same sender. In contrast, collective operations require strict, global coordination across all ranks in the communicator. The MPI standard dictates that collectives execute in a dedicated, serialized sequence within that communicator context; therefore, messages within collectives are matched unambiguously by order of invocation and context ID, making message tags unnecessary.

### Q2: What is the difference between Two-Sided and One-Sided communication?
**Answer:**
- In **Two-Sided Communication** (Point-to-Point and Collectives), data exchange requires active, matching function calls by both parties: the sender must call `MPI_Send` and the receiver must call `MPI_Recv`.
- In **One-Sided Communication** (RMA), only the initiator invokes an MPI call (`MPI_Put` or `MPI_Get`). The remote target process exposes a designated memory window (`MPI_Win`) but does not execute any active send or receive routine; the network interface card (NIC) reads or writes the remote memory directly via RDMA.

### Q3: Why is `MPI_Allreduce` preferred over calling `MPI_Reduce` followed by `MPI_Bcast`?
**Answer:** While calling `MPI_Reduce` to root followed by `MPI_Bcast` produces the same mathematical result (all processes receive the aggregated total), it requires two distinct collective phases across the network. `MPI_Allreduce` combines reduction and broadcast into a single unified communication phase (often using optimized butterfly or ring algorithms), cutting network latency and bandwidth consumption roughly in half.

---

## 12. Quick Revision Sheet

- **Four Paradigms:** Point-to-Point (1-to-1), Collectives (All-in-Comm), One-Sided RMA (Put/Get), Dynamic Ports.
- **Point-to-Point:** Uses tags; requires matching sender and receiver.
- **Collectives:** No tags; every rank in communicator **must** call it; scales logarithmically $\mathcal{O}(\log P)$.
- **Deadlock Rule:** Never put a collective inside a rank-conditional `if` branch that excludes some ranks!
- **Hockney Complexity:** Collectives scale as $\lceil \log_2 P \rceil (\alpha + \beta m)$, beating manual serial rings by orders of magnitude.
- **RMA:** Initiator accesses remote memory window (`MPI_Win`) without remote process participation.

---

## 13. What You Should Be Able to Do

1. **Classify any distributed computational task** into its appropriate MPI communication paradigm.
2. **Contrast Point-to-Point and Collective operations** in terms of participant scope, tags, and scaling complexity.
3. **Identify and correct collective deadlocks** caused by rank-conditional branching.
4. **Calculate communication speedups** achieved by tree collectives over point-to-point chains using the Hockney model.
5. **Implement basic multi-process ring and collective reduction workflows** in compilable C.
