# Module 09: Pthreads Synchronization Primitives (Lectures 20–23)
# Subtopic 09.02: Condition Variables and Coordination (`pthread_cond_t`, `wait`, `signal`, `broadcast`, Spurious Wakeups)

---

## 1. Basic Idea

A mutex enables **mutual exclusion**—ensuring that only one thread accesses a critical section at a time. However, in many concurrent systems, a thread must wait until a specific application state or logical condition becomes true (e.g., waiting for a task queue to become non-empty, waiting for a buffer to empty, or waiting for all worker threads to reach a computational checkpoint).

Attempting to wait using a mutex alone leads to **busy-waiting (polling)**:
```c
/* DISASTROUS POLLING PATTERN: BURNS 100% CPU */
pthread_mutex_lock(&lock);
while (queue_empty) {
    pthread_mutex_unlock(&lock);
    // Sleep briefly or spin
    pthread_mutex_lock(&lock);
}
// Process queue...
pthread_mutex_unlock(&lock);
```
Busy-waiting wastes billions of CPU cycles, creates heavy lock contention, and increases power consumption.

A **Condition Variable**, represented by `pthread_cond_t`, provides an event-driven notification mechanism that allows a thread to **sleep without consuming CPU cycles** until another thread modifies the shared state and signals the condition variable.

```
+-----------------------------------------------------------------------------------------+
|                              Condition Variable Mechanism                               |
+-----------------------------------------------------------------------------------------+
| Waiting Thread:                                                                         |
|   1. Locks associated mutex.                                                            |
|   2. Checks state predicate: while (!work_available)                                    |
|   3. Calls pthread_cond_wait(&cv, &mutex)                                               |
|      ===> ATOMICALLY releases mutex AND puts thread to sleep in kernel wait-queue.      |
|                                                                                         |
| Signaling Thread:                                                                       |
|   1. Locks associated mutex.                                                            |
|   2. Modifies shared state: work_available = 1                                           |
|   3. Calls pthread_cond_signal(&cv) (or broadcast)                                      |
|   4. Unlocks associated mutex.                                                          |
|                                                                                         |
| Awakened Thread:                                                                        |
|   4. Wakes up, RE-ACQUIRES the mutex automatically before pthread_cond_wait returns.    |
|   5. Re-evaluates predicate loop. If true, proceeds into critical section.              |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. Eliminating CPU Waste and Energy Drain
When a thread calls `pthread_cond_wait()`, the Linux kernel transitions the thread from `TASK_RUNNING` to `TASK_INTERRUPTIBLE` (sleeping on a futex). While sleeping, the thread consumes **$0.0\%$ CPU time** and yields the core to other processes or allows the processor to enter low-power C-states.

### 2. The Lost Wakeup Problem (Why Mutex Pairing is Mandatory)
A condition variable has **no memory**. It does not maintain a counter of signals sent in the past. If a thread signals a condition variable when no thread is actively blocked inside `pthread_cond_wait()`, that signal is **silently dropped and lost forever**.

Consider what would happen if condition variables operated without a paired mutex:

```
Thread A (Consumer)                            Thread B (Producer)
-------------------                            -------------------
1. Checks condition: queue is empty!
                                               2. Adds item to queue.
                                               3. Signals condition variable!
                                                  (Signal lost: Nobody waiting!)
4. Calls cond_wait()!
   ===> Thread A SLEEPS FOREVER!
```

Because Step 1 and Step 4 are separated by Step 2 and 3, a race window occurs. By requiring that both threads hold the **same mutex**, the check of the condition and the transition into sleep become strictly serialized:
- The consumer checks the predicate and calls `pthread_cond_wait()` under the lock.
- The producer cannot modify the predicate or issue a signal until the consumer has safely registered itself on the condition variable's wait queue and released the lock.

### 3. Spurious Wakeups
A thread sleeping in `pthread_cond_wait()` may occasionally wake up **even though no thread signaled the condition variable**. This occurs due to:
1. Low-level OS kernel signal delivery (e.g., `SIGCHLD`, `SIGINT`).
2. Multi-core kernel futex implementations where an internal race or context switch triggers a return.
3. Multiple consumers waking up simultaneously where the first consumer consumes the resource, leaving subsequent awakened consumers with nothing.

Therefore, POSIX explicitly mandates: **A condition variable wait must ALWAYS be enclosed inside a `while` loop, never an `if` statement!**

---

## 3. How It Works

### The Atomic Triad of `pthread_cond_wait`

The defining technical feature of `pthread_cond_wait(&cv, &mutex)` is its three-phase atomic transition:

```
[ PHASE 1: ATOMIC UNLOCK & ENQUEUE ]
  - Registers the calling thread on the condition variable's wait queue.
  - Releases the mutex (lock count goes to 0).
  - CRITICAL: These two actions happen atomically within the kernel!
  - Thread enters deep sleep (futex sleep).

                 ... Time passes while thread is asleep ...
                 (Signaling thread modifies state and signals)

