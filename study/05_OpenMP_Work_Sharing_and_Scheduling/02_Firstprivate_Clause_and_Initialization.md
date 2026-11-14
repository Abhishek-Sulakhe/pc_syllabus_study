# 02. The firstprivate Clause and Variable Initialization

---

## 1. The Basic Idea

In OpenMP parallel regions and work-sharing loops, worker threads often need to work on their own private scratchpad variables, but require that variable to start with an exact initial value derived from sequential execution (such as a calibrated scaling factor, an initial configuration struct, or a coordinate offset).

As established in data scoping fundamentals:
- The standard `private` clause allocates an uninitialized variable on each thread's private stack frame containing **unpredictable memory garbage**.
- The `shared` clause maintains a single physical address, inviting catastrophic **data races** if any thread attempts to mutate the variable.

To solve this dilemma, OpenMP provides the **`firstprivate` clause**:

```c
int base_offset = 100;

#pragma omp parallel for firstprivate(base_offset)
for (int i = 0; i < N; i++) {
    /* Each thread gets a private 'base_offset' initialized to 100 */
    base_offset += i; 
    process(base_offset);
}
```

The `firstprivate` clause combines the **memory isolation of `private`** with the **initial value inheritance of `shared`**. Each thread in the team allocates its own independent storage location and initializes it with the value that the original variable possessed immediately before the parallel or work-sharing construct was encountered.

```
                  BEFORE CONSTRUCT:
                  Master Stack: base_offset = 100
                                 |
                                 v  [#pragma omp ... firstprivate(base_offset)]
         +-----------------------+-----------------------+
         |                       |                       |
         v                       v                       v
  Thread 0 Stack          Thread 1 Stack          Thread 2 Stack
  base_offset_0 = 100     base_offset_1 = 100     base_offset_2 = 100
  (Mutations isolated)    (Mutations isolated)    (Mutations isolated)
         |                       |                       |
         +-----------------------+-----------------------+
                                 |
                                 v  [End of Construct: Thread Stacks Unwound]
                  AFTER CONSTRUCT:
                  Master Stack: base_offset = 100 (Unchanged by worker mutations)
```

---

## 2. Why We Need It

Consider an image processing filter where each pixel processing block requires an initial transformation matrix or coordinate offset:

```c
/* ATTEMPT 1: Using private() - CATASTROPHIC BUG */
int threshold = compute_dynamic_threshold(); // e.g., returns 128

#pragma omp parallel for private(threshold)
for (int i = 0; i < num_pixels; i++) {
    // BUG: threshold contains stack garbage (e.g., -842150451)!
    if (pixels[i] > threshold) {
        pixels[i] = 255;
    }
}
```

Because `private(threshold)` allocates uninitialized stack space, the condition `pixels[i] > threshold` compares against garbage values, corrupting the image.

```c
/* ATTEMPT 2: Using shared() - RACE CONDITION BUG */
int running_bias = get_initial_bias();

#pragma omp parallel for shared(running_bias)
for (int i = 0; i < N; i++) {
    // BUG: Multiple threads mutate running_bias simultaneously!
    running_bias += adjustment[i];
    output[i] = compute(running_bias);
}
```

Here, multiple threads write to `running_bias` concurrently without synchronization, causing non-deterministic data races.

### The `firstprivate` Solution
By applying `firstprivate(running_bias)`:
1. Every thread starts with the clean, validated `get_initial_bias()` value.
2. Every thread modifies its own local copy independently on its private stack.
3. No locks, no critical sections, and no data races occur.

---

## 3. How It Works: Step-by-Step Architecture and Storage Mechanics

### 3.1 Memory Allocation and Initialization Protocol
When the OpenMP runtime executes a construct annotated with `firstprivate(x)`:

1. **Stack Frame Creation:** When the outlined function (`.omp_fn`) is dispatched to the thread team, the function prologue allocates $\text{sizeof}(x)$ bytes on each thread's private stack frame:
   $$\forall\, t \in \{0, 1, \dots, T-1\}, \quad \text{Addr}(x_t) \in \text{Stack}_t$$
