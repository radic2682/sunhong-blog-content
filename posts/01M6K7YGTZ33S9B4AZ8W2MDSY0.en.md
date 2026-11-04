---
id: "01M6K7YGTZ33S9B4AZ8W2MDSY0"
title_en: "FENCE.I and Self-Modifying Code: What the Instruction Side Must Honor"
summary_en: "Part 22 of the RISC-V series. Covers FENCE.I and self-modifying code. Without fence.i, QEMU ran the new code when it was written elsewhere but the old code inside the block being executed. Running 16 combinations on an instruction-set model, I identified the minimal FENCE.I implementation for each core organization."
---
:::note
**Key points in three lines**
- RISC-V does not guarantee that a stored value is visible to the same hart's instruction fetches until FENCE.I. There are three places where written code can be invisible: a write-back data cache, the instruction cache, and instructions already fetched (the front of the pipeline and prefetch buffers).
- I ran the same self-modifying-code ELF on 16 combinations of a Python instruction-set model (4 core configurations × 4 FENCE.I implementations). FENCE.I could be a NOP only on a core with neither caches nor prefetching. Even without caches, prefetching required a flush; an I$ required invalidation; and a write-back D$ additionally required a write-back.
- Without fence.i, QEMU 11.1.3 ran the new code (2) when the code was written elsewhere, but ran the old code (11) when the very next instruction was modified. TCG detects writes to translated pages, but the translation block currently executing runs to the end on the old translation. Shrinking blocks to one instruction made both cases run the new code.
:::

Parts 19–21 added M, A, and C to the RV32I 5-stage pipeline core. This part, the last of Part II, covers the one remaining rule on the instruction side: FENCE.I. Where part 20's FENCE orders data accesses between harts, FENCE.I reconciles "what was written as data" with "what is read as instructions" within a single hart. JIT compilers, bootloaders that copy code into RAM and jump to it, and debuggers' software breakpoints all rely on this rule.

## 1. Why Written Code Is Not Visible

In a von Neumann architecture, instructions and data live in the same memory. A pipelined core, however, reads instructions and data along separate paths. The IF stage goes through the instruction cache (I$) or instruction bus, and the MEM stage through the data cache (D$) or data bus. They are split so that one instruction and one data word can be read in the same cycle. By default there is no connection at all between these separate paths.

When new instructions are written with `sw`, there are three places where the instruction side can still hold the old bits.

| Location | Why old bits remain |
|---|---|
| ① Write-back D$ | The new code is only in a dirty D$ line and not yet in memory. The I$ fills from memory |
| ② I$ | A line remains from previously executing that address |
| ③ Already-fetched instructions | Instructions in IF and ID of the 5-stage pipeline, the prefetch buffer, part 21's fetch buffer |

![New code written by sw goes to the D$, while the instruction side may hold old bits in the I$ and in already-fetched instructions. FENCE.I performs a D$ write-back, an I$ invalidation, and a flush, in that order|full](/img/01M6K7YGTZ33S9B4AZ8W2MDSY0/d6c8f8a51152.webp)

③ exists even in the simplest core with no caches. Let's trace by hand the case of modifying the very next instruction. Test 2 in the lab code looks like this.

```text
80000110: sw   t1,0(t0)    # t0 = 0x80000114
80000114: addi a0,x0,11    # change this instruction to addi a0,x0,22
```

Here is where each instruction is in the 5-stage pipeline, cycle by cycle.

```text
Cycle    IF          ID      EX      MEM
  1      sw
  2      addi(114)   sw
  3      next        addi    sw
  4                  next    addi    sw → writes 0x114
```

In cycle 4, when `sw` writes 0x80000114 in MEM, the old `addi` at that address is already executing in EX. The old instruction was fetched in cycle 2. With a prefetch buffer, instructions even farther ahead may already be inside.

## 2. What FENCE.I Means: Same Hart Only

FENCE.I (the Zifencei extension, Chapter 6 of the Unprivileged specification) is I-type, with opcode MISC-MEM (`0001111`) and funct3 `001`. The rs1, rd, and imm fields are reserved for finer-grained fences in the future, and implementations must ignore them. The specification's rule comes down to two sentences.

- RISC-V does not guarantee that stores to instruction memory will be made visible to instruction fetches on that hart until the hart executes a FENCE.I.
- FENCE.I ensures that subsequent instruction fetches see all previous data stores "already visible to the same hart." **It does not guarantee anything for fetches on other harts.**

