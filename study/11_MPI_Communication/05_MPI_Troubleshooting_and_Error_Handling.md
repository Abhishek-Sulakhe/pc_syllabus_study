# Module 11: MPI Point-to-Point Communication & Dynamic Processes (Lectures 26–29)
# Subtopic 11.05: MPI Troubleshooting and Error Handling (`MPI_Comm_set_errhandler`, `MPI_ERRORS_RETURN`, Error Classes, Diagnostic Tree)

---

## 1. Basic Idea

By default in all standard MPI implementations, if any MPI library function encounters an error (such as passing an invalid destination rank, specifying an unallocated buffer, or attempting to communicate over a destroyed communicator), the runtime library immediately executes **`MPI_ERRORS_ARE_FATAL`**. This default policy instantly crashes the process and all cooperating ranks in `MPI_COMM_WORLD`, printing an unformatted low-level stack trace to standard error.

While immediate aborting prevents silent data corruption during early development, production software, fault-tolerant supercomputing applications, and resilient distributed microservices require graceful error recovery.

MPI provides a structured **Error Handling Framework**:
1. **Configurable Error Handlers:** Developers can override the default abort behavior by setting the error handler of a communicator to **`MPI_ERRORS_RETURN`**.
2. **Error Codes and Classes:** When an error occurs, the function returns a non-zero integer error code. MPI maps platform-dependent error codes into standardized **Error Classes** (e.g., `MPI_ERR_RANK`, `MPI_ERR_TRUNCATE`).
3. **Diagnostic Strings:** Developers can convert numeric error codes into human-readable explanatory text using **`MPI_Error_string()`**.

```
+-----------------------------------------------------------------------------------------+
|                              The MPI Error Handling Architecture                        |
+-----------------------------------------------------------------------------------------+
| DEFAULT BEHAVIOR (MPI_ERRORS_ARE_FATAL):                                                |
|   MPI_Send(..., dest = 999 /* Invalid */, ...) ---> PROCESS ABORTS! (Entire Job Dies)   |
|                                                                                         |
| CONTROLLED BEHAVIOR (MPI_ERRORS_RETURN):                                                |
|   1. MPI_Comm_set_errhandler(MPI_COMM_WORLD, MPI_ERRORS_RETURN);                        |
|   2. int rc = MPI_Send(..., dest = 999, ...);                                           |
|      ===> rc != MPI_SUCCESS (Function does NOT abort; returns error code!)              |
|   3. MPI_Error_class(rc, &err_class);  // Maps to MPI_ERR_RANK                          |
|   4. MPI_Error_string(rc, msg, &len);  // Formats: "Invalid destination rank 999"       |
|   5. Application logs error and executes safe recovery or fallback pathway.            |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Resilient Distributed Services
In distributed client-server applications (such as those studied in Subtopics 11.03 and 11.04), network disconnects, invalid input arguments, or dropped client connections are normal operational occurrences. Crashing the entire server daemon because a single client sent a malformed message is unacceptable. By enabling `MPI_ERRORS_RETURN`, the server traps the error, closes the broken client connection, and continues serving other clients.

### 2. Standardized Error Classes vs. Vendor Implementations
Different MPI vendors (Open MPI, Intel MPI, MPICH, Cray MPICH) utilize different internal numeric error codes. For example, an invalid rank error might be code `14` in MPICH and code `29` in Open MPI. The MPI standard provides **Error Classes** (e.g., `MPI_ERR_RANK`), guaranteeing that portable code can check error categories across any cluster or supercomputer architecture.

### 3. Rapid Debugging and Diagnostic Logging
Deciphering cryptic network stack traces or segmentation faults during large-scale runs is painful. Structured error handling allows developers to trap errors locally, print the offending rank, tag, communicator, and buffer dimensions, and cleanly flush diagnostic logs to disk before terminating.

---

## 3. How It Works

### The Error Class Hierarchy

The MPI standard structures errors into a two-level hierarchy:

```
+-----------------------------------------------------------------------------------------+
|                     MPI Error Code vs. Error Class Architecture                         |
+-----------------------------------------------------------------------------------------+
| Concrete Implementation Code:  rc = 0x8004120A (Open MPI Specific Error Code)           |
|                                         |                                               |
|                                         v  MPI_Error_class(rc, &errorclass)             |
| Standardized Error Class:       MPI_ERR_TRUNCATE (Portable Category across all MPIs)    |
|                                         |                                               |
|                                         v  MPI_Error_string(rc, string, &len)           |
| Human-Readable Description:     "Message truncated: receive buffer smaller than payload"|
+-----------------------------------------------------------------------------------------+
```

---

### Standard MPI Error Classes

| Standard Error Class | Underlying Defect / Trigger Condition |
| :--- | :--- |
| **`MPI_ERR_BUFFER`** | Invalid buffer pointer address (e.g., passing unallocated memory). |
| **`MPI_ERR_COUNT`** | Invalid count argument (negative number or invalid dimensions). |
| **`MPI_ERR_TYPE`** | Invalid or uncommitted datatype argument. |
| **`MPI_ERR_TAG`** | Invalid tag (negative integer or exceeding `MPI_TAG_UB`). |
| **`MPI_ERR_COMM`** | Invalid communicator handle (e.g., using `MPI_COMM_NULL` or destroyed comm). |
| **`MPI_ERR_RANK`** | Destination or source rank is negative, out-of-bounds, or invalid. |
| **`MPI_ERR_TRUNCATE`** | Receive buffer capacity is smaller than the transmitted message count. |
| **`MPI_ERR_NAME`** | Service name lookup failed, or name already published. |
| **`MPI_ERR_PORT`** | Invalid or unclosed network port string. |
| **`MPI_ERR_INTERN`** | Low-level internal network fabric or operating system failure. |

---

## 4. Important Mechanisms and Functions

### 1. Setting and Querying Error Handlers

```c
#include <mpi.h>

