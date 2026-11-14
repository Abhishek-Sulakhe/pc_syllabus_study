# Module 11: MPI Point-to-Point Communication & Dynamic Processes (Lectures 26–29)
# Subtopic 11.02: Non-Blocking Communication (`MPI_Isend`, `MPI_Irecv`, `MPI_Request`, `MPI_Waitall`, Overlapping Compute and Communication)

---

## 1. Basic Idea

In blocking communication (`MPI_Send` and `MPI_Recv`), the calling process is suspended until the transfer is locally complete. During this waiting time, the CPU core sits completely idle, burning power without executing productive application instructions.

**Non-Blocking Communication** breaks message transfer into two independent phases:
1. **Initiation (Posting):** `MPI_Isend()` or `MPI_Irecv()` posts the communication request to the network subsystem and **returns immediately** (in nanoseconds), without waiting for data to be transmitted or received.
2. **Completion (Synchronization):** `MPI_Wait()` or `MPI_Test()` suspends or queries execution until the background network transfer has finished and the memory buffer is safe to access.

By decoupling initiation from completion, an application can execute millions of floating-point operations while the network interface card (NIC) moves data across physical cables in the background. This fundamental technique is called **Computation–Communication Overlap** (or *Latency Hiding*).

```
+-----------------------------------------------------------------------------------------+
|                  Blocking vs. Non-Blocking Execution Timeline                           |
+-----------------------------------------------------------------------------------------+
| BLOCKING EXECUTION (Sequential Wasted Time):                                            |
|   [ Compute Phase 1 ] ---> [ MPI_Send (BLOCKED Waiting on Network) ] ---> [ Compute 2 ] |
|   |<---------------- T_total = T_compute + T_network ---------------------------------->|
|                                                                                         |
| NON-BLOCKING EXECUTION (Overlap / Latency Hiding):                                      |
|   MPI_Isend() posted immediately (Returns in 10 ns)                                     |
|   +-----------------------------------------------------------------------------------+ |
|   | CPU Core    : [ Compute Phase 2 on Interior Domain ..................... ]       | |
|   | Network NIC : [ Background DMA / RDMA Packet Transmission .............. ]       | |
|   +-----------------------------------------------------------------------------------+ |
|   MPI_Wait() confirms transfer finished                                                 |
|   |<---------------- T_total = max(T_compute, T_network) ------------------------------>|
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Hiding the Network Latency Gap
As established by the Hockney model ($T_{\text{comm}} = \alpha + \beta m$), network latency $\alpha$ is hundreds of times slower than local CPU memory bandwidth. If an algorithm must communicate at every step:
- Blocking code spends $50\%\text{ to }80\%$ of total wall-clock time waiting on network packets.
- Non-blocking code hides network transfers behind concurrent interior matrix arithmetic, reducing effective communication overhead to near zero.

### 2. Complete Immunity to Send-Send Deadlocks
In blocking communication, two processes calling `MPI_Send` to each other with large messages deadlock under the Rendezvous protocol. 
With non-blocking communication:
- Rank 0 posts `MPI_Irecv` from Rank 1, then posts `MPI_Isend` to Rank 1.
- Rank 1 posts `MPI_Irecv` from Rank 0, then posts `MPI_Isend` to Rank 0.
- Both calls return instantly! The background network daemons execute the RTS/CTS handshakes asynchronously. Deadlock is **completely impossible**.

### 3. Multi-Neighbor Halo Exchanges
In a 3D scientific grid simulation, each domain cube has **6 spatial neighbors** (North, South, East, West, Up, Down).
- With blocking communication, orchestrating 6 sends and 6 receives without deadlock requires complex, fragile alternating schedules.
- With non-blocking communication, the process simply posts **all 6 receives** (`MPI_Irecv`) and **all 6 sends** (`MPI_Isend`) simultaneously using an array of `MPI_Request` handles, calls `MPI_Waitall()`, and lets the hardware NIC route packets concurrently in full duplex.

---

## 3. How It Works

### The Anatomy of an Asynchronous Request (`MPI_Request`)

When `MPI_Isend` or `MPI_Irecv` is invoked, the function generates an opaque handle of type **`MPI_Request`**:

```
+-----------------------------------------------------------------------------------------+
|                                The MPI_Request Object                                   |
+-----------------------------------------------------------------------------------------+
| [ Internal State: Buffer Pointer | Bytes Remaining | Network Queue Pair | Active Flag ]  |
|                                                                                         |
| Initial State:  MPI_Isend(...) ---> request = 0x7ffd... (Active Transfer)               |
|                                                                                         |
| Completed:      MPI_Wait(&request, &status)                                             |
|                 ===> Blocks until transfer finishes.                                    |
|                 ===> Automatically sets: request = MPI_REQUEST_NULL                     |
+-----------------------------------------------------------------------------------------+
```

Once `MPI_Wait()` returns successfully:
- The internal request object is freed by the runtime.
- The request handle is set to the sentinel value **`MPI_REQUEST_NULL`**.
- Passing `MPI_REQUEST_NULL` to `MPI_Wait` is a legal no-op (returns immediately).

---

### The Strict Buffer Safety Contract

The greatest hazard in non-blocking communication is the **Buffer Safety Violation (Data Race)**:

```
[ THE FORBIDDEN BUFFER ZONE ]

