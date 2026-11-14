# Module 11: MPI Point-to-Point Communication & Dynamic Processes (Lectures 26–29)
# Subtopic 11.03: Dynamic Process Management and Network Ports (`MPI_Comm_spawn`, `MPI_Open_port`, `MPI_Comm_accept`, `MPI_Comm_connect`, Intercommunicators)

---

## 1. Basic Idea

In foundational MPI (MPI-1), parallel programs strictly follow the **Static Process Model**:
- The total number of processes $P$ is determined at launch time via `mpirun -np P`.
- The process team encapsulated within `MPI_COMM_WORLD` is fixed, immutable, and permanent.
- If $P$ processes are launched, exactly those $P$ processes must run until `MPI_Finalize()`. No new processes can be spawned, and no external programs can join the computation.

While the static model is simple, modern computing demands dynamic flexibility:
- **Cloud Elasticity:** Scaling worker counts up or down based on task queue depth.
- **Client–Server Architecture:** Allowing external visualization clients or database monitors to connect dynamically to a running supercomputer simulation.
- **Adaptive Mesh Refinement (AMR):** Spawning additional specialized solver processes when numerical shock waves develop.

Introduced in **MPI-2**, **Dynamic Process Management** provides two powerful mechanisms:
1. **Dynamic Process Spawning (`MPI_Comm_spawn`):** A running MPI application spawns new child processes at runtime.
2. **Client–Server Connection via Dynamic Ports (`MPI_Open_port`, `MPI_Comm_connect`, `MPI_Comm_accept`):** Two completely independent MPI programs, launched separately at different times, discover each other across the network and establish an **Intercommunicator** bridge.

```
+-----------------------------------------------------------------------------------------+
|                  Static vs. Dynamic Client-Server Process Models                        |
+-----------------------------------------------------------------------------------------+
| STATIC MODEL (MPI-1):                                                                   |
|   mpirun -np 4 ./simulation                                                             |
|   [ Rank 0 ] <======> [ Rank 1 ] <======> [ Rank 2 ] <======> [ Rank 3 ]                |
|   (MPI_COMM_WORLD is fixed for the entire duration of the run)                          |
|                                                                                         |
| DYNAMIC CLIENT-SERVER MODEL (MPI-2):                                                    |
|   Job 1: mpirun -np 1 ./server               Job 2: mpirun -np 1 ./client               |
|   +-----------------------+                  +-----------------------+                  |
|   | SERVER PROCESS        |                  | CLIENT PROCESS        |                  |
|   | MPI_Open_port()       |                  | Gets Port Name String |                  |
|   | MPI_Comm_accept()     | <==============> | MPI_Comm_connect()    |                  |
|   +-----------------------+                  +-----------------------+                  |
|                               INTERCOMMUNICATOR                                         |
|                 (Bi-directional communication bridge across jobs)                       |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Decoupling Independent Software Lifecycles
Consider a massive climate simulation running continuously for two weeks on 4,096 cores. A scientist wants to periodically connect a 3D visualization dashboard to inspect real-time atmospheric moisture fields.
- Under the static model, the visualization code would have to be compiled directly into the 4,096-core executable and run for all two weeks, consuming immense GPU/CPU resources pointlessly.
- Under the dynamic model, the simulation acts as an **MPI Server**. The visualization dashboard launches as an independent **MPI Client**, connects to the running server via a port, streams a data snapshot, and disconnects cleanly.

### 2. Task Farming and Elastic Master-Worker Architectures
In parameter-sweep applications (e.g., genetic algorithms, financial Monte Carlo simulations), the optimal number of worker processes is not known at launch time. `MPI_Comm_spawn()` allows a master coordinator to inspect queue depth and spawn batches of 16 or 32 worker ranks on demand.

### 3. Bridging Disjoint Communication Worlds (Intercommunicators)
When two independent jobs connect, they do not share `MPI_COMM_WORLD`. They belong to distinct process groups. MPI provides **Intercommunicators** to establish point-to-point and collective communication across disjoint process groups.

---

## 3. How It Works

### Intracommunicators vs. Intercommunicators

Understanding the distinction between communicator types is fundamental to dynamic MPI:

```
+-----------------------------------------------------------------------------------------+
|                        Intracommunicator vs. Intercommunicator                          |
+-----------------------------------------------------------------------------------------+
| INTRACOMMUNICATOR (Standard e.g., MPI_COMM_WORLD):                                      |
|   - Coordinates processes within a SINGLE group.                                        |
|   - Ranks are indexed from 0 to size - 1.                                               |
|   - Point-to-point and collectives operate across members of the same group.             |
|                                                                                         |
| INTERCOMMUNICATOR (Dynamic Bridge):                                                     |
|   - Connects TWO DISJOINT groups (Group A and Group B).                                 |
|   - Each group has its own local ranks (0 .. size_A-1 and 0 .. size_B-1).               |
|   - Point-to-point send from Group A specifies target rank in Group B!                  |
|                                                                                         |
|       Group A (e.g., Server)                      Group B (e.g., Client)                |
|       [ Rank 0 ] [ Rank 1 ]                       [ Rank 0 ] [ Rank 1 ]                 |
|            \        /                                  \        /                       |
|             \      /                                    \      /                        |
|              v    v                                      v    v                         |
|           [ Local Comm ]                              [ Local Comm ]                    |
|                 |                                           |                           |
|                 +============= INTERCOMMUNICATOR ===========+                           |
+-----------------------------------------------------------------------------------------+
```

---

### The Client-Server Connection Handshake Protocol

```
SERVER WORKFLOW:                                CLIENT WORKFLOW:
----------------                                ----------------
1. MPI_Init()                                   1. MPI_Init()
2. MPI_Open_port(MPI_INFO_NULL, port_name)      
   ===> Generates unique string:                
        "tag#0$description#host1:45129"         
