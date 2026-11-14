# Master Examination Formula Sheet: Parallel Programming & High-Performance Computing

**Target Course:** Parallel Programming & High-Performance Computing (Lectures 1–32)  
**Applicability:** University Theory Examinations (MidSem & EndSem), Laboratory Exams, and Numerical Problem Solving  
**Notation Standard:** Rigorous KaTeX mathematical typesetting with explicit variable definitions, SI units, architectural assumptions, and step-by-step mathematical derivations.

---

## Table of Contents
1. [HPC Architecture, Microarchitecture & Hardware Metrics (Lectures 1–2)](#1-hpc-architecture-microarchitecture--hardware-metrics-lectures-12)
2. [Parallel Speedup, Efficiency & Scaling Laws (Lectures 3–5)](#2-parallel-speedup-efficiency--scaling-laws-lectures-35)
3. [Parallel Overhead, Imbalance & Analytical Performance Models (Lectures 3–5)](#3-parallel-overhead-imbalance--analytical-performance-models-lectures-35)
4. [OpenMP Loop Scheduling, Vectorization & False Sharing (Lectures 8–10, 15–16)](#4-openmp-loop-scheduling-vectorization--false-sharing-lectures-810-1516)
5. [Distributed Communication & Network Performance Models (Lectures 26–31)](#5-distributed-communication--network-performance-models-lectures-2631)
6. [Collective Communication Complexity & Scalability Models (Lectures 30–31)](#6-collective-communication-complexity--scalability-models-lectures-3031)
7. [Hybrid MPI + OpenMP Hierarchical Core Allocation Models (Lecture 32)](#7-hybrid-mpi--openmp-hierarchical-core-allocation-models-lecture-32)

---

## 1. HPC Architecture, Microarchitecture & Hardware Metrics (Lectures 1–2)

### 1.1 Peak Theoretical Floating-Point Performance ($R_{\text{peak}}$)
The maximum theoretical arithmetic computation rate of a multicore CPU or cluster, assuming all execution units complete maximum fused operations every clock cycle:

$$R_{\text{peak}} = N_{\text{cores}} \times f_{\text{clock}} \times \text{FLOPs/cycle}$$

| Parameter | Meaning | Typical Units |
| :--- | :--- | :--- |
| $N_{\text{cores}}$ | Total number of physical execution cores | Cores |
| $f_{\text{clock}}$ | Sustained base or turbo clock frequency | $\text{Hz}$ or $\text{GHz}$ ($10^9\text{ s}^{-1}$) |
| $\text{FLOPs/cycle}$ | Vector arithmetic capability per core per cycle | $\text{Operations/cycle}$ |

#### Calculating $\text{FLOPs/cycle}$ across Vector Architectures:
- **Scalar Double Precision (64-bit IEEE 754):** $1 \text{ op/cycle}$ (or $2$ with scalar FMA).
- **Intel SSE (128-bit vector):** $\frac{128\text{ bits}}{64\text{ bits}} = 2 \text{ doubles} \times 2 \text{ (FMA)} = 4 \text{ FLOPs/cycle}$.
- **Intel AVX2 (256-bit vector):** $\frac{256\text{ bits}}{64\text{ bits}} = 4 \text{ doubles} \times 2 \text{ (FMA)} \times 2 \text{ ports} = 16 \text{ FLOPs/cycle}$.
- **Intel AVX-512 (512-bit vector):** $\frac{512\text{ bits}}{64\text{ bits}} = 8 \text{ doubles} \times 2 \text{ (FMA)} \times 2 \text{ units} = 32 \text{ FLOPs/cycle}$.

---

### 1.2 High-Performance Linpack (HPL) Efficiency ($\eta_{\text{HPL}}$)
The ratio of sustained Linpack benchmark performance ($R_{\text{max}}$) reported on the TOP500 list to the theoretical hardware peak ($R_{\text{peak}}$):

$$\eta_{\text{HPL}} = \frac{R_{\text{max}}}{R_{\text{peak}}} \times 100\%$$

- **Good Supercomputing Efficiency:** $\eta_{\text{HPL}} \ge 70\%$.
- **Causes of Deficit ($1 - \eta_{\text{HPL}}$):** Inter-node interconnect latency, cache-line misses, memory bus contention, and matrix factorization overhead ($O(N^3)$ compute vs. $O(N^2)$ communication).

---

### 1.3 Theoretical Memory Bandwidth ($B_{\text{theoretical}}$)
The peak theoretical data transfer rate between main DRAM memory and the processor's memory controllers:

$$B_{\text{theoretical}} = N_{\text{channels}} \times W_{\text{bus}} \times f_{\text{data}} \times \text{Transfers/cycle}$$

$$B_{\text{DDR}} = N_{\text{channels}} \times 8\text{ bytes} \times f_{\text{clock}} \times 2 \quad [\text{Bytes/s}]$$

| Parameter | Meaning | Standard Value |
| :--- | :--- | :--- |
| $N_{\text{channels}}$ | Memory channels per CPU socket | 4 to 8 channels |
| $W_{\text{bus}}$ | Standard DIMM bus width | 64 bits = 8 bytes |
| $f_{\text{data}}$ | DDR transfer rate rating | e.g., $3200\text{ MT/s} \implies 3.2 \times 10^9 \text{ transfers/s}$ |

---

### 1.4 Arithmetic Intensity and the Roofline Model
The computational density of an algorithm defining whether performance is memory-bound or compute-bound:

$$I = \frac{\text{Floating-Point Operations Executed (FLOPs)}}{\text{DRAM Memory Traffic (Bytes Transferred)}} \quad \left[\frac{\text{FLOPs}}{\text{Byte}}\right]$$

The Roofline attainable performance $P_{\text{attainable}}$ is given by:

$$P_{\text{attainable}} = \min\left( R_{\text{peak}}, \; I \times B_{\text{memory}} \right) \quad [\text{FLOP/s}]$$

- **Memory-Bound Regime:** $I < \frac{R_{\text{peak}}}{B_{\text{memory}}}$ (Kernel bounded by DRAM bandwidth; increasing CPU frequency yields 0% speedup).
- **Compute-Bound Regime:** $I \ge \frac{R_{\text{peak}}}{B_{\text{memory}}}$ (Kernel bounded by arithmetic execution units; saturated vector registers).

---

### 1.5 Pipeline Speedup ($S_k$)
The ideal speedup achieved by an instruction pipeline of $k$ stages executing $n$ independent instructions compared to unpipelined serial execution:

$$S_k(n) = \frac{T_{\text{unpipelined}}}{T_{\text{pipelined}}} = \frac{n \cdot k \cdot \tau}{(k + n - 1) \cdot \tau} = \frac{n \cdot k}{k + n - 1}$$

Where $\tau$ is the clock period. As the number of instructions $n \to \infty$:

$$\lim_{n \to \infty} S_k(n) = k$$

---

### 1.6 Supercomputer Energy & Power Metrics
- **Power Usage Effectiveness (PUE):**
  $$\text{PUE} = \frac{\text{Total Data Center Energy Consumption}}{\text{IT Equipment Energy Consumption}} \ge 1.0$$
- **Energy Efficiency Metric (Green500 Ranking):**
  $$\text{Energy Efficiency} = \frac{R_{\text{max}} \text{ (FLOP/s)}}{P_{\text{consumed}} \text{ (Watts)}} \quad \left[\frac{\text{GFLOP/s}}{\text{Watt}}\right]$$

---

## 2. Parallel Speedup, Efficiency & Scaling Laws (Lectures 3–5)

### 2.1 Fundamental Speedup and Efficiency
- **Parallel Speedup:**
  $$S(N) = \frac{T(1)}{T(N)}$$
- **Parallel Efficiency:**
  $$\varepsilon(N) = \frac{S(N)}{N} = \frac{T(1)}{N \cdot T(N)}$$
- **Superlinear Speedup Condition:** $S(N) > N \iff \varepsilon(N) > 1.0$ (typically caused by cache-aggregation effects where the combined aggregate L2/L3 caches of $N$ processors hold the entire problem dataset, eliminating slow main-memory DRAM accesses).

---

### 2.2 Amdahl's Law (Strong Scaling / Fixed-Size Workload)
Let $s$ be the inherently sequential fraction of execution time on 1 processor ($0 \le s \le 1$), and $p = 1 - s$ be the perfectly parallelizable fraction:

$$T(N) = s \cdot T(1) + \frac{(1 - s) \cdot T(1)}{N}$$

$$S_{\text{Amdahl}}(N) = \frac{T(1)}{T(N)} = \frac{T(1)}{s \cdot T(1) + \frac{1 - s}{N} \cdot T(1)} = \frac{1}{s + \frac{1 - s}{N}}$$

#### Asymptotic Amdahl Limit:
$$\lim_{N \to \infty} S_{\text{Amdahl}}(N) = \frac{1}{s}$$

- **Efficiency under Amdahl's Law:**
  $$\varepsilon(N) = \frac{1}{s \cdot N + (1 - s)}$$
- **Rule of Thumb:** If $5\%$ of a program is serial ($s = 0.05$), the maximum possible speedup on an infinite number of processors is $\frac{1}{0.05} = 20\times$.

---

### 2.3 Gustafson's Law (Weak Scaling / Scaled-Size Workload)
Let $s$ be the fraction of time spent in serial code *measured on the parallel machine* executing the scaled problem, and $1 - s$ be the parallel time. Normalizing parallel execution time to $T(N) = 1$:

$$T(N) = s + (1 - s) = 1$$

The equivalent time required by 1 processor to execute this scaled problem is:

$$T(1) = s + (1 - s) \cdot N$$

$$S_{\text{Gustafson}}(N) = \frac{T(1)}{T(N)} = \frac{s + (1 - s) \cdot N}{1} = s + (1 - s) \cdot N = N + s(1 - N)$$

- **Scaled Parallel Efficiency:**
  $$\varepsilon_{\text{Gustafson}}(N) = \frac{S(N)}{N} = 1 - s \left( \frac{N - 1}{N} \right) \quad \xrightarrow{N \to \infty} \quad 1 - s$$

---

### 2.4 The Karp-Flatt Metric (Experimentally Determined Serial Fraction)
An experimental diagnostic metric to assess whether parallel efficiency loss is caused by inherent algorithmic serial bottlenecks or communication/synchronization overhead.

Given empirical measurements of $S(N)$ on $N$ processors, the experimentally observed serial fraction $e$ is:

$$e = \frac{\frac{1}{S(N)} - \frac{1}{N}}{1 - \frac{1}{N}}$$

#### Diagnostic Decision Rules:
1. If $e$ remains **constant** as $N$ increases $\implies$ Speedup loss is dominated by inherent **algorithmic serial code** ($s$).
2. If $e$ **monotonically increases** as $N$ increases $\implies$ Speedup loss is dominated by **parallel communication overhead, synchronization contention, or load imbalance**.

---

## 3. Parallel Overhead, Imbalance & Analytical Performance Models (Lectures 3–5)

### 3.1 Parallel Overhead Model ($c(N)$)
Including parallelization overheads (thread creation, context switching, barrier synchronization, inter-node network transmission):

$$T(N) = \frac{T(1)}{N} + c(N)$$

$$S(N) = \frac{T(1)}{\frac{T(1)}{N} + c(N)} = \frac{N}{1 + \frac{N \cdot c(N)}{T(1)}}$$

#### 1. Linear Overhead Model: $c(N) = \alpha \cdot N$
Occurs in centralized lock contention, bus snooping, and master-bottleneck architectures:
$$T(N) = \frac{T(1)}{N} + \alpha \cdot N$$
To find the optimal processor count $N^*$ that minimizes execution time, set $\frac{dT(N)}{dN} = 0$:
$$\frac{dT(N)}{dN} = -\frac{T(1)}{N^2} + \alpha = 0 \implies N^* = \sqrt{\frac{T(1)}{\alpha}}$$
$$T_{\text{min}} = 2\sqrt{\alpha \cdot T(1)}$$

#### 2. Logarithmic Overhead Model: $c(N) = \beta \log_2 N$
Occurs in tree-based reductions and barrier synchronizations:
$$T(N) = \frac{T(1)}{N} + \beta \log_2 N$$

---

### 3.2 Load Imbalance Metrics
Let $T_i$ be the execution time of processor/thread $i \in \{0, 1, \dots, N-1\}$.

- **Maximum and Average Thread Times:**
  $$T_{\text{max}} = \max_{0 \le i < N} T_i, \qquad T_{\text{avg}} = \frac{1}{N} \sum_{i=0}^{N-1} T_i$$
- **Load Imbalance Factor ($\lambda$):**
  $$\lambda = \frac{T_{\text{max}} - T_{\text{avg}}}{T_{\text{max}}} \times 100\% \quad (0 \le \lambda < 1)$$
- **Parallel Speedup Loss Due to Imbalance:**
  $$S(N) \le \frac{T(1)}{T_{\text{max}}} = \frac{T(1)}{T_{\text{avg}} / (1 - \lambda)} = (1 - \lambda) \cdot S_{\text{ideal}}$$

---

## 4. OpenMP Loop Scheduling, Vectorization & False Sharing (Lectures 8–10, 15–16)

### 4.1 Static Loop Scheduling
Given total loop iterations $I$, thread count $T$, and chunk size $C$:
- **Total Chunks Generated:**
  $$N_{\text{chunks}} = \left\lceil \frac{I}{C} \right\rceil$$
- **Chunk-to-Thread Mapping (Round-Robin):**
  $$\text{Chunk } k \implies \text{Assigned to Thread } \left( k \pmod T \right)$$
- **Default Static Chunk Size (when $C$ is omitted):**
  $$C_{\text{default}} = \left\lceil \frac{I}{T} \right\rceil$$

---

### 4.2 Guided Loop Scheduling
In `#pragma omp for schedule(guided, min_chunk)`, chunk size $C_k$ decreases exponentially with each dispatched chunk:

$$C_k = \max\left( \left\lceil \frac{R_k}{2 \cdot T} \right\rceil, \; C_{\text{min}} \right)$$

Where $R_k$ is the number of unassigned iterations remaining when chunk $k$ is requested, $T$ is the number of threads, and $C_{\text{min}}$ is the optional minimum chunk size parameter (default = 1).

---

### 4.3 Hardware False Sharing Collision Condition
In multi-threaded shared memory, false sharing occurs when two distinct threads $T_a$ and $T_b$ concurrently modify variables stored at memory byte addresses $A_a$ and $A_b$ that reside on the same physical $L$-byte cache line (standard $L = 64\text{ bytes}$):

$$\left\lfloor \frac{A_a}{L} \right\rfloor == \left\lfloor \frac{A_b}{L} \right\rfloor \quad \text{where } A_a \ne A_b$$

- **Padding Requirement:** To isolate independent thread accumulators in an array `sum[T]`:
  $$\text{Element Stride} \ge \frac{L}{\text{sizeof}(\text{data\_type})} = \frac{64\text{ bytes}}{8\text{ bytes}} = 8 \text{ elements}$$

---

### 4.4 SIMD Vectorization Speedup
The theoretical speedup of an inner loop vectorized via `#pragma omp simd` with hardware vector register width $W_{\text{SIMD}}$ bits:

$$\text{Vector Capacity (elements)} = V = \frac{W_{\text{SIMD}}}{\text{sizeof}(\text{type}) \times 8}$$

$$S_{\text{SIMD}} = \frac{V}{1 + (V - 1) \cdot f_{\text{unaligned}} + \sigma_{\text{peel/remainder}}}$$

---

## 5. Distributed Communication & Network Performance Models (Lectures 26–31)

### 5.1 Hockney's $(\alpha, \beta)$ Communication Model
The standard analytical model for point-to-point network communication time transmitting a message of $m$ bytes between two nodes:

$$T_{\text{comm}}(m) = \alpha + m \cdot \beta = \alpha + \frac{m}{B}$$

| Parameter | Meaning | Typical Interconnect Value (InfiniBand HDR) |
| :--- | :--- | :--- |
| $\alpha$ | **Network Latency (Startup Time):** Hardware injection latency, NIC firmware traversal, and protocol header processing time. | $\approx 0.5\text{ \textmu s} - 1.5\text{ \textmu s}$ |
| $\beta$ | **Inverse Bandwidth ($1/B$):** Time required to transfer a single byte across the physical network link. | $\approx 5 \times 10^{-11}\text{ s/byte}$ ($B \approx 200\text{ Gbps} = 25\text{ GB/s}$) |
| $m$ | Message payload size | Bytes |

- **Effective Bandwidth ($B_{\text{eff}}$):**
  $$B_{\text{eff}}(m) = \frac{m}{T_{\text{comm}}(m)} = \frac{m}{\alpha + m \cdot \beta} = \frac{B}{1 + \frac{\alpha \cdot B}{m}}$$
- **Half-Performance Message Size ($n_{1/2}$):** The message size required to achieve exactly $50\%$ of peak wire bandwidth:
  $$B_{\text{eff}}(n_{1/2}) = \frac{B}{2} \iff n_{1/2} = \frac{\alpha}{\beta} = \alpha \times B$$

---

### 5.2 Non-Blocking Computation-Communication Overlap
In non-blocking communication (`MPI_Isend`, `MPI_Irecv`), computation proceeds concurrently with background network data transfer.

Let $T_{\text{comp}}$ be the local calculation time, and $T_{\text{comm}}$ be the message transfer time:

$$T_{\text{hybrid}} = \max\left( T_{\text{comp}}, \; T_{\text{comm}} \right) + \tau_{\text{init}}$$

Where $\tau_{\text{init}}$ is the small non-blocking initiation overhead ($\approx 100\text{ ns}$).
- **Perfect Latency Hiding:** Achieved when $T_{\text{comp}} \ge T_{\text{comm}}$, meaning effective communication cost to the application is reduced to $\tau_{\text{init}} \approx 0$.

---

## 6. Collective Communication Complexity & Scalability Models (Lectures 30–31)

Let $P$ be the number of MPI processes in the communicator, $m$ be the message size in bytes, $\alpha$ be network latency, $\beta$ be inverse bandwidth, and $\gamma$ be the local computational time required per byte for an arithmetic reduction operator.

### 6.1 Summary of Collective Time Complexity

| Collective Routine | Algorithmic Implementation | Total Communication Time $T(m, P)$ | Bandwidth Scalability with $P$ |
| :--- | :--- | :--- | :--- |
| **`MPI_Barrier`** | Dissemination / Recursive Doubling | $\lceil \log_2 P \rceil \alpha$ | Indep. of $m$ (transfers 0 bytes) |
| **`MPI_Bcast`** | Binomial Spanning Tree | $\lceil \log_2 P \rceil (\alpha + m \cdot \beta)$ | $O(\log_2 P)$ latency, $O(1)$ bandwidth |
| **`MPI_Bcast`** | Van de Geijn (Scatter-Allgather) | $2 \left( \frac{P - 1}{P} \right) m \beta + 2 \lceil \log_2 P \rceil \alpha$ | Optimal for large messages ($m \gg 1\text{ MB}$) |
| **`MPI_Scatter`** | Linear Tree (Root Unicast) | $(P - 1)\alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta$ | Unscalable for large $P$ (Root bottleneck) |
| **`MPI_Scatter`** | Binomial Tree | $\lceil \log_2 P \rceil \alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta$ | $O(\log_2 P)$ latency |
| **`MPI_Gather`** | Binomial Tree | $\lceil \log_2 P \rceil \alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta$ | $O(\log_2 P)$ latency |
| **`MPI_Allgather`** | Ring Algorithm | $(P - 1)\alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta$ | Near-optimal for large $m$ |
| **`MPI_Reduce`** | Binomial Reduction Tree | $\lceil \log_2 P \rceil \left( \alpha + m \cdot \beta + m \cdot \gamma \right)$ | $O(\log_2 P)$ latency & compute |
| **`MPI_Allreduce`** | Rabenseifner's Algorithm | $2 \lceil \log_2 P \rceil \alpha + 2 \left( \frac{P - 1}{P} \right) m \cdot \beta + \left( \frac{P - 1}{P} \right) m \cdot \gamma$ | State-of-the-art supercomputing standard |

---

### 6.2 Mathematical Derivations of Spanning Tree Collectives

#### Derivation 1: Binomial Tree Broadcast
In a binomial tree broadcast on $P = 2^k$ processes, the tree has a depth of $\log_2 P$ stages. In each stage $i \in \{1, 2, \dots, \log_2 P\}$, processes that already have the data send the entire message of $m$ bytes to one new process concurrently:
$$T_{\text{stage}} = \alpha + m \cdot \beta$$
$$T_{\text{Bcast}}(m, P) = \sum_{i=1}^{\log_2 P} T_{\text{stage}} = \lceil \log_2 P \rceil (\alpha + m \cdot \beta)$$

#### Derivation 2: Rabenseifner's Allreduce Algorithm
Rabenseifner's algorithm executes Allreduce in two phases without a centralized root bottleneck:
1. **Reduce-Scatter Phase:** Processes perform recursive pairwise exchanges, reducing chunks of size $\frac{m}{2^i}$. Total data sent per process:
   $$T_{\text{red-scat}} = \log_2 P \cdot \alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta + \left( \frac{P - 1}{P} \right) m \cdot \gamma$$
2. **Allgather Phase:** Processes circulate the reduced chunks using an allgather ring or recursive doubling:
   $$T_{\text{allgather}} = \log_2 P \cdot \alpha + \left( \frac{P - 1}{P} \right) m \cdot \beta$$
Summing both phases yields:
$$T_{\text{Allreduce}} = 2 \log_2 P \cdot \alpha + 2 \left( \frac{P - 1}{P} \right) m \cdot \beta + \left( \frac{P - 1}{P} \right) m \cdot \gamma$$
Notice that the bandwidth term $2\left(\frac{P-1}{P}\right)m\beta \approx 2m\beta$, which is **completely independent of process count $P$**, making it perfectly scalable!

---

## 7. Hybrid MPI + OpenMP Hierarchical Core Allocation Models (Lecture 32)

### 7.1 Hardware Core Saturation Equations
For an HPC cluster composed of $N_{\text{nodes}}$ physical nodes, where each node contains $S$ CPU sockets, and each socket has $C_{\text{socket}}$ physical CPU cores:

$$C_{\text{node}} = S \times C_{\text{socket}}$$

$$C_{\text{total}} = N_{\text{nodes}} \times C_{\text{node}} = N_{\text{nodes}} \times S \times C_{\text{socket}}$$

Let $M_{\text{total}}$ be the total number of MPI processes launched, and $T_{\text{omp}}$ be the number of OpenMP threads per process (`OMP_NUM_THREADS`).

#### Golden Non-Oversubscription Rule:
$$M_{\text{total}} \times T_{\text{omp}} = C_{\text{total}}$$

$$\text{Ranks per Node } (M_{\text{node}}) \times T_{\text{omp}} = C_{\text{node}}$$

---

### 7.2 Optimal NUMA Mapping
To eliminate cross-socket Non-Uniform Memory Access (NUMA) interconnect penalties, allocate exactly **1 MPI process per CPU socket**:

$$M_{\text{node}} = S \implies M_{\text{total}} = N_{\text{nodes}} \times S$$

$$T_{\text{omp}} = C_{\text{socket}}$$

---

### 7.3 Hierarchical Domain Decomposition Problem Sizing
When dividing a global problem array of $N$ elements across an $M$-process, $T$-thread hybrid execution hierarchy:

1. **Per-Process Workload Chunk:**
   $$W_{\text{process}} = \left\lfloor \frac{N}{M_{\text{total}}} \right\rfloor$$
   $$\text{Process Rank } r \text{ slice: } \left[ r \cdot W_{\text{process}}, \; (r + 1) \cdot W_{\text{process}} - 1 \right]$$
2. **Per-Thread Workload Chunk:**
   $$W_{\text{thread}} = \left\lfloor \frac{W_{\text{process}}}{T_{\text{omp}}} \right\rfloor = \left\lfloor \frac{N}{M_{\text{total}} \times T_{\text{omp}}} \right\rfloor$$
3. **Remainder Allocation:**
   $$\text{Remaining Elements} = R = N \pmod{M_{\text{total}} \times T_{\text{omp}}}$$
   Distributed 1 element per unit to the first $R$ threads to maintain strict load balance.

---

### 7.4 Core Over-Subscription Penalty Ratio ($\rho$)
The ratio of requested execution threads to available physical hardware execution units:

$$\rho = \frac{M_{\text{total}} \times T_{\text{omp}}}{C_{\text{total}}}$$

$$\text{Effective Wall Time} \approx T_{\text{ideal}} \times \left( \rho + (\rho - 1) \frac{\tau_{\text{context\_switch}} + \tau_{\text{cache\_miss}}}{\Delta t_{\text{quantum}}} \right)$$

- If $\rho = 1.0$: Optimal bare-metal execution.
- If $\rho > 1.0$: Operating system scheduler induces severe context switching; L1/L2 cache invalidations dominate; runtime degrades by up to $10\times$.

---
*End of Master Examination Formula Sheet. Keep this reference accessible for all numerical and analytical derivations.*
