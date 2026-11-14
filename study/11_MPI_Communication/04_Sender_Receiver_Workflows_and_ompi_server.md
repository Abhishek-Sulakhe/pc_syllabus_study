# Module 11: MPI Point-to-Point Communication & Dynamic Processes (Lectures 26–29)
# Subtopic 11.04: Practical Sender–Receiver Workflows and Name Publishing (`ompi-server`, `MPI_Publish_name`, `MPI_Lookup_name`, Troubleshooting)

---

## 1. Basic Idea

In Subtopic 11.03, we explored the dynamic client–server model where independent MPI programs connect via raw network port strings. In that demonstration, the client and server shared the port string by writing and reading a temporary text file on a shared filesystem.

While a file-based exchange works on a local machine or shared NFS mount, modern enterprise and cloud clusters often lack a common shared directory, or filesystem permissions prevent cross-job file sharing. 

To provide a robust, production-grade discovery mechanism, the MPI standard defines a **Distributed Name Publishing Service**:
- An MPI server publishes a human-readable service name (e.g., `"WeatherForecastingEngine"`) bound to its open port using **`MPI_Publish_name()`**.
- An MPI client searches for the service using **`MPI_Lookup_name()`**, which automatically queries the registry and resolves the human-readable string into the physical network port.
- In Open MPI, this directory service is powered by a lightweight, background daemon called **`ompi-server`**.

```
+-----------------------------------------------------------------------------------------+
|                         The ompi-server Name Publishing Architecture                    |
+-----------------------------------------------------------------------------------------+
|                                    [ ompi-server Daemon ]                               |
|                               (Centralized / Distributed Registry)                      |
|                               URI: "349182390.0;tcp://192.168.1.10:45129"               |
|                                       ^                     ^                           |
|      1. MPI_Publish_name()            |                     |   3. MPI_Lookup_name()    |
|         "WeatherEngine" -> Port_XYZ   |                     |      "WeatherEngine" ?    |
|                                       |                     |      ===> Returns Port_XYZ|
|                          +------------+                     +------------+              |
|                          |                                               |              |
|                +-------------------+                           +-------------------+    |
|                |   SERVER JOB      |   4. Handshake: Connect   |   CLIENT JOB      |    |
|                |  (mpirun -np 1)   |<=========================>|  (mpirun -np 1)   |    |
|                | MPI_Comm_accept() |                           | MPI_Comm_connect()|    |
|                +-------------------+                           +-------------------+    |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Eliminating Shared Filesystem Dependencies
Writing port strings to a file creates brittle software architectures:
- Race conditions: The client might read a half-written file or a stale file left over from a crashed server instance.
- Filesystem caching delays: Network filesystems (NFS, Lustre, GPFS) introduce attribute caching latency (up to 30 seconds), causing clients to fail to detect recently written port files.
- `ompi-server` operates purely via in-memory network sockets, guaranteeing zero caching latency and instantaneous discovery.

### 2. Multi-Service Microservices in High-Performance Computing
A large-scale scientific ecosystem may run multiple independent simulation microservices:
- Service A: `"HydrodynamicsEngine"`
- Service B: `"RadiationTransport"`
- Service C: `"VisualizationRenderer"`
A client application can dynamically query and bind to each service on demand using `MPI_Lookup_name()`, creating modular, loosely coupled HPC pipelines.

---

## 3. How It Works

### The Step-by-Step `ompi-server` Workflow

Connecting two independent MPI jobs via `ompi-server` follows a strict six-step operational sequence:

```
Step 1: Start ompi-server Daemon
  $ ompi-server --as-server-uri-file /tmp/ompi_uri.txt -d
  ===> Generates URI token string and listens on an ephemeral network socket.

Step 2: Start the Server Application
  $ mpirun --ompi-server file:/tmp/ompi_uri.txt -np 1 ./server_app
  ===> 1. Calls MPI_Open_port(..., port_name)
  ===> 2. Calls MPI_Publish_name("ComputeService", ..., port_name)
          (ompi-server registers "ComputeService" -> port_name)
  ===> 3. Calls MPI_Comm_accept(port_name, ..., &client_intercomm)
          (Blocks waiting for client)

Step 3: Start the Client Application
  $ mpirun --ompi-server file:/tmp/ompi_uri.txt -np 1 ./client_app
  ===> 1. Calls MPI_Lookup_name("ComputeService", ..., port_name)
          (ompi-server resolves "ComputeService" to port_name)
  ===> 2. Calls MPI_Comm_connect(port_name, ..., &server_intercomm)
          (Completes handshake with server)

