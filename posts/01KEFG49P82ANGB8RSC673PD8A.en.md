---
id: "01KEFG49P82ANGB8RSC673PD8A"
title_en: "Intro to ARM Embedded Systems: ARM vs. Thumb Instruction Sets"
summary_en: "Notes from an introduction to ARM embedded systems. Covers what an embedded system is and why ARM is used there, and compares the data sizes and characteristics of the 16-bit Thumb instructions introduced in ARMv4 with the 32-bit ARM instructions."
---
## Chapter 1 Introduction: ARM and Embedded Systems Overview

An **embedded system** is a **system whose hardware or software is optimized** for a specific purpose and application. ARM is the best-known processor in embedded systems, characterized by low power consumption, small die size, and high performance.

## Data Sizes and Instruction Sets

ARM has supported a variety of data sizes and instruction sets across its generations.

- **Before ARMv4**: mostly 32-bit instructions
- **ARMv4 and later**: the **Thumb instruction set** was introduced
  - Supports both the original 32-bit ARM instructions and the 16-bit Thumb instructions
  - Defines design elements such as short physical registers, virtual addresses, and physical addresses

### ARM and Thumb Instructions

- **ARM instruction**: 32-bit, the full instruction set. Uses short physical registers, virtual addresses, physical addresses, and so on
- **Thumb instruction**: 16-bit, a condensed version of ARM. Introduced to reduce code size, which helps in embedded environments with little memory

## Instruction Sets

### ARM Instruction Set

- 32-bit fixed-length instructions
- Each instruction can execute in the same amount of time (a RISC trait)
- The pipeline runs efficiently

### Thumb Instruction Set

The Thumb instruction set **compresses** ARM's 32-bit instructions **into 16 bits**.

- **Variable-length instructions**: because they are 16-bit, they can be mixed with ARM instructions
- Code gets smaller, but some instructions need multiple cycles for complex operations
- With Thumb-2, ARM improved this by supporting both 16-bit and 32-bit Thumb instructions

### Basic Characteristics

- Why Thumb instructions were added: embedded environments have **small memory capacity**, so code space has to be saved
- They are slightly slower than ARM instructions, but fewer memory accesses can improve overall performance

![ARM/Thumb Instruction Set|full](/img/01KEFG49P82ANGB8RSC673PD8A/3e4386efee48.webp)

## Summary

- **ARM is the best-known processor in embedded systems** (systems whose hardware or software is optimized for a specific purpose and application)
- **ARM is becoming the face of RISC** (small size, high performance) vs. Intel's **CISC** (small memory access, growth after caches, ↓ pipelining, complex instructions)
  - **RISC (Reduced Instruction Set Computer)**: simple, fast instructions
  - **CISC (Complex Instruction Set Computer)**: complex instructions, more clock cycles
- **Pipelining is essential** (the basic technique of RISC)
  - Instructions can overlap and execute concurrently, at the risk of hazards
- **Low power is essential** (power vs. performance) → reducing leakage power through process development is important

![Summary of ARM characteristics|full](/img/01KEFG49P82ANGB8RSC673PD8A/4c99cba13219.webp)
