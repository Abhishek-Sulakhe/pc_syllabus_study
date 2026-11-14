# Last-Minute Revision & High-Yield Cramming Sheet

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 1–32)  
**Applicability:** Night-Before Exam Review, Rapid Formula Recall, and Viva Voce Prep  
**Purpose:** High-density, high-yield summary of critical concepts, common exam traps, syntax cheat sheets, and instant formula recall.

---

## 1. Top 10 Fatal Exam Traps & Common Mistakes

### Trap 1: The Pthreads Loop Index Pointer Bug
```c
/* FATAL BUG: Passing the address of the loop variable! */
for (int i = 0; i < N; i++) {
    pthread_create(&threads[i], NULL, worker, (void *)&i); /* ALL THREADS SHARE &i */
}
```
> **The Fix:** Allocate a dedicated struct on the heap per thread (`malloc(sizeof(ThreadArg))`), or cast the integer directly by value if word-sized: `(void *)(intptr_t)i`.

---

### Trap 2: `pthread_cond_wait` inside an `if` Statement
```c
/* FATAL BUG: Vulnerable to Spurious Wakeups and Stolen State! */
pthread_mutex_lock(&mutex);
if (buffer_count == 0) {
    pthread_cond_wait(&cond, &mutex);
}
```
> **The Fix:** ALWAYS use a `while` loop:
> ```c
> while (buffer_count == 0) {
>     pthread_cond_wait(&cond, &mutex);
> }
> ```

---

### Trap 3: Silent OpenMP Serialization (Missing `-fopenmp`)
Compiling with `mpicc -O3 code.c -o app` ignores all `#pragma omp` directives without warning!
> **The Fix:** Always verify compiler flags: `mpicc -O3 -fopenmp code.c -o app`. Use `#ifdef _OPENMP` guards inside the code to detect OpenMP support at compile time.

---

### Trap 4: Circular Wait Deadlock in MPI Point-to-Point
If all ranks execute `MPI_Recv` before `MPI_Send`, or if all ranks execute blocking `MPI_Send` simultaneously for large messages exceeding eager buffer limits, every process blocks forever waiting for its partner.
> **The Fix:** Break communication symmetry (e.g., even ranks send first then receive; odd ranks receive first then send), or replace with non-blocking `MPI_Isend` / `MPI_Irecv` followed by `MPI_Waitall`.

---

### Trap 5: False Sharing on Multi-Core Processors
Writing to contiguous array indices (`sum[tid]++`) where elements share a single 64-byte cache line causes the MESI protocol to bounce the cache line between cores.
> **The Fix:** Accumulate inside a thread-private stack variable (`double local_sum = 0.0`), and perform a single update at the end, or use the OpenMP `reduction(+:sum)` clause.

---

### Trap 6: Missing `MPI_THREAD_MULTIPLE` in Hybrid Codes
Spawning OpenMP worker threads that issue concurrent MPI communication calls when the runtime was initialized with `MPI_Init()` or `MPI_THREAD_FUNNELED`.
> **The Fix:** Request `MPI_THREAD_MULTIPLE` via `MPI_Init_thread(&argc, &argv, MPI_THREAD_MULTIPLE, &provided)` and verify that `provided >= MPI_THREAD_MULTIPLE`.

---

### Trap 7: Misunderstanding Amdahl vs. Gustafson Scaling
- **Amdahl's Law:** Fixed problem size (Strong Scaling). Speedup has an asymptotic ceiling of $\frac{1}{s}$.
- **Gustafson's Law:** Scaled problem size (Weak Scaling). Speedup scales linearly: $s + (1 - s)N$.

---

### Trap 8: Assuming `MPI_Barrier` Exchanges Application Data
`MPI_Barrier` transfers **0 bytes** of application data. It provides purely temporal execution synchronization.

---

### Trap 9: Core Over-Subscription in Hybrid MPI + OpenMP
Submitting an MPI job on a 32-core node with 16 MPI ranks and `OMP_NUM_THREADS=32` launches $16 \times 32 = 512$ threads on 32 cores, resulting in catastrophic context-switching penalties.
> **The Fix:** Always enforce: $\text{Ranks per Node } (M) \times \text{Threads per Rank } (T) = \text{Physical Cores } (C)$.

