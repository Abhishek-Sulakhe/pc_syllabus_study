# Module 11: MPI Point-to-Point Communication & Dynamic Processes (Lectures 26–29)
# Subtopic 11.01: Blocking Point-to-Point Communication (`MPI_Send`, `MPI_Recv`, `MPI_Status`, `MPI_Sendrecv`, Deadlocks)

---

## 1. Basic Idea

Point-to-point communication is the foundational mechanism for data transmission between pairs of processes in MPI. In this model, data is transmitted directly from exactly **one sender process** to exactly **one receiver process**.

Every point-to-point message in MPI is enclosed within a structured **Message Envelope** consisting of six essential parameters:
1. **Buffer Address (`buf`):** Pointer to contiguous memory containing data to send or allocated to receive.
2. **Element Count (`count`):** Number of elements (not number of bytes!).
3. **Datatype (`datatype`):** The MPI representation of the C data type (e.g., `MPI_DOUBLE`, `MPI_INT`).
4. **Partner Rank (`dest` or `source`):** The integer ID of the target recipient or expected sender.
5. **Message Tag (`tag`):** An integer identifier used to categorize and distinguish multiple concurrent message streams.
6. **Communicator (`comm`):** The communication context (e.g., `MPI_COMM_WORLD`).

```
+-----------------------------------------------------------------------------------------+
|                                The MPI Message Envelope                                 |
+-----------------------------------------------------------------------------------------+
| [ Payload: buf, count, datatype ]  +  [ Routing: dest/source, tag, comm ]               |
|                                                                                         |
| SENDER:   MPI_Send(&array, 100, MPI_DOUBLE,  1 /*dest*/,   42 /*tag*/, MPI_COMM_WORLD)  |
|                                                     |                                   |
|                                                     v  (Must match on receiver)         |
| RECEIVER: MPI_Recv(&array, 100, MPI_DOUBLE,  0 /*src*/,    42 /*tag*/, MPI_COMM_WORLD)  |
+-----------------------------------------------------------------------------------------+
```

A **Blocking Send (`MPI_Send`)** does not return until the sender's memory buffer is **safe to reuse or overwrite**. 
A **Blocking Receive (`MPI_Recv`)** does not return until the incoming data has arrived over the network and been fully written into the user's receive buffer.

---

## 2. Why We Need It

### 1. Directed Neighbor Communication (Halo Exchanges)
In grid-based scientific simulations (e.g., fluid dynamics, climate models), each cluster node computes a physical domain tile. Each node must periodically send its boundary cells (ghost cells or "halo") to its immediate neighboring tiles. Point-to-point communication allows Rank $r$ to send directly to Rank $r+1$ and Rank $r-1$ without involving or interrupting the remaining 10,000 ranks in the supercomputer.

### 2. Message Disambiguation via Tags
Suppose Rank 0 sends two distinct arrays to Rank 1:
- Array $A$: Velocity field vectors ($1{,}000$ doubles).
- Array $B$: Temperature field scalars ($1{,}000$ doubles).
Because both arrays have the identical count and datatype, Rank 1 needs a mechanism to distinguish them. By assigning `tag = 100` to velocities and `tag = 200` to temperatures, MPI guarantees that incoming packets are matched strictly to their designated reception buffers, preventing catastrophic data mix-ups.

### 3. The Deadlock Hazard in Blocking Calls
Because `MPI_Send` and `MPI_Recv` block the caller, naive code that arranges sends and receives improperly can cause **distributed deadlocks**:
- If Rank 0 calls `MPI_Recv` from Rank 1, and Rank 1 calls `MPI_Recv` from Rank 0, both processes freeze permanently, waiting for the other to send.
- If both processes call `MPI_Send` with large messages that exceed the operating system's internal network buffer capacity, both block waiting for network buffers to clear, freezing the entire cluster.
Mastering blocking mechanics, buffer thresholds, and `MPI_Sendrecv` is vital to writing safe MPI programs.

---

## 3. How It Works

### The Envelope Matching Rules

For a message to be transferred successfully, the parameters posted by the sender and receiver must satisfy strict matching rules:

```
Sender Parameters               Receiver Parameters             Match Condition
-----------------               -------------------             ---------------
Communicator: comm       <===>  Communicator: comm              MUST BE IDENTICAL
Datatype:     type       <===>  Datatype:     type              MUST BE COMPATIBLE
Destination:  dest (1)   <===>  Receiver Rank: 1                MUST MATCH (or MPI_ANY_SOURCE)
Sender Rank:  0          <===>  Source:       source (0)        MUST MATCH (or MPI_ANY_SOURCE)
Tag:          tag (42)   <===>  Tag:          tag (42)          MUST MATCH (or MPI_ANY_TAG)
Send Count:   sendcount  <===>  Recv Count:   recvcount         recvcount >= sendcount
```

> [!IMPORTANT]
> The receive buffer capacity (`recvcount`) must be **greater than or equal to** the transmitted count (`sendcount`). If a sender transmits 100 elements and the receiver posts `recvcount = 50`, an `MPI_ERR_TRUNCATE` error occurs, immediately crashing the receiver.

---

### Wildcards: `MPI_ANY_SOURCE` and `MPI_ANY_TAG`

A receiver process can accept messages opportunistically without knowing in advance which rank will send or what tag will be attached, using MPI wildcards:
- **`source = MPI_ANY_SOURCE`:** Accepts an incoming message from **any** process in the communicator (first-come, first-served worker harvesting).
- **`tag = MPI_ANY_TAG`:** Accepts a message carrying **any** tag integer.

When wildcards are used, the receiver inspects the `MPI_Status` object to discover who actually sent the message and what tag was attached.

---

### Low-Level Network Transport: Eager vs. Rendezvous Protocol

Why does `MPI_Send()` sometimes return immediately, and other times block indefinitely? The answer lies in the two transport protocols used by modern MPI libraries:

```
+-----------------------------------------------------------------------------------------+
|                  EAGER PROTOCOL (Small Messages: < 16 KB - 64 KB)                       |
+-----------------------------------------------------------------------------------------+
| Rank 0: MPI_Send() ---> Copies data directly into receiver's temporary MPI system buffer|
|                         ===> SENDER RETURNS IMMEDIATELY! (Does not wait for MPI_Recv)   |
|                                                                                         |
| Rank 1: Later calls MPI_Recv() ---> Copies from internal system buffer into user array.  |
+-----------------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------------+
|                  RENDEZVOUS PROTOCOL (Large Messages: > 64 KB)                          |
+-----------------------------------------------------------------------------------------+
| Rank 0: MPI_Send() ---> Sends small "Request-to-Send" (RTS) envelope handshake.         |
|                         [ SENDER BLOCKS in kernel waiting for Clear-to-Send (CTS) ]     |
|                                                                                         |
| Rank 1: Calls MPI_Recv() ---> Allocates user buffer; sends "Clear-to-Send" (CTS).       |
|                                                                                         |
| Direct RDMA Transfer : Data moves directly from Rank 0 RAM to Rank 1 RAM via NIC.       |
| Rank 0 & Rank 1 Finish : Handshake ACK clears; BOTH calls unblock and return.           |
+-----------------------------------------------------------------------------------------+
```

```
[ THE "LATENT DEADLOCK" TRAP ]
A naive Send-Send code:
  Rank 0: MPI_Send(to 1) -> MPI_Recv(from 1)
  Rank 1: MPI_Send(to 0) -> MPI_Recv(from 0)

- If tested with 10 integers (40 bytes), it uses the Eager protocol:
  Both sends copy into system buffers and return immediately. The code appears to WORK!
- When deployed in production with 1,000,000 doubles (8 MB), it switches to Rendezvous:
  Both sends block waiting for a handshake from MPI_Recv that never executes!
  ===> THE CLUSTER PERMANENTLY DEADLOCKS IN PRODUCTION!
```

---

## 4. Important Mechanisms and Functions

### 1. `MPI_Send` and `MPI_Recv` Syntax