3. Publishes port_name (or writes to file)      2. Reads port_name (from file or service)
4. MPI_Comm_accept(port_name, ..., &intercomm)  3. MPI_Comm_connect(port_name, ..., &intercomm)
        |                                               |
        +======== HANDSHAKE: Socket Connection =========+
        |                                               |
5. Established! Intercomm created.              4. Established! Intercomm created.
6. MPI_Recv(..., intercomm)                     5. MPI_Send(..., intercomm)
7. MPI_Comm_disconnect(&intercomm)              6. MPI_Comm_disconnect(&intercomm)
8. MPI_Close_port(port_name)                    7. MPI_Finalize()
9. MPI_Finalize()
```

---

## 4. Important Mechanisms and Functions

### 1. Dynamic Process Spawning

```c
#include <mpi.h>

int MPI_Comm_spawn(const char *command, char *argv[], int maxprocs,
                   MPI_Info info, int root, MPI_Comm comm,
                   MPI_Comm *intercomm, int array_of_errcodes[]);
```
- **`command`:** Path to the executable to spawn (e.g., `"./worker"`).
- **`maxprocs`:** Number of child processes to create.
- **`root`:** Rank in `comm` that provides spawn arguments (typically Rank 0).
- **`intercomm`:** Output intercommunicator connecting the parent process group to the newly spawned child process group.
- *Inside Child Process:* Children call `MPI_Init()`, then call `MPI_Comm_get_parent(&parent_intercomm)` to obtain the bridge back to the parent.

---

### 2. Opening and Closing Network Ports

```c
int MPI_Open_port(MPI_Info info, char *port_name);
int MPI_Close_port(const char *port_name);
```
- **`port_name`:** Output buffer of size at least `MPI_MAX_PORT_NAME` (typically 1024 characters).
- The MPI runtime binds an ephemeral network port and returns a formatted address string (e.g., containing IP address, port, and security token).

---

### 3. Establishing and Terminating Connections

```c
int MPI_Comm_accept(const char *port_name, MPI_Info info, int root,
                    MPI_Comm comm, MPI_Comm *newcomm);

int MPI_Comm_connect(const char *port_name, MPI_Info info, int root,
                     MPI_Comm comm, MPI_Comm *newcomm);

