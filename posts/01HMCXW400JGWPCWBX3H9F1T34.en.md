---
id: "01HMCXW400JGWPCWBX3H9F1T34"
title_en: "Multiprocessors and Shared-Memory Sync: Critical Sections, Producer-Consumer"
summary_en: "Notes on multiprocessor architecture. Covers UMA and NUMA shared memory, a sum reduction example, the producer-consumer and critical-section problems, Test-and-Set mutual exclusion and optimized synchronization, message passing and clusters, and interconnect structures."
---
## Chapter 6: Multiprocessor Overview

As single processors hit the limits of clock frequency, **multiprocessor architectures that use many processors in parallel** became the mainstream of computer architecture. This chapter covers shared memory / message passing, cache coherence, synchronization, clusters, and interconnect topologies.

## Shared Memory Multiprocessor

```
[P1]──[Cache]──┐
[P2]──[Cache]──┼──[Interconnection Network]──[Memory]
[P3]──[Cache]──┘
```

- **UMA (Uniform Memory Access)**: every processor has the same memory access time
- **NUMA (Non-Uniform Memory Access)**: memory close to a processor is fast to access, distant memory is slow
- Memory access times differ as well

### Example: Sum Reduction

Code in which 100 processors sum 100,000 numbers:

```c
sum[Pn] = 0;
for (i = 10000*Pn; i < 10000*(Pn+1); i = i + 1)
    sum[Pn] = sum[Pn] + A[i];
/* now add the sub-sums */
half = 100;
repeat
    synch();
    if (half%2 != 0 && Pn == 0)
        sum[0] = sum[0] + sum[half-1];
    /* Conditional sum needed when half is odd */
    half = half/2;  /* dividing line on who sums */
    if (Pn < half)
        sum[Pn] = sum[Pn] + sum[Pn+half];
until (half == 1);
```

Why synchronization is needed: a race condition occurs when several processors read and write the same variable at the same time.

## Synchronization in Shared Memory

### Example: the producer-consumer problem

```c
Shared data
int counter = 0;
int buffer[BUFFER_SIZE];
int in = 0; int out = 0;

/* Producer */
while (counter == BUFFER_SIZE) ;  // do nothing
buffer[in] = next_produced;
in = (in + 1) % BUFFER_SIZE;
counter++;

/* Consumer */
while (counter == 0) ;  // do nothing
next_consumed = buffer[out];
out = (out + 1) % BUFFER_SIZE;
counter--;
```

If `counter++` and `counter--` are not atomic, a **race condition** occurs.

Example: `counter++` = `register1 = counter; register1 = register1 + 1; counter = register1;`

If two processors interleave, an update can be lost → the **critical section** problem.

![Shared memory and synchronization|full](/img/01HMCXW400JGWPCWBX3H9F1T34/4fe26555a34b.webp)

## The Critical-Section Problem

```
repeat
    Entry Section
       critical section  ← only one processor enters at a time
    Exit Section
       remainder section
until false;
```

### Correctness Criteria for a Solution

- **Mutual Exclusion**: only one processor can be in a critical section
- **Progress**: if a processor wants to enter the critical section, it can eventually enter
- **Bounded Waiting**: a processor trying to enter the critical section must get in within finite time (prevents starvation)

![Critical-section problem|full](/img/01HMCXW400JGWPCWBX3H9F1T34/ae36348ceada.webp)

## Mutual Exclusion with Test-and-Set

```
repeat
    while Test-and-Set(lock) do no-op;   // Entry Section
        critical section
    lock := false;                       // Exit Section
       remainder section
until false;
```

- If the lock value is 1, execute a no-op (loop, busy-wait)
- A software instruction sets the lock back to 0

### Problem when P0 and P1 read the lock bit at the same time

Both read the lock bit as 0 at the same time, which is the problem. **Test-and-Set** prevents this as an atomic operation (the moment a processor tries to use it, the lock bit is set to 1 right as it is read).

→ **It satisfies mutual exclusion and progress, but not bounded waiting** (since the probability is low, it may be left unimplemented).

## Naive Synchronization vs Optimized Synchronization

### Naive (Test-and-Set in a loop)

```
Try to lock variable using swap:
    read lock variable and then set
    variable to locked value (1)

Succeed? (= 0?)
    Yes → Begin update, Finish update, Unlock (set lock variable to 0)
    No  → loop
```

It is functionally correct, but it has a **performance problem**:
- P0 in use → lock bit: 1
- P1 cannot access → lock bit: 1 (invalidates the lock bit in the other caches)
- P2 cannot access → lock bit: 1 (invalidates the lock bit in the other caches)

→ **Memory accesses keep happening** (interconnect delay, power consumption).

### Optimized Synchronization (Load-first)

- No memory traffic → good for power
- **Load the lock value** before Test-and-Set
- `lock: 1` → keep loading
- `lock: 0` → Test-and-Set

