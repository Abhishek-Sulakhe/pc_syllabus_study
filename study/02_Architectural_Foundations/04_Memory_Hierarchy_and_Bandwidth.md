# Module 02.04 — The Memory Hierarchy, Cache Organization & Memory Bandwidth

> **Prerequisites:** Module 02.01 (Stored-Program Model & Von Neumann Bottleneck), Module 02.03 (Vector Extensions)  
> **Lecture Mapping:** Lecture 2 (Architectural Foundations: Memory Hierarchy, Cache Lines, Memory Bandwidth)  
> **Exam Relevance:** MidSem (Cache Miss Classifications, Locality Principles, Bandwidth Calculations) & EndSem  

---

## 1. Understanding the Basic Idea

Suppose you are a scholar writing a research paper at your study desk.
- On your **desk directly in front of you**, you have a notepad and pencil. You can write on it instantaneously ($0\text{ seconds}$ delay). But your desk surface is small; it only holds a couple of pages.
- Right beside your desk is a **bookshelf**. It holds 50 reference books. Reaching for a book takes $5\text{ seconds}$.
- In the hallway outside your room is a **cabinet**. It holds 500 books. Walking to it takes $30\text{ seconds}$.
- Across town is the **City Central Library**. It holds 10 million books. Driving there, finding a book, and returning takes **2 hours**.

```
Scholar's Desk Analogy               Microprocessor Memory Hierarchy
─────────────────────────────────────────────────────────────────────────
Notepad on Desk (Instant)        ──► CPU Registers (0.3 ns, ~1 KB)
Small Bookshelf (5 seconds)      ──► L1 Cache (1 ns, ~32–64 KB)
Hallway Cabinet (30 seconds)     ──► L2 / L3 Cache (3–15 ns, ~1–64 MB)
City Central Library (2 hours!)  ──► Main Memory DRAM (60–100 ns, ~32–512 GB)
```

Now imagine what happens if, for **every single word** you write on your paper, you are forced to get in your car, drive across town to the central library, look up one fact, and drive back. You would write only three sentences a day!

To write efficiently, you must use your desk and bookshelf intelligently:
1. When you fetch a book, you keep it on your desk because you will likely read from it again soon (**Temporal Locality**).
2. When you open a page, you read the entire paragraph, not just one isolated letter (**Spatial Locality**).

This is why modern microprocessors are built with a multi-layered **Memory Hierarchy**. The CPU's ALUs are blindingly fast, but they are utterly useless unless data is staged close to them in high-speed hardware caches.

---

## 2. The Memory Hierarchy Pyramid

The memory hierarchy is organized as a pyramid governed by an inescapable law of semiconductor physics:
> **Memory can be FAST, or it can be LARGE, or it can be CHEAP. Physics does not permit all three simultaneously.**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        THE HARDWARE MEMORY HIERARCHY                            │
└─────────────────────────────────────────────────────────────────────────────────┘

                ▲               Speed: Fastest | Cost: Highest | Size: Smallest
               / \
              /   \             LEVEL 0: CPU REGISTERS
             / REG \            Size: ~1–2 KB | Latency: 0.3–0.5 ns (1 cycle)
            /───────\           Bandwidth: > 10,000 GB/s (Multi-Ported Internal Wire)
           /   L1    \
          /   CACHE   \         LEVEL 1: L1 CACHE (Split: 32 KB I-Cache + 32 KB D-Cache)
         /─────────────\        Size: 32–64 KB per core | Latency: 1 ns (~4 cycles)
        /      L2       \       Bandwidth: ~3,000–5,000 GB/s
       /     CACHE       \
      /───────────────────\     LEVEL 2: L2 UNIFIED CACHE
     /         L3          \    Size: 512 KB–1 MB per core | Latency: 3–4 ns (~12–14 cycles)
    /     SHARED CACHE      \   Bandwidth: ~1,000–2,000 GB/s
   /─────────────────────────\
  /        MAIN MEMORY        \ LEVEL 3: L3 LAST-LEVEL CACHE (LLC)
 /        (DRAM / HBM)         \Size: 16–256 MB shared per socket | Latency: 10–15 ns (~40 cycles)