int MPI_Comm_disconnect(MPI_Comm *comm);
```

- **`MPI_Comm_accept()`:** Server blocks waiting for a client to connect to `port_name`. Returns an intercommunicator `newcomm`.
- **`MPI_Comm_connect()`:** Client initiates connection to server at `port_name`. Returns matching intercommunicator `newcomm`.
- **`MPI_Comm_disconnect()`:** Closes the intercommunicator cleanly, guaranteeing that all pending communications across the bridge finish before deallocation.

---

### 4. Name Publishing and Lookup Services

```c
int MPI_Publish_name(const char *service_name, MPI_Info info, const char *port_name);
int MPI_Lookup_name(const char *service_name, MPI_Info info, char *port_name);
int MPI_Unpublish_name(const char *service_name, MPI_Info info, const char *port_name);
```
- Resolves human-readable service names (e.g., `"ClimateSimulationEngine"`) to physical `port_name` strings via a shared nameserver (such as Open MPI's `ompi-server`).

---

## 5. Compilable C Code Examples

### Example 1: Full Client-Server Architecture via File-Based Port Sharing
This demonstration consists of two independent programs:
1. **`mpi_server.c`:** Initializes, opens an MPI port, writes the port string to a shared file (`server_port.txt`), and calls `MPI_Comm_accept()`.
2. **`mpi_client.c`:** Reads `server_port.txt`, calls `MPI_Comm_connect()`, sends an array of integers across the intercommunicator bridge, and disconnects.

#### The Server Program (`mpi_server.c`):
```c
/*
 * File: mpi_server.c
 * Compilation:
 *   mpicc -O2 mpi_server.c -o mpi_server
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define PORT_FILE "server_port.txt"

int main(int argc, char** argv) {
    int rank, size;
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm client_intercomm;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    printf("=== [SERVER] Starting MPI Server Daemon (Rank %d) ===\n", rank);

    /* Step 1: Open an MPI network port */
    MPI_Open_port(MPI_INFO_NULL, port_name);
    printf("[SERVER] Opened Port: %s\n", port_name);

    /* Step 2: Write port string to a shared file for client discovery */
    FILE* fp = fopen(PORT_FILE, "w");
    if (!fp) {
        perror("Failed to write port file");
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }
    fprintf(fp, "%s", port_name);
    fclose(fp);
    printf("[SERVER] Published port to '%s'. Awaiting client connection...\n", PORT_FILE);

    /* Step 3: Accept incoming connection from client */
    MPI_Comm_accept(port_name, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &client_intercomm);
    printf("[SERVER] Client successfully connected! Intercommunicator established.\n");

    /* Step 4: Receive data across intercommunicator from Client Rank 0 */
    int received_payload[5];
    MPI_Recv(received_payload, 5, MPI_INT, 0, 99, client_intercomm, MPI_STATUS_IGNORE);

    printf("[SERVER] Received data from Client Rank 0: [ ");
    for (int i = 0; i < 5; ++i) {
        printf("%d ", received_payload[i]);
    }
    printf("]\n");

    /* Step 5: Clean disconnect and port closure */
    printf("[SERVER] Disconnecting client bridge...\n");
    MPI_Comm_disconnect(&client_intercomm);
    MPI_Close_port(port_name);
    remove(PORT_FILE); /* Clean up file */

    printf("[SERVER] Server shutting down cleanly.\n");
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

#### The Client Program (`mpi_client.c`):
```c
/*
 * File: mpi_client.c
 * Compilation:
 *   mpicc -O2 mpi_client.c -o mpi_client
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define PORT_FILE "server_port.txt"

int main(int argc, char** argv) {
    int rank;
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm server_intercomm;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    printf("=== [CLIENT] Starting MPI Client (Rank %d) ===\n", rank);

    /* Step 1: Read port string from server_port.txt */
    FILE* fp = NULL;
    int retries = 0;
    while (!fp && retries < 20) {
        fp = fopen(PORT_FILE, "r");
        if (!fp) {
            usleep(200000); /* Wait 200ms */
            retries++;
        }
    }

    if (!fp) {
        fprintf(stderr, "[CLIENT] Error: Could not locate server port file '%s'\n", PORT_FILE);
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }

    if (fgets(port_name, sizeof(port_name), fp) == NULL) {
        fprintf(stderr, "[CLIENT] Error reading port file\n");
        fclose(fp);
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }
    fclose(fp);

    printf("[CLIENT] Read server port: %s\n", port_name);

    /* Step 2: Connect to the server */
    printf("[CLIENT] Connecting to server...\n");
    MPI_Comm_connect(port_name, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &server_intercomm);
    printf("[CLIENT] Connected! Sending payload to Server Rank 0...\n");

    /* Step 3: Send data across intercommunicator to Server Rank 0 */
    int payload[5] = {101, 202, 303, 404, 505};
    MPI_Send(payload, 5, MPI_INT, 0, 99, server_intercomm);
    printf("[CLIENT] Payload sent successfully.\n");

    /* Step 4: Clean disconnect */
    MPI_Comm_disconnect(&server_intercomm);
    printf("[CLIENT] Disconnected from server. Exiting cleanly.\n");

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Execution Instructions (Two Separate Shells) ---
Terminal 1:  mpirun -np 1 ./mpi_server
Terminal 2:  mpirun -np 1 ./mpi_client

--- Terminal 1 (Server Output) ---
=== [SERVER] Starting MPI Server Daemon (Rank 0) ===
[SERVER] Opened Port: tag#0$description#workstation:49152
[SERVER] Published port to 'server_port.txt'. Awaiting client connection...
[SERVER] Client successfully connected! Intercommunicator established.
[SERVER] Received data from Client Rank 0: [ 101 202 303 404 505 ]
[SERVER] Disconnecting client bridge...
[SERVER] Server shutting down cleanly.

--- Terminal 2 (Client Output) ---
=== [CLIENT] Starting MPI Client (Rank 0) ===
[CLIENT] Read server port: tag#0$description#workstation:49152
[CLIENT] Connecting to server...
[CLIENT] Connected! Sending payload to Server Rank 0...
[CLIENT] Payload sent successfully.
[CLIENT] Disconnected from server. Exiting cleanly.
```