MPI_Isend(send_buf, count, MPI_DOUBLE, 1, 0, comm, &req);
      |
      +========================================================================+
      |  DANGER: THE FORBIDDEN ZONE!                                           |
      |  The NIC is actively reading send_buf in the background!               |
      |                                                                        |
      |  ILLEGAL: Modifying send_buf[0] = 99.9;                                |
      |  ===> DATA CORRUPTION! Packets in transit will contain half-written    |
      |       corrupted garbage!                                               |
      +========================================================================+
      |
MPI_Wait(&req, &status);
      |
      v
[ SAFE ZONE: Buffer is 100% unlocked and safe to modify or read ]
```

#### The Two Golden Rules of Buffer Safety:
1. **For `MPI_Isend(buf, ...)`:** You **MUST NOT WRITE** to `buf` between the initiation call and the completion call (`MPI_Wait`). (Reading is theoretically possible, but modifying produces non-deterministic corruption).
2. **For `MPI_Irecv(buf, ...)`:** You **MUST NOT READ OR WRITE** to `buf` between the initiation call and the completion call (`MPI_Wait`). Reading will yield uninitialized garbage or partially transferred packets.

---

## 4. Important Mechanisms and Functions

### 1. Initiation Functions

```c
#include <mpi.h>

int MPI_Isend(const void *buf, int count, MPI_Datatype datatype,
              int dest, int tag, MPI_Comm comm,
              MPI_Request *request);

int MPI_Irecv(void *buf, int count, MPI_Datatype datatype,
              int source, int tag, MPI_Comm comm,
              MPI_Request *request);
```
- Parameters are identical to `MPI_Send` / `MPI_Recv`, except:
  - `status` is omitted from `MPI_Irecv`.
  - Both accept an output pointer `MPI_Request *request`.
- Return immediately with `MPI_SUCCESS` (or error code) within nanoseconds.

---

### 2. Single Completion Functions

```c
int MPI_Wait(MPI_Request *request, MPI_Status *status);
int MPI_Test(MPI_Request *request, int *flag, MPI_Status *status);
```

- **`MPI_Wait()`:** Blocking completion. Suspends execution until the operation associated with `*request` completes. Sets `*request = MPI_REQUEST_NULL`.
- **`MPI_Test()`:** Non-blocking query (polling). Returns immediately.
  - If operation completed: Sets `*flag = 1`, frees request (`MPI_REQUEST_NULL`), populates `status`.
  - If operation still in progress: Sets `*flag = 0`; request remains active.

---

### 3. Multi-Request Vector Completion Functions

When managing multiple simultaneous non-blocking operations (e.g., 6 halo faces):

```c
int MPI_Waitall(int count, MPI_Request array_of_requests[],
                MPI_Status array_of_statuses[]);

int MPI_Waitany(int count, MPI_Request array_of_requests[],
                int *index, MPI_Status *status);