int MPI_Comm_set_errhandler(MPI_Comm comm, MPI_Errhandler errhandler);
int MPI_Comm_get_errhandler(MPI_Comm comm, MPI_Errhandler *errhandler);
```

- **`comm`:** The communicator whose error handling behavior is being modified (e.g., `MPI_COMM_WORLD`).
- **`errhandler`:** The target error handler:
  - `MPI_ERRORS_ARE_FATAL` (Default): Aborts process immediately on error.
  - `MPI_ERRORS_RETURN`: Suppresses abort; returns integer error code to caller.
- *Note:* In MPI-1, this was called `MPI_Errhandler_set()`. The modern standard formalizes `MPI_Comm_set_errhandler()`.

---

### 2. Error Class and Diagnostic Translation

```c
int MPI_Error_class(int errorcode, int *errorclass);
int MPI_Error_string(int errorcode, char *string, int *resultlen);
```

- **`errorcode`:** The return code returned by any MPI function.
- **`errorclass`:** Output integer mapped to a standardized `MPI_ERR_*` constant.
- **`string`:** Output character buffer of size at least `MPI_MAX_ERROR_STRING` (typically 512 bytes).
- **`resultlen`:** Output integer storing the number of characters written into `string`.

---

### 3. Custom Error Handlers

Developers can register a custom callback function that executes automatically whenever an error occurs on a communicator:

```c
void my_error_callback(MPI_Comm *comm, int *err_code, ...) {
    char err_msg[MPI_MAX_ERROR_STRING];
    int len;
    MPI_Error_string(*err_code, err_msg, &len);
    fprintf(stderr, "[CUSTOM ERROR HANDLER] Caught MPI Error: %s\n", err_msg);
    // Custom recovery, log flushing, or graceful abort logic here
}

