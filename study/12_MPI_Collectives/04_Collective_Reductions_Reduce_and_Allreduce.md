# Module 12: MPI Collective Operations (Lectures 30–31)
# Subtopic 12.04: Collective Reductions (`MPI_Reduce`, `MPI_Allreduce`, Predefined Operators, Floating-Point Non-Associativity, Custom `MPI_Op`)

---

## 1. Basic Idea

In parallel programming, a **Reduction Operation** combines a collection of distributed data values across multiple processes into a single aggregated result using an associative binary operator $\oplus$ (such as addition, multiplication, minimum, or maximum):

$$\text{Result} = V_0 \oplus V_1 \oplus V_2 \oplus \dots \oplus V_{P-1}$$

The Message Passing Interface provides two primary collective reduction primitives:

1. **`MPI_Reduce` (All-to-One Reduction):**
   Every process contributes a local operand buffer. The reduction operator is applied across all processes, and the final aggregated result is stored **exclusively in the receive buffer of the Root process**. Non-root processes receive nothing.

2. **`MPI_Allreduce` (All-to-All Reduction):**
   Applies the exact same reduction operator across all distributed operands, but the final aggregated result is delivered into the receive buffers of **every process** in the communicator. It is functionally equivalent to an `MPI_Reduce` followed by an `MPI_Bcast`, executed in a single communication phase.

```
+-----------------------------------------------------------------------------------------+
|                              The MPI Reduction Workflow                                 |
+-----------------------------------------------------------------------------------------+
| LOCAL OPERANDS (e.g., local partial sums):                                              |
|   Rank 0: [ 10 ]      Rank 1: [ 20 ]      Rank 2: [ 30 ]      Rank 3: [ 40 ]            |
|                                                                                         |
| MPI_Reduce (Operator = MPI_SUM, Root = 0):                                              |
|   Rank 0 Receive Buffer: [ 100 ]  <=== Aggregated Sum (10 + 20 + 30 + 40)               |
|   Ranks 1, 2, 3        : Receive nothing.                                               |
|                                                                                         |
| MPI_Allreduce (Operator = MPI_SUM):                                                     |
|   Rank 0 Receive Buffer: [ 100 ]                                                        |
|   Rank 1 Receive Buffer: [ 100 ]  <=== Identical aggregated result delivered to ALL!     |
|   Rank 2 Receive Buffer: [ 100 ]                                                        |
|   Rank 3 Receive Buffer: [ 100 ]                                                        |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Why We Need It

### 1. The Core of Iterative Convergent Solvers
In scientific computing (e.g., iterative linear system solvers like Conjugate Gradient or Jacobi relaxation):
- Each process computes a local residual error $e_{\text{local}} = \sum (x_i - x_{\text{old}})^2$.
- The global residual error $E_{\text{global}} = \sum e_{\text{local}}$ must be compared against a convergence threshold $\epsilon$ (e.g., $10^{-6}$).
- Because **every process must independently decide** whether to continue iterating or terminate, all processes need the global error value. `MPI_Allreduce` calculates the global sum and updates all ranks in a single step.

### 2. Eliminating $\mathcal{O}(P)$ Serialization
A naive reduction implemented with a loop of `MPI_Send` calls to Rank 0 requires $P-1$ sequential additions:
$$T_{\text{naive}} = (P - 1) \cdot (\alpha + \beta m + t_{\text{add}}) = \mathcal{O}(P)$$
`MPI_Reduce` and `MPI_Allreduce` implement **Binomial Trees** and **Recursive Doubling (Butterfly)** algorithms, completing the global reduction in:
$$T_{\text{reduction}} = \lceil \log_2 P \rceil \cdot (\alpha + \beta m + t_{\text{add}}) = \mathcal{O}(\log P)$$
On $P = 4{,}096$ cores, this cuts execution time from $4{,}095$ steps to just $12$ steps!

### 3. Locating Global Extrema (`MPI_MAXLOC` / `MPI_MINLOC`)
In particle physics and optimization, finding not only the global maximum temperature or energy, but also **which process rank and array index owns that maximum**, is a common requirement. `MPI_MAXLOC` computes the maximum value and the rank identifier atomically.

---

## 3. How It Works

### The Recursive Doubling / Butterfly Algorithm in `MPI_Allreduce`

For power-of-two process counts ($P = 2^k$), `MPI_Allreduce` uses a **Recursive Doubling (Hypercube / Butterfly)** topology:

```
Step 1: Exchange with neighbor at distance 1 (Pairs: (0,1), (2,3))
  Rank 0 <===> Rank 1  (Both now have V0 + V1)
  Rank 2 <===> Rank 3  (Both now have V2 + V3)

