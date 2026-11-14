# Module 09: Pthreads Synchronization Primitives (Lectures 20–23)
# Subtopic 09.03: Producer–Consumer and POSIX Semaphores (`sem_t`, `sem_init`, `sem_wait`, `sem_post`, Bounded Buffer)

---

## 1. Basic Idea

The **Producer–Consumer Problem** (also known as the **Bounded-Buffer Problem**) is a foundational synchronization paradigm in parallel computing, operating systems, and distributed systems. 

In this model:
- One or more **Producers** generate data items and deposit them into a shared, finite-capacity buffer of size $K$.
- One or more **Consumers** retrieve items from the buffer and process them.

```
                  +-----------------------------------+
                  |      Shared Bounded Buffer (K)    |
                  |  +-----+-----+-----+-----+-----+  |
[ Producers ] --->|  | D_0 | D_1 | D_2 |     |     |  | ---> [ Consumers ]
 (Deposit Items)  |  +-----+-----+-----+-----+-----+  |     (Extract Items)
                  |     ^                 ^           |
                  |     |                 |           |
                  |   Out (Extract)      In (Insert)  |
                  +-----------------------------------+
```

Two fundamental boundary conditions must be enforced:
1. **Buffer-Full Condition:** A producer must block (sleep) when the buffer contains $K$ items, waiting until a consumer frees at least one slot.
2. **Buffer-Empty Condition:** A consumer must block (sleep) when the buffer contains $0$ items, waiting until a producer deposits at least one item.
3. **Mutual Exclusion:** Simultaneous access to buffer pointers (`in` and `out`) by multiple producers or consumers must be strictly serialized to prevent data corruption.

While this can be solved using a mutex and two condition variables (`not_full`, `not_empty`), POSIX provides a dedicated, highly efficient primitive: the **Counting Semaphore** (`sem_t`), defined in `<semaphore.h>`.

---

## 2. Why We Need It

### 1. Decoupling Execution Rates (Asynchronous Pipelining)
In High-Performance Computing and media processing pipelines, data generation and data consumption rarely proceed at identical speeds. For example:
- A disk reader thread reads chunks at variable I/O speeds (e.g., encountering disk seek latency).
- A GPU worker kernel consumes frames at high computational throughput.
A bounded buffer acts as an **elastic reservoir**, absorbing bursty production or consumption without stalling the system, while bounding maximum memory usage to $K$ slots.

### 2. POSIX Realtime Extensions (`<semaphore.h>`)
Unlike mutexes and condition variables, which were historically part of the base Pthreads specification (`libpthread`), POSIX semaphores originated from the **POSIX Realtime Extensions** (IEEE Std 1003.1b-1993). They provide:
- **State Memory:** A semaphore maintains an integer counter. If a producer calls `sem_post()` before a consumer calls `sem_wait()`, the signal is **remembered** (incrementing the counter). This completely eliminates the "lost wakeup" problem inherent to condition variables.
- **Asynchronous Signal Safety:** In contrast to `pthread_mutex_unlock()`, calling `sem_post()` is officially **async-signal-safe** in POSIX, making it legal to use inside OS signal handlers.

---

## 3. How It Works

### The Anatomy of a Counting Semaphore

A counting semaphore $S$ is an integer variable managed atomically by the operating system kernel, supporting two primitive atomic operations:

```
Operation 1: sem_wait(S)   [Historically called P(S) or down(S)]
  - If S > 0: Atomically decrements S by 1 and returns immediately.
  - If S == 0: Blocks the calling thread, placing it onto the semaphore's
               sleep queue until S becomes greater than 0.

Operation 2: sem_post(S)   [Historically called V(S) or up(S)]
  - Atomically increments S by 1.
  - If threads are blocked in sem_wait(S), the OS kernel wakes up one of
    the sleeping threads, which then decrements S and proceeds.
```