/───────────────────────────────\
│      NON-VOLATILE STORAGE     │LEVEL 4: MAIN MEMORY (DDR4 / DDR5 DRAM)
│     (NVMe SSD / Hard Disk)    │Size: 32 GB–2 TB | Latency: 60–100 ns (~200 cycles)
└───────────────────────────────┘Bandwidth: ~50–300 GB/s (DRAM) or ~1,000–3,000 GB/s (HBM)
                                Speed: Slowest | Cost: Lowest | Size: Massive
```

### Detailed Layer Characteristics

| Memory Level | Typical Capacity (per core/socket) | Typical Latency (Cycles) | Typical Latency (Time) | Peak Bandwidth | Technology | Managed By |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Registers** | $\approx 1\text{ to }2\text{ KB}$ | 1 cycle | $\approx 0.3\text{ ns}$ | $> 10,000\text{ GB/s}$ | Multi-ported D-flip-flops | Compiler / Hardware |
| **L1 Cache** | $32\text{ to }64\text{ KB}$ | $4\text{ to }5\text{ cycles}$ | $\approx 1.0\text{ ns}$ | $\approx 4,000\text{ GB/s}$ | Static RAM (6T SRAM) | Hardware |
| **L2 Cache** | $512\text{ KB to }1\text{ MB}$ | $12\text{ to }14\text{ cycles}$| $\approx 3.5\text{ ns}$ | $\approx 1,500\text{ GB/s}$ | Static RAM (6T/8T SRAM) | Hardware |
| **L3 Cache (LLC)**| $16\text{ to }256\text{ MB}$ | $35\text{ to }50\text{ cycles}$| $\approx 12.0\text{ ns}$| $\approx 500\text{ to }800\text{ GB/s}$ | High-density eDRAM/SRAM | Hardware |
| **Main Memory (DDR5)**| $32\text{ to }512\text{ GB}$ | $150\text{ to }250\text{ cycles}$| $\approx 60\text{ to }90\text{ ns}$| $\approx 50\text{ to }200\text{ GB/s}$ | Dynamic RAM (1T-1C DRAM) | Operating System / MMU |
| **NVMe SSD** | $1\text{ to }16\text{ TB}$ | $10^5\text{ cycles}$ | $\approx 10\text{ to }50\ \mu\text{s}$ | $\approx 3\text{ to }7\text{ GB/s}$ | NAND Flash EEPROM | OS / File System |

---

## 3. Principles of Locality: Temporal and Spatial

Why do caches work so miraculously well? Why does a tiny $64\text{ KB}$ cache satisfy over $95\%$ of a program's memory requests?

The answer lies in the **Principles of Locality**: empirical laws describing the predictable behavior of computer software.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          THE PRINCIPLES OF LOCALITY                             │
└─────────────────────────────────────────────────────────────────────────────────┘

 1. TEMPORAL LOCALITY (Locality in Time)
    "If a memory location is accessed now, it will likely be accessed again soon."
    
    Example: Loop accumulators, induction variables, stack pointers.
    for (int i = 0; i < N; i++) {
        sum += a[i];  // The variable 'sum' has extreme TEMPORAL locality!
    }

 2. SPATIAL LOCALITY (Locality in Space)
    "If a memory location is accessed now, nearby memory locations will likely
     be accessed soon."
    
    Example: Sequential array traversals, instruction streams.
    for (int i = 0; i < N; i++) {
        sum += a[i];  // Array 'a' has perfect SPATIAL locality!
                      // Accessing a[0] means a[1], a[2], a[3] will follow immediately!
    }
```

---

## 4. Cache Organization: The 64-Byte Cache Line

A fundamental rule of computer architecture that every parallel programmer must memorize is:
> **The CPU NEVER transfers a single byte or a single float from main memory. It ALWAYS transfers an entire block called a Cache Line.**

On virtually all modern x86, ARM, and RISC-V processors, the standard cache line size is **exactly 64 bytes** (512 bits).

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       ANATOMY OF A 64-BYTE CACHE LINE                           │
└─────────────────────────────────────────────────────────────────────────────────┘

 Byte Offset: 0        8        16       24       32       40       48       56      63
             ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
 Cache Line: │Double 0│Double 1│Double 2│Double 3│Double 4│Double 5│Double 6│Double 7│
             └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
             ◄──────────────────────── 64 Bytes (512 Bits) ──────────────────────────►
             [ Contains EIGHT 64-bit doubles OR SIXTEEN 32-bit floats ]