Step 4: Active Intercommunicator Data Exchange
  ===> Client and Server exchange messages across the bridge using standard MPI_Send / MPI_Recv.

Step 5: Clean Teardown and Service Unpublishing
  ===> Both call MPI_Comm_disconnect().
  ===> Server calls MPI_Unpublish_name("ComputeService", ...).
  ===> Server calls MPI_Close_port(port_name).

Step 6: Stop ompi-server Daemon
```

---

## 4. Important Mechanisms and Functions

### 1. The Name Publishing API

```c
#include <mpi.h>

int MPI_Publish_name(const char *service_name, MPI_Info info, const char *port_name);
int MPI_Lookup_name(const char *service_name, MPI_Info info, char *port_name);
int MPI_Unpublish_name(const char *service_name, MPI_Info info, const char *port_name);
```

- **`service_name`:** User-defined string (e.g., `"MatrixInversionService"`).
- **`info`:** Configuration hints (typically `MPI_INFO_NULL`).
- **`port_name`:** The port string returned by `MPI_Open_port()`.
- **Return Values:**
  - `MPI_SUCCESS`: Success.
  - `MPI_ERR_NAME`: The specified `service_name` already exists during publish, or was not found during lookup.
  - `MPI_ERR_SERVICE`: The name publishing service (`ompi-server`) is unreachable or not running.

---

### 2. Passing the Server URI to `mpirun`

When using Open MPI, both the client and server jobs must be informed of the `ompi-server` daemon's network address:

#### Option A: Direct CLI URI Argument
```bash
mpirun --ompi-server "349182390.0;tcp://192.168.1.10:45129" -np 1 ./server
```

#### Option B: URI File Redirection (Recommended)
```bash
# Start daemon and output URI to a known file
ompi-server --as-server-uri-file /tmp/my_ompi_server.txt &

# Pass the file reference to mpirun
mpirun --ompi-server file:/tmp/my_ompi_server.txt -np 1 ./server
mpirun --ompi-server file:/tmp/my_ompi_server.txt -np 1 ./client
```

---

## 5. Compilable C Code Examples

### Example 1: Full Sender and Receiver with Name Publishing and Dynamic Intercommunicator
This complete implementation consists of:
1. **`receiver_service.c` (Server):** Opens a port, publishes the service name `"HpcDataReceiver"`, accepts the incoming connection, receives an array of floating-point data, unpublishes the name, and disconnects.
2. **`sender_client.c` (Client):** Looks up `"HpcDataReceiver"`, connects to the resolved port, transmits the array, and disconnects cleanly.

#### The Receiver Service Program (`receiver_service.c`):
```c
/*
 * File: receiver_service.c
 * Compilation:
 *   mpicc -O2 receiver_service.c -o receiver_service
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SERVICE_NAME "HpcDataReceiver"
#define DATA_COUNT 5

int main(int argc, char** argv) {
    int rank;
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm client_intercomm;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    printf("=== [RECEIVER SERVICE] Initializing Service (Rank %d) ===\n", rank);

    /* Step 1: Open an MPI port */
    int rc = MPI_Open_port(MPI_INFO_NULL, port_name);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "[RECEIVER] Failed to open port!\n");
        MPI_Abort(MPI_COMM_WORLD, rc);
    }
    printf("[RECEIVER] Bound to Port: %s\n", port_name);

    /* Step 2: Publish service name to ompi-server registry */
    printf("[RECEIVER] Publishing service name '%s' to ompi-server...\n", SERVICE_NAME);
    rc = MPI_Publish_name(SERVICE_NAME, MPI_INFO_NULL, port_name);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "[RECEIVER] Error: MPI_Publish_name failed! (Is ompi-server running?)\n");
        MPI_Close_port(port_name);
        MPI_Abort(MPI_COMM_WORLD, rc);
    }
    printf("[RECEIVER] Service published successfully. Awaiting client connection...\n");

    /* Step 3: Accept connection from client */
    MPI_Comm_accept(port_name, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &client_intercomm);
    printf("[RECEIVER] Connection established! Intercommunicator active.\n");

    /* Step 4: Receive data across intercommunicator from Client Rank 0 */
    double received_data[DATA_COUNT];
    MPI_Recv(received_data, DATA_COUNT, MPI_DOUBLE, 0, 100, client_intercomm, MPI_STATUS_IGNORE);

    printf("[RECEIVER] Received Data: [ ");
    for (int i = 0; i < DATA_COUNT; ++i) {
        printf("%.2f ", received_data[i]);
    }
    printf("]\n");

    /* Step 5: Clean Teardown */
    printf("[RECEIVER] Disconnecting client...\n");
    MPI_Comm_disconnect(&client_intercomm);

    printf("[RECEIVER] Unpublishing service name '%s'...\n", SERVICE_NAME);
    MPI_Unpublish_name(SERVICE_NAME, MPI_INFO_NULL, port_name);

    printf("[RECEIVER] Closing port...\n");
    MPI_Close_port(port_name);

    printf("[RECEIVER] Receiver service completed successfully.\n");
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