MPI_Errhandler custom_handler;
MPI_Comm_create_errhandler(my_error_callback, &custom_handler);
MPI_Comm_set_errhandler(MPI_COMM_WORLD, custom_handler);
```

---

## 5. Compilable C Code Examples

### Example 1: Trapping Intentional Runtime Errors and Generating Diagnostic Reports
This complete program configures `MPI_COMM_WORLD` to use `MPI_ERRORS_RETURN`. It deliberately executes two severe runtime errors:
1. Attempting an `MPI_Send` with an out-of-bounds destination rank ($999$).
2. Attempting an `MPI_Recv` with an illegal negative tag ($-42$).

It traps both error codes without crashing, maps them to standard MPI error classes, translates them to human-readable strings via `MPI_Error_string()`, and prints an audit report.

```c
/*
 * File: mpi_error_handling.c
 * Compilation:
 *   mpicc -O2 mpi_error_handling.c -o mpi_error_handling
 *
 * Execution:
 *   mpirun -np 2 ./mpi_error_handling
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void handle_mpi_error(int error_code, const char* operation_name) {
    if (error_code == MPI_SUCCESS) return;

    int error_class;
    char error_string[MPI_MAX_ERROR_STRING];
    int string_length;

    /* Extract standardized error class */
    MPI_Error_class(error_code, &error_class);

    /* Extract implementation-specific error string */
    MPI_Error_string(error_code, error_string, &string_length);

    printf("  [TRAPPED ERROR in '%s']\n", operation_name);
    printf("    Return Code  : %d\n", error_code);
    printf("    Error Class  : %d ", error_class);

    if (error_class == MPI_ERR_RANK) {
        printf("(MPI_ERR_RANK: Invalid destination or source rank)\n");
    } else if (error_class == MPI_ERR_TAG) {
        printf("(MPI_ERR_TAG: Invalid message tag)\n");
    } else if (error_class == MPI_ERR_TRUNCATE) {
        printf("(MPI_ERR_TRUNCATE: Message truncated by receiver)\n");
    } else {
        printf("(Other Standard Error Class)\n");
    }

    printf("    Diagnostic   : \"%s\"\n\n", error_string);
}

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* -------------------------------------------------------------
     * STEP 1: OVERRIDE DEFAULT FATAL ABORT POLICY
     * Direct MPI to return error codes instead of killing process
     * ------------------------------------------------------------- */
    int rc = MPI_Comm_set_errhandler(MPI_COMM_WORLD, MPI_ERRORS_RETURN);
    if (rc != MPI_SUCCESS) {
        fprintf(stderr, "Failed to set error handler\n");
        MPI_Abort(MPI_COMM_WORLD, rc);
    }

    if (rank == 0) {
        printf("=================================================================\n");
        printf("  MPI CONTROLLED ERROR HANDLING DEMONSTRATION                    \n");
        printf("  Error Handler Configured: MPI_ERRORS_RETURN                    \n");
        printf("=================================================================\n\n");

        int test_payload = 42;

        /* -------------------------------------------------------------
         * TEST 1: DELIBERATE OUT-OF-BOUNDS DESTINATION RANK
         * Rank 999 does not exist in a 2-process run
         * ------------------------------------------------------------- */
        printf("[Test 1] Attempting MPI_Send to invalid destination rank 999...\n");
        rc = MPI_Send(&test_payload, 1, MPI_INT, 999 /* Bad Rank */, 0, MPI_COMM_WORLD);
        handle_mpi_error(rc, "MPI_Send with Invalid Rank");

        /* -------------------------------------------------------------
         * TEST 2: DELIBERATE ILLEGAL NEGATIVE MESSAGE TAG
         * Tags must be non-negative integers
         * ------------------------------------------------------------- */
        int recv_buffer;
        printf("[Test 2] Attempting MPI_Recv with illegal negative tag -42...\n");
        rc = MPI_Recv(&recv_buffer, 1, MPI_INT, 0, -42 /* Bad Tag */, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        handle_mpi_error(rc, "MPI_Recv with Invalid Tag");

        printf("-----------------------------------------------------------------\n");
        printf("[SUCCESS] Both errors were trapped and handled without crashing!\n");
        printf("=================================================================\n");
    }

    /* Clean Finalize */
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 2 Processes) ---
=================================================================
  MPI CONTROLLED ERROR HANDLING DEMONSTRATION                    
  Error Handler Configured: MPI_ERRORS_RETURN                    
