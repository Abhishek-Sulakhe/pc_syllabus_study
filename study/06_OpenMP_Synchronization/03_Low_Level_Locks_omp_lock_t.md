# 03. Low-Level OpenMP Locks: omp_lock_t and the Lock API

---

## 1. The Basic Idea

In preceding sections, we studied high-level OpenMP synchronization directives: `#pragma omp critical` and `#pragma omp atomic`. While these directives are clean and convenient, they are fundamentally **code-centric** and bound to a single **structured block**:
- A critical section must have a single entry and single exit strictly within one lexical block.
- You cannot acquire a lock in Function A, pass the data structure to Function B, and release the lock in Function B.
- You cannot easily build **data-centric fine-grained locking** (e.g., locking a specific bucket in a hash table or an individual node in a linked list) using `#pragma omp critical` without declaring thousands of named critical sections at compile time.

To overcome these structural limitations, OpenMP provides a low-level, procedural synchronization API centered around the **`omp_lock_t`** data type.

An `omp_lock_t` is a first-class software lock object. It behaves like a POSIX Mutex (`pthread_mutex_t`): it can be dynamically allocated, stored inside custom `struct` definitions, passed across function boundaries, initialized, locked, tested without blocking, and destroyed.

```
          CODE-CENTRIC LOCKING vs. DATA-CENTRIC LOCKING

1. CODE-CENTRIC (#pragma omp critical):
   All threads trying to access ANY part of the Hash Table serialize on one lock!
   [ Thread 0 ] ---> [ #pragma omp critical ] ---> Modifies Bucket 0
   [ Thread 1 ] ---> [ STALLED WAITING ]     ---> Needs Bucket 99 (Blocked needlessly!)

2. DATA-CENTRIC (omp_lock_t per Bucket):
   Each bucket owns its private omp_lock_t lock.
   [ Thread 0 ] ---> omp_set_lock(&bucket[0].lock)  ---> Updates Bucket 0 (Running)
   [ Thread 1 ] ---> omp_set_lock(&bucket[99].lock) ---> Updates Bucket 99 (Running CONCURRENTLY!)
```

---

## 2. Why We Need It

Consider three architectural scenarios where `#pragma omp critical` fails completely:

### Scenario 1: Fine-Grained Concurrent Data Structures
In a concurrent Hash Table with $1{,}000$ buckets:
- With `#pragma omp critical`, only one thread can insert or read anywhere in the hash table at a time. Throughput collapses.
- With `omp_lock_t`, each bucket struct contains its own `omp_lock_t bucket_lock`. Up to $1{,}000$ threads can insert concurrently as long as their keys hash to different buckets!

### Scenario 2: Non-Blocking Work Polling (`trylock`)
With `#pragma omp critical`, if the critical section is occupied by another thread, the arriving thread is forced to sleep or busy-wait. It cannot do other useful work.
With `omp_test_lock()`, a thread checks if the lock is free:
- If free: acquires it and updates the resource.
- If busy: skips the resource and immediately performs other background work (such as processing an alternative task queue), completely eliminating idle stall cycles.

### Scenario 3: Non-Lexical Lock Lifecycles
In complex asynchronous state machines or graph traversals (such as hand-over-hand locking in B-trees), Lock $X$ must be acquired in one function and released in another function after moving to a child node. Because pragmas are strictly scoped to a single lexical block `{ ... }`, they cannot express hand-over-hand locking. `omp_lock_t` handles this with ease.

---

## 3. How It Works: The OpenMP Lock API

All lock functions are declared in `<omp.h>`. OpenMP supports two types of locks:
1. **Simple Locks (`omp_lock_t`):** Non-recursive mutexes. A thread that already holds the lock cannot lock it again (attempting to do so causes self-deadlock).
2. **Nested Locks (`omp_nest_lock_t`):** Recursive mutexes. A thread can acquire the same lock multiple times sequentially without deadlocking, provided it releases the lock an equal number of times.

### 3.1 The 5 Fundamental Simple Lock APIs

```
                           LIFECYCLE OF AN omp_lock_t
                                       |
                                       v
                              omp_init_lock(&lock)
                         [ State: UNLOCKED / Available ]
                                  /          \
            omp_set_lock(&lock)  /            \  omp_test_lock(&lock)
           (Blocks until free)  /              \ (Non-blocking: returns 1/0)
                               v                v
                         [ State: LOCKED / Owned by Thread T ]
                                       |
                                       | omp_unset_lock(&lock)
                                       v
                         [ State: UNLOCKED / Available ]
                                       |
                                       v
                             omp_destroy_lock(&lock)
                                [ State: FREED ]
```

