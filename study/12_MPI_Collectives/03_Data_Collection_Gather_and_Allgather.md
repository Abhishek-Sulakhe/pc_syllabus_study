# Module 12: MPI Collective Operations (Lectures 30–31)
# Subtopic 12.03: Data Collection with Gather and Allgather (`MPI_Gather`, `MPI_Allgather`, `MPI_Gatherv`, In-Place Assembly)

---

## 1. Basic Idea

In parallel algorithms, after a large problem domain has been partitioned and computed across worker processes, the distributed results must be collected and reassembled into a single coherent data structure.

The primary collective primitives for data assembly are:

1. **`MPI_Gather` (All-to-One Collection):**
   The exact mathematical inverse of `MPI_Scatter`. Every process in the communicator (including the Root) contributes a local data slice. The **Root process** collects these slices and concatenates them in strict **rank order** ($0, 1, 2, \dots, P-1$) into a contiguous global receive buffer. Non-root processes receive nothing.

2. **`MPI_Allgather` (All-to-All Multi-Collection):**
   A symmetric variant where **every process** in the communicator receives the complete concatenated array of all $P$ slices. It is functionally equivalent to an `MPI_Gather` to root followed immediately by an `MPI_Bcast` to all ranks, but executed in a single, highly optimized communication phase.

```
+-----------------------------------------------------------------------------------------+
|                              The MPI_Gather vs. MPI_Allgather                           |
+-----------------------------------------------------------------------------------------+
| LOCAL CONTRIBUTIONS:                                                                    |
|   Rank 0: [ A_0 | A_1 ]    Rank 1: [ B_0 | B_1 ]                                        |
|   Rank 2: [ C_0 | C_1 ]    Rank 3: [ D_0 | D_1 ]                                        |
|                                                                                         |
| MPI_Gather (Root = 0):                                                                  |
|   Rank 0 Receive Buffer: [ A_0 | A_1 | B_0 | B_1 | C_0 | C_1 | D_0 | D_1 ]               |
|   Ranks 1, 2, 3        : Receive nothing.                                               |
|                                                                                         |
| MPI_Allgather:                                                                          |
|   Rank 0 Receive Buffer: [ A_0 | A_1 | B_0 | B_1 | C_0 | C_1 | D_0 | D_1 ]               |
|   Rank 1 Receive Buffer: [ A_0 | A_1 | B_0 | B_1 | C_0 | C_1 | D_0 | D_1 ]               |
|   Rank 2 Receive Buffer: [ A_0 | A_1 | B_0 | B_1 | C_0 | C_1 | D_0 | D_1 ]               |
|   Rank 3 Receive Buffer: [ A_0 | A_1 | B_0 | B_1 | C_0 | C_1 | D_0 | D_1 ]               |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Reassembly Phase in Divide-and-Conquer Workflows
Almost all distributed numerical algorithms operate in a three-stage pipeline:
1. **Scatter:** Root partitions and distributes initial data (`MPI_Scatter`).
2. **Compute:** Every rank computes on its local private partition in parallel.
3. **Gather:** Root reassembles the updated partitions back into the global result array (`MPI_Gather`).

Without `MPI_Gather`, the Root would have to execute a loop of $P-1$ blocking `MPI_Recv` calls, re-introducing linear latency bottlenecks and complex memory offset calculations.

### 2. Eliminating Two-Phase Gather-Broadcast Latency with `MPI_Allgather`
In many linear algebra algorithms (such as dense Matrix Multiplication, N-Body gravitational simulations, or all-pairs shortest paths), every process needs to know the data owned by *all* other processes to evaluate cross-terms.
- Executing `MPI_Gather` followed by `MPI_Bcast` requires **two distinct network phases**, traversing the interconnect twice.
- `MPI_Allgather` merges both operations into a single ring or butterfly collective, cutting network latency $\alpha$ in half.

### 3. Handling Variable Slice Sizes (`MPI_Gatherv`)
In parallel text processing, sparse matrix operations, or adaptive particle solvers, different worker processes produce varying quantities of output. `MPI_Gatherv` allows the root to assemble unequal segments from each rank using displacement offsets.

---

## 3. How It Works

### The Memory Mechanics of `MPI_Gather`

```
Process Memory Spaces:
  Rank 0 (Root) : sendbuf [ A0, A1 ] ---> recvbuf [ A0, A1 | B0, B1 | C0, C1 | D0, D1 ]
  Rank 1        : sendbuf [ B0, B1 ] ---> recvbuf (Ignored / NULL)
  Rank 2        : sendbuf [ C0, C1 ] ---> recvbuf (Ignored / NULL)
  Rank 3        : sendbuf [ D0, D1 ] ---> recvbuf (Ignored / NULL)