=================================================================

[Test 1] Attempting MPI_Send to invalid destination rank 999...
  [TRAPPED ERROR in 'MPI_Send with Invalid Rank']
    Return Code  : 14
    Error Class  : 14 (MPI_ERR_RANK: Invalid destination or source rank)
    Diagnostic   : "Invalid rank, points to a process outside of the communicator"

[Test 2] Attempting MPI_Recv with illegal negative tag -42...
  [TRAPPED ERROR in 'MPI_Recv with Invalid Tag']
    Return Code  : 4
    Error Class  : 4 (MPI_ERR_TAG: Invalid message tag)
    Diagnostic   : "Invalid tag, must be non-negative"

-----------------------------------------------------------------
[SUCCESS] Both errors were trapped and handled without crashing!
=================================================================
```

---

## 6. Comprehensive MPI Troubleshooting Diagnostic Tree

When debugging distributed MPI applications, failures generally fall into four distinct categories. Use this diagnostic tree to locate and eliminate errors:

```
                                    MPI EXECUTION FAILURE
                                              |
         +--------------------+---------------+---------------+--------------------+
         |                    |                               |                    |
         v                    v                               v                    v
     DEADLOCKS           CRASHES / ABORTS               CORRUPT DATA         PERFORMANCE DROP
         |                    |                               |                    |
         +---> P2P Send-Send  +---> MPI_ERR_TRUNCATE          +---> Buffer Race    +---> Oversubscription
         |     Ordering             (Recv buffer small)       |     (Wrote buffer        (Processes > Cores)
         |                    |                               |      before Wait)  |
         +---> Tag Mismatch   +---> Segmentation Fault        |                    +---> Small Message
         |     (Send tag !=         (Null / Unmapped buffer)  +---> Endianness /         Latency Thrashing
         |      Recv tag)     |                               |     Datatype             (Overcome with
         |                    +---> Communicator Leak         |     Mismatch              aggregation)
         +---> Conditional          (Too many Comm_dup)       |                    |
               Collectives    |                               +---> Uninitialized  +---> NUMA Migration
               (Ranks omit    +---> Port Leaks                      Padding Bytes        (Fix via Core
                Bcast call)         (Unclosed ports)                                      Binding)