```c
#include <mpi.h>

int MPI_Send(const void *buf, int count, MPI_Datatype datatype,
             int dest, int tag, MPI_Comm comm);

int MPI_Recv(void *buf, int count, MPI_Datatype datatype,
             int source, int tag, MPI_Comm comm,
             MPI_Status *status);
```

- **`buf`:** Memory buffer address.
- **`count`:** Number of elements of type `datatype`.
- **`datatype`:** Standard MPI datatypes:
  - C signed ints: `MPI_INT`, `MPI_SHORT`, `MPI_LONG`, `MPI_LONG_LONG`.
  - C unsigned ints: `MPI_UNSIGNED`, `MPI_UNSIGNED_LONG`.
  - Floating point: `MPI_FLOAT`, `MPI_DOUBLE`, `MPI_LONG_DOUBLE`.
  - Characters / Bytes: `MPI_CHAR`, `MPI_BYTE`.
- **`dest` / `source`:** Rank number ($0 \le r < \text{size}$) or wildcard `MPI_ANY_SOURCE`.
- **`tag`:** Non-negative integer ($0 \le \text{tag} \le \text{MPI\_TAG\_UB} \ge 32767$) or wildcard `MPI_ANY_TAG`.
- **`status`:** Pointer to `MPI_Status` structure (or pass `MPI_STATUS_IGNORE`).

---

### 2. Inspecting `MPI_Status` and `MPI_Get_count`

When receiving with wildcards, inspect the status structure:

```c
MPI_Status status;
MPI_Recv(buffer, 100, MPI_DOUBLE, MPI_ANY_SOURCE, MPI_ANY_TAG, MPI_COMM_WORLD, &status);

/* Discover true sender and tag */
int actual_sender = status.MPI_SOURCE;
int actual_tag    = status.MPI_TAG;

/* Discover actual number of elements received */
int actual_count = 0;
MPI_Get_count(&status, MPI_DOUBLE, &actual_count);

printf("Received %d doubles from Rank %d with Tag %d\n",
       actual_count, actual_sender, actual_tag);
```

---

### 3. The Deadlock-Free Primitive: `MPI_Sendrecv`

To simultaneously send data to one process and receive data from another without deadlock:

```c
int MPI_Sendrecv(const void *sendbuf, int sendcount, MPI_Datatype sendtype,
                 int dest, int sendtag,
                 void *recvbuf, int recvcount, MPI_Datatype recvtype,
                 int source, int recvtag,
                 MPI_Comm comm, MPI_Status *status);
```

- Atomically executes both a send and a receive.
- Under the hood, the MPI runtime schedules the network operations to prevent circular wait deadlocks.
- Extremely useful in periodic shift topologies: send to right neighbor, receive from left neighbor.
- If sending to a non-existent neighbor at boundary, pass `MPI_PROC_NULL` as destination or source (treated as a successful no-op).

---

## 5. Compilable C Code Examples

### Example 1: High-Precision Ping-Pong Latency & Bandwidth Benchmark
This complete program executes a ping-pong exchange between Rank 0 and Rank 1 across varying message sizes, calculating network half-trip latency and effective bandwidth.