```

- On **Non-Root ranks** (`rank != root`): `recvbuf`, `recvcount`, and `recvtype` are ignored.
- On the **Root rank** (`rank == root`): `recvbuf` must point to allocated memory capable of holding $P \times \text{recvcount}$ elements.

---

### The `recvcount` Semantic Trap in `MPI_Gather`

Just as in `MPI_Scatter`, `MPI_Gather` has an identical, critical parameter trap:

> [!WARNING]
> **The `recvcount` Parameter Rule in `MPI_Gather`:**
> In `MPI_Gather`, **`recvcount` is the number of elements received from EACH individual process**, NOT the total capacity of the assembled array!
> 
> If $4$ processes each contribute $250$ elements to assemble a global array of $1{,}000$ elements, the Root process must specify:
> ```c
> /* CORRECT: recvcount is 250 (per process), NOT 1000! */
> MPI_Gather(local_data, 250, MPI_INT, global_data, 250, MPI_INT, 0, MPI_COMM_WORLD);
> ```
> Specifying `recvcount = 1000` directs MPI to expect 1,000 elements from *each* of the 4 processes, attempting to write $4{,}000$ elements into `global_data` and triggering a catastrophic **Segmentation Fault (`SIGSEGV`)**!

In standard uniform `MPI_Gather`:
$$\text{sendcount} = \text{recvcount} = \frac{N_{\text{total}}}{P}$$

---

## 4. Important Mechanisms and Functions

### 1. The `MPI_Gather` Function Signature

```c
#include <mpi.h>

int MPI_Gather(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
               void *recvbuf, int recvcount, MPI_Datatype recvtype,
               int root, MPI_Comm comm);
```

- **`sendbuf`:** Pointer to local input data (valid on all ranks).
- **`sendcount`:** Number of elements in `sendbuf`.
- **`sendtype`:** Datatype of elements in `sendbuf`.
- **`recvbuf`:** Pointer to global destination array on Root (ignored on non-root).
- **`recvcount`:** Number of elements received **from each process**.
- **`recvtype`:** Datatype of elements in `recvbuf`.
- **`root`:** Rank of the collecting process.
- **`comm`:** Communicator handle.

---

### 2. The `MPI_Allgather` Function Signature

```c
int MPI_Allgather(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
                  void *recvbuf, int recvcount, MPI_Datatype recvtype,
                  MPI_Comm comm);
```

- Notice that **there is no `root` argument**! Every rank is simultaneously a root and a receiver.
- `recvbuf` must be allocated on **every process** with capacity $P \times \text{recvcount}$.
- Upon completion, `recvbuf` on every rank contains identical, fully assembled global data.

---

### 3. In-Place Assembly on Root: `MPI_IN_PLACE`

In `MPI_Gather`, the Root process contributes its own data from `sendbuf` into the first block of `recvbuf`. If the Root has already placed its output into the beginning of `recvbuf`, passing a separate `sendbuf` wastes memory.

On the **Root process only**, passing `sendbuf = MPI_IN_PLACE` instructs MPI that the root's contribution is already positioned in `recvbuf`:

```c
if (rank == root) {
    /* Root passes MPI_IN_PLACE as sendbuf; its data is already in recvbuf */
    MPI_Gather(MPI_IN_PLACE, 0, MPI_DATATYPE_NULL,
               global_array, count, MPI_INT, root, comm);
} else {
    MPI_Gather(local_array, count, MPI_INT,
               NULL, 0, MPI_DATATYPE_NULL, root, comm);
}
```

In `MPI_Allgather`, **every process** can specify `MPI_IN_PLACE` as its `sendbuf`, provided each rank has placed its local contribution at its corresponding offset inside `recvbuf`!

---

### 4. Vector Variants for Variable Slices: `MPI_Gatherv` & `MPI_Allgatherv`

When local slice lengths vary across processes:

```c
int MPI_Gatherv(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
                void *recvbuf, const int recvcounts[], const int displs[],
                MPI_Datatype recvtype, int root, MPI_Comm comm);

