# Module 09: Pthreads Synchronization Primitives (Lectures 20–23)
# Subtopic 09.04: Read-Write Locks and Read-Heavy Workloads (`pthread_rwlock_t`, Concurrent Readers, Exclusive Writers, Writer Starvation)

---

## 1. Basic Idea

A standard POSIX mutex enforces strict **mutual exclusion**: only one thread may enter the critical section at any given time, regardless of whether that thread intends to modify shared memory or merely read it.

However, in many real-world High-Performance Computing (HPC), database, and server applications, data access patterns are fundamentally asymmetric:
- **Readers** inspect the shared data structure (e.g., querying an in-memory routing table, looking up user authentication tokens, or evaluating neural network weights). Two or more readers accessing data simultaneously cause **no data races** because neither alters the underlying state.
- **Writers** modify, append, or delete records in the shared structure. A writer requires strict, exclusive access to prevent race conditions with readers or other writers.

A **Read-Write Lock** (represented by `pthread_rwlock_t`) exploits this concurrency property:
1. **Shared Read Access (`rdlock`):** Arbitrarily many threads may acquire the lock concurrently for reading, provided no writer holds the lock and no writer is actively prioritized.
2. **Exclusive Write Access (`wrlock`):** Exactly one writer may hold the lock. While held, all other readers and writers are blocked.

```
+-----------------------------------------------------------------------------------------+
|                              Read-Write Lock Concurrency Matrix                         |
+-----------------------------------------------------------------------------------------+
| Current Lock State | Incoming Request | Result                                          |
|:-------------------|:-----------------|:------------------------------------------------|
| Free (Unlocked)    | Reader           | Granted (Lock enters Shared Read mode)          |
| Free (Unlocked)    | Writer           | Granted (Lock enters Exclusive Write mode)      |
| Shared Read (N)    | Reader           | GRANTED (Concurrent access! N -> N+1)           |
| Shared Read (N)    | Writer           | BLOCKED (Must wait until ALL N readers release) |
| Exclusive Write    | Reader or Writer | BLOCKED (Must wait until current writer unlocks)|
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Mutex Serialization Bottleneck in Read-Heavy Workloads
Consider an in-memory cache accessed by 64 threads on a 64-core compute node. Suppose $95\%$ of all operations are lookups (`read`) and only $5\%$ are updates (`write`).

If protected by a standard `pthread_mutex_t`:
- Even though 60 reader threads arrive simultaneously and intend only to inspect identical memory addresses, the mutex forces them to line up sequentially in single file.
- Total lookup throughput is restricted by the serial execution time of each read.
- 63 cores sit permanently throttled or sleeping, destroying parallel scalability.

Under `pthread_rwlock_t`:
- All 60 reader threads acquire the read lock simultaneously.
- All 60 cores execute their lookups in true hardware parallel fashion, achieving near-linear speedup.

### 2. The Writer Starvation Hazard
In a naive read-write lock implementation, if reader threads arrive in a continuous, overlapping stream:
- Reader 1 arrives and holds `rdlock`.
- Reader 2 arrives and is granted `rdlock` concurrently.
- Reader 1 finishes and unlocks, but Reader 3 arrives and is granted `rdlock`.
- A Writer thread arrives and requests `wrlock`, but must wait for the active reader count to reach 0.
- Because new readers continuously arrive before prior readers finish, the active reader count never drops to zero.
- **The Writer suffers indefinite delay (Writer Starvation).**

POSIX and modern Linux implementations resolve this via **Writer-Preference / Fair Queuing Policies**, ensuring that once a writer requests the lock, subsequent arriving readers are queued behind the writer.

---

## 3. How It Works

### Internal State of `pthread_rwlock_t`

A read-write lock maintains an atomic 32-bit counter and internal wait queues:

```
+-----------------------------------------------------------------------------------------+
|                  Internal Bitfield/State of a Read-Write Lock                           |
+-----------------------------------------------------------------------------------------+
| Bits [30..0] : Reader Count (Active concurrent readers: 0, 1, 2, ... N)                |
| Bit  [31]    : Write Lock Flag (1 = Writer actively holds lock, 0 = No writer)          |
| Waiters Count: Number of blocked readers and blocked writers waiting in futex queue     |
+-----------------------------------------------------------------------------------------+
```

```
[ READER ACQUISITION (rdlock) ]
  1. Atomic check: Is Bit 31 (Write Flag) set OR are writers waiting (under writer preference)?
     - If NO : Atomically increment Reader Count by 1. Return immediately! (Fast user-space)
     - If YES: Block on kernel futex wait queue.

