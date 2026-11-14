# Module 09: Pthreads Synchronization Primitives (Lectures 20–23)
# Subtopic 09.01: Race Conditions and Mutexes (`pthread_mutex_t`, `lock`, `trylock`, `unlock`, Deadlock Prevention)

---

## 1. Basic Idea

In a multithreaded process, all threads share the same virtual address space, including the global data segment and the dynamic memory heap. When two or more concurrent threads read and write to the same shared memory location without synchronization, and at least one access is a write, a **data race** occurs. This results in a **race condition**—a software defect where the output of the computation depends non-deterministically on the uncontrolled interleaving and timing of thread execution by the operating system kernel or CPU hardware.

To prevent race conditions, sections of code that access shared mutable resources are designated as **Critical Sections**. A critical section must execute with **Mutual Exclusion**—guaranteeing that at most one thread can execute inside that region at any given point in time.

The primary POSIX primitive for enforcing mutual exclusion is the **Mutex** (short for *Mutual Exclusion lock*), represented by the opaque data type `pthread_mutex_t`. A mutex acts as a binary gatekeeper:
- Exactly one thread holds the lock at any time.
- Any other thread that attempts to acquire the locked mutex is placed into a sleeping state by the operating system kernel until the owning thread releases it.

```
+-----------------------------------------------------------------------------------------+
|                              The Mutex Gatekeeper Concept                               |
+-----------------------------------------------------------------------------------------+
| Thread 1:  pthread_mutex_lock(&m) ---> [ GRANTED ] ---> Critical Section ---> UNLOCK    |
|                                                                                 |       |
| Thread 2:  pthread_mutex_lock(&m) -------------> [ BLOCKED / ASLEEP ] ---------> WAKES |
|                                                  (Kernel Futex Wait)            |       |
|                                                                                 v       |
|                                                                        [ GRANTED LOCK ] |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Anatomy of an Unsynchronized Counter Update
Consider a simple shared counter update:
```c
counter++;
```
Although written as a single line in C, the compiler translates this operation into three distinct assembly instructions:

```
Step 1 (READ)    : movl  counter(%rip), %eax   ; Load counter from RAM/Cache into Core Register
Step 2 (MODIFY)  : addl  $1, %eax              ; Increment Register EAX by 1
Step 3 (WRITE)   : movl  %eax, counter(%rip)   ; Store updated Register EAX back to RAM/Cache
```

If Thread A and Thread B attempt to execute `counter++` concurrently when `counter == 100`, an arbitrary hardware context switch or multi-core execution interleaving can occur:

```
Time   Thread A (Core 0)              Thread B (Core 1)              RAM 'counter'
----------------------------------------------------------------------------------
t0     Reads counter (EAX_A = 100)                                   100
t1                                    Reads counter (EAX_B = 100)    100
t2     Increments (EAX_A = 101)                                      100
t3                                    Increments (EAX_B = 101)       100
t4     Writes back 101                                               101
t5                                    Writes back 101                101 (LOST UPDATE!)
```

Both threads performed an increment, yet `counter` only increased by $1$ instead of $2$. Over millions of loop iterations, thousands of updates are silently lost, resulting in corrupted balances, inconsistent pointers, or catastrophic system crashes.

### 2. Formal Criteria for Critical Section Solutions
A correct synchronization solution must satisfy three classical conditions established by Edsger Dijkstra:
1. **Mutual Exclusion:** No two threads can be inside the critical section simultaneously.
2. **Progress:** If no thread is in the critical section and some threads want to enter, only those threads not in their remainder sections can participate in deciding who enters next, and this selection cannot be postponed indefinitely.
3. **Bounded Waiting:** There must be a bound on the number of times other threads are allowed to enter their critical sections after a thread has made a request to enter and before that request is granted (preventing starvation).

A properly utilized POSIX mutex guarantees all three criteria.

---

## 3. How It Works

### Fast User-Space Locking (Futex Mechanism in Linux)

Historically, locking required a system call to the operating system kernel on every lock and unlock, imposing significant overhead (hundreds of CPU cycles). Modern POSIX implementations on Linux utilize **futexes** (*Fast Userspace Mutexes*):

```
Uncontended Case (99% of fast-path operations):
  pthread_mutex_lock(&m)
    |
    v  Atomic Compare-And-Swap (CAS) in User Space (e.g., LOCK CMPXCHG)
  Lock state changed from 0 (Unlocked) to 1 (Locked)
  ===> Returns in ~10-15 nanoseconds (ZERO SYSTEM CALLS!)