2. **Copy Initialization:**
   - In **C**: The master thread's variable $x_{\text{master}}$ is bitwise copied (equivalent to `memcpy(&x_t, &x_master, sizeof(x))`) into each thread's private memory location before the structured block begins execution.
   - In **C++**: The runtime invokes the **copy constructor** of the object for each thread instance:
     ```cpp
     Type x_t(x_master); // C++ Copy Construction
     ```
3. **Execution Isolation:** All references to identifier $x$ within thread $t$'s execution path resolve strictly to $x_t$.
4. **Termination Cleanup:**
   - In **C**: The thread stack is unwound at the closing brace. The updates made to $x_t$ are discarded. The original variable in the master thread's outer stack frame remains completely unchanged.
   - In **C++**: The runtime invokes the **destructor** for each private instance as threads exit the construct.

---

### 3.2 The Critical Pointer Trap: Deep vs. Shallow Copy
A notorious pitfall in C/C++ parallel programming occurs when applying `firstprivate` to a pointer.

```c
typedef struct {
    int *buffer;
    int size;
} Dataset;

Dataset d;
d.buffer = (int*)malloc(1024 * sizeof(int));
d.size = 1024;

#pragma omp parallel firstprivate(d)
{
    /* DANGER: What was copied? */
}
```

#### What Actually Happens in Memory?
OpenMP performs a **shallow bitwise copy** of the variable:
- `d.size` is copied $\implies$ each thread gets its own integer holding `1024` (Safe).
- `d.buffer` is a pointer holding the memory address `0x600000` $\implies$ each thread receives its own copy of the *pointer variable*, but **all copies point to the exact same heap memory block** at `0x600000`!

```
                SHALLOW COPY POINTER HAZARD
  Master Stack:
    d.size   = 1024
    d.buffer = 0x600000 -----------------------+
                                               |
  Thread 0 Stack (firstprivate):               |
    d_0.size   = 1024                          |
    d_0.buffer = 0x600000 ---------------------+
                                               |
  Thread 1 Stack (firstprivate):               |
    d_1.size   = 1024                          |
    d_1.buffer = 0x600000 ---------------------+---> [ HEAP BUFFER at 0x600000 ]
                                               |     (Shared concurrent access!)
  Thread 2 Stack (firstprivate):               |
    d_2.size   = 1024                          |
    d_2.buffer = 0x600000 ---------------------+
```

> [!CAUTION]
> If threads write to `d.buffer[i]` without index partitioning or synchronization, a catastrophic **data race on the heap** occurs despite the struct being marked `firstprivate`! To achieve true isolation for dynamic buffers, each thread must manually allocate its own private heap memory.

---

### 3.3 Contrast: `private` vs. `firstprivate` vs. `lastprivate`

| Clause | Storage Location | Initial Value Upon Entry | Final Value Written to Outer Scope? |
|:---|:---:|:---:|:---:|
| **`private(x)`** | Thread Stack | **Undefined (Garbage)** | **No** (Outer variable retains pre-region value) |
| **`firstprivate(x)`** | Thread Stack | **Copied from outer $x$** | **No** (Outer variable retains pre-region value) |
| **`lastprivate(x)`** | Thread Stack | **Undefined (Garbage)** | **Yes** (Value from sequentially *last* iteration) |
| **`firstprivate(x) lastprivate(x)`** | Thread Stack | **Copied from outer $x$** | **Yes** (Value from sequentially *last* iteration) |

*Note: In OpenMP, a variable can appear simultaneously in both `firstprivate` and `lastprivate` clauses! This provides full initialization upon entry AND preserves the sequentially final result upon exit.*

---

## 4. Important Mechanisms & Syntax Rules