```

### What Happens on a Cache Miss?
Suppose your C program requests a single double-precision float: `double x = a[0];`.
1. The CPU checks L1 cache. It is not there (**L1 Cache Miss**).
2. It checks L2 and L3. Not there (**L2/L3 Cache Miss**).
3. The memory controller issues a DRAM read command across the memory bus.
4. DRAM does not return just `a[0]`. **It returns the entire 64-byte block containing `a[0]` through `a[7]`!**
5. All 64 bytes are written into an L1 cache line slot.
6. When your code executes the next loop iteration `double y = a[1];`, the data is already sitting in L1 cache! Access takes **1 nanosecond instead of 80 nanoseconds** (**Cache Hit**).

---

## 5. Cache Miss Classifications: The 3 C’s (and the 4th C in Parallel Systems)

When a memory access fails to find the requested data in cache, a **cache miss** occurs. In 1989, Mark Hill classified cache misses into **The 3 C's** (later expanded to 4 for parallel computing):

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        THE 4 C's OF CACHE MISSES                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
      ┌─────────────────┬──────────────┴──────────────────┬─────────────────┐
      ▼                 ▼                                 ▼                 ▼
 1. COMPULSORY     2. CAPACITY                      3. CONFLICT        4. COHERENCE
 (Cold Start)      (Working Set Too Big)            (Collision)        (Multi-Core)
 First access to   Cache is too small to            Multiple addresses Invalidation caused
 a memory block;   hold all data needed during      map to the exact   by another core
 never seen before.program execution.               same cache set.    writing to the line.
```

1. **Compulsory Misses (Cold Misses):**
   - Occur on the very first reference to a memory block. The data has never been loaded into cache before.
   - *Mitigation:* Hardware prefetchers that detect sequential strides and pre-load lines into cache before the CPU asks for them.
2. **Capacity Misses:**
   - Occur when the total working data set of the program exceeds the physical size of the cache (e.g., trying to fit a $500\text{ MB}$ matrix into a $32\text{ MB}$ L3 cache). Lines must be evicted and re-read later.
   - *Mitigation:* Algorithmic tiling / cache blocking (breaking big matrices into small sub-blocks that fit into L1/L2).
3. **Conflict Misses (Collision Misses):**
   - Occur in set-associative or direct-mapped caches when multiple memory addresses map to the exact same cache set, evicting each other prematurely even though the rest of the cache is half empty!
   - *Mitigation:* Increasing cache associativity (e.g., 8-way or 16-way associative caches) or array padding.
4. **Coherence Misses (The 4th C in Parallel Computing):**
   - Occur exclusively in multi-core / shared-memory parallel systems.
   - If Core 0 has a cache line in its L1 cache, and Core 1 writes to that same cache line, Core 0's copy is marked **INVALID**. When Core 0 reads it again, it experiences a **Coherence Miss** (explored in depth as *False Sharing* in Module 06.04).

---

## 6. Memory Latency vs. Memory Bandwidth

Students frequently conflate **Latency** and **Bandwidth**. They measure completely different physical properties of the memory subsystem:

```
Water Pipe Analogy:
- LATENCY   = Time taken for the first drop of water to travel down the pipe.
- BANDWIDTH = Diameter of the pipe (volume of water delivered per second).
```

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      LATENCY vs. BANDWIDTH COMPARISON                           │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │ MEMORY LATENCY:                                         │
  │ The delay between issuing a read request and receiving  │
  │ the first byte of data. Measured in NANOSECONDS (ns).   │
  └─────────────────────────────────────────────────────────┘
                            vs.
  ┌─────────────────────────────────────────────────────────┐
  │ MEMORY BANDWIDTH:                                       │
  │ The rate at which data can be continuously streamed     │
  │ from memory into the processor. Measured in GB/s.       │
  └─────────────────────────────────────────────────────────┘