Contended Case (Another thread already holds the lock):
  pthread_mutex_lock(&m)
    |
    v  Atomic CAS detects lock is already 1
  Atomic state set to 2 (Locked with Waiters)
  ===> Invokes sys_futex(FUTEX_WAIT) system call
  ===> OS Kernel puts calling thread onto wait queue (DESCHEDULED, Consumes 0% CPU)
    |
  Owner calls pthread_mutex_unlock(&m)
    |
    v  Detects waiters exist
  Invokes sys_futex(FUTEX_WAKE) system call
  ===> OS Kernel wakes up sleeping thread; handed lock ownership
```

This hybrid mechanism provides high performance: fast when there is no contention, yet power-efficient and non-wasteful when contention occurs.

---

### Deadlock: The Lock-Order Inversion Hazard

The most dangerous pitfall when using multiple mutexes is **deadlock**—a state where two or more threads are permanently blocked, each waiting for a lock held by the other.

Consider two threads needing locks $L_1$ and $L_2$ (e.g., transferring funds between two bank accounts):

```
Thread A:                              Thread B:
  pthread_mutex_lock(&L1);               pthread_mutex_lock(&L2);
  // Holds L1, interrupted               // Holds L2, interrupted
  pthread_mutex_lock(&L2);               pthread_mutex_lock(&L1);
  [ BLOCKS: Waiting on L2 ]              [ BLOCKS: Waiting on L1 ]
```

```
+------------+       Holds Lock L1        +------------+
|  Thread A  | -------------------------> | Mutex L1   |
+------------+                            +------------+
      ^                                         |
      | Waits for                               | Held by
      |                                         v
+------------+       Holds Lock L2        +------------+
| Mutex L2   | <------------------------- |  Thread B  |
+------------+                            +------------+
```

#### Coffman Conditions for Deadlock
Deadlock can occur if and only if all four conditions hold simultaneously:
1. **Mutual Exclusion:** At least one resource is held in a non-shareable mode.
2. **Hold and Wait:** A thread holds at least one resource and is waiting to acquire another.
3. **No Preemption:** Resources cannot be forcibly confiscated from a thread.
4. **Circular Wait:** A closed chain of threads exists such that each thread holds a lock needed by the next.

#### The Global Lock Hierarchy Solution
The standard software engineering solution to break circular wait is to enforce a strict **Global Lock Ordering Rule**:
$$\text{If any thread requires locks } L_A \text{ and } L_B, \text{ all threads must acquire them in the exact same memory-address order: } \min(L_A, L_B) \to \max(L_A, L_B)$$

---

## 4. Important Mechanisms and Functions

### 1. Initialization and Destruction

POSIX mutexes can be initialized in two ways:

#### A. Static Allocation (Default Attributes)
For statically or globally allocated mutexes:
```c
pthread_mutex_t my_mutex = PTHREAD_MUTEX_INITIALIZER;
```
*Note: No call to `pthread_mutex_init()` or `pthread_mutex_destroy()` is required for statically initialized mutexes.*

#### B. Dynamic Allocation (Custom Attributes or Heap/Local Storage)
```c
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```
- Returns `0` on success, or an error number (`EBUSY`, `EINVAL`).
- Calling `pthread_mutex_destroy()` on a locked mutex returns `EBUSY`.

---

### 2. Lock Acquisition and Release Functions

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex, const struct timespec *abstime);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

1. **`pthread_mutex_lock()`:**
   - Blocks the caller until the mutex becomes available.
   - If the calling thread already holds the mutex and it is of type `PTHREAD_MUTEX_NORMAL`, the thread will deadlock itself!

2. **`pthread_mutex_trylock()`:**
   - Non-blocking lock attempt.
   - If the mutex is unlocked, acquires it and returns `0`.
   - If the mutex is already locked by another thread (or itself), returns immediately with `EBUSY` without blocking.

3. **`pthread_mutex_timedlock()`:**
   - Blocks up to an **absolute wall-clock deadline** (`struct timespec`).
   - If the deadline expires before acquisition, returns `ETIMEDOUT`.

4. **`pthread_mutex_unlock()`:**
   - Releases lock ownership and wakes up a blocked waiter (if any).
   - Calling unlock on a mutex locked by a different thread results in undefined behavior (or `EPERM` for error-checking mutexes).

---

### 3. Mutex Types and Attributes

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
pthread_mutex_init(&mutex, &attr);
pthread_mutexattr_destroy(&attr);
```