---

### Trap 10: Modifying Data in Non-Blocking MPI Buffers before `MPI_Wait`
Modifying or reading an application buffer after calling `MPI_Isend` or `MPI_Irecv` before `MPI_Wait()` returns produces race conditions and memory corruption.

---

## 2. Master Syntax Cheat Sheet

### 2.1 OpenMP Quick Reference (C/C++)
| Construct / Clause | Exact Syntax | Description |
| :--- | :--- | :--- |
| **Parallel Region** | `#pragma omp parallel num_threads(T)` | Spawns a team of $T$ threads executing the block. |
| **Loop Work-Sharing** | `#pragma omp for schedule(static, C)` | Divides loop iterations among threads. |
| **Combined Loop** | `#pragma omp parallel for reduction(+:sum)` | Spawns threads, splits loop, and safely reduces sum. |
| **Sections** | `#pragma omp sections` / `#pragma omp section`| Divides discrete code blocks among threads. |
| **Single Execution** | `#pragma omp single nowait` | Executed by 1 arbitrary thread; no barrier if `nowait`. |
| **Critical Section** | `#pragma omp critical (optional_name)` | Software mutual exclusion (one thread at a time). |
| **Atomic Update** | `#pragma omp atomic` | Hardware-level atomic read-modify-write. |
| **Barrier** | `#pragma omp barrier` | Explicit barrier; all threads must arrive before any proceed. |
| **Task Creation** | `#pragma omp task default(none) shared(A)` | Packages code block as an asynchronous task. |
| **Task Synchronization**| `#pragma omp taskwait` | Waits for all child tasks generated by current task. |
| **SIMD Vectorization**| `#pragma omp simd aligned(ptr:64)` | Guides vectorizer without spawning threads. |

---

### 2.2 POSIX Threads Quick Reference (`<pthread.h>`)
```c
/* Thread Management */
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);
int pthread_join(pthread_t thread, void **retval);
int pthread_detach(pthread_t thread);
void pthread_exit(void *retval);

/* Mutexes */
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
int pthread_mutex_destroy(pthread_mutex_t *mutex);

/* Condition Variables */
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex); /* ATOMIC RELEASE & SLEEP */
int pthread_cond_signal(pthread_cond_t *cond);                        /* WAKES ONE WAITER */
int pthread_cond_broadcast(pthread_cond_t *cond);                     /* WAKES ALL WAITERS */
int pthread_cond_destroy(pthread_cond_t *cond);

/* Barriers */
int pthread_barrier_init(pthread_barrier_t *barrier, const pthread_barrierattr_t *attr, unsigned int count);
int pthread_barrier_wait(pthread_barrier_t *barrier); /* Returns PTHREAD_BARRIER_SERIAL_THREAD to one */
int pthread_barrier_destroy(pthread_barrier_t *barrier);

/* POSIX Semaphores (<semaphore.h>) */
int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_wait(sem_t *sem); /* Decrement (blocks if 0) */
int sem_post(sem_t *sem); /* Increment (awakens waiter) */
int sem_destroy(sem_t *sem);
```

---

