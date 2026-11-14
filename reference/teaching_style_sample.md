The goal will be:
1. Understand the concept first
2. Learn the exact terminology/formulas
3. Know how to explain it in an exam
4. Practice with questions + answers
5. Include the important diagrams/code ideas from the notes
6. Clearly mark must-remember points and common traps.
The first portion of the notes is mainly building the foundation of parallel computing → shared memory → OpenMP → performance → speedup → efficiency → Amdahl's Law → scalability. The notes introduce these concepts across roughly pages 1–17.     3d773f8d-2e9e-426e-8ef8-0f93e44…
PART 1 — FOUNDATIONS OF PARALLEL COMPUTING
From basic parallelism to Amdahl's Law
1. What is Parallel Computing?
Basic idea
In a traditional sequential program, instructions execute one after another:
\[
I_1 \rightarrow I_2 \rightarrow I_3 \rightarrow I_4 \rightarrow \cdots
\]
In parallel computing, multiple parts of a problem are executed simultaneously using multiple processing elements/cores.

For example, suppose we need to add 1 million numbers.

Instead of:
Core 1:
1 → 2 → 3 → ... → 1,000,000

we can divide the work:
Core 1 → elements 1 to 250,000
Core 2 → elements 250,001 to 500,000
Core 3 → elements 500,001 to 750,000
Core 4 → elements 750,001 to 1,000,000

Then the results can be combined.
The fundamental idea
Divide a large problem into smaller independent pieces and execute those pieces concurrently.

2. Why Do We Need Parallel Computing?
There is an important motivation behind parallel computing.

Modern processors cannot simply keep increasing the clock frequency indefinitely because of problems such as:
- power consumption
- heat generation
- physical limitations
- energy efficiency
Therefore, instead of making one core infinitely fast, processors increasingly use multiple cores.

For example:
Single-core processor

       CPU
        |
        ↓
      Program
        |
     Sequential

versus:
Multi-core processor

       CPU
 ┌─────┼─────┐
 ↓     ↓     ↓
C1    C2    C3    C4
 │     │     │     │
 └─────┴─────┴─────┘
        Program

The challenge is:
How do we divide the program so that different cores can work simultaneously?

That is the central problem of parallel programming.
3. What Kind of Problems Can Be Parallelized?
Not every program can be parallelized equally well.

Consider:
a = b + c;
d = a + e;
f = d + g;

Here:
a must be calculated
       ↓
d depends on a
       ↓
f depends on d

So these operations have a dependency chain.

They cannot simply be executed simultaneously.

But consider:
a = b + c;
d = e + f;
g = h + i;

These operations are independent.

Therefore:
Thread 1 → a = b+c
Thread 2 → d = e+f
Thread 3 → g = h+i

can execute in parallel.
Key concept: Dependency
Parallelism is easier when operations are independent.

So whenever you see a parallel algorithm, ask:
Does one computation depend on the result of another computation?

If yes → parallelization may be difficult.

If no → good opportunity for parallelism.
4. Shared-Memory Parallel Computing
A major concept in the first part of your notes is shared memory. The notes distinguish shared memory from distributed memory and discuss cache coherence.     3d773f8d-2e9e-426e-8ef8-0f93e44…

Imagine a system with four processors:
             Shared Memory
        ┌─────────────────────┐
        │                     │
        │   Common Address    │
        │       Space         │
        │                     │
        └─────────────────────┘
          ↑     ↑     ↑     ↑
          │     │     │     │
         CPU1  CPU2  CPU3  CPU4

All processors can access the same memory address space.

This is called a:
Shared-Memory System
Definition
A shared-memory parallel system is a system in which multiple processors/cores access a common memory address space.

5. Shared Memory vs Distributed Memory
This distinction is important for examinations.
Shared Memory
All processors have access to a common memory.
CPU1 ──┐
CPU2 ──┤
CPU3 ──┼── Shared Memory
CPU4 ──┘

Advantages:
- easier communication between processors
- common address space
- programming can be relatively simpler
Disadvantages:
- memory contention
- synchronization issues
- cache coherence problems
- scalability can become difficult
Distributed Memory
Each processor/node has its own local memory.
CPU1 ─ Memory1

CPU2 ─ Memory2

CPU3 ─ Memory3

CPU4 ─ Memory4

Processors communicate by sending messages.
CPU1 ↔ CPU2
 ↑          ↑
 ↓          ↓
Memory1    Memory2

Exam comparison
Shared Memory	Distributed Memory
Common address space	Separate address spaces
Communication through shared variables/memory	Communication through messages
Easier programming model	More explicit communication
Synchronization is important	Communication cost is important
Example: OpenMP	Example: MPI


Remember:
OpenMP → primarily shared-memory parallel programming.

6. Why Do We Need Cache?
This is a very important understanding point.

CPU is extremely fast compared with main memory.

If every memory access had to go to RAM, the CPU would frequently sit idle waiting for data.

Therefore processors use caches.

Typical hierarchy:
CPU
 ↓
Registers
 ↓
L1 Cache
 ↓
L2 Cache
 ↓
L3 Cache
 ↓
Main Memory (RAM)

The closer the memory is to the CPU:
- faster
- smaller
- more expensive per byte
7. The Cache Coherence Problem
This is one of the most important concepts in shared-memory systems.

Suppose:
CPU1 has cache containing:

x = 10

CPU2 also has a cached copy:
x = 10

Now CPU1 executes:
x = 20;

CPU1's cache becomes:
x = 20

But CPU2 may still have:
x = 10

Now we have inconsistent copies.
CPU1 cache → x = 20

CPU2 cache → x = 10

Shared memory → ?

This is the cache coherence problem.
Definition
Cache coherence is the mechanism that ensures that multiple cached copies of shared data remain consistent.

Your notes specifically emphasize that when one cache changes a shared value, the change must be communicated appropriately to other processors/caches.     3d773f8d-2e9e-426e-8ef8-0f93e44…
8. Why is Cache Coherence Important?
Suppose two threads execute:
x++;

If both threads have inconsistent copies of x, they may work with stale values.

Therefore shared-memory parallel programs need mechanisms to maintain consistency.

This leads naturally to another important concept:
Synchronization
9. Synchronization
When multiple threads operate on shared data, we sometimes need to control when they access that data.

For example:
Thread 1:
x = x + 1;

Thread 2:
x = x + 1;

If both access x simultaneously, the result may be incorrect.