Step 2: Exchange with partner at distance 2 (Pairs: (0,2), (1,3))
  Rank 0 <===> Rank 2  (Both now have V0 + V1 + V2 + V3)
  Rank 1 <===> Rank 3  (Both now have V0 + V1 + V2 + V3)
```

$$\text{Total Communication Steps} = \log_2(P)$$

Every process performs identical communication and addition steps concurrently. When $\log_2(P)$ steps complete, **every process holds the exact global sum simultaneously**—with zero broadcast overhead!

---

### Mathematical Invariants: Associativity and Commutativity

For a reduction operator $\oplus$ to be legally used in MPI, it must satisfy fundamental mathematical axioms:

1. **Associativity (Mandatory):**
   $$(a \oplus b) \oplus c = a \oplus (b \oplus c)$$
   Associativity allows the MPI runtime to group reductions into arbitrary binary trees (e.g., combining $(P_0 + P_1)$ on Core 0 while combining $(P_2 + P_3)$ on Core 2) without changing the result.

2. **Commutativity (Default Assumption):**
   $$a \oplus b = b \oplus a$$
   Commutativity allows the runtime to combine incoming messages in arbitrary arrival order. (User-defined operators can optionally declare whether they are non-commutative).

---

### The Floating-Point Non-Associativity Trap

In pure mathematics, real addition is associative: $(a + b) + c = a + (b + c)$.
However, in **IEEE 754 Floating-Point Computer Arithmetic**, addition is **NOT strictly associative**:

$$(a + b) + c \neq a + (b + c) \quad \text{(due to finite precision rounding)}$$

#### Demonstration:
Consider 64-bit double precision floats:
- Let $a = 10^{20}$
- Let $b = -10^{20}$
- Let $c = 3.1415926535$

$$\text{Evaluation 1 (Sequential Left-to-Right): } (a + b) + c = (10^{20} - 10^{20}) + 3.14159 = 0 + 3.14159 = \mathbf{3.14159}$$

$$\text{Evaluation 2 (Right-to-Left / Tree Order): } a + (b + c) = 10^{20} + (-10^{20} + 3.14159) = 10^{20} + (-10^{20}) = \mathbf{0.0}$$
*(In Evaluation 2, $3.14159$ is smaller than the precision threshold of $10^{20}$ and is completely lost to machine epsilon rounding).*

> [!IMPORTANT]
> **Exam & Production Principle:**
> Because parallel reductions evaluate sums in tree order (differing from sequential linear loops), **the parallel sum of floating-point numbers will rarely be bit-for-bit identical to the sequential sum!** 
> 
> Furthermore, running the same MPI reduction with different numbers of processes ($P = 4$ vs $P = 8$) changes the tree structure, producing slight variations in the least significant floating-point bits. This is normal mathematical behavior, not an MPI software bug.

---

## 4. Important Mechanisms and Functions

### 1. `MPI_Reduce` and `MPI_Allreduce` Function Signatures

```c
#include <mpi.h>

int MPI_Reduce(const void *sendbuf, void *recvbuf, int count,
               MPI_Datatype datatype, MPI_Op op, int root, MPI_Comm comm);

int MPI_Allreduce(const void *sendbuf, void *recvbuf, int count,
                  MPI_Datatype datatype, MPI_Op op, MPI_Comm comm);