[ READER RELEASE (unlock) ]
  1. Atomically decrement Reader Count by 1.
  2. If Reader Count drops to 0 AND writers are waiting:
     - Invoke kernel futex wake to awaken the pending writer.

[ WRITER ACQUISITION (wrlock) ]
  1. Atomic check: Is Reader Count == 0 AND Bit 31 == 0?
     - If YES: Set Bit 31 to 1. Return immediately!
     - If NO : Increment blocked writers counter; sleep on kernel futex.

[ WRITER RELEASE (unlock) ]
  1. Clear Bit 31 to 0.
  2. Wake up waiting readers (in broadcast batch) or the next waiting writer.
```

---

## 4. Important Mechanisms and Functions

### 1. Initialization and Destruction

```c
#include <pthread.h>

/* Static Initialization (Default Attributes) */
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

/* Dynamic Initialization */
int pthread_rwlock_init(pthread_rwlock_t *rwlock, const pthread_rwlockattr_t *attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```
- `pthread_rwlock_init()` returns `0` on success, or an error number.
- `pthread_rwlock_destroy()` releases internal resources; returns `EBUSY` if the lock is held.

---

### 2. Read Lock Acquisition

```c
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_timedrdlock(pthread_rwlock_t *rwlock, const struct timespec *abstime);
```
- **`pthread_rwlock_rdlock()`:** Blocks if a writer holds the lock or if a writer has priority on the queue. Returns `0` on success.
- **`pthread_rwlock_tryrdlock()`:** Non-blocking. If the lock is held by a writer, returns immediately with `EBUSY`.

---

### 3. Write Lock Acquisition

```c
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *rwlock, const struct timespec *abstime);
```
- **`pthread_rwlock_wrlock()`:** Blocks until all active readers have released the lock AND any preceding writers have finished.
- **`pthread_rwlock_trywrlock()`:** Non-blocking. If any reader or writer holds the lock, returns immediately with `EBUSY`.

---

### 4. Release Function

```c
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```
- Unlocks the read-write lock, whether currently held for reading or writing.
- Calling `pthread_rwlock_unlock()` on a lock not owned by the calling thread results in undefined behavior.

---

### 5. Configuring Writer-Preference Attributes (GNU Extension)

Under standard POSIX, the choice of reader vs. writer preference is implementation-defined. In glibc on Linux, writer preference can be explicitly enforced to eliminate writer starvation:

```c
#define _GNU_SOURCE
#include <pthread.h>

pthread_rwlockattr_t attr;
pthread_rwlockattr_init(&attr);

/* Prefer writers: blocks new readers if a writer is waiting */
pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP);

pthread_rwlock_init(&rwlock, &attr);
pthread_rwlockattr_destroy(&attr);
```

---

## 5. Compilable C Code Examples

### Example 1: Read-Heavy Shared Configuration Cache Benchmark (RWLock vs Mutex)
This complete program models a read-heavy key-value cache where 8 Reader threads perform 100,000 lookups each, while 1 Writer thread periodically updates the cache. It demonstrates using `pthread_rwlock_t` and measures execution time.

```c
/*
 * File: rwlock_cache_demo.c
 * Compilation:
 *   gcc -O2 -pthread rwlock_cache_demo.c -o rwlock_cache_demo
 *
 * Execution:
 *   ./rwlock_cache_demo
 */

#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>

#define NUM_READERS 4
#define READS_PER_THREAD 500000L
#define WRITES_COUNT 10

/* Shared Cache Data Structure */
typedef struct {
    int key;
    long long value;
    long long update_timestamp;
    pthread_rwlock_t rwlock;
} SharedCache;

SharedCache cache;

/* Reader Thread: Concurrent Read Access */
void* reader_worker(void* arg) {
    long id = (long)arg;
    long long checksum = 0;

    for (long i = 0; i < READS_PER_THREAD; ++i) {
        pthread_rwlock_rdlock(&cache.rwlock);

        /* Read critical section: Shared by all readers */
        checksum += cache.value;

        pthread_rwlock_unlock(&cache.rwlock);
    }

    printf("[Reader %ld] Completed %ld reads. Final Checksum = %lld\n",
           id, READS_PER_THREAD, checksum);
    return NULL;
}