This is called a race condition.
Race condition
A race condition occurs when the correctness of a program depends on the timing/order in which multiple threads access shared data.

Example:

Initial:
\[
x=0
\]
Both threads read:
\[
x=0
\]
Then both calculate:
\[
x=1
\]
Final result:
\[
x=1
\]
But mathematically we expected:
\[
x=2
\]
This is why synchronization mechanisms are required.
10. Threads
A thread is an execution unit within a process.

A parallel program can create multiple threads:
                 Program
                    |
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Thread 1  Thread 2  Thread 3

Each thread can perform part of the computation.
Important distinction
Process
- independent execution environment
- generally has its own address space
Thread
- execution unit within a process
- threads of the same process can share memory
This shared memory is what makes threads useful for shared-memory parallel programming.
11. OpenMP
Now the notes move toward OpenMP.     3d773f8d-2e9e-426e-8ef8-0f93e44…
What is OpenMP?
Precise examination definition
OpenMP is an API/specification for shared-memory parallel programming in C, C++ and Fortran using compiler directives, runtime library routines and environment variables.

The important phrase is:
Shared-memory parallel programming
OpenMP allows us to tell the compiler:
"This portion of the program can be executed using multiple threads."

12. Basic OpenMP Idea
Consider a normal loop:
for(int i = 0; i < n; i++)
    a[i] = b[i] + c[i];

Normally:
Thread
  ↓
i=0
i=1
i=2
i=3
...

With OpenMP:
#pragma omp parallel for
for(int i = 0; i < n; i++)
    a[i] = b[i] + c[i];

Now iterations can be distributed among threads.

For example:
Thread 0 → i = 0,1,2,3
Thread 1 → i = 4,5,6,7
Thread 2 → i = 8,9,10,11
Thread 3 → i = 12,13,14,15

This is one of the most fundamental ideas you should understand.
13. OpenMP Programming Model
The notes introduce the concept of fork-join parallelism.     3d773f8d-2e9e-426e-8ef8-0f93e44…

Imagine initially:
             Master Thread
                  |
                  |
             Serial Code
                  |
               FORK
            /    |    \
           /     |     \
       Thread  Thread  Thread
          1       2       3
           \      |      /
            \     |     /
                JOIN
                  |
             Serial Code

This is called the:
Fork-Join Model
Fork
The master thread creates multiple worker threads.
Parallel region
Threads execute simultaneously.
Join
Threads synchronize and come back together.
14. OpenMP Example
#pragma omp parallel
{
    // code executed by multiple threads
}

Suppose there are 4 threads.

Then conceptually:
              parallel
                  |
       ┌──────────┼──────────┐
       ↓          ↓          ↓
      T0         T1         T2        T3
       ↓          ↓          ↓         ↓
      code       code       code      code
       └──────────┼──────────┘
                  ↓
                 join

Important
A parallel region means:
Multiple threads execute the enclosed block.

But it does not automatically mean that every loop iteration is distributed intelligently.

For loops, we commonly use:
#pragma omp parallel for

15. Why Parallelization Does NOT Give Infinite Speedup
This is where your notes transition into performance analysis. Pages 10 onward discuss speedup, efficiency, sequential portions and Amdahl's Law.     3d773f8d-2e9e-426e-8ef8-0f93e44…

Suppose:
Total program:

Sequential part = 20%
Parallel part   = 80%

Even if we use 1000 processors, the sequential part still has to execute sequentially.

So:
Sequential portion
       ↓
cannot benefit from more processors

This places a fundamental limit on speedup.
16. Speedup
This is a must-know formula.

Let:
- \(T_1\) = execution time using 1 processor
- \(T_p\) = execution time using \(p\) processors
Then:
\[
\boxed{S(p)=\frac{T_1}{T_p}}
\]
where \(S(p)\) is the speedup obtained using \(p\) processors.
Example
Suppose:
\[
T_1=100s
\]
and with 4 processors:
\[
T_4=30s
\]
Then:
\[
S(4)=\frac{100}{30}=3.33
\]
So the program is approximately 3.33 times faster.
17. Ideal Speedup
If we have \(p\) processors, the best possible speedup is theoretically:
\[
\boxed{S(p)=p}
\]
For example:
Processors	Ideal Speedup
1	1
2	2
4	4
8	8
16	16


This is called linear speedup.

But in real systems:
\[
S(p)<p
\]
because of:
- sequential portions
- communication
- synchronization
- load imbalance
- memory bottlenecks
- thread management overhead
18. Parallel Efficiency
Speedup alone isn't enough.

Suppose:
\[
S(8)=6
\]
Is that good?

We compare it with the ideal speedup of 8.
Efficiency
\[
\boxed{E(p)=\frac{S(p)}{p}}
\]
Therefore:
\[
E(8)=\frac{6}{8}=0.75
\]
or:
\[
75\%
\]
Interpretation
Efficiency tells us:
How effectively the available processors are being utilized compared with ideal parallel execution.

19. Speedup vs Efficiency
Very important distinction.
Speedup
Answers:
How many times faster did the program become?

\[
S(p)=\frac{T_1}{T_p}
\]
Efficiency
Answers:
How effectively are the processors being utilized?

\[
E(p)=\frac{S(p)}p
\]
20. Amdahl's Law
This is probably one of the highest-priority topics from this portion of your notes.

The notes introduce the idea that a program contains both:
- sequential work
- parallelizable work
and the sequential portion limits the achievable speedup.     3d773f8d-2e9e-426e-8ef8-0f93e44…

Let:
\[
s = \text{fraction of program that is sequential}
\]
Then:
\[
1-s = \text{parallelizable fraction}
\]
With \(p\) processors:
- sequential part takes \(sT\)
- parallel part ideally takes:
\[
\frac{(1-s)T}{p}
\]
Therefore:
\[
T_p=sT+\frac{(1-s)T}{p}
\]
Since:
\[
T_1=T
\]
we get:
\[
\boxed{
S(p)=
\frac{1}
{s+\frac{1-s}{p}}
}
\]
This is:
Amdahl's Law
21. Understanding Amdahl's Law Intuitively
Suppose 10% of your program is sequential.

Therefore:
\[
s=0.1
\]
and:
\[
1-s=0.9
\]
With infinitely many processors:
\[
p\rightarrow\infty
\]
Then:
\[
\frac{1-s}{p}\rightarrow0
\]
Therefore:
\[
S_{\max}
=
\frac{1}{s}
\]
So:
\[
S_{\max}=\frac{1}{0.1}=10
\]
Meaning
Even with infinitely many processors:
\[
\boxed{S_{\max}=10}
\]
You can never exceed 10× speedup if 10% of the program must remain sequential.