```

### The Theoretical Memory Bandwidth Formula
For a DDR5 memory channel:
\[
\boxed{\text{Peak Bandwidth} = \text{Data Transfer Rate}\ (\text{MT/s}) \times \text{Bus Width}\ (\text{Bytes}) \times N_{\text{channels}}}
\]
*Example:* A server with dual-socket CPUs, each supporting **8 memory channels** of DDR5-4800 (4,800 MT/s). Each memory channel has an 8-byte (64-bit) bus:
\[
\text{Bandwidth per Channel} = 4,800 \times 10^6\text{ transfers/s} \times 8\text{ bytes} = 38.4\text{ GB/s}
\]
\[
\text{Total Socket Bandwidth} = 8\text{ channels} \times 38.4\text{ GB/s} = \mathbf{307.2\text{ GB/s}}
\]

### Bytes-per-FLOP Ratio: The Starvation Indicator
To see if a processor's compute units will be starved for data, HPC architects calculate the **Bytes-per-FLOP ratio**:
\[
\boxed{\beta = \frac{\text{Memory Bandwidth (Bytes/s)}}{\text{Peak Compute Throughput (FLOP/s)}}}
\]
- On early vector supercomputers (like the Cray-1 in 1976), $\beta \approx 1.0\text{ to }2.0$ (1 to 2 bytes of memory bandwidth for every single FLOP/s).
- On modern x86 multicore CPUs, peak compute is $\approx 4\text{ TFLOP/s}$, but memory bandwidth is only $\approx 200\text{ GB/s}$:
  \[
  \beta = \frac{200 \times 10^9\text{ Bytes/s}}{4,000 \times 10^9\text{ FLOP/s}} = \mathbf{0.05\text{ Bytes/FLOP}}
  \]
- **The Tragedy:** The CPU can compute 20 floating-point operations in the time memory delivers a single byte! Unless algorithms exhibit massive data reuse in caches, the ALUs will starve.

---

## 7. Practical Implementation: The Stride Benchmark in C

To prove experimentally that cache lines exist and that jumping across memory destroys performance, examine this C program. It accesses the same total amount of data in memory, but with different **strides** (step sizes).

### Cache Stride Benchmark (`cache_stride.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define ARRAY_SIZE (64 * 1024 * 1024) // 64 Million ints = 256 MB (Exceeds L3 cache!)

int main(void) {
    printf("============================================================\n");
    printf("        DEMONSTRATING CACHE LINE STRIDE & LOCALITY          \n");
    printf("============================================================\n");

    int *arr = (int *)malloc(ARRAY_SIZE * sizeof(int));
    if (!arr) {
        fprintf(stderr, "Error: Memory allocation failed!\n");
        return 1;
    }

    // Initialize array
    for (int i = 0; i < ARRAY_SIZE; i++) arr[i] = 1;

    int strides[] = {1, 2, 4, 8, 16, 32, 64};
    int num_strides = sizeof(strides) / sizeof(strides[0]);

    printf("Array Size: 256 MB (Forces DRAM fetches on misses)\n");
    printf("Cache Line: 64 Bytes = 16 Integers\n\n");
    printf("%-10s %-20s %-15s\n", "Stride", "Bytes Jumped", "Time (Seconds)");
    printf("------------------------------------------------------------\n");

    struct timespec start, end;

    for (int s = 0; s < num_strides; s++) {
        int stride = strides[s];

        clock_gettime(CLOCK_MONOTONIC, &start);

        long long sum = 0;
        // Access array with stride step
        for (int i = 0; i < ARRAY_SIZE; i += stride) {
            sum += arr[i];
        }

        clock_gettime(CLOCK_MONOTONIC, &end);
        double elapsed = (end.tv_sec - start.tv_sec) + 
                         (end.tv_nsec - start.tv_nsec) / 1e9;

        printf("Stride %-4d (%3d bytes)       : %.5f s (Checksum: %lld)\n", 
               stride, stride * (int)sizeof(int), elapsed, sum);
    }

    printf("============================================================\n");
    free(arr);
    return 0;
}
```

#### Compilation & Execution
```bash
gcc -O2 cache_stride.c -o cache_stride
./cache_stride
```

*Expected Output (on an Intel x86-64 CPU):*
```
============================================================
        DEMONSTRATING CACHE LINE STRIDE & LOCALITY          
