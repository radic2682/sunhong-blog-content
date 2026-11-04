---
id: "01HJM94A00G4VPKA2KHMFBKTYY"
title_en: "Radix-8 Booth Multiplier: Encoding Table and TDM Trade-offs (Verilog)"
summary_en: "I designed a multiplier in Verilog that combines Radix-8 Booth encoding with TDM. This post covers how the Booth algorithm works, the number of partial products and hardware trade-offs at each radix, the Radix-8 encoding table, implementation and verification, and lessons learned."
---
![Radix-8 Booth Encoding Multiplier using TDM|large](/img/01HJM94A00G4VPKA2KHMFBKTYY/af70d3cf8de5.webp)

This design project combines a multiplier using Radix-8 Booth encoding with TDM (time-division multiplexing). It is an extension of the basic Booth algorithm, and the goal is to get the most hardware efficiency out of multiplication.

## Overview of the Booth multiplication algorithm

Conventional binary multiplication repeats a shift and a conditional add for every bit of the multiplier. A 32-bit multiplication needs up to 32 additions, so it is slow. The Booth algorithm reduces the number of additions by replacing runs of consecutive 1 bits with a combination of a subtraction and an addition.

For example, multiplying by binary `0111` (decimal 7) normally takes three additions, but Booth reads it as `1000 - 0001` (= 8 - 1 = 7) and handles it with one addition and one subtraction. The Modified Booth algorithm extends this idea and has evolved into Radix-2, Radix-4, Radix-8, and so on.

## What the radix means, and the trade-offs

Radix-N refers to how many bits are encoded at a time:

- **Radix-2**: encodes 2 bits at a time, N/2 partial products
- **Radix-4**: encodes 3 bits at a time (2 bits + 1 previous bit), N/2 partial products (half of the previous)
- **Radix-8**: encodes 4 bits at a time (3 bits + 1 previous bit), N/3 partial products

A larger radix means fewer partial products and fewer pipeline stages, but the logic that generates each partial product gets more complex. From Radix-8 on, you need non-power-of-two multiples such as 3× (3×multiplicand), which must be precomputed.

## Radix-8 Booth encoding table

A 3-bit window plus 1 previous bit gives 4 bits, which select one of the following operations:

| Bit pattern | Operation | Description |
|----------|------|------|
| 0000, 1111 | 0 | Add nothing |
| 0001, 0010 | +1× | Add the multiplicand |
| 0011, 0100 | +2× | multiplicand × 2 (shift) |
| 0101, 0110 | +3× | multiplicand × 3 (precomputed) |
| 0111 | +4× | multiplicand × 4 (shift) |
| 1000 | -4× | -multiplicand × 4 |
| 1001, 1010 | -3× | -multiplicand × 3 |
| 1011, 1100 | -2× | -multiplicand × 2 |
| 1101, 1110 | -1× | -multiplicand |

The key is generating **3× multiplicand**. It is computed as `1× + 2×`, once before the multiplication starts, and then reused throughout.

## Applying TDM (time-division multiplexing)

TDM is a technique in which one hardware unit is shared by different data across time slots. In this project, the adder is shared through TDM, which greatly reduces the total hardware area.

A Radix-8 Booth multiplier normally sums its N/3 partial products with a CSA (carry-save adder) tree such as a Wallace tree. That is fast but large. With TDM, a single CSA line is reused over time: area goes down, while throughput drops by a factor equal to the number of time slots.

This design is useful when throughput is not critical for the application, or when FPGA/ASIC area is limited.

## Design approach

The detailed explanation is in the Radix-4 post. The design approach is not very different.

[Read the Radix-4 explanation](post:01HJM94A00V6YP4RXA0WMNCGGA)

Main differences from Radix-4:
- The encoder window grows to 3 bits + 1 = 4 bits
- 3× precomputation logic is added
- The number of partial products drops from N/2 to N/3 (16 → 11 for 32 bits)

## Implementation and verification

I wrote the RTL in Verilog and built a testbench with a variety of inputs (positive, negative, extreme values) to verify the products. I focused in particular on whether sign extension is correct for negative inputs in signed multiplication, and on how the result is handled when overflow occurs.

The code is available on GitHub.

[View on GitHub](https://github.com/radic2682/radix-8_Booth_Multiplier.git)

## Lessons learned

- **The area-speed trade-off**: Raising the radix reduces the number of partial products, but the complexity of the encoder and partial product generator grows exponentially. This is why Radix-4 is the most widely used in real projects.
- **The value of TDM**: If the clock frequency is high enough, TDM can save a lot of area. It suits low-power IoT chips.
- **The importance of sign extension**: Handling negative partial products is tricky in the Booth algorithm. Sign extension bugs only affect the upper bits of the result, so they are easy to miss.
- **The value of precomputation**: Computing values like 3× every time lengthens the critical path. The key is to compute them once and keep them in a register.
