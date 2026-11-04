---
id: "01HMCXW4002J3E8WZETZZMXANN"
title_en: "Computer Architecture Basics: ISA, Abstraction Layers, and Performance"
summary_en: "Notes on the introduction to a computer architecture course. Covers abstract data types (ADTs) with a stack example, the instruction set architecture (ISA), Hennessy and Patterson's engineering methodology and its two golden rules, and performance evaluation methods and design techniques."
---
## Chapter 1 Introduction: Computer Architecture Overview

These notes cover the introductory part of a computer architecture course: abstract data types (ADTs), the instruction set architecture (ISA), engineering methodology, and performance metrics.

## Abstract Data Type (ADT)

An **ADT** defines a data type **independently**. Separated from the implementation, it specifies only the set of values (state) and the operations defined on those values.

> A set of data values (state) and associated operations that are precisely specified **independent of any particular implementation**.

### Example: Stack

A stack is defined as follows.
- State: a linear data structure that supports `push, pop, top`
- Operation: `push(x)`, `x = pop()`, `x = top()`

Once the ADT is defined, it can be used through a logical interface regardless of the concrete implementation (array, linked list, and so on).

![Overview of ADT and ISA|full](/img/01HMCXW4002J3E8WZETZZMXANN/76626e6d4927.webp)

## Instruction Set Architecture (ISA)

The **ISA** is the **contract** where software meets hardware. It defines the set of instructions the processor understands, the register organization, the addressing modes, and so on.

### Key Assumptions

- **Registers** (general-purpose registers, PC, etc.), **Memory**: where programs and data are stored
- Every instruction causes a state change from Before to After.

### Example Instruction Behavior

When memory holds instructions such as `(j 15)`, `beq r0, r1, 20`, `sw r2, 0(r0)`, `lw r2, 1(r0)`, and `add r0, r1, r2`:

- **beq r0, r1, 2**: branch if equal. If `r0 == r1`, go to PC + 2 (offset); otherwise execute the next instruction (the instruction at address `24`).
- **Jump j 15**: an **unconditional jump** to address 15; the PC changes to 15.

Describing the register and memory state precisely before and after each instruction executes, as above, is the role of the ISA.

## Engineering Methodology (Hennessy & Patterson)

> The discipline of a [computing] system as seen by the programmer, i.e. the conceptual structure and functional behavior, as distinct from the organization of the data flow and controls, the logical design, and the physical implementation.
> — Amdahl, Blaauw, and Brooks, 1964

### The Two Golden Rules

- **Rule 1 — Identify and optimize the common case**: what happens often **has to be made fast**
- **Rule 2 — Make the rare case correct and reasonably fast**: what rarely happens **only needs to work**

In other words, the philosophy is to spend the optimization budget on the common path and only guarantee correctness for exception handling.

## Correctness Criteria

The criteria for **checking that a designed system is implemented correctly**. These conditions must be met before any performance optimization.

## Performance Evaluation Methods

### Performance Types

- **Time**
  - Response time — time from request to response
  - Execution time — time taken to run a program
  - Latency — delay of an individual operation
- **Rate** (amount of work per unit time)
  - Throughput: MIPS, MFLOPS
  - Bandwidth: Mbps
- **Ratio**
  - Relative performance — comparing performance between systems (either time or rate can be the basis)

## Design Techniques

The overall design of a computer system splits into the four large blocks below.

- **Processor** (including I-cache, D-cache, unified cache)
- **Memory Hierarchy** (memory organized in levels)
- **Interconnection Network**
- **Input/Output and Storage**

Representative techniques for optimizing these blocks:
- **Sequential execution / Pipelined execution**
- **Out-of-order execution / Speculative execution** — executing instructions out of order or ahead of time
- **Cache Design** — I/D-cache, unified cache
- **Cache Coherence, Synchronization, Interconnection network** — optimization for multiprocessor environments

→ The following chapters cover **Pipelining (Chapter 2)**, **Memory Hierarchy (Chapter 3)**, **Virtual Memory (Chapter 4)**, **Storage/IO (Chapter 5)**, and **Multiprocessor (Chapter 6)** in depth, one at a time.

![Engineering methodology and design techniques|full](/img/01HMCXW4002J3E8WZETZZMXANN/eb8b28702570.webp)
