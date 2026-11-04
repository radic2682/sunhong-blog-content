---
id: "01M6K7F34E024SNZYYF8JSZTCA"
title_en: "V Extension 1: VLEN, SEW, LMUL, vsetvli, and Vector-Length-Agnostic Code"
summary_en: "Part 13 of the RISC-V series. Covers vector registers, VLEN, SEW, and LMUL, and the rules by which vsetvli sets vl. Running the same machine code on QEMU at VLEN 128 to 1024, a 100-element add looped 25, 13, 7, and 4 times and executed 313 to 61 instructions (813 for scalar)."
---
:::note
**Key points in three lines**
- The RISC-V vector extension (V) does not bake the register width into the ISA. The length VLEN of the 32 vector registers is chosen by the implementation, and the program uses `vsetvli` to say "n elements remain, each 32 bits," and receives from the hardware the count vl to process this time.
- The maximum number of elements handled at once is VLMAX = VLEN / SEW × LMUL. SEW is the element width, and LMUL is how many registers to group together (1/8~8). These three plus vl determine the behavior of every vector instruction.
- Running the same machine code that adds 100 integers on QEMU at VLEN 128, 256, 512, and 1024, the loop ran 25, 13, 7, and 4 times and executed 313, 169, 97, and 61 instructions (813 for the scalar loop). The results matched the Python reference in all four cases.
:::

Part 1 mentioned V only as one entry in the list of single-letter extensions and as an extension the RVA23 profile makes mandatory. The base integer ISA in parts 2~5, multiplication in part 7, floating point in part 10, and compressed instructions in part 11 were all scalar instructions that handle one value at a time. The B extension in part 12 also only works on bits within one register. Parts 13 and 14 cover the V extension, which processes dozens of elements with one instruction. This part looks at its foundation: vector registers, `vtype` and `vl`, and the principle that lets the same code run on hardware of any width (vector-length-agnostic code). Masks, load/store types, reductions, and permutations are in part 14. The source is Chapter 31 of the Unprivileged specification, version 20240411, "V" Standard Extension for Vector Operations, Version 1.0.

## 1. Why Not Fixed-Width SIMD

x86 SSE uses 128-bit registers, AVX 256-bit, and AVX-512 512-bit. Each width change brought new instructions and new registers, and programs had to be recompiled or written separately for each width. Fixed-width SIMD also has a tail problem. Processing 100 elements 4 at a time divides evenly into 25 passes, but with 102 elements the remaining 2 must be handled separately in a scalar loop.

RISC-V V does not write the register width into the ISA. The width VLEN is a constant chosen by the implementation, and code does not need to know it. Each time through, the loop tells the hardware "how many elements remain," and the hardware answers "I'll do vl this time." Wide hardware processes many at once, narrow hardware a few at a time, and in the last iteration vl shrinks to the remaining count, so there is no separate tail to handle. This revives the approach of 1970s vector computers like the Cray (a vector length register).

From the designer's side, this decision widens the options. A small MCU can use VLEN=128 and process one element per cycle, while a server core can use VLEN=512 with several lanes. Both run the same binary.

## 2. Vector Registers, VLEN, and ELEN

The V extension adds 32 vector registers (`v0`~`v31`). This is a new register file, separate from the integer and FP registers. Each register is VLEN bits, and there are two implementation-defined constants.

- **VLEN**: the number of bits in one vector register. A power of 2, at most 2^16.
- **ELEN**: the maximum element width in bits that one operation can handle. A power of 2, at least 8 and at most VLEN.

The full V extension (V among the standard vector extensions in 31.18) requires VLEN ≥ 128 and ELEN = 64. Smaller embedded bundles exist separately under names like Zve32x and Zve64d. In practice, the extension string in an ELF compiled with `-march=rv64gcv` contained `zvl128b` (guaranteeing VLEN ≥ 128) and `zve64d`.

Seven CSRs hold the vector state.

```text
vstart  0x008  element index to restart from after an exception
vxsat   0x009  fixed-point saturation flag
vxrm    0x00A  fixed-point rounding mode
vcsr    0x00F  vxrm and vxsat combined
vl      0xC20  number of elements to process (read-only)
vtype   0xC21  SEW, LMUL, policies (read-only)
vlenb   0xC22  VLEN/8 (read-only constant)
```

`vl` and `vtype` cannot be written with CSR instructions. Only the `vsetvli` family (and the fault-only-first loads of part 14) change them. Reading `vlenb` gave 16, 32, 64, and 128 at QEMU VLEN 128, 256, 512, and 1024.