```c
/*
 * File: mpi_pingpong_bench.c
 * Compilation:
 *   mpicc -O2 mpi_pingpong_bench.c -o mpi_pingpong_bench
 *
 * Execution:
 *   mpirun -np 2 ./mpi_pingpong_bench
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NUM_WARMUP 10
#define NUM_REPEATS 100
#define MAX_PAYLOAD_DOUBLES (1024 * 1024) /* 1 Million Doubles = 8 MB */

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (size != 2) {
        if (rank == 0) {
            fprintf(stderr, "Error: Ping-pong benchmark requires exactly 2 processes. (Launched %d)\n", size);
        }
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }

    double* buffer = (double*)malloc(MAX_PAYLOAD_DOUBLES * sizeof(double));
    if (!buffer) {
        perror("malloc failed");
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }
    memset(buffer, 0, MAX_PAYLOAD_DOUBLES * sizeof(double));

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  MPI BLOCKING POINT-TO-POINT PING-PONG BENCHMARK                \n");
        printf("  Testing across message sizes from 8 Bytes to 8 Megabytes       \n");
        printf("=================================================================\n");
        printf(" %-12s | %-12s | %-16s | %-14s\n",
               "Elements", "Size (Bytes)", "Avg Latency (us)", "Bandwidth (MB/s)");
        printf("-----------------------------------------------------------------\n");
    }

    /* Test sizes: 1, 64, 1024, 65536, 1048576 doubles */
    long test_counts[] = {1, 64, 512, 4096, 32768, 262144, 1048576};
    int num_tests = sizeof(test_counts) / sizeof(test_counts[0]);

    for (int t = 0; t < num_tests; ++t) {
        long count = test_counts[t];
        long bytes = count * sizeof(double);

        /* Warmup runs */
        for (int w = 0; w < NUM_WARMUP; ++w) {
            if (rank == 0) {
                MPI_Send(buffer, count, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD);
                MPI_Recv(buffer, count, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            } else {
                MPI_Recv(buffer, count, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
                MPI_Send(buffer, count, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
            }
        }

        MPI_Barrier(MPI_COMM_WORLD);
        double t_start = MPI_Wtime();

        /* Timed benchmark loop */
        for (int r = 0; r < NUM_REPEATS; ++r) {
            if (rank == 0) {
                MPI_Send(buffer, count, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD);
                MPI_Recv(buffer, count, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            } else {
                MPI_Recv(buffer, count, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
                MPI_Send(buffer, count, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
            }
        }

        double t_end = MPI_Wtime();

        if (rank == 0) {
            double total_time = t_end - t_start;
            /* One round trip = 2 one-way transfers */
            double one_way_latency_sec = total_time / (2.0 * NUM_REPEATS);
            double one_way_latency_us  = one_way_latency_sec * 1e6;
            double bandwidth_mbytes_sec = ((double)bytes / (1024.0 * 1024.0)) / one_way_latency_sec;

            printf(" %-12ld | %-12ld | %-16.3f | %-14.2f\n",
                   count, bytes, one_way_latency_us, bandwidth_mbytes_sec);
        }
    }

    if (rank == 0) {
        printf("=================================================================\n");
    }

    free(buffer);
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running locally on 2 processes) ---
=================================================================
  MPI BLOCKING POINT-TO-POINT PING-PONG BENCHMARK                
  Testing across message sizes from 8 Bytes to 8 Megabytes       
=================================================================
 Elements     | Size (Bytes) | Avg Latency (us) | Bandwidth (MB/s)
-----------------------------------------------------------------
 1            | 8            | 0.412            | 18.52         
 64           | 512          | 0.435            | 1121.78       
 512          | 4096         | 0.680            | 5735.29       
 4096         | 32768        | 2.140            | 14616.82      
 32768        | 262144       | 12.450           | 20080.32      
 262144       | 2097152      | 94.200           | 21231.42      
 1048576      | 8388608      | 374.800          | 21344.71      
=================================================================
```

---

### Example 2: Deadlock-Free Bidirectional Ring Shift using `MPI_Sendrecv`
This program demonstrates circular halo exchange across $P$ ranks: each process passes its boundary value to its right neighbor (`(rank+1)%size`) and receives from its left neighbor (`(rank-1+size)%size`) using `MPI_Sendrecv()`, completely eliminating circular wait deadlocks.

