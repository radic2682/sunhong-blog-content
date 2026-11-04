---
id: "01M6K1PY0PQGCWC5TMFEAYF4DE"
title_en: "RISC-V Overview: The Spec, ISA Names, and What Extensions Do to Code"
summary_en: "Part 1 of the RISC-V series. Covers the design principles of a modular ISA, how the specification is split into two volumes (Unprivileged and Privileged), and how to read names like rv64gc, then compiles the same C functions under different extension combinations and measures the code size dropping from 972 bytes to 402 bytes."
---
:::note
**Key points in three lines**
- RISC-V is a modular instruction set: one small base integer ISA plus only the extensions you need. The specification is split into two volumes, Unprivileged (Volume I) and Privileged (Volume II), plus specifications outside the ISA.
- A name like `rv64gc` literally spells out "which base ISA, with which extensions," so it becomes the first spec sheet you settle on when designing a core.
- Compiling the same 7 C functions while changing only the extension combination shrank the code from 972 bytes for `rv32i` → 502 bytes for `rv32imc` → 402 bytes for `rv32imfc_zba_zbb`. The extensions you choose change the code and the hardware together.
:::

This is part 1 of the RISC-V series, the first set of posts in the Digital Design category. The goal of the series is to cover the RISC-V specification thoroughly, so that it can sit open beside me as a reference when I later design a core in RTL. Before getting into individual instructions, part 1 draws the big map: where RISC-V came from, how the specification is divided, how to read names like `rv64gc`, and what such a name determines in a design.

## 1. What Is RISC-V

An instruction set architecture (ISA) is the contract between software and hardware. It defines the machine as programs see it, as in "when this 32-bit value arrives, add two registers and put the result in a third." As long as the same ISA is honored, the same program runs on anything from a tiny microcontroller to a large server core. How the inside is built (number of pipeline stages, cache size) is the job of the microarchitecture.

RISC-V is an ISA that started at UC Berkeley in 2010. Krste Asanović, Yunsup Lee, and Andrew Waterman created it together with David Patterson for research and education. The V in the name means it is the fifth RISC design from Berkeley. The RISC-V Foundation was formed in 2015 and moved to Switzerland as RISC-V International in 2020.

With x86 and ARM already around, why was a new ISA needed? The reasons boil down to three.

- **It is open.** Anyone can read the specification for free and implement it without permission or license fees. When a lab builds a chip or a company designs its own core, there is no one to negotiate with.
- **It is small and simple.** The base integer ISA has about 40 instructions. Unlike ISAs that have piled up instructions over decades for compatibility, it was cleanly redesigned from scratch.
- **It is modular.** The small base ISA is frozen and never changes. Features such as multiplication, floating point, and vectors are defined separately as extensions, and you pick only the ones you need.

The third point is what most distinguishes RISC-V from other ISAs, and the rest of this post is all about it.

## 2. Design Principles: What Were They Trying to Save

The people who created RISC-V named cost, simplicity, performance, separation of architecture from implementation, room for growth, program size, and ease of programming, compiling, and linking as the criteria for evaluating an ISA. These criteria show up as concrete decisions throughout the specification. Picking only the ones you feel when designing hardware:

| Decision | Reason | Effect felt in the design |
|---|---|---|
| Register numbers (rs1, rs2, rd) are in the same bit positions in every format | Register file reads can start before the instruction type is known | Decode and register read in parallel in one stage |
| The immediate's sign bit is always bit 31 | Sign extension starts at the same time as format detection | Shorter immediate generator |
| `x0` is always 0 | No need for separate move, compare, or NOP instructions | Writes are discarded, reads return 0; simpler hardware |
| No condition code (flag) register | Compare and branch in one instruction | No flag dependencies to track |
| No branch delay slot | Avoid being tied to a particular pipeline structure | Free choice of pipeline depth |
| Only loads and stores access memory | Arithmetic instructions never touch memory | Simpler pipeline |

The fifth row is a good example of "separation of architecture from implementation." The old MIPS had a delay slot that unconditionally executed the one instruction after a branch. It was a win in a 5-stage pipeline, but became a burden in cores with a different number of stages. RISC-V kept to the principle of not putting mechanisms that favor a particular implementation into the ISA.

## 3. How the Specification Is Organized

![The RISC-V specification is split into two volumes, Unprivileged and Privileged, plus specifications outside the ISA such as ABI, SBI, and Debug; a single chip is a combination chosen from these|full](/img/01M6K1PY0PQGCWC5TMFEAYF4DE/ec6b27b3b50f.webp)