int MPI_Waitsome(int incount, MPI_Request array_of_requests[],
                 int *outcount, int array_of_indices[],
                 MPI_Status array_of_statuses[]);
```

1. **`MPI_Waitall()`:** Blocks until **every single request** in the array has completed. (Standard for halo exchanges).
2. **`MPI_Waitany()`:** Blocks until **at least one** request has completed. Populates `*index` with the array position of the completed request. (Ideal for master harvesting completed worker results in arbitrary order).
3. **`MPI_Waitsome()`:** Blocks until one or more requests have completed, returning a list of completed indices in `array_of_indices`.

---

## 5. Compilable C Code Examples

### Example 1: Non-Blocking Bidirectional Ring with Computation Overlap
This complete program demonstrates overlapping communication and computation across $P$ ranks. Each process initiates non-blocking halo exchanges with its left and right neighbors, computes independent interior domain calculations while packets travel across the network, and synchronizes via `MPI_Waitall()`.

```c
/*
 * File: mpi_nonblocking_overlap.c
 * Compilation:
 *   mpicc -O2 mpi_nonblocking_overlap.c -o mpi_nonblocking_overlap
 *
 * Execution:
 *   mpirun -np 4 ./mpi_nonblocking_overlap
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define HALO_SIZE 1000000 /* 1 Million doubles = 8 MB */
#define INTERIOR_ITERATIONS 50000000L

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int left_neighbor  = (rank - 1 + size) % size;
    int right_neighbor = (rank + 1) % size;

    /* Allocate communication buffers */
    double* send_right = (double*)malloc(HALO_SIZE * sizeof(double));
    double* recv_left  = (double*)malloc(HALO_SIZE * sizeof(double));

    for (int i = 0; i < HALO_SIZE; ++i) {
        send_right[i] = (double)(rank + 1);
        recv_left[i]  = -1.0;
    }

    /* We need 2 requests: 1 for send, 1 for receive */
    MPI_Request requests[2];
    MPI_Status statuses[2];

    double t_start = MPI_Wtime();

    /* =========================================================================
     * STEP 1: POST ASYNCHRONOUS NON-BLOCKING OPERATIONS
     * Post receive FIRST, then post send. Both return immediately!
     * ========================================================================= */
    MPI_Irecv(recv_left, HALO_SIZE, MPI_DOUBLE, left_neighbor, 0,
              MPI_COMM_WORLD, &requests[0]);

    MPI_Isend(send_right, HALO_SIZE, MPI_DOUBLE, right_neighbor, 0,
              MPI_COMM_WORLD, &requests[1]);

    printf("[Rank %d] Non-blocking transfers posted in %.2f us. Starting interior compute...\n",
           rank, (MPI_Wtime() - t_start) * 1e6);

    /* =========================================================================
     * STEP 2: COMPUTATION-COMMUNICATION OVERLAP
     * While the NIC moves 8 MB over the network wire, the CPU computes!
     * (CRITICAL: Do NOT touch send_right or recv_left here!)
     * ========================================================================= */
    double interior_sum = 0.0;
    for (long i = 0; i < INTERIOR_ITERATIONS; ++i) {
        interior_sum += 0.0000001; /* Simulate heavy mathematical floating-point work */
    }

    printf("[Rank %d] Interior computation finished (Sum = %.2f). Awaiting network sync...\n",
           rank, interior_sum);

    /* =========================================================================
     * STEP 3: SYNCHRONIZATION POINT (MPI_Waitall)
     * Wait for both background network transfers to complete
     * ========================================================================= */
    MPI_Waitall(2, requests, statuses);

    double t_total = MPI_Wtime() - t_start;

    /* Verify received data */
    double expected_val = (double)(left_neighbor + 1);
    int correct = 1;
    for (int i = 0; i < HALO_SIZE; ++i) {
        if (recv_left[i] != expected_val) {
            correct = 0;
            break;
        }
    }

    printf("  >>> [Rank %d] Completed in %.4f sec! Received from Left Neighbor (%d) Verified: %s <<<\n",
           rank, t_total, left_neighbor, correct ? "SUCCESS" : "CORRUPT");

    free(send_right);
    free(recv_left);
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[Rank 0] Non-blocking transfers posted in 12.80 us. Starting interior compute...
[Rank 1] Non-blocking transfers posted in 13.10 us. Starting interior compute...
[Rank 2] Non-blocking transfers posted in 12.50 us. Starting interior compute...
[Rank 3] Non-blocking transfers posted in 12.90 us. Starting interior compute...
[Rank 0] Interior computation finished (Sum = 5.00). Awaiting network sync...
[Rank 1] Interior computation finished (Sum = 5.00). Awaiting network sync...
[Rank 2] Interior computation finished (Sum = 5.00). Awaiting network sync...
[Rank 3] Interior computation finished (Sum = 5.00). Awaiting network sync...
  >>> [Rank 0] Completed in 0.0841 sec! Received from Left Neighbor (3) Verified: SUCCESS <<<
  >>> [Rank 1] Completed in 0.0841 sec! Received from Left Neighbor (0) Verified: SUCCESS <<<
  >>> [Rank 2] Completed in 0.0842 sec! Received from Left Neighbor (1) Verified: SUCCESS <<<
  >>> [Rank 3] Completed in 0.0842 sec! Received from Left Neighbor (2) Verified: SUCCESS <<<
