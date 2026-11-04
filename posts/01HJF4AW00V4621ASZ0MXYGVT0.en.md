---
id: "01HJF4AW00V4621ASZ0MXYGVT0"
title_en: "ARM Architecture and Registers: Banked Registers by Processor Mode"
summary_en: "Notes on the programmer's model of the ARM architecture. Covers the version lineage, visible registers, banked registers by processor mode, the PC and CPSR, supervisor mode, the load-store architecture, and data processing and control flow instructions."
---
## Chapter 3 ARM Architecture Overview

These notes cover the core parts of the ARM architecture: the Programmer\'s Model, visible registers, modes, exception handling, and development tools.

## Acorn RISC Machine → Advanced RISC Machine

ARM started as the Acorn RISC Machine and evolved into the Advanced RISC Machine. **It is RISC, but it has some CISC features**.
- Examples: **shifting the last operand** in an ALU operation, an I (MLA) that does a multiply and an add at once, and so on

## ARM Relationship Versions

- **Thumb I support**: Thumb I uses half the space (Thumb: 16-bit)
- ARM I, Data I → 32-bit processor/architecture support
- ARM I, Data I, Thumb I, Thumb decompressor support (ARM v7 family)
- ARM I, Data I, Thumb I, Thumb II

## Programmer's Model

Everything visible to the programmer: state + operations
- **State**: 15 general-purpose registers, PC, memory
- **Operation**: instructions
- Separation between **user code** ↔ **system code** areas

![ARM architecture overview|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/b0034c8c2bdd.webp)

## ARM's Visible Registers

ARM shows the programmer a different set of registers depending on the **mode**.

### Main Registers (User Mode)

- **R0 ~ R12**: general-purpose registers
- **R13 (SP)**: Stack Pointer
- **R14 (LR)**: Link Register (holds the return address on a branch)
- **R15 (PC)**: Program Counter
- **CPSR (Current Program Status Register)**: holds the current mode, flags, and interrupt masks

### Banked Registers by Mode

- User mode: R0~R15, CPSR
- **Privileged modes** (FIQ, IRQ, Supervisor, Abort, Undefined): **banked registers** used only in that mode (some of R8~R14) + the **SPSR** (Saved Program Status Register)
- ARMv7 added things such as **sticky overflow**

## Program Counter

- Before ARMv6: the PC is **the ordinary register R15**, holding a 32-bit aligned address.
- Because of the pipeline, using it as in `Mov Pc, rn` **acts like a jump (branch)**.

## CPSR (Current Program Status Register)

```
31 30 29 28    24   8 7 6 5   4   0
 N  Z  C  V    Q    I F T  M[4:0]
```

- **N**: Negative flag
- **Z**: Zero flag
- **C**: Carry flag
- **V**: Overflow flag
- **Q**: Sticky overflow (ARMv5TE+)
- **I**: IRQ disable
- **F**: FIQ disable
- **T**: Thumb mode flag
- **M[4:0]**: Processor mode

## Mode (Processor Mode)

- **User**: the mode for running ordinary applications
- **FIQ (Fast Interrupt)**: high-speed data transfer or channel process. Many banked registers → fast
- **IRQ (Interrupt)**: general-purpose interrupt handler
- **Supervisor (SVC)**: OS / kernel mode. Entered on reset and SWI
- **Abort**: memory abort (data/prefetch)
- **Undefined**: undefined instruction handler
- **System** (added in ARMv4): same as user mode but privileged → used to run OS tasks

Each mode has a **CPSR, SPSR**, and some banked registers.

![Registers, modes, CPSR|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/88920d5e03b0.webp)

## Supervisor Mode / Load-Store Architecture

### Supervisor Mode

When code running in user mode issues an SWI (supervisor call), it enters the OS. This mode is entered when system resources are needed.

### Load-Store Architecture

- A **RISC** trait: ALU operations are register-to-register only, and memory is accessed only through loads/stores
- Only loads/stores move data between memory and registers
- In contrast: in **CISC**, ALU I can also operate on memory

### ARM I: Data Processing I, Load/Store, Control Flow

- **Data processing I**: ALU, shift, and so on
- **Load/Store (transfer) I**: memory ↔ register
- **Control Flow**: Branch (B), BL, jump, conditional instructions

![SPSR, memory system, ARM little-endian|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/dd6d57e4c5ea.webp)

## ARM Instruction Feature

### Data Processing I

- I for operations such as addition, subtraction, and multiplication
- Using three registers is typical: `ADD r2, r1, r0`
- 2-operand form: `ADD r2, r1` (r2 ← r2 + r1)

### Load and Store I

- Post-indexing / pre-indexing (auto-indexing)
- Multiple load/store: LDM, STM

### Open Instruction Set Extension

- Used together with coprocessor I
- If the coprocessor is managed through interrupts, ARM waits
- The coprocessor issues a ready-bit interrupt
- If the coprocessor cannot execute the I, an Undefined I trap occurs (enters Abort mode)