#### The Sender Client Program (`sender_client.c`):
```c
/*
 * File: sender_client.c
 * Compilation:
 *   mpicc -O2 sender_client.c -o sender_client
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define SERVICE_NAME "HpcDataReceiver"
#define DATA_COUNT 5

int main(int argc, char** argv) {
    int rank;
    char port_name[MPI_MAX_PORT_NAME];
    MPI_Comm server_intercomm;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    printf("=== [SENDER CLIENT] Starting Client Process (Rank %d) ===\n", rank);

    /* Step 1: Query ompi-server to resolve service name to port string */
    printf("[SENDER] Looking up service '%s' in ompi-server registry...\n", SERVICE_NAME);
    
    int rc = MPI_ERR_NAME;
    int retries = 0;
    while (rc != MPI_SUCCESS && retries < 10) {
        rc = MPI_Lookup_name(SERVICE_NAME, MPI_INFO_NULL, port_name);
        if (rc != MPI_SUCCESS) {
            usleep(300000); /* Wait 300ms before retrying */
            retries++;
        }
    }

    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "[SENDER] Fatal: Service '%s' not found in ompi-server!\n", SERVICE_NAME);
        MPI_Abort(MPI_COMM_WORLD, rc);
    }

    printf("[SENDER] Service resolved successfully to Port: %s\n", port_name);

    /* Step 2: Connect to the server */
    printf("[SENDER] Connecting to receiver service...\n");
    MPI_Comm_connect(port_name, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &server_intercomm);
    printf("[SENDER] Connected successfully!\n");

    /* Step 3: Send data across intercommunicator to Receiver Rank 0 */
    double data_payload[DATA_COUNT] = {3.14, 2.71, 1.41, 1.73, 9.81};
    printf("[SENDER] Sending payload [ 3.14 2.71 1.41 1.73 9.81 ] to Receiver...\n");
    MPI_Send(data_payload, DATA_COUNT, MPI_DOUBLE, 0, 100, server_intercomm);
    printf("[SENDER] Send complete.\n");

    /* Step 4: Disconnect cleanly */
    printf("[SENDER] Disconnecting intercommunicator...\n");
    MPI_Comm_disconnect(&server_intercomm);

    printf("[SENDER] Client shutting down cleanly.\n");
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Execution Walkthrough ---
1. Start ompi-server in terminal:
   $ ompi-server --as-server-uri-file /tmp/ompi.uri &

2. Start the Receiver Service:
   $ mpirun --ompi-server file:/tmp/ompi.uri -np 1 ./receiver_service

3. In another terminal, start the Sender Client:
   $ mpirun --ompi-server file:/tmp/ompi.uri -np 1 ./sender_client

--- Expected Receiver Output ---
=== [RECEIVER SERVICE] Initializing Service (Rank 0) ===
[RECEIVER] Bound to Port: tag#0$description#workstation:49152
[RECEIVER] Publishing service name 'HpcDataReceiver' to ompi-server...
[RECEIVER] Service published successfully. Awaiting client connection...
[RECEIVER] Connection established! Intercommunicator active.
[RECEIVER] Received Data: [ 3.14 2.71 1.41 1.73 9.81 ]
[RECEIVER] Disconnecting client...
[RECEIVER] Unpublishing service name 'HpcDataReceiver'...
[RECEIVER] Closing port...
[RECEIVER] Receiver service completed successfully.

--- Expected Sender Output ---
=== [SENDER CLIENT] Starting Client Process (Rank 0) ===
[SENDER] Looking up service 'HpcDataReceiver' in ompi-server registry...
[SENDER] Service resolved successfully to Port: tag#0$description#workstation:49152
[SENDER] Connecting to receiver service...
[SENDER] Connected successfully!
[SENDER] Sending payload [ 3.14 2.71 1.41 1.73 9.81 ] to Receiver...
[SENDER] Send complete.
[SENDER] Disconnecting intercommunicator...
[SENDER] Client shutting down cleanly.
```