"Already visible to the same hart" is the basis for ①. New code sitting only in the D$ is already visible to that hart's loads, so after FENCE.I it must be visible to fetches too. A core with a write-back D$ and an I$ that fills from memory must therefore either write the D$ back on FENCE.I or have I$ fills snoop the D$.

The name resembles part 20's FENCE, but the job is different.

| Item | FENCE | FENCE.I |
|---|---|---|
| What it orders | Data accesses (R, W, I, O) relative to each other | Data stores → instruction fetches |
| From whose view | The order observed by other harts and devices | Same hart only |
| Simple in-order core | NOP if nothing reorders accesses | At least a flush on a 5-stage pipeline |
| Extension | Base RV32I | Zifencei (originally in the base ISA, later split out) |

The specification's commentary also explains why FENCE.I was removed from the base ISA. In designs where the I$ and D$ are not coherent with each other, FENCE.I has to flush both caches entirely, which is expensive, and in user mode in a multi-hart environment it is not even sufficient on its own (explained below). So even on a simple core that can treat FENCE as a NOP, FENCE.I needs separate thought.

To make new code visible to multiple harts that have instruction caches, the specification says the writing hart should first execute a data FENCE and then request that all other harts execute FENCE.I. How to make that request is outside the specification (it belongs to the execution environment). There are two common methods.

| Method | What it does |
|---|---|
| IPI + FENCE.I | The writing hart sends a software interrupt to the other harts, and each receiving hart runs FENCE.I in its handler |
| SBI `sbi_remote_fence_i` | The S-mode OS asks the M-mode firmware (OpenSBI, etc.). The SBI RFENCE extension |

Linux removed FENCE.I from the user-mode ABI for the same reason: if a thread is migrated to another hart right after FENCE.I, it does no good. Instead, it leaves the job to the OS through a system call (`riscv_flush_icache`). The commentary in Chapter 6 of the specification describes exactly this situation.

## 3. FENCE.I Implementation by Core Organization

The specification does not prescribe how to implement it, only the result. The commentary says "a simple implementation can flush the local instruction cache and the instruction pipeline," and "if the I$ and D$ are kept coherent, or the memory is uncached RAM only, it only needs to flush the fetch pipeline." Organized by structure:

| Core organization | What FENCE.I must do |
|---|---|
| No cache, no fetched instructions | Nothing (NOP) |
| No cache, pipeline/prefetch present | Flush younger instructions, refetch from pc+4 |
| I$ (write-through D$ or no D$) | Flush + I$ invalidation |
| I$ + write-back D$ | Flush + I$ invalidation + write back dirty D$ lines |
| I$ snoops stores and clears itself | Flush only |

"No cache, no fetched instructions" does not actually apply to a 5-stage pipeline. As the table in section 1 shows, a 5-stage pipeline by itself fetches two or three instructions ahead. So on this series' 5-stage core, the minimal FENCE.I implementation is "squash everything behind it and restart at pc+4, like a branch misprediction." Since it can reuse the existing flush path (part 18's branch handling), a simple approach is to have the decoder mark FENCE.I as "an unconditional jump to pc+4 that is always mispredicted." If there is a store buffer, it must also wait until earlier stores reach memory (or a coherent cache).

## 4. Lab 1: Self-Modifying Code in QEMU

The test code (`smc.c`) does two things. The setup is a Mac M1 Max, QEMU 11.1.3, and `riscv64-elf-gcc` 16.2 (`-march=rv32i_zifencei`).

- **Test 1 (code written elsewhere):** write `addi a0,x0,1; ret` into the data array `code[]` and call it after FENCE.I. Write `addi a0,x0,2; ret` to the same place and call it **without FENCE.I**. Then call it again after FENCE.I.
- **Test 2 (the very next instruction):** run the `sw` + `addi` pair from section 1 once without FENCE.I, and once with FENCE.I between `sw` and `addi`.

```c
/* addi a0, x0, k */
#define LI_A0(k) ((uint32_t)(k) << 20 | 10u << 7 | 0x13u)
#define RET 0x00008067u   /* jalr x0, 0(ra) */
write_code(1); __asm__ volatile("fence.i" ::: "memory");
put("t1 first      ", ((fn)code)());
write_code(2);            /* no fence.i */
put("t1 no fence.i ", ((fn)code)());
```

| Test | QEMU default | QEMU one instruction at a time |
|---|---|---|
| Test 1, first | 1 | 1 |
| Test 1, without FENCE.I | **2** | 2 |
| Test 1, after FENCE.I | 2 | 2 |
| Test 2, without FENCE.I | **11** | 22 |
| Test 2, after FENCE.I | 22 | 22 |

