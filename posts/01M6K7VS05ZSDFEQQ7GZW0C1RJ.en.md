---
id: "01M6K7VS05ZSDFEQQ7GZW0C1RJ"
title_en: "Small Extensions: Zicsr, Zifencei, Zicntr·Zihpm, B (Zba·Zbb·Zbs), Zicond, Zicbom"
summary_en: "Part 12 of the RISC-V series. I check in QEMU the rule by which CSR instructions decide whether to read or write based on whether rd and rs1 are x0, and run the B and Zicond instructions over 2,772 combinations on RV32 and 3,720 on RV64, all matching a reference model. Enabling B shrank the code by 6.6–9.0%."
---
:::note
**Key points in three lines**
- The 6 CSR instructions of Zicsr decide whether to read or write based on "which field is `x0`." `csrrw` does not read when rd=`x0`, and `csrrs`/`csrrc` do not write when rs1=`x0`. They look at the field, not the value. In QEMU, `csrrs` on a read-only CSR raised an illegal instruction exception even when the rs1 register held 0.
- I ran the bit-manipulation instructions of the B extension (Zba, Zbb, Zbs) in QEMU over 2,772 combinations on RV32 and 3,720 on RV64, and all of them matched a Python reference model. The same C code got 6.6–9.0% smaller with B enabled.
- Zicond's `czero.eqz` and `czero.nez` build a branchless select out of three instructions. The default CPU in QEMU 11.1.3 has Zba, Zbb, Zbs, and Zicbom enabled but Zicond disabled, so `-cpu rv32,zicond=true` was needed.
:::

The previous parts covered the base integer ISA and the M, A, F, D, and C extensions. Part 12 groups together the small extensions whose names start with Z (Zicsr, Zifencei, Zicntr/Zihpm, B, Zicond, Zicbom). Each one is small, but Zicsr's read/write rules lead into the CSR file design in Part 24, and Zifencei into Part 22.

The experiments used the QEMU 11.1.3 `virt` board (Mac M1 Max) and `riscv64-elf-gcc` 16.2, and I checked for illegal instructions with a trap handler that records `mcause` and skips the instruction.

## 1. Zicsr: The 6 CSR Instructions

A CSR (Control and Status Register) is a special register selected by a 12-bit address. Unlike general-purpose registers, CSRs are not accessed with loads and stores; they are read and written only with dedicated instructions. There are three operations, each with a register form and an immediate form, for 6 instructions in total (SYSTEM opcode, I-type, with the CSR address in the immediate field).

| Instruction | Operation (t = old CSR value) | Immediate form |
|---|---|---|
| `csrrw rd, csr, rs1` | rd ← t, csr ← rs1 | `csrrwi` |
| `csrrs rd, csr, rs1` | rd ← t, csr ← t \| rs1 | `csrrsi` |
| `csrrc rd, csr, rs1` | rd ← t, csr ← t & ~rs1 | `csrrci` |

The immediate forms use the 5 bits of the rs1 field, zero-extended (uimm, 0–31). The read and the write happen atomically within one instruction, so you can, for example, set the interrupt enable bit and get the old value back with no window for anything to sneak in.

I traced this by hand in QEMU with `mscratch` (a scratch CSR for M-mode).

| Step | Instruction | rd (old value) | CSR after |
|---|---|---|---|
| 1 | `csrw` `0x12345678` | | `12345678` |
| 2 | `csrrw` `0xaaaa5555` | `12345678` | `aaaa5555` |
| 3 | `csrrs` `0x0000000f` | `aaaa5555` | `aaaa555f` |
| 4 | `csrrc` `0x0000ff00` | `aaaa555f` | `aaaa005f` |
| 5 | `csrrsi` 0x10, `csrrci` 0x5 | | `aaaa005a` |

In step 5, setting `0x10` changes nothing because that bit was already 1, and clearing `0x5` turns `f` → `a`. The common pseudo-instructions are built from these 6. `csrr rd, csr` = `csrrs rd, csr, x0`, `csrw csr, rs` = `csrrw x0, csr, rs`, and `csrs`/`csrc` are `csrrs`/`csrrc` with rd=`x0`.

## 2. The x0 Rules: The Most Important Table for Design

Some CSRs change something merely by being read, and writes can have side effects too. So the specification defines when a CSR is "not read" and when it is "not written."

