# Master Laboratory & Practical Code Bank: MPI Programming

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 24–31)  
**Applicability:** University Laboratory Examinations, Practical Coding Tests, and Viva Voce  
**Implementation Standard:** Every program is a self-contained, fully compilable, production-grade C application conforming to the MPI-2/MPI-3 standard. Features complete initialization, explicit rank/size extraction, wall-clock timing via `MPI_Wtime()`, deadlock-free communication protocols, and full dynamic cleanup.

---

## Table of Contents
1. [Program 1: Token Ring Pipeline with Deadlock Prevention (Blocking Send & Recv)](#program-1-token-ring-pipeline-with-deadlock-prevention)
2. [Program 2: 1D Stencil Halo Exchange with Overlap (Non-Blocking Isend & Irecv)](#program-2-1d-stencil-halo-exchange-with-overlap)
3. [Program 3: Parallel Array Distribution & Reduction (Scatter & Reduce)](#program-3-parallel-array-distribution--reduction)
4. [Program 4: Distributed Matrix-Vector Multiplication (Bcast, Scatter & Gather)](#program-4-distributed-matrix-vector-multiplication)
5. [Program 5: Parallel Numerical Quadrature for $\pi$ (Bcast & Allreduce)](#program-5-parallel-numerical-quadrature-for-pi)
6. [Program 6: Dynamic Client-Server Connection over Named Ports](#program-6-dynamic-client-server-connection-over-named-ports)

---

## Program 1: Token Ring Pipeline with Deadlock Prevention

### 1.1 Objective & Concepts
Pass a token integer around a virtual ring of $P$ processes ($0 \to 1 \to 2 \to \dots \to P-1 \to 0$), incrementing the token at each hop.
Demonstrates:
- Blocking point-to-point communication (`MPI_Send`, `MPI_Recv`).
- Ring topology indexing using modulo arithmetic: `next = (rank + 1) % size`, `prev = (rank - 1 + size) % size`.
- Preventing circular wait deadlock by breaking symmetry: Rank 0 sends first, while other ranks receive first.

### 1.2 Compilable C Code
```c
/*
 * mpi_ring_deadlock_free.c
 * Deadlock-free Token Ring Communication using MPI Blocking Point-to-Point.
 *
 * Compilation:
 *   mpicc -O3 mpi_ring_deadlock_free.c -o mpi_ring
 *
 * Execution:
 *   mpirun -np 4 ./mpi_ring
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

int main(int argc, char *argv[]) {
    int world_rank, world_size;
    int token = 0;
    const int NUM_LAPS = 3;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    if (world_size < 2) {
        if (world_rank == 0) {
            fprintf(stderr, "This program requires at least 2 MPI processes!\n");
        }
        MPI_Finalize();
        return 1;
    }

    int next_rank = (world_rank + 1) % world_size;
    int prev_rank = (world_rank - 1 + world_size) % world_size;

    if (world_rank == 0) {
        printf("=================================================================\n");
        printf("         MPI TOKEN RING COMMUNICATION (%d PROCESSES)             \n", world_size);
        printf("=================================================================\n");
    }

    MPI_Barrier(MPI_COMM_WORLD);
    double start_time = MPI_Wtime();

    for (int lap = 1; lap <= NUM_LAPS; lap++) {
        if (world_rank == 0) {
            token += 10;
            printf("[Lap %d] Rank 0 sending token (%d) to Rank %d\n", lap, token, next_rank);
            MPI_Send(&token, 1, MPI_INT, next_rank, 0, MPI_COMM_WORLD);

            /* Rank 0 waits to receive the token back from the last rank */
            MPI_Recv(&token, 1, MPI_INT, prev_rank, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            printf("[Lap %d] Rank 0 received token back (%d) from Rank %d\n\n", lap, token, prev_rank);
        } else {
            /* All worker ranks receive first, increment, then forward to next rank */
            MPI_Recv(&token, 1, MPI_INT, prev_rank, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            token += 1;
            printf("  -> Rank %d received token (%d) from Rank %d, forwarding to Rank %d\n",
                   world_rank, token, prev_rank, next_rank);
            MPI_Send(&token, 1, MPI_INT, next_rank, 0, MPI_COMM_WORLD);
        }
    }

    double end_time = MPI_Wtime();

    if (world_rank == 0) {
        printf("All %d laps completed in %.6f seconds.\n", NUM_LAPS, end_time - start_time);
        printf("Final Token Value: %d\n", token);
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return 0;
}
```

### 1.3 Expected Output
```
=================================================================
         MPI TOKEN RING COMMUNICATION (4 PROCESSES)             
=================================================================
[Lap 1] Rank 0 sending token (10) to Rank 1
  -> Rank 1 received token (11) from Rank 0, forwarding to Rank 2
  -> Rank 2 received token (12) from Rank 1, forwarding to Rank 3
  -> Rank 3 received token (13) from Rank 2, forwarding to Rank 0
[Lap 1] Rank 0 received token back (13) from Rank 3

[Lap 2] Rank 0 sending token (23) to Rank 1
...
All 3 laps completed in 0.000412 seconds.
Final Token Value: 39
=================================================================
```

---

## Program 2: 1D Stencil Halo Exchange with Overlap

### 2.1 Objective & Concepts
Perform a 1D nearest-neighbor halo exchange for iterative PDE solving.
Demonstrates:
- Non-blocking communication (`MPI_Isend`, `MPI_Irecv`).
- Using `MPI_Waitall` to synchronize multiple pending communication requests.
- Computation-communication overlap: Computing interior points while halo cells are in transit.

### 2.2 Compilable C Code
```c
/*
 * mpi_halo_exchange_overlap.c
 * Non-blocking 1D Stencil Halo Exchange with Computation Overlap.
 *
 * Compilation:
 *   mpicc -O3 mpi_halo_exchange_overlap.c -o mpi_halo
 *
 * Execution:
 *   mpirun -np 4 ./mpi_halo
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

#define LOCAL_SIZE 1000000 /* 1 Million elements per rank */

int main(int argc, char *argv[]) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* Allocate local grid with 2 halo/ghost cells: index 0 and index LOCAL_SIZE + 1 */
    int total_cells = LOCAL_SIZE + 2;
    double *u_old = (double *)malloc(total_cells * sizeof(double));
    double *u_new = (double *)malloc(total_cells * sizeof(double));

    /* Initialize local grid */
    for (int i = 1; i <= LOCAL_SIZE; i++) {
        u_old[i] = (double)(rank * 100 + i);
    }
    u_old[0] = -1.0;
    u_old[LOCAL_SIZE + 1] = -1.0;

    int left_neighbor  = (rank == 0) ? MPI_PROC_NULL : rank - 1;
    int right_neighbor = (rank == size - 1) ? MPI_PROC_NULL : rank + 1;

    MPI_Request requests[4];
    int req_count = 0;

    double t_start = MPI_Wtime();

    /* -------------------------------------------------------------
     * STEP 1: INITIATE NON-BLOCKING HALO EXCHANGES (BACKGROUND)
     * ------------------------------------------------------------- */
    /* Receive left ghost cell into index 0 */
    MPI_Irecv(&u_old[0], 1, MPI_DOUBLE, left_neighbor, 0, MPI_COMM_WORLD, &requests[req_count++]);
    /* Receive right ghost cell into index LOCAL_SIZE + 1 */
    MPI_Irecv(&u_old[LOCAL_SIZE + 1], 1, MPI_DOUBLE, right_neighbor, 1, MPI_COMM_WORLD, &requests[req_count++]);

    /* Send left boundary element (index 1) to left neighbor */
    MPI_Isend(&u_old[1], 1, MPI_DOUBLE, left_neighbor, 1, MPI_COMM_WORLD, &requests[req_count++]);
    /* Send right boundary element (index LOCAL_SIZE) to right neighbor */
    MPI_Isend(&u_old[LOCAL_SIZE], 1, MPI_DOUBLE, right_neighbor, 0, MPI_COMM_WORLD, &requests[req_count++]);

    /* -------------------------------------------------------------
     * STEP 2: OVERLAP COMPUTE (INTERIOR CELLS)
     * While network transfers proceed, compute cells that don't need halos
     * ------------------------------------------------------------- */
    for (int i = 2; i <= LOCAL_SIZE - 1; i++) {
        u_new[i] = 0.5 * u_old[i] + 0.25 * (u_old[i - 1] + u_old[i + 1]);
    }

    /* -------------------------------------------------------------
     * STEP 3: WAIT FOR HALOS AND COMPLETE BOUNDARY COMPUTE
     * ------------------------------------------------------------- */
    MPI_Waitall(req_count, requests, MPI_STATUSES_IGNORE);

    /* Update boundary cells using received halo data */
    if (left_neighbor != MPI_PROC_NULL) {
        u_new[1] = 0.5 * u_old[1] + 0.25 * (u_old[0] + u_old[2]);
    }
    if (right_neighbor != MPI_PROC_NULL) {
        u_new[LOCAL_SIZE] = 0.5 * u_old[LOCAL_SIZE] + 0.25 * (u_old[LOCAL_SIZE - 1] + u_old[LOCAL_SIZE + 1]);
    }

    double t_end = MPI_Wtime();

    printf("[Rank %d] Completed stencil step with overlap in %.6f seconds. Ghost left: %.1f, Ghost right: %.1f\n",
           rank, t_end - t_start, u_old[0], u_old[LOCAL_SIZE + 1]);

    free(u_old);
    free(u_new);
    MPI_Finalize();
    return 0;
}
```

### 2.3 Expected Output
```
[Rank 0] Completed stencil step with overlap in 0.003412 seconds. Ghost left: -1.0, Ghost right: 101.0
[Rank 1] Completed stencil step with overlap in 0.003425 seconds. Ghost left: 1000000.0, Ghost right: 201.0
[Rank 2] Completed stencil step with overlap in 0.003419 seconds. Ghost left: 1000100.0, Ghost right: 301.0
[Rank 3] Completed stencil step with overlap in 0.003410 seconds. Ghost left: 1000200.0, Ghost right: -1.0
```

---

## Program 3: Parallel Array Distribution & Reduction

### 3.1 Objective & Concepts
Distribute a master array of $N$ integers from Rank 0 to all processes, compute local partial sums, and reduce the global sum back to Rank 0.
Demonstrates:
- `MPI_Scatter` to partition data evenly across processes.
- Local computation.
- `MPI_Reduce` with operator `MPI_SUM`.

### 3.2 Compilable C Code
```c
/*
 * mpi_scatter_reduce.c
 * Parallel Array Sum using MPI_Scatter and MPI_Reduce.
 *
 * Compilation:
 *   mpicc -O3 mpi_scatter_reduce.c -o mpi_scatter_reduce
 *
 * Execution:
 *   mpirun -np 4 ./mpi_scatter_reduce
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

#define N 16000000 /* 16 Million Elements */

int main(int argc, char *argv[]) {
    int rank, size;
    long *global_array = NULL;
    long *local_array = NULL;
    long local_sum = 0, global_sum = 0;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int elements_per_proc = N / size;

    /* Allocate local sub-array for every rank */
    local_array = (long *)malloc(elements_per_proc * sizeof(long));

    /* Root process allocates and initializes the global array */
    if (rank == 0) {
        global_array = (long *)malloc(N * sizeof(long));
        for (long i = 0; i < N; i++) {
            global_array[i] = 1; /* Each element initialized to 1 */
        }
        printf("=================================================================\n");
        printf("         MPI ARRAY SCATTER & REDUCE (N = %d Elements)            \n", N);
        printf("=================================================================\n");
        printf("Distributing %d elements to each of the %d processes...\n", elements_per_proc, size);
    }

    MPI_Barrier(MPI_COMM_WORLD);
    double start_time = MPI_Wtime();

    /* Distribute contiguous chunks of global_array from Root to all ranks */
    MPI_Scatter(global_array, elements_per_proc, MPI_LONG,
                local_array, elements_per_proc, MPI_LONG,
                0, MPI_COMM_WORLD);

    /* Compute local partial sum */
    for (int i = 0; i < elements_per_proc; i++) {
        local_sum += local_array[i];
    }

    /* Reduce all partial sums to Rank 0 */
    MPI_Reduce(&local_sum, &global_sum, 1, MPI_LONG, MPI_SUM, 0, MPI_COMM_WORLD);

    double end_time = MPI_Wtime();

    if (rank == 0) {
        printf("Global Sum Calculated : %ld\n", global_sum);
        printf("Expected Sum          : %ld\n", (long)N);
        printf("Execution Time        : %.6f seconds\n", end_time - start_time);
        printf("Status                : %s\n", (global_sum == (long)N) ? "PASSED" : "FAILED");
        printf("=================================================================\n");
        free(global_array);
    }

    free(local_array);
    MPI_Finalize();
    return 0;
}
```

### 3.3 Expected Output
```
=================================================================
         MPI ARRAY SCATTER & REDUCE (N = 16000000 Elements)            
=================================================================
Distributing 4000000 elements to each of the 4 processes...
Global Sum Calculated : 16000000
Expected Sum          : 16000000
Execution Time        : 0.014280 seconds
Status                : PASSED
=================================================================
```

---

## Program 4: Distributed Matrix-Vector Multiplication

### 4.1 Objective & Concepts
Compute the matrix-vector product $\mathbf{y} = \mathbf{A} \cdot \mathbf{x}$ across distributed memory.
Demonstrates:
- `MPI_Bcast` to replicate vector $\mathbf{x}$ to all ranks.
- `MPI_Scatter` to partition the rows of matrix $\mathbf{A}$ across ranks.
- `MPI_Gather` to collect partial result segments of vector $\mathbf{y}$ back at Rank 0.

### 4.2 Compilable C Code
```c
/*
 * mpi_matvec.c
 * Parallel Matrix-Vector Multiplication using Bcast, Scatter, and Gather.
 *
 * Compilation:
 *   mpicc -O3 mpi_matvec.c -o mpi_matvec
 *
 * Execution:
 *   mpirun -np 4 ./mpi_matvec
 */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <mpi.h>

#define ROWS 4096
#define COLS 4096

int main(int argc, char *argv[]) {
    int rank, size;
    double *A = NULL;
    double *x = NULL;
    double *y = NULL;
    double *local_A = NULL;
    double *local_y = NULL;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int rows_per_proc = ROWS / size;

    /* Allocate local buffers on all processes */
    x = (double *)malloc(COLS * sizeof(double));
    local_A = (double *)malloc(rows_per_proc * COLS * sizeof(double));
    local_y = (double *)malloc(rows_per_proc * sizeof(double));

    /* Root process initializes matrix A and vector x */
    if (rank == 0) {
        A = (double *)malloc(ROWS * COLS * sizeof(double));
        y = (double *)malloc(ROWS * sizeof(double));

        for (int i = 0; i < ROWS; i++) {
            for (int j = 0; j < COLS; j++) {
                A[i * COLS + j] = 1.0;
            }
        }
        for (int j = 0; j < COLS; j++) {
            x[j] = 2.0;
        }

        printf("=================================================================\n");
        printf("         MPI MATRIX-VECTOR MULTIPLICATION (%d x %d)             \n", ROWS, COLS);
        printf("=================================================================\n");
    }

    MPI_Barrier(MPI_COMM_WORLD);
    double start = MPI_Wtime();

    /* 1. Broadcast entire vector x to all ranks */
    MPI_Bcast(x, COLS, MPI_DOUBLE, 0, MPI_COMM_WORLD);

    /* 2. Scatter rows of A to each rank */
    MPI_Scatter(A, rows_per_proc * COLS, MPI_DOUBLE,
                local_A, rows_per_proc * COLS, MPI_DOUBLE,
                0, MPI_COMM_WORLD);

    /* 3. Compute local matrix-vector product */
    for (int i = 0; i < rows_per_proc; i++) {
        local_y[i] = 0.0;
        for (int j = 0; j < COLS; j++) {
            local_y[i] += local_A[i * COLS + j] * x[j];
        }
    }

    /* 4. Gather local_y pieces back into global y at root */
    MPI_Gather(local_y, rows_per_proc, MPI_DOUBLE,
               y, rows_per_proc, MPI_DOUBLE,
               0, MPI_COMM_WORLD);

    double end = MPI_Wtime();

    if (rank == 0) {
        /* Verify correctness (Expected: 1.0 * 2.0 * COLS = 8192.0) */
        double expected = 2.0 * COLS;
        double max_err = 0.0;
        for (int i = 0; i < ROWS; i++) {
            double err = fabs(y[i] - expected);
            if (err > max_err) max_err = err;
        }

        printf("Execution Time    : %.4f seconds\n", end - start);
        printf("Max Error         : %.6e\n", max_err);
        printf("Status            : %s\n", (max_err < 1e-9) ? "PASSED" : "FAILED");
        printf("=================================================================\n");

        free(A);
        free(y);
    }

    free(x);
    free(local_A);
    free(local_y);
    MPI_Finalize();
    return 0;
}
```

### 4.3 Expected Output
```
=================================================================
         MPI MATRIX-VECTOR MULTIPLICATION (4096 x 4096)             
=================================================================
Execution Time    : 0.0182 seconds
Max Error         : 0.000000e+00
Status            : PASSED
=================================================================
```

---

## Program 5: Parallel Numerical Quadrature for $\pi$

### 5.1 Objective & Concepts
Compute $\pi$ via numerical integration across distributed memory using `MPI_Bcast` and `MPI_Allreduce`.
Demonstrates:
- Broadcasting parameter data (`NUM_STEPS`).
- Embarrassingly parallel domain slicing.
- `MPI_Allreduce` so that every process obtains the final reduced value of $\pi$.

### 5.2 Compilable C Code
```c
/*
 * mpi_pi_allreduce.c
 * Numerical Quadrature for Pi using MPI_Bcast and MPI_Allreduce.
 *
 * Compilation:
 *   mpicc -O3 mpi_pi_allreduce.c -o mpi_pi
 *
 * Execution:
 *   mpirun -np 4 ./mpi_pi
 */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <mpi.h>

int main(int argc, char *argv[]) {
    int rank, size;
    long num_steps = 100000000L; /* 100 Million Steps */
    double step, x, local_sum = 0.0, global_sum = 0.0, pi;
    const double PI_REF = 3.14159265358979323846;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* Broadcast integration steps to all ranks */
    MPI_Bcast(&num_steps, 1, MPI_LONG, 0, MPI_COMM_WORLD);

    step = 1.0 / (double)num_steps;

    MPI_Barrier(MPI_COMM_WORLD);
    double start_time = MPI_Wtime();

    /* Stride-based cyclic loop assignment across MPI processes */
    for (long i = rank; i < num_steps; i += size) {
        x = (i + 0.5) * step;
        local_sum += 4.0 / (1.0 + x * x);
    }

    /* Allreduce aggregates sum and stores result in EVERY rank */
    MPI_Allreduce(&local_sum, &global_sum, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);

    pi = step * global_sum;

    double end_time = MPI_Wtime();

    if (rank == 0) {
        printf("=================================================================\n");
        printf("         MPI NUMERICAL INTEGRATION FOR PI (ALLREDUCE)            \n");
        printf("=================================================================\n");
        printf("Processes          : %d\n", size);
        printf("Calculated Pi      : %.16f\n", pi);
        printf("True Pi            : %.16f\n", PI_REF);
        printf("Absolute Error     : %.16e\n", fabs(pi - PI_REF));
        printf("Execution Time     : %.6f seconds\n", end_time - start_time);
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return 0;
}
```

### 5.3 Expected Output
```
=================================================================
         MPI NUMERICAL INTEGRATION FOR PI (ALLREDUCE)            
=================================================================
Processes          : 4
Calculated Pi      : 3.1415926535897711
True Pi            : 3.1415926535897931
Absolute Error     : 2.1760371282655160e-14
Execution Time     : 0.061420 seconds
=================================================================
```

---

## Program 6: Dynamic Client-Server Connection over Named Ports

### 6.1 Objective & Concepts
Establish a dynamic network intercommunicator between two separately launched MPI programs (Server and Client) using `MPI_Open_port` and file/service name exchange.
Demonstrates:
- Dynamic process management in MPI-2.
- `MPI_Open_port()`, `MPI_Comm_accept()`, `MPI_Comm_connect()`.
- Exchanging data over an `MPI_Comm` Intercommunicator.

### 6.2 Compilable C Code (Unified Server & Client)
```c
/*
 * mpi_dynamic_ports.c
 * Dynamic Process Connection using MPI Ports and Intercommunicators.
 *
 * Compilation:
 *   mpicc -O3 mpi_dynamic_ports.c -o mpi_dynamic_ports
 *
 * Execution (Terminal 1 - Server):
 *   ./mpi_dynamic_ports server
 *
 * Execution (Terminal 2 - Client):
 *   ./mpi_dynamic_ports client
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mpi.h>

#define PORT_FILE "mpi_server_port.txt"

void run_server(void) {
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm intercomm;
    int received_msg = 0;

    /* 1. Open dynamic port */
    MPI_Open_port(MPI_INFO_NULL, port_name);
    printf("[Server] Dynamic Port Opened: %s\n", port_name);

    /* 2. Write port string to shared file for client discovery */
    FILE *fp = fopen(PORT_FILE, "w");
    if (!fp) { perror("fopen"); exit(1); }
    fprintf(fp, "%s", port_name);
    fclose(fp);
    printf("[Server] Published port to file '%s'. Waiting for client...\n", PORT_FILE);

    /* 3. Accept incoming client connection */
    MPI_Comm_accept(port_name, MPI_INFO_NULL, 0, MPI_COMM_SELF, &intercomm);
    printf("[Server] Client successfully connected! Established Intercommunicator.\n");

    /* 4. Receive message over intercommunicator */
    MPI_Recv(&received_msg, 1, MPI_INT, 0, 99, intercomm, MPI_STATUS_IGNORE);
    printf("[Server] Received data payload: %d from Client.\n", received_msg);

    /* 5. Cleanup */
    MPI_Comm_disconnect(&intercomm);
    MPI_Close_port(port_name);
    remove(PORT_FILE);
    printf("[Server] Disconnected and shut down port cleanly.\n");
}

void run_client(void) {
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm intercomm;
    int send_msg = 4242;

    printf("[Client] Looking up server port from file '%s'...\n", PORT_FILE);
    FILE *fp = NULL;
    while (!fp) {
        fp = fopen(PORT_FILE, "r");
    }
    fscanf(fp, "%s", port_name);
    fclose(fp);
    printf("[Client] Discovered Server Port: %s\n", port_name);

    /* 1. Connect to server port */
    MPI_Comm_connect(port_name, MPI_INFO_NULL, 0, MPI_COMM_SELF, &intercomm);
    printf("[Client] Connected to server! Established Intercommunicator.\n");

    /* 2. Send data to server */
    printf("[Client] Sending payload: %d to Server.\n", send_msg);
    MPI_Send(&send_msg, 1, MPI_INT, 0, 99, intercomm);

    /* 3. Disconnect */
    MPI_Comm_disconnect(&intercomm);
    printf("[Client] Disconnected cleanly.\n");
}

int main(int argc, char *argv[]) {
    MPI_Init(&argc, &argv);

    if (argc < 2) {
        printf("Usage: %s <server|client>\n", argv[0]);
        MPI_Finalize();
        return 1;
    }

    if (strcmp(argv[1], "server") == 0) {
        run_server();
    } else if (strcmp(argv[1], "client") == 0) {
        run_client();
    } else {
        fprintf(stderr, "Invalid mode: %s (choose 'server' or 'client')\n", argv[1]);
    }

    MPI_Finalize();
    return 0;
}
```

### 6.3 Expected Output

**Server Terminal:**
```
[Server] Dynamic Port Opened: tag#0$description#node01$port#49152$
[Server] Published port to file 'mpi_server_port.txt'. Waiting for client...
[Server] Client successfully connected! Established Intercommunicator.
[Server] Received data payload: 4242 from Client.
[Server] Disconnected and shut down port cleanly.
```

**Client Terminal:**
```
[Client] Looking up server port from file 'mpi_server_port.txt'...
[Client] Discovered Server Port: tag#0$description#node01$port#49152$
[Client] Connected to server! Established Intercommunicator.
[Client] Sending payload: 4242 to Server.
[Client] Disconnected cleanly.
```

---
*End of Master MPI Laboratory Code Bank. All programs are production-grade and fully compilable.*