QEMU ran the new code in test 1 even without FENCE.I. On real hardware with an I$, this is where the old code (1) could come out. The reason lies in how QEMU's TCG works. TCG translates guest instructions into host code in translation blocks (basic-block units), and when a write hits a guest page containing a translated block, it discards that page's translations (inferred: QEMU's self-modifying code detection). In hardware terms, it behaves like "a fully coherent I$ that snoops every store."

Test 2, by contrast, ran the old code (11) without FENCE.I. The modified instruction was inside the translation block currently executing. The block was already translated, so it runs to the end on the old translation. Rerunning with the option that splits blocks into single instructions (`-accel tcg,one-insn-per-tb=on`) produced 22. In QEMU, FENCE.I ends the block and forces the next block to be translated fresh, so both cases give 22.

In short, QEMU just exhibits one of the behaviors the specification allows; it is not the same as hardware. A software bug that omits FENCE.I may not show up in QEMU, and conversely, when using QEMU as the reference model for RTL verification, self-modifying code without FENCE.I should be excluded from comparison.

## 5. Lab 2: 16 Combinations on an Instruction-Set Model

I ran the same `smc.elf` on a small RV32I instruction-set model I wrote myself (`iss.py`). It loads the ELF's loadable segments into memory, collects stores to the UART address as characters, and stops on a write to the exit device. Three things can be added on the instruction side.

- **Prefetch queue:** fetches in order up to 3 instructions ahead of the one being executed (a modeling choice meant to correspond to IF, ID, and one buffer in the 5-stage pipeline). It is emptied when a branch is taken.
- **I$:** direct-mapped, 16 lines × 16 bytes. Fills from memory on a miss.
- **Write-back D$:** holds stores as "dirty words" and writes them to memory only when FENCE.I asks for a write-back. Loads check it first.

```python
def fence_i(self):
    if self.fencei == 'nop': return
    if self.fencei == 'full' and self.wb:   # ① D$ write-back
        for a, v in self.dirty.items():
            self.write_mem(a, v)
        self.dirty = {}
    if self.fencei in ('flush+inval', 'full'):
        self.itag = [None] * len(self.itag) # ② I$ invalidation
    self.redirect(self.pc + 4)              # ③ flush
```

First, as a correctness check, I confirmed that the output of configuration A, with neither caches nor prefetching, matched all five lines of QEMU run one instruction at a time (it did). Then I ran every combination of 4 core configurations and 4 FENCE.I implementations (NOP, flush only, flush + I$ invalidation, and that plus a D$ write-back).

![The combinations of 4 core configurations × 4 FENCE.I implementations that honored the specification: with no caches and no prefetching even a NOP works, and each added structure required one more step of flush, invalidation, or write-back|full](/img/01M6K7YGTZ33S9B4AZ8W2MDSY0/b556fb14dd26.webp)

| Core configuration | Test 1 / 2 without FENCE.I | Minimal spec-compliant implementation |
|---|---|---|
| A No cache, prefetch 0 | 2 / 22 | NOP |
| B No cache, prefetch 3 | 2 / 11 | Flush |
| C I$ + prefetch 3 | 1 / 11 | Flush + I$ invalidation |
| D I$ + write-back D$ + prefetch 3 | Fails from the first call | + D$ write-back |

In configuration D, if FENCE.I did not write back the D$, it failed from the very first call. `code[]` starts out in a zero-filled region (.bss), and the newly written `addi a0,x0,1` was only in the D$. The I$ fetched the zeros from memory, so the core ended up executing `0x00000000` (an illegal instruction) at 0x80000200. As seen in part 21, because the specification defines the all-zeros instruction as illegal, a bug like this shows up immediately as a trap instead of as wild execution.

Looking at the results without FENCE.I, QEMU's default run (2 / 11) had the same shape as configuration B, "a core with no cache but with instructions fetched ahead."

I also measured one cost. In configuration C, the whole program (693 instructions, 3 FENCE.Is) had 37 I$ misses when FENCE.I did not invalidate and 55 when it did. Each FENCE.I discards the entire I$, so in real code it causes misses in proportion to the size of the code executed after the invalidation. This is why the specification's commentary cites, for JIT compilers, the approach of "generating a long run of translated instructions and then calling FENCE.I only once."

## 6. Implementation Details on the 5-Stage Core

A few things must be decided when moving to RTL. The specification does not define them, so I list the common choices.

**Which stage handles it.** FENCE.I must take effect after earlier stores complete. On a 5-stage core, the simplest method is to send FENCE.I down to the MEM stage (by which point the preceding `sw` has moved on toward WB), then trigger the I$ invalidation, flush IF, ID, and EX, and refetch from pc+4. Handling it in EX risks reversing the order, since the immediately preceding `sw` would still be in MEM. The flush path is the same as for a branch misprediction, so the penalty is the number of flushed stages (3 cycles with this approach) plus the I$ miss cost (inferred; varies by implementation).

**How many cycles to invalidate the I$.** If the valid bits are kept in flip-flops, they can all be cleared in one cycle. If they are kept in SRAM with the tags, each line must be written once, taking as many cycles as there are lines, and fetch must stall in the meantime. Small cores with few lines often pick the former, and large I$s the latter (inferred).

**D$ write-back.** Writing back every dirty line of a write-back D$ is expensive. So designs either have I$ fills snoop the D$ first, or place a unified L2 beneath both and make the L1 D$ write-through to eliminate the problem. The specification's commentary also gives several examples of keeping the I$ and D$ coherent.

**When it is actually used.** Self-modifying code sounds like an exotic technique, but it hides in common places: when a bootloader copies the next-stage code from flash into RAM and jumps to it, when the OS loads a program into memory, when a debugger writes in a software breakpoint (`ebreak`), and when a JIT generates code. Configuration D failing from the first call in section 5 is exactly the symptom of "a bootloader that copied and jumped without FENCE.I."

## 7. Design Checklist: FENCE.I

| Item | Rule |
|---|---|
| Encoding | MISC-MEM, funct3 `001`. rs1, rd, imm ignored (software sets them to 0) |
| Minimal implementation | Flush younger instructions + refetch from pc+4. Always needed on a 5-stage pipeline |
| I$ | Invalidate it, or use a coherence structure that snoops stores and clears lines |
| Write-back D$ | Write back dirty lines, or have I$ fills snoop the D$ |
| Store buffer | Flush after earlier stores reach where the I$ looks |
| Fetch buffer (C extension) | Empty it. Leftover 16-bit slots also hold old bits |
| Branch predictor/BTB | Stale predictions based on old code are not a correctness problem (fixed by misprediction). Clearing them is optional |
| Scope | Same hart only. Other harts via IPI or SBI `sbi_remote_fence_i` |
| User mode | The instruction can execute, but the Linux ABI uses a system call |
| Traps | FENCE.I itself does not trap. If Zifencei is not implemented, treating it as an illegal instruction is the common choice |
| Debug breakpoints | A debugger writing in `ebreak` is also self-modifying code. Debug mode entry and exit need the same handling (part 37) |

## Summary

- Stored code can be invisible to the instruction side in three places: a write-back D$, the I$, and already-fetched instructions. Even a 5-stage pipeline with no caches has the third.
- FENCE.I makes the same hart's subsequent fetches see all earlier stores visible to that hart. Other harts must each be made to execute FENCE.I through an IPI or SBI `sbi_remote_fence_i`.
- Across the 16 combinations on the instruction-set model, FENCE.I's job grew with the structure: NOP → flush → invalidation → D$ write-back. Missing any one of them meant running old code (1, 11) or hitting an illegal instruction from the first call.
- Without FENCE.I, QEMU 11.1.3 ran the new code when it was written elsewhere, and the old code for the very next instruction inside the executing block. This comes from translating in translation-block units, and it differs from hardware.

**Next post:** Starting with part 23, the series moves on to Part III, the privileged architecture. The first post covers privilege modes and CSRs: why M, S, and U modes are separated, how a CSR's 12-bit address encodes its privilege level and whether it is read-only, what WARL and WLRL fields leave to the implementation, and a map of the main CSRs covered from here on.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 6, "Zifencei" extension; Section 2.7, memory ordering instructions; Chapter 17, RVWMO. https://github.com/riscv/riscv-isa-manual
- RISC-V Supervisor Binary Interface Specification, RFENCE extension (`sbi_remote_fence_i`). https://github.com/riscv-non-isa/riscv-sbi-doc
- D. Harris et al., *RISC-V System-on-Chip Design*, 2023. Caches and the FENCE.I implementation (CORE-V Wally)
- D. Patterson, J. Hennessy, *Computer Organization and Design: RISC-V Edition*. Chapter 4, pipelining; Chapter 5, memory hierarchy
- QEMU TCG documentation (translation blocks and self-modifying code). https://www.qemu.org/docs/master/devel/tcg.html