/* Writer Thread: Exclusive Write Access */
void* writer_worker(void* arg) {
    (void)arg;
    for (int i = 1; i <= WRITES_COUNT; ++i) {
        usleep(15000); /* Write every 15ms */

        pthread_rwlock_wrlock(&cache.rwlock);

        /* Write critical section: Strictly exclusive */
        cache.key = i;
        cache.value = (long long)i * 1000;
        cache.update_timestamp = (long long)time(NULL);

        printf("  >>> [Writer] Updated Cache to Key: %d, Value: %lld <<<\n",
               cache.key, cache.value);

        pthread_rwlock_unlock(&cache.rwlock);
    }
    return NULL;
}

int main(void) {
    pthread_t readers[NUM_READERS];
    pthread_t writer;
    pthread_rwlockattr_t attr;

    printf("=== POSIX Read-Write Lock: Read-Heavy Workload Benchmark ===\n");
    printf("Readers: %d (%ld lookups each) | Writers: 1 (%d updates)\n\n",
           NUM_READERS, READS_PER_THREAD, WRITES_COUNT);

    /* Initialize RWLock with Writer Preference */
    pthread_rwlockattr_init(&attr);
    pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP);
    pthread_rwlock_init(&cache.rwlock, &attr);
    pthread_rwlockattr_destroy(&attr);

    cache.key = 0;
    cache.value = 500;
    cache.update_timestamp = 0;

    struct timespec start_time, end_time;
    clock_gettime(CLOCK_MONOTONIC, &start_time);

    /* Spawn Writer and Readers */
    pthread_create(&writer, NULL, writer_worker, NULL);
    for (long i = 0; i < NUM_READERS; ++i) {
        pthread_create(&readers[i], NULL, reader_worker, (void*)i);
    }

    /* Await completion */
    for (int i = 0; i < NUM_READERS; ++i) {
        pthread_join(readers[i], NULL);
    }
    pthread_join(writer, NULL);

    clock_gettime(CLOCK_MONOTONIC, &end_time);
    double elapsed_sec = (end_time.tv_sec - start_time.tv_sec) +
                         (end_time.tv_nsec - start_time.tv_nsec) / 1e9;

    printf("\n=== Benchmark Completed in %.3f seconds ===\n", elapsed_sec);
    printf("Total Read Operations : %ld\n", (long)NUM_READERS * READS_PER_THREAD);
    printf("Throughput           : %.2f Million Ops/sec\n",
           ((double)(NUM_READERS * READS_PER_THREAD) / elapsed_sec) / 1e6);

    pthread_rwlock_destroy(&cache.rwlock);
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== POSIX Read-Write Lock: Read-Heavy Workload Benchmark ===
Readers: 4 (500000 lookups each) | Writers: 1 (10 updates)

  >>> [Writer] Updated Cache to Key: 1, Value: 1000 <<<
  >>> [Writer] Updated Cache to Key: 2, Value: 2000 <<<
  >>> [Writer] Updated Cache to Key: 3, Value: 3000 <<<
[Reader 0] Completed 500000 reads. Final Checksum = 1245800000
[Reader 2] Completed 500000 reads. Final Checksum = 1245800000
[Reader 1] Completed 500000 reads. Final Checksum = 1245800000
[Reader 3] Completed 500000 reads. Final Checksum = 1245800000
  >>> [Writer] Updated Cache to Key: 10, Value: 10000 <<<