int MPI_Allgatherv(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
                   void *recvbuf, const int recvcounts[], const int displs[],
                   MPI_Datatype recvtype, MPI_Comm comm);
```

- **`recvcounts[r]`:** Array of size $P$ specifying elements to receive from Rank $r$.
- **`displs[r]`:** Memory offset displacement into `recvbuf` for Rank $r$'s data.

---

## 5. Compilable C Code Examples

### Example 1: Full Scatter-Compute-Gather Pipeline with `MPI_Gather`
This complete program demonstrates the standard parallel scientific computing pipeline:
1. Rank 0 initializes a 16-element array and distributes 4-element chunks via `MPI_Scatter`.
2. All ranks square their local elements in parallel.
3. Rank 0 gathers the computed chunks back into a global result array via `MPI_Gather`.
4. Rank 0 validates the final global result.

```c
/*
 * File: mpi_scatter_gather_pipeline.c
 * Compilation:
 *   mpicc -O2 mpi_scatter_gather_pipeline.c -o mpi_scatter_gather_pipeline
 *
 * Execution:
 *   mpirun -np 4 ./mpi_scatter_gather_pipeline
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

#define TOTAL_ELEMENTS 16

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (TOTAL_ELEMENTS % size != 0) {
        if (rank == 0) {
            fprintf(stderr, "Error: Elements (%d) must divide evenly by size (%d)\n",
                    TOTAL_ELEMENTS, size);
        }
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }

    int chunk_size = TOTAL_ELEMENTS / size;
    int* original_array = NULL;
    int* gathered_result = NULL;

    /* Allocate and initialize global arrays ONLY on Root (Rank 0) */
    if (rank == 0) {
        original_array  = (int*)malloc(TOTAL_ELEMENTS * sizeof(int));
        gathered_result = (int*)malloc(TOTAL_ELEMENTS * sizeof(int));

        printf("=================================================================\n");
        printf("  MPI SCATTER-COMPUTE-GATHER PARALLEL PIPELINE                   \n");
        printf("  Total Elements: %d | Processes: %d | Chunk/Rank: %d            \n",
               TOTAL_ELEMENTS, size, chunk_size);
        printf("=================================================================\n\n");

        printf("[Root Rank 0] Original Global Array:\n[ ");
        for (int i = 0; i < TOTAL_ELEMENTS; ++i) {
            original_array[i] = i + 1; /* 1, 2, 3, ... 16 */
            printf("%d ", original_array[i]);
        }
        printf("]\n\n");
    }

    /* Allocate local chunk on EVERY rank */
    int* local_chunk = (int*)malloc(chunk_size * sizeof(int));

    /* -------------------------------------------------------------
     * STAGE 1: SCATTER - Distribute distinct slices to all ranks
     * ------------------------------------------------------------- */
    MPI_Scatter(original_array, chunk_size, MPI_INT,
                local_chunk, chunk_size, MPI_INT,
                0 /* root */, MPI_COMM_WORLD);

    /* -------------------------------------------------------------
     * STAGE 2: PARALLEL COMPUTE - Local transformations on private RAM
     * ------------------------------------------------------------- */
    for (int i = 0; i < chunk_size; ++i) {
        local_chunk[i] = local_chunk[i] * 10; /* Multiply each element by 10 */
    }

    printf("  [Rank %d] Computed Local Chunk: [ ", rank);
    for (int i = 0; i < chunk_size; ++i) {
        printf("%d ", local_chunk[i]);
    }
    printf("]\n");

    /* -------------------------------------------------------------
     * STAGE 3: GATHER - Reassemble all chunks on Root
     * Notice: recvcount is chunk_size (4), NOT TOTAL_ELEMENTS (16)!
     * ------------------------------------------------------------- */
    MPI_Gather(local_chunk, chunk_size, MPI_INT,
               gathered_result, chunk_size, MPI_INT,
               0 /* root */, MPI_COMM_WORLD);

    /* -------------------------------------------------------------
     * STAGE 4: VERIFICATION ON ROOT
     * ------------------------------------------------------------- */
    if (rank == 0) {
        printf("\n[Root Rank 0] Reassembled Gathered Result:\n[ ");
        int success = 1;
        for (int i = 0; i < TOTAL_ELEMENTS; ++i) {
            printf("%d ", gathered_result[i]);
            if (gathered_result[i] != (i + 1) * 10) {
                success = 0;
            }
        }
        printf("]\n\n");

        if (success) {
            printf(">>> VERIFICATION SUCCESS: All elements correctly scattered, computed, and gathered! <<<\n");
        } else {
            printf(">>> VERIFICATION FAILED: Data corruption detected! <<<\n");
        }

        free(original_array);
        free(gathered_result);
    }

    free(local_chunk);
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
=================================================================
  MPI SCATTER-COMPUTE-GATHER PARALLEL PIPELINE                   
  Total Elements: 16 | Processes: 4 | Chunk/Rank: 4            