```
+-----------------------------------------------------------------------------------------+
|                                Semaphore State Transitions                              |
+-----------------------------------------------------------------------------------------+
| Counter Value S:                                                                        |
|   S > 0  : Exactly S resources are available immediately.                               |
|   S == 0 : No resources available. Any subsequent sem_wait() will block.                |
|                                                                                         |
| sem_post(&s) ---> Atomically does: S = S + 1; wake_one_waiter_if_any();                 |
| sem_wait(&s) ---> Atomically does: while (S == 0) sleep(); S = S - 1;                   |
+-----------------------------------------------------------------------------------------+
```

---

### The Three-Semaphore Solution for Bounded Buffers

To coordinate multi-producer multi-consumer access to a buffer of capacity $K$, three semaphores are employed:

1. **`sem_t empty_slots` (Counting Semaphore):**
   - Initialized to **$K$** (all slots are initially empty).
   - Producers wait on `empty_slots` before inserting.
   - Consumers post to `empty_slots` after removing.

2. **`sem_t full_slots` (Counting Semaphore):**
   - Initialized to **$0$** (no slots initially contain data).
   - Consumers wait on `full_slots` before removing.
   - Producers post to `full_slots` after inserting.

3. **`pthread_mutex_t buffer_mutex` (Binary Exclusion):**
   - Serializes manipulation of the circular queue pointers (`in`, `out`) among multiple producers or consumers.

```
PRODUCER WORKFLOW:                       CONSUMER WORKFLOW:
  sem_wait(&empty_slots);  // Wait for space   sem_wait(&full_slots);   // Wait for item
  pthread_mutex_lock(&mtx);                   pthread_mutex_lock(&mtx);
  [ INSERT ITEM INTO BUFFER ]                 [ REMOVE ITEM FROM BUFFER ]
  pthread_mutex_unlock(&mtx);                 pthread_mutex_unlock(&mtx);
  sem_post(&full_slots);   // Signal data     sem_post(&empty_slots);  // Signal space
```

```
+-----------------------------------------------------------------------------------------+
|                       THE DEADLOCK HAZARD: SEMAPHORE ORDERING                           |
+-----------------------------------------------------------------------------------------+
| WRONG PRODUCER SEQUENCE:                                                                |
|   pthread_mutex_lock(&mtx);       <--- ACQUIRES MUTEX FIRST!                            |
|   sem_wait(&empty_slots);         <--- BLOCKS! (Buffer is full, waiting for consumer)   |
|                                                                                         |
| CONSUMER SEQUENCE:                                                                      |
|   sem_wait(&full_slots);                                                                |
|   pthread_mutex_lock(&mtx);       <--- BLOCKS FOREVER! (Mutex is held by Producer!)     |
|                                                                                         |
| ===> SYSTEM DEADLOCK! Producer holds mutex waiting for Consumer to free a slot;         |
|      Consumer cannot free a slot because it is blocked waiting for the Mutex!          |
|                                                                                         |
| GOLDEN RULE: ALWAYS acquire the counting semaphore BEFORE locking the mutex!            |
+-----------------------------------------------------------------------------------------+
```

---

## 4. Important Mechanisms and Functions

### 1. Initialization and Destruction

```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_destroy(sem_t *sem);
```

- **`sem`:** Pointer to uninitialized `sem_t` structure.
- **`pshared`:**
  - `0`: **Thread-shared** (private to the current process). The semaphore is shared only between threads of the calling process.
  - Non-zero: **Process-shared**. The semaphore resides in a shared memory region (e.g., POSIX `shm_open` or `mmap`) and can synchronize multiple distinct processes.
- **`value`:** Initial integer counter value ($\ge 0$).
- Returns `0` on success, or `-1` with `errno` set on error (`EINVAL`).

---

### 2. Waiting and Decrementing

```c
int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem);
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);
```

- **`sem_wait()`:** Blocks if semaphore value is 0; decrements once value is $> 0$. Returns `0` on success, or `-1` on error (e.g., interrupted by signal: `EINTR`).
- **`sem_trywait()`:** Non-blocking check. If counter $> 0$, decrements and returns `0`. If counter is 0, returns `-1` with `errno = EAGAIN`.
- **`sem_timedwait()`:** Blocks until absolute wall-clock timeout expires. If timeout elapses before decrement, returns `-1` with `errno = ETIMEDOUT`.