### 4.1 Permitted Directives for `firstprivate`
The `firstprivate` clause can be attached to any OpenMP construct that establishes a private variable scope:
- `#pragma omp parallel firstprivate(...)`
- `#pragma omp for firstprivate(...)`
- `#pragma omp sections firstprivate(...)`
- `#pragma omp single firstprivate(...)`
- `#pragma omp task firstprivate(...)` *(Crucial for OpenMP tasks, covered in Lecture 13)*
- Combined constructs: `#pragma omp parallel for firstprivate(...)`

### 4.2 Type Requirements for `firstprivate`
1. In C, the variable type must be complete and copyable via assignment or `memcpy` (arrays, structs, primitives, and pointers are legal; incomplete types are forbidden).
2. In C++, classes passed to `firstprivate` must have an accessible **copy constructor** (`Class(const Class&)`). If the copy constructor is `delete`d or `private`, compilation will fail.
3. If a statically sized array `int arr[10]` is listed in `firstprivate(arr)`, OpenMP copies all 10 elements into each thread's stack frame.

---

## 5. Compilable C Code Examples

### 5.1 Program: Demonstration of `firstprivate`, `private`, and Pointer Pitfalls
Save this file as `omp_firstprivate_demo.c`:

```c
/* =========================================================================
 * File: omp_firstprivate_demo.c
 * Description: Comprehensive demonstration of:
 *              1. Initial value preservation with firstprivate
 *              2. The garbage hazard of private vs firstprivate
 *              3. Isolation of thread mutations from outer scope
 *              4. The Shallow Copy Pointer Trap
 * Compilation:
 *   gcc -O3 -fopenmp omp_firstprivate_demo.c -o omp_firstprivate_demo
 * Execution:
 *   ./omp_firstprivate_demo
 * ========================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define NUM_THREADS 4

int main(void) {
    printf("============================================================\n");
    printf("       OpenMP firstprivate Clause: Architecture & Usage     \n");
    printf("============================================================\n\n");

    /* -------------------------------------------------------------
     * EXPERIMENT 1: Contrast private vs firstprivate Initialization
     * ------------------------------------------------------------- */
    printf("--- EXPERIMENT 1: private vs firstprivate Initialization ---\n");
    int var_priv  = 500;
    int var_first = 500;

    printf("[Outer Scope] Before region: var_priv = %d, var_first = %d\n\n",
           var_priv, var_first);

    #pragma omp parallel num_threads(NUM_THREADS) \
                         private(var_priv) \
                         firstprivate(var_first)
    {
        int tid = omp_get_thread_num();

        /* Print initial values captured upon entering region */
        printf("  [Thread %d] Initial: private = %11d (Garbage) | firstprivate = %d (Inherited)\n",
               tid, var_priv, var_first);

        /* Mutate both variables locally */
        var_priv  = tid * 10;
        var_first += (tid + 1) * 100;

        printf("  [Thread %d] After local mutation: private = %2d | firstprivate = %d\n",
               tid, var_priv, var_first);
    }

    printf("\n[Outer Scope] After region finishes:\n");
    printf("  var_priv  = %d (Retained original value; worker writes discarded)\n", var_priv);
    printf("  var_first = %d (Retained original value; worker writes discarded)\n\n", var_first);

    /* -------------------------------------------------------------
     * EXPERIMENT 2: Loop Work-Sharing with firstprivate
     * ------------------------------------------------------------- */
    printf("--- EXPERIMENT 2: Loop Work-Sharing with firstprivate Offset ---\n");
    int base_multiplier = 7;
    int results[8];

    #pragma omp parallel for num_threads(NUM_THREADS) \
                             firstprivate(base_multiplier) \
                             shared(results)
    for (int i = 0; i < 8; i++) {
        int tid = omp_get_thread_num();
        
        /* Each thread computes using its own inherited copy of base_multiplier */
        results[i] = i * base_multiplier;
        
        /* Mutating local base_multiplier does not corrupt other iterations */
        base_multiplier += 1;

        printf("  [Loop Iter %d by Thread %d] results[%d] = %2d (local multiplier became %d)\n",
               i, tid, i, results[i], base_multiplier);
    }

    printf("[Outer Scope] Master base_multiplier is still = %d\n\n", base_multiplier);

    /* -------------------------------------------------------------
     * EXPERIMENT 3: The Shallow-Copy Pointer Hazard
     * ------------------------------------------------------------- */
    printf("--- EXPERIMENT 3: The Shallow Copy Pointer Hazard ---\n");
    int *shared_heap_array = (int*)malloc(NUM_THREADS * sizeof(int));
    for (int i = 0; i < NUM_THREADS; i++) shared_heap_array[i] = 0;

    /* DANGER: Passing a pointer via firstprivate copies the 8-byte pointer,
     * NOT the underlying heap array! */
    #pragma omp parallel num_threads(NUM_THREADS) firstprivate(shared_heap_array)
    {
        int tid = omp_get_thread_num();

        /* All threads have identical pointer values pointing to same heap block */
        shared_heap_array[tid] = (tid + 1) * 111;

        printf("  [Thread %d] Local pointer copy = %p -> Wrote shared_heap_array[%d] = %d\n",
               tid, (void*)shared_heap_array, tid, shared_heap_array[tid]);
    }

    printf("\n[Heap Verification] Values written into heap buffer:\n");
    for (int i = 0; i < NUM_THREADS; i++) {
        printf("  shared_heap_array[%d] = %d\n", i, shared_heap_array[i]);
    }

    free(shared_heap_array);
    printf("============================================================\n");

    return EXIT_SUCCESS;
}
```

