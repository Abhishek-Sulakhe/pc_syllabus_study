# OpenMP vs. POSIX Threads (Pthreads): The Definitive Comparative Code Guide

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 6–23)  
**Applicability:** Theory Examinations, Laboratory Practical Exams, System Architecture & Concurrency Design  
**Implementation Standard:** Every pattern is presented with fully compilable, self-contained C code for **both** OpenMP and POSIX Threads, featuring compilation commands, explicit data scoping, memory management, and side-by-side technical trade-off analyses.

---

## Table of Contents
1. [Architectural & Conceptual Paradigm Shift](#1-architectural--conceptual-paradigm-shift)
2. [Pattern 1: Thread Spawning, Identification & Joining](#pattern-1-thread-spawning-identification--joining)
3. [Pattern 2: Work-Sharing & Parallel For Loops](#pattern-2-work-sharing--parallel-for-loops)
4. [Pattern 3: Critical Sections (Mutual Exclusion)](#pattern-3-critical-sections-mutual-exclusion)
5. [Pattern 4: Atomic Operations](#pattern-4-atomic-operations)
6. [Pattern 5: Parallel Reduction (Array Sum)](#pattern-5-parallel-reduction-array-sum)
7. [Pattern 6: Barrier Synchronization (Phase Alignment)](#pattern-6-barrier-synchronization-phase-alignment)
8. [Pattern 7: Thread Coordination & Event Signaling (Condition Variables vs. Locks)](#pattern-7-thread-coordination--event-signaling)
9. [Pattern 8: Producer–Consumer Bounded Buffer](#pattern-8-producerconsumer-bounded-buffer)
10. [Pattern 9: Read-Write Locks for Read-Heavy Workloads](#pattern-9-read-write-locks-for-read-heavy-workloads)
11. [Pattern 10: Dynamic Asynchronous Tasking](#pattern-10-dynamic-asynchronous-tasking)
12. [Master Architectural Comparison Matrix](#12-master-architectural-comparison-matrix)
13. [Engineering Decision Matrix: When to Choose Which?](#13-engineering-decision-matrix-when-to-choose-which)

---

## 1. Architectural & Conceptual Paradigm Shift

Before analyzing code implementations, understand the fundamental design divergence between the two paradigms:

```
+-----------------------------------------------------------------------------------------+
|                               PARADIGM ARCHITECTURAL SHIFT                              |
+-----------------------------------------------------------------------------------------+
| Feature                   | OpenMP                               | POSIX Threads (Pthreads)     |
|---------------------------+--------------------------------------+------------------------------|
| Model Classification      | Declarative (Directive-based)        | Imperative (Procedural API)  |
| Primary Actor             | Compiler & Runtime Library           | Programmer & OS Kernel       |
| Thread Management         | Automatic (Managed Fork-Join pool)   | Explicit (Manual create/join)|
| Execution Model           | Fork-Join team across code blocks    | Free-running peer threads    |
| Data Scoping              | Directives (shared, private, etc.)   | Manual (Heap, stack, globals)|
| Work Distribution         | Automatic loop chunk partitioning    | Manual arithmetic on indices |
| Compiler Flag             | `-fopenmp` (GCC/Clang), `-qopenmp`   | `-pthread` (Links libpthread)|
| Headers                   | `<omp.h>`                            | `<pthread.h>`, `<semaphore.h>`|
+-----------------------------------------------------------------------------------------+
```

---

## Pattern 1: Thread Spawning, Identification & Joining

### 1.1 Objective
Spawn $T$ parallel threads, retrieve each thread's unique integer ID and the total thread count, execute work concurrently, and synchronize back into a single thread.

---

### 1.2 OpenMP Implementation
```c
/*
 * openmp_spawn.c
 * Spawning threads and retrieving IDs using OpenMP.
 *
 * Compilation: gcc -O2 -fopenmp openmp_spawn.c -o openmp_spawn
 * Execution:   ./openmp_spawn
 */
#include <stdio.h>
#include <omp.h>

int main(void) {
    int requested_threads = 4;
    omp_set_num_threads(requested_threads);

    printf("[OpenMP] Starting sequential execution on Master Thread.\n");

    /* Fork a team of threads */
    #pragma omp parallel default(none)
    {
        int tid = omp_get_thread_num();
        int num_threads = omp_get_num_threads();

        printf("  -> Hello from OpenMP Thread %d of %d\n", tid, num_threads);
    } /* Implicit Barrier & Join occur at this closing brace */

    printf("[OpenMP] Joined back into single Master Thread.\n");
    return 0;
}
```

---

### 1.3 POSIX Threads Implementation
```c
/*
 * pthreads_spawn.c
 * Explicit thread creation, argument passing, and joining using Pthreads.
 *
 * Compilation: gcc -O2 -pthread pthreads_spawn.c -o pthreads_spawn
 * Execution:   ./pthreads_spawn
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 4

/* Dedicated struct to prevent argument aliasing */
typedef struct {
    int thread_id;
    int total_threads;
} ThreadData;

void *worker_function(void *arg) {
    ThreadData *data = (ThreadData *)arg;
    printf("  -> Hello from POSIX Thread %d of %d\n", data->thread_id, data->total_threads);
    free(data); /* Reclaim heap memory allocated by main thread */
    pthread_exit(NULL);
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    int rc;

    printf("[Pthreads] Starting sequential execution in main process.\n");

    /* Explicit thread creation loop */
    for (int t = 0; t < NUM_THREADS; t++) {
        ThreadData *data = (ThreadData *)malloc(sizeof(ThreadData));
        data->thread_id = t;
        data->total_threads = NUM_THREADS;

        rc = pthread_create(&threads[t], NULL, worker_function, (void *)data);
        if (rc != 0) {
            fprintf(stderr, "pthread_create failed for thread %d (code %d)\n", t, rc);
            return 1;
        }
    }

    /* Explicit joining loop (Resource reclamation) */
    for (int t = 0; t < NUM_THREADS; t++) {
        rc = pthread_join(threads[t], NULL);
        if (rc != 0) {
            fprintf(stderr, "pthread_join failed for thread %d (code %d)\n", t, rc);
            return 1;
        }
    }

    printf("[Pthreads] All worker threads joined and reclaimed.\n");
    return 0;
}
```

---

### 1.4 Technical Comparison
- **Code Verbosity:** OpenMP requires **4 lines** of parallel code; Pthreads requires **~35 lines** (struct definition, heap allocation, loop creation, error checking, explicit join loop, and memory freeing).
- **Thread Lifecycle Cost:** OpenMP maintains a persistent thread pool across parallel regions, avoiding repeated OS kernel thread destruction; Pthreads creates and tears down OS kernel thread structures unless a custom thread pool is written.
- **Safety Hazards:** In Pthreads, passing `&t` directly to `pthread_create` is a fatal bug (race condition on pointer). OpenMP handles thread indexing internally without pointers.

---

## Pattern 2: Work-Sharing & Parallel For Loops

### 2.1 Objective
Distribute $N$ iterations of a computationally intensive loop across $T$ threads with static chunking.

---

### 2.2 OpenMP Implementation
```c
/*
 * openmp_for.c
 * Loop parallelization using OpenMP #pragma omp for.
 *
 * Compilation: gcc -O3 -fopenmp openmp_for.c -o openmp_for
 */
#include <stdio.h>
#include <omp.h>

#define N 1000

int main(void) {
    int a[N], b[N], c[N];
    for (int i = 0; i < N; i++) { a[i] = i; b[i] = i * 2; }

    #pragma omp parallel for default(none) shared(a, b, c) schedule(static, 250)
    for (int i = 0; i < N; i++) {
        c[i] = a[i] + b[i];
    }

    printf("[OpenMP] Parallel for loop completed. c[500] = %d\n", c[500]);
    return 0;
}
```

---

### 2.3 POSIX Threads Implementation
```c
/*
 * pthreads_for.c
 * Manual loop index partitioning and execution using Pthreads.
 *
 * Compilation: gcc -O3 -pthread pthreads_for.c -o pthreads_for
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define N 1000
#define NUM_THREADS 4

int a[N], b[N], c[N];

typedef struct {
    int start_idx;
    int end_idx;
} LoopChunk;

void *loop_worker(void *arg) {
    LoopChunk *chunk = (LoopChunk *)arg;
    for (int i = chunk->start_idx; i < chunk->end_idx; i++) {
        c[i] = a[i] + b[i];
    }
    free(chunk);
    pthread_exit(NULL);
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    for (int i = 0; i < N; i++) { a[i] = i; b[i] = i * 2; }

    int chunk_size = N / NUM_THREADS;

    /* Manual index decomposition */
    for (int t = 0; t < NUM_THREADS; t++) {
        LoopChunk *chunk = (LoopChunk *)malloc(sizeof(LoopChunk));
        chunk->start_idx = t * chunk_size;
        chunk->end_idx = (t == NUM_THREADS - 1) ? N : (t + 1) * chunk_size;

        pthread_create(&threads[t], NULL, loop_worker, (void *)chunk);
    }

    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_join(threads[t], NULL);
    }

    printf("[Pthreads] Manual parallel loop completed. c[500] = %d\n", c[500]);
    return 0;
}
```

---

### 2.4 Technical Comparison
- **Loop Canonical Constraints:** OpenMP requires canonical `for` loops (`for (i = 0; i < N; i++)`) so the runtime can pre-calculate the trip count. Pthreads accepts any arbitrary looping structure or `while` loop.
- **Scheduling Flexibility:** Switching from `static` to `dynamic` or `guided` in OpenMP requires changing one word: `schedule(dynamic, 64)`. In Pthreads, dynamic scheduling requires building a synchronized shared work queue with mutexes and condition variables.

---

## Pattern 3: Critical Sections (Mutual Exclusion)

### 3.1 Objective
Protect a shared memory location from concurrent conflicting writes across multiple threads.

---

### 3.2 OpenMP Implementation
```c
/*
 * openmp_critical.c
 * Critical section in OpenMP.
 *
 * Compilation: gcc -O2 -fopenmp openmp_critical.c -o openmp_critical
 */
#include <stdio.h>
#include <omp.h>

long shared_balance = 1000;

void deposit(long amount) {
    /* Enforces that only one thread executes this structured block at a time */
    #pragma omp critical(bank_account_lock)
    {
        shared_balance += amount;
    }
}

int main(void) {
    #pragma omp parallel for default(none) shared(shared_balance)
    for (int i = 0; i < 10000; i++) {
        deposit(10);
    }
    printf("[OpenMP] Final Balance: %ld (Expected: 101000)\n", shared_balance);
    return 0;
}
```

---

### 3.3 POSIX Threads Implementation
```c
/*
 * pthreads_mutex.c
 * Mutual exclusion using pthread_mutex_t.
 *
 * Compilation: gcc -O2 -pthread pthreads_mutex.c -o pthreads_mutex
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 4
#define OPS_PER_THREAD 2500

long shared_balance = 1000;
pthread_mutex_t balance_mutex;

void deposit(long amount) {
    pthread_mutex_lock(&balance_mutex);
    shared_balance += amount;
    pthread_mutex_unlock(&balance_mutex);
}

void *worker(void *arg) {
    for (int i = 0; i < OPS_PER_THREAD; i++) {
        deposit(10);
    }
    pthread_exit(NULL);
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    pthread_mutex_init(&balance_mutex, NULL);

    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_create(&threads[t], NULL, worker, NULL);
    }
    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_join(threads[t], NULL);
    }

    pthread_mutex_destroy(&balance_mutex);
    printf("[Pthreads] Final Balance: %ld (Expected: 101000)\n", shared_balance);
    return 0;
}
```

---

### 3.4 Technical Comparison
- **Explicit Lifecycle:** Pthreads requires explicit initialization (`pthread_mutex_init`) and destruction (`pthread_mutex_destroy`). Forgetting destruction leaks operating system kernel mutex structures. OpenMP automatically initializes and finalizes named critical sections.
- **Deadlock Risk:** If an exception, `return`, or `goto` exits a Pthreads critical section without calling `pthread_mutex_unlock()`, the mutex remains permanently locked (causing global deadlock). OpenMP `#pragma omp critical` is lexically scoped; exiting the structured block automatically releases the lock.

---

## Pattern 4: Atomic Operations

### 4.1 Objective
Execute a single read-modify-write arithmetic update (such as `counter++` or `sum += val`) with minimal synchronization latency by bypassing heavyweight operating system mutexes and using CPU hardware bus locks.

---

### 4.2 OpenMP Implementation
```c
/*
 * openmp_atomic.c
 * Hardware atomic increment in OpenMP.
 *
 * Compilation: gcc -O3 -fopenmp openmp_atomic.c -o openmp_atomic
 */
#include <stdio.h>
#include <omp.h>

long counter = 0;

int main(void) {
    #pragma omp parallel for default(none) shared(counter)
    for (int i = 0; i < 1000000; i++) {
        /* Emits CPU hardware bus lock (e.g., LOCK XADD) */
        #pragma omp atomic
        counter++;
    }

    printf("[OpenMP Atomic] Counter: %ld (Expected: 1000000)\n", counter);
    return 0;
}
```

---

### 4.3 POSIX Threads Implementation (C11 Stdatomic / GCC Built-in)
*Note: Pthreads does not define an atomic API; standard POSIX concurrent programming relies on C11 `<stdatomic.h>` or compiler built-ins.*

```c
/*
 * pthreads_atomic.c
 * Atomic operations in POSIX C using C11 stdatomic.
 *
 * Compilation: gcc -O3 -pthread pthreads_atomic.c -o pthreads_atomic
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <stdatomic.h>

#define NUM_THREADS 4
#define OPS_PER_THREAD 250000

atomic_long counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < OPS_PER_THREAD; i++) {
        /* Atomic hardware operation with sequential consistency */
        atomic_fetch_add(&counter, 1);
    }
    pthread_exit(NULL);
}

int main(void) {
    pthread_t threads[NUM_THREADS];

    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_create(&threads[t], NULL, worker, NULL);
    }
    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_join(threads[t], NULL);
    }

    printf("[Pthreads Atomic] Counter: %ld (Expected: 1000000)\n", atomic_load(&counter));
    return 0;
}
```

---

### 4.4 Technical Comparison
- **Performance:** Both compile to identical hardware assembly instructions (`lock addq` on x86-64).
- **Expressiveness:** OpenMP `#pragma omp atomic` supports `read`, `write`, `update`, and `capture`. In pure Pthreads, if C11 atomics are unavailable on legacy platforms, you must fall back to a full `pthread_mutex_t`, which is **10x–20x slower**.

---

## Pattern 5: Parallel Reduction (Array Sum)

### 5.1 Objective
Sum an array of $N$ double-precision numbers in parallel, avoiding race conditions and cache-line false sharing.

---

### 5.2 OpenMP Implementation
```c
/*
 * openmp_reduction.c
 * Lock-free tree reduction in OpenMP.
 *
 * Compilation: gcc -O3 -fopenmp openmp_reduction.c -o openmp_reduction
 */
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define N 10000000

int main(void) {
    double *A = (double *)malloc(N * sizeof(double));
    for (int i = 0; i < N; i++) A[i] = 1.0;

    double total_sum = 0.0;

    /* OpenMP allocates thread-private accumulators, then tree-reduces */
    #pragma omp parallel for default(none) shared(A) reduction(+:total_sum)
    for (int i = 0; i < N; i++) {
        total_sum += A[i];
    }

    printf("[OpenMP Reduction] Sum: %.2f (Expected: %.2f)\n", total_sum, (double)N);
    free(A);
    return 0;
}
```

---

### 5.3 POSIX Threads Implementation
```c
/*
 * pthreads_reduction.c
 * Manual parallel reduction using thread-local accumulators and return values.
 *
 * Compilation: gcc -O3 -pthread pthreads_reduction.c -o pthreads_reduction
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define N 10000000
#define NUM_THREADS 4

double *A;

typedef struct {
    int start;
    int end;
} Chunk;

void *reduction_worker(void *arg) {
    Chunk *c = (Chunk *)arg;
    
    /* Allocate thread-local accumulator on heap to return to main thread */
    double *local_sum = (double *)malloc(sizeof(double));
    *local_sum = 0.0;

    for (int i = c->start; i < c->end; i++) {
        *local_sum += A[i];
    }

    free(c);
    pthread_exit((void *)local_sum);
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    A = (double *)malloc(N * sizeof(double));
    for (int i = 0; i < N; i++) A[i] = 1.0;

    int chunk_size = N / NUM_THREADS;

    for (int t = 0; t < NUM_THREADS; t++) {
        Chunk *c = (Chunk *)malloc(sizeof(Chunk));
        c->start = t * chunk_size;
        c->end = (t == NUM_THREADS - 1) ? N : (t + 1) * chunk_size;
        pthread_create(&threads[t], NULL, reduction_worker, (void *)c);
    }

    double total_sum = 0.0;
    for (int t = 0; t < NUM_THREADS; t++) {
        void *res;
        pthread_join(threads[t], &res);
        double *partial = (double *)res;
        total_sum += *partial;
        free(partial);
    }

    printf("[Pthreads Reduction] Sum: %.2f (Expected: %.2f)\n", total_sum, (double)N);
    free(A);
    return 0;
}
```

---

### 5.4 Technical Comparison
- **False Sharing Immunity:** OpenMP's `reduction(+:var)` allocates temporary accumulators on each thread's stack (or registers), guaranteeing zero false sharing. In Pthreads, naive programmers often create a global array `double partial[NUM_THREADS]`, inducing catastrophic cache-line bouncing unless padded to 64 bytes.
- **Tree Reduction Optimization:** OpenMP uses a logarithmic $O(\log_2 T)$ reduction tree to combine values; manual Pthreads implementations typically sum sequentially in `main()` ($O(T)$).

---

## Pattern 6: Barrier Synchronization (Phase Alignment)

### 6.1 Objective
Force all threads to arrive at a common execution point before any thread is allowed to enter the subsequent computational phase.

---

### 6.2 OpenMP Implementation
```c
/*
 * openmp_barrier.c
 * Barrier synchronization in OpenMP.
 *
 * Compilation: gcc -O2 -fopenmp openmp_barrier.c -o openmp_barrier
 */
#include <stdio.h>
#include <omp.h>

int main(void) {
    #pragma omp parallel num_threads(4) default(none)
    {
        int tid = omp_get_thread_num();

        printf("Phase 1: Thread %d working...\n", tid);

        /* All threads must reach here before any proceed */
        #pragma omp barrier

        #pragma omp single
        printf("--- Global Checkpoint: Phase 1 Complete ---\n");

        printf("Phase 2: Thread %d advancing...\n", tid);
    }
    return 0;
}
```

---

### 6.3 POSIX Threads Implementation
```c
/*
 * pthreads_barrier.c
 * Barrier synchronization using pthread_barrier_t.
 *
 * Compilation: gcc -O2 -pthread pthreads_barrier.c -o pthreads_barrier
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 4

pthread_barrier_t phase_barrier;

void *worker(void *arg) {
    int tid = *(int *)arg;
    free(arg);

    printf("Phase 1: Thread %d working...\n", tid);

    /* Block until NUM_THREADS distinct threads invoke this call */
    int rc = pthread_barrier_wait(&phase_barrier);
    if (rc == PTHREAD_BARRIER_SERIAL_THREAD) {
        printf("--- Global Checkpoint: Phase 1 Complete ---\n");
    }

    printf("Phase 2: Thread %d advancing...\n", tid);
    pthread_exit(NULL);
}

int main(void) {
    pthread_t threads[NUM_THREADS];
    pthread_barrier_init(&phase_barrier, NULL, NUM_THREADS);

    for (int t = 0; t < NUM_THREADS; t++) {
        int *tid = (int *)malloc(sizeof(int));
        *tid = t;
        pthread_create(&threads[t], NULL, worker, (void *)tid);
    }
    for (int t = 0; t < NUM_THREADS; t++) {
        pthread_join(threads[t], NULL);
    }

    pthread_barrier_destroy(&phase_barrier);
    return 0;
}
```

---

### 6.4 Technical Comparison
- **Serial Thread Designation:** `pthread_barrier_wait()` returns `PTHREAD_BARRIER_SERIAL_THREAD` to exactly one arbitrary thread, allowing convenient single-thread master execution without an extra mutex. OpenMP achieves this via `#pragma omp single`.
- **Implicit vs. Explicit:** OpenMP work-sharing constructs (`#pragma omp for`, `single`, `sections`) have an **implicit barrier** at their closing brace (unless suppressed with `nowait`). In Pthreads, barriers are always explicit.

---

## Pattern 7: Thread Coordination & Event Signaling

### 7.1 Objective
Put a thread to sleep until a specific application condition becomes true, signaled by another thread.

---

### 7.2 OpenMP Implementation (Locks / Tasks)
*OpenMP does not have condition variables.* Coordination is traditionally implemented via low-level locks (`omp_lock_t`), custom spin-loops with `#pragma omp flush`, or task dependencies (`depend(in/out)`).

```c
/*
 * openmp_lock_signal.c
 * Coordination using OpenMP locks.
 *
 * Compilation: gcc -O2 -fopenmp openmp_lock_signal.c -o openmp_lock_signal
 */
#include <stdio.h>
#include <omp.h>

int main(void) {
    omp_lock_t lock;
    omp_init_lock(&lock);
    omp_set_lock(&lock); /* Master locks it */

    #pragma omp parallel num_threads(2) default(none) shared(lock)
    {
        int tid = omp_get_thread_num();
        if (tid == 1) {
            printf("[Worker 1] Waiting for signal...\n");
            omp_set_lock(&lock); /* Blocks until Thread 0 unsets it */
            printf("[Worker 1] Signal received! Executing task.\n");
            omp_unset_lock(&lock);
        } else if (tid == 0) {
            printf("[Worker 0] Preparing data...\n");
            printf("[Worker 0] Signaling Worker 1.\n");
            omp_unset_lock(&lock); /* Releases lock, unblocking Thread 1 */
        }
    }

    omp_destroy_lock(&lock);
    return 0;
}
```

---

### 7.3 POSIX Threads Implementation (Condition Variables)
*Pthreads provides full condition variable primitives (`pthread_cond_t`).*

```c
/*
 * pthreads_condvar_signal.c
 * Precise event signaling using pthread_cond_t.
 *
 * Compilation: gcc -O2 -pthread pthreads_condvar_signal.c -o pthreads_condvar_signal
 */
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int ready = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

void *worker_wait(void *arg) {
    pthread_mutex_lock(&mutex);
    printf("[Wait Thread] Sleeping until ready flag is set...\n");

    /* Mandatory while loop to handle spurious wakeups */
    while (!ready) {
        pthread_cond_wait(&cond, &mutex); /* Atomically unlocks mutex and sleeps */
    }

    printf("[Wait Thread] Awakened! Condition met (ready = %d).\n", ready);
    pthread_mutex_unlock(&mutex);
    pthread_exit(NULL);
}

void *worker_signal(void *arg) {
    usleep(50000); /* Simulate work */
    pthread_mutex_lock(&mutex);
    ready = 1;
    printf("[Signal Thread] State updated. Signaling condition variable.\n");
    pthread_cond_signal(&cond); /* Wake up one waiting thread */
    pthread_mutex_unlock(&mutex);
    pthread_exit(NULL);
}

int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, worker_wait, NULL);
    pthread_create(&t2, NULL, worker_signal, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    return 0;
}
```

---

### 7.4 Technical Comparison
- **Condition Support:** Pthreads is overwhelmingly superior for state-based event waiting. A sleeping thread in `pthread_cond_wait` consumes **0% CPU** inside an OS wait queue.
- **OpenMP Limitations:** Emulating condition variables in OpenMP requires busy-waiting or lock gymnastics that waste CPU cycles and introduce potential priority inversion.

---

## Pattern 8: Producer–Consumer Bounded Buffer

### 8.1 Objective
Implement a thread-safe circular bounded buffer where producer threads wait when the buffer is full, and consumer threads wait when the buffer is empty.

---

### 8.2 OpenMP Implementation (Work-Sharing Sections & Locks)
```c
/*
 * openmp_producer_consumer.c
 * Producer-Consumer pipeline using OpenMP sections and locks.
 *
 * Compilation: gcc -O2 -fopenmp openmp_producer_consumer.c -o openmp_prod_cons
 */
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define CAPACITY 4
#define TOTAL_ITEMS 10

int queue[CAPACITY];
int count = 0, in = 0, out = 0;
omp_lock_t q_lock;

void produce(int val) {
    int placed = 0;
    while (!placed) {
        omp_set_lock(&q_lock);
        if (count < CAPACITY) {
            queue[in] = val;
            in = (in + 1) % CAPACITY;
            count++;
            printf("[Producer] Produced: %d (Queue: %d/%d)\n", val, count, CAPACITY);
            placed = 1;
        }
        omp_unset_lock(&q_lock);
    }
}

int consume(void) {
    int val = -1;
    int retrieved = 0;
    while (!retrieved) {
        omp_set_lock(&q_lock);
        if (count > 0) {
            val = queue[out];
            out = (out + 1) % CAPACITY;
            count--;
            printf("  [Consumer] Consumed: %d (Queue: %d/%d)\n", val, count, CAPACITY);
            retrieved = 1;
        }
        omp_unset_lock(&q_lock);
    }
    return val;
}

int main(void) {
    omp_init_lock(&q_lock);

    #pragma omp parallel sections num_threads(2) default(none)
    {
        #pragma omp section
        {
            for (int i = 1; i <= TOTAL_ITEMS; i++) produce(i);
        }
        #pragma omp section
        {
            for (int i = 1; i <= TOTAL_ITEMS; i++) consume();
        }
    }

    omp_destroy_lock(&q_lock);
    return 0;
}
```

---

### 8.3 POSIX Threads Implementation (Semaphores)
```c
/*
 * pthreads_producer_consumer.c
 * Optimal Producer-Consumer bounded buffer using POSIX counting semaphores.
 *
 * Compilation: gcc -O2 -pthread pthreads_producer_consumer.c -o pthreads_prod_cons
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>

#define CAPACITY 4
#define TOTAL_ITEMS 10

int queue[CAPACITY];
int in = 0, out = 0;

sem_t empty_slots; /* Initialized to CAPACITY */
sem_t full_slots;  /* Initialized to 0 */
pthread_mutex_t q_mutex = PTHREAD_MUTEX_INITIALIZER;

void *producer(void *arg) {
    for (int i = 1; i <= TOTAL_ITEMS; i++) {
        sem_wait(&empty_slots); /* Blocks without CPU spinning if full */
        pthread_mutex_lock(&q_mutex);

        queue[in] = i;
        in = (in + 1) % CAPACITY;
        printf("[Producer] Produced: %d\n", i);

        pthread_mutex_unlock(&q_mutex);
        sem_post(&full_slots); /* Wake waiting consumer */
    }
    pthread_exit(NULL);
}

void *consumer(void *arg) {
    for (int i = 1; i <= TOTAL_ITEMS; i++) {
        sem_wait(&full_slots); /* Blocks without CPU spinning if empty */
        pthread_mutex_lock(&q_mutex);

        int val = queue[out];
        out = (out + 1) % CAPACITY;
        printf("  [Consumer] Consumed: %d\n", val);

        pthread_mutex_unlock(&q_mutex);
        sem_post(&empty_slots); /* Wake waiting producer */
    }
    pthread_exit(NULL);
}

int main(void) {
    pthread_t prod, cons;

    sem_init(&empty_slots, 0, CAPACITY);
    sem_init(&full_slots, 0, 0);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty_slots);
    sem_destroy(&full_slots);
    return 0;
}
```

---

### 8.4 Technical Comparison
- **Spinning vs. Sleeping:** The OpenMP implementation relies on polling `while (!placed)`, causing CPU spinlock churn when the queue is saturated. The POSIX semaphore solution suspends waiting threads directly in OS kernel queues, achieving **zero wasted CPU clock cycles**.
- **Asymmetric Workflows:** Pthreads is naturally suited for heterogeneous Producer-Consumer pipelines. OpenMP was engineered primarily for homogeneous data-parallel loops.

---

## Pattern 9: Read-Write Locks for Read-Heavy Workloads

### 9.1 Objective
Permit multiple threads to read a shared database concurrently, but grant exclusive access to writers.

---

### 9.2 OpenMP Implementation
*OpenMP does not define native reader-writer locks.* All critical sections default to exclusive mutual exclusion, forcing readers to execute sequentially!

```c
/*
 * openmp_exclusive.c
 * OpenMP lacks native reader-writer locks; all readers are serialized.
 *
 * Compilation: gcc -O2 -fopenmp openmp_exclusive.c -o openmp_exclusive
 */
#include <stdio.h>
#include <omp.h>

int shared_data = 100;

int main(void) {
    #pragma omp parallel num_threads(4) default(none) shared(shared_data)
    {
        /* Readers are forced to serialize! Heavy lock contention. */
        #pragma omp critical(db_lock)
        {
            printf("[Reader Thread %d] Data = %d\n", omp_get_thread_num(), shared_data);
        }
    }
    return 0;
}
```

---

### 9.3 POSIX Threads Implementation (`pthread_rwlock_t`)
```c
/*
 * pthreads_rwlock.c
 * Native Read-Write Lock synchronization.
 *
 * Compilation: gcc -O2 -pthread pthreads_rwlock.c -o pthreads_rwlock
 */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

int shared_data = 100;
pthread_rwlock_t db_rwlock = PTHREAD_RWLOCK_INITIALIZER;

void *reader(void *arg) {
    int id = *(int *)arg;
    free(arg);

    /* Shared read lock: multiple readers execute simultaneously! */
    pthread_rwlock_rdlock(&db_rwlock);
    printf("[Reader %d] Reading shared_data = %d\n", id, shared_data);
    usleep(10000);
    pthread_rwlock_unlock(&db_rwlock);

    pthread_exit(NULL);
}

void *writer(void *arg) {
    /* Exclusive write lock: blocks all readers and other writers */
    pthread_rwlock_wrlock(&db_rwlock);
    shared_data += 50;
    printf("  *** [WRITER] Updated shared_data to %d ***\n", shared_data);
    pthread_rwlock_unlock(&db_rwlock);

    pthread_exit(NULL);
}

int main(void) {
    pthread_t r[4], w;
    pthread_create(&w, NULL, writer, NULL);

    for (int i = 0; i < 4; i++) {
        int *id = (int *)malloc(sizeof(int));
        *id = i;
        pthread_create(&r[i], NULL, reader, (void *)id);
    }

    pthread_join(w, NULL);
    for (int i = 0; i < 4; i++) pthread_join(r[i], NULL);

    pthread_rwlock_destroy(&db_rwlock);
    return 0;
}
```

---

### 9.4 Technical Comparison
- **Read Throughput:** In read-heavy environments (95% reads, 5% writes), Pthreads `pthread_rwlock_t` achieves up to **$8\times - 16\times$ higher throughput** than OpenMP's exclusive critical sections by allowing parallel cache reads without cache-line invalidation.

---

## Pattern 10: Dynamic Asynchronous Tasking

### 10.1 Objective
Execute a recursive divide-and-conquer algorithm (e.g., recursive Fibonacci or Quicksort) by spawning dynamic asynchronous tasks.

---

### 10.2 OpenMP Implementation (`#pragma omp task`)
```c
/*
 * openmp_tasks_fib.c
 * Recursive Fibonacci using OpenMP Tasks.
 *
 * Compilation: gcc -O3 -fopenmp openmp_tasks_fib.c -o openmp_fib
 */
#include <stdio.h>
#include <omp.h>

long fib(int n) {
    if (n < 2) return n;
    long x, y;

    #pragma omp task shared(x) if(n > 20)
    x = fib(n - 1);

    #pragma omp task shared(y) if(n > 20)
    y = fib(n - 2);

    #pragma omp taskwait /* Barrier waiting for both child tasks */
    return x + y;
}

int main(void) {
    int n = 35;
    long result;

    #pragma omp parallel num_threads(4) default(none) shared(n, result)
    {
        #pragma omp single
        result = fib(n);
    }

    printf("[OpenMP Tasks] Fib(%d) = %ld\n", n, result);
    return 0;
}
```

---

### 10.3 POSIX Threads Implementation
*In Pthreads, there is no language or library tasking abstraction.* Attempting to call `pthread_create()` for every recursive call results in millions of operating system threads, crashing the kernel (`EAGAIN: Resource temporarily unavailable`).
Implementing this in Pthreads requires manually coding a **Thread Pool**, a thread-safe **Task Queue**, a work-stealing deque, and condition-variable synchronizers (requiring **over 250 lines of complex C code**).

---

### 10.4 Technical Comparison
- **Task Parallelism Winner:** OpenMP is the undisputed winner for task-based divide-and-conquer parallelism. Its runtime automatically manages dynamic work-stealing thread pools, task queues, and graph dependencies with negligible developer effort.

---

## 12. Master Architectural Comparison Matrix

| Comparison Dimension | OpenMP (v4.5 / 5.2) | POSIX Threads (Pthreads / IEEE 1003.1c) |
| :--- | :--- | :--- |
| **Language Abstraction** | High-level compiler pragmas (`#pragma omp`). | Low-level procedural C API (`<pthread.h>`). |
| **Primary Domain** | High-Performance Computing, numerical grids, linear algebra. | Systems programming, daemon servers, graphical user interfaces. |
| **Work Distribution** | Automated loop chunking (`static`, `dynamic`, `guided`). | Manual programmer arithmetic on loop start/end indices. |
| **Thread Lifecycle** | Implicit persistent thread pool; amortized creation cost. | Explicit `pthread_create()` and `pthread_join()` per thread. |
| **Data Scoping** | Built-in directives (`shared`, `private`, `firstprivate`). | Manual memory allocation (stack vs. heap structs). |
| **Reductions** | Native, lock-free, tree-reduced (`reduction(+:var)`). | Manual implementation via locks or thread return values. |
| **Event Coordination** | Weak (locks, spinlocks, or tasks only). | **Superior** (`pthread_cond_t` condition variables). |
| **Counting Semaphores**| None. | **Native** (`<semaphore.h>`, `sem_init`, `sem_wait`). |
| **Reader-Writer Locks**| None (all critical sections are exclusive). | **Native** (`pthread_rwlock_t`). |
| **Task Parallelism** | **Native** (`#pragma omp task`, `taskloop`). | Manual (Requires building a complete custom thread pool). |
| **SIMD Vectorization** | **Native** (`#pragma omp simd`). | None (Must use compiler auto-vectorizer or intrinsics). |
| **Code Verbosity** | Ultra-concise (1–4 lines of pragmas). | Highly verbose (30–60 lines of boilerplate C). |
| **Portability** | Requires an OpenMP-compliant compiler. | Standard across all POSIX-compliant Unix/Linux systems. |

---

## 13. Engineering Decision Matrix: When to Choose Which?

```
                                  START
                                    |
                    Is the problem loop-based or
                    regular numerical computing?
                               /         \
                             YES          NO
                             /              \
                     Use [ OpenMP ]          Is it an irregular pipeline,
                                             event listener, or server?
                                             /               \
                                           YES                NO
                                           /                    \
                                   Use [ Pthreads ]        Does it use dynamic
                                                           recursive tasking?
                                                           /                \
                                                         YES                 NO
                                                         /                     \
                                                 Use [ OpenMP Tasks ]     Use [ OpenMP ]
```

### Choose OpenMP When:
1. You are parallelizing for-loops, matrix multiplications, stencils, or numerical simulations.
2. You need lock-free array reductions without managing thread stacks.
3. You want hardware vectorization (`#pragma omp simd`) coupled with multi-threading.
4. You are writing recursive divide-and-conquer algorithms that benefit from automatic runtime task queues.
5. You want to maintain a single codebase that compiles serially on compilers lacking parallel flags.

### Choose POSIX Threads When:
1. You are developing systems software, network servers, web daemons, or database engines.
2. You require complex thread coordination using Condition Variables (`pthread_cond_wait`).
3. You need Producer-Consumer bounded buffers controlled by Counting Semaphores.
4. Your application is read-heavy and requires concurrent reader execution via Read-Write Locks (`pthread_rwlock_t`).
5. You require fine-grained operating system control over thread priorities, stack sizes, and real-time scheduling policies (`SCHED_FIFO`, `SCHED_RR`).

---
*End of Master OpenMP vs. POSIX Threads Comparative Code Guide.*