---

### 3. Posting (Incrementing)

```c
int sem_post(sem_t *sem);
```

- Atomically increments the semaphore counter.
- If any threads are sleeping on `sem_wait()`, the kernel awakens one thread.
- Returns `0` on success, or `-1` on error (`EOVERFLOW`).
- **Async-signal-safe:** Can be invoked from UNIX signal handlers.

---

### 4. Querying Semaphore Value

```c
int sem_getvalue(sem_t *sem, int *sval);
```
- Copies current counter into `*sval`.
- *Note:* In concurrent environments, this value is purely informational and may become outdated immediately after the call returns.

---

## 5. Compilable C Code Examples

### Example 1: Multi-Producer Multi-Consumer Bounded Buffer with POSIX Semaphores
This complete program creates a circular bounded FIFO buffer of size 5, spawned with 2 Producer threads and 2 Consumer threads, demonstrating mutual exclusion, clean blocking, and graceful termination using sentinel values.

```c
/*
 * File: producer_consumer_sem.c
 * Compilation:
 *   gcc -O2 -pthread producer_consumer_sem.c -o producer_consumer_sem
 *
 * Execution:
 *   ./producer_consumer_sem
 */

#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define BUFFER_SIZE 5
#define ITEMS_PER_PRODUCER 10
#define NUM_PRODUCERS 2
#define NUM_CONSUMERS 2
#define SENTINEL_VALUE -1

/* Circular Bounded Buffer Structure */
typedef struct {
    int data[BUFFER_SIZE];
    int in;
    int out;
    pthread_mutex_t lock;
    sem_t empty_slots;
    sem_t full_slots;
} BoundedBuffer;

BoundedBuffer buffer;

void buffer_init(BoundedBuffer* b) {
    b->in = 0;
    b->out = 0;
    pthread_mutex_init(&b->lock, NULL);
    /* Initially: BUFFER_SIZE empty slots, 0 full slots */
    sem_init(&b->empty_slots, 0, BUFFER_SIZE);
    sem_init(&b->full_slots, 0, 0);
}

void buffer_destroy(BoundedBuffer* b) {
    pthread_mutex_destroy(&b->lock);
    sem_destroy(&b->empty_slots);
    sem_destroy(&b->full_slots);
}

/* Insert item into buffer */
void buffer_insert(BoundedBuffer* b, int item) {
    /* 1. Wait for an available empty slot */
    sem_wait(&b->empty_slots);

    /* 2. Lock buffer for exclusive pointer manipulation */
    pthread_mutex_lock(&b->lock);

    b->data[b->in] = item;
    b->in = (b->in + 1) % BUFFER_SIZE;

    pthread_mutex_unlock(&b->lock);

    /* 3. Signal that a new full slot is available */
    sem_post(&b->full_slots);
}

/* Remove item from buffer */
int buffer_remove(BoundedBuffer* b) {
    /* 1. Wait for an available full slot */
    sem_wait(&b->full_slots);

    /* 2. Lock buffer for exclusive pointer manipulation */
    pthread_mutex_lock(&b->lock);

    int item = b->data[b->out];
    b->out = (b->out + 1) % BUFFER_SIZE;

    pthread_mutex_unlock(&b->lock);

    /* 3. Signal that an empty slot has been freed */
    sem_post(&b->empty_slots);

    return item;
}

/* Producer Thread Routine */
void* producer_worker(void* arg) {
    long id = (long)arg;
    for (int i = 0; i < ITEMS_PER_PRODUCER; ++i) {
        int item = (int)(id * 100 + i + 1);
        buffer_insert(&buffer, item);
        printf("[Producer %ld] Deposited Item: %d\n", id, item);
        usleep(30000); /* Simulate production latency (30ms) */
    }
    return NULL;
}

/* Consumer Thread Routine */
void* consumer_worker(void* arg) {
    long id = (long)arg;
    int items_processed = 0;

    while (1) {
        int item = buffer_remove(&buffer);
        if (item == SENTINEL_VALUE) {
            printf("[Consumer %ld] Received SENTINEL. Terminating. Total processed: %d\n",
                   id, items_processed);
            break;
        }
        items_processed++;
        printf("  [Consumer %ld] Consumed Item: %d\n", id, item);
        usleep(50000); /* Simulate processing latency (50ms) */
    }
    return NULL;
}

int main(void) {
    pthread_t producers[NUM_PRODUCERS];
    pthread_t consumers[NUM_CONSUMERS];

    printf("=== Multi-Producer Multi-Consumer Bounded Buffer with POSIX Semaphores ===\n");
    printf("Buffer Capacity: %d | Producers: %d | Consumers: %d\n\n",
           BUFFER_SIZE, NUM_PRODUCERS, NUM_CONSUMERS);

    buffer_init(&buffer);

    /* Spawn Consumers */
    for (long i = 0; i < NUM_CONSUMERS; ++i) {
        pthread_create(&consumers[i], NULL, consumer_worker, (void*)i);
    }

    /* Spawn Producers */
    for (long i = 0; i < NUM_PRODUCERS; ++i) {
        pthread_create(&producers[i], NULL, producer_worker, (void*)i);
    }

    /* Wait for Producers to finish generating work */
    for (int i = 0; i < NUM_PRODUCERS; ++i) {
        pthread_join(producers[i], NULL);
    }

    /* Send Sentinel shutdown tokens to consumers */
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        buffer_insert(&buffer, SENTINEL_VALUE);
    }

    /* Wait for Consumers to terminate */
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        pthread_join(consumers[i], NULL);
    }

    buffer_destroy(&buffer);
    printf("\n=== All items produced and consumed with zero data loss or deadlocks! ===\n");
    return EXIT_SUCCESS;
}
```