---

## 6. Mathematical Formulations & Intercommunicator Addressing

### Intercommunicator Addressing Rules

In standard intracommunicator communication (`MPI_COMM_WORLD`), process addressing is symmetric:

$$\text{Valid Destination Ranks} \in [0, \text{size}_{\text{local}} - 1]$$

In an **Intercommunicator** connecting two disjoint process groups:
- Let Group $L$ (Local Group) have size $P_L$.
- Let Group $R$ (Remote Group) have size $P_R$.

When a process in Group $L$ calls:

$$\text{MPI\_Send}(..., \text{dest}, \dots, \text{intercomm})$$

The parameter $\text{dest}$ is an integer indexed into the **Remote Group**:

$$\text{dest} \in [0, P_R - 1]$$

```
Group L (Server Group: Size = 2)             Group R (Client Group: Size = 3)
+-------------------------------+             +-------------------------------+
| Rank 0                        |             | Rank 0                        |
| Rank 1                        |             | Rank 1                        |
|                               |             | Rank 2                        |
+-------------------------------+             +-------------------------------+
       |                                             ^
       | MPI_Send(..., dest=2, intercomm)            |
       +---------------------------------------------+
       (Rank 0 of Server sends directly to Rank 2 of Client!)
```

To query the size of the remote group from within Group $L$:

```c
int remote_size;
MPI_Comm_remote_size(intercomm, &remote_size); // Returns P_R
```

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Dynamic Process Management Trade-offs

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Comm_spawn()`        | - Elastic resource scaling              | - Spawn latency is high (~100 ms)       |
|                           | - On-demand worker allocation           | - Requires scheduler/daemon permission  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Dynamic Ports             | - Connects completely independent jobs  | - Port string sharing requires shared   |
| (`Open_port` / `connect`) | - True Client-Server microservices      |   filesystem or running nameserver      |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Comm_disconnect()`   | - Guarantees pending messages clear     | - Omitting disconnect causes hanging    |
|                           |   before intercommunicator destruction  |   at MPI_Finalize()                     |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Dynamic Process Model:** An execution model introduced in MPI-2 that allows an active MPI computation to spawn new processes dynamically or connect to independently running MPI applications across network ports.
>
> **Intercommunicator:** An opaque MPI communicator object that links two disjoint process groups, enabling point-to-point and collective message passing between members of the two distinct groups.
>
> **Intracommunicator:** A standard MPI communicator (e.g., `MPI_COMM_WORLD`) encompassing a single group of processes that share a common rank numbering space.
>
> **MPI Port (`MPI_Open_port`):** A network abstraction represented as a unique character string generated by the MPI runtime that enables client processes to discover, address, and connect to a listening server process.
>
> **`MPI_Comm_spawn`:** A collective function that requests the runtime environment to start a specified number of new MPI processes, returning an intercommunicator linking the parent group to the newly created child group.

---

## 9. Comparison Tables

### Static Launch vs Dynamic Spawning vs Dynamic Ports

| Dimension | Static Launch (`mpirun -np N`) | Dynamic Spawn (`MPI_Comm_spawn`) | Dynamic Ports (`MPI_Open_port`) |
| :--- | :--- | :--- | :--- |
| **Launch Time** | All processes start at $t = 0$ | Children spawned at runtime $t > 0$ | Client and Server start independently |
| **Parent/Child Tree**| Single flat process team | Strict hierarchical parent-child | Peer relationship (no parent-child) |
| **Initial Communicator**| `MPI_COMM_WORLD` shared by all | Parent & Child have separate `WORLD` | Client & Server have separate `WORLD` |
| **Connection Link** | Intracommunicator | Intercommunicator | Intercommunicator |
| **Use Case** | Traditional HPC grid solvers | Adaptive Mesh Refinement, Task farms | Live in-situ visualization, GUI monitors |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Analyzing the Client-Server Hang Bug
**Question:** A student writes an MPI client-server application. The server executes `MPI_Open_port()` and writes the port to a file. The client reads the port and executes `MPI_Comm_connect()`. However, the server immediately proceeds to call `MPI_Recv()` without calling `MPI_Comm_accept()`. What happens?