The vector unit can start out turned off. When `mstatus.VS` (bits 10:9) is 0 (Off), vector instructions and vector CSR accesses all raise illegal instruction exceptions. It has the same structure as `mstatus.FS` for FP, and lets the operating system avoid saving and restoring 32×VLEN bits of registers for processes that do not use vectors. So the startup code for this exercise turned VS on to 1 (Initial) with `csrs mstatus` before `main`. Running `vsetvli` with it off produced mcause=2, as in section 7.

## 3. SEW, LMUL, and VLMAX

How many bits each element in a register has is **SEW** (selected element width), chosen from 8, 16, 32, and 64 bits. How many registers are used as one bundle (a register group) is **LMUL** (length multiplier). Values 1, 2, 4, and 8 group registers together, and 1/2, 1/4, and 1/8 use only part of a register. The maximum number of elements one instruction can handle is:

$$\text{VLMAX} = \frac{\text{VLEN}}{\text{SEW}} \times \text{LMUL}$$

Dividing the register bit count by the element width gives the number of elements in one register, and that is multiplied by the number of grouped registers. With VLEN=128, SEW=32, and LMUL=4, it is 128/32×4 = 16.

![At VLEN 128, one register holds 16 elements at e8 and 4 at e32; e32 m4 groups v8~v11 for 16 elements, and e32 mf2 uses only half for 2|full](/img/01M6K7F34E024SNZYYF8JSZTCA/5da113415e32.webp)

With LMUL=4, a group is named by its lowest register number. Writing `v8` means `v8`~`v11`, with elements 0~3 in `v8`, 4~7 in `v9`, and so on. So a group can only start at a register number that is a multiple of LMUL. Using `v1` with LMUL=2 is a reserved encoding.

Fractional LMUL is used when mixing elements of different widths. Consider an operation that widens 8-bit values to 32-bit. If the VLMAX of e32·m1 is 4, the 8-bit side holding the same 4 elements needs only 1/4 of a 128-bit register (e8·mf4, VLMAX = 128/8×1/4 = 4). Without fractional LMUL, either the narrow side would take a whole register and waste the rest, or the wide side would have to group several registers.

These are the VLMAX values returned when setting all four SEWs × seven LMULs with `vsetvl` on QEMU.

| Setting | VLEN 128 | VLEN 512 | Note |
|---|---|---|---|
| e8, m1 | 16 | 64 | |
| e8, mf8 | 2 | 8 | Smallest unit |
| e32, m1 | 4 | 16 | |
| e32, m4 | 16 | 64 | Groups v8~v11 |
| e32, mf2 | 2 | 8 | |
| e32, mf4 | vill | vill | 32 > 1/4 × 64 |
| e64, m8 | 16 | 64 | |
| e64, mf2 | vill | vill | 64 > 1/2 × 64 |

For fractional LMUL, the specification requires support only for combinations where SEW ≤ LMUL × ELEN. Other combinations need not be supported, and receiving an unsupported `vtype` sets the `vtype.vill` bit. Of the 28 combinations, QEMU set vill on exactly the 6 that break this rule (e16·mf8, e32·mf8, e32·mf4, e64·mf8~mf2), and the VLMAX of the remaining 22 matched the formula above at all four VLENs.

## 4. `vtype` Encoding and the Three Forms of `vsetvli`

The bit layout of the `vtype` CSR is as follows.

```text
bit       name    meaning
XLEN-1    vill    1 if the setting is unsupported
XLEN-2:8  (rsvd)  must be 0
7         vma     mask policy (part 14)
6         vta     tail policy (part 14)
5:3       vsew    000 e8, 001 e16, 010 e32, 011 e64
2:0       vlmul   000 m1, 001 m2, 010 m4, 011 m8,
                  101 mf8, 110 mf4, 111 mf2
```

`vlmul` value 100 is reserved. Three configuration instructions write this value. All three use opcode OP-V (`1010111`) with funct3 `111`, and are distinguished by the top bits.

```text
vsetvli  rd, rs1, vtypei  [31]=0, [30:20]=vtypei (11 bits)
vsetivli rd, uimm, vtypei [31:30]=11, [29:20]=vtypei,
                          [19:15]=uimm(AVL 0~31)
vsetvl   rd, rs1, rs2     [31:25]=1000000, rs2=vtype
```