This is the key insight.
22. Amdahl's Law Example
Suppose a program has:
\[
s=0.2
\]
So:
- 20% sequential
- 80% parallelizable
Using 4 processors:
\[
S(4)
=
\frac{1}
{0.2+\frac{0.8}{4}}
\]
\[
=
\frac{1}
{0.2+0.2}
\]
\[
=\frac{1}{0.4}
\]
\[
\boxed{S(4)=2.5}
\]
Notice something interesting.

We used 4 processors, but only got 2.5× speedup.

Ideal speedup was 4.

This is because 20% of the program cannot be parallelized.
23. Maximum Speedup
A very important exam result:

As:
\[
p\rightarrow\infty
\]
Amdahl's Law becomes:
\[
S_{\max}=\frac{1}{s}
\]
Therefore:
\[
\boxed{S_{\max}=\frac{1}{\text{sequential fraction}}}
\]
Quick table
Sequential fraction	Maximum speedup
50%	2×
25%	4×
10%	10×
5%	20×
1%	100×


Exam insight
To obtain large speedups, the sequential fraction must be extremely small.

24. Why Adding More Processors Eventually Stops Helping
Imagine:
\[
s=0.1
\]
Then:
\[
S(p)=\frac{1}{0.1+\frac{0.9}{p}}
\]
For increasing \(p\):
Processors       Speedup
1                1
2                ~1.82
4                ~3.08
8                ~4.71
16               ~6.40
32               ~7.80
64               ~8.77
∞                10

The speedup curve starts increasing rapidly and then flattens.

Why?

Because the sequential part becomes the dominant limitation.

This is the bottleneck concept.
25. Bottleneck
A bottleneck is a part of a system that limits the overall performance.

Think of a highway:
Wide road → Wide road → Narrow road → Wide road
                           ↑
                       Bottleneck

Even if every other section is extremely fast, traffic is limited by the narrow section.

Similarly:
Parallel portion → Parallel portion → Sequential portion
                                           ↑
                                       Bottleneck

The sequential portion limits overall speedup.
26. Critical Path Idea
Another useful way to think about performance:
The total execution time is constrained by work that cannot be overlapped.

If operations have dependencies:
A
↓
B
↓
C
↓
D

they form a dependency chain.

Even if we have many processors, this chain cannot be arbitrarily shortened.

Therefore, good parallel algorithms try to:
- expose more independent work
- reduce dependencies
- reduce sequential sections
27. Scalability
The notes then discuss scalability and ask how performance changes when the number of processors increases.     3d773f8d-2e9e-426e-8ef8-0f93e44…
Definition
Scalability is the ability of a parallel program/system to effectively utilize increasing computational resources as the number of processors increases.

A scalable program should continue to obtain useful performance improvements as processors are added.
28. Strong Scaling vs Weak Scaling
These concepts are extremely important for exams.
Strong Scaling
Problem size stays fixed.

We increase the number of processors.
Same problem
     ↓
1 CPU → 2 CPU → 4 CPU → 8 CPU

Question:
How much faster can we solve the same problem?

Weak Scaling
The amount of work per processor stays approximately constant.

As processors increase, problem size also increases.
1 CPU  → small problem
2 CPU  → 2× problem
4 CPU  → 4× problem
8 CPU  → 8× problem

Question:
Can we solve a proportionally larger problem in approximately the same amount of time?

29. Strong Scaling Example
Suppose:
\[
T_1=100s
\]
Then:
Processors	Time
1	100 s
2	50 s
4	25 s
8	12.5 s


This is ideal strong scaling.

Speedup:
\[
S(8)=\frac{100}{12.5}=8
\]
Efficiency:
\[
E(8)=\frac88=1=100\%
\]
Real systems generally perform worse.
30. Why Real Parallel Programs Don't Scale Perfectly
Even if an algorithm is theoretically parallelizable, practical issues appear.
1. Sequential work
Some work cannot be parallelized.
2. Synchronization
Threads may need to wait for one another.
3. Communication
Data may need to move between processors.
4. Load imbalance
Some threads get more work than others.

Example:
Thread 1 → ██████████
Thread 2 → █████
Thread 3 → ███
Thread 4 → ███████

Thread 3 finishes early and waits.
5. Parallelization overhead
Creating/managing threads itself takes time.
6. Memory bottleneck
Multiple processors may compete for memory bandwidth.
31. A Very Important Concept: Overhead
Suppose a sequential program takes:
\[
100s
\]
You parallelize it using 4 threads.

You might expect:
\[
25s
\]
But actual time could be:
\[
30s
\]
because of overhead.

So:
\[
T_p =
\text{useful computation}
+
\text{parallel overhead}
\]
Overhead may include:
- creating threads
- scheduling
- synchronization
- communication
- waiting
- combining results
32. Putting Everything Together
At this point, you should be able to visualize the entire story:
                  Parallel Computing
                         |
              ┌──────────┴──────────┐
              ↓                     ↓
       Shared Memory          Distributed Memory
              |
           Threads
              |
           OpenMP
              |
        Fork-Join Model
              |
        Parallel Region
              |
       Performance Analysis
              |
      ┌───────┼────────┐
      ↓       ↓        ↓
   Speedup Efficiency Scalability
      |
   Amdahl's Law
      |
 Sequential fraction
      ↓
 Limits maximum speedup

This chain is very important for understanding rather than memorizing isolated definitions.
33. EXAM-READY DEFINITIONS
These are worth memorizing almost word-for-word.
Q1. What is parallel computing?
Answer:
Parallel computing is a computing approach in which a problem is divided into multiple parts that can be executed simultaneously using multiple processors or cores.

Q2. What is shared-memory computing?
Answer:
Shared-memory computing is a parallel computing model in which multiple processors or threads access a common memory address space.

Q3. What is OpenMP?
Answer:
OpenMP is an API for shared-memory parallel programming that provides compiler directives, runtime library routines and environment variables for creating and managing parallel programs.

Q4. What is a thread?
Answer:
A thread is a lightweight unit of execution within a process. Threads belonging to the same process can share its memory.

Q5. What is cache coherence?
Answer:
Cache coherence is the mechanism that maintains consistency among multiple cached copies of shared data in a multiprocessor system.