![CSR instructions decide whether to read or write based on whether a register field is x0; executing csrrs on the read-only mhartid with an rs1 value of 0 raised an illegal instruction exception in QEMU|full](/img/01M6K7VS05ZSDFEQQ7GZW0C1RJ/8b4da0ab8ed1.webp)

- `csrrw`/`csrrwi`: if rd=`x0`, the CSR is **not read**, and there are no read side effects. The write always happens.
- `csrrs`/`csrrc`: if rs1=`x0`, the CSR is **not written**. There are no write side effects, and no exception even for a read-only CSR. The read always happens (even if rd=`x0`).
- `csrrsi`/`csrrci`: if uimm=0, there is no write.

The key point is that it looks at **the field, not the value**. If rs1 is `x5` and its value is 0, no bit changes, but per the specification it is an "attempted write." I checked this with the read-only CSR `mhartid` (core number, address `0xF14`).

| Instruction | QEMU result |
|---|---|
| `csrrs rd, mhartid, x0` | 0 (OK) |
| `csrrs rd, mhartid, t` (t=0) | Illegal instruction (`mcause`=2) |
| `csrrsi rd, mhartid, 0` | 0 (OK) |
| `csrrsi rd, mhartid, 1` | Illegal instruction |
| `csrrc rd, mhartid, t` (t=0) | Illegal instruction |
| `csrrw x0, mhartid, x0` | Illegal instruction |
| `csrrwi x0, mhartid, 0` | Illegal instruction |

The second row is where people often get it wrong. Whether a write happens is decided in the decode stage from the instruction bits alone (the rs1 and rd fields), so the permission check can finish before the register values are read.

Read-only status is also encoded in the address bits. The privileged specification says that if bits [11:10] of a CSR address are `11`, the CSR is read-only, and bits [9:8] give the minimum privilege level needed to access it. For `mhartid` `0xF14`, [11:10]=11 (read-only) and [9:8]=11 (M-mode). Reading a nonexistent CSR (`0x8ff`) also raised an illegal instruction exception.

I also looked at one WARL (Write Any, Read Legal) field. The low 2 bits of `mtvec`, MODE, define only 0 (direct) and 1 (vectored); 2 and 3 are reserved. Writing `0x80001003` or `0x80001002` in QEMU ignored the write entirely and left the old value `0x80000030`, while `0x80001001` went in as written. What to keep when a reserved value is written is up to the implementation (Parts 23 and 24).

When a CSR instruction reads a CSR that changes as a side effect of execution (e.g., `instret`), it reads the value from **before** that instruction executes, and when it writes one, the write happens "instead of the increment."

## 3. Zifencei: Synchronizing the Instruction Side

`fence.i` guarantees that later instruction fetches on this hart (hardware thread) see the memory writes it made earlier. Self-modifying code and JIT compilers need it, because a separate instruction cache may still hold the old code. It was originally part of the base ISA and was split out because its implementation cost varies widely between cores. It has no effect on other harts. Implementation is covered in Part 22 (in QEMU it executes without a trap).

## 4. Zicntr and Zihpm: Counters

Zicntr defines three read-only 64-bit counters.

| CSR (address) | Pseudo-instruction | Counts |
|---|---|---|
| `cycle` (`0xC00`) | `rdcycle` | Clock cycles |
| `time` (`0xC01`) | `rdtime` | Real time (fixed frequency) |
| `instret` (`0xC02`) | `rdinstret` | Retired instructions |

Zihpm adds 29 hardware performance counters, `hpmcounter3`–`hpmcounter31` (`0xC03`–`0xC1F`). What to count (cache misses, branch mispredictions, etc.) is chosen with M-mode's `mhpmevent`, and the event types are implementation-defined. These unprivileged CSRs are read-only shadows of the M-mode counters (`mcycle`, `minstret`, etc.), and whether lower modes can read them is controlled by `mcounteren` and `scounteren`.

On RV32 a 64-bit counter cannot be read in one go, so the upper 32 bits are read from a separate CSR such as `cycleh` (`0xC80`). To avoid the moment when the lower half wraps, software reads high-low-high and repeats until the two high reads match.

```asm
1:  rdinstreth a1        # high
    rdinstret  a0        # low
    rdinstreth a2        # high again
    bne a1, a2, 1b       # retry if it wrapped in between
```