Let's encode `vsetvli t0, x0, e32, m1, ta, ma` by hand. vtypei is vma=1, vta=1, vsew=010, vlmul=000, so `1101 0000` = `0x0d0`. Append rs1=0, funct3=`111`, rd=5 (t0), and opcode=`1010111`.

```text
[31] [30:20]      rs1   f3  rd    opcode
 0   00011010000 00000 111 00101 1010111
= 0x0d0072d7
```

When I ran this instruction with VS turned off in section 7, the value QEMU left in mtval was exactly `0x0d0072d7`.

AVL (application vector length, the number of elements you want to process) is the value of register rs1. When the register number is x0, the meaning changes.

- rs1 = x0, rd ≠ x0: AVL is treated as infinite (~0). The result is vl = VLMAX. This is the idiom for "tell me the VLMAX of this setting."
- rs1 = x0, rd = x0: keep the current vl and change only `vtype`. Cases where VLMAX changes are reserved.

`vsetivli` gives AVL as a 5-bit immediate. It saves a register when the element count is small and fixed (e.g., a 4×4 matrix). The result of `vsetivli rd, 31, e8, mf8` was 2 at VLEN 128 and 16 at 1024. VLMAX is smaller than the AVL of 31, so VLMAX came out. `vsetvl` takes `vtype` from a register. It is used when the setting is decided at run time, such as restoring after a context switch. The 28-combination test in this exercise also looped with `vsetvl`.

## 5. How vl Is Determined from AVL

`vsetvli` looks at AVL and VLMAX, decides vl, and writes it to rd and the `vl` CSR. The specification's rules (31.6.3) are three.