Q6. What is speedup?
\[
\boxed{S(p)=\frac{T_1}{T_p}}
\]
Answer:
Speedup is the ratio of the execution time of a program using one processor to its execution time using \(p\) processors.

Q7. What is efficiency?
\[
\boxed{E(p)=\frac{S(p)}p}
\]
Answer:
Parallel efficiency measures how effectively the available processors are utilized compared with ideal linear speedup.

Q8. State Amdahl's Law.
\[
\boxed{
S(p)=
\frac{1}
{s+\frac{1-s}{p}}
}
\]
where \(s\) is the sequential fraction.
Q9. What is the maximum speedup according to Amdahl's Law?
\[
\boxed{S_{\max}=\frac1s}
\]
Answer:
The maximum speedup is limited by the sequential fraction of the program. Even with infinitely many processors, the maximum speedup is \(1/s\).

Q10. What is scalability?
Answer:
Scalability is the ability of a parallel system or program to effectively utilize additional processors and obtain useful performance improvement as computational resources increase.

34. NUMERICAL QUESTIONS YOU SHOULD PRACTICE
Question 1
A program takes 100 seconds on one processor and 25 seconds on 4 processors. Find speedup and efficiency.
Solution
\[
S(4)=\frac{100}{25}=4
\]
Efficiency:
\[
E(4)=\frac{4}{4}=1
\]
Therefore:
\[
\boxed{S=4,\quad E=100\%}
\]
This represents ideal linear speedup.
Question 2
A program has 20% sequential execution. Find maximum possible speedup.
Solution
\[
s=0.2
\]
\[
S_{\max}=\frac1s
\]
\[
=\frac1{0.2}
\]
\[
\boxed{S_{\max}=5}
\]
So no matter how many processors are used, speedup cannot exceed 5.
Question 3
A program has 10% sequential work. Find speedup using 5 processors.
Solution
\[
s=0.1
\]
\[
p=5
\]
Using Amdahl's Law:
\[
S(5)=
\frac1{0.1+\frac{0.9}{5}}
\]
\[
=\frac1{0.1+0.18}
\]
\[
=\frac1{0.28}
\]
\[
\boxed{S(5)\approx3.57}
\]
Although we use 5 processors, we get only about 3.57× speedup.
35. CONCEPTUAL EXAM QUESTIONS
Q1. Why can't adding more processors provide unlimited speedup?
Answer:

Because a program generally contains a sequential portion that cannot be executed in parallel. According to Amdahl's Law, as the number of processors approaches infinity, the parallel portion approaches zero execution time, but the sequential portion remains. Therefore:
\[
S_{\max}=\frac1s
\]
Q2. Why is speedup generally less than the number of processors?
Answer:

Because of sequential execution, synchronization, communication, load imbalance, memory contention and parallelization overhead.
Q3. Why is cache coherence required in shared-memory systems?
Answer:

Because multiple processors may have cached copies of the same shared variable. If one processor modifies the variable, other processors may hold stale copies. Cache coherence mechanisms ensure that these copies remain consistent.
Q4. What is the difference between strong and weak scaling?
Answer:

Strong scaling: Problem size remains fixed while the number of processors increases.

Weak scaling: Problem size increases proportionally with the number of processors, keeping the workload per processor approximately constant.
36. QUICK REVISION SHEET
Before moving to the next portion, make sure you can reproduce this without looking:
Parallel computing
\[
\boxed{\text{Divide work + execute simultaneously}}
\]
Shared memory
\[
\boxed{\text{Multiple processors + common address space}}
\]
Thread
\[
\boxed{\text{Unit of execution}}
\]
OpenMP
\[
\boxed{\text{Shared-memory parallel programming API}}
\]
Fork-Join
\[
\boxed{\text{Fork → Parallel execution → Join}}
\]
Speedup
\[
\boxed{S(p)=\frac{T_1}{T_p}}
\]
Efficiency
\[
\boxed{E(p)=\frac{S(p)}p}
\]
Amdahl
\[
\boxed{
S(p)=\frac1{s+\frac{1-s}{p}}
}
\]
Maximum Amdahl speedup
\[
\boxed{S_{\max}=\frac1s}
\]
Strong scaling
\[
\boxed{\text{Fixed problem size}}
\]
Weak scaling
\[
\boxed{\text{Fixed work/processor}}
\]
What you should be able to do after Part 1
Before proceeding, you should be able to answer these without memorizing a paragraph:
1. Why do we need parallel computing?
2. What makes a problem suitable for parallelization?
3. What is shared-memory computing?
4. Why do cache coherence and synchronization matter?
5. What is a thread?
6. What exactly does OpenMP provide?
7. Explain the fork-join model with a diagram.
8. Calculate speedup.
9. Calculate efficiency.
10. Derive/use Amdahl's Law.
11. Explain why speedup saturates.
12. Find maximum possible speedup.
13. Explain strong vs weak scaling.
14. List practical reasons for non-ideal speedup.
The next portion should naturally continue from here into the much more implementation-oriented material in the notes: performance measurement, bottlenecks, OpenMP directives, parallel, for, scheduling, work sharing, synchronization, critical, barrier, reduction, and the associated C/C++ examples. The later pages contain a substantial amount of code and questions around these topics.     3d773f8d-2e9e-426e-8ef8-0f93e44…






        






continue





 






Worked for 1m 34s






PART 2 — OPENMP: THREADS, DATA SCOPE, SCHEDULING & SYNCHRONIZATION
This part continues directly from the previous section. The notes now move from “why parallel computing?” to “how do we actually write and control a parallel program using OpenMP?”

The main topics covered here are:
1. OpenMP software architecture
2. Threads and data scope
3. parallel and for
4. Number of threads
5. private variables
6. Loop scheduling
7. Static scheduling
8. Dynamic scheduling
9. Guided scheduling
10. Auto scheduling
11. Synchronization
12. critical
13. atomic
14. barrier
These topics appear mainly across pages 20–30 of your notes.     3d773f8d-2e9e-426e-8ef8-0f93e44…
1. OpenMP — The Big Picture
Before learning individual directives, understand what OpenMP actually does.

Your notes show an OpenMP software architecture with three major pieces:
                 APPLICATION
                      |
             OpenMP directives
                      |
              Runtime Library
                      |
             Threads / Cores

