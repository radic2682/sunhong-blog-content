---
id: "01HMCXW400W72D2WF48ZD1BAVM"
title_en: "Datapath Functional Units: Barrel Shifters, Comparators, Multipliers"
summary_en: "Notes on the functional units of a processor datapath. Covers the structure and area/delay characteristics of barrel shifters, equality and magnitude comparators, array multipliers and Booth encoding, Wallace and Dadda trees, and restoring and non-restoring dividers."
---
## Chapter 3: Datapath Functional Units Overview

This chapter studies the main functional units that make up a processor's **datapath**. It covers the core building blocks inside an ALU, such as the shifter, comparator, multiplier, divider, and counter, along with their design techniques.

## Shifter

Shift operations are essential for **fast approximations of multiplication and division** and for bit manipulation. The main kinds:

- **Logical shift** (LSL, LSR): fills with 0
- **Arithmetic shift** (ASL, ASR): preserves the sign bit (signed division)
- **Rotate** (ROL, ROR): circular
- **Rotate through carry**: circular, including the carry bit

### Barrel Shifter

An n-bit shifter that handles an arbitrary shift amount **within 1 cycle**. A signature unit of ARM.

- MUX tree structure: log n stages
- 8-bit, 16-bit, 32-bit … each stage either shifts or passes through
- Area: O(n log n), Delay: O(log n)

![Shifter types and barrel shifter|full](/img/01HMCXW400W72D2WF48ZD1BAVM/5477fdfa62b6.webp)

## Comparator

Compares two numbers A and B and decides A=B, A>B, or A<B.

### Equality Comparator

$A = B \iff \text{XNOR}(a_i, b_i) = 1 \text{ for all } i$

That is, XNOR each bit, then combine with AND.

### Magnitude Comparator

- Cascaded: compare from the most significant bit; if equal, move to the next bit
- Parallel: compare each bit pair, then use a priority encoder

![Comparator|full](/img/01HMCXW400W72D2WF48ZD1BAVM/d7694752df04.webp)

## Multiplier

### Array Multiplier

Generates every partial product in parallel and sums them with an adder tree:
```
   A3 A2 A1 A0
×  B3 B2 B1 B0
───────────────
4 partial products
   →  shift + add
```

Delay O(n), area O(n²).

### Booth Encoding

In signed multiplication, groups runs of consecutive 1 bits to reduce the number of partial products.
- **Booth-2 (radix-4)**: groups 3 bits at a time, halving the partial products
- **Booth-4 (radix-8)**: groups 4 bits at a time, cutting the partial products to 1/3

![Booth multiplier|full](/img/01HMCXW400W72D2WF48ZD1BAVM/d29eb1a3dd7a.webp)

### Wallace Tree Multiplier

A **reduction tree** that compresses the partial products with CSAs (carry-save adders):
- Depth = $O(\log n)$
- The final two numbers are summed with a CPA
- The standard integer multiplier in modern CPUs

![Wallace tree|full](/img/01HMCXW400W72D2WF48ZD1BAVM/603de6025bfb.webp)

### Dadda Tree

Similar to Wallace but uses slightly fewer adders. More area-efficient.

![Dadda tree|full](/img/01HMCXW400W72D2WF48ZD1BAVM/0e1a92fc1562.webp)

## Divider

Division is far more complex than multiplication.

### Restoring Division

- After subtracting, if the result is negative, **restore** the original value
- n-bit division: n cycles

### Non-Restoring Division

- No restore step; the sign decides the next operation
- Faster, and the common modern implementation

### SRT Division

- Redundant digit set (−2, −1, 0, +1, +2)
- Radix-4 and radix-8 SRT dividers → fewer cycles
- The Intel Pentium's famous FDIV bug was an error in the SRT lookup table

![Divider|full](/img/01HMCXW400W72D2WF48ZD1BAVM/35d15be823e2.webp)

## Counter

- **Binary counter**: n-bit ripple counter
- **Synchronous counter**: the same clock drives every flip-flop → less skew, higher frequency possible
- **Johnson counter, ring counter**: based on shift registers

![Counter|full](/img/01HMCXW400W72D2WF48ZD1BAVM/b8c70016e76e.webp)

## Multiplexer / Demultiplexer

- 2:1 MUX = `y = s·b + s̄·a` (AOI gate)
- An n:1 MUX takes log n select signals
- Large MUXes are decomposed into a tree

![MUX/DEMUX|full](/img/01HMCXW400W72D2WF48ZD1BAVM/cbf6fb2bca7e.webp)

## Register File

- SRAM-based storage with multiple ports (multi-port)
- Typical RISC: 2 read ports + 1 write port
- Out-of-order processors: 8+ read ports, 4+ write ports, etc.
- Area grows with the square of the port count

![Register file|full](/img/01HMCXW400W72D2WF48ZD1BAVM/3b005730d194.webp)

## Memory Arrays (SRAM/DRAM)

- **SRAM**: 6T cell, fast (0.5~1 ns), expensive, volatile
- **DRAM**: 1T1C cell, slow (tens of ns), cheap and dense
- SRAM for caches, DRAM for main memory

![Memory arrays|full](/img/01HMCXW400W72D2WF48ZD1BAVM/e044769e65ae.webp)

## Summary

| Unit | Purpose | Key technique |
|------|------|-----------|
| **Shifter** | Shift/rotate | Barrel shifter (log n) |
| **Comparator** | Comparison | Parallel prefix |
| **Multiplier** | Multiplication | Booth + Wallace tree |
| **Divider** | Division | SRT (Radix-4/8) |
| **Counter** | Counting | Synchronous |
| **MUX** | Selection | Tree of 2:1 |
| **RegFile** | Storage | Multi-port SRAM |

An ALU is a composite circuit that combines these functional units. Datapath performance is determined by the **combination of their latency and throughput**.