### 5.2 Build Command and Execution Output

```bash
$ gcc -O3 -fopenmp omp_firstprivate_demo.c -o omp_firstprivate_demo
$ ./omp_firstprivate_demo
```

```text
============================================================
       OpenMP firstprivate Clause: Architecture & Usage     
============================================================

--- EXPERIMENT 1: private vs firstprivate Initialization ---
[Outer Scope] Before region: var_priv = 500, var_first = 500

  [Thread 0] Initial: private =       32767 (Garbage) | firstprivate = 500 (Inherited)
  [Thread 0] After local mutation: private =  0 | firstprivate = 600
  [Thread 2] Initial: private =           0 (Garbage) | firstprivate = 500 (Inherited)
  [Thread 2] After local mutation: private = 20 | firstprivate = 800
  [Thread 1] Initial: private =           0 (Garbage) | firstprivate = 500 (Inherited)
  [Thread 1] After local mutation: private = 10 | firstprivate = 700
  [Thread 3] Initial: private = -1429810240 (Garbage) | firstprivate = 500 (Inherited)
  [Thread 3] After local mutation: private = 30 | firstprivate = 900

[Outer Scope] After region finishes:
  var_priv  = 500 (Retained original value; worker writes discarded)
  var_first = 500 (Retained original value; worker writes discarded)

--- EXPERIMENT 2: Loop Work-Sharing with firstprivate Offset ---
  [Loop Iter 0 by Thread 0] results[0] =  0 (local multiplier became 8)
  [Loop Iter 1 by Thread 0] results[1] =  8 (local multiplier became 9)
  [Loop Iter 2 by Thread 1] results[2] = 14 (local multiplier became 8)
  [Loop Iter 3 by Thread 1] results[3] = 24 (local multiplier became 9)
  [Loop Iter 4 by Thread 2] results[4] = 28 (local multiplier became 8)
  [Loop Iter 5 by Thread 2] results[5] = 40 (local multiplier became 9)
  [Loop Iter 6 by Thread 3] results[6] = 42 (local multiplier became 8)
  [Loop Iter 7 by Thread 3] results[7] = 56 (local multiplier became 9)
[Outer Scope] Master base_multiplier is still = 7

--- EXPERIMENT 3: The Shallow Copy Pointer Hazard ---
  [Thread 0] Local pointer copy = 0x55d491f0a2a0 -> Wrote shared_heap_array[0] = 111
  [Thread 2] Local pointer copy = 0x55d491f0a2a0 -> Wrote shared_heap_array[2] = 333
  [Thread 1] Local pointer copy = 0x55d491f0a2a0 -> Wrote shared_heap_array[1] = 222
  [Thread 3] Local pointer copy = 0x55d491f0a2a0 -> Wrote shared_heap_array[3] = 444

[Heap Verification] Values written into heap buffer:
  shared_heap_array[0] = 111
  shared_heap_array[1] = 222
  shared_heap_array[2] = 333
  shared_heap_array[3] = 444
============================================================
```