$$
\text{vl} = \begin{cases}
\text{AVL} & \text{AVL} \le \text{VLMAX} \\
\text{implementation's choice in } \lceil \text{AVL}/2 \rceil \sim \text{VLMAX} & \text{VLMAX} < \text{AVL} < 2\,\text{VLMAX} \\
\text{VLMAX} & \text{AVL} \ge 2\,\text{VLMAX}
\end{cases}
$$

If the remaining elements are at most VLMAX, process exactly that many. If at least twice VLMAX remain, process a full VLMAX. In between, the implementation has freedom. When AVL is slightly more than VLMAX, using all of VLMAX now leaves only a few for the next iteration. Splitting in half instead evens out the work across the two iterations. The specification leaves this choice to the implementation. Also, the same AVL and `vtype` must always produce the same vl (deterministic).

These are the results of feeding AVL 0~40 at VLEN 128, e32·m1 (VLMAX=4).

| AVL | vl allowed by the spec | QEMU |
|---|---|---|
| 0~4 | AVL as is | 0~4 |
| 5 | 3~4 | 4 |
| 6 | 3~4 | 4 |
| 7 | 4 | 4 |
| 8 or more | 4 | 4 |

QEMU always chose vl = min(AVL, VLMAX). All 164 cases, 41 AVLs at each of four VLENs, matched this formula. Real hardware may choose differently, so code must always use the returned vl. Code that assumes "it shrinks by VLMAX each time" will break on another chip. If AVL=0 then vl=0, and a vector instruction with vl=0 touches no elements.

## 6. Strip Mining: Length-Agnostic Loops

Cutting a long array into chunks of vl and processing them is called strip mining. GCC 16.2 compiled a loop for `c[i] = a[i] + b[i]` written with intrinsics like this (the loop part).

```text
loop:
  vsetvli a5, a3, e32, m1, ta, ma  # vl = f(remaining n)
  vle32.v v1, (a1)       # vl elements from a
  vle32.v v2, (a2)       # vl elements from b
  vadd.vv v1, v1, v2
  vse32.v v1, (a4)       # vl elements to c
  slli    a6, a5, 2      # byte count = vl*4
  add     a1, a1, a6
  add     a2, a2, a6
  add     a4, a4, a6
  sub     a3, a3, a5     # n -= vl
  addi    a0, a0, 1      # count iterations
  bnez    a3, loop
```

VLEN appears nowhere in these 12 instructions. `vle32.v` is a unit-stride load: it reads vl consecutive 4-byte elements starting at the address and fills the register group from element 0. `vse32.v` does the reverse. The element width is the EEW written in the instruction (32 here), and the count follows `vl`. Other kinds of loads and stores come in part 14.

The `vsetvli` result in a5 sets both the pointer increment and the decrement of the remaining count. Whatever vl the hardware chooses, the loop ends correctly. With 100 elements, vl went like this in each iteration.

![The same loop adding 100 elements runs 25 times with vl=4 at VLEN 128, 13 times by 8 at 256, 7 times by 16 at 512, and 4 times by 32 at 1024. Only the last iteration has vl=4|xl](/img/01M6K7F34E024SNZYYF8JSZTCA/27fd34c2067c.webp)

In the last iteration, vl dropped to the remaining 4. With fixed-width SIMD, these 4 would have been handled in a scalar tail loop.

## 7. Hands-On: The Same Machine Code at Four VLENs

I built a single `vlen.c` once with `-march=rv64gcv -mabi=lp64d -O1` and ran the same ELF on QEMU 11.1.3 `virt` with `-cpu rv64,v=true,vlen=N,elen=64`, changing only VLEN (first 16 characters of the `.text` sha256: `b32fb1ef7eb4fe84`). `riscv_vector.h` is a header supplied by the compiler, so it compiled as is under `-ffreestanding` without newlib. The executed instruction count is the difference in `minstret` read before and after the function, with QEMU run under `-icount shift=0`. The machine was a Mac M1 Max and the compiler `riscv64-elf-gcc` 16.2.

```c
uint64_t vadd_m1(int32_t *c, const int32_t *a,
                 const int32_t *b, size_t n) {
  uint64_t it = 0;
  while (n) {
    size_t vl = __riscv_vsetvl_e32m1(n);
    vint32m1_t x = __riscv_vle32_v_i32m1(a, vl);
    vint32m1_t y = __riscv_vle32_v_i32m1(b, vl);
    __riscv_vse32_v_i32m1(c,
        __riscv_vadd_vv_i32m1(x, y, vl), vl);
    a += vl; b += vl; c += vl; n -= vl; it++;
  }
  return it;
}
```

`vadd_m4` just changes m1 to m4 in the names, and `vcopy` copies 1,000 bytes with e8·m8. The scalar baseline is the same addition compiled with auto-vectorization turned off (8 instructions per element).

| VLEN | m1 iterations (instructions) | m4 iterations (instructions) | 1000B copy iterations |
|---|---|---|---|
| 128 | 25 (313) | 7 (97) | 8 |
| 256 | 13 (169) | 4 (61) | 4 |
| 512 | 7 (97) | 2 (37) | 2 |
| 1024 | 4 (61) | 1 (25) | 1 |

The m1 instruction count is always 12 × iterations + 13. The 12 per iteration are the loop above, and the 13 are the call, return, and code before and after the loop. The scalar loop was 813 instructions regardless of VLEN. Each doubling of VLEN nearly halves the iterations, and LMUL=4 groups four registers to quadruple VLMAX at the same VLEN. Since m4 uses 8 registers (two groups for a and b), m1 may be better in large loops that run short on registers.

![Executed instruction count for a 100-element add: scalar 813; vector m1 drops from 313 at VLEN 128 to 61 at 1024, and m4 from 97 to 25|xl](/img/01M6K7F34E024SNZYYF8JSZTCA/f132f520fc46.webp)

Instruction count here is not performance. QEMU counts one vector instruction as one instruction. How many cycles a single `vadd.vv` takes on real hardware depends on the number of lanes. With VLEN=512 and one lane, 16 elements might take 16 cycles (an estimate). Still, fetching and decoding fewer instructions is a gain that does not depend on lane count.

The output was checked against a Python reference model.

| Test | Count | Matched |
|---|---|---|
| VLMAX and vill (28 combinations × 4 VLENs) | 112 | 112 |
| AVL → vl (41 × 4 VLENs) | 164 | 164 |
| Iteration counts and vl sequences | 16 | 16 |
| 100 add results, 1000B copy | 8 | 8 |

I also checked four illegal cases with a trap handler. The handler records mcause and mtval, advances mepc by 4, and returns to the next instruction.

| Case | Result |
|---|---|
| `vsetvli` with VS=Off | mcause 2, mtval `0x0d0072d7` |
| `vadd.vv v1,v2,v4` under m2 | mcause 2 (v1 is odd) |
| `vsetvl` with e64·mf8 | vl 0, vtype `0x8000…0` (vill) |
| `vadd.vv` in the vill state | mcause 2 |

The second row is interesting. GNU as encoded `vadd.vv v1,v2,v4` as `0x022200d7` without any warning, because the assembler does not know the LMUL at run time. The same machine code is legal under m1 and illegal under m2. In the same place, `vadd.vv v2,v4,v6` (all even) did not trap. Running `vsetvl` with reserved bit 8 of `vtype` set to 1 also set vill.

## 8. Design Checklist: `vtype`, `vl`, and the Decoder

| Item | Rule |
|---|---|
| VS=Off | Vector instructions and vector CSR accesses (including reads of `vl`, `vtype`, `vlenb`) are all illegal instructions |
| VS update | Set VS to Dirty (3) when a vector register or vector CSR changes |
| Writing `vl`/`vtype` | Writing with CSR instructions is illegal (read-only). Only `vset*` and fault-only-first loads write them |
| vill check | On a reserved vsew/vlmul value, nonzero reserved bits, or an unsupported fractional LMUL: vill=1, other bits 0, vl=0 |
| After vill | Vector instructions that depend on `vtype` are illegal. `vset*` and whole-register loads, stores, and moves can still execute |
| AVL rule | The range VLMAX < AVL < 2·VLMAX is the implementation's choice. A common choice is min(AVL, VLMAX) (QEMU) |
| x0 special case | If rs1=x0, AVL=~0 (rd≠x0) or keep vl (rd=x0). Decide by register number before reading the rs1 value |
| Group alignment | A register number that is not a multiple of LMUL is illegal. The decoder cannot decide alone; it must also look at `vtype.vlmul` |
| Reset | Recommended: `vill`=1, all else 0, vl=0 (not mandated by the spec) |
| `vlenb` | The constant VLEN/8. A hardwired constant in hardware |

The group alignment row is what makes a vector decoder different from a scalar one. A scalar decoder decides legality from the 32 bits alone, but for vectors the current `vtype` (LMUL, and vill) becomes a decode input. In a pipelined core, a `vsetvli` ahead in the pipeline may not have written `vtype` yet. Common choices are to rename `vtype` like a register, or to stall vector instructions behind a `vsetvli` (an estimate). Part 15 on the decoder revisits this.

## Summary

- The V extension adds 32 vector registers of VLEN bits and 7 CSRs including `vl` and `vtype`. VLEN is chosen by the implementation (at least 128 for V) and does not appear in the code. `mstatus.VS` must be turned on to use it.
- VLMAX = VLEN / SEW × LMUL. LMUL ranges from 1/8 to 8, and a group starts only at a register number that is a multiple of LMUL. QEMU set vill on the 6 combinations where SEW > LMUL × ELEN.
- `vsetvli` sets vl from AVL and VLMAX. Between VLMAX and 2·VLMAX it is the implementation's choice, so code must trust only the returned vl. QEMU gave min(AVL, VLMAX) in all 164 cases.
- The same 12-instruction machine-code loop ran 25, 13, 7, and 4 times at VLEN 128, 256, 512, and 1024. It executed 313, 169, 97, and 61 instructions, versus 813 for the scalar loop.

**Next post:** Part 14 covers the other half of the V extension. On QEMU it runs and checks conditional operations that use `v0` as a mask; unit-stride, strided, indexed (gather/scatter), and segment loads and stores; a `strlen` written with fault-only-first loads that do not stop at the end of a page; reductions such as `vredsum` and permutations such as `vslideup`, `vrgather`, and `vcompress`; and the freedom that tail- and mask-agnostic policies give the hardware.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 31 "V" Standard Extension for Vector Operations, Version 1.0: 31.2 Implementation-defined Constant Parameters (VLEN, ELEN), 31.3 Vector Extension Programmer's Model (`mstatus.VS`, `vtype`, `vl`, `vlenb`), 31.4 Mapping of Vector Elements to Vector Register State, 31.6 Configuration-Setting Instructions (31.6.3 Constraints on Setting vl), 31.7.4 Vector Unit-Stride Instructions, 31.18 Standard Vector Extensions (Zvl, Zve). https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. 3.1.6 `mstatus` (VS field).
- RISC-V Vector C Intrinsics specification v1.0 (the `__riscv_`-prefixed API). https://github.com/riscv-non-isa/rvv-intrinsic-doc
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 8 (RV32V). It is based on a pre-ratification draft, so instruction names and encodings differ from 1.0. I used it only for its conceptual explanation of the vector length register.
- QEMU RISC-V documentation (`virt` board, CPU properties `v`, `vlen`, `elen`). https://www.qemu.org/docs/master/system/target-riscv.html