[ PHASE 2: WAKEUP NOTIFICATION ]
  - Kernel moves thread from CV wait-queue to the Mutex acquisition queue.

[ PHASE 3: MUTEX RE-ACQUISITION ]
  - Thread wakes up and contends for the mutex just like any thread calling pthread_mutex_lock().
  - pthread_cond_wait() DOES NOT RETURN until the mutex is successfully re-acquired!
  - Once the lock is held, pthread_cond_wait() returns control to user code.
```

```
+-----------------------------------------------------------------------------------------+
|                  The Complete Wait-in-a-While-Loop Pattern                              |
+-----------------------------------------------------------------------------------------+
|   pthread_mutex_lock(&mutex);                                                           |
|                                                                                         |
|   while (condition_is_false) {          <--- MANDATORY: while loop, NOT if!             |
|       pthread_cond_wait(&cv, &mutex);   <--- Releases mutex, sleeps, re-locks           |
|   }                                                                                     |
|                                                                                         |
|   // Now we hold the mutex AND condition_is_false is guaranteed to be FALSE             |
|   perform_work();                                                                       |
|                                                                                         |
|   pthread_mutex_unlock(&mutex);                                                         |
+-----------------------------------------------------------------------------------------+
```

---

### `pthread_cond_signal()` vs. `pthread_cond_broadcast()`

POSIX provides two functions to wake sleeping threads:

```
Signal:   pthread_cond_signal(&cv)    ---> Wakes EXACTLY ONE waiting thread.
Broadcast: pthread_cond_broadcast(&cv) ---> Wakes ALL waiting threads simultaneously.
```

```
           pthread_cond_signal(&cv)
                     |
                     v
   Wait Queue: [ Thread 1 ]  [ Thread 2 ]  [ Thread 3 ]
                     |
                     +---> Thread 1 woken up (Thread 2 & 3 stay asleep)


           pthread_cond_broadcast(&cv)
                     |
                     v
   Wait Queue: [ Thread 1 ]  [ Thread 2 ]  [ Thread 3 ]
                     |             |             |
                     v             v             v
   ALL threads woken! Each contends sequentially for the single mutex.
```

- Use **`pthread_cond_signal()`** when:
  - Exactly one thread can make progress on the newly changed state (e.g., adding one item to a queue where one consumer will pop it).
  - Avoids the "thundering herd" problem where $N$ threads wake up, but only one acquires the lock while $N-1$ threads immediately go back to sleep.
- Use **`pthread_cond_broadcast()`** when:
  - The state change affects all threads (e.g., "all calculation phases complete", "shutdown initiated", or a custom barrier reached).
  - Multiple threads can make progress (e.g., adding multiple resources to a pool).

---

## 4. Important Mechanisms and Functions

### 1. Initialization and Destruction

```c
#include <pthread.h>

/* Static Initialization */
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;

/* Dynamic Initialization */
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);
int pthread_cond_destroy(pthread_cond_t *cond);
```
- `pthread_cond_init()`: Returns `0` on success, or an error code.
- `pthread_cond_destroy()`: Destroys the condition variable. If threads are currently blocked on the variable, it returns `EBUSY`.

---

### 2. Waiting Functions

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
                           const struct timespec *abstime);
```

#### Understanding `abstime` in `pthread_cond_timedwait()`
The deadline argument `abstime` is an **absolute wall-clock time**, not a relative duration! To wait for 2.5 seconds:

```c
struct timespec deadline;
clock_gettime(CLOCK_REALTIME, &deadline);
deadline.tv_sec  += 2;              // Add 2 seconds
deadline.tv_nsec += 500000000L;      // Add 500 million nanoseconds (0.5s)
if (deadline.tv_nsec >= 1000000000L) {
    deadline.tv_sec  += 1;
    deadline.tv_nsec -= 1000000000L;
}

int rc = pthread_cond_timedwait(&cv, &mutex, &deadline);
if (rc == ETIMEDOUT) {
    // Timed out before condition was signaled
}
```

---

### 3. Signaling Functions

```c
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```
- Both functions return `0` on success.
- If no threads are waiting on `cond`, the function succeeds and the signal is silently discarded.

---

## 5. Compilable C Code Examples

### Example 1: Multi-Thread Worker Coordination Barrier with Condition Variables
This complete program coordinates a team of worker threads that perform Phase 1 computations, sleep on a condition variable until all workers arrive at the checkpoint, and are then simultaneously released via `pthread_cond_broadcast()` to execute Phase 2.

```c
/*
 * File: condvar_barrier.c
 * Compilation:
 *   gcc -O2 -pthread condvar_barrier.c -o condvar_barrier
 *
 * Execution:
 *   ./condvar_barrier
 */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define NUM_WORKERS 4

/* Custom Barrier State using Condition Variable and Mutex */
typedef struct {
    pthread_mutex_t lock;
    pthread_cond_t  cv;
    int threshold;
    int arrived_count;
    int phase;
} CustomBarrier;

CustomBarrier barrier;

void barrier_init(CustomBarrier* b, int count) {
    pthread_mutex_init(&b->lock, NULL);
    pthread_cond_init(&b->cv, NULL);
    b->threshold     = count;
    b->arrived_count = 0;
    b->phase         = 0;
}

void barrier_wait(CustomBarrier* b, int worker_id) {
    pthread_mutex_lock(&b->lock);

    int current_phase = b->phase;
    b->arrived_count++;

    if (b->arrived_count == b->threshold) {
        /* Last arriving thread flips phase and releases all waiting workers */
        printf("[Worker %d] Reached barrier (LAST THREAD). Broadcasting wakeup!\n", worker_id);
        b->arrived_count = 0;
        b->phase++;
        pthread_cond_broadcast(&b->cv);
    } else {
        printf("[Worker %d] Reached barrier (%d/%d arrived). Sleeping...\n",
               worker_id, b->arrived_count, b->threshold);

        /* Wait in a while loop checking for phase change to guard against spurious wakeups */
        while (b->phase == current_phase) {
            pthread_cond_wait(&b->cv, &b->lock);
        }
        printf("[Worker %d] Woke up! Proceeding to next phase.\n", worker_id);
    }

    pthread_mutex_unlock(&b->lock);
}

void barrier_destroy(CustomBarrier* b) {
    pthread_mutex_destroy(&b->lock);
    pthread_cond_destroy(&b->cv);
}

void* worker_thread(void* arg) {
    long id = (long)arg;

    /* Phase 1: Local Computation */
    printf("[Worker %ld] Executing Phase 1 computational work...\n", id);
    usleep((useconds_t)((id + 1) * 50000)); /* Stagger arrival times */

    /* Synchronization Point */
    barrier_wait(&barrier, (int)id);

    /* Phase 2: Parallel Aggregation */
    printf("[Worker %ld] Executing Phase 2 post-barrier work...\n", id);
    return NULL;
}

int main(void) {
    pthread_t workers[NUM_WORKERS];
    printf("=== Condition Variable Barrier Coordination Demo ===\n");

    barrier_init(&barrier, NUM_WORKERS);

    for (long i = 0; i < NUM_WORKERS; ++i) {
        pthread_create(&workers[i], NULL, worker_thread, (void*)i);
    }

    for (int i = 0; i < NUM_WORKERS; ++i) {
        pthread_join(workers[i], NULL);
    }

    barrier_destroy(&barrier);
    printf("=== All workers finished both phases successfully. ===\n");
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== Condition Variable Barrier Coordination Demo ===
[Worker 0] Executing Phase 1 computational work...
[Worker 1] Executing Phase 1 computational work...
[Worker 2] Executing Phase 1 computational work...
[Worker 3] Executing Phase 1 computational work...
[Worker 0] Reached barrier (1/4 arrived). Sleeping...
[Worker 1] Reached barrier (2/4 arrived). Sleeping...
[Worker 2] Reached barrier (3/4 arrived). Sleeping...
[Worker 3] Reached barrier (LAST THREAD). Broadcasting wakeup!
[Worker 0] Woke up! Proceeding to next phase.
[Worker 1] Woke up! Proceeding to next phase.
[Worker 2] Woke up! Proceeding to next phase.
[Worker 0] Executing Phase 2 post-barrier work...
[Worker 1] Executing Phase 2 post-barrier work...
[Worker 2] Executing Phase 2 post-barrier work...
[Worker 3] Executing Phase 2 post-barrier work...
=== All workers finished both phases successfully. ===
```