There are essentially two views of OpenMP:
Programmer's view
The programmer specifies:
- compiler directives/pragmas
- runtime library functions
- environment variables
System/runtime view
The OpenMP runtime handles things such as:
- thread creation
- thread management
- scheduling
- resource allocation
This distinction is useful in an examination because it explains why OpenMP is more than just a compiler keyword.
2. OpenMP Is Not a Programming Language
This is specifically emphasized in the notes.

OpenMP is not a separate programming language.

Instead, it provides extensions to languages such as C/C++ through:
1. Compiler directives
For example:
#pragma omp parallel

2. Runtime library functions
For example:
omp_get_thread_num()
omp_get_num_threads()

3. Environment variables
For example, the number of threads can be controlled through the OpenMP environment.
Exam answer
OpenMP is an API/specification for shared-memory parallel programming. It extends C/C++/Fortran through compiler directives, runtime library routines and environment variables.

3. The Most Important OpenMP Mental Model
Suppose you have:
int main() {

    // serial code

    #pragma omp parallel
    {
        // parallel code
    }

    // serial code
}

Think of it as:
                    Master thread
                         |
                    Serial code
                         |
                       FORK
              ┌──────────┼──────────┐
              ↓          ↓          ↓
             T0         T1         T2 ...
              |          |          |
              └──────────┼──────────┘
                         |
                        JOIN
                         |
                    Serial code

This is the fork-join execution model.
4. Number of Threads
One of the questions in the notes is essentially:
How do we control the number of threads?

OpenMP can create multiple threads for a parallel region.

You can explicitly specify the number:
#pragma omp parallel num_threads(4)
{
    // parallel work
}

Conceptually:
          Parallel Region
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
      T0       T1       T2       T3

If the number isn't explicitly specified, the OpenMP runtime determines the number according to its configuration.
Important distinction
Don't confuse:
num_threads(4)

with:
omp_get_num_threads()

The first requests/specifies the number of threads for a parallel region.

The second returns the number of threads currently participating in the parallel region.
5. omp_get_thread_num()
This function tells a thread its ID.
int id = omp_get_thread_num();

Thread IDs typically range from:
\[
0,1,2,\ldots,p-1
\]
where \(p\) is the number of threads.

For example, with 4 threads:
Thread ID
   0
   1
   2
   3

So:
printf("Thread %d\n", omp_get_thread_num());

may produce:
Thread 0
Thread 1
Thread 2
Thread 3

Important
The order in which these lines appear is not guaranteed.

For example:
Thread 2
Thread 0
Thread 3
Thread 1

is perfectly possible.

Why?

Because threads execute concurrently and the operating system/runtime scheduler determines when each thread gets CPU time.
6. Why Does Output Come in Different Orders?
This is an important conceptual question.

Suppose:
#pragma omp parallel
{
    printf("Hello from thread %d\n",
           omp_get_thread_num());
}

You might expect:
Thread 0
Thread 1
Thread 2
Thread 3

But you could get:
Thread 2
Thread 0
Thread 3
Thread 1

Why?
Because parallel execution does not guarantee execution order.

Each thread can reach printf() at a different time.

Therefore:
Concurrent execution does not imply ordered execution.

This becomes especially important when writing exam answers involving OpenMP output.
7. Data Scoping in OpenMP
This is a very important topic.

When multiple threads execute a parallel region, we must determine:
Which variables are shared and which variables are private?

The notes explicitly introduce this as data scoping of threads — shared vs private data.     3d773f8d-2e9e-426e-8ef8-0f93e44…
8. Shared Variables
A shared variable is accessible by multiple threads.

Conceptually:
             Shared variable x
                    |
        ┌───────────┼───────────┐
        ↓           ↓           ↓
       T0          T1          T2

All threads can access the same variable.

If one thread modifies it, other threads may observe the modified value, subject to synchronization and memory-consistency rules.
Example
int x = 10;

#pragma omp parallel
{
    printf("%d\n", x);
}

Here x is shared by default in the typical case because it existed before the parallel region.
9. Private Variables
A private variable gives each thread its own separate instance.

Think:
                x
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
     T0:x      T1:x      T2:x
      10        20        30

These are logically separate variables.

Changing T0's x does not directly change T1's private x.
10. Why Do We Need Private Variables?
Consider:
int i;

#pragma omp parallel
{
    i = omp_get_thread_num();
}

If i is shared, multiple threads write to the same memory location.

That creates a potential race.

Instead:
#pragma omp parallel private(i)
{
    i = omp_get_thread_num();
}

Now:
T0 → private i
T1 → private i
T2 → private i
T3 → private i

Each thread has its own copy.
11. Local Variables Inside a Parallel Block
Suppose:
#pragma omp parallel
{
    int x;
    x = omp_get_thread_num();
}

x is declared inside the parallel block, so each thread gets its own local instance.

Conceptually:
T0 → x0
T1 → x1
T2 → x2
T3 → x3

This is different from a variable declared outside the parallel region.
12. Stack vs Shared Memory
Your notes also mention the stack.

Each thread has its own execution context and stack-related local storage.

Think:
Process
 |
 ├── Thread 0 → Stack 0
 ├── Thread 1 → Stack 1
 ├── Thread 2 → Stack 2
 └── Thread 3 → Stack 3

But dynamically allocated/shared objects can be accessible by multiple threads depending on how the program handles their addresses.

The key examination distinction is:
Private data belongs to an individual thread, whereas shared data can be accessed by multiple threads.

13. parallel vs parallel for
This distinction is very important.
parallel
Creates a parallel region.
#pragma omp parallel
{
    // executed by all threads
}

Every thread executes the block.
parallel for
Creates a parallel region and distributes loop iterations among the threads.
#pragma omp parallel for
for(int i = 0; i < n; i++) {
    // work
}

Instead of every thread executing all iterations, iterations are divided.
14. Why Not Just Put for Inside parallel?
Suppose:
#pragma omp parallel
{
    for(int i = 0; i < n; i++) {
        work(i);
    }
}

If there are 4 threads, each thread can execute the entire loop.

So you could accidentally get:
T0 → 0 1 2 3 ... n
T1 → 0 1 2 3 ... n
T2 → 0 1 2 3 ... n
T3 → 0 1 2 3 ... n

The work is duplicated.

Instead:
#pragma omp parallel for
for(int i = 0; i < n; i++) {
    work(i);
}

gives:
T0 → 0 1 2 ...
T1 → ...
T2 → ...
T3 → ...

with iterations distributed among threads.
Therefore:
parallel parallelizes a block; parallel for parallelizes the iterations of a loop.

15. Loop Scheduling
Now we reach one of the most important implementation topics in this section.