=================================================================

[Root Rank 0] Original Global Array:
[ 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 ]

  [Rank 0] Computed Local Chunk: [ 10 20 30 40 ]
  [Rank 1] Computed Local Chunk: [ 50 60 70 80 ]
  [Rank 2] Computed Local Chunk: [ 90 100 110 120 ]
  [Rank 3] Computed Local Chunk: [ 130 140 150 160 ]

[Root Rank 0] Reassembled Gathered Result:
[ 10 20 30 40 50 60 70 80 90 100 110 120 130 140 150 160 ]

>>> VERIFICATION SUCCESS: All elements correctly scattered, computed, and gathered! <<<
```

---

### Example 2: Symmetric All-to-All Knowledge Sharing with `MPI_Allgather`
This program demonstrates `MPI_Allgather` where each of 4 ranks produces an identifier and status flag, and every process receives the global directory of all active ranks.

```c
/*
 * File: mpi_allgather_demo.c
 * Compilation:
 *   mpicc -O2 mpi_allgather_demo.c -o mpi_allgather_demo
 *
 * Execution:
 *   mpirun -np 4 ./mpi_allgather_demo
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* Each rank contributes 2 integers: [rank, rank * 100] */
    int local_data[2] = {rank, rank * 100};

    /* Allocate global receive buffer on EVERY rank: capacity = size * 2 integers */
    int* global_registry = (int*)malloc(size * 2 * sizeof(int));

    /* Symmetric Allgather: Every rank receives the complete concatenated array */
    MPI_Allgather(local_data, 2, MPI_INT,
                  global_registry, 2, MPI_INT,
                  MPI_COMM_WORLD);

    /* Verify that every rank has the complete assembled view */
    printf("[Rank %d] Complete Assembled Global Registry: [ ", rank);
    for (int i = 0; i < size * 2; ++i) {
        printf("%d ", global_registry[i]);
    }
    printf("]\n");

    free(global_registry);
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[Rank 0] Complete Assembled Global Registry: [ 0 0 1 100 2 200 3 300 ]
[Rank 1] Complete Assembled Global Registry: [ 0 0 1 100 2 200 3 300 ]
[Rank 2] Complete Assembled Global Registry: [ 0 0 1 100 2 200 3 300 ]
[Rank 3] Complete Assembled Global Registry: [ 0 0 1 100 2 200 3 300 ]
```

---

## 6. Mathematical Formulations & Ring Communication Algorithms

### Algorithmic Complexity of `MPI_Allgather`

Modern MPI implementations utilize the **Ring Allgather Algorithm** (also known as the Bruck algorithm) for large arrays:

```
Processes 0, 1, 2, 3 arranged in a logical ring.
At each step k in [1 .. P-1]:
  Rank r sends its current block to Rank (r + 1) % P
  Rank r receives a new block from Rank (r - 1 + P) % P