```

---

## 6. Mathematical Formulations & Latency Hiding Efficiency

### Mathematical Model of Computation–Communication Overlap

Let:
- $T_{\text{comp}}$ = Time required to execute interior domain computation on the CPU.
- $T_{\text{comm}} = \alpha + \beta m$ = Time required for the network to transfer the boundary halo.
- $T_{\text{post}}$ = Overhead of invoking `MPI_Isend` and `MPI_Irecv` ($\approx 20\text{–}50\text{ ns}$).
- $T_{\text{wait}}$ = Overhead of resolving `MPI_Waitall` ($\approx 50\text{ ns}$).

#### Sequential (Blocking) Time:
$$T_{\text{blocking}} = T_{\text{comp}} + T_{\text{comm}}$$

#### Overlapped (Non-Blocking) Time:
Under non-blocking communication, computation and transmission proceed concurrently:

$$T_{\text{non-blocking}} = T_{\text{post}} + \max\left( T_{\text{comp}}, T_{\text{comm}} \right) + T_{\text{wait}}$$

Since $T_{\text{post}} + T_{\text{wait}} \ll \max(T_{\text{comp}}, T_{\text{comm}})$:

$$T_{\text{non-blocking}} \approx \max\left( T_{\text{comp}}, T_{\text{comm}} \right)$$

```
Case A: Compute-Bound (T_comp >= T_comm)
  +-----------------------------------+
  | Computation (T_comp)              |
  +-----------------------------------+
  | Network (T_comm) | (Idle)         |
  +-----------------------------------+
  ===> Communication is 100% HIDDEN! Effective network cost = 0!

Case B: Communication-Bound (T_comm > T_comp)
  +-----------------------------------+
  | Computation (T_comp) | (Idle)     |
  +-----------------------------------+
  | Network (T_comm)                  |
  +-----------------------------------+
  ===> Computation is 100% HIDDEN!