Suppose:
#pragma omp parallel for
for(int i = 0; i < 100; i++) {
    work(i);
}

There are 100 iterations and perhaps 4 threads.

The question becomes:
How should these 100 iterations be assigned to the 4 threads?

This is the job of scheduling.

Your notes identify four scheduling types:
1. Static
2. Dynamic
3. Guided
4. Auto     3d773f8d-2e9e-426e-8ef8-0f93e44…
16. Static Scheduling
Basic idea
The iterations are divided into chunks before/during the setup of execution, and the assignment is predetermined.

Example:
#pragma omp parallel for schedule(static)
for(int i=0; i<8; i++)
    work(i);

With 4 threads, conceptually:
T0 → 0,1
T1 → 2,3
T2 → 4,5
T3 → 6,7

assuming the default equal division.
17. Static Scheduling With Chunk Size
You can specify:
schedule(static, 2)

The 2 is the chunk size.

A chunk contains 2 loop iterations.

For:
0 1 2 3 4 5 6 7

chunks are:
[0,1]
[2,3]
[4,5]
[6,7]

These chunks are assigned to threads.
18. Why Would We Specify a Chunk Size?
Suppose:
schedule(static, 4)

Each chunk contains four iterations.

This provides more control over how iterations are grouped.

But there's a trade-off.
Larger chunks
- less scheduling overhead
- potentially poorer load balancing
Smaller chunks
- potentially better distribution
- more scheduling/assignment overhead
This trade-off is a recurring theme in parallel programming.
19. Static Scheduling Example
Suppose:
N = 8
threads = 4

and:
schedule(static)

A natural division is:
Thread 0 → 0 1
Thread 1 → 2 3
Thread 2 → 4 5
Thread 3 → 6 7

Now suppose:
schedule(static, 4)

Then:
Chunk 1 → 0 1 2 3
Chunk 2 → 4 5 6 7

Only two chunks exist, so only two threads get chunks at that stage.

The other threads may remain idle.

This is exactly the type of situation your notes are warning about.
20. The Load-Balancing Problem
Suppose every iteration does different amounts of work.

For example:
for(int i=0; i<n; i++)
    work(i);

Maybe:
iteration 0 → 1 ms
iteration 1 → 1 ms
iteration 2 → 10 ms
iteration 3 → 20 ms
...

Now static scheduling can produce:
T0 → ███
T1 → ██████████████
T2 → ████
T3 → █████████

Some threads finish much earlier.

They become idle.

This is called:
Load Imbalance
21. Dynamic Scheduling
Dynamic scheduling addresses this problem.

Syntax:
#pragma omp parallel for schedule(dynamic, 1)

The important idea is:
Threads obtain new chunks dynamically as they finish their current work.

Imagine:
              Work queue
        ┌────────────────────┐
        │ C1 C2 C3 C4 C5 C6  │
        └────────────────────┘
          ↑    ↑    ↑
         T0   T1   T2

When T0 finishes its chunk:
T0 → requests another chunk

rather than remaining idle.
22. Why Dynamic Scheduling Helps
Suppose work is uneven:
Iteration work:

0 → █
1 → █
2 → █████████
3 → █████████████
4 → ██
5 → ██████

Static scheduling may accidentally give one thread most of the expensive iterations.

Dynamic scheduling can redistribute work as threads become free.

Therefore:
Dynamic scheduling improves load balancing when iteration execution times are unpredictable or uneven.

23. Dynamic Scheduling Trade-Off
Dynamic scheduling is not always better.

Why?

Because assigning chunks dynamically requires runtime scheduling work.

Therefore:
Dynamic
   ↓
Better load balancing
   +
More scheduling overhead

Static:
Static
   ↓
Low overhead
   +
Potential load imbalance

Exam-friendly comparison
Static	Dynamic
Assignment predetermined	Assignment happens during execution
Lower scheduling overhead	Higher scheduling overhead
Good for uniform work	Good for irregular work
Can suffer from load imbalance	Better load balancing


24. Guided Scheduling
Guided scheduling is essentially a compromise between static and dynamic approaches.

Syntax:
#pragma omp parallel for schedule(guided)

Your notes describe the idea as:
Start with large chunks and gradually reduce chunk size as threads continue working.     3d773f8d-2e9e-426e-8ef8-0f93e44…

Think:
Initially:

████████████████

then:

████████

then:

████

then:

██

then:

█

So chunk sizes decrease over time.
25. Why Use Guided Scheduling?
At the beginning, there is lots of work remaining.

Large chunks mean:
- fewer scheduling operations
- lower overhead
Later, fewer iterations remain.

Smaller chunks allow:
- better load balancing
So guided scheduling tries to obtain:
\[
\boxed{\text{Low overhead + good load balancing}}
\]
It is especially useful when work is irregular and the loop has many iterations.
26. Auto Scheduling
The fourth scheduling type in your notes is:
schedule(auto)

Here, the scheduling decision is left to the compiler/runtime.

Conceptually:
Programmer
    ↓
schedule(auto)
    ↓
Compiler / Runtime
    ↓
chooses scheduling strategy

When useful?
When you don't want to manually decide the scheduling policy and want the implementation to choose.

The notes mention situations such as experimentation/prototyping and allowing compiler/runtime decisions.     3d773f8d-2e9e-426e-8ef8-0f93e44…
27. Scheduling — One Table to Memorize
Schedule	Main idea	Best suited for
static	Predetermined chunks	Uniform work
dynamic	Threads obtain chunks as they finish	Uneven work
guided	Large → progressively smaller chunks	Uneven work with reduced overhead
auto	Compiler/runtime chooses	When programmer wants automatic choice


Memory trick
S-D-G-A
Static → Dynamic → Guided → Auto

28. A Very Important Numerical/Conceptual Question
Suppose there are:
\[
8 \text{ iterations}
\]
and:
\[
4 \text{ threads}
\]
with:
schedule(static)

Expected distribution:
T0 → 0,1
T1 → 2,3
T2 → 4,5
T3 → 6,7

Now suppose:
schedule(static, 4)

Chunks:
C0 → 0,1,2,3
C1 → 4,5,6,7

So:
T0 → C0
T1 → C1
T2 → idle
T3 → idle

This illustrates why chunk size matters.
29. Measuring Performance Correctly
Your notes also ask an important question:
Why isn't one execution enough to measure performance?