### Not Thumb I: Thumb Included

Includes the most common basic Thumb instructions rather than a subset of ARM I.

## I/O System

Interrupts have all memory-mapped registers.

- I/O controllers are operated with LDR/STR
- Most I/O devices signal **completion through an interrupt**
- **FIQ mode** for fast interrupts
- **Local-sensitive and maskable**
- I/O reports through interrupts

## Exception

Situations that differ from normal flow, such as a memory abort, an undefined instruction, or a reset.

### Types of Exceptions

- **FIQ (Fast Interrupt Request)**: the interrupt that must be handled most urgently → generated by driving nFIQ low
- **IRQ (Interrupt Request)**: a less urgent interrupt → generated by driving nIRQ low
- **ABORT**: when a memory access fails
- **Software Interrupt (SWI)**: generated by an SWI (software interrupt instruction) when a user program needs more privileges → runs with supervisor privileges
- **Undefined Instruction Trap**: when ARM decodes an I, a coprocessor may also be involved in matching it against all instructions → the coprocessor tries to execute the I first, and it is handled by the coprocessor (if the coprocessor is ready but an interrupt occurs, ARM waits). If the coprocessor cannot execute the I, an Undefined I trap occurs (Abort mode check).

### Exception Priority

- Reset (highest priority) — highest because something went wrong
- Data abort
- FIQ
- IRQ
- Prefetch abort (when the following I involves memory)
- Undefined Instruction, Software Interrupt

When exception handling takes a while, the current processor state has to be restored. In other words, it must be possible to return to the running program after the exception and restore its state.

### Exception Vector

| Address | Exception | Mode on entry |
|---------|-----------|--------------|
| 0x00000000 | Reset | Supervisor |
| 0x00000004 | Undefined instruction | Undefined |
| 0x00000008 | Software interrupt | Supervisor |
| 0x0000000C | Abort (prefetch) | Abort |
| 0x00000010 | Abort (data) | Abort |
| 0x00000014 | Reserved | Reserved |
| 0x00000018 | IRQ | IRQ |
| 0x0000001C | FIQ | FIQ |

→ When an exception occurs, the PC is forced to branch to that address (handled by the branch/jump placed there)

### Entering an Exception: Procedure

1. The processor is running in user or another general mode (some registers are shared with user mode, while others are banked, so the handler can use its own registers without disturbing user mode)
2. Save the current PC (the return location) in R14 (LR)
3. Save the CPSR to the SPSR
4. Disable interrupts, because another interrupt arriving while one is being handled would cause problems
5. Change the CPSR mode bits to the value for that exception
6. Move the PC to the matching exception vector → the vector holds a branch I that jumps into the exception handler

![ARM instruction features, I/O, exceptions|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/d4f3ca547301.webp)
![Exception types and vectors|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/43d4e2f3649e.webp)

## Development Tools

### Example: User Mode vs FIQ Mode

- User: standard R0~R15, CPSR
- FIQ: banked R8~R14, SPSR → fast switching
- On a mode switch, R8~R14 are automatically swapped for the banked registers

### ARM Development Tools

- **Debugger** (ARMsd): software debugging
- **ARMulator**: a simulator that emulates the ARM processor core in software
- **ARM C Compiler**: compiles C code for ARM
- **ARM Assembler and Linker**: produces and links object files

### ARM C Compiler

- Complies with the **ANSI C** standard
- **Optimizations**: common sub-expression elimination, loop optimization, and so on

### ARM Assembler and Linker

- Combines object files into the final executable
- Resolves instruction addresses from the symbol table

![Example and development tools|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/aa356d4e32e9.webp)

## ARMsd (ARM Symbolic Debugger)

A **front-end interface** that helps with debugging.

### Ways to See Results

- **Under emulation** → ARMulator (rarely used)
- **Communication interface** → a serial line (UART), a JTAG interface (can reveal even hardware problems)

### Features

- Works much like a C debugger: **you can set breakpoints and inspect variables, registers, and memory**
- Graphical interface

## ARMulator

A **simulator** that emulates the ARM processor core in software. Only the terminology is covered here.

### Levels of Accuracy

- **Instruction accurate**: enough for a typical software engineer
- **Cycle accurate**: needed by chip designers (whether setup and hold times are met)
- **Timing accurate**: accurate per operation, needed by layout designers

## Summary

- **ARM: Acorn RISC Machine → Advanced RISC Machine**
- ARM is RISC but has some CISC features. Examples: shifting the last operand in an ALU operation, an I that does a multiply and an add at once
- **Programmer\'s model**:
  - State: 15 general-purpose registers, PC, memory
  - Operation: instructions
- **ARM development tools**: Debugger, ARMulator

![ARMsd and ARMulator|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/a78e55746098.webp)