I read the counters before and after a 2,000-instruction loop (`addi`/`bnez` × 1000) in QEMU, and the results were not what I expected.

| Measurement | Default run | `-icount shift=0` |
|---|---|---|
| `instret` delta across the loop | 16,000 | 2,004 |
| `cycle` delta across the loop | 16,000 | 2,004 |
| Two back-to-back `rdinstret` | 4,000 | 1 |
| Delta with `mcountinhibit.IR`=1 | 0 | 0 |

In the default run, even two back-to-back reads differed by 4,000, and the values changed from run to run. This seems to be because QEMU does not count instructions one by one but derives the counter value from host time (my guess). Running with `-icount shift=0` (one instruction = one unit of virtual time) gave exactly 2,004: the `rdtime`, `li`, the 2,000 loop instructions, `rdcycle`, and the second `rdinstret` itself, all between the two `rdinstret`s, add up to 2,004. Back-to-back reads give 1. This matches the rule from Section 2 that a read returns "the value before the reading instruction executes." In this mode `cycle` and `instret` were equal, because QEMU models one cycle per instruction.

Setting the IR bit (bit 2) of `mcountinhibit` stopped `instret`. Right after writing 0 to `minstret`, `rdinstret` returned 1 in icount mode. Under the specification's "write instead of increment" rule, the writing instruction itself is not counted, so I expected 0, but QEMU returned 1. QEMU appears to add that instruction's increment after the write (my guess; not confirmed).

`time` is a wall clock that increases at a constant rate even when the clock frequency changes, and usually exposes the memory-mapped timer `mtime`. The hardware does not have to provide it directly; M-mode may emulate it with a trap.

## 5. The B Extension: Zba, Zbb, Zbs

The B extension (version 1.0) bundles three extensions ratified in 2021: Zba, Zbb, and Zbs. It reduces bit-manipulation patterns that compilers often generate from C code to single instructions.

| Extension | Instructions | Use |
|---|---|---|
| Zba (address) | `sh1add` `sh2add` `sh3add` | Array address a + i×2/4/8 |
| Zbb (basic) | `andn` `orn` `xnor` | Mask operations |
| Zbb | `clz` `ctz` `cpop` | Count leading/trailing zeros, count ones |
| Zbb | `min` `max` `minu` `maxu` | Branchless min/max |
| Zbb | `sext.b` `sext.h` `zext.h` | Sign/zero extension |
| Zbb | `rol` `ror` `rori` | Rotate |
| Zbb | `rev8` `orc.b` | Byte reverse, per-byte zero test |
| Zbs (single bit) | `bset` `bclr` `binv` `bext` (+i) | Set/clear/invert/extract one bit |

RV64 adds 32-bit forms (`clzw`, `cpopw`, `rolw`, etc.) and `add.uw`, `sh1add.uw`–`sh3add.uw`, and `slli.uw`, which zero-extend the low 32 bits of rs1 (for `unsigned int` indices).

Let's trace just two by hand. `orc.b` turns each byte into `0xff` if it is nonzero and `0x00` if it is zero. Applied to `0xff00ff01`, the bytes `ff`, `00`, `ff`, `01` become `ff`, `00`, `ff`, `ff`, giving `0xff00ffff`. `strlen` uses it to find a zero byte while reading 4 bytes at a time. `rev8` reverses the byte order (on RV32, `0xff00ff01` → `0x01ff00ff`), which is used when reading big-endian network data. Zbs's `bset a0, a1, a2` sets bit (a2 mod XLEN) of a1. Like shifts, it looks only at the low 5 bits of the bit index (6 bits on RV64).

`clz` and `ctz` return XLEN when the input is 0. In C, `__builtin_clz(0)` is undefined behavior, so the compiler inserts a separate zero check, but `clz` has a defined answer for 0, so `x ? __builtin_clz(x) : 32` compiled to a single `clz` (table in Section 6).

## 6. Experiment: Checking QEMU Against a Reference Model, and Code Size

First I checked what QEMU supports. On the default `rv32` CPU, the Zba, Zbb, and Zbs instructions and `cbo.*` executed without a trap, but `czero.eqz` was an illegal instruction (`mcause`=2). With `-cpu rv32,zicond=true` it executed. `misa` was `0x401411ad` (A, C, D, F, H, I, M, S, U).