The RISC-V specification comes in three big chunks (this series is based on the April 2024 edition, document date 20240411).

**Volume I, the Unprivileged specification.** This is the part application programs see. It contains the base integer ISAs, the standard extensions, and the memory model.

- You pick exactly one base integer ISA. There are `RV32I` (32 registers, 32-bit), `RV64I` (64-bit), and the embedded `RV32E` and `RV64E` with the register count cut to 16. `RV128I` has not been finalized yet.
- Standard extensions are chosen and added. There are single-letter extensions (M, A, F, D, Q, C, B, V, etc.) and multi-letter extensions starting with `Z` (Zicsr, Zifencei, Zicond, etc.).
- The memory model RVWMO defines how the order of memory accesses appears when multiple cores (harts) access memory.

**Volume II, the Privileged specification.** This is the part the OS, firmware, and hypervisor see. It contains the privilege modes (M, S, U), CSRs (control and status registers), traps, physical memory protection, virtual memory, and the hypervisor extension. Extensions starting with `S` (Sstc, Svpbmt, etc.) belong here.

**Specifications outside the ISA.** These are not instructions, but agreements needed to build a real system. They include the psABI, which defines the calling convention and ELF format; SBI, the interface between the OS and firmware; the Debug specification; interrupt controllers (ACLINT, PLIC, AIA); and the Trace specification.

Each specification carries a status. Draft means it is still changing, frozen means it is in public review, and ratified means it has become an official standard. **When designing, use only ratified specifications as the baseline.** Ratified content comes with a promise that it will not change in ways that break compatibility.

:::info
**Hart** is short for hardware thread, the unit that independently fetches and executes instructions. One core may be one hart, or a simultaneous multithreading (SMT) core may have several harts. The specification uses this term instead of "core."
:::

## 4. Reading the Names

![rv64gc reads as rv + 64 + g + c, and the canonical name GCC 16 records in the object file is rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0 …|full](/img/01M6K1PY0PQGCWC5TMFEAYF4DE/4438851d53b0.webp)

RISC-V cores and compiler options carry names like `rv32imc` and `rv64gc`. Such a name literally spells out the ISA that the core supports. The reading rules are simple.

1. After `rv`, write XLEN (32 or 64).
2. Write the base ISA: `i`, `e`, or the bundle `g`.
3. Append single-letter extensions in the defined order (`imafdqc…`).
4. Append multi-letter extensions separated by underscores. `Z…` comes first, then `S…`, and vendor-defined `X…` extensions go last.
5. A version may follow an extension. `2p1` means version 2.1.

`g` is the "general" bundle: the five `imafd` plus `zicsr` and `zifencei`. CSR instructions and `FENCE.I` were originally part of base I, but were split out into the `Zicsr` and `Zifencei` extensions in the 2019 specification. So **a core that supports only `rv32i` has no CSR instructions.** Since taking interrupts requires CSRs, real cores almost always include `zicsr` as well.

You can check directly how the compiler reads a name. When GCC builds an object file, the expanded name is recorded in an attribute called `Tag_RISCV_arch` (view it with `riscv64-elf-readelf -A`). Below are the values GCC 16.2 actually recorded.

```text
-march=rv32i    →  rv32i2p1
-march=rv32imc  →  rv32i2p1_m2p0_c2p0_zmmul1p0_zca1p0
-march=rv64gc   →  rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0
                   _zicsr2p0_zifencei2p0_zmmul1p0
                   _zaamo1p0_zalrsc1p0_zca1p0_zcd1p0
```

Two things stand out. First, `g` really does expand to `imafd_zicsr_zifencei`. Second, large extensions are split into smaller pieces that are listed alongside them. `zmmul` is just the multiplication part of M; A splits into `zaamo` (atomic read-modify-write) and `zalrsc` (LR/SC); and C splits into `zca` (integer compressed instructions) and `zcd` (compressed loads and stores for D). They were split so that a small core that wants only a multiplier without a divider can say it supports just `zmmul`.

I also tested how strict the naming rules are. GCC rejected `rv32mi`, which lacks a base ISA at the front, with `first ISA subset must be 'e', 'i' or 'g'`. `rv32icm`, with single-letter extensions out of order, was accepted by GCC 16, which fixed it to the canonical order. Even if the tools are lenient, it is best to use the canonical order in documentation and hardware.

### Profiles