```c
/*
 * File: mpi_sendrecv_ring.c
 * Compilation:
 *   mpicc -O2 mpi_sendrecv_ring.c -o mpi_sendrecv_ring
 *
 * Execution:
 *   mpirun -np 4 ./mpi_sendrecv_ring
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int send_val = (rank + 1) * 10; /* Rank 0 has 10, Rank 1 has 20, etc. */
    int recv_val = -1;

    int right_neighbor = (rank + 1) % size;
    int left_neighbor  = (rank - 1 + size) % size;

    printf("[Rank %d] Initial Value = %d | Will send to Right (%d), receive from Left (%d)\n",
           rank, send_val, right_neighbor, left_neighbor);

    /* Atomic, Deadlock-Free Bidirectional Transfer */
    MPI_Sendrecv(
        &send_val, 1, MPI_INT, right_neighbor, 0, /* Send parameters */
        &recv_val, 1, MPI_INT, left_neighbor,  0, /* Recv parameters */
        MPI_COMM_WORLD, MPI_STATUS_IGNORE
    );

    printf("  >>> [Rank %d] Received Value = %d from Left Neighbor (%d) <<<\n",
           rank, recv_val, left_neighbor);

    /* Verify consistency */
    int expected_recv = (left_neighbor + 1) * 10;
    if (recv_val == expected_recv) {
        printf("  [Rank %d] Verification SUCCESS!\n", rank);
    } else {
        printf("  [Rank %d] ERROR: Expected %d, got %d\n", rank, expected_recv, recv_val);
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[Rank 0] Initial Value = 10 | Will send to Right (1), receive from Left (3)
[Rank 1] Initial Value = 20 | Will send to Right (2), receive from Left (0)
[Rank 2] Initial Value = 30 | Will send to Right (3), receive from Left (1)
[Rank 3] Initial Value = 40 | Will send to Right (0), receive from Left (2)
  >>> [Rank 0] Received Value = 40 from Left Neighbor (3) <<<
  [Rank 0] Verification SUCCESS!
  >>> [Rank 1] Received Value = 10 from Left Neighbor (0) <<<
  [Rank 1] Verification SUCCESS!
  >>> [Rank 2] Received Value = 20 from Left Neighbor (1) <<<
  [Rank 2] Verification SUCCESS!
  >>> [Rank 3] Received Value = 30 from Left Neighbor (2) <<<
  [Rank 3] Verification SUCCESS!
```

---

## 6. Mathematical Formulations & Deadlock Analysis

### The Deadlock Invariant in Cyclic Dependencies

Consider $P$ processes executing cyclic communication where each rank sends to $(r+1)\%P$ and receives from $(r-1+P)\%P$.

Let $S_r$ denote `MPI_Send` and $R_r$ denote `MPI_Recv`.

#### The Deadlock Ordering:
If all processes execute:
$$\forall r \in [0, P-1]: \quad \text{Call } S_r \to \text{Call } R_r$$

Under the Rendezvous protocol (large messages $m > M_{\text{eager}}$):
- To complete $S_r$, the receiver $(r+1)\%P$ must post $R_{(r+1)\%P}$.
- However, process $(r+1)\%P$ is currently blocked inside its own send call $S_{(r+1)\%P}$.
- Formally, a directed wait graph $G = (V, E)$ is formed where vertices $V = \{P_0, P_1, \dots, P_{P-1}\}$:

$$E = \{ (P_r, P_{(r+1)\%P}) \mid r \in [0, P-1] \}$$

Because $E$ forms a closed directed Hamiltonian cycle:

$$P_0 \to P_1 \to P_2 \to \dots \to P_{P-1} \to P_0$$

The system satisfies the Coffman **Circular Wait Condition**. Because all processes are blocked waiting for an event that only another blocked process can cause, the system is in an **unbreakable distributed deadlock**.

#### Solutions to Break the Cycle:
1. **Even/Odd Alternation:**
   - Even ranks execute `Send` then `Recv`.
   - Odd ranks execute `Recv` then `Send`.
   - Removes cycle: no two processes block waiting on each other simultaneously.
2. **`MPI_Sendrecv()`:** Internal MPI runtime handles scheduling.
3. **Non-Blocking Communication (`MPI_Isend`, `MPI_Irecv`):** Decouples initiation from completion.

---

## 7. Advantages, Disadvantages & Limitations