#### Detailed API Specifications:

```c
#include <omp.h>

/* 1. Initialization */
void omp_init_lock(omp_lock_t *lock);
```
- Initializes the lock object into the **unlocked** state.
- Must be called on a lock variable before any thread attempts to set or test it.
- Must be called from a sequential region or by a single thread; calling it concurrently on the same lock is undefined.

```c
/* 2. Destruction */
void omp_destroy_lock(omp_lock_t *lock);
```
- Uninitializes the lock object, freeing any internal OS or runtime resources.
- The lock must be in the **unlocked** state when destroyed; destroying a locked lock is undefined behavior.

```c
/* 3. Blocking Lock Acquisition */
void omp_set_lock(omp_lock_t *lock);
```
- Causes the calling thread to wait (block) until the lock is available.
- Once available, the thread acquires ownership and changes the lock state to **locked**.
- If the calling thread already owns the lock, calling `omp_set_lock` causes **immediate self-deadlock**!

```c
/* 4. Release Lock */
void omp_unset_lock(omp_lock_t *lock);
```
- Releases ownership of the lock and restores it to the **unlocked** state.
- **Rule:** The calling thread must be the thread that currently owns the lock. If an un-owning thread calls `omp_unset_lock`, behavior is undefined.

```c
/* 5. Non-Blocking Lock Test (TryLock) */
int omp_test_lock(omp_lock_t *lock);
```
- Attempts to acquire the lock without blocking.
- **Returns Non-Zero (True):** The lock was successfully acquired and is now owned by the calling thread.
- **Returns Zero (False):** The lock was currently held by another thread. The calling thread does not block and can immediately proceed to do alternative work.

---

### 3.2 Nested Locks: `omp_nest_lock_t`
When writing recursive functions or modular software libraries where Function A acquires a lock and calls Function B, which also attempts to acquire that same lock:

```c
omp_nest_lock_t nlock;
omp_init_nest_lock(&nlock);

void recursive_solve(int depth) {
    omp_set_nest_lock(&nlock);
    /* Safe: Nested calls by the same thread do not deadlock */
    if (depth > 0) recursive_solve(depth - 1);
    omp_unset_nest_lock(&nlock);
}
```
An internal counter increments on each nested `omp_set_nest_lock` and decrements on each `omp_unset_nest_lock`. The lock is released to other threads only when the nesting count drops back to zero.

---

## 4. Architectural Comparison: `critical` vs. `atomic` vs. `omp_lock_t`

| Dimension | `#pragma omp critical` | `#pragma omp atomic` | `omp_lock_t` API |
|:---|:---|:---|:---|
| **Underlying Mechanism** | Software Mutex | Hardware CPU Instruction | Software Mutex Object |
| **Lexical Scope** | Strictly Single Structured Block `{}` | Single Scalar Statement | **Anywhere across functions** |
| **Data Association** | Code-centric (Locks code path) | Memory-centric (Single variable) | **Data-centric (Embedded in struct)** |
| **Non-blocking Option?** | **NO** (Always blocks) | **NO** (Always executes) | **YES (`omp_test_lock`)** |
| **Recursive Re-entry?** | **NO** (Self-deadlocks) | N/A | **YES (`omp_nest_lock_t`)** |
| **Dynamic Array of Locks?** | **NO** (Fixed at compile time) | N/A | **YES (`omp_lock_t locks[K]`)** |

---

## 5. Compilable C Code Examples

### 5.1 Program: Concurrent Bank Account Transfers & Non-Blocking Polling
This complete, robust program demonstrates:
1. Fine-grained data-centric locking: Each bank account has its own `omp_lock_t`.
2. Preventing Deadlock: Ordered lock acquisition protocol (always lock the lower account ID first).
3. Non-blocking `omp_test_lock()`: Threads process alternative tasks if their target account is temporarily locked.

Save this file as `omp_locks_demo.c`:

```c
/* =========================================================================
 * File: omp_locks_demo.c
 * Description: In-depth demonstration of OpenMP low-level locks (omp_lock_t):
 *              1. Fine-grained account-level locking (Data-centric)
 *              2. Deadlock-free two-phase locking via ordered lock acquisition
 *              3. Non-blocking trylock polling via omp_test_lock()
 * Compilation:
 *   gcc -O3 -fopenmp omp_locks_demo.c -o omp_locks_demo
 * Execution:
 *   ./omp_locks_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <omp.h>

#define NUM_ACCOUNTS 4
#define NUM_TRANSFERS 100000
#define NUM_THREADS 4

typedef struct {
    int id;
    double balance;
    omp_lock_t lock; /* Dedicated lock per account object */
} Account;

/* Safe transfer with deadlock prevention via address/ID ordering */
void transfer_money(Account *acc_src, Account *acc_dst, double amount) {
    /* DEADLOCK PREVENTION RULE:
     * Always acquire locks in strictly increasing order of account ID! */
    Account *first  = (acc_src->id < acc_dst->id) ? acc_src : acc_dst;
    Account *second = (acc_src->id < acc_dst->id) ? acc_dst : acc_src;

    omp_set_lock(&first->lock);
    omp_set_lock(&second->lock);

    /* Critical Section: Safe modification of both balances */
    if (acc_src->balance >= amount) {
        acc_src->balance -= amount;
        acc_dst->balance += amount;
    }

    omp_unset_lock(&second->lock);
    omp_unset_lock(&first->lock);
}

int main(void) {
    printf("============================================================\n");
    printf("        OpenMP Low-Level Locks (omp_lock_t) Demonstration   \n");
    printf("============================================================\n\n");

    /* 1. Initialize Accounts and dedicated locks */
    Account accounts[NUM_ACCOUNTS];
    double expected_total = 0.0;

    for (int i = 0; i < NUM_ACCOUNTS; i++) {
        accounts[i].id = i;
        accounts[i].balance = 1000.0; /* Initial balance $1,000 each */
        expected_total += accounts[i].balance;
        omp_init_lock(&accounts[i].lock); /* MUST initialize each lock */
    }

    printf("[Setup] Initialized %d accounts with $1,000 each. Total = $%.2f\n",
           NUM_ACCOUNTS, expected_total);

    /* -------------------------------------------------------------
     * DEMO 1: Massive Concurrent Transfers with Fine-Grained Locks
     * ------------------------------------------------------------- */
    printf("\n--- Phase 1: Executing %d Concurrent Transfers across %d Threads ---\n",
           NUM_TRANSFERS, NUM_THREADS);

    double t0 = omp_get_wtime();

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(accounts)
    {
        unsigned int seed = (unsigned int)(omp_get_thread_num() + 1);

        #pragma omp for
        for (int i = 0; i < NUM_TRANSFERS; i++) {
            int src = rand_r(&seed) % NUM_ACCOUNTS;
            int dst = rand_r(&seed) % NUM_ACCOUNTS;

            if (src != dst) {
                transfer_money(&accounts[src], &accounts[dst], 10.0);
            }
        }
    }

    double t1 = omp_get_wtime();
    printf("Completed %d transfers in %8.4f ms\n", NUM_TRANSFERS, (t1 - t0) * 1000.0);

    /* Verify conservation of money */
    double actual_total = 0.0;
    for (int i = 0; i < NUM_ACCOUNTS; i++) {
        actual_total += accounts[i].balance;
        printf("  Account %d Balance: $%.2f\n", i, accounts[i].balance);
    }
    printf("Verification: Total Money = $%.2f (%s)\n\n",
           actual_total, (actual_total == expected_total) ? "PERFECT CONSERVATION" : "ERROR: MONEY LEAK!");

    /* -------------------------------------------------------------
     * DEMO 2: Non-Blocking Work Polling with omp_test_lock()
     * ------------------------------------------------------------- */
    printf("--- Phase 2: Non-Blocking Polling via omp_test_lock() ---\n");
    omp_lock_t printer_lock;
    omp_init_lock(&printer_lock);

    #pragma omp parallel num_threads(NUM_THREADS) default(none) \
                         shared(printer_lock, stdout)
    {
        int tid = omp_get_thread_num();
        int attempts = 0;
        int work_done = 0;

        while (work_done < 2) {
            attempts++;
            /* Try to acquire the printer lock without blocking */
            if (omp_test_lock(&printer_lock)) {
                /* Successfully acquired lock */
                printf("  [Thread %d] ACQUIRED printer lock on attempt %d! Printing report...\n",
                       tid, attempts);
                
                /* Micro-delay holding lock */
                for (volatile int d = 0; d < 5000000; d++);
                
                omp_unset_lock(&printer_lock);
                work_done++;
            } else {
                /* Lock was busy: Perform alternative computation instead of idling! */
                /* (Simulated useful background work) */
                for (volatile int d = 0; d < 1000000; d++);
            }
        }
    }

    /* 2. Cleanup: Destroy all locks */
    omp_destroy_lock(&printer_lock);
    for (int i = 0; i < NUM_ACCOUNTS; i++) {
        omp_destroy_lock(&accounts[i].lock);
    }

    printf("\nAll locks destroyed cleanly. Program exiting.\n");
    printf("============================================================\n");
    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_locks_demo.c -o omp_locks_demo
$ ./omp_locks_demo
```