Notice:
1. In Experiment 1, every thread's `var_first` cleanly began at `500`, whereas `var_priv` held arbitrary garbage.
2. In Experiment 2, each thread received its own `base_multiplier` starting at `7`, and thread-local increments did not alter the master thread's outer variable.
3. In Experiment 3, all threads printed the **exact same pointer address** (`0x55d491f0a2a0`), proving that `firstprivate` on a pointer performs only a shallow copy.

---

## 6. Mathematical Model & Memory Complexity

### 6.1 Initialization Latency and Memory Overhead
When $T$ threads enter a region with a list of $K$ variables marked `firstprivate`:

$$\text{Memory Overhead} = \sum_{k=1}^K T \cdot \text{sizeof}(\text{var}_k)$$

$$\text{Copy Cycles Overhead} \approx \sum_{k=1}^K T \cdot \left\lceil \frac{\text{sizeof}(\text{var}_k)}{W_{\text{bus}}} \right\rceil$$

Where $W_{\text{bus}}$ is the L1 cache / CPU register transfer width (typically 8 bytes for 64-bit architectures, or 32/64 bytes for AVX vector registers).

#### Scalability Implication:
- For primitive scalar variables (`int`, `double`), $\text{sizeof} \le 8$ bytes $\implies$ copy overhead is negligible ($< 5$ clock cycles per thread).
- For large structs (e.g., a $64\,\text{KB}$ configuration structure), copying across $T = 128$ threads requires copying $128 \times 64\,\text{KB} = 8\,\text{MB}$ of data into L1/L2 caches upon region entry. This introduces cache pollution and synchronization stalls.

---

## 7. Advantages & Limitations

### Advantages
1. **Thread-Safe Initial State:** Guarantees that every worker thread starts with a known, valid application state without needing manual initialization loops.
2. **Elimination of Critical Sections:** By giving each thread its own mutable copy, code that previously required synchronization locks can run concurrently at full speed.
3. **Encapsulation:** Thread-local mutations are completely isolated and cannot corrupt outer program state.

### Limitations & Traps
1. **Shallow Copy Hazard:** Pointers inside structs are copied verbatim; dynamic memory buffers remain shared.
2. **No Value Write-Back:** Changes made to a `firstprivate` variable are discarded upon construct exit (use `lastprivate` if the final iteration value is needed).
3. **Memory Footprint for Structs:** Large data types incur both stack memory consumption and initialization copy overhead.

---

## 8. Formal Definitions for Exam

- **`firstprivate` Clause:** An OpenMP data scoping clause providing private storage for listed variables on each thread's stack frame, initializing each private instance with the value of the original master variable immediately prior to entering the construct.
- **Copy Construction:** In C++, the process of initializing a new object from an existing object of the same class, invoked automatically for each thread-private instance under `firstprivate`.
- **Bitwise Copy (Shallow Copy):** The default C mechanism used by OpenMP to copy the raw memory bytes of a variable from master stack to thread stack, copying pointer addresses rather than the underlying referenced heap memory.
- **`lastprivate` Clause:** An OpenMP clause that causes the original outer variable to be updated with the value computed by whichever thread executed the sequentially last iteration of a loop or last section of a sections construct.

---

## 9. Comparison Table