```

Let:
- $P$ = Number of processes.
- $S$ = Elements contributed per process.
- $m = S \cdot \text{sizeof(type)}$ = Payload size per rank in bytes.

#### Total Execution Steps:
The ring requires exactly $P - 1$ shift steps:

$$T_{\text{allgather}}(P, m) = (P - 1) \cdot \alpha + \left( \frac{P - 1}{P} \right) \cdot (P \cdot m) \cdot \beta = (P - 1) \cdot \alpha + (P - 1) \cdot m \cdot \beta$$

For large payloads where bandwidth dominates ($\beta m \gg \alpha$):

$$T_{\text{allgather}} \approx (P - 1) \cdot m \cdot \beta = \left( \frac{P - 1}{P} \right) \cdot \frac{M_{\text{total}}}{B} \approx \frac{M_{\text{total}}}{B}$$

Where $M_{\text{total}} = P \cdot m$ is the total assembled size. 

*Crucial Scaling Insight:* Every physical network link in the cluster is used simultaneously in parallel during every step, achieving **near-$100\%$ link bandwidth utilization**!

---

## 7. Advantages, Disadvantages & Limitations

### Trade-off Analysis

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Gather`              | - Inverse of Scatter; clean reassembly  | - Centralizes all data onto Root node;  |
|                           | - Preserves strict rank-order indexing  |   Root memory can overflow if N is huge |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Allgather`           | - Gives all ranks full global visibility| - Memory scales as O(P * S) on EVERY node|
|                           | - 2x faster than Gather + Broadcast     | - High network volume for large arrays  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_IN_PLACE`            | - Eliminates duplicate buffer copies    | - Permitted on Root in Gather;          |
|                           |                                         |   permitted on all ranks in Allgather   |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Data Gather (`MPI_Gather`):** An all-to-one collective communication routine where every process in a communicator transmits its local data buffer to a single root process, which concatenates the segments in ascending process rank order.
>
> **`recvcount` in `MPI_Gather`:** The number of data elements received **from each individual process**, mathematically equal to $\frac{\text{Total Assembled Elements}}{P}$, and **not** the total length of the assembled receive array.
>
> **Allgather (`MPI_Allgather`):** A symmetric collective communication routine where every process contributes a local data slice and all processes in the communicator receive the complete concatenated global array.
>
> **Vector Gather (`MPI_Gatherv`):** An extended gather operation allowing the root process to assemble unequal segments from different ranks using arrays of counts (`recvcounts[]`) and memory displacements (`displs[]`).

---

## 9. Comparison Tables

### `MPI_Scatter` vs `MPI_Gather` vs `MPI_Allgather`

| Feature | `MPI_Scatter` | `MPI_Gather` | `MPI_Allgather` |
| :--- | :--- | :--- | :--- |
| **Communication Direction**| One-to-All | All-to-One | All-to-All |
| **Root Argument?** | **Yes** (Source rank) | **Yes** (Destination rank) | **No** (Symmetric) |
| **Data In Flow** | 1 slice per rank | 1 slice per rank | 1 slice per rank |
| **Data Out Flow** | Local slice on all | Global array on **Root only** | Global array on **All ranks** |
| **`recvcount` Meaning** | Size received by caller | Size received **per rank** | Size received **per rank** |
| **Total Memory at Finish** | $N / P$ on each rank | $N$ on Root only | $N$ on **every** rank |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Calculating Memory Allocations for `MPI_Allgather`
**Question:** A cluster with $P = 64$ processes executes `MPI_Allgather`. Each process generates an array of $10{,}000$ double-precision floats ($8\text{ bytes}$ each).
1. What must be the allocated capacity of `recvbuf` on each process?
2. What is the total memory consumed by `recvbuf` across the entire 64-node cluster?

**Solution:**
1. **Per-Process Capacity:**
   - In `MPI_Allgather`, every process receives elements from all $P$ ranks.
   - Elements received per process = $P \times \text{sendcount} = 64 \times 10{,}000 = \mathbf{640{,}000\text{ doubles}}$.
   - Byte footprint per node = $640{,}000 \times 8\text{ bytes} = 5{,}120{,}000\text{ bytes} \approx \mathbf{5.12\text{ MB}}$.