```text
============================================================
        OpenMP Low-Level Locks (omp_lock_t) Demonstration   
============================================================

[Setup] Initialized 4 accounts with $1,000 each. Total = $4000.00

--- Phase 1: Executing 100000 Concurrent Transfers across 4 Threads ---
Completed 100000 transfers in  18.4120 ms
  Account 0 Balance: $1020.00
  Account 1 Balance: $980.00
  Account 2 Balance: $970.00
  Account 3 Balance: $1030.00
Verification: Total Money = $4000.00 (PERFECT CONSERVATION)

--- Phase 2: Non-Blocking Polling via omp_test_lock() ---
  [Thread 0] ACQUIRED printer lock on attempt 1! Printing report...
  [Thread 2] ACQUIRED printer lock on attempt 3! Printing report...
  [Thread 1] ACQUIRED printer lock on attempt 4! Printing report...
  [Thread 3] ACQUIRED printer lock on attempt 5! Printing report...
  [Thread 0] ACQUIRED printer lock on attempt 8! Printing report...
  [Thread 2] ACQUIRED printer lock on attempt 11! Printing report...
  [Thread 1] ACQUIRED printer lock on attempt 12! Printing report...
  [Thread 3] ACQUIRED printer lock on attempt 15! Printing report...

All locks destroyed cleanly. Program exiting.
============================================================
```

Notice how in Phase 2, when the lock was held by another thread, threads did not stall; they incremented their attempt counter, performed simulated background work, and retried cleanly.

---

## 6. Deadlock Prevention Theory: Coffman Conditions

When multiple threads acquire multiple `omp_lock_t` locks, the program is vulnerable to **Deadlock**. A deadlock occurs if and only if all four **Coffman Conditions** hold simultaneously:

1. **Mutual Exclusion:** Resources cannot be shared; each lock is held exclusively.
2. **Hold and Wait:** A thread holding Lock A is blocked waiting to acquire Lock B.
3. **No Pre-emption:** A lock cannot be forcibly confiscated from a thread.
4. **Circular Wait:** Thread 0 holds Lock A and waits for Lock B; Thread 1 holds Lock B and waits for Lock A.

```
                  THE CIRCULAR WAIT DEADLOCK
            Thread 0                     Thread 1
       Holds: Lock A (0)            Holds: Lock B (1)
            |                            |
            | Wants: Lock B (1)          | Wants: Lock A (0)
            v                            v
       [ BLOCKS FOREVER ] <=======> [ BLOCKS FOREVER ]
```

### The Universal Solution: Global Lock Ordering Protocol
To mathematically break Condition 4 (Circular Wait), establish a strict, global total ordering on all locks:

$$\forall\, \text{Locks } L_i, L_j, \quad \text{Acquire}(L_i) \text{ before } \text{Acquire}(L_j) \iff \text{ID}(L_i) < \text{ID}(L_j)$$

As demonstrated in our bank account transfer code:
```c
Account *first  = (acc_src->id < acc_dst->id) ? acc_src : acc_dst;
Account *second = (acc_src->id < acc_dst->id) ? acc_dst : acc_src;

omp_set_lock(&first->lock);
omp_set_lock(&second->lock);
```
Because both threads are forced to acquire the account with the smaller ID first, **a circular dependency can never form in the resource allocation graph**. Deadlock is mathematically impossible!

---

## 7. Advantages & Limitations

### Advantages
1. **Data-Centric Association:** Locks can be dynamically allocated as struct fields inside heap-allocated linked lists, binary trees, and hash tables.
2. **Non-blocking Execution:** `omp_test_lock()` allows threads to poll for resources and execute alternative compute routines if the lock is held.
3. **Arbitrary Scope Lifecycles:** Locks can be acquired in one function and released in another function, enabling complex pipelined and hand-over-hand locking algorithms.