### Scoping Matrix: Initial Value vs. Final Value Behavior

| Clause | Memory Location | Value on Construct Entry | Value on Construct Exit | Common HPC Use Case |
|:---|:---:|:---:|:---:|:---|
| **`shared`** | Master / Heap | Preserved | Preserves latest write | Read-only inputs, large shared arrays |
| **`private`** | Thread Stack | **Garbage (Undefined)** | Discarded | Scratchpad index variables, temporary loop math |
| **`firstprivate`** | Thread Stack | **Inherited from master** | Discarded | Thread-local coefficients, base offsets, filters |
| **`lastprivate`** | Thread Stack | **Garbage (Undefined)** | **Written back to master** | Finding index of match, loop tail states |
| **`firstprivate` + `lastprivate`** | Thread Stack | **Inherited from master** | **Written back to master** | Cumulative state that enters and leaves loop |

---

## 10. Solved Analytical & Code-Tracing Problems

### Problem 1: Code Trace with `firstprivate` vs `private`
**Question:** Predict the exact printed output of the following program:

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    int x = 10, y = 20;

    #pragma omp parallel num_threads(2) firstprivate(x) private(y)
    {
        int tid = omp_get_thread_num();
        x += tid + 5;
        y = tid + 100;
    }

    printf("x = %d, y = %d\n", x, y);
    return 0;
}
```

**Step-by-Step Analysis:**
1. **Variable `x`:**
   - Scoped as `firstprivate(x)`.
   - Initial value before region: $10$.
   - Thread 0 creates local $x_0 = 10$, computes $x_0 = 10 + 0 + 5 = 15$.
   - Thread 1 creates local $x_1 = 10$, computes $x_1 = 10 + 1 + 5 = 16$.
   - At construct exit, local $x_0$ and $x_1$ are destroyed.
   - The master variable $x$ in `main()` was never modified $\implies \mathbf{x = 10}$.
2. **Variable `y`:**
   - Scoped as `private(y)`.
   - Initial value before region: $20$.
   - Thread 0 writes local $y_0 = 100$.
   - Thread 1 writes local $y_1 = 101$.
   - At construct exit, local $y_0$ and $y_1$ are destroyed.
   - The master variable $y$ in `main()` retains its original value $\implies \mathbf{y = 20}$.
3. **Final Printed Output:**
   ```text
   x = 10, y = 20
   ```

---

### Problem 2: Simultaneous `firstprivate` and `lastprivate`
**Question:** Explain what the following code prints, and why combining `firstprivate` and `lastprivate` is necessary here.

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    int state = 50;

    #pragma omp parallel for num_threads(4) firstprivate(state) lastprivate(state)
    for (int i = 0; i < 4; i++) {
        state += i;
    }

    printf("Final state = %d\n", state);
    return 0;
}
```

**Solution:**
1. **`firstprivate(state)` Role:**
   - Every thread receives a local `state` initialized to $50$.
   - Thread 0 executes iteration $i = 0 \implies \text{state}_0 = 50 + 0 = 50$.
   - Thread 1 executes iteration $i = 1 \implies \text{state}_1 = 50 + 1 = 51$.
   - Thread 2 executes iteration $i = 2 \implies \text{state}_2 = 50 + 2 = 52$.
   - Thread 3 executes iteration $i = 3 \implies \text{state}_3 = 50 + 3 = 53$.
2. **`lastprivate(state)` Role:**
   - The sequentially *last* iteration of the loop is $i = 3$, executed by **Thread 3**.
   - Upon loop completion, the OpenMP runtime copies Thread 3's final value ($\text{state}_3 = 53$) back into the master thread's outer variable `state`.
3. **Final Output:**
   ```text
   Final state = 53
   ```
4. **Why Both Were Needed:**
   - If only `lastprivate` had been used, `state` would have started with uninitialized garbage on entry.
   - If only `firstprivate` had been used, the final result ($53$) would have been lost upon exit, printing $50$.

---