### 2.3 MPI Quick Reference (`<mpi.h>`)
```c
/* Environment Setup */
int MPI_Init(int *argc, char ***argv);
int MPI_Init_thread(int *argc, char ***argv, int required, int *provided);
int MPI_Comm_rank(MPI_Comm comm, int *rank);
int MPI_Comm_size(MPI_Comm comm, int *size);
double MPI_Wtime(void);
int MPI_Finalize(void);

/* Blocking Point-to-Point */
int MPI_Send(const void *buf, int count, MPI_Datatype datatype, int dest, int tag, MPI_Comm comm);
int MPI_Recv(void *buf, int count, MPI_Datatype datatype, int source, int tag, MPI_Comm comm, MPI_Status *status);

/* Non-Blocking Point-to-Point */
int MPI_Isend(const void *buf, int count, MPI_Datatype datatype, int dest, int tag, MPI_Comm comm, MPI_Request *request);
int MPI_Irecv(void *buf, int count, MPI_Datatype datatype, int source, int tag, MPI_Comm comm, MPI_Request *request);
int MPI_Wait(MPI_Request *request, MPI_Status *status);
int MPI_Waitall(int count, MPI_Request array_of_requests[], MPI_Status array_of_statuses[]);

/* Collective Operations */
int MPI_Bcast(void *buffer, int count, MPI_Datatype datatype, int root, MPI_Comm comm);
int MPI_Scatter(const void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm);
int MPI_Gather(const void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm);
int MPI_Allgather(const void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int recvcount, MPI_Datatype recvtype, MPI_Comm comm);
int MPI_Reduce(const void *sendbuf, void *recvbuf, int count, MPI_Datatype datatype, MPI_Op op, int root, MPI_Comm comm);
int MPI_Allreduce(const void *sendbuf, void *recvbuf, int count, MPI_Datatype datatype, MPI_Op op, MPI_Comm comm);
int MPI_Barrier(MPI_Comm comm);
```

---

## 3. High-Yield Formula Flashcard Reference