**Solution:**
- **The Defect:** Establishing a connection requires a **two-sided synchronization handshake**.
- `MPI_Comm_connect()` on the client is a blocking call that waits for the server to accept the connection.
- Because the server called `MPI_Recv()` without calling `MPI_Comm_accept()`, the server is waiting for an intercommunicator message, while the client is still blocked in `MPI_Comm_connect()` trying to create the intercommunicator!
- **Result:** **Distributed Deadlock.** Neither process can make progress.
- **Correction:** The server **must** call `MPI_Comm_accept()` to complete the connection handshake before any send or receive calls can be made on the intercommunicator.

---

### Problem 2: Communicator Disconnect vs Finalize
**Question:** Why does the MPI standard provide `MPI_Comm_disconnect()` rather than simply allowing developers to call `MPI_Comm_free()` on intercommunicators before `MPI_Finalize()`?

**Solution:**
- `MPI_Comm_free()` deallocates the local communicator data structure, but it **does not synchronize** with the remote group or guarantee that in-flight network packets have been delivered.
- `MPI_Comm_disconnect()` is a **collective synchronization primitive** across both groups. It acts as an implicit barrier that guarantees:
  1. All pending communications initiated on the intercommunicator have completed.
  2. All network buffers have been acknowledged and flushed.
  3. Both sides agree to sever the network connection cleanly.
- Calling `MPI_Comm_free()` on an active dynamic connection often causes premature socket resets, corrupting final data transfers and triggering `SIGPIPE` or network crashes during `MPI_Finalize()`.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Can a process in Group A use `MPI_Bcast()` across an intercommunicator?
**Answer:** Yes! In MPI-2, collective operations are legally defined on intercommunicators. In an intercommunicator broadcast:
- The root process in Group A specifies `root = MPI_ROOT`.
- All other processes in Group A specify `root = MPI_PROC_NULL`.
- All processes in Group B specify `root = <rank_of_root_in_Group_A>`.
The data is then multicast from the root in Group A to all processes in Group B.

### Q2: How does a child process spawned via `MPI_Comm_spawn` obtain the intercommunicator to its parent?
**Answer:** The child process calls `MPI_Init()`, and then invokes the query function:
```c
MPI_Comm parent_intercomm;
MPI_Comm_get_parent(&parent_intercomm);
```
If the process was spawned by `MPI_Comm_spawn`, `parent_intercomm` is populated with the intercommunicator bridging back to the parent process team. If the process was started as a normal standalone application via `mpirun`, `parent_intercomm` is set to `MPI_COMM_NULL`.

### Q3: What is the purpose of `ompi-server` in Open MPI?
**Answer:** When using `MPI_Publish_name()` and `MPI_Lookup_name()`, MPI processes need a centralized directory service to resolve human-readable service strings (e.g., `"DBService"`) into physical network port names. In Open MPI, `ompi-server` is a background daemon that acts as this distributed name registry, eliminating the need to manually pass port names through shared disk files.

---

## 12. Quick Revision Sheet

- **Static vs Dynamic:** Static fixes process count at launch (`mpirun -np N`); dynamic allows runtime spawning (`MPI_Comm_spawn`) and dynamic port connections.
- **Intercommunicator:** Communicator bridging two disjoint process groups (Server vs Client).
- **Client-Server Workflow:**
  1. Server: `MPI_Open_port(&port)` $\to$ publishes port string.
  2. Server: `MPI_Comm_accept(port, ..., &intercomm)` (blocks for client).
  3. Client: `MPI_Comm_connect(port, ..., &intercomm)` (connects to server).
  4. Communicate: `MPI_Send` / `MPI_Recv` on `intercomm`.
  5. Teardown: `MPI_Comm_disconnect(&intercomm)` $\to$ Server: `MPI_Close_port(port)`.
- **Addressing Rule:** In an intercommunicator, `dest` refers to the rank in the **remote** group ($0 \dots \text{remote\_size}-1$).

---

## 13. What You Should Be Able to Do

1. **Implement independent Client and Server applications** that establish dynamic MPI intercommunicator connections across network ports.
2. **Contrast Intracommunicators and Intercommunicators** in terms of process groups and addressing rules.
3. **Utilize `MPI_Comm_spawn()`** to dynamically launch child processes and exchange data via `MPI_Comm_get_parent()`.
4. **Prevent distributed deadlocks** by correctly ordering `MPI_Comm_accept` and `MPI_Comm_connect` handshakes.
5. **Ensure clean teardown** of dynamic network connections using `MPI_Comm_disconnect()`.