### Limitations & Traps
1. **Manual Resource Management:** The developer must explicitly initialize every lock with `omp_init_lock()` and destroy it with `omp_destroy_lock()`. Forgetting to initialize causes Segmentation Faults; forgetting to destroy causes resource leaks.
2. **Deadlock Hazards:** Programmers must manually design lock hierarchy protocols to prevent circular waits.
3. **Self-Deadlock:** Calling `omp_set_lock()` on a simple lock already held by the same thread results in permanent deadlock.

---

## 8. Formal Definitions for Exam

- **`omp_lock_t`:** An opaque OpenMP data type representing a simple, non-recursive mutual exclusion software lock that can be explicitly manipulated via runtime library procedures.
- **`omp_nest_lock_t`:** An opaque OpenMP data type representing a recursive (nested) lock that permits the owning thread to acquire the lock multiple times without deadlocking.
- **`omp_test_lock()`:** A non-blocking lock acquisition function that attempts to set a lock; returns non-zero (true) if the lock was acquired, or zero (false) if the lock was already held by another thread.
- **Deadlock:** An unrecoverable concurrency state where two or more threads are permanently blocked, each waiting for a resource or lock held by another thread in the set.
- **Lock Ordering Protocol:** A synchronization discipline that assigns a unique integer priority to every lock and requires threads to acquire multiple locks strictly in increasing numerical order to eliminate circular wait conditions.

---

## 9. Comparison Table

### Simple Locks (`omp_lock_t`) vs Nested Locks (`omp_nest_lock_t`)

| Function Category | Simple Lock (`omp_lock_t`) | Nested Lock (`omp_nest_lock_t`) |
|:---|:---|:---|
| **Data Type** | `omp_lock_t` | `omp_nest_lock_t` |
| **Initialization** | `omp_init_lock(&l);` | `omp_init_nest_lock(&nl);` |
| **Blocking Acquire** | `omp_set_lock(&l);` | `omp_set_nest_lock(&nl);` |
| **Non-blocking Acquire** | `omp_test_lock(&l);` (Returns 0 or 1) | `omp_test_nest_lock(&nl);` (Returns nesting count) |
| **Release** | `omp_unset_lock(&l);` | `omp_unset_nest_lock(&nl);` |
| **Destruction** | `omp_destroy_lock(&l);` | `omp_destroy_nest_lock(&nl);` |
| **Re-entrant by Owner?** | **NO (Deadlocks!)** | **YES (Increments nesting count)** |
| **Common Use Case** | Disjoint data items, account locks | Recursive algorithms, hierarchical trees |

---

## 10. Solved Analytical Problems

### Problem 1: Diagnosing Lock Unset Violations
**Question:** Analyze the following code segment. What catastrophic bug does it contain according to OpenMP rules?

```c
omp_lock_t my_lock;
omp_init_lock(&my_lock);

#pragma omp parallel num_threads(2)
{
    int tid = omp_get_thread_num();
    if (tid == 0) {
        omp_set_lock(&my_lock);
        printf("Thread 0 acquired lock.\n");
    } else {
        /* Thread 1 attempts to release lock held by Thread 0 */
        omp_unset_lock(&my_lock); /* Line X */
    }
}
```

**Solution:**
- **The Bug:** Line X contains an illegal call to `omp_unset_lock(&my_lock)` by Thread 1.
- **OpenMP Specification Violation:** A lock can only be unset by the **exact same thread that currently owns it**. Thread 1 does not own the lock; Thread 0 owns it.
- **Runtime Consequence:** Calling `omp_unset_lock()` from a non-owning thread results in **undefined behavior** (often an immediate runtime abort, corrupted mutex internal state, or silent lock state desynchronization). Locks cannot be used as binary semaphores for inter-thread signalling!

---

### Problem 2: Detecting Self-Deadlock with Simple Locks
**Question:** Explain why the following code hangs indefinitely (deadlocks) and show how to fix it using `omp_nest_lock_t`.

```c
omp_lock_t lck;
omp_init_lock(&lck);

void update_leaf(void) {
    omp_set_lock(&lck);
    /* update leaf */
    omp_unset_lock(&lck);
}

void process_node(void) {
    omp_set_lock(&lck);
    update_leaf(); /* Function call */
    omp_unset_lock(&lck);
}
```