```
--- Expected Output ---
=== Multi-Producer Multi-Consumer Bounded Buffer with POSIX Semaphores ===
Buffer Capacity: 5 | Producers: 2 | Consumers: 2

[Producer 0] Deposited Item: 1
[Producer 1] Deposited Item: 101
  [Consumer 0] Consumed Item: 1
  [Consumer 1] Consumed Item: 101
[Producer 0] Deposited Item: 2
[Producer 1] Deposited Item: 102
...
[Consumer 0] Received SENTINEL. Terminating. Total processed: 10
[Consumer 1] Received SENTINEL. Terminating. Total processed: 10

=== All items produced and consumed with zero data loss or deadlocks! ===
```

---

## 6. Mathematical Formulations & Overhead Analysis

### Semaphore Invariant and Bounded-Buffer Properties

Let $N_{\text{empty}}$ be the value of `empty_slots` and $N_{\text{full}}$ be the value of `full_slots`. Let $K$ be the fixed buffer capacity. At all points during correct execution, the following invariant holds:

$$N_{\text{empty}} + N_{\text{full}} = K$$

Furthermore, the bounds on each counter are strictly constrained:

$$0 \le N_{\text{empty}} \le K$$
$$0 \le N_{\text{full}} \le K$$

Let $P(t)$ be the total cumulative items inserted by all producers up to time $t$, and let $C(t)$ be the total cumulative items removed by all consumers up to time $t$. The physical items currently present in the buffer $B(t)$ satisfies:

$$B(t) = P(t) - C(t) = N_{\text{full}}(t) = K - N_{\text{empty}}(t)$$

Since $0 \le B(t) \le K$:

$$0 \le P(t) - C(t) \le K \iff C(t) \le P(t) \le C(t) + K$$

This mathematical inequality formally proves that:
1. **No underflow occurs:** Consumers can never consume more items than have been produced ($C(t) \le P(t)$).
2. **No overflow occurs:** Producers can never outpace consumers by more than the buffer capacity $K$ ($P(t) \le C(t) + K$).

---

## 7. Advantages, Disadvantages & Limitations