---

## 6. Common Troubleshooting and Error Diagnostic Matrix

When developing dynamic sender–receiver workflows, errors typically stem from networking, ordering, or lifecycle violations. This diagnostic matrix provides instant solutions:

| Error / Symptom | Root Cause | Exact Solution |
| :--- | :--- | :--- |
| **`MPI_ERR_NAME: Name not found`** | Client executed `MPI_Lookup_name()` before server published, or `service_name` has a typo. | Implement a retry loop in client; verify exact string spelling. |
| **`MPI_ERR_SERVICE: Service not found`** | `ompi-server` daemon is not running, or `--ompi-server` URI flag was omitted from `mpirun`. | Start daemon with `ompi-server &`; verify URI path passed to `mpirun`. |
| **`Connection Refused` / Socket Error** | Firewall (e.g., `iptables`, `ufw`) is blocking the ephemeral port opened by `MPI_Open_port`. | Open ephemeral port range or specify interface: `mpirun --mca btl_tcp_if_include eth0`. |
| **Process Hangs on `MPI_Comm_accept`** | Server is waiting, but client crashed, used wrong port string, or failed lookup. | Check client error logs; ensure `MPI_Comm_connect` is called with matching arguments. |
| **`MPI_ERR_COMM: Invalid communicator`** | Attempted to call `MPI_Send` or `MPI_Recv` after calling `MPI_Comm_disconnect()`. | Complete all data transfers **before** calling `MPI_Comm_disconnect()`. |
| **Memory / Port Leak on Exit** | Server terminated without calling `MPI_Close_port()` and `MPI_Unpublish_name()`. | Register cleanup handlers; always close ports before calling `MPI_Finalize()`. |

---

## 7. Advantages, Disadvantages & Limitations

### Trade-off Analysis

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `ompi-server` Registry    | - Pure in-memory network discovery      | - Single point of failure if daemon dies|
|                           | - Zero shared filesystem dependency     | - Extra infrastructure step in scripts  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Publish_name()`      | - Clean, human-readable service mapping | - Name collision if two servers publish |
|                           | - Dynamic service directory             |   the exact same string                 |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Unpublish_name()`    | - Prevents stale registry records       | - If server crashes abruptly, stale     |
|                           |   when server restarts                  |   name remains in ompi-server cache     |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Name Publishing Service (`MPI_Publish_name`):** An MPI mechanism that registers a human-readable service name string with an active directory service, associating it with a physical network port opened by `MPI_Open_port`.
>
> **Name Lookup Service (`MPI_Lookup_name`):** A client-side query function that resolves a published human-readable service name into its physical MPI network port string representation.
>
> **`ompi-server`:** A dedicated background service daemon provided by Open MPI that functions as the centralized distributed directory registry for `MPI_Publish_name` and `MPI_Lookup_name`.
>
> **`MPI_Unpublish_name`:** An administrative teardown function that unlinks and removes a previously published service name from the active name registry.
>
> **Port Leak:** A resource exhaustion bug occurring when an application invokes `MPI_Open_port` but fails to invoke `MPI_Close_port` before termination, leaving kernel network sockets bound and inaccessible to future runs.

---

## 9. Comparison Tables

### File-Based Port Sharing vs Name Publishing Service (`ompi-server`)

| Dimension | File-Based Port Sharing | Name Publishing (`ompi-server`) |
| :--- | :--- | :--- |
| **Infrastructure Needed** | Common Shared Disk (NFS, Lustre) | Running `ompi-server` daemon |
| **Network Overhead** | Heavy disk I/O / NFS polling | Lightweight in-memory TCP socket queries |
| **Stale State Risk** | High (Leftover files on disk) | Medium (Cleared on daemon restart) |
| **Filesystem Caching Delay**| **Severe** (Up to 30s NFS attribute lag) | **Zero** (Instantaneous memory lookup) |
| **Portability** | Universal across all systems | Standardized API, implementation-dependent daemon |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Resolving Stale Name Publishing Collisions
**Question:** An MPI server application publishes the name `"ComputeWorker"` and terminates abnormally (due to a `SIGKILL` or power failure) before calling `MPI_Unpublish_name()`. When the server is restarted, `MPI_Publish_name("ComputeWorker", ...)` immediately fails with error code `MPI_ERR_NAME`. Explain why this happens and how to resolve it.

