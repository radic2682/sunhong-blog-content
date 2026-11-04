---
id: "01HMCXW4008G4Z2T0SFWT0ZCMB"
title_en: "SoC Design Frameworks and the Design Productivity Gap"
summary_en: "Notes on the SoC design flow. Covers the design productivity gap, where design productivity fails to keep up with process advances, and its remedies (abstraction, EDA automation, IP reuse), an ARM-based SoC example, a comparison of SoC, MCU, and CPU, and Apple SoCs with their die sizes."
---
## Chapter 3: SoC Design Overview

**SoC (System-on-Chip)** design depends on **IP reuse, the design framework, and the choice of methodology**. This chapter covers the SoC design flow, required characteristics, a comparison of MCU vs. CPU vs. MPU, and the Apple SoC family.

## SoC Design Framework

### The Design Productivity Gap

The growth in semiconductor process complexity outpaces designer productivity. Thanks to **Moore's Law**, transistor counts grow explosively every year, but designer productivity does not keep up → the **productivity gap**.

### Ways to Improve Productivity

- **① Design Abstraction**: work at a higher level of abstraction (behavioral → RTL → gate level)
- **② Design Automation**: automate with EDA tools
- **③ IP Packaging (Best Strategy)**: reuse already-verified IP in many places
- **④ Design Reuse**: use open-source IP and standard IP
- **⑤ Standard Tools and Platforms**: standard interfaces (AMBA, etc.)

![SoC Design Framework|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/7d309fd9fb01.webp)

## SoC Example: ARM-Based SoC

### Example: Bluetooth Platform SoC

- ARM7 processor + OS and support logic
- ARBITER, PLL (Lock Loop), shared memory → the basic building blocks

### SoC Structure

- **System Master**: initiators of transactions, such as the CPU and DMA
- **System Slave**: passive responders, such as memory and peripherals

## Required SoC Characteristics

- **Low cost**
- **High performance**
- **Lighter footprint** (light)
- **Lower efficiency** (higher power efficiency)
- **Less flexibility** (because it is single-use, lower complexity means lower cost)
- **Application specific** (chips are usually specialized for an application domain)

## SoC vs. MCU vs. CPU

| Item | SoC | MCU | CPU |
|------|-----|-----|-----|
| **CPU** | Yes | Yes | Yes |
| **Memory** | Separate board (external) | Built-in ROM/RAM | Separate |
| **Peripheral** | Many built in | Built in (timer, UART, etc.) | Separate |
| **Use** | Mobile/embedded systems | Small embedded | General-purpose desktop/server |
| **Examples** | Apple A-series | STM32, PIC | Intel x86, ARM Cortex-A |

## SoC Design Flow

SoC design proceeds in the following flow:

1. **Design specs**: define the requirements
2. **Partitioning**: decide the HW/SW split
3. **HW co-verification**: verify HW-SW integration on a virtual platform
4. **Software dev**: develop the compiler, OS, and apps in parallel
5. **Packaging**: GDSII → Fab → Package → Test

![SoC required characteristics and design flow|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/4bac483e0d06.webp)

## ARM Families

- **A (Application)**: high performance (mobile/tablet APs)
- **R (Real-time)**: real-time systems (automotive, HDD)
- **M (MCU)**: low-power microcontrollers

## Example: Apple SoC Families

| SoC | Model No. | CPU | CPU ISA | Technology | Die size | Date | Devices |
|-----|-----------|-----|---------|-----------|----------|------|---------|
| N/A | APL0098 | ARM11 | ARMv6 | 90 nm | N/A | 6/2007 | iPhone, iPod Touch (1st gen) |
| A4 | APL0398 | ARM Cortex-A8 | ARMv7 | 45 nm | 53.29 mm² | 3/2010 | iPad, iPhone 4, Apple TV (2nd gen) |
| A5 | APL0498 | ARM Cortex-A9 | ARMv7 | 45 nm | 122.6 mm² | 3/2011 | iPad 2, iPhone 4S |
| A5X | APL5498 | ARM Cortex-A9 | ARMv7 | 45 nm | 162.94 mm² | 3/2012 | iPad (3rd gen) |
| A6 | APL0598 | Swift | ARMv7s | 32 nm | 96.71 mm² | 9/2012 | iPhone 5 |
| A6X | APL5598 | Swift | ARMv7s | 32 nm | 123 mm² | 10/2012 | iPad (4th gen) |
| A7 (64-bit) | APL0698 | Cyclone | ARMv8-A | 28 nm | 102 mm² | 9/2013 | iPhone 5S, iPad mini (2nd gen) |
| A7 | APL5698 | Cyclone | ARMv8-A | 28 nm | 102 mm² | 10/2013 | iPad Air |

### Die Size: The Size (Area) of a Chip

→ As processes shrink, designers try to pack in more features, so the die grows larger.

With the A7, Apple became the first in the industry to bring **64-bit ARMv8-A** to a smartphone. The Cyclone core was the starting point of Apple's vertically integrated silicon.

![Apple SoC Families|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/236fd5ee5aa1.webp)

## Summary

| Concept | Key point |
|------|------|
| **Productivity Gap** | Process complexity > design productivity → overcome with IP reuse |
| **SoC** | CPU + memory + peripherals on one chip |
| **Low cost, High perf, Low power** | Basic SoC requirements |
| **ARM A/R/M** | Cortex series split by use |
| **Apple A-series** | Cortex-based SoCs for iPhone/iPad, 64-bit from the A7 |

The success of an SoC design depends on **IP reuse** and **HW-SW co-verification**.