```

### Detailed Diagnostic Workflows:

#### 1. Diagnosing Distributed Deadlocks
- **Symptom:** Application hangs indefinitely; CPU cores spin at $100\%$; zero console output.
- **Diagnostic Method:**
  1. Attach a debugger (e.g., `gdb` or `lldb`) to running rank PIDs: `gdb -p <PID>` and run `backtrace`.
  2. If all ranks show call stacks blocked in `sys_poll` or `PMPI_Waitall`, identify the partner rank each is waiting for.
  3. Look for circular wait: $A$ waiting on $B$ while $B$ is waiting on $A$.
  4. Look for unmatched tags: Rank 0 sending tag `1`, Rank 1 waiting for tag `2`.

#### 2. Diagnosing Message Truncation (`MPI_ERR_TRUNCATE`)
- **Symptom:** Program crashes with `MPI_ERR_TRUNCATE` during `MPI_Recv`.
- **Diagnostic Method:**
  - Inspect sender's `sendcount` vs receiver's `recvcount`.
  - Remember: `recvcount >= sendcount`. Allocate a buffer large enough to hold the maximum expected upper bound.

#### 3. Diagnosing Non-Blocking Data Corruption
- **Symptom:** Numerical output contains garbage numbers or random NaN values.
- **Diagnostic Method:**
  - Search for writes to the send buffer or reads from the receive buffer located **between** `MPI_Isend`/`MPI_Irecv` and `MPI_Wait`.
  - Run the code under **Valgrind** with the MPI wrapper (`valgrind mpirun -np 2 ./app`) or use thread sanitizers (`-fsanitize=thread`).

---

## 7. Advantages, Disadvantages & Limitations

### Error Handling Comparison

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_ERRORS_ARE_FATAL`    | - Fail-fast: halts corrupted jobs       | - Unacceptable for long-running servers |
| (Default)                 |   immediately before wasting budget     | - Abrupt death: cannot flush disk logs  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_ERRORS_RETURN`       | - Enables graceful error recovery       | - Developer MUST check every return code|
|                           | - Essential for dynamic client-servers  | - Ignored error codes lead to undefined |
|                           | - Allows custom diagnostic logging      |   cascading failures                    |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **`MPI_ERRORS_ARE_FATAL`:** The default MPI error handler configured on `MPI_COMM_WORLD` that forces the immediate, unrecoverable termination of all processes in the communicator upon encountering any MPI library error.
>
> **`MPI_ERRORS_RETURN`:** A predefined MPI error handler that suppresses automatic process termination, causing MPI functions to return an integer error code directly to the calling program for inspection and recovery.
>
> **Error Class:** A standardized integer constant defined by the MPI specification (e.g., `MPI_ERR_RANK`, `MPI_ERR_TAG`) that categorizes an error into a portable, implementation-independent group.
>
> **`MPI_Error_string`:** A utility function that converts a numeric MPI error code or error class into a human-readable text string describing the underlying operational failure.
>
> **Message Truncation (`MPI_ERR_TRUNCATE`):** A fatal error occurring during receive operations when the incoming payload contains more elements than the allocated receive buffer capacity.

---

## 9. Comparison Tables

### `MPI_ERRORS_ARE_FATAL` vs `MPI_ERRORS_RETURN`

| Dimension | `MPI_ERRORS_ARE_FATAL` | `MPI_ERRORS_RETURN` |
| :--- | :--- | :--- |
| **Default Setting?** | **Yes** (Default on all communicators) | No (Must be explicitly configured) |
| **Behavior on Error** | Instant abort of all ranks in communicator | Returns non-zero integer code; continues running |
| **Developer Burden** | Zero (Automatic abort) | High (Must inspect return codes: `if (rc != 0)`) |
| **Log Flushing** | Often truncates unwritten logs | Can flush and sync logs cleanly before exit |
| **Ideal Context** | Batch compute jobs, rapid prototyping | Microservices, client-server, resilient solvers |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Implementing a Safe Error Checking Macro
**Question:** In production C code, checking return codes manually after every MPI function creates visual clutter. Write a robust C preprocessor macro `MPI_CHECK(call)` that invokes an MPI function, checks its return code, and automatically logs the error string and aborts cleanly if an error occurs.

**Solution:**

```c
#define MPI_CHECK(call) \
    do { \
        int _rc = (call); \
        if (_rc != MPI_SUCCESS) { \
            char _err_str[MPI_MAX_ERROR_STRING]; \
            int _len; \
            MPI_Error_string(_rc, _err_str, &_len); \
            fprintf(stderr, "[MPI FATAL ERROR] File: %s | Line: %d\n", __FILE__, __LINE__); \
            fprintf(stderr, "  Function Call : %s\n", #call); \
            fprintf(stderr, "  Error Message : %s\n", _err_str); \
            MPI_Abort(MPI_COMM_WORLD, _rc); \
        } \
    } while (0)

/* Usage in application: */
MPI_CHECK(MPI_Send(buffer, count, MPI_DOUBLE, target_rank, tag, MPI_COMM_WORLD));
```

- **Explanation:** 
  - The `do { ... } while(0)` idiom guarantees syntactic safety across `if-else` blocks.
  - `#call` stringifies the exact function invocation that failed.
  - Automatically captures file name (`__FILE__`) and line number (`__LINE__`).
  - Calls `MPI_Abort()` with the precise error code.

---

### Problem 2: Unmatched Datatypes in Point-to-Point
**Question:** Rank 0 executes `MPI_Send(buf, 10, MPI_INT, 1, 0, comm)`. Rank 1 executes `MPI_Recv(buf, 10, MPI_DOUBLE, 0, 0, comm, &status)`. What happens on the receiver?

**Solution:**
- In standard MPI implementations, datatypes are **not converted automatically** across point-to-point calls.
- The sender transmits $10 \times 4\text{ bytes} = 40\text{ bytes}$ of integer data.
- The receiver expects $10 \times 8\text{ bytes} = 80\text{ bytes}$ of double-precision floating-point data.
- The receiver receives $40$ bytes, interprets them as raw IEEE-754 double-precision bits, resulting in **corrupted floating-point numbers** or `NaN` values.
- If the sender transmits more bytes than the receiver allocated, an `MPI_ERR_TRUNCATE` crash occurs.
- *Rule:* Datatypes between sender and receiver must strictly match!

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Can `MPI_Error_string()` be called before `MPI_Init` or after `MPI_Finalize`?
**Answer:** No. `MPI_Error_string()` requires that the MPI runtime environment be active. The only two functions permitted outside the `MPI_Init` / `MPI_Finalize` boundary are `MPI_Initialized()` and `MPI_Finalized()`.

### Q2: Why is `MPI_ERR_TRUNCATE` considered an error rather than just filling whatever fits?
**Answer:** Because message truncation indicates a fundamental mismatch in algorithmic assumptions between the sender and receiver. If MPI silently discarded the trailing bytes of a message, the receiving process would compute on incomplete data, producing mathematically incorrect results that could go undetected for weeks in supercomputer simulations. By throwing `MPI_ERR_TRUNCATE`, MPI guarantees failure visibility.

### Q3: How do error handlers propagate across duplicated communicators?
**Answer:** When a communicator is duplicated using `MPI_Comm_dup()`, the newly created communicator **inherits the error handler** of the parent communicator. If `MPI_COMM_WORLD` was configured with `MPI_ERRORS_RETURN`, all communicators duplicated from it will also use `MPI_ERRORS_RETURN` by default.

---

## 12. Quick Revision Sheet

- **Default Handler:** `MPI_ERRORS_ARE_FATAL` (crashes all processes on error).
- **Return Code Handler:** `MPI_Comm_set_errhandler(comm, MPI_ERRORS_RETURN)`.
- **Diagnostic Functions:**
  - `MPI_Error_class(rc, &class)`: Maps to standard `MPI_ERR_*` constants.
  - `MPI_Error_string(rc, str, &len)`: Returns human-readable error text.
- **Common Error Classes:**
  - `MPI_ERR_RANK`: Rank out of range.
  - `MPI_ERR_TAG`: Tag negative or too large.
  - `MPI_ERR_TRUNCATE`: Buffer smaller than incoming message.
  - `MPI_ERR_COMM`: Invalid communicator.
  - `MPI_ERR_NAME`: Service lookup failed.
- **Diagnostic Best Practice:** Use an `MPI_CHECK(call)` macro for clean, informative debugging.

---

## 13. What You Should Be Able to Do

1. **Configure robust error handling** on communicators using `MPI_Comm_set_errhandler()`.
2. **Trap runtime exceptions cleanly** without aborting using `MPI_ERRORS_RETURN`.
3. **Translate numeric error codes** into standardized error classes and diagnostic strings via `MPI_Error_string()`.
4. **Implement robust error-checking macros** capturing file and line information.
5. **Diagnose and resolve the root causes** of common MPI errors including deadlocks, truncation, and tag mismatches.