```

- **`sendbuf`:** Pointer to local input data (operand) on all ranks.
- **`recvbuf`:** Pointer to destination memory:
  - In `MPI_Reduce`: Valid and written **only on Root** (ignored / can be `NULL` on non-root).
  - In `MPI_Allreduce`: Valid and written on **all ranks**.
- **`count`:** Number of elements in `sendbuf` (and `recvbuf`). If `count > 1`, the reduction operator is applied **element-by-element** across the arrays!
- **`op`:** Reduction operator handle (e.g., `MPI_SUM`).
- **`root`:** Rank of result recipient in `MPI_Reduce` (omitted in `MPI_Allreduce`).

---

### 2. Predefined MPI Reduction Operators

| Predefined Operator (`MPI_Op`) | Mathematical Operation | Applicable Types |
| :--- | :--- | :--- |
| **`MPI_SUM`** | Summation ($\sum x_i$) | Integer, Floating-Point |
| **`MPI_PROD`** | Product ($\prod x_i$) | Integer, Floating-Point |
| **`MPI_MAX`** | Maximum value | Integer, Floating-Point |
| **`MPI_MIN`** | Minimum value | Integer, Floating-Point |
| **`MPI_MAXLOC`** | Maximum value and rank/index of owner | Struct of (Value, Rank/Index) |
| **`MPI_MINLOC`** | Minimum value and rank/index of owner | Struct of (Value, Rank/Index) |
| **`MPI_LAND`** | Logical AND ($a \land b$) | Integer, C++ bool |
| **`MPI_LOR`** | Logical OR ($a \lor b$) | Integer, C++ bool |
| **`MPI_BAND`** | Bitwise AND ($a \ \& \ b$) | Integer, Byte |
| **`MPI_BOR`** | Bitwise OR ($a \mid b$) | Integer, Byte |
| **`MPI_BXOR`** | Bitwise XOR ($a \oplus b$) | Integer, Byte |

---

### 3. In-Place Reductions: `MPI_IN_PLACE`

To avoid allocating a separate `recvbuf`, processes can specify `MPI_IN_PLACE`:
- In `MPI_Reduce`: On the **Root rank only**, pass `sendbuf = MPI_IN_PLACE`. The root's initial value is taken from `recvbuf`, and the final aggregated result overwrites `recvbuf`.
- In `MPI_Allreduce`: **All ranks** pass `sendbuf = MPI_IN_PLACE`. Each rank's local value in `recvbuf` is replaced with the global reduced value.

```c
/* In-Place Allreduce on all ranks */
double value = calculate_local_energy();
MPI_Allreduce(MPI_IN_PLACE, &value, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
// Now 'value' contains the global sum on every process!
```

---

### 4. Custom User-Defined Reduction Operators (`MPI_Op_create`)

Developers can register custom mathematical reductions (e.g., matrix addition, complex number multiplication, or 3D vector combinations):

```c
typedef void MPI_User_function(void *invec, void *inoutvec, int *len,
                               MPI_Datatype *datatype);

int MPI_Op_create(MPI_User_function *user_fn, int commute, MPI_Op *op);
int MPI_Op_free(MPI_Op *op);
```

- **`user_fn`:** Callback function executing the element-wise operation:
  `inoutvec[i] = inoutvec[i] OP invec[i]`.
- **`commute`:** Set to `1` (true) if the operation is commutative ($a \oplus b = b \oplus a$); set to `0` if non-commutative.

---

## 5. Compilable C Code Examples

### Example 1: Parallel Numerical Integration for $\pi$ via `MPI_Reduce`
This complete program computes $\pi$ via numerical integration:
$$\int_0^1 \frac{4}{1 + x^2} \, dx = \pi$$
Each rank computes a local slice of trapezoids, and `MPI_Reduce` aggregates the partial sums onto Rank 0.

```c
/*
 * File: mpi_pi_reduce.c
 * Compilation:
 *   mpicc -O2 mpi_pi_reduce.c -o mpi_pi_reduce
 *
 * Execution:
 *   mpirun -np 4 ./mpi_pi_reduce
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

#define NUM_STEPS 100000000L /* 100 Million Steps */

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    double step_size = 1.0 / (double)NUM_STEPS;
    double local_sum = 0.0;
    double global_sum = 0.0;

    double t_start = MPI_Wtime();

    /* Partition 100 million steps across 'size' ranks in cyclic fashion */
    for (long i = rank; i < NUM_STEPS; i += size) {
        double x = (i + 0.5) * step_size;
        local_sum += 4.0 / (1.0 + x * x);
    }

    /* Multiply local sum by step size */
    double local_pi_slice = local_sum * step_size;

    printf("[Rank %02d] Local partial sum computed = %.10f\n", rank, local_pi_slice);

    /* =========================================================================
     * COLLECTIVE REDUCTION: MPI_Reduce with MPI_SUM operator
     * All ranks contribute local_pi_slice; Root (Rank 0) receives global_sum
     * ========================================================================= */
    MPI_Reduce(
        &local_pi_slice, &global_sum, 1, MPI_DOUBLE,
        MPI_SUM, 0 /* root */, MPI_COMM_WORLD
    );

    double t_end = MPI_Wtime();

    if (rank == 0) {
        double pi_exact = 3.14159265358979323846;
        double error = fabs(global_sum - pi_exact);

        printf("\n=================================================================\n");
        printf("  NUMERICAL INTEGRATION OF PI VIA MPI_Reduce                     \n");
        printf("  Total Intervals : %ld across %d processes                      \n", NUM_STEPS, size);
        printf("  Calculated Pi   : %.14f                                        \n", global_sum);
        printf("  Exact Pi        : %.14f                                        \n", pi_exact);
        printf("  Absolute Error  : %.3e                                         \n", error);
        printf("  Execution Time  : %.4f seconds                                 \n", t_end - t_start);
        printf("=================================================================\n");
    }

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[Rank 00] Local partial sum computed = 0.7853981634
[Rank 01] Local partial sum computed = 0.7853981634
[Rank 02] Local partial sum computed = 0.7853981634
[Rank 03] Local partial sum computed = 0.7853981634

=================================================================
  NUMERICAL INTEGRATION OF PI VIA MPI_Reduce                     
  Total Intervals : 100000000 across 4 processes                      
  Calculated Pi   : 3.14159265358979                                        
  Exact Pi        : 3.14159265358979                                        
  Absolute Error  : 4.441e-16                                         
  Execution Time  : 0.0612 seconds                                 
=================================================================
```

---

### Example 2: Finding Global Maximum and Owner Rank via `MPI_MAXLOC`
This program demonstrates finding the global maximum value and the rank that generated it using `MPI_MAXLOC`.

```c
/*
 * File: mpi_maxloc_demo.c
 * Compilation:
 *   mpicc -O2 mpi_maxloc_demo.c -o mpi_maxloc_demo
 *
 * Execution:
 *   mpirun -np 4 ./mpi_maxloc_demo
 */

#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* Structure format required by MPI_DOUBLE_INT */
    struct {
        double val;
        int rank;
    } local_input, global_result;

    /* Seed differently per rank */
    srand((unsigned int)(time(NULL) + rank * 100));
    local_input.val  = (double)(rand() % 1000) / 10.0; /* Random float 0.0 - 99.9 */
    local_input.rank = rank;

    printf("[Rank %d] Generated Local Value: %.1f\n", rank, local_input.val);

    /* =========================================================================
     * ALLREDUCE WITH MPI_MAXLOC
     * Every rank receives the maximum value AND the rank that generated it!
     * ========================================================================= */
    MPI_Allreduce(
        &local_input, &global_result, 1,
        MPI_DOUBLE_INT, MPI_MAXLOC, MPI_COMM_WORLD
    );

    printf("  >>> [Rank %d] Agrees: Global Maximum = %.1f (Owned by Rank %d) <<<\n",
           rank, global_result.val, global_result.rank);

    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

```
--- Expected Output (Running with 4 Processes) ---
[Rank 0] Generated Local Value: 42.1
[Rank 1] Generated Local Value: 89.7
[Rank 2] Generated Local Value: 12.3
[Rank 3] Generated Local Value: 75.4
  >>> [Rank 0] Agrees: Global Maximum = 89.7 (Owned by Rank 1) <<<
  >>> [Rank 1] Agrees: Global Maximum = 89.7 (Owned by Rank 1) <<<
  >>> [Rank 2] Agrees: Global Maximum = 89.7 (Owned by Rank 1) <<<
  >>> [Rank 3] Agrees: Global Maximum = 89.7 (Owned by Rank 1) <<<
```

---

## 6. Mathematical Formulations & Reduction Tree Scaling

### Hockney Latency Model for Reductions

Let $\alpha$ be network latency, $\beta$ be byte transmission time, $m$ be element count, and $\gamma$ be the local CPU arithmetic time per element operation.

#### 1. Naive Linear Loop on Root:
$$T_{\text{naive}} = (P - 1) \cdot \alpha + (P - 1) \cdot m \cdot \beta + (P - 1) \cdot m \cdot \gamma = \mathcal{O}(P)$$

#### 2. Binomial Tree Collective Reduction (`MPI_Reduce`):
In each of the $\lceil \log_2 P \rceil$ stages, data is sent across the network and added locally:

$$T_{\text{reduce}} = \lceil \log_2(P) \rceil \cdot \left( \alpha + m \cdot \beta + m \cdot \gamma \right) = \mathcal{O}(\log P)$$

#### 3. Recursive Doubling (`MPI_Allreduce`):
Every rank transmits and computes at each step of the butterfly:

$$T_{\text{allreduce}} = \lceil \log_2(P) \rceil \cdot \left( \alpha + m \cdot \beta + m \cdot \gamma \right) = \mathcal{O}(\log P)$$

#### Analytical Scaling Comparison across Core Counts ($P$):

| Process Count ($P$) | Naive Reduction Steps | Collective Tree Steps | Reduction Acceleration |
| :--- | :--- | :--- | :--- |
| **$P = 8$** | $7$ | $3$ | $2.3\times$ faster |
| **$P = 64$** | $63$ | $6$ | $10.5\times$ faster |
| **$P = 1{,}024$** | $1{,}023$ | $10$ | **$102.3\times$ faster** |
| **$P = 65{,}536$** | $65{,}535$ | $16$ | **$4{,}095.9\times$ faster** |

---

## 7. Advantages, Disadvantages & Limitations

### Comprehensive Trade-off Matrix

```
+---------------------------+-----------------------------------------+-----------------------------------------+
| Feature                   | Key Advantages                          | Critical Limitations / Risks            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Reduce`              | - Logarithmic scaling O(log P)          | - Result delivered ONLY to Root rank;   |
|                           | - Highly optimized vendor tree networks |   workers have no visibility            |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_Allreduce`           | - Delivers result to ALL ranks          | - Incurs slightly higher network volume |
|                           | - 2x faster than Reduce + Broadcast     |   than pure Reduce                      |
+---------------------------+-----------------------------------------+-----------------------------------------+
| `MPI_MAXLOC` / `MINLOC`   | - Atomically finds extremum and rank ID | - Requires paired datatypes             |
|                           |   in a single communication pass        |   (e.g., MPI_DOUBLE_INT)                |
+---------------------------+-----------------------------------------+-----------------------------------------+
| User-Defined `MPI_Op`     | - Allows custom complex algebra         | - Cannot be hardware offloaded as easily|
|                           |   (matrices, quaternions, geometry)     |   as native MPI_SUM                     |
+---------------------------+-----------------------------------------+-----------------------------------------+
```

---

## 8. Standard Exam Definitions

> [!IMPORTANT]
> **Collective Reduction (`MPI_Reduce`):** An all-to-one collective communication routine that applies an associative mathematical operator (such as addition or maximum) across distributed arrays from all processes, storing the final accumulated result exclusively on the Root process.
>
> **Allreduce (`MPI_Allreduce`):** An all-to-all collective reduction routine that applies an associative operator across all process contributions, delivering the identical final accumulated result into the receive buffers of every process in the communicator.
>
> **Reduction Operator (`MPI_Op`):** A handle representing a binary mathematical operation (predefined like `MPI_SUM` or custom-created via `MPI_Op_create`) that must be associative for parallel tree execution.
>
> **`MPI_MAXLOC`:** A specialized reduction operator that evaluates an array of pairs, returning both the global maximum value and the rank identifier (or index) of the process that generated that maximum.
>
> **Floating-Point Non-Associativity:** The mathematical reality that finite-precision computer floating-point addition violates the associative law ($(a+b)+c \neq a+(b+c)$), causing parallel reduction sums to vary slightly from sequential sums and across differing core counts.

---

## 9. Comparison Tables

### `MPI_Reduce` vs `MPI_Allreduce`

| Feature | `MPI_Reduce` | `MPI_Allreduce` |
| :--- | :--- | :--- |
| **Destination of Result** | **Root rank only** | **All processes** in communicator |
| **Root Parameter?** | **Yes** (`int root`) | **No** (Symmetric) |
| **Typical Use Case** | Final output printing, writing to disk | Convergence testing in iterative loops |
| **Equivalence** | Standalone reduction | Equivalent to `MPI_Reduce` + `MPI_Bcast` |
| **`MPI_IN_PLACE` Support**| Supported on Root only | Supported on **all** processes |

---

## 10. Solved Problems / Analytical Exercises

### Problem 1: Vector Reductions with `count > 1`
**Question:** Explain the operation when `count = 3` is passed to `MPI_Reduce` with `MPI_SUM`:
```c
double local_vec[3] = {rank, 10.0, 100.0};
double global_vec[3];
MPI_Reduce(local_vec, global_vec, 3, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
```
Across 4 processes ($P = 4$), what is the exact content of `global_vec` on Rank 0?

**Solution:**
- In MPI reductions, when `count > 1`, the reduction operator is applied **element-by-element (component-wise)** across the corresponding indices of all vectors:
  - $\text{global\_vec}[0] = \sum_{r=0}^3 \text{local\_vec}[0] = 0 + 1 + 2 + 3 = \mathbf{6.0}$
  - $\text{global\_vec}[1] = \sum_{r=0}^3 \text{local\_vec}[1] = 10.0 + 10.0 + 10.0 + 10.0 = \mathbf{40.0}$
  - $\text{global\_vec}[2] = \sum_{r=0}^3 \text{local\_vec}[2] = 100.0 + 100.0 + 100.0 + 100.0 = \mathbf{400.0}$
- On Rank 0: `global_vec = [ 6.0, 40.0, 400.0 ]`.
- Non-root ranks leave `global_vec` untouched.

---

### Problem 2: Tie-Breaking in `MPI_MAXLOC`
**Question:** If Rank 1 and Rank 3 both generate the identical maximum value ($99.9$), what does `MPI_MAXLOC` return as the owner rank?

**Solution:**
- The MPI specification explicitly defines tie-breaking behavior for `MPI_MAXLOC` and `MPI_MINLOC`:
- If two or more processes generate the identical maximum value, the operator selects the process with the **smallest rank number**.
- Therefore, `MPI_MAXLOC` returns value $= 99.9$ and rank $= \mathbf{1}$.

---

## 11. Conceptual Questions & Answers (Exam & Viva Focus)

### Q1: Why must reduction operators be associative in MPI?
**Answer:** Because MPI implementations execute reductions across logarithmic trees (such as binomial trees or butterflies) rather than sequentially. In a tree, values from different processes are combined concurrently in arbitrary binary sub-trees (e.g., $((P_0 \oplus P_1) \oplus (P_2 \oplus P_3))$). If the operator were not associative ($(a \oplus b) \oplus c \neq a \oplus (b \oplus c)$), evaluating in different tree topologies would produce completely different and invalid mathematical results.

### Q2: Why is `MPI_Allreduce` preferred over `MPI_Reduce` followed by `MPI_Bcast` in iterative loops?
**Answer:** While both produce the identical data state on all ranks, calling `MPI_Reduce` to root followed by `MPI_Bcast` requires two distinct communication rounds: all data must first travel up the tree to the root node, and then be re-broadcast back down the tree. In contrast, `MPI_Allreduce` uses a single butterfly or ring algorithm where processes exchange and reduce data concurrently in a single phase, saving up to $50\%$ in communication latency and avoiding the root node memory and network bottleneck.

### Q3: What is the purpose of the `len` argument in custom user reduction functions (`MPI_User_function`)?
**Answer:** When an application calls `MPI_Reduce` with `count > 1`, the MPI library passes arrays of elements to the reduction function. The `*len` parameter tells the user's custom function how many elements to process in the current vector chunk, allowing the user function to loop from `0` to `*len - 1` applying the custom operation element-wise.

---

## 12. Quick Revision Sheet

- **`MPI_Reduce`:** All-to-One reduction. Result stored on **Root only**.
- **`MPI_Allreduce`:** All-to-All reduction. Result stored on **All ranks**.
- **Predefined Operators:** `MPI_SUM`, `MPI_PROD`, `MPI_MAX`, `MPI_MIN`, `MPI_MAXLOC`, `MPI_MINLOC`.
- **`MPI_MAXLOC` Pair Types:** Requires paired datatypes like `MPI_DOUBLE_INT` or `MPI_FLOAT_INT`.
- **Associativity Axiom:** $(a \oplus b) \oplus c = a \oplus (b \oplus c)$ is required for tree parallelism.
- **Floating-Point Non-Associativity:** IEEE 754 rounding differences mean parallel sum $\neq$ sequential sum.
- **Complexity:** Scales logarithmically as $\mathcal{O}(\log P)$ via binomial and butterfly trees.
- **In-Place:** Pass `sendbuf = MPI_IN_PLACE` to reduce directly into `recvbuf`.

---

## 13. What You Should Be Able to Do

1. **Implement parallel global reductions** using `MPI_Reduce` and `MPI_Allreduce` in compilable C.
2. **Utilize `MPI_MAXLOC` and `MPI_MINLOC`** to discover global extrema and identifying ranks.
3. **Defend why floating-point sums differ across core counts** in technical vivas and exams.
4. **Construct user-defined reduction operators** using `MPI_Op_create()` and callback functions.
5. **Calculate scaling acceleration** of tree reductions over naive linear loops using the Hockney model.