## 11. Frequently Asked Conceptual Questions & Answers

### Q1: Can a dynamically allocated array (`int *arr = malloc(...)`) be made truly private using `firstprivate`?
**Answer:** **NO.** In C/C++, `firstprivate(arr)` will copy the pointer `arr` (the memory address), but will **not** copy the memory block pointed to on the heap. All threads will end up with identical pointer values pointing to the single shared heap buffer. To make dynamic heap data truly private, each thread must call `malloc()` individually inside the parallel region.

### Q2: What is the computational cost of using `firstprivate` on a large struct?
**Answer:** The cost is proportional to $T \times \text{sizeof}(\text{struct})$. If a struct is $1\,\text{MB}$ and you run on $64$ threads, the OpenMP master thread and runtime must copy $64\,\text{MB}$ of data across thread stacks upon entering the region. This wastes CPU memory bandwidth and pollutes the CPU cache hierarchy. For large read-only data, `shared` (with `const` qualifiers) is vastly superior.

### Q3: Why does OpenMP allow combining `firstprivate` and `lastprivate` on the same variable, but forbids combining `private` and `firstprivate`?
**Answer:**
- Combining `private` and `firstprivate` is semantically contradictory: `private` specifies that the variable should NOT be initialized from the outer variable, while `firstprivate` specifies that it MUST be initialized from the outer variable.
- In contrast, `firstprivate` governs **entry semantics** (initialization from outer scope), while `lastprivate` governs **exit semantics** (copying the sequentially last iteration back to outer scope). They operate at completely different points in the construct lifecycle and complement each other.

---

## 12. Quick Revision Sheet

```
+----------------------------------------------------------------------------------------------------+
|                                    FIRSTPRIVATE CHEAT SHEET                                        |
+----------------------------------------------------------------------------------------------------+
| Core Purpose           | Provides isolated thread-stack variable initialized with pre-region value. |
| Syntax                 | #pragma omp [parallel|for|sections|single|task] firstprivate(list)        |
| Initialization Method  | C: Bitwise memcpy from master stack. C++: Copy Constructor invoked.        |
| Termination Behavior   | Updates by worker threads are DISCARDED; outer master variable unchanged.  |
| The Pointer Hazard     | Shallow copy only! Pointers are duplicated, but heap memory remains shared.|
+----------------------------------------------------------------------------------------------------+
|                                     SCOPING COMPARISON                                             |
+----------------------------------------------------------------------------------------------------+
| private(x)             | Stack allocated, UNINITIALIZED GARBAGE, writes discarded upon exit.       |
| firstprivate(x)        | Stack allocated, COPY-INITIALIZED, writes discarded upon exit.             |
| lastprivate(x)         | Stack allocated, UNINITIALIZED, sequentially LAST iteration writes back.   |
| firstprivate + lastpriv| Stack allocated, COPY-INITIALIZED on entry, LAST iteration writes back.    |
+----------------------------------------------------------------------------------------------------+
|                                   PERFORMANCE ADVICE                                               |
+----------------------------------------------------------------------------------------------------+
| Scalars (int, double)  | Free ($< 5$ clock cycles). Ideal for thread-local multipliers and offsets.|
| Large Structs / Buffers| Avoid! Causes high copy overhead and thread stack memory exhaustion.       |
+----------------------------------------------------------------------------------------------------+
```

---

## 13. What You Should Be Able to Do

After studying this topic thoroughly, you should be able to:
1. Explain the architectural and memory differences between `private` and `firstprivate` in terms of stack allocation and bitwise copying.
2. Identify and resolve data races and uninitialized memory bugs by applying `firstprivate`.
3. Predict the exact variable values printed before, during, and after parallel regions utilizing `firstprivate` and `lastprivate`.
4. Explain the Shallow Copy Pointer Trap when applying `firstprivate` to dynamic memory structures.
5. Contrast C `memcpy` semantics with C++ copy-constructor semantics under `firstprivate`.