Because execution time can vary due to:
- operating system activity
- CPU scheduling
- cache state
- system load
- runtime behavior
Therefore, timing a parallel program once isn't necessarily a reliable measurement.

You should generally execute it multiple times and analyze the timings.

The notes also point out that runtime may depend on the system and scheduling behavior.     3d773f8d-2e9e-426e-8ef8-0f93e44…
30. Synchronization
Now we come to another very high-priority exam topic.

Multiple threads are useful, but they can cause problems when they access shared data.

For example:
count++;

Suppose 4 threads execute this simultaneously.

The programmer might expect:
\[
count = 4
\]
but due to concurrent reads/writes, the result can be incorrect.

We therefore need:
Synchronization
Definition
Synchronization is the coordination of multiple threads so that operations on shared resources occur in a controlled and correct manner.

31. Critical Section
A critical section is a portion of code where access must be restricted because it manipulates shared data/resource.

For example:
count++;

may need protection.

OpenMP provides:
#pragma omp critical
{
    count++;
}

This means:
Only one thread at a time may execute the critical region associated with that construct.

32. Why Does critical Work?
Suppose:
T0 ───────┐
T1 ───────┤
T2 ───────┼── critical section
T3 ───────┘

Only one thread enters:
T0 → enters critical
T1 → waits
T2 → waits
T3 → waits

When T0 leaves:
T1 → enters

Then:
T2 → enters

and so on.

Thus simultaneous modification of the shared resource is prevented.
33. critical Example
Unsafe:
#pragma omp parallel
{
    count++;
}

Potential race condition.

Safer:
#pragma omp parallel
{
    #pragma omp critical
    {
        count++;
    }
}

Now only one thread updates count at a time.
34. But critical Has a Cost
This is an important performance insight.

If 100 threads all need:
#pragma omp critical
{
    count++;
}

then only one thread can perform the operation at a time.

So the code becomes partially sequential.

Therefore:
Correctness ↑
Parallelism ↓

This is why synchronization should be used only where necessary.
35. atomic
Your notes introduce another mechanism:
#pragma omp atomic
count++;

atomic protects a specific memory update.

For a simple operation such as:
count++;

atomic is often more lightweight than protecting a whole block with critical.
Basic distinction
critical:
#pragma omp critical
{
    // multiple operations
}

atomic:
#pragma omp atomic
count++;

36. critical vs atomic
This is a likely examination question.
critical	atomic
Protects a region/block	Protects a specific atomic memory operation
Can contain multiple statements	Usually one supported update/access
More general	More specialized
Often higher overhead	Often lower overhead for simple updates
Only one thread executes critical region at a time	Ensures atomicity of the specified operation


Easy way to remember
Critical = protect a block.

Atomic = protect one update.

37. When Should You Use Which?
Suppose:
count++;

Use:
#pragma omp atomic
count++;

if the operation fits the atomic construct.

But suppose you need:
update A
update B
check C
modify D

as one protected sequence.

Then:
#pragma omp critical
{
    update A;
    update B;
    check C;
    modify D;
}

may be appropriate.
38. Barrier
The third synchronization mechanism introduced in your notes is:
#pragma omp barrier

This is very important.
Definition
A barrier forces threads to wait until all threads in the team have reached the barrier before any of them proceeds beyond it.

Think of it as a checkpoint.
T0 ────────────────┐
                   │
T1 ──────────┐     │
             │     │
T2 ──────────┼─────┤ ← barrier
             │     │
T3 ──────────┘     │
                   ↓
              Continue

39. Why Do We Need a Barrier?
Suppose we calculate:
\[
c_i=a_i b_i
\]
in parallel.

Then we want to use all the calculated values.

Without synchronization:
T0 → calculation finished
T1 → calculation finished
T2 → still calculating
T3 → starts using results

T3 might access incomplete data.

With a barrier:
All threads calculate
        ↓
     BARRIER
        ↓
All threads know previous stage is complete
        ↓
Continue

40. Example of Barrier
#pragma omp parallel
{
    int id = omp_get_thread_num();

    // Stage 1
    compute(id);

    #pragma omp barrier

    // Stage 2
    use_results();
}

Execution:
Stage 1
   ↓
Thread 0 ──┐
Thread 1 ──┤
Thread 2 ──┼── BARRIER
Thread 3 ──┘
            ↓
         Stage 2

No thread proceeds into Stage 2 until all participating threads reach the barrier.
41. Barrier vs Critical
Very common confusion.
critical
Controls:
Who can enter a particular section at one time?

barrier
Controls:
When can threads proceed to the next stage?

So:
critical → mutual exclusion
barrier  → synchronization point

This distinction is worth memorizing.
42. The Three Synchronization Mechanisms
Your notes now have a very useful progression:
                 Synchronization
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       critical      atomic       barrier
          |            |            |
     protect block  protect      wait for
                    update       everyone

critical
One thread at a time in a protected block.
atomic
Protect a specific shared-memory update.
barrier
Everyone waits until everyone reaches the point.
43. Understanding the Example in Your Notes
Your notes use an example involving two arrays and a computation such as:
\[
c[i]=a[i]\times b[i]
\]
Suppose there are 4 threads.

The iterations might be distributed:
T0 → c[0], c[1]
T1 → c[2], c[3]
T2 → c[4], c[5]
T3 → c[6], c[7]

After that computation, perhaps another stage needs all c[i] values.

Then:
#pragma omp barrier

ensures:
All c[i] calculations completed
             ↓
          barrier
             ↓
Next computation

This is the key reasoning behind barriers.
44. VERY IMPORTANT: Don't Confuse Parallelism With Synchronization
Parallelism says:
"Do these things simultaneously."

Synchronization says:
"Coordinate these simultaneous activities when necessary."

You need both in real parallel programs.

Example:
Parallel computation
      ↓
Threads work independently
      ↓
Need shared result
      ↓
Synchronization
      ↓
Next stage

45. Exam Questions — OpenMP Basics
Q1. What is the difference between parallel and parallel for?
Answer:

parallel creates a parallel region in which all threads execute the enclosed block. parallel for additionally distributes the iterations of the associated loop among the threads.
Q2. What does omp_get_thread_num() return?
Answer:

It returns the unique ID of the calling thread within the current OpenMP team.
Q3. Why can OpenMP output appear in random order?
Answer:

Because threads execute concurrently and OpenMP does not guarantee the order in which threads execute statements such as printf().
Q4. What is the difference between shared and private variables?
Answer:

A shared variable has one instance accessible by multiple threads, whereas a private variable has a separate instance for each thread.
46. Exam Questions — Scheduling
Q5. What is loop scheduling?
Answer:
Loop scheduling determines how the iterations of a parallel loop are divided into chunks and assigned to threads.

Q6. Explain static scheduling.
Answer:
In static scheduling, loop iterations are divided into chunks and assigned to threads in a predetermined manner, resulting in low scheduling overhead. It is suitable when iteration workloads are relatively uniform.

Q7. Explain dynamic scheduling.
Answer:
In dynamic scheduling, threads obtain new chunks of loop iterations as they finish their current work. It improves load balancing for irregular workloads but introduces additional scheduling overhead.

Q8. Explain guided scheduling.
Answer:
Guided scheduling assigns relatively large chunks initially and gradually reduces the chunk size as execution progresses. It attempts to balance scheduling overhead and load balancing.

Q9. What is schedule(auto)?
Answer:
schedule(auto) allows the compiler/runtime to determine the scheduling strategy.

47. Exam Questions — Synchronization
Q10. What is a critical section?
Answer:
A critical section is a section of code that accesses a shared resource and must not be executed by multiple threads simultaneously.

Q11. What does #pragma omp critical do?
Answer:
It ensures that only one thread at a time executes the associated critical region.

Q12. What does #pragma omp atomic do?
Answer:
It ensures that a supported individual memory update is performed atomically, preventing interference from simultaneous threads.

Q13. What does #pragma omp barrier do?
Answer:
It forces each thread to wait until all threads in the team have reached the barrier before any thread continues beyond it.

48. Important Comparison Question
critical vs atomic vs barrier
Feature	Critical	Atomic	Barrier
Purpose	Mutual exclusion	Atomic memory update	Synchronize threads
Protects	Code block	Specific operation	Execution point
One thread at a time?	Yes in region	Operation is atomic	Not exactly
Threads wait?	Threads attempting region may wait	Threads may contend on update	All wait
Typical use	Complex shared operation	count++	Phase synchronization


49. Numerical/Code-Based Question
Consider:
int count = 0;

#pragma omp parallel for
for(int i = 0; i < 1000; i++) {
    count++;
}

Question
Is this guaranteed to produce:
\[
count=1000?
\]
Answer
No.

count++ involves a read-modify-write sequence. Multiple threads may access it concurrently, causing a race condition.

A possible solution:
#pragma omp parallel for
for(int i = 0; i < 1000; i++) {
    #pragma omp atomic
    count++;
}

50. Another Important Question
Consider:
#pragma omp parallel
{
    printf("Hello\n");

    #pragma omp barrier

    printf("World\n");
}

Question
Can World be printed before every Hello?
Answer
No.

The barrier guarantees that all threads must reach the barrier before any thread proceeds beyond it.

Therefore all threads execute their first printf() before any thread can execute the second printf().

However, the ordering among the different threads' "Hello" messages and among their "World" messages is not necessarily fixed.
51. High-Priority Things to Memorize
If your mid-sem is close, these are the must-know items from this section:
OpenMP
\[
\boxed{\text{Shared-memory parallel programming API}}
\]
Fork-Join
\[
\boxed{\text{Fork → parallel execution → join}}
\]
Thread ID
omp_get_thread_num()

Number of threads
omp_get_num_threads()

Parallel region
#pragma omp parallel

Parallel loop
#pragma omp parallel for

Scheduling
schedule(static)
schedule(dynamic)
schedule(guided)
schedule(auto)

Critical
#pragma omp critical

Atomic
#pragma omp atomic

Barrier
#pragma omp barrier

52. The Conceptual Chain You Should Remember
Don't memorize these as isolated OpenMP commands.

Understand the problem progression:
We have multiple cores
        ↓
Create multiple threads
        ↓
Threads share memory
        ↓
Shared data can cause races
        ↓
Need data scoping
        ↓
Need shared/private decisions
        ↓
Need to distribute loop work
        ↓
Scheduling
        ↓
Static / Dynamic / Guided / Auto
        ↓
Threads may need coordination
        ↓
Synchronization
        ↓
Critical / Atomic / Barrier

If you understand this chain, OpenMP becomes much easier.
53. MINI MID-SEM TEST
Try answering these without looking back.
Q1.
What is the difference between:
#pragma omp parallel

and
#pragma omp parallel for

Q2.
With 4 threads, what thread IDs can omp_get_thread_num() return?
Q3.
Why is the order of printf() statements from different threads unpredictable?
Q4.
What is the difference between a shared and a private variable?
Q5.
Why is static scheduling generally good for uniformly distributed workloads?
Q6.
Why is dynamic scheduling useful for irregular workloads?
Q7.
Why does guided scheduling start with larger chunks?
Q8.
What problem does critical solve?
Q9.
Why can atomic be preferable to critical for count++?
Q10.
What exactly does a barrier guarantee?
Q11.
Consider 16 iterations and 4 threads with schedule(static). Approximately how would the iterations be divided?
Q12.
What happens if schedule(static,4) is used for 8 iterations and 4 threads?
Answers
Q1: parallel executes a block using multiple threads; parallel for distributes loop iterations among the threads.

Q2: 0,1,2,3.

Q3: Threads execute concurrently and there is no guaranteed execution order.

Q4: Shared → common instance accessible by threads; private → separate instance for each thread.

Q5: Because work is predictable/even, so predetermined assignment has low overhead.

Q6: Because threads can obtain more work after finishing their current chunks, improving load balance.

Q7: To reduce scheduling overhead initially while still allowing finer load balancing later.

Q8: It prevents multiple threads from executing the protected region simultaneously.

Q9: Because atomic is designed for a specific memory update and can have lower overhead than protecting a whole critical region.

Q10: No thread proceeds beyond the barrier until all threads in the team have reached it.
One final distinction you absolutely should know
There are three different problems here:
Problem 1 — Who gets the work?
Scheduling
static / dynamic / guided / auto

Problem 2 — Who can access the data?
Data scoping
shared / private

Problem 3 — Who is allowed to proceed/access at a particular time?
Synchronization
critical / atomic / barrier

If you can clearly explain these three distinctions in an exam, you've understood the core of this section rather than simply memorizing OpenMP syntax.

Next section: the notes move deeper into synchronization and parallel execution, including ordering, implicit barriers, single, master, nowait, reductions, and more involved OpenMP code/output questions. Those are especially important for mid-sem because they combine the concepts above rather than asking only definitions.