=== Benchmark Completed in 0.185 seconds ===
Total Read Operations : 2000000
Throughput           : 10.81 Million Ops/sec
```

---

## 6. Mathematical Formulations & Overhead Analysis

### RWLock Speedup Model vs Mutex

Let $R$ be the proportion of read operations ($0 \le R \le 1$), and $W = 1 - R$ be the proportion of write operations. Let:
- $N$ = Number of concurrent threads (cores).
- $T_r$ = Duration of a read critical section.
- $T_w$ = Duration of a write critical section.

#### Mutex Execution Time
Under a mutex, all operations are completely serialized:

$$T_{\text{mutex}} = M \cdot \left[ R \cdot T_r + (1 - R) \cdot T_w \right]$$

Where $M$ is the total number of operations.

#### RWLock Execution Time
Under a read-write lock, all readers execute concurrently in parallel across $N$ cores, while writers execute sequentially:

$$T_{\text{rwlock}} \approx M \cdot \left[ \frac{R \cdot T_r}{N} + (1 - R) \cdot T_w \right] + T_{\text{overhead}}(N)$$

#### The Crossover Threshold
Read-write locks have higher internal book-keeping overhead ($T_{\text{overhead}} \approx 2\times \text{ to } 3\times$ that of a simple mutex) due to maintaining atomic reader counters and tracking waiting writers.

An RWLock yields higher performance than a mutex if and only if:

$$T_{\text{rwlock}} < T_{\text{mutex}}$$

Substituting and simplifying:

$$\frac{R \cdot T_r}{N} + (1 - R) \cdot T_w + \frac{T_{\text{overhead}}}{M} < R \cdot T_r + (1 - R) \cdot T_w$$

$$R \cdot T_r \left( 1 - \frac{1}{N} \right) > \frac{T_{\text{overhead}}}{M}$$

$$R > \frac{T_{\text{overhead}}}{M \cdot T_r \left( \frac{N - 1}{N} \right)}$$

**Empirical HPC Rule of Thumb:**
- If $R < 80\%$ (more than $20\%$ writes), a standard `pthread_mutex_t` outperforms `pthread_rwlock_t` because the lock acquisition overhead outweighs the concurrency benefits.
- If $R \ge 90\%$ (read-heavy) and $T_r$ involves non-trivial memory inspection, `pthread_rwlock_t` achieves up to an $\mathcal{O}(N)$ throughput improvement.

---

## 7. Advantages, Disadvantages & Limitations

### Summary of Trade-offs

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_rwlock_t`        | - High concurrency for read workloads   | - Higher base latency than mutex        |
|                           | - True multi-core parallel lookups      | - Vulnerable to Writer Starvation       |
|                           | - Prevents unnecessary serialization    |   without explicit preference settings  |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Writer Preference         | - Guarantees timely data updates        | - If writers are frequent, readers      |
| (`PREFER_WRITER`)         | - Completely eliminates writer starving |   suffer severe starvation latency      |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_rwlock_wrlock()` | - Strict exclusive access               | - Cannot recursively lock: self-deadlock|
|                           | - Compatible with any data mutation     |   if same thread calls wrlock twice     |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Read-Write Lock (`pthread_rwlock_t`):** A synchronization primitive that permits concurrent, non-exclusive access to multiple reader threads while enforcing exclusive, single-thread access for writer threads.
>
> **Shared Read Mode (`pthread_rwlock_rdlock`):** A lock acquisition state where multiple threads may concurrently read shared memory without serialization, granted only when no thread holds or is queued to acquire the write lock.
>
> **Exclusive Write Mode (`pthread_rwlock_wrlock`):** A lock acquisition state granting a single thread complete, exclusive access to modify shared memory, blocking all other readers and writers until unlocked.
>
> **Writer Starvation:** A concurrency anomaly occurring in read-preference read-write locks where a continuous stream of overlapping reader threads prevents the active reader count from reaching zero, permanently blocking waiting writer threads.
>
> **Reader-Preference vs Writer-Preference:** A policy distinction determining lock allocation priority; reader-preference maximizes read concurrency at the risk of delaying writes, while writer-preference prioritizes queued writers to guarantee fresh state propagation.

---

## 9. Comparison Tables

### Mutex vs Read-Write Lock

| Feature | `pthread_mutex_t` | `pthread_rwlock_t` |
| :--- | :--- | :--- |
| **Max Concurrent Readers**| **1** (Strictly serialized) | **$\infty$** (Limited only by CPU core count) |
| **Max Concurrent Writers**| **1** | **1** |
| **Acquisition Overhead** | Very Low ($\approx 15\text{ ns}$) | Medium ($\approx 35\text{ to }50\text{ ns}$) |
| **Optimal Read Ratio** | Balanced workloads ($R < 80\%$) | Read-heavy workloads ($R \ge 90\%$) |
| **Deadlock Risk** | Re-locking non-recursive lock | Calling `wrlock()` while holding `rdlock` |
| **Memory Footprint** | 40 bytes (on x86_64) | 56 bytes (on x86_64) |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Lock Upgrade Self-Deadlock
**Question:** Analyze the following code snippet. Explain why the thread deadlocks itself and how to implement the upgrade correctly.

```c
/* ATTEMPTED CACHE LOOKUP AND UPDATE */
pthread_rwlock_rdlock(&cache_lock);
if (lookup(key) == NOT_FOUND) {
    /* Attempting to upgrade lock to write */
    pthread_rwlock_wrlock(&cache_lock); /* HANGS PERMANENTLY! */
    insert(key, value);
    pthread_rwlock_unlock(&cache_lock);
}
pthread_rwlock_unlock(&cache_lock);
```

**Solution:**
- **The Defect:** POSIX read-write locks do **not** support atomic lock upgrades. When `pthread_rwlock_wrlock()` is called, the calling thread blocks waiting for all readers to release the lock. However, the calling thread *itself* is currently holding one read lock! Because the thread is suspended inside `wrlock()`, it can never execute the subsequent `unlock()` to release its own read lock.
- **Result:** **Self-Deadlock.**
- **The Correct Pattern (Release, Reacquire Write, Re-check):**

```c
pthread_rwlock_rdlock(&cache_lock);
if (lookup(key) == NOT_FOUND) {
    /* 1. Must release read lock FIRST */
    pthread_rwlock_unlock(&cache_lock);

    /* 2. Acquire write lock exclusively */
    pthread_rwlock_wrlock(&cache_lock);

    /* 3. RE-CHECK: Another thread may have inserted key while lock was released! */
    if (lookup(key) == NOT_FOUND) {
        insert(key, value);
    }
    pthread_rwlock_unlock(&cache_lock);
} else {
    pthread_rwlock_unlock(&cache_lock);
}
```

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: When is a mutex faster than a read-write lock?
**Answer:** A mutex is faster than a read-write lock under two primary conditions:
1. **Write-heavy or balanced workloads:** When writes constitute more than $15\text{–}20\%$ of all accesses.
2. **Very short critical sections:** If the work inside the critical section takes only a few CPU cycles (e.g., reading an integer flag), the higher internal CPU instruction overhead of updating atomic reader counters and checking writer wait queues in `pthread_rwlock_rdlock` exceeds the cost of a simple mutex lock.

### Q2: Can a thread hold both a read lock and a write lock on the same `pthread_rwlock_t` simultaneously?
**Answer:** No. POSIX explicitly specifies that attempting to acquire a write lock while holding a read lock, or attempting to acquire a read lock while holding a write lock, produces **undefined behavior** or **deadlock**. 

### Q3: How does `PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP` prevent writer starvation?
**Answer:** When this attribute is set, if a writer calls `pthread_rwlock_wrlock()` while readers are currently holding the lock, the lock flags that a writer is waiting. Any **subsequent** reader that arrives and calls `pthread_rwlock_rdlock()` is immediately blocked and enqueued behind the writer, even though the active readers have not yet finished. Once the existing active readers finish, the waiting writer executes immediately, preventing readers from infinitely monopolizing the lock.

---

## 12. Quick Revision Sheet

- **Read-Write Lock:** Multiple concurrent readers (`rdlock`), exactly one exclusive writer (`wrlock`).
- **Best Use Case:** Read-heavy workloads ($R \ge 90\%$) with non-trivial read duration.
- **Writer Starvation:** Caused when infinite overlapping readers prevent writers from ever locking. Solved by writer-preference attributes.
- **Lock Upgrades:** Calling `wrlock()` while holding `rdlock` causes **self-deadlock**. Always unlock `rdlock` first, acquire `wrlock`, and re-check state.
- **Functions:**
  - `pthread_rwlock_rdlock()`: Shared read lock.
  - `pthread_rwlock_wrlock()`: Exclusive write lock.
  - `pthread_rwlock_unlock()`: Releases either read or write lock.
  - `pthread_rwlock_destroy()`: Destroys lock (returns `EBUSY` if held).

---

## 13. What You Should Be Able to Do

1. **Implement concurrent shared lookups** and exclusive updates using `pthread_rwlock_t`.
2. **Calculate the theoretical speedup and break-even point** comparing read-write locks to mutexes based on read/write ratios.
3. **Diagnose and eliminate writer starvation** using writer-preference configuration attributes.
4. **Identify and correct lock-upgrade deadlocks** in concurrent caching algorithms.
5. **Select appropriately** between a mutex and a read-write lock based on profiling data.