============================================================
Array Size: 256 MB (Forces DRAM fetches on misses)
Cache Line: 64 Bytes = 16 Integers

Stride     Bytes Jumped         Time (Seconds) 
------------------------------------------------------------
Stride 1    (  4 bytes)       : 0.04210 s (Checksum: 67108864)
Stride 2    (  8 bytes)       : 0.03850 s (Checksum: 33554432)
Stride 4    ( 16 bytes)       : 0.03510 s (Checksum: 16777216)
Stride 8    ( 32 bytes)       : 0.03320 s (Checksum: 8388608)
Stride 16   ( 64 bytes)       : 0.03280 s (Checksum: 4194304)  ◄── Critical Point!
Stride 32   (128 bytes)       : 0.01690 s (Checksum: 2097152)
Stride 64   (256 bytes)       : 0.00860 s (Checksum: 1048576)
============================================================
```

#### The Fascinating Analysis:
- Notice Stride 1 vs. Stride 16:
  - Stride 16 reads **$16\times$ fewer integers** than Stride 1 ($4.19\text{ million}$ vs $67.1\text{ million}$)!
  - Yet Stride 16 took **almost the exact same amount of time** ($0.0328\text{ s}$ vs $0.0421\text{ s}$)!
- **Why?** Because 16 integers $\times 4\text{ bytes} = \mathbf{64\text{ bytes}}$ (one full cache line)!
- At Stride 1, loading `arr[0]` pulled in `arr[0..15]`, so the next 15 loop iterations were instant L1 cache hits.
- At Stride 16, every single read landed in a **different cache line**, triggering a DRAM fetch every single step! You moved 16 times less data, but suffered the exact same number of DRAM cache line transfers!

---

## 8. Exam-Ready Definitions

### Cache Line (Cache Block)
> A **Cache Line** is the minimum unit of contiguous data transferred between main memory and CPU hardware cache levels, typically fixed at 64 bytes on modern microprocessor architectures.

### Temporal Locality
> **Temporal Locality** is the principle of program behavior stating that if a particular memory location is referenced at a given point in time, it is highly probable that the exact same location will be referenced again in the near future.

### Spatial Locality
> **Spatial Locality** is the principle of program behavior stating that if a particular memory location is referenced, it is highly probable that memory locations physically adjacent to it will be referenced in the near future.

### Working Set
> The **Working Set** of a process is the collection of distinct memory pages and cache blocks actively referenced by the application during a given execution time window.

---

## 9. Important Comparisons

### SRAM (Cache) vs. DRAM (Main Memory)

| Characteristic | Static RAM (SRAM - Caches) | Dynamic RAM (DRAM - Main Memory) |
|:---|:---|:---|
| **Storage Element** | Bistable latch circuit: 6 transistors (6T) | 1 transistor + 1 microscopic capacitor (1T-1C)|
| **Speed / Access Time**| Blindingly fast ($0.5\text{ to }3\text{ ns}$) | Moderate to slow ($50\text{ to }100\text{ ns}$) |
| **Refresh Requirement**| None (holds data continuously while powered) | Must be periodically refreshed every few milliseconds |
| **Density (Bits/mm²)** | Low (large transistor area per bit) | Extremely High (capacitor stores single bit) |
| **Manufacturing Cost** | Very high ($>\$100\text{ per GB}$) | Cheap and commodity ($\approx \$2\text{--}\$4\text{ per GB}$) |
| **Role in Computer** | On-chip L1, L2, and L3 caches | Off-chip system RAM sticks (DIMMs) |

---

## 10. Solved Problems

### Problem 1: Average Memory Access Time (AMAT) Calculation
A high-performance computing core has the following memory hierarchy characteristics:
- **Clock Cycle Time ($\tau$):** $0.5\text{ ns}$ ($2.0\text{ GHz}$).
- **L1 Cache:** Hit time = $1\text{ cycle}$ ($0.5\text{ ns}$), Miss rate = $5\%$.
- **L2 Cache:** Hit time = $10\text{ cycles}$ ($5.0\text{ ns}$), Miss rate = $15\%$.
- **L3 Cache:** Hit time = $30\text{ cycles}$ ($15.0\text{ ns}$), Miss rate = $20\%$.
- **Main Memory DRAM:** Access latency = $160\text{ cycles}$ ($80.0\text{ ns}$).

Calculate:
1. The Average Memory Access Time (AMAT) in clock cycles.
2. The Average Memory Access Time in nanoseconds.
3. If an algorithmic optimization reduces the L1 miss rate from $5\%$ to $2\%$, calculate the new AMAT in nanoseconds and the resulting memory speedup.

#### Solution:
#### 1. AMAT Formula for a 3-Level Cache
\[
\text{AMAT} = \text{HitTime}_{L1} + \text{MissRate}_{L1} \times \left( \text{HitTime}_{L2} + \text{MissRate}_{L2} \times \left( \text{HitTime}_{L3} + \text{MissRate}_{L3} \times \text{MissPenalty}_{\text{DRAM}} \right) \right)
\]

#### 2. Step-by-Step Substitution (in Cycles)
- **L3 Miss Penalty to DRAM:** $160\text{ cycles}$.
- **Cost of an L2 Miss:**
  \[
  \text{Penalty}_{L2} = \text{HitTime}_{L3} + (\text{MissRate}_{L3} \times \text{DRAM}) = 30 + (0.20 \times 160) = 30 + 32 = 62\text{ cycles}
  \]
- **Cost of an L1 Miss:**
  \[
  \text{Penalty}_{L1} = \text{HitTime}_{L2} + (\text{MissRate}_{L2} \times \text{Penalty}_{L2}) = 10 + (0.15 \times 62) = 10 + 9.3 = 19.3\text{ cycles}
  \]
- **Total AMAT:**
  \[
  \text{AMAT} = \text{HitTime}_{L1} + (\text{MissRate}_{L1} \times \text{Penalty}_{L1}) = 1 + (0.05 \times 19.3) = 1 + 0.965 = \mathbf{1.965\text{ cycles}}
  \]
- **In Nanoseconds:**
  \[
  \text{AMAT}_{\text{ns}} = 1.965\text{ cycles} \times 0.5\text{ ns/cycle} = \mathbf{0.9825\text{ ns}}
  \]

#### 3. Recalculation with Optimized L1 Miss Rate ($2\%$):
\[
\text{AMAT}_{\text{new}} = 1 + (0.02 \times 19.3) = 1 + 0.386 = \mathbf{1.386\text{ cycles}}
\]
\[
\text{AMAT}_{\text{new, ns}} = 1.386 \times 0.5\text{ ns} = \mathbf{0.693\text{ ns}}
\]
\[
\text{Speedup} = \frac{0.9825\text{ ns}}{0.693\text{ ns}} = \mathbf{1.42\times\ (42\%\text{ faster memory system!})}
\]

---

### Problem 2: Cache Capacity and Matrix Tiling
A program multiplies two dense $N \times N$ matrices of 64-bit IEEE double-precision floats ($8\text{ bytes}$ per element). 
The CPU has a private L1 Data Cache of size $32\text{ KB}$ ($32,768\text{ bytes}$).
To prevent capacity misses during matrix multiplication, cache blocking (tiling) is applied using square sub-blocks of size $B \times B$.

To ensure that the sub-blocks of matrix $A$, matrix $B$, and matrix $C$ fit comfortably in L1 cache simultaneously without capacity evictions:
1. Formulate the memory footprint constraint in terms of $B$.
2. Calculate the maximum integer block size $B$ that fits into the $32\text{ KB}$ L1 cache.

#### Solution:
1. **Memory Footprint Formulation:**
   The algorithm holds three $B \times B$ tiles in cache concurrently: Tile $A$, Tile $B$, and Tile $C$.
   \[
   \text{Total Footprint} = 3 \times B^2 \times 8\text{ bytes} = 24 B^2\text{ bytes}
   \]
   Constraint:
   \[
   24 B^2 \le 32,768\text{ bytes}
   \]
2. **Solve for $B$:**
   \[
   B^2 \le \frac{32,768}{24} \approx 1,365.33
   \]
   \[
   B \le \sqrt{1,365.33} \approx 36.95
   \]
   Choosing an optimal power-of-two block size to align with 64-byte cache line boundaries ($64\text{ bytes} = 8\text{ doubles}$):
   \[
   \boxed{B = 32}
   \]
   Verification: $3 \times (32 \times 32) \times 8 = 3 \times 1024 \times 8 = 24,576\text{ bytes} \approx 24\text{ KB} < 32\text{ KB}$.
   The remaining $8\text{ KB}$ provides a safety cushion against conflict misses.

---

## 11. Conceptual Examination Questions & Answers

### Q1: Why do modern processors feature a split L1 cache (L1-Instruction and L1-Data) instead of a unified L1 cache?
**Model Answer:**
Modern processors feature separate L1-I (Instruction) and L1-D (Data) caches to **eliminate structural pipeline hazards**.

In a classical 5-stage RISC pipeline:
- The **Instruction Fetch (IF)** stage must access memory to read the next machine instruction.
- The **Memory Access (MEM)** stage must simultaneously access memory to read or write data operands for a `load` or `store` instruction.

If the L1 cache were unified (sharing a single read/write memory port), an instruction in the MEM stage would collide with an instruction in the IF stage, forcing the pipeline to insert a stall cycle. Splitting L1 into two independent hardware caches allows IF and MEM to access memory concurrently in the same clock cycle with zero contention.

---

### Q2: What is the "Memory Wall," and how does the Bytes-per-FLOP ratio illustrate it?
**Model Answer:**
The **Memory Wall** refers to the widening performance chasm between CPU computing speed and off-chip DRAM memory latency/bandwidth. 

Historically, CPU compute throughput increased by $\approx 55\%$ per year (via Dennard scaling, higher clock speeds, and SIMD vectorization), while DRAM latency improved by only $\approx 7\%$ per year. 

This disparity is reflected in the **Bytes-per-FLOP ratio ($\beta$)**:
While a modern CPU core can execute trillions of floating-point operations per second ($4\text{ TFLOP/s}$), memory bus bandwidth is limited to only a few hundred gigabytes per second ($200\text{ GB/s}$), yielding a ratio of $\beta \approx 0.05\text{ Bytes/FLOP}$. Because the CPU can execute 20 arithmetic operations in the time it takes memory to deliver a single byte, computational execution units spend the vast majority of their operational life stalled waiting for memory transfers.

---

## 12. Quick Revision Sheet

- **Core Pyramid Hierarchy:**
  \[
  \boxed{\text{Registers (0.5 ns)} \longrightarrow \text{L1 (1 ns)} \longrightarrow \text{L2 (4 ns)} \longrightarrow \text{L3 (15 ns)} \longrightarrow \text{DRAM (80 ns)}}
  \]
- **Standard Cache Line Size:**
  \[
  \boxed{\text{Standard Cache Line} = \mathbf{64\text{ Bytes}} = 8\text{ doubles} = 16\text{ floats}}
  \]
- **Average Memory Access Time (AMAT):**
  \[
  \boxed{\text{AMAT} = \text{Hit Time} + (\text{Miss Rate} \times \text{Miss Penalty})}
  \]
- **The 4 C's of Cache Misses:**
  1. **Compulsory:** First time accessing data (cold start).
  2. **Capacity:** Working set exceeds physical cache size.
  3. **Conflict:** Multiple addresses hash to the same set.
  4. **Coherence:** Multi-core line invalidation by another core.
- **Common Exam Trap:**
  - *Trap:* Assuming the CPU loads only the 4-byte integer requested in code.
  - *Reality:* The CPU **always loads the entire 64-byte cache line** containing that integer.

---

## 13. What You Should Be Able to Do After This Topic

Before proceeding to Module 02.05, verify that you can independently:
1. Draw and annotate the complete multi-level memory hierarchy pyramid with capacities and access latencies.
2. Differentiate between Temporal Locality and Spatial Locality with clear code examples.
3. State the standard cache line size (64 bytes) and explain how cache lines accelerate sequential array traversal.
4. Classify any given cache miss into one of the 4 C's.
5. Calculate AMAT for multi-level cache systems and calculate theoretical DDR5 memory bandwidth.