```

#### Communication Hiding Efficiency ($\eta_{\text{hide}}$):

$$\eta_{\text{hide}} = \frac{T_{\text{blocking}} - T_{\text{non-blocking}}}{T_{\text{comm}}} = \frac{(T_{\text{comp}} + T_{\text{comm}}) - \max(T_{\text{comp}}, T_{\text{comm}})}{T_{\text{comm}}} = \frac{\min(T_{\text{comp}}, T_{\text{comm}})}{T_{\text{comm}}}$$

- If $T_{\text{comp}} \ge T_{\text{comm}}$:
$$\eta_{\text{hide}} = \frac{T_{\text{comm}}}{T_{\text{comm}}} = 1.0 \quad \mathbf{(100\%\text{ Network Latency Hiding!})}$$

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Trade-off Matrix

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Non-Blocking Primitives  | - Enables 100% computation-communication| - Extreme risk of buffer race conditions|
| (`MPI_Isend`, `MPI_Irecv`)|   latency hiding                        |   if modified before MPI_Wait           |
|                           | - Immune to circular send deadlocks     | - Additional memory for request arrays  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Waitall()`           | - Atomically synchronizes entire array  | - Straggler effect: waits for slowest   |
|                           |   of requests                           |   transfer among all posted requests    |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Test()` (Polling)    | - Allows polling network progress while | - Excessive polling burns CPU cycles    |
|                           |   interleaving incremental work         |                                         |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Non-Blocking Communication:** A communication protocol where the initiation call (`MPI_Isend` / `MPI_Irecv`) posts the network request and returns immediately to the application without waiting for data transmission, requiring a separate synchronization call (`MPI_Wait` / `MPI_Test`) to guarantee completion.
>
> **Computation–Communication Overlap:** A high-performance computing optimization technique wherein CPU arithmetic computation executes concurrently with background network data transmission, effectively hiding network transfer latencies.
>
> **`MPI_Request`:** An opaque handle that tracks the status and progress of an active asynchronous non-blocking communication operation until deallocated by a completion call.
>
> **Buffer Safety Contract:** The operational requirement stipulating that memory buffers passed to non-blocking MPI calls must not be accessed or modified by application code until an explicit completion function (`MPI_Wait` or `MPI_Test`) verifies completion.
>
> **`MPI_Waitall`:** A synchronization function that suspends execution until all non-blocking operations represented in an array of `MPI_Request` handles have completely finished.

---

## 9. Comparison Tables

### `MPI_Wait` vs `MPI_Test` vs `MPI_Waitall` vs `MPI_Waitany`

| Function | Blocking? | Target Scope | Output / Information |
| :--- | :--- | :--- | :--- |
| **`MPI_Wait(&req, &st)`** | **Yes** (Blocks) | Single Request | Suspends until request completes |
| **`MPI_Test(&req, &flag, &st)`**| **No** (Returns fast) | Single Request | Sets `flag = 1` if done, `flag = 0` if pending |
| **`MPI_Waitall(N, reqs, sts)`** | **Yes** (Blocks) | All $N$ Requests | Blocks until **every** request is finished |
| **`MPI_Waitany(N, reqs, &idx, &st)`**| **Yes** (Blocks) | First of $N$ Requests | Blocks until **any single** request finishes; returns `idx` |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Detecting the Non-Blocking Buffer Data Race
**Question:** Analyze the following code snippet. Identify the catastrophic concurrency bug and write the minimal correct fix.

```c
MPI_Request req;
double local_buffer[100];

/* Initialize buffer */
for (int i = 0; i < 100; ++i) local_buffer[i] = 1.0;

/* Post asynchronous send */
MPI_Isend(local_buffer, 100, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, &req);

/* Compute new data into buffer */
for (int i = 0; i < 100; ++i) {
    local_buffer[i] = local_buffer[i] * 5.0; /* MODIFIES BUFFER! */
}

MPI_Wait(&req, MPI_STATUS_IGNORE);
```

**Analysis & Solution:**
- **The Defect:** Between `MPI_Isend()` and `MPI_Wait()`, the application executes a loop that modifies `local_buffer`.
- Because `MPI_Isend()` returns immediately, the operating system and NIC hardware read `local_buffer` **asynchronously in the background**.
- The CPU write loop races with the network transmission: some packets will carry the initial `1.0`, while other packets will carry the modified `5.0`. The receiving process receives unpredictable, corrupt data.
- **The Fix:** Either allocate a **separate compute buffer** (double buffering) or defer updating `local_buffer` until **after `MPI_Wait()`**:

```c
/* Correct Approach: Defer write until after Wait */
MPI_Isend(local_buffer, 100, MPI_DOUBLE, 1, 0, MPI_COMM_WORLD, &req);

/* Compute work that does NOT touch local_buffer */
do_independent_compute();

/* Ensure transmission finished before overwriting */
MPI_Wait(&req, MPI_STATUS_IGNORE);

