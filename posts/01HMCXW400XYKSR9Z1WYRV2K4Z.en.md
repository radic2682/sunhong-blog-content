---
id: "01HMCXW400XYKSR9Z1WYRV2K4Z"
title_en: "Cortex-M NVIC Interrupt Handling: Vector Table and Exception Priority"
summary_en: "Notes on interrupt handling in the Cortex-M. Covers the difference between interrupts and exceptions, vectored and nested interrupts in the NVIC, the vector table and handling sequence, exception priority and preemption, tail-chaining and late arrival, and PRIMASK, FAULTMASK, and BASEPRI masking."
---
## Chapter 8: Interrupt Mechanism Overview

An **interrupt** is a mechanism that halts the processor's normal flow of execution, runs a specific handler routine, and then returns. It is essential for reacting immediately to asynchronous events such as peripherals, timers, and faults.

## Interrupt vs Exception

- **Interrupt**: Asynchronous (raised by external events or peripherals)
- **Exception**: Synchronous (raised during instruction execution, e.g. SVC, faults)
- ARM handles both uniformly as **exceptions**

## NVIC (Nested Vectored Interrupt Controller)

**The heart of the Cortex-M**: the NVIC handles interrupts.

### Key Features

- **Vectored interrupts**: A dedicated vector for each IRQ (automatic branching)
- **Nested interrupts**: A higher-priority IRQ preempts a lower one (preemption)
- **Priority levels**: Programmer-configurable (M0: 4 levels, M3: 8~256)
- **Dynamic priority**: Can be changed at runtime
- **Masking**: PRIMASK, FAULTMASK, BASEPRI

## Vector Table

Located at memory address 0x00000000. Stores the handler address of each exception:

| Offset | Vector | Exception |
|--------|--------|-----------|
| 0x00 | Initial SP | — |
| 0x04 | Reset | Reset |
| 0x08 | NMI | Non-maskable |
| 0x0C | HardFault | Fault escalation |
| ... | ... | ... |
| 0x2C | SVCall | SVC instruction |
| 0x38 | PendSV | Pended Service |
| 0x3C | SysTick | System timer |
| 0x40 ~ | IRQ0 ~ IRQn | External IRQs |

On reset, the Initial SP value is loaded into R13, and the PC jumps to the Reset vector.

![NVIC and Vector Table|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/b004412d3647.webp)

## Interrupt Handling Sequence

1. **External event** (e.g. UART RX) → the peripheral asserts its IRQ signal
2. The **NVIC** detects the IRQ and checks its priority
3. If its priority is higher than the currently running code, it **preempts**
4. **Stacking**: Push xPSR, PC, LR, R12, R3, R2, R1, R0 onto the stack (8 words)
5. **Vector fetch**: Read the handler address from the vector table
6. **Run the handler** (Handler mode, privileged)
7. **Exit**: `BX LR` or `POP {..., PC}` — loads the EXC_RETURN value
8. **Unstacking**: Automatically restores from the stack frame and returns to the original code

## Exception Priority

- Fixed priority: Reset (-3) > NMI (-2) > HardFault (-1)
- Configurable: All other exceptions
- **Lower number = higher priority**

The Cortex-M0 priority field is 2 bits → 4 levels. The Cortex-M3 uses 3~8 bits.

## Preemption & Nesting

```
Main → IRQ_A handler running
      → higher-priority IRQ_B occurs
      → IRQ_A suspended (preempted), new stack frame pushed
      → IRQ_B handler runs
      → IRQ_B done → return to IRQ_A
      → IRQ_A done → return to Main
```

Frames pile up on the stack, so enough stack depth must be reserved.

## Tail-Chaining

A Cortex-M optimization: when handling back-to-back IRQs, **unstacking + restacking is skipped**:

```
Main → IRQ_A → (IRQ_B pending as soon as it finishes)
                → skip the stack pop and call the IRQ_B handler directly
                → after IRQ_B, unstack only once → return to Main
```

→ Lower latency and power savings.

## Late Arrival

If a higher-priority IRQ_B arrives while IRQ_A is still being stacked → the core branches to IRQ_B as soon as stacking completes (IRQ_A stays pending).

## Interrupt Masking

### PRIMASK

- bit 0: When 1, masks all configurable exceptions (except NMI and HardFault)
- Set with `CPSID I` / `CPSIE I`

### FAULTMASK (Cortex-M3 and above)

- When 1, masks everything except NMI

### BASEPRI (Cortex-M3 and above)

- Masks IRQs with priority lower than this value

![Interrupt handling and masking|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/3d1733422057.webp)

## NVIC Registers

- **ISER** (Interrupt Set-Enable Register)
- **ICER** (Interrupt Clear-Enable)
- **ISPR** (Set Pending)
- **ICPR** (Clear Pending)
- **IP** (Interrupt Priority) — sets the priority of each IRQ

Example: enabling IRQ 5:
```c
NVIC->ISER[0] = (1 << 5);
```

## Writing an Exception Handler

Handlers can be written in C. The compiler handles everything automatically according to the AAPCS (ARM calling convention):

```c
void USART1_IRQHandler(void) {
    if (USART1->SR & USART_SR_RXNE) {
        uint8_t data = USART1->DR;
        // handle
    }
}
```

The handler address must be registered at 0x4C in the vector table (assuming IRQ 3). This is set up in the startup code (linker script).

![Interrupt handler example|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/ccbd3f4fb47b.webp)

## Summary

| Concept | Description |
|------|------|
| **NVIC** | Nested Vectored Interrupt Controller |
| **Vector Table** | Exception handler addresses starting at 0x00000000 |
| **Priority** | Lower number = higher priority |
| **Preemption** | A higher-priority IRQ preempts the running handler |
| **Tail-chaining** | Skips stacking when handling back-to-back IRQs |
| **Late-arriving** | Branches to a new IRQ that arrives during stacking |
| **Masking** | PRIMASK / FAULTMASK / BASEPRI |

By providing **real-time responsiveness and priority-based preemption**, the NVIC is the key reason the Cortex-M became the standard for real-time embedded systems.

![Interrupt overview|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/510fdbf101a4.webp)