---

### Example 2: Non-Spurious Timed Wait with State Predicate
This program demonstrates `pthread_cond_timedwait()` by creating an alert monitor that waits for an emergency signal, safely timing out if no signal arrives within a 1-second deadline.

```c
/*
 * File: condvar_timedwait.c
 * Compilation:
 *   gcc -O2 -pthread condvar_timedwait.c -o condvar_timedwait
 *
 * Execution:
 *   ./condvar_timedwait
 */

#define _POSIX_C_SOURCE 199309L
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <errno.h>
#include <unistd.h>

typedef struct {
    pthread_mutex_t lock;
    pthread_cond_t  cv;
    int alert_triggered;
} AlertSystem;

AlertSystem sys = {
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .cv   = PTHREAD_COND_INITIALIZER,
    .alert_triggered = 0
};

void* alert_watcher(void* arg) {
    (void)arg;
    struct timespec ts;
    int status = 0;

    pthread_mutex_lock(&sys.lock);

    /* Compute 1.0 second absolute deadline */
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += 1;

    printf("[Watcher] Waiting up to 1 second for emergency alert...\n");

    while (!sys.alert_triggered && status != ETIMEDOUT) {
        status = pthread_cond_timedwait(&sys.cv, &sys.lock, &ts);
    }

    if (sys.alert_triggered) {
        printf("[Watcher] ALERT RECEIVED! Initiating system lockdown!\n");
    } else if (status == ETIMEDOUT) {
        printf("[Watcher] TIMEOUT: No alert was triggered within 1 second. System normal.\n");
    }

    pthread_mutex_unlock(&sys.lock);
    return NULL;
}

int main(void) {
    pthread_t watcher;
    printf("=== POSIX Condition Variable Timed Wait Test ===\n");

    /* Test Case: No signal sent -> watcher should time out cleanly */
    pthread_create(&watcher, NULL, alert_watcher, NULL);
    pthread_join(watcher, NULL);

    printf("=== Test Completed Cleanly ===\n");
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== POSIX Condition Variable Timed Wait Test ===
[Watcher] Waiting up to 1 second for emergency alert...
[Watcher] TIMEOUT: No alert was triggered within 1 second. System normal.
=== Test Completed Cleanly ===
```

---

## 6. Mathematical Formulations & Overhead Analysis

### The Thundering Herd Problem: `signal` vs `broadcast`

Suppose $M$ threads are sleeping on a single condition variable. When `pthread_cond_broadcast()` is called:
- All $M$ threads are moved from the condition variable wait-queue to the mutex acquisition queue.
- Let $C_{\text{wake}}$ be the kernel cost of waking a thread ($\approx 2\,\mu\text{s}$).
- Let $C_{\text{contend}}$ be the cache invalidation and bus lock cost per failed mutex acquisition attempt.

The total wake-up latency and contention penalty across all threads is:

$$T_{\text{herd}}(M) = M \cdot C_{\text{wake}} + \sum_{k=1}^{M-1} k \cdot C_{\text{contend}} = M \cdot C_{\text{wake}} + \frac{M(M-1)}{2} C_{\text{contend}}$$

Notice the quadratic term:

$$\mathcal{O}(M^2) \text{ contention overhead}$$

Only **one thread** will successfully acquire the mutex and consume the available resource. The remaining $M-1$ threads wake up, find the resource gone (because the while-loop condition evaluates back to false), and re-enter `pthread_cond_wait()`, wasting substantial CPU time.

Therefore, when only one thread can consume the resource:

$$\text{Synchronous Cost Ratio} = \frac{T(\text{broadcast})}{T(\text{signal})} \approx \frac{\mathcal{O}(M^2)}{\mathcal{O}(1)}$$