As extensions grew to dozens, a new problem appeared. For a Linux distribution to claim it "runs on any RISC-V chip," it cannot accommodate every chip's different combination of extensions. The answer is the **profile**. A profile is a list that says "a core of this class must have these extensions." For application processors there are `RVA20`, `RVA22`, and `RVA23`, and RVA23, ratified in 2024, made vectors (V) mandatory. Profiles matter little for microcontrollers, but if you are designing a core that runs Linux, the target profile effectively becomes the spec sheet.

## 5. Hands-On: How Much Do Extensions Change the Code

I measured what a single letter in the name actually changes. I wrote 7 C functions of different character and compiled them with only `-march` changed, leaving the source untouched.

| Function | What it does | Extension that should affect it |
|---|---|---|
| `crc32` | Bit operations and branches | None (baseline) |
| `matmul` | Integer matrix multiply | M (multiply) |
| `utoa10` | Integer to decimal string | M (divide, remainder) |
| `isort` | Insertion sort | None |
| `popcount_sum` | Count 1 bits | Zbb from B (`cpop`) |
| `index_sum` | Sum `base[idx[i]]` | Zba from B (`sh2add`) |
| `dot` | float dot product | F |

The machine is a Mac M1 Max and the compiler is Homebrew's `riscv64-elf-gcc` 16.2.0. Everything was built as object files only with `-O2 -ffreestanding -c`, and I counted symbol sizes, instruction counts, and library calls. The core of it is these few lines.

```python
CFGS = [('rv32i', 'ilp32'), ('rv32im', 'ilp32'),
        ('rv32imc', 'ilp32'), ('rv32imfc', 'ilp32f'),
        ('rv32imfc_zba_zbb', 'ilp32f'),
        ('rv64imafdc', 'lp64d'), ('rv64imafdc_zba_zbb', 'lp64d')]
run = lambda *a: subprocess.run(a, capture_output=True,
                                text=True, check=True).stdout
for march, mabi in CFGS:
    obj = f'/tmp/k-{march}.o'
    run('riscv64-elf-gcc', f'-march={march}', f'-mabi={mabi}',
        '-O2', '-ffreestanding', '-c', 'kernels.c', '-o', obj)
    nm = run('riscv64-elf-nm', '-S', obj)          # size per function
    raw = run('riscv64-elf-objdump', '-dr', obj)   # instructions and calls
```

`-mabi` is the ABI that determines which registers carry function arguments. Combinations without floating-point registers use `ilp32`, those with F use `ilp32f`, and 64-bit with D uses `lp64d`.

![The same 7 C functions shrink from 972 bytes on rv32i to 772 bytes on rv32im, 502 bytes on rv32imc, and 402 bytes on rv32imfc_zba_zbb|xl](/img/01M6K1PY0PQGCWC5TMFEAYF4DE/a2e478033eb8.webp)

Let's follow the results one extension at a time.

**Without M, multiplication becomes a function call.** On `rv32i`, `matmul` calls `__mulsi3` for every multiplication, and `utoa10` calls `__udivsi3` and `__umodsi3`. With no hardware multiplier, the compiler multiplies in software. Turning on M shrank `matmul` from 260 bytes to 132 bytes and `utoa10` from 204 bytes to 132 bytes. The figure counts only the calling side; the size and time of the library functions being called are not included. The real difference is larger.

**C keeps the instruction count the same and only reduces bytes.** `rv32im` and `rv32imc` both have 193 instructions. Yet the size dropped 35%, from 772 bytes to 502 bytes, because 135 of the 193 became 16-bit compressed instructions. C extension instructions are not new operations; they are 16-bit aliases for frequently used 32-bit instructions (e.g., `addi a1,a1,4`). So by placing one "expand 16-bit to 32-bit" stage in front of the decoder, the rest of the hardware can be used as is.

**Without F, float operations become function calls.** On `rv32imc`, `dot` called `__addsf3` and `__mulsf3` and was 96 bytes. Turning on F replaced those with instructions such as `fmadd.s` (multiply-add), giving 32 bytes and 12 instructions.

**B merges frequent instruction sequences into one.** Without Zbb, `popcount_sum` called `__popcountsi2`; with it, a single `cpop` instruction did the job (68 bytes → 32 bytes). In `index_sum`, "multiply the index by 4 and add it to the address" went from two instructions, `slli`+`add`, to a single `sh2add`. Below is the actual loop of `index_sum` with Zba enabled.