If the load returned 0 but Test-and-Set returns 1, someone else grabbed it first.

- P0 in use → lock bit: 1
- P1, P2 access → load (lock bit in cache: 1) → keep hitting in the cache (no memory access)
- P0 finishes → lock bit becomes 0, and the invalidate protocol makes the lock bit 0 in the other processors' caches
- P1, P2 access → load (lock bit in cache: 0) → Test-and-Set, and one of P1 and P2 grabs it

![Test-and-Set and optimized sync|full](/img/01HMCXW400JGWPCWBX3H9F1T34/3f779401e80d.webp)

## Message Passing

- Each processor has its own physical address space
- The hardware sends and receives messages between processors

```
[Processor]─[Cache]─[Memory]  ...  [Processor]─[Cache]─[Memory]
          └──────── Interconnection Network ────────┘
```

### Loosely Coupled Clusters

- Requires **a network connecting independent computers** → uses the I/O system (Ethernet/switch/Internet)
- Each has its own memory and CPU
- Suited to applications with independent tasks (web servers, databases, simulations …)
- High **availability, scalable, affordable**

→ **Shared memory has a limit on how many processors can be connected**, while message passing can use more processors.

[Drawbacks]: management is expensive, and if the interconnect performs poorly, overall performance drops → **use SMP (Symmetric Multiprocessor) → high performance, high bandwidth**.

### Sum Reduction in Message Passing

- ① Someone has to hand out the numbers to add, 1000 at a time (with shared memory there is no need to distribute them)
- ② Each processor adds its 1000 numbers
- ③ The results of the 100 processors are combined (half send, half receive and add) — repeat
- **No explicit synchronization step needed** → sending or receiving a message provides synchronization (if the sending or receiving processor is not ready, there is no response → synchronization happens naturally)

```c
limit = 100; half = 100;  /* 100 processors */
repeat
    half = (half+1)/2;  /* send vs. receive dividing line */
    if (Pn >= half && Pn < limit)
        send(Pn - half, sum);
    if (Pn < (limit/2))
        sum = sum + receive();
    limit = half;  /* upper limit of senders */
until (half == 1);  /* exit with final sum */
```

## Network Topology

### Example NUMA topology

```
[Switch]─[Switch]─[Switch]─[Switch]
   │        │        │        │
[Switch]─[Switch]─[Switch]─[Switch]
   │        │        │        │
  ...
```

**2-D grid or mesh**, **3-D n-cube (hypercube)**, and so on.

- Multiple **processing elements (PEs)** connected by switches
- Number of hops to reach the farthest node = $n$ (n-cube)

![Message passing and network topology|full](/img/01HMCXW400JGWPCWBX3H9F1T34/352d6e88ea0f.webp)

## Crossbar & Multi-Stage Interconnection Network

### Crossbar

Can connect several pairs at once. Example: P0 and P3 can be connected.

### Omega Network (Multi-stage interconnection network)

- With 2×2 switches: for 16 processors, $\log_2(16) = 4$ → **4 stages needed**
- With 4×4 switches: for 16 processors, $\log_4(16) = 2$ → **2 stages**

```
Switch (connects only one pair)
Crossbar (can connect several pairs)
```

## The Evolution-Revolution Spectrum of Computer Architecture

```
Evolutionary ────────────────────────────→ Revolutionary
Pipelining → Cache → Timeshared → Virtual → RISC → CC-UMA → CC-NUMA → Not-CC-NUMA → Message-passing → Massive SIMD → Parallel processing
```

- **Biggest advances**: pipelining, caches
- **CC (Cache Coherency)**: keeps the caches coherent
- **SIMD**: Single Instruction Multiple Data — same instruction, different data (used most for prediction)

### RISC vs CISC

- **RISC (Reduced Instruction Set Computer)**: uniform execution time, **used by most processors today**. Advantage: efficient pipelining.
- **CISC (Complex Instruction Set Computer)**: instructions have different lengths (and different execution times). It came from a time when memory was expensive and instruction space had to be minimized.

→ Intel processors are RISC-style in hardware and CISC-style in software, translating between them → **for application compatibility**. Most processors are RISC-style.

![Crossbar, Omega, RISC vs CISC|full](/img/01HMCXW400JGWPCWBX3H9F1T34/7165d41b37e6.webp)

## Summary

| Concept | Description |
|------|------|
| **Shared Memory MP** | All processors share the same memory space (UMA/NUMA) |
| **Message Passing MP** | Each has its own memory and communicates by messages (cluster, HPC) |
| **Cache Coherence** | Keeps values consistent across multi-level caches |
| **Synchronization** | Atomic operations via Test-and-Set / load-first |
| **Network Topology** | Crossbar, Omega, 2D mesh, n-cube |
| **SIMD** | One instruction processes many data at once (the basis of GPUs) |

Multiprocessors are the theoretical foundation of all the parallel computing that followed: **multi-core CPUs + GPUs + clusters**.