Always prefer `pthread_cond_signal()` over `broadcast()` unless the state change genuinely affects all waiting threads.

---

## 7. Advantages, Disadvantages & Limitations

### Trade-off Analysis

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Condition Variable        | - True zero CPU consumption while asleep| - No internal memory: signals without   |
| (`pthread_cond_t`)        | - Event-driven, reactive coordination   |   waiting threads are permanently lost  |
|                           | - Immune to polling latency             | - Subject to Spurious Wakeups           |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_cond_wait()`     | - Atomic mutex release and sleep        | - Calling without holding the mutex     |
|                           | - Re-acquires mutex before returning    |   results in undefined behavior         |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `pthread_cond_broadcast()`| - Wakes all threads simultaneously      | - "Thundering Herd" lock contention     |
|                           | - Ideal for phase barriers / shutdowns  |   overhead scales quadratically O(M^2)  |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Condition Variable (`pthread_cond_t`):** A synchronization primitive that allows threads to suspend execution (sleep without burning CPU cycles) until another thread notifies them that a shared state predicate has been satisfied.
>
> **Lost Wakeup:** A race condition where a thread signals a condition variable before a waiting thread has entered the wait state, causing the signal to be discarded and leaving the waiting thread suspended indefinitely.
>
> **Spurious Wakeup:** An occurrence in which a thread waiting on a condition variable is awakened without any explicit call to `pthread_cond_signal()` or `pthread_cond_broadcast()`, or where the underlying condition predicate remains unsatisfied.
>
> **Atomic Mutex Release:** The property of `pthread_cond_wait()` ensuring that the release of the associated mutex and the registration of the thread onto the condition variable's sleep queue occur as an indivisible operation, eliminating the race window for lost wakeups.
>
> **Thundering Herd Problem:** A performance degradation phenomenon occurring when `pthread_cond_broadcast()` awakens many waiting threads simultaneously, causing them to contend aggressively for the single shared mutex, resulting in severe CPU overhead and cache-line invalidation.

---

## 9. Comparison Tables

### `pthread_cond_signal()` vs `pthread_cond_broadcast()`

| Criterion | `pthread_cond_signal(&cv)` | `pthread_cond_broadcast(&cv)` |
| :--- | :--- | :--- |
| **Number of Threads Awakened**| Exactly **one** thread | **All** waiting threads |
| **Wait Queue Behavior** | Dequeues the first thread | Flushes the entire wait queue |
| **Contention Overhead** | Minimal ($\mathcal{O}(1)$) | Heavy / Thundering Herd ($\mathcal{O}(M^2)$) |
| **Typical Use Case** | Producer pushes one item to a queue | Barrier completion, system shutdown, reset |
| **Safety with Multiple Waiters**| Safe only if any waiter can handle state | Safe for all synchronization patterns |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Diagnosing the `if` vs `while` Flaw
**Question:** Explain what will go wrong in the following multi-consumer queue implementation if two consumer threads are sleeping and one item is pushed to the queue.

```c
/* DEFECTIVE CONSUMER CODE */
pthread_mutex_lock(&q_lock);
if (queue_is_empty()) {
    pthread_cond_wait(&q_cv, &q_lock);
}
Item* item = dequeue(); /* Segfault or corruption! */
pthread_mutex_unlock(&q_lock);
```

**Solution:**
1. Both Consumer 1 and Consumer 2 are blocked in `pthread_cond_wait()`.
2. A producer adds **one** item to the queue and calls `pthread_cond_broadcast(&q_cv)` (or a spurious wakeup occurs).
3. Both consumers wake up and attempt to acquire `q_lock`.
4. Consumer 1 wins the lock race, exits `pthread_cond_wait()`, successfully dequeues the single item, and unlocks `q_lock`.
5. Consumer 2 now acquires `q_lock`. Because it used an **`if`** statement instead of a **`while`** loop, it does not re-evaluate `queue_is_empty()`. It proceeds directly to `dequeue()`.
6. Since the queue is now empty, `dequeue()` reads a `NULL` pointer or uninitialized memory, triggering a segmentation fault (`SIGSEGV`) or data corruption.
- **The Fix:** Change `if` to `while (queue_is_empty())`. When Consumer 2 acquires the lock, the loop re-checks the condition, detects that the queue is empty, and safely goes back to sleep.

---

### Problem 2: Signal Inside vs Outside Mutex
**Question:** In POSIX, is it legally permissible to call `pthread_cond_signal(&cv)` after unlocking the mutex? What are the trade-offs?

```c
/* Approach A: Signal INSIDE lock */
pthread_mutex_lock(&lock);
condition = 1;
pthread_cond_signal(&cv);
pthread_mutex_unlock(&lock);