```asm
158:  419c        lw      a5,0(a1)        # idx[i]   (16-bit C instruction)
15a:  0591        addi    a1,a1,4
15c:  20a7c7b3    sh2add  a5,a5,a0        # base + idx*4 (Zba)
160:  439c        lw      a5,0(a5)
162:  973e        add     a4,a4,a5
164:  feb61ae3    bne     a2,a1,158
```

Two-byte and four-byte instructions are mixed within a single loop. The hardware has to determine instruction length by looking at the value read from memory. To make this easy, RISC-V defined that **if the low two bits of an instruction are `11`, it is 32-bit; otherwise it is 16-bit**. The low two bits of `20a7c7b3` are `11`, and the low two bits of `419c` are `00`. Since only two bits need checking, length detection takes just a few gates.

**Going to 64-bit makes it slightly bigger.** `rv64imafdc` is 468 bytes, larger than `rv32imfc` at 438 bytes. In `utoa10`, handling 32-bit values in 64-bit registers added sign- and zero-extension instructions (110 bytes vs. 94 bytes). RV64's W instructions (32-bit operation followed by sign extension) exist to reduce this problem. Part 5 covers them in detail.

## 6. Design Checklist: What the Name Decides in Hardware

Before it is a software option, the name is a hardware spec sheet. Collecting what the name immediately determines when you start designing a core:

| To decide | How to choose | Easy to miss |
|---|---|---|
| XLEN and base ISA | RV32I for a 32-bit MCU (or RV32E with fewer registers), RV64I for Linux | E halves the register file, but the calling convention differs, so it needs the `ilp32e` ABI |
| Extension list | Decide by required operations and area | Instructions of any extension not chosen must all raise an **illegal instruction exception**. They must not silently do nothing |
| C support | Nearly mandatory if code memory is small | With C, instructions sit on 2-byte boundaries (IALIGN=16). A 4-byte instruction can straddle two words, complicating fetch |
| Misaligned jump targets | Without C, IALIGN=32 | A branch or jump target that is not a multiple of 4 raises an instruction address misaligned exception |
| Zicsr | Effectively mandatory to take interrupts and exceptions | Writing just `rv32i` leaves out CSR instructions |
| `misa` CSR | Reports supported extensions as bits | It is a WARL (write any value, reads return only legal values) field, so it may be made read-only. Details in part 23 |
| Endianness | Little-endian by default | Instructions are always read as little-endian. Big-endian data is an optional feature of the Privileged specification |
| Specification edition | Pick one ratified edition and record it in the documentation | Keep extension names and versions (e.g., `zicsr2p0`) identical in hardware documentation and compiler options |

The last row is where accidents often happen in practice. If the hardware is `rv32imc` but the firmware is built for `rv32imac`, the atomic instructions inserted by the compiler raise illegal instruction exceptions. Make it your first rule to keep the name identical in three places: the hardware documentation, the compiler options, and the `misa` value.

## Summary

- RISC-V is a modular ISA that adds selected extensions to a small, unchanging base integer ISA. The specification is split into Volume I (Unprivileged), Volume II (Privileged), and external specifications such as ABI, SBI, Debug, and interrupts.
- `rv64gc` is short for `rv64i_m_a_f_d_c_zicsr_zifencei`. The name is both a compiler option and a hardware spec sheet.
- Measured: the same 7 C functions are 972 bytes on `rv32i`, 772 bytes on `rv32im`, 502 bytes on `rv32imc`, and 402 bytes on `rv32imfc_zba_zbb`. M and F turn function calls into instructions, and C cut size by 35% while keeping the instruction count the same.
- Design starts with choosing the name. Unchosen extensions raise illegal instruction exceptions, choosing C means 2-byte-aligned fetch, and taking interrupts needs Zicsr.

**Next post:** Part 2 goes into the base integer ISA RV32I. It looks at the 32 registers and the 6 instruction formats (R, I, S, B, U, J), and follows from the decoder's point of view why the immediate bits are scattered the way they are. I will hand-encode a few instructions and check them against disassembler output.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. https://github.com/riscv/riscv-isa-manual
- RISC-V Profiles (RVA20, RVA22, RVA23). https://github.com/riscv/riscv-profiles
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, Strawberry Canyon, 2017. Chapter 1 (Why RISC-V?)
- A. Waterman, Y. Lee, D. Patterson, K. Asanović, "The RISC-V Instruction Set Manual, Volume I: Base User-Level ISA," UCB/EECS-2011-62, 2011. https://www2.eecs.berkeley.edu/Pubs/TechRpts/2011/EECS-2011-62.html
- RISC-V ELF psABI. https://github.com/riscv-non-isa/riscv-elf-psabi-doc