for (int i = 0; i < 100; ++i) {
    local_buffer[i] = local_buffer[i] * 5.0; /* 100% SAFE */
}
```

---

### Problem 2: Calculating Maximum Overlap Speedup
**Question:** In a parallel solver, each step takes $T_{\text{comp}} = 40\text{ ms}$ on the CPU and $T_{\text{comm}} = 35\text{ ms}$ on the network.
1. What is the execution time per step using blocking communication?
2. What is the execution time per step using non-blocking communication assuming ideal overlap?
3. Calculate the overall percentage execution time reduction.

**Solution:**
1. **Blocking Time:**
$$T_{\text{blocking}} = T_{\text{comp}} + T_{\text{comm}} = 40\text{ ms} + 35\text{ ms} = \mathbf{75\text{ ms}}$$

2. **Non-Blocking Overlapped Time:**
$$T_{\text{non-blocking}} = \max(T_{\text{comp}}, T_{\text{comm}}) = \max(40\text{ ms}, 35\text{ ms}) = \mathbf{40\text{ ms}}$$

3. **Time Reduction:**
$$\Delta T = \frac{75 - 40}{75} \times 100\% = \frac{35}{75} \times 100\% \approx \mathbf{46.67\% \text{ runtime reduction}}$$

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why does `MPI_Irecv` not accept an `MPI_Status` argument, whereas `MPI_Recv` does?
**Answer:** Because `MPI_Irecv` only *initiates* the receive request and returns immediately before any data or network packets have arrived. At the instant `MPI_Irecv` returns, the message source, tag, and actual count are not yet known. Therefore, the `MPI_Status` object is passed to `MPI_Wait(&request, &status)` or `MPI_Test(&request, &flag, &status)`, which populates the status structure only after the message has arrived.

### Q2: What happens if an application neglects to call `MPI_Wait` or `MPI_Test` on an active `MPI_Request`?
**Answer:** A severe **resource leak** occurs. For every posted non-blocking call, the MPI runtime allocates internal memory descriptors and tracks network queue pairs. If `MPI_Wait` is omitted, these descriptors are never deallocated, exhausting MPI system memory. Furthermore, the application has no guarantee that the transmission finished, leading to buffer corruption bugs.

### Q3: Can an array of requests passed to `MPI_Waitall` contain `MPI_REQUEST_NULL`?
**Answer:** Yes. The MPI standard explicitly specifies that `MPI_REQUEST_NULL` is a valid entry in an array passed to `MPI_Waitall()`, `MPI_Waitany()`, or `MPI_Waitsome()`. The function simply ignores any `MPI_REQUEST_NULL` handles and returns an empty `status` (`source = MPI_PROC_NULL`, count $= 0$). This is useful for boundary conditions where a corner or edge process does not have a neighbor.

---

## 12. Quick Revision Sheet

- **Core Non-Blocking Primitives:**
  - `MPI_Isend(buf, count, type, dest, tag, comm, &req)`
  - `MPI_Irecv(buf, count, type, src,  tag, comm, &req)`
- **Completion Primitives:**
  - `MPI_Wait(&req, &status)`: Blocks until complete; sets `req = MPI_REQUEST_NULL`.
  - `MPI_Test(&req, &flag, &status)`: Non-blocking check; sets `flag=1` if done.
  - `MPI_Waitall(N, req_array, status_array)`: Blocks until all $N$ finish.
- **The Golden Rule:** Never touch `send_buf` or `recv_buf` between `MPI_I*` and `MPI_Wait`!
- **Overlap Formula:** $T_{\text{overlapped}} \approx \max(T_{\text{comp}}, T_{\text{comm}})$.
- **Deadlock Immunity:** Non-blocking calls eliminate send-send circular wait deadlocks.

---

## 13. What You Should Be Able to Do

1. **Implement computation-communication overlap** by splitting halo exchanges from interior domain processing.
2. **Safely coordinate multiple non-blocking messages** using `MPI_Request` arrays and `MPI_Waitall()`.
3. **Detect and fix buffer race conditions** caused by premature buffer access before `MPI_Wait()`.
4. **Calculate theoretical overlap efficiency** and latency hiding percentages.
5. **Implement dynamic worker harvesting** using `MPI_Waitany()` and polling via `MPI_Test()`.