/* Approach B: Signal OUTSIDE lock */
pthread_mutex_lock(&lock);
condition = 1;
pthread_mutex_unlock(&lock);
pthread_cond_signal(&cv);
```

**Analysis & Solution:**
- **Correctness:** Both approaches are 100% legal under the POSIX standard. The mutex is required when waiting, but is not required when signaling.
- **Performance Trade-offs:**
  - **In Approach A (Inside):** The signaled thread wakes up immediately, but the signaling thread still holds the mutex! The awakened thread may immediately block on the mutex before being able to run. Modern Linux kernels optimize this via *wait morphing* (moving the thread from the CV queue directly to the mutex queue without a full context switch).
  - **In Approach B (Outside):** The mutex is released before signaling, so the awakened thread can often acquire the mutex without contention. However, if priority inheritance or real-time scheduling is active, unlocking before signaling can cause unpredictable scheduling order.
- **Standard Recommendation:** Signaling inside the lock is standard practice because it prevents subtle race conditions when condition variables are dynamically destroyed.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why does `pthread_cond_wait()` require the mutex parameter?
**Answer:** It requires the mutex to prevent the **lost wakeup race condition**. Inside `pthread_cond_wait()`, the operating system must atomically place the calling thread onto the condition variable's sleep queue and release the mutex. Without the mutex, another thread could modify the predicate and send the signal between the time the waiting thread checked the condition and the time it went to sleep, resulting in a permanent hang.

### Q2: What happens if a condition variable is signaled when NO thread is waiting?
**Answer:** The signal is **silently ignored and permanently lost**. Condition variables do not have state memory or counters. If state persistence is required, use a counting semaphore (`sem_t`) or maintain an explicit state variable (predicate) protected by a mutex.

### Q3: Why does `pthread_cond_timedwait()` use an absolute time instead of a relative interval?
**Answer:** Because `pthread_cond_timedwait()` is typically called inside a `while` loop to guard against spurious wakeups. If it used a relative interval (e.g., "wait 500 ms"), each time the thread woke up spuriously, it would sleep for another full 500 ms, drastically stretching the intended timeout. By using an absolute timestamp (e.g., "wait until 14:00:05.500"), the thread can loop repeatedly upon spurious wakeups and still expire at the exact intended deadline.

---

## 12. Quick Revision Sheet

- **Condition Variable Role:** Sleep without CPU burning until an application state predicate is true.
- **Pairing Requirement:** Must ALWAYS be paired with a `pthread_mutex_t`.
- **The Golden Pattern:**
  ```c
  pthread_mutex_lock(&m);
  while (!predicate) {
      pthread_cond_wait(&cv, &m);
  }
  // Mutex is held and predicate is true!
  pthread_mutex_unlock(&m);
  ```
- **Spurious Wakeup:** A thread waking without a signal. Prevented entirely by the `while` loop.
- **Signaling:**
  - `pthread_cond_signal(&cv)`: Wakes 1 thread (efficient for queues).
  - `pthread_cond_broadcast(&cv)`: Wakes all threads (best for barriers/shutdown).
- **Timed Wait:** `pthread_cond_timedwait(&cv, &m, &abstime)` requires absolute wall-clock time (`CLOCK_REALTIME`).

---

## 13. What You Should Be Able to Do

1. **Implement custom synchronization barriers** using `pthread_cond_t` and `pthread_mutex_t`.
2. **Defend why condition variables must wait in a while loop** in theory exams and technical interviews.
3. **Prevent lost wakeups** by correctly serializing state predicate modification and signaling.
4. **Choose appropriately** between `pthread_cond_signal()` and `pthread_cond_broadcast()` to avoid thundering herd contention.
5. **Implement robust timeout mechanisms** using `pthread_cond_timedwait()` with correct `struct timespec` arithmetic.