### Comparing Synchronization Paradigms for Bounded Buffers

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Primitive                 | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| POSIX Semaphores          | - State memory: signals are never lost  | - Prone to deadlocks if ordering with   |
| (`sem_t`)                 | - Clean mathematical model for pools    |   mutex is reversed                     |
|                           | - Async-signal-safe (sem_post)          | - No ownership tracking                 |
+---------------------------+-----------------------------------------+-----------------------------------------+
| Mutex + Cond Variables    | - Flexible predicates (arbitrary logic) | - Lost wakeups if signaled without lock |
| (`pthread_cond_t`)        | - Explicit ownership on mutex           | - Spurious wakeups require while-loop   |
|                           | - Broadcast capability to all waiters   | - Slightly higher code complexity       |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Counting Semaphore (`sem_t`):** A synchronization primitive initialized with an integer counter value, providing two atomic operations: `sem_wait()` (which decrements the counter or blocks if the counter is 0) and `sem_post()` (which increments the counter and awakens a sleeping thread).
>
> **Binary Semaphore:** A semaphore whose counter value can only alternate between `0` and `1`, functionally equivalent to a mutual exclusion lock without ownership enforcement.
>
> **Bounded Buffer Problem:** A classic synchronization challenge where multiple producer and consumer threads share a fixed-size queue, requiring producers to block when the queue is full and consumers to block when the queue is empty.
>
> **POSIX Realtime Extensions (IEEE 1003.1b):** An amendment to the POSIX standard that introduced real-time capabilities to UNIX systems, including counting semaphores (`<semaphore.h>`), priority scheduling, timers, and shared memory.
>
> **Async-Signal-Safe Function:** A function that is guaranteed to execute safely and without data corruption or deadlocks when invoked from inside an asynchronous UNIX signal handler (e.g., `sem_post()`).

---

## 9. Comparison Tables

### Mutex vs Binary Semaphore vs Condition Variable

| Dimension | Mutex (`pthread_mutex_t`) | Binary Semaphore (`sem_t`) | Condition Variable (`pthread_cond_t`) |
| :--- | :--- | :--- | :--- |
| **Ownership** | **Strict Ownership** (Must be unlocked by locker) | **No Ownership** (Any thread can post) | **No Ownership** (Associated with mutex) |
| **State Memory** | Single bit (0 or 1) | Integer counter ($\ge 0$) | **No Memory** (Signals without waiters vanish) |
| **Header File** | `<pthread.h>` | `<semaphore.h>` | `<pthread.h>` |
| **Primary Use** | Mutual Exclusion in Critical Sections | Resource counting / Signaling across threads | Waiting for complex state predicates |
| **Signal Safety**| **Not** async-signal-safe | `sem_post()` **is** async-signal-safe | **Not** async-signal-safe |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Reversing the Wait Order in Producer
**Question:** Explain what catastrophic bug occurs if a student writes the producer routine as:

```c
/* ATTEMPTED PRODUCER */
void produce(int item) {
    pthread_mutex_lock(&buffer_mutex);  /* Line 1 */
    sem_wait(&empty_slots);             /* Line 2 */
    insert_item(item);
    pthread_mutex_unlock(&buffer_mutex);
    sem_post(&full_slots);
}
```

**Solution:**
- **The Defect:** The producer locks the mutex **before** waiting on `empty_slots`.
- **The Execution Scenario:**
  1. The buffer becomes full ($N_{\text{empty}} = 0$).
  2. A producer arrives, executes Line 1, and acquires `buffer_mutex`.
  3. The producer proceeds to Line 2 and calls `sem_wait(&empty_slots)`. Because `empty_slots == 0`, the producer is put to sleep in the kernel.
  4. Now, a consumer arrives to consume an item and free a slot.
  5. The consumer executes:
     ```c
     sem_wait(&full_slots);
     pthread_mutex_lock(&buffer_mutex); // BLOCKS!
     ```
  6. The consumer blocks because `buffer_mutex` is held by the sleeping producer!
  7. The producer cannot wake up until the consumer executes `sem_post(&empty_slots)`. But the consumer cannot reach `sem_post(&empty_slots)` because it is blocked waiting for `buffer_mutex`.
- **Result:** **Deadlock** (circular wait between `buffer_mutex` and `empty_slots`).
- **Correction:** Always acquire `sem_wait(&empty_slots)` *first*, and *then* lock `buffer_mutex`.