| Concept | Mathematical Formula | Units / Notes |
| :--- | :--- | :--- |
| **Peak FLOPS** | $R_{\text{peak}} = N_{\text{cores}} \times f_{\text{clock}} \times \text{FLOPs/cycle}$ | $\text{GFLOP/s}$ or $\text{PFLOP/s}$ |
| **Linpack Efficiency** | $\eta = \frac{R_{\text{max}}}{R_{\text{peak}}} \times 100\%$ | Percentage ($\%$) |
| **Amdahl's Speedup** | $S(N) = \frac{1}{s + \frac{1 - s}{N}} \quad \xrightarrow{N \to \infty} \quad \frac{1}{s}$ | Strong scaling (Fixed problem) |
| **Gustafson's Speedup**| $S(N) = s + (1 - s)N = N + s(1 - N)$ | Weak scaling (Scaled problem) |
| **Parallel Efficiency**| $\varepsilon(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$ | Ideal = $1.0$ ($100\%$) |
| **Karp-Flatt Metric** | $e = \frac{\frac{1}{S(N)} - \frac{1}{N}}{1 - \frac{1}{N}}$ | Diagnostic: rising $e \implies$ comm overhead |
| **Optimal Cores ($N^*$)**| $N^* = \sqrt{\frac{T(1)}{\alpha}}$ (for linear overhead $c(N) = \alpha N$) | Minimizes execution time |
| **Roofline Balance** | $I_{\text{crit}} = \frac{R_{\text{peak}}}{B_{\text{memory}}}$ | $\text{FLOPs/Byte}$; bounds memory vs compute |
| **Hockney Comm Time** | $T_{\text{comm}}(m) = \alpha + m \cdot \beta = \alpha + \frac{m}{B}$ | $\alpha = \text{latency}$, $\beta = \text{inverse bandwidth}$ |
| **Half-Perf Message** | $n_{1/2} = \frac{\alpha}{\beta} = \alpha \times B$ | Bytes needed to reach $50\%$ wire bandwidth |
| **Binomial Tree Bcast**| $T_{\text{Bcast}} = \lceil \log_2 P \rceil (\alpha + m \cdot \beta)$ | Scalable $O(\log_2 P)$ latency |
| **Hybrid Sizing** | $\text{Ranks/Node } (M) \times \text{Threads/Rank } (T) = \text{Physical Cores } (C)$ | Zero over-subscription rule |

---

## 4. Rapid-Fire Exam Viva Voce (Top 15 Q&A)

1. **Q: What is the difference between ILP and TLP?**  
   *A:* ILP (Instruction-Level Parallelism) executes multiple instructions from a single thread simultaneously via hardware pipelining/superscalar units; TLP (Thread-Level Parallelism) executes multiple distinct threads concurrently across multiple processor cores.

2. **Q: What is the significance of the UC Berkeley "Seven Dwarfs"?**  
   *A:* They define seven fundamental algorithmic computation/communication patterns that represent scientific computing workloads across architectures.

3. **Q: Why does Amdahl's Law impose a limit on speedup?**  
   *A:* Because the sequential portion of code ($s$) cannot be accelerated by adding processors, acting as an asymptotic performance ceiling ($1/s$).

4. **Q: What is False Sharing?**  
   *A:* When multiple cores modify independent variables residing within the same 64-byte cache line, causing the cache coherency protocol to continuously invalidate and ping-pong the line between caches.

5. **Q: What is the difference between `#pragma omp critical` and `atomic`?**  
   *A:* `critical` uses software mutex locks to protect arbitrary blocks of statements; `atomic` emits hardware CPU bus lock instructions to protect a single basic read-modify-write operation.

6. **Q: What happens if you omit `schedule()` in `#pragma omp for`?**  
   *A:* The compiler defaults to `schedule(static)` with a chunk size of $\lceil \text{Iterations} / \text{Threads} \rceil$.

7. **Q: Why is `default(none)` considered best practice?**  
   *A:* It disables implicit variable scoping, forcing the developer to explicitly scope every variable, catching data races at compile time.

8. **Q: What are the two essential requirements of an OpenMP reduction operator?**  
   *A:* Mathematical **Associativity** (for hierarchical grouping) and **Commutativity** (for arbitrary completion ordering).

9. **Q: Why must `pthread_cond_wait()` release the mutex atomically?**  
   *A:* To prevent the "Lost Wakeup Problem," where a signaling thread sends a signal between the mutex unlock and the thread enrolling on the wait queue.

10. **Q: Why must condition variables always be checked in a `while` loop?**  
    *A:* To protect against spurious wakeups (awakening without a signal) and stolen state (another thread consuming the resource before the woken thread acquires the mutex).

11. **Q: What is the difference between Joinable and Detached threads?**  
    *A:* Joinable threads retain their termination status and resources until joined via `pthread_join()`; Detached threads automatically free all OS resources immediately upon termination.

12. **Q: What is an MPI Communicator?**  
    *A:* An opaque handle encapsulating a group of processes with an isolated communication context, preventing message collisions between libraries.

13. **Q: What is the difference between `MPI_Bcast` and `MPI_Scatter`?**  
    *A:* `MPI_Bcast` sends the **identical** full buffer to all processes; `MPI_Scatter` splits a contiguous buffer into $P$ equal pieces and sends a **distinct** piece to each process.

14. **Q: What are the four MPI thread safety levels?**  
    *A:* `MPI_THREAD_SINGLE` (1 thread), `MPI_THREAD_FUNNELED` (master thread calls MPI), `MPI_THREAD_SERIALIZED` (1 thread at a time calls MPI), and `MPI_THREAD_MULTIPLE` (any thread calls MPI concurrently).

15. **Q: Why should you bind threads to cores (`OMP_PROC_BIND=close`) in HPC?**  
    *A:* It prevents operating system thread wandering across cores and sockets, preserving L1/L2 cache locality and eliminating remote NUMA crossbar memory latency penalties.

---

## 5. Exam Day Final Checklist
- [ ] Carry your scientific calculator for Amdahl, Gustafson, Hockney, and Peak FLOPS numericals.
- [ ] In coding questions, remember to include `#include <omp.h>`, `<pthread.h>`, or `<mpi.h>`.
- [ ] Always add compilation commands in exam answers:
  - OpenMP: `gcc -O2 -fopenmp code.c -o app`
  - Pthreads: `gcc -O2 -pthread code.c -o app`
  - MPI: `mpicc -O2 code.c -o app`
  - Hybrid: `mpicc -O2 -fopenmp code.c -o app`
- [ ] Double-check data scoping: mark shared variables and loop indices as private.
- [ ] Remember units in numerical problems: GFLOP/s, PFLOP/s, GB/s, $\mu\text{s}$, and speedup ($\times$).

---
*Best of luck with your examinations! You are fully prepared to excel.*