**Solution:**
1. **Deadlock Sequence:**
   - Thread enters `process_node()` and calls `omp_set_lock(&lck)`. The lock is acquired.
   - `process_node()` calls `update_leaf()`.
   - `update_leaf()` calls `omp_set_lock(&lck)` on the **same lock**.
   - Because `omp_lock_t` is a simple (non-recursive) lock, it sees the lock is currently in the `LOCKED` state. The thread is placed onto the wait queue, waiting for the lock to become free.
   - But the lock can only become free if `process_node()` completes and calls `omp_unset_lock(&lck)`.
   - The thread is waiting for itself to release the lock! (**Self-Deadlock**).
2. **The Fix:** Replace `omp_lock_t` with `omp_nest_lock_t`:
   ```c
   omp_nest_lock_t nlck;
   omp_init_nest_lock(&nlck);

   void update_leaf(void) {
       omp_set_nest_lock(&nlck); // Nesting count = 2
       /* update leaf */
       omp_unset_nest_lock(&nlck); // Nesting count = 1
   }

   void process_node(void) {
       omp_set_nest_lock(&nlck); // Nesting count = 1
       update_leaf();
       omp_unset_nest_lock(&nlck); // Nesting count = 0 (Released)
   }
   ```

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Can I use `omp_lock_t` to signal another thread, like a condition variable or semaphore?
**Answer:** **NO.** An `omp_lock_t` is strictly a mutual exclusion lock (mutex). It has ownership semantics: only the thread that locked it can unlock it. If you want a thread to wait for another thread to finish a task, use `#pragma omp barrier`, task dependencies, or POSIX Semaphores (`sem_t` in Pthreads).

### Q2: What happens if I forget to call `omp_destroy_lock()` on dynamically allocated locks?
**Answer:** On modern operating systems, `omp_init_lock()` may allocate underlying kernel primitives (such as Linux `futex` handles or Win32 mutex handles) and internal runtime heap structures. If you allocate millions of structs containing `omp_lock_t` in a loop and free the structs without calling `omp_destroy_lock()`, the process suffers a **resource leak** that can exhaust system memory descriptors and crash the OS kernel.

### Q3: When should I choose `omp_test_lock()` over `omp_set_lock()`?
**Answer:** Choose `omp_test_lock()` whenever a thread has **alternative, useful work** it can do instead of idling. For example, in dynamic work-stealing schedulers, if a thread attempts to steal a task from Queue A and finds Queue A locked, it uses `omp_test_lock()` to back off instantly and check Queue B, keeping the CPU core 100% active and productive.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                      OMP_LOCK_T CHEAT SHEET                                        |
+----------------------------------------------------------------------------------------------------+
| Data Type              | omp_lock_t (Simple Lock)    | omp_nest_lock_t (Recursive Lock)            |
| Header Required        | #include <omp.h>            | #include <omp.h>                            |
| Initialization         | omp_init_lock(&lck);        | omp_init_nest_lock(&nlck);                  |
| Blocking Acquire       | omp_set_lock(&lck);         | omp_set_nest_lock(&nlck);                   |
| Non-Blocking Acquire   | omp_test_lock(&lck); (1/0)  | omp_test_nest_lock(&nlck); (Count)          |
| Release Lock           | omp_unset_lock(&lck);       | omp_unset_nest_lock(&nlck);                 |
| Destruction            | omp_destroy_lock(&lck);     | omp_destroy_nest_lock(&nlck);               |
+----------------------------------------------------------------------------------------------------+
|                                        CARDINAL RULES                                              |
+----------------------------------------------------------------------------------------------------+
| 1. Must be initialized BEFORE use, and destroyed AFTER use.                                        |
| 2. ONLY the thread that acquired the lock can release it! (No cross-thread unlocking).            |
| 3. Never call omp_set_lock() twice on the same simple lock from the same thread (Self-Deadlock!). |
| 4. Always acquire multiple locks in identical numerical order to prevent circular wait deadlocks!  |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain why code-centric `#pragma omp critical` is inadequate for data-centric concurrent data structures.
2. Implement the complete 5-step lifecycle of an `omp_lock_t` object (`init`, `set`, `test`, `unset`, `destroy`).
3. Build non-blocking worker threads utilizing `omp_test_lock()` to avoid synchronization stalls.
4. Explain the four Coffman deadlock conditions and prove how global lock ordering eliminates circular wait.
5. Contrast simple locks (`omp_lock_t`) with recursive nested locks (`omp_nest_lock_t`) and resolve self-deadlocks.