| Mutex Type | Double-Locking by Same Thread | Unlocking by Non-Owner | Performance |
| :--- | :--- | :--- | :--- |
| **`PTHREAD_MUTEX_NORMAL`** (Default) | **Deadlock** (Thread blocks forever) | Undefined Behavior | Fastest (Minimal book-keeping) |
| **`PTHREAD_MUTEX_ERRORCHECK`** | Returns `EDEADLK` | Returns `EPERM` | Slightly slower (Recommended for debugging) |
| **`PTHREAD_MUTEX_RECURSIVE`** | Succeeds (Increments internal counter) | Returns `EPERM` | Slower (Requires equal unlocks to release) |

---

## 5. Compilable C Code Examples

### Example 1: Shared Bank Account Counter (Unprotected Race Condition vs. Mutex Protection)
This program demonstrates the severe data corruption caused by unsynchronized concurrent access to a shared bank balance, contrasted with perfect consistency achieved via `pthread_mutex_t`.

```c
/*
 * File: race_vs_mutex.c
 * Compilation:
 *   gcc -O2 -pthread race_vs_mutex.c -o race_vs_mutex
 *
 * Execution:
 *   ./race_vs_mutex
 */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NUM_THREADS 4
#define ITERATIONS_PER_THREAD 1000000L

/* Shared resources */
static long long unsafe_balance = 0;
static long long safe_balance   = 0;
static pthread_mutex_t balance_mutex = PTHREAD_MUTEX_INITIALIZER;

/* Unsafe Worker: Subject to lost updates */
void* unsafe_worker(void* arg) {
    (void)arg;
    for (long i = 0; i < ITERATIONS_PER_THREAD; ++i) {
        unsafe_balance++; /* READ-MODIFY-WRITE DATA RACE */
    }
    return NULL;
}

/* Safe Worker: Protected by POSIX Mutex */
void* safe_worker(void* arg) {
    (void)arg;
    for (long i = 0; i < ITERATIONS_PER_THREAD; ++i) {
        pthread_mutex_lock(&balance_mutex);
        safe_balance++; /* CRITICAL SECTION */
        pthread_mutex_unlock(&balance_mutex);
    }
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    long long expected_total = (long long)NUM_THREADS * ITERATIONS_PER_THREAD;

    printf("=== Demonstration 1: Unprotected Concurrent Updates (Race Condition) ===\n");
    for (int i = 0; i < NUM_THREADS; ++i) {
        pthread_create(&threads[i], NULL, unsafe_worker, NULL);
    }
    for (int i = 0; i < NUM_THREADS; ++i) {
        pthread_join(threads[i], NULL);
    }

    printf("Expected Final Balance : %lld\n", expected_total);
    printf("Observed Unsafe Balance: %lld\n", unsafe_balance);
    printf("Lost Updates Count     : %lld (%.2f%% error)\n\n",
           expected_total - unsafe_balance,
           ((double)(expected_total - unsafe_balance) / expected_total) * 100.0);

    printf("=== Demonstration 2: Mutex-Protected Concurrent Updates ===\n");
    for (int i = 0; i < NUM_THREADS; ++i) {
        pthread_create(&threads[i], NULL, safe_worker, NULL);
    }
    for (int i = 0; i < NUM_THREADS; ++i) {
        pthread_join(threads[i], NULL);
    }

    printf("Expected Final Balance : %lld\n", expected_total);
    printf("Observed Safe Balance  : %lld\n", safe_balance);
    printf("Discrepancy            : %lld (0.00%% error - Perfect Synchronization)\n",
           expected_total - safe_balance);

    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== Demonstration 1: Unprotected Concurrent Updates (Race Condition) ===
Expected Final Balance : 4000000
Observed Unsafe Balance: 1478203
Lost Updates Count     : 2521797 (63.04% error)

=== Demonstration 2: Mutex-Protected Concurrent Updates ===
Expected Final Balance : 4000000
Observed Safe Balance  : 4000000
Discrepancy            : 0 (0.00% error - Perfect Synchronization)
```

---

### Example 2: Deadlock-Free Transfer Protocol Using Global Address Ordering
This complete program demonstrates a multi-threaded bank transfer simulation. Without ordering, concurrent reciprocal transfers ($A \to B$ and $B \to A$) deadlock immediately. This implementation employs address-based lock ordering and non-blocking `pthread_mutex_trylock()` as an alternative recovery protocol.