### Trade-off Analysis

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Send` / `MPI_Recv`   | - Strict synchronization semantics      | - Highly prone to deadlocks if ordering |
|                           | - Zero dynamic memory management needed |   is asymmetric or unbuffered           |
|                           | - Simple deterministic control flow     | - Cannot overlap compute with network   |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_ANY_SOURCE`          | - Dynamic work distribution             | - Non-deterministic message arrival     |
| (Wildcard)                | - Adapts to variable worker speeds      | - Must inspect MPI_Status for sender ID |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Sendrecv`            | - 100% immune to circular deadlocks     | - Limited to paired exchanges; cannot   |
|                           | - Highly optimized by MPI vendors       |   handle arbitrary dynamic graphs       |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Blocking Point-to-Point Communication:** A communication call (`MPI_Send` / `MPI_Recv`) that does not return control to the calling program until the operation has completed—meaning the send buffer is safe to modify, or the receive buffer contains the fully arrived payload.
>
> **Message Envelope:** The quintuple of metadata attributes (`count`, `datatype`, `source/dest`, `tag`, `communicator`) accompanying a message buffer required by the MPI subsystem to route and match incoming packets correctly.
>
> **Message Tag:** An integer identifier assigned to a point-to-point message used by the receiving process to selectively filter, distinguish, and demultiplex concurrent communication streams between the same pair of ranks.
>
> **`MPI_Status`:** A C structure returned by `MPI_Recv` containing information about an incoming message, including the sender rank (`MPI_SOURCE`), the message tag (`MPI_TAG`), and internal error flags.
>
> **`MPI_Sendrecv`:** A compound MPI function that concurrently executes a blocking send to a destination process and a blocking receive from a source process within a single call, preventing circular wait deadlocks.

---

## 9. Comparison Tables

### Eager vs Rendezvous Protocol

| Feature | Eager Protocol | Rendezvous Protocol |
| :--- | :--- | :--- |
| **Payload Size** | Small (typically $< 16\text{ KB} - 64\text{ KB}$) | Large (typically $> 64\text{ KB}$) |
| **Handshake Required?**| **No** (Sends payload immediately) | **Yes** (Requires RTS $\to$ CTS handshake) |
| **Buffering Requirement**| Copies into system buffer on receiver | Direct Zero-Copy RDMA into user buffer |
| **`MPI_Send` Return** | Returns immediately upon local buffer copy | Returns only after remote receive is posted |
| **Deadlock Vulnerability**| Masks latent deadlocks in send-send loops | Exposes deadlocks immediately |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Fixing the Send-Send Deadlock
**Question:** Analyze the following code designed to exchange a 10 MB array between Rank 0 and Rank 1. Explain why it deadlocks and provide two distinct fixes.

```c
/* DEFECTIVE BIDIRECTIONAL EXCHANGE */
if (rank == 0) {
    MPI_Send(send_buf, 1250000, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD);
    MPI_Recv(recv_buf, 1250000, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
} else if (rank == 1) {
    MPI_Send(send_buf, 1250000, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
    MPI_Recv(recv_buf, 1250000, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
}
```

**Solution:**
- **The Defect:** Message size is $1{,}250{,}000 \times 8\text{ bytes} = 10\text{ MB}$. This heavily exceeds the Eager buffer threshold ($\approx 64\text{ KB}$), forcing the MPI runtime to use the **Rendezvous protocol**.
- Both Rank 0 and Rank 1 call `MPI_Send()` simultaneously.
- Rank 0's send blocks waiting for Rank 1 to execute `MPI_Recv()`.
- Rank 1's send blocks waiting for Rank 0 to execute `MPI_Recv()`.
- Neither process can reach its `MPI_Recv()` call. **Result: Deadlock.**

- **Fix 1: Even/Odd Reversal (Reordering)**
```c
if (rank == 0) {
    MPI_Send(send_buf, 1250000, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD);
    MPI_Recv(recv_buf, 1250000, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
} else if (rank == 1) {
    /* Reverse order on Rank 1: Recv FIRST, then Send */
    MPI_Recv(recv_buf, 1250000, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    MPI_Send(send_buf, 1250000, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
}
```

- **Fix 2: Using `MPI_Sendrecv()`**
```c
int partner = (rank == 0) ? 1 : 0;
MPI_Sendrecv(send_buf, 1250000, MPI_DOUBLE, partner, 0,
             recv_buf, 1250000, MPI_DOUBLE, partner, 0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
```

---

### Problem 2: Buffer Truncation Error
**Question:** Rank 0 executes `MPI_Send(data, 100, MPI_INT, 1, 0, comm)`. Rank 1 executes `MPI_Recv(buffer, 80, MPI_INT, 0, 0, comm, &status)`. What error occurs and why?

**Solution:**
- Under MPI semantics, the receive buffer must have sufficient capacity to hold the entire incoming message count.
- The sender transmitted $100$ integers, but the receiver posted a buffer of only $80$ integers.
- The MPI runtime detects buffer overflow and aborts the execution with the fatal error **`MPI_ERR_TRUNCATE`** (Message truncated).
- *Correction:* The receiver must allocate and specify `count >= 100`.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Does `MPI_Send()` returning guarantee that the receiver has received the message?
**Answer:** **No!** This is a critical misconception. `MPI_Send()` guarantees only that the sender's buffer is **safe to reuse or overwrite**. 
- Under the Eager protocol (small messages), `MPI_Send()` copies the data into a local network buffer and returns immediately, even if the receiving process has not even started or called `MPI_Recv()` yet!
- Only synchronous send (`MPI_Ssend()`) guarantees that the receiver has actively begun or completed reception.

### Q2: What is the purpose of `MPI_Get_count()` and why is it needed when `status` already exists?
**Answer:** The `MPI_Status` structure stores fields such as `MPI_SOURCE`, `MPI_TAG`, and `MPI_ERROR`, but it does **not** store an explicit integer count field. The byte length of the transfer is encoded in private, platform-dependent internal header bits inside `status`. To extract the human-readable element count according to a specific datatype (e.g., how many `MPI_DOUBLE`s or `MPI_INT`s arrived), you must pass `&status` and the datatype to `MPI_Get_count(&status, datatype, &count)`.

### Q3: What happens when `MPI_PROC_NULL` is passed as a destination or source in `MPI_Send` or `MPI_Recv`?
**Answer:** `MPI_PROC_NULL` is a special dummy rank constant defined in `<mpi.h>`. When passed as a destination in `MPI_Send`, the call returns immediately with `MPI_SUCCESS` without sending any data. When passed as a source in `MPI_Recv`, the call returns immediately with `MPI_SUCCESS`, leaving the receive buffer untouched and returning an `MPI_Status` indicating `source = MPI_PROC_NULL` and count $= 0$. This eliminates boundary `if (rank > 0)` checks in linear halo exchanges.

---

## 12. Quick Revision Sheet

- **Message Envelope:** Buffer address, element count, datatype, destination/source rank, tag, communicator.
- **Blocking Semantics:**
  - `MPI_Send()`: Returns when sender buffer is reusable.
  - `MPI_Recv()`: Returns when receive buffer contains valid incoming data.
- **Protocols:**
  - Eager: Small messages; buffered internally; returns fast; can mask deadlocks.
  - Rendezvous: Large messages; requires RTS/CTS handshake; causes deadlocks in naive send-send loops.
- **Deadlock Fixes:**
  - Even/Odd send-recv ordering.
  - `MPI_Sendrecv()`: Atomic bidirectional exchange.
  - Non-blocking communication (`MPI_Isend` / `MPI_Irecv`).
- **Wildcards:** `MPI_ANY_SOURCE`, `MPI_ANY_TAG`. Query true values via `status.MPI_SOURCE` and `MPI_Get_count()`.
- **Truncation Rule:** `recvcount >= sendcount`, otherwise `MPI_ERR_TRUNCATE`.

---

## 13. What You Should Be Able to Do

1. **Write point-to-point communication routines** using `MPI_Send` and `MPI_Recv` with correct envelope arguments.
2. **Implement latency and bandwidth ping-pong benchmarks** calculating microsecond transfer times.
3. **Diagnose and eliminate circular wait deadlocks** using `MPI_Sendrecv` and alternating rank orders.
4. **Inspect message metadata** using `MPI_Status` and query received element counts using `MPI_Get_count()`.
5. **Explain the difference between Eager and Rendezvous protocols** in university exams and viva defenses.