---

### Problem 2: Process-Shared Semaphores
**Question:** How does setting the `pshared` parameter to `1` in `sem_init()` alter the behavior and storage requirements of the semaphore?

**Solution:**
- When `pshared == 0`, the semaphore is **thread-shared**; it resides in process-local memory and can only synchronize threads belonging to the same address space.
- When `pshared != 0`, the semaphore is **process-shared**; it can synchronize completely independent processes running on the operating system.
- **Storage Requirement:** A process-shared semaphore must reside in a memory region that is accessible to all interacting processes—such as a POSIX shared memory object (`shm_open()` + `mmap()`) or a shared memory segment mapped with `MAP_SHARED`. Storing a process-shared semaphore in standard stack or unshared heap memory results in failure or undefined behavior.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why is a binary semaphore NOT identical to a mutex?
**Answer:** Although both can enforce mutual exclusion by alternating between 0 and 1, they have fundamental differences:
1. **Ownership:** A mutex has an owner (the thread that locked it). It is illegal or undefined for another thread to unlock it. A semaphore has no owner; any thread can execute `sem_post()` to unblock a thread waiting in `sem_wait()`.
2. **Priority Inversion Protocols:** Mutexes support priority inheritance protocols (`PTHREAD_PRIO_INHERIT`) to prevent priority inversion in real-time systems. Semaphores cannot support priority inheritance because the operating system does not know which thread will eventually call `sem_post()`.

### Q2: Why is `sem_post()` allowed inside UNIX signal handlers while `pthread_mutex_unlock()` is forbidden?
**Answer:** `sem_post()` is specifically designed to be **async-signal-safe**. It executes a simple atomic fetch-and-add instruction followed by a kernel futex wake, without taking internal user-space locks or allocating memory. In contrast, `pthread_mutex_unlock()` interacts with thread ownership structures and robust mutex tracking lists; calling it from a signal handler that interrupted another thread inside the runtime library would cause reentrancy deadlocks.

### Q3: What is the significance of the `<semaphore.h>` header being separate from `<pthread.h>`?
**Answer:** POSIX threads (`<pthread.h>`) and POSIX semaphores (`<semaphore.h>`) originated from two separate POSIX working group standards. Semaphores were introduced in IEEE Std 1003.1b (Realtime Extensions) to provide inter-process communication, whereas Pthreads were formalized in IEEE Std 1003.1c (Threads Extensions). Consequently, they reside in distinct headers and historically required distinct library links (`-lrt` vs `-lpthread`, though consolidated into `libc`/`libpthread` on modern Linux).

---

## 12. Quick Revision Sheet

- **Producer-Consumer Problem:** Manages shared bounded buffer of capacity $K$.
- **Required Semaphores:**
  - `empty_slots` initialized to $K$.
  - `full_slots` initialized to $0$.
  - `buffer_mutex` (mutex or binary semaphore initialized to 1).
- **Core Rule for Deadlock Prevention:**
  - Always call `sem_wait(&empty_slots)` or `sem_wait(&full_slots)` **BEFORE** acquiring `pthread_mutex_lock()`.
- **Functions:**
  - `sem_init(&sem, pshared, value)`: `pshared=0` (threads), `pshared=1` (processes).
  - `sem_wait(&sem)`: Decrements or blocks if 0.
  - `sem_post(&sem)`: Increments and wakes one waiter.
  - `sem_destroy(&sem)`: Cleans up semaphore.
- **Invariant:** `N_empty + N_full == K`.

---

## 13. What You Should Be Able to Do

1. **Implement a complete, deadlock-free multi-producer multi-consumer bounded buffer** in C using POSIX semaphores.
2. **Explain the deadlock vulnerability** that arises when semaphore wait calls are nested inside mutex locks.
3. **Contrast POSIX semaphores with condition variables** in terms of state memory and signal safety.
4. **Configure thread-shared vs. process-shared semaphores** using the `pshared` parameter.
5. **Implement clean shutdown protocols** for multi-consumer queues using sentinel values.