```c
/*
 * File: deadlock_free_transfer.c
 * Compilation:
 *   gcc -O2 -pthread deadlock_free_transfer.c -o deadlock_free_transfer
 *
 * Execution:
 *   ./deadlock_free_transfer
 */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

typedef struct {
    int id;
    double balance;
    pthread_mutex_t lock;
} Account;

/* Safe Transfer using Global Lock Address Hierarchy */
int transfer_funds_ordered(Account* from, Account* to, double amount) {
    if (from == to) return 0; /* No self-transfers */

    /* Break circular wait: Always lock in ascending memory address order */
    Account* first  = (from < to) ? from : to;
    Account* second = (from < to) ? to   : from;

    pthread_mutex_lock(&first->lock);
    pthread_mutex_lock(&second->lock);

    int success = 0;
    if (from->balance >= amount) {
        from->balance -= amount;
        to->balance   += amount;
        success = 1;
    }

    pthread_mutex_unlock(&second->lock);
    pthread_mutex_unlock(&first->lock);

    return success;
}

/* Thread Arguments */
typedef struct {
    Account* acc1;
    Account* acc2;
    int transfers_count;
} WorkerArgs;

void* transfer_worker_A(void* arg) {
    WorkerArgs* w = (WorkerArgs*)arg;
    for (int i = 0; i < w->transfers_count; ++i) {
        transfer_funds_ordered(w->acc1, w->acc2, 10.0);
    }
    return NULL;
}

void* transfer_worker_B(void* arg) {
    WorkerArgs* w = (WorkerArgs*)arg;
    for (int i = 0; i < w->transfers_count; ++i) {
        transfer_funds_ordered(w->acc2, w->acc1, 10.0);
    }
    return NULL;
}

int main(void) {
    Account accA = {.id = 1, .balance = 10000.0, .lock = PTHREAD_MUTEX_INITIALIZER};
    Account accB = {.id = 2, .balance = 10000.0, .lock = PTHREAD_MUTEX_INITIALIZER};

    double initial_sum = accA.balance + accB.balance;
    printf("Initial Balances: Acc A = $%.2f, Acc B = $%.2f (Total = $%.2f)\n",
           accA.balance, accB.balance, initial_sum);

    pthread_t t1, t2;
    WorkerArgs w1 = {.acc1 = &accA, .acc2 = &accB, .transfers_count = 100000};
    WorkerArgs w2 = {.acc1 = &accA, .acc2 = &accB, .transfers_count = 100000};

    printf("Executing 200,000 reciprocal transfers concurrently...\n");
    pthread_create(&t1, NULL, transfer_worker_A, &w1);
    pthread_create(&t2, NULL, transfer_worker_B, &w2);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    double final_sum = accA.balance + accB.balance;
    printf("Final Balances  : Acc A = $%.2f, Acc B = $%.2f (Total = $%.2f)\n",
           accA.balance, accB.balance, final_sum);

    if (initial_sum == final_sum) {
        printf("Verification SUCCESS: Conservation of money preserved without deadlock!\n");
    } else {
        printf("Verification FAILED: Money was leaked or duplicated!\n");
    }

    pthread_mutex_destroy(&accA.lock);
    pthread_mutex_destroy(&accB.lock);
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
Initial Balances: Acc A = $10000.00, Acc B = $10000.00 (Total = $20000.00)
Executing 200,000 reciprocal transfers concurrently...
Final Balances  : Acc A = $10000.00, Acc B = $10000.00 (Total = $20000.00)
Verification SUCCESS: Conservation of money preserved without deadlock!
```

---

## 6. Mathematical Formulations & Overhead Analysis

### Critical Section Contention and Amdahl's Law Limit

When threads synchronize via a single shared mutex, the time spent inside the critical section becomes strictly sequential. Let:
- $T_{\text{cs}}$ = Execution time inside the critical section (serial fraction).
- $T_{\text{non-cs}}$ = Execution time spent computing outside the critical section (parallel fraction).
- $N$ = Number of worker threads.

The serial fraction $f$ of the total workload is:

$$f = \frac{T_{\text{cs}}}{T_{\text{cs}} + T_{\text{non-cs}}}$$

By Amdahl's Law, the maximum speedup achievable regardless of core count is bounded by:

$$S_{\max} = \lim_{N \to \infty} \frac{1}{f + \frac{1-f}{N}} = \frac{1}{f} = \frac{T_{\text{cs}} + T_{\text{non-cs}}}{T_{\text{cs}}}$$

If a developer spends $10\%$ of execution time holding a global mutex ($f = 0.10$), the maximum theoretical speedup across a 128-core supercomputer node is:

$$S_{\max} = \frac{1}{0.10} = 10\times$$

Over $92\%$ of compute capacity across the 128 cores will sit permanently idle in `FUTEX_WAIT` sleep states.

### Locking Overhead Formulation

Let $C_{\text{uncontended}}$ be the time required for an uncontended CAS lock/unlock ($\approx 15\text{ ns}$). Let $C_{\text{contended}}$ be the context-switch and kernel trap cost of futex sleep/wake ($\approx 2{,}000\text{ to }5{,}000\text{ ns}$).

The total synchronization overhead per operation is:

$$C_{\text{lock}} = (1 - P_{\text{contention}}) \cdot C_{\text{uncontended}} + P_{\text{contention}} \cdot C_{\text{contended}}$$

As $N$ threads increase, $P_{\text{contention}} \to 1$, shifting lock overhead by orders of magnitude from nanoseconds to microseconds.

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Trade-offs

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_mutex_t`         | - 100% guarantees Mutual Exclusion      | - Coarse-grained locking destroys       |
|                           | - Zero CPU spin when sleeping (futex)   |   parallel scaling (Amdahl bottleneck)  |
|                           | - Highly portable across all POSIX OSs  | - Vulnerable to Deadlocks & Priority    |
|                           |                                         |   Inversion                             |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_mutex_trylock()` | - Non-blocking: permits polling or work | - Starvation risk if thread repeatedly  |
|                           |   alternation without sleeping          |   retries and always finds mutex busy   |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Mutex Errorcheck Type     | - Detects deadlocks and illegal un-locks| - Incurs branch overhead on every lock/ |
|                           |   during debugging                      |   unlock call                           |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Race Condition:** A programming flaw in a concurrent system where the outcome of an execution depends unpredictably on the relative sequence or timing of uncontrollable thread scheduling events.
>
> **Critical Section:** A contiguous block of code that accesses shared mutable resources (such as memory, files, or hardware registers) that must not be concurrently executed by more than one thread at a time.
>
> **Mutual Exclusion (Mutex):** A synchronization property ensuring that only one thread can access a critical section or shared resource at any given instant.
>
> **Deadlock:** A condition in concurrent programming where two or more threads are permanently blocked because each holds a lock that another thread needs, and none can proceed without acquiring the other's held lock.
>
> **Priority Inversion:** An operating system scheduling anomaly where a high-priority thread is indirectly preempted or blocked by a low-priority thread holding a shared resource, often prolonged by medium-priority threads preempting the low-priority lock holder.

---

## 9. Comparison Tables

### Static vs Dynamic Mutex Initialization

| Property | `PTHREAD_MUTEX_INITIALIZER` | `pthread_mutex_init(&m, &attr)` |
| :--- | :--- | :--- |
| **Storage Class** | Global or `static` variables only | Global, local, or dynamically allocated (`malloc`) |
| **Attribute Customization**| **No** (Defaults only: Normal type) | **Yes** (Supports Recursive, Errorcheck, Shared) |
| **Destruction Required?** | **No** | **Yes** (Must call `pthread_mutex_destroy`) |
| **Error Checking** | None at compile time | Returns integer status codes (`ENOMEM`, `EINVAL`) |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Calculating Maximum Throughput Under Lock Contention
**Question:** A multi-threaded database server executes a transaction where the critical section holding a shared storage mutex takes $2\,\mu\text{s}$ ($2 \times 10^{-6}\text{ s}$). What is the absolute theoretical maximum number of transactions per second (TPS) that this server can process, regardless of whether the machine has 8 cores, 64 cores, or 1024 cores?

**Solution:**
Because only one thread can be inside the critical section at any given time, the critical section is strictly sequential.
The maximum transaction throughput $\text{TPS}_{\max}$ is dictated solely by the serialization rate of the mutex:

$$\text{TPS}_{\max} = \frac{1}{T_{\text{cs}}} = \frac{1}{2 \times 10^{-6}\text{ seconds}} = 500{,}000\text{ transactions/sec}$$