The test program used 12 values in total: boundary values (0, 1, −1, only the top bit set, all ones below it, `0x80`, `0xff00ff01`) plus 5 xorshift random values with a fixed seed. Register-register instructions ran on every pair (144), and unary and immediate instructions ran on the 12 values. The results were checked against a reference model I wrote in Python (a one-line lambda per instruction).

```python
'rol':   lambda a, b: rol(a, b & sh, X),
'clz':   lambda a, _: X - a.bit_length(),
'orc.b': lambda a, _: int.from_bytes(bytes(
    0xff if x else 0 for x in a.to_bytes(X//8, 'little')),
    'little'),
'czero.eqz': lambda a, b: 0 if b == 0 else a,
```

| Target | Instruction kinds | Executions | Matches |
|---|---|---|---|
| RV32 (`rv32,zicond=true`) | 31 | 2,772 | 2,772 |
| RV64 (`rv64,zicond=true`) | 42 | 3,720 | 3,720 |
| Zicsr rules (table in Section 2) | | 7 | 7 (same as the spec's expectation) |

For code size, I compiled the test code `bench.c` from Part 11 and `kernels.c` from Part 1 with the C extension enabled, with and without B.

![Enabling Zba, Zbb, and Zbs on the same code took bench.c on RV32 from 1,546 to 1,444 bytes (-6.6%) and kernels.c on RV64 from 468 to 426 bytes (-9.0%)|xl](/img/01M6K7VS05ZSDFEQQ7GZW0C1RJ/1132e6f2dcac.webp)

| Code | Instruction count (without B → with B) | Most-used B instructions |
|---|---|---|
| bench.c RV32 | 523 → 492 | `rori` 10, `sh2add` 8 |
| bench.c RV64 | 525 → 493 | `roriw` 10, `sh2add` 6 |
| kernels.c RV32 | 167 → 144 | `sh2add` 11, `cpop` 1 |
| kernels.c RV64 | 175 → 151 | `sh2add` 10, `cpopw` 1 |

The 10 `rori`s are SHA-256's rotate operations (the `ROR` macro). Without B, one rotate takes three instructions: `srli`, `slli`, `or`. `sh2add` is `int32` array indexing, and `cpop` is Part 1's `popcount_sum` turning from a library call into a single instruction.

Looking at one function at a time makes the difference clearer (RV32, `-O2`).

| C code | Without B/Zicond | With |
|---|---|---|
| `p[i]` (uint32) | `slli`+`add`+`lw` | `sh2add`+`lw` |
| `x ? clz(x) : 32` | Branch + library call | `clz` |
| `x & ~(1u << n)` | `li`+`sll`+`not`+`and` | `bclr` |
| `bswap32(x)` | Library call | `rev8` |
| `c ? a : 0` | `beqz`+`mv` | `czero.eqz` |
| `c ? a : b` | `beqz`+`mv`+`mv` | `czero.nez`+`czero.eqz`+`add` |

## 7. Zicond: Branchless Select

Zicond (version 1.0) has only two instructions.

- `czero.eqz rd, rs1, rs2`: if rs2 is 0, rd ← 0; otherwise rd ← rs1.
- `czero.nez rd, rs1, rs2`: if rs2 is nonzero, rd ← 0; otherwise rd ← rs1.

Each can only do "zero if the condition is false," but combined they give a general select. `c ? a : b` is the sum of `czero.eqz t0, a, c` (a if c is true, otherwise 0) and `czero.nez t1, b, c` (b if c is false, otherwise 0). One of the two is always 0, so they can be combined with `add` or `or`. GCC emitted exactly these three instructions, as in the table above.

Why not make "rd ← c ? a : b" a single instruction, like x86's `cmov`? That would require reading three registers (c, a, b). RISC-V integer instructions have at most two source registers, so to avoid adding register file read ports, the operation was split into two two-operand instructions. The hardware only needs one extra "0 or rs1" selector at the ALU output.

Branches that go either way at random depending on the data are often mispredicted, so always executing two or three instructions can be faster.

## 8. Zicbom: Cache Block Management

Among the CMO (Cache Management Operation) extensions, Zicbom defines three instructions that manage data per cache block. The address is given in rs1 alone, and the operation applies to the whole cache block containing that address.

| Instruction | Operation | Use |
|---|---|---|
| `cbo.clean` | Writes modified data back to memory (keeps it in the cache) | Before DMA reads memory |
| `cbo.flush` | Writes it back and evicts it from the cache | When handing over a shared buffer |
| `cbo.inval` | Evicts it from the cache without writing back | After DMA writes to memory |

These are needed when exchanging data with devices such as DMA engines whose coherence the hardware does not maintain. Because `cbo.inval` can discard data that has not been written back, in modes other than M-mode the CBIE field of `menvcfg`/`senvcfg` decides whether it is allowed; if not, it can be made an illegal instruction or made to behave like `cbo.flush`. The block size is not defined by the ISA and is conveyed by external information such as the device tree. In QEMU all three instructions executed without a trap (it is an emulator without caches, so their effect cannot be observed).

## 9. Design Checklist: Small Extensions

| Item | Rule |
|---|---|
| CSR read or not | `csrrw(i)` does not read when rd=`x0`. `csrrs/c(i)` always read |
| CSR write or not | `csrrs/c` do not write when the rs1 field=`x0`, `csrrs/ci` when uimm=0. Never decided by register value |
| Read-only CSR | Address [11:10]=11. An attempted write is an illegal instruction; a read without a write is fine |
| Privilege | Illegal instruction if address [9:8] > current mode. Nonexistent CSRs are also illegal |
| WARL | What to keep on a reserved value is an implementation choice (QEMU `mtvec`: write ignored) |
| `instret` writes | The writing instruction writes instead of incrementing. A reading instruction sees the pre-execution value |
| RV32 counters | Upper 32 bits in the `…h` CSRs. Software reads high-low-high |
| `time` | Fixed frequency. May be emulated with a trap instead of provided as a CSR |
| `fence.i` | Makes this hart's earlier writes visible to its later instruction fetches. No effect on other harts |
| B shift amount | `rol`, `ror`, `bset`, etc. use only the low log2(XLEN) bits of rs2 |
| `clz`/`ctz` | XLEN for input 0 (32 for `clzw`/`ctzw`) |
| `rev8`/`orc.b` | Per byte across the full XLEN. `rev8` has different encodings on RV32 and RV64 |
| Zicond | 2 source registers. A "0 or rs1" selector at the ALU output |
| `cbo.inval` | For lower modes, CBIE in `menvcfg`/`senvcfg` selects allowed, illegal, or flush |

## Summary

- Zicsr's 6 instructions decide whether to read or write based on whether the rd and rs1 fields are `x0`. In QEMU, `csrrs` with a register holding 0 still raised an illegal instruction exception on a read-only CSR.
- The counters `cycle`, `time`, and `instret` are read-only shadow CSRs, and RV32 reads the upper half separately. Without `-icount`, QEMU did not count instructions accurately; with `-icount shift=0`, a 2,004-instruction span came out at exactly 2,004.
- Running the B extension instructions over 2,772 combinations on RV32 and 3,720 on RV64 matched the reference model in every case. The same code got 6.6–9.0% smaller, mostly from rotates, array addressing, and bit counting.
- Zicond builds a branchless select in three instructions while keeping the two-source-register rule, and Zicbom provides cache block instructions for exchanging data with non-coherent devices.

**Next post:** Part 13 starts the vector extension. It looks at what the V extension's register length VLEN, element size SEW, and register grouping LMUL are, and how setting the number of elements processed at once with `vsetvli` produces code that is not tied to a vector length.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 6 Zifencei, Chapter 7 Zicsr (7.1 CSR instructions), Chapter 8 counters (Zicntr, Zihpm), Chapter 12 Zicond, Chapter 19 CMO (Zicbom), Chapter 28 B extension (Zba, Zbb, Zbs). https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. 2.1 CSR address conventions, 3.1.7 `mtvec`, 3.1.10–3.1.12 counters, `mcounteren`, `mcountinhibit`, CBIE in `menvcfg`.
- RISC-V Bit-Manipulation ISA-extensions, Version 1.0.0, 2021. https://github.com/riscv/riscv-bitmanip
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 2 (CSR instructions), Chapter 10 (privileged architecture)
- QEMU documentation: `-icount`. https://www.qemu.org/docs/master/system/qemu-manpages.html