**Solution:**
- **The Defect:** Because the server was killed abruptly without calling `MPI_Unpublish_name()`, the entry `"ComputeWorker"` remains registered in the `ompi-server` daemon's active memory table.
- When the new server instance calls `MPI_Publish_name()`, the daemon detects that the service name is already taken and rejects the duplicate registration with **`MPI_ERR_NAME`**.
- **The Solutions:**
  1. **Programmatic Recovery:** In the server startup routine, call `MPI_Unpublish_name()` *before* calling `MPI_Publish_name()` to clear any leftover stale registrations from previous crashed runs.
  2. **Daemon Restart:** Restart the `ompi-server` daemon (`killall ompi-server` followed by restarting the daemon), which flushes all in-memory tables.

---

### Problem 2: Analyzing the Communicator Disconnect Hang
**Question:** Explain why omitting `MPI_Comm_disconnect(&intercomm)` in a client-server application causes `MPI_Finalize()` to hang indefinitely.

**Solution:**
- When `MPI_Finalize()` is called, the MPI runtime attempts to gracefully terminate all internal network subsystems and close all open sockets.
- If an intercommunicator is still active, the underlying transport layer cannot determine whether the remote peer intends to send further messages.
- The runtime waits for remaining network flow-control packets to settle.
- Calling `MPI_Comm_disconnect()` performs a coordinated, two-sided handshake between the client and server process teams, explicitly signaling that no further traffic will cross the bridge.
- Omitting this call leaves the communication channel in an uncoordinated, half-open state, causing `MPI_Finalize()` to block indefinitely waiting for socket quiescence.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: What is the purpose of passing `--ompi-server` to `mpirun`?
**Answer:** By default, an `mpirun` job has no knowledge of background daemons running on the host or cluster. The `--ompi-server` flag provides the client and server jobs with the exact network URI (IP address, port, and security token) of the running `ompi-server` process, enabling the runtime's internal `MPI_Publish_name` and `MPI_Lookup_name` functions to route service discovery queries to the correct registry daemon.

### Q2: Can `MPI_Publish_name()` and `MPI_Lookup_name()` work across different machines on a local area network?
**Answer:** Yes. Provided that:
1. `ompi-server` is started on a node with an IP address accessible by all cluster machines.
2. The URI generated by `ompi-server` is provided to `mpirun` on all participating nodes.
3. Network firewalls allow TCP/IP traffic on the ephemeral port range used by `ompi-server` and `MPI_Open_port`.

### Q3: Why must `MPI_Close_port()` be called even after `MPI_Comm_disconnect()` has succeeded?
**Answer:** `MPI_Comm_disconnect()` tears down the *intercommunicator connection* between the client and server, but the *network port itself* remains bound and listening in the operating system kernel. If the server does not call `MPI_Close_port()`, the port descriptor leaks, and the operating system continues to reserve the port, eventually exhausting system socket resources.

---

## 12. Quick Revision Sheet

- **Name Publishing Triad:**
  - `MPI_Publish_name(service, info, port)`: Registers service name with registry.
  - `MPI_Lookup_name(service, info, port)`: Queries registry and resolves port string.
  - `MPI_Unpublish_name(service, info, port)`: Unlinks service name upon teardown.
- **`ompi-server` Daemon:**
  - Start daemon: `ompi-server --as-server-uri-file /path/uri.txt &`
  - Launch jobs: `mpirun --ompi-server file:/path/uri.txt -np 1 ./app`
- **Error Modes:**
  - `MPI_ERR_NAME`: Stale registration or name not found.
  - `MPI_ERR_SERVICE`: `ompi-server` unreachable.
- **Teardown Sequence:**
  `MPI_Comm_disconnect()` $\to$ `MPI_Unpublish_name()` $\to$ `MPI_Close_port()` $\to$ `MPI_Finalize()`.

---

## 13. What You Should Be Able to Do

1. **Deploy and manage the `ompi-server` daemon** for multi-job dynamic discovery in Open MPI.
2. **Implement dynamic Name Publishing and Lookup** using `MPI_Publish_name()` and `MPI_Lookup_name()`.
3. **Diagnose and eliminate connection errors** such as `MPI_ERR_NAME`, socket connection drops, and stale service collisions.
4. **Enforce clean teardown protocols** ensuring ports are closed and intercommunicators are cleanly disconnected.
5. **Architect modular HPC microservice pipelines** communicating across independent `mpirun` job launches.