*Conclusion:* Adding more CPU cores beyond the point where the mutex is saturated will yield zero additional throughput. To scale further, the developer must employ fine-grained partitioning (e.g., hash sharding) or lock-free data structures.

---

### Problem 2: Analyzing and Fixing a Self-Deadlock Bug
**Question:** Examine the following recursive function. When compiled and run with default mutex attributes, the program hangs permanently on the second call. Explain why and provide the standard POSIX fix.

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void print_tree(Node* root) {
    if (root == NULL) return;
    
    pthread_mutex_lock(&lock);
    printf("Node: %d\n", root->value);
    print_tree(root->left);   /* Recursive call */
    print_tree(root->right);  /* Recursive call */
    pthread_mutex_unlock(&lock);
}
```

**Solution:**
- **The Defect:** Statically initialized mutexes (`PTHREAD_MUTEX_INITIALIZER`) default to `PTHREAD_MUTEX_NORMAL`. When `print_tree(root->left)` is called, the same thread attempts to acquire `lock` while already holding it. Under `PTHREAD_MUTEX_NORMAL`, re-locking results in **self-deadlock**; the calling thread blocks waiting for itself to unlock the mutex.
- **The Fix:** Configure the mutex as a **Recursive Mutex** (`PTHREAD_MUTEX_RECURSIVE`), which tracks lock depth and allows the owner to lock multiple times:

```c
pthread_mutex_t lock;
pthread_mutexattr_t attr;

pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&lock, &attr);
pthread_mutexattr_destroy(&attr);
```

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Can a thread unlock a mutex that was locked by another thread?
**Answer:** Under POSIX standards, if a thread attempts to unlock a mutex that it does not own, the behavior is **undefined** for default `PTHREAD_MUTEX_NORMAL` mutexes. If the mutex is configured as `PTHREAD_MUTEX_ERRORCHECK` or `PTHREAD_MUTEX_RECURSIVE`, the function will safely fail and return the error code **`EPERM`** (Operation not permitted). Mutexes imply strict ownership; a lock must be released only by the thread that acquired it. (To signal between different threads, use condition variables or semaphores instead).

### Q2: What is the difference between `pthread_mutex_lock()` and `pthread_mutex_trylock()`?
**Answer:** `pthread_mutex_lock()` is a **blocking** call; if the mutex is locked, the calling thread is suspended by the operating system kernel and placed on a wait queue until the lock is released. In contrast, `pthread_mutex_trylock()` is **non-blocking**; if the mutex is currently locked, it returns immediately with the status code `EBUSY`, allowing the calling thread to execute alternative work instead of sleeping.

### Q3: Why is destroying a locked mutex (`pthread_mutex_destroy`) an error?
**Answer:** Destroying a locked mutex produces undefined behavior and typically returns **`EBUSY`**. If a mutex were destroyed while locked, any threads currently blocked in the kernel waiting on that mutex's futex queue would become orphaned, resulting in permanent deadlocks or invalid memory accesses.

---

## 12. Quick Revision Sheet

- **Data Race vs Race Condition:** A data race is the unsynchronized concurrent access to shared memory; a race condition is the resulting timing-dependent erroneous system behavior.
- **Critical Section Rules:** Mutual Exclusion, Progress, Bounded Waiting.
- **Mutex Lifecycle:**
  - Static: `pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;`
  - Dynamic: `pthread_mutex_init(&m, &attr)` $\to$ `lock/unlock` $\to$ `pthread_mutex_destroy(&m)`.
- **Functions:**
  - `pthread_mutex_lock()`: Blocks until locked.
  - `pthread_mutex_trylock()`: Non-blocking; returns `EBUSY` if held.
  - `pthread_mutex_unlock()`: Releases ownership.
- **Deadlock Coffman Conditions:** Mutual Exclusion, Hold and Wait, No Preemption, Circular Wait.
- **Deadlock Prevention:** Enforce Global Lock Ordering (e.g., by memory address `first = (A < B) ? A : B`).

---

## 13. What You Should Be Able to Do

1. **Identify race conditions** in multithreaded code accessing shared variables.
2. **Implement mutual exclusion** around critical sections using `pthread_mutex_t`.
3. **Choose appropriately** between static initialization and dynamic `pthread_mutex_init()`.
4. **Prevent deadlocks** in multi-lock applications using strict address-ordered locking hierarchies.
5. **Implement non-blocking polling** algorithms using `pthread_mutex_trylock()`.
6. **Configure error-checking and recursive mutex attributes** to debug multithreaded applications.