2. **Total Cluster Memory:**
   - Total memory across all 64 nodes = $64 \times 5.12\text{ MB} \approx \mathbf{327.68\text{ MB}}$.

---

### Problem 2: Detecting the Asymmetric `Gather` Deadlock
**Question:** Explain what happens when 4 processes execute:

```c
int rank;
MPI_Comm_rank(MPI_COMM_WORLD, &rank);
int local_val = rank;
int global_vals[4];

if (rank == 0) {
    MPI_Gather(&local_val, 1, MPI_INT, global_vals, 1, MPI_INT, 0, MPI_COMM_WORLD);
}
```

**Solution:**
- **The Defect:** `MPI_Gather()` is enclosed inside an `if (rank == 0)` block.
- Under the **Universal Participation Invariant**, *every* process in `MPI_COMM_WORLD` must invoke `MPI_Gather()`.
- Rank 0 executes `MPI_Gather()` and blocks waiting for Ranks 1, 2, and 3 to transmit their local values.
- Ranks 1, 2, and 3 skip the `if` block entirely and continue executing other code or exit.
- **Result:** Rank 0 hangs permanently in an unrecoverable **Distributed Deadlock**.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why is `MPI_Allgather` functionally superior to calling `MPI_Gather` followed by `MPI_Bcast`?
**Answer:** While calling `MPI_Gather` to root followed by `MPI_Bcast` to all ranks yields the exact same data result, it requires **two separate communication phases**. The root node acts as a temporary bottleneck where all data must first be collected and then re-transmitted. In contrast, `MPI_Allgather` uses a symmetric ring or recursive-doubling algorithm where all nodes transmit and receive simultaneously in a single phase, cutting network latency in half and eliminating the root node memory and bandwidth bottleneck.

### Q2: What is the purpose of the `displs[]` array in `MPI_Gatherv`?
**Answer:** In `MPI_Gatherv`, the segments collected from different processes may have unequal lengths. The `displs[]` array specifies the exact memory offset (in element indices from the start of `recvbuf`) where the data from each specific rank must be stored. This allows the root process to assemble non-uniform segments into contiguous memory or place them into designated memory slots without memory fragmentation.

### Q3: Can a process pass `recvbuf = NULL` in `MPI_Gather`?
**Answer:** Yes, but **only if the process is NOT the Root process** (`rank != root`). On non-root processes, the `recvbuf`, `recvcount`, and `recvtype` arguments are completely ignored by the MPI library; passing `NULL` is standard practice. On the Root process (`rank == root`), passing `recvbuf = NULL` (without using `MPI_IN_PLACE`) will trigger a fatal segmentation fault when MPI attempts to write incoming data.

---

## 12. Quick Revision Sheet

- **`MPI_Gather`:** All-to-One collection. Root concatenates slices in ascending rank order.
- **`MPI_Allgather`:** All-to-All collection. Every rank receives the full concatenated array. (No `root` parameter).
- **The `recvcount` Rule:** `recvcount` is elements received **from EACH process**, NOT total elements!
- **`MPI_IN_PLACE`:** Supported on Root in `MPI_Gather` (data already in `recvbuf`); supported on all ranks in `MPI_Allgather`.
- **Uneven Slices:** Use `MPI_Gatherv` or `MPI_Allgatherv` with `recvcounts[]` and `displs[]`.
- **Invariants:** Every process in communicator must participate. No tags are used.

---

## 13. What You Should Be Able to Do

1. **Implement complete Scatter-Compute-Gather pipelines** in compilable C.
2. **Utilize `MPI_Allgather`** for all-to-all global knowledge sharing without root bottlenecks.
3. **Debug segmentation faults** caused by confusing per-rank `recvcount` with global array capacity in `MPI_Gather`.
4. **Calculate memory footprints** across cluster nodes for large-scale allgather operations.
5. **Implement vector gathering of variable-sized segments** using `MPI_Gatherv`.
