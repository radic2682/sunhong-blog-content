---
id: "01M70HYVSMKS8FPAYNWXMWSDHT"
title_en: "RISC-V Privilege Modes and CSRs: M/S/U, Address Rules, WARL Fields on QEMU"
summary_en: "Part 23 of the RISC-V series. We tested the M/S/U privilege modes, the CSR address rule ([11:10] read-only, [9:8] lowest privilege) and WARL/WLRL/WPRI fields on QEMU. The address rule predicted all 67 CSR cells, and illegal mtvec and satp values had the whole write ignored."
---
:::note
**Key points of this post**
- RISC-V has three privilege modes, U (00), S (01) and M (11), and only M is mandatory. On QEMU, running `ecall` after dropping into each mode left `mcause` = 11 for M, 9 for S and 8 for U. When S or U is removed from the configuration, the mode written to `mstatus.MPP` is not kept (writing 1 on an M+U hart reads back 0; an M-only hart always reads 3).
- The top bits of the 12-bit CSR address are the rule. [11:10]=11 means read-only, and [9:8] is the lowest privilege that may access it. Counting the 4,096 addresses from the spec table gives 3,184 standard, 896 custom and 16 debug-mode-only. This rule predicted every one of the 67 cells for existing CSRs among the 85 read/write cells measured on QEMU.
- A WARL field raises no exception and only guarantees a legal value on read. On QEMU, writing 2 or 3 to `mtvec.MODE` ignored the whole write, `satp.MODE` accepted only 0, 8, 9 and 10, and `misa` ignored every write in the default configuration.
:::

The earlier parts covered the unprivileged ISA and core implementation. From part 23 on we are in Part 3 of the series, the privileged architecture. A core that stops at running one application cannot support an operating system, interrupts or virtual memory. For that, the hardware has to know "which privilege is running right now" and "what state that privilege may touch". The answers are privilege modes and CSRs. [Part 12](post:01M6K7VS05ZSDFEQQ7GZW0C1RJ) covered the six instructions that read and write CSRs, so this post looks at the CSR side: which CSRs exist, how the addresses are laid out, and which fields do not read back what you wrote. Part 24 turns these rules into a hardware CSR file and part 25 covers traps. In this post `mcause` appears only as a number, as evidence that an exception happened; trap handling itself is left to part 25.

The experiments ran on QEMU 11.1.3 (`virt` board, Mac M1 Max) with `riscv64-elf-gcc` 16.2. The specification is the Privileged spec, version 20240411.

## 1. Privilege modes: why three, and how many to implement

A hart (hardware thread) always runs at exactly one privilege level. Section 1.2 of the spec defines three.

| Encoding | Name | Abbrev. | Typical use |
|---|---|---|---|
| `00` | User/Application | U | applications |
| `01` | Supervisor | S | OS kernel |
| `10` | (reserved) | | |
| `11` | Machine | M | firmware, trusted code |

M is the only mode every implementation must have. The simplest core may have only M (small embedded systems); the spec lists three combinations: M, M+U (secure embedded) and M+S+U (Unix-like OS). A higher mode can reach more, and the basic rule is that the CSRs and instructions of one level are also available at every higher level. Doing something not permitted raises an exception.

![Privilege modes are M (11), S (01) and U (00). On QEMU, ecall left mcause 11 in M, 9 in S and 8 in U. Traps and ecall go up; mret and sret come back down|full](/img/01M70HYVSMKS8FPAYNWXMWSDHT/f9086291715f.webp)

There are only two ways to move between modes. Going from low to high happens through a **trap** (exception, interrupt or `ecall`), and coming back uses `mret` or `sret`. The spec calls a trap that raises the privilege level vertical and one that stays at the same level horizontal. Which trap goes to which mode (delegation) is part 26, and the handling sequence is part 25.

### Checking it: which mode am I in?

There is no CSR that reads the current mode. The spec made it that way on purpose: lower modes do not need to know their own mode, and a virtualization layer can hide it. Instead, the cause code of `ecall` differs per mode, which works as a de facto mode indicator (table in Privileged spec 3.1.15). On QEMU we ran `ecall` in M-mode and in S- and U-mode entered with `mret`, and recorded `mcause`.

| Mode run in | `mcause` | Name |
|---|---|---|
| U | 8 | environment call from U-mode |
| S | 9 | environment call from S-mode |
| M | 11 | environment call from M-mode |

RV64 and RV32 gave the same values. When S is removed from the configuration (M+U), U was still 8, and there is no S line because there is no S.

## 2. mstatus.MPP: you find implemented modes by writing

When a trap is taken, the hardware puts the "mode before the trap" into `MPP` of `mstatus` (bits 12:11, two bits), and `mret` returns to that value. The field is WARL, so it can only hold implemented modes. Section 3.1.6 of the spec says that M-mode software can find out whether a mode is implemented by writing it to MPP and reading it back. We checked three configurations. Each cell is what was read back after writing the value in the first column.

| Written | M+S+U | M+U | M only |
|---|---|---|---|
| 0 (U) | 0 | 0 | 3 |
| 1 (S) | 1 | 0 | 3 |
| 2 (reserved) | 0 | 0 | 3 |
| 3 (M) | 3 | 3 | 3 |

On the M+U hart, writing 1 (S) read back 0 (U), and on the M-only hart anything written read back 3. An implemented mode is kept in place of the unimplemented one (which one is kept is an implementation choice). 2 is reserved and was never kept in any configuration (the value before the write, 0, was read back).

M-mode code drops to a lower mode in three steps: write the destination address to `mepc`, write the mode to `MPP`, and run `mret`. The experiment's `enter_mode` does exactly this.

```asm
enter_mode:             # a0 = 0 (U) or 1 (S), a1 = function to run
    csrw mepc, a1       # address to return to
    li   t1, 0x1800
    csrc mstatus, t1    # clear the two MPP bits
    slli a0, a0, 11
    csrs mstatus, a0    # put the target mode in
    mret                # jump into that mode
```

### mstatus written with all ones and all zeros

We wrote all ones to the whole of `mstatus` and read it back, then wrote all zeros and read back (restoring the original at the end).

| Written | RV64 read | RV32 read |
|---|---|---|
| all ones | `0x8000000a007e79aa` | `0x807e79aa` |
| all zeros | `0x0000000a00000000` | `0x00000000` |

Decoding the RV64 all-ones result into fields (bit positions from spec 3.1.6): `SIE`, `MIE`, `SPIE`, `MPIE`, `SPP`, `MPRV`, `SUM`, `MXR`, `TVM`, `TW` and `TSR` are 1, `MPP`=3 and `FS`=3. `UBE`, `MBE`, `SBE`, `VS` and `XS` stayed 0 (this QEMU CPU has no V extension and no big-endian support). The top bit `SD` (bit 63) is 1 although we did not write it: it is a read-only summary that becomes 1 when `FS` or `XS` is dirty. WPRI bits such as 0, 2, 4 and 62:38 were read-only zero. Even after writing all zeros, `0xa00000000` remains. Bits 33 and 35, that is `UXL`=2 and `SXL`=2 (both 64-bit), are WARL and cannot hold 0 (reserved), so the legal value 2 stays. RV32 has no such fields and returns to 0.

## 3. CSR address rules

CSR addresses are 12 bits (`csr[11:0]`), so there are 4,096 of them. The rule in section 2.1 of the spec is that the top four bits are the **default** access permission.

![With bits 11:10 equal to 11 a CSR is read-only, and bits 9:8 give the lowest privilege. The four example CSRs read in M, S and U exactly as the rule predicts, except cycle, blocked in S and U because mcounteren=0|full](/img/01M70HYVSMKS8FPAYNWXMWSDHT/a4d5b74115ac.webp)

- `csr[11:10]`: `00`, `01` or `10` means read/write, `11` means read-only.
- `csr[9:8]`: lowest privilege needed. `00` U, `01` S, `10` hypervisor (HS and VS CSRs of the H extension), `11` M.
- The rest, `csr[7:4]` and so on: the number inside the block. These bits also separate standard from custom CSRs.

Thanks to this layout, the privilege check is one comparator. Compare `[9:8]` of the CSR address field with the current mode, and if a write is attempted, check only whether `[11:10]` is `11`. The spec note also explains that this simplifies error checking and gives a larger address space. Together with the rule from part 12 that whether a CSR is written is decided by instruction fields and not register values, both write intent and permission can be settled in decode, before reading anything.

### Counting the 4,096 addresses

We transcribed the spec's CSR address range table and counted the 4,096 addresses by block in Python. Each level has 1,024.

| Level ([9:8]) | Standard R/W | Standard read-only | Custom |
|---|---|---|---|
| U (00) | 512 | 192 | 320 |
| S (01) | 640 | 192 | 192 |
| H (10) | 640 | 192 | 192 |
| M (11) | 624 (+16 debug-only) | 192 | 192 |

In total: 3,184 standard (768 of them read-only), 896 custom and 16 debug-mode-only (`0x7B0` to `0x7BF`). Custom ranges are addresses promised never to be touched by future standard extensions, so a vendor can place its own CSRs there. In M-mode, `0x7A0` to `0x7AF` is for the debug system and `0x7B0` to `0x7BF` is visible only in debug mode (an M-mode access should be an illegal instruction). One caveat: the 20240411 table lists `0xE00` to `0xEBF` and `0xF00` to `0xFBF` as "read/write", but the body rule says [11:10]=11 is read-only and entries such as `mvendorid` are marked MRO, so this post counts them as read-only (the table looks like a typo, an inference).

### Cross-checking against QEMU: predicting from the address bits

The reference model is two lines: "a CSR is readable if mode >= [9:8]" and "it can be written back with the value just read if it is readable and [11:10] != 11". We read 18 CSRs with `csrr` in M, S and U, wrote the same value back with `csrw` when the read succeeded, and recorded `mcause`. S and U ran after opening one PMP entry (everything allowed) and dropping with `mret`.

```c
#define T(n, a) { ul v_ = 0; trap_cause = 0; \
  __asm__ volatile("csrr %0, " a : "=r"(v_)); \
  ul rc_ = cause_take(), wc_ = 0xff; \
  if (!rc_) { __asm__ volatile("csrw " a ", %0" :: "r"(v_)); \
              wc_ = cause_take(); } \
  emit(..., n, rc_, wc_); }
CSRS(T)     /* X("cycle","cycle") X("mstatus","mstatus") ... */
```

| Comparison | RV64 | RV32 |
|---|---|---|
| Cells tested (read and write) | 85 | 85 |
| Matches the model | 67 | 67 |
| Differs, with a reason | 18 | 18 |
| Differs, no reason | 0 | 0 |

The 18 differing cells have only two reasons. (1) 12 cells for non-existent CSRs: the custom addresses `0x8FF` (U level), `0x5C0` (S) and `0x7C0` (M) do not exist on QEMU, so they were illegal even in M-mode. Section 2.1 says accessing a non-existent CSR is reserved, that is, illegal. (2) 6 cells for counters: by address `cycle`, `time` and `instret` are readable from U, but `mcounteren` was 0 (its reset value), so S and U got illegal. The address rule is only a default, and a CSR can block more. We confirmed it with the counter-enable bits.

| `mcounteren`.CY | `scounteren`.CY | `rdcycle` in S | `rdcycle` in U |
|---|---|---|---|
| 0 | 0 | illegal | illegal |
| 0 | 1 | illegal | illegal |
| 1 | 0 | read OK | illegal |
| 1 | 1 | read OK | read OK |

For U-mode to read, the enable bits in both M and S had to be on. Part 24 treats this counter structure as a design topic. Running privileged instructions in a lower mode gave the same illegal instruction: `mret`, `sret` and `sfence.vma` in U-mode and `mret` in S-mode all gave `mcause`=2. The spec says it does not distinguish privileged-instruction violations from illegal instructions (note in 3.1.15).

## 4. WARL, WLRL, WPRI: three field rules

Each CSR field has rules for writing and reading (spec 2.3), because not every bit pattern is meaningful.

| Abbrev. | Name | Write | Read |
|---|---|---|---|
| WPRI | Reserved Writes Preserve, Reads Ignore | preserve the value when writing other fields | software ignores it; read-only 0 if absent |
| WLRL | Write/Read Only Legal | only legal values may be written | guaranteed only if the last write was legal |
| WARL | Write Any, Read Legal | any value, no exception | always a legal value |

WPRI is blank space for future extensions. The implementation makes it read-only zero, and software preserves what it read when writing back. For WLRL, writing an illegal value may make the read return any value (deterministic from the written value and the previous value), and an exception is allowed. The exception code of `mcause` is WLRL. WARL is the most common: it accepts the write without an exception but reads back a legal value, so software finds the supported range by **writing and reading back** (probing). What to keep after an illegal write is up to the implementation as long as it is deterministic from the written value and hart state. For a hardware designer that freedom is a choice; for a verifier it means values that differ per implementation.

![Written and read values: mstatus.MPP 2 is ignored, mtvec.MODE 2 and 3 ignore the whole write, satp.MODE accepts only 0, 8, 9 and 10, and misa ignores everything by default and only C, H and M cleared when writes are enabled|full](/img/01M70HYVSMKS8FPAYNWXMWSDHT/3caeaa808b92.webp)

The WARL experiments used one macro that writes, reads and restores the original value.

```c
#define WARL(kind, name, csr, val) do { ul o_ = RD(csr), r_, w_ = (ul)(val); \
  __asm__ volatile("csrw " csr ", %1\n\tcsrr %0, " csr \
                   : "=&r"(r_) : "r"(w_)); \
  WR(csr, o_); emit(kind, name, w_, r_); } while (0)
```

### mtvec: the whole write is ignored

The low two bits MODE of `mtvec` define only 0 (direct) and 1 (vectored); values >= 2 are reserved (spec 3.1.7). We wrote MODE values on top of base `0x80004000`.

| Written | Read back |
|---|---|
| `0x80004000` (MODE 0) | `0x80004000` |
| `0x80004001` (MODE 1) | `0x80004001` |
| `0x80004002` (MODE 2) | old value |
| `0x80004003` (MODE 3) | old value |
| `0x8000407c` | `0x8000407c` |
| all ones | old value (includes MODE=3) |

Not only MODE but the whole write disappeared, so even the BASE part kept its old value. The spec only requires "a legal value on read", so an implementation that takes the new BASE and reverts only MODE is also allowed. You pick this when you implement it in part 24. The old value in this experiment is the trap handler address (`0x80000040`) written by the startup code.

### satp.MODE and misa

The MODE of `satp` (address translation, chapter 4) is four bits. The spec says that writing an unsupported MODE has no effect on the whole write. On RV64 we wrote MODE 0 to 15 with PPN `0x80000`, and only 0 (Bare), 8 (Sv39), 9 (Sv48) and 10 (Sv57) read back; the other 12 left the old value 0. RV32 has a single MODE bit (bit 31) and accepted both 0 and 1 (Sv32). Finding the supported MODEs is exactly this "write and read back".

`misa` (3.1.1) is WARL; `MXL` (top two bits) gives the width, and the low 26 bits are extensions A to Z. This is what QEMU read.

| Item | RV64 | RV32 |
|---|---|---|
| `misa` | `0x80000000001411ad` | `0x401411ad` |
| MXL | 2 (XLEN 64) | 1 (XLEN 32) |
| Extensions on | A C D F H I M S U | A C D F H I M S U |

Writing all zeros, all ones, clearing one bit or setting an absent bit to `misa` always read back the original value in QEMU's default configuration. That is correct, since the spec allows `misa` to be read-only (a WARL with a read-only value). With `x-misa-w=true`, writes open up, but of the 9 bits that are on, only `C`, `H` and `M` were cleared; writes clearing `A`, `D`, `F`, `I`, `S` and `U`, and writes setting bits that were off, were all ignored. The spec says that `F`=0 with `D`=1 turns off both, but QEMU does not accept a write that clears `F`, so we could not test this rule. The original value was restored after the test. The spec also says that if turning off `C` would raise IALIGN to 32 bits and the next instruction is not aligned, the write is suppressed (the same root as the alignment issue in part 21).

## 5. Identification CSRs

The five M-mode information CSRs are all read-only and live at `0xF11` to `0xF15` (spec 3.1.2 to 3.1.5). This is what QEMU read.

| CSR (address) | Value | Meaning |
|---|---|---|
| `mvendorid` (`0xF11`) | 0 | JEDEC manufacturer ID; 0 means non-commercial or not implemented |
| `marchid` (`0xF12`) | `0x2a` | microarchitecture ID |
| `mimpid` (`0xF13`) | 0 | implementation version |
| `mhartid` (`0xF14`) | 0 | hart number (one must be 0) |

The fifth, `mconfigptr` (`0xF15`), points to a configuration structure. In QEMU it was readable in M-mode, and writing the value back was an illegal instruction (cause 2) because it is read-only.

`marchid` is `0x2a` (42 in decimal): its top bit is 0 and it is nonzero, which matches the spec's form for open-source projects, allocated by RISC-V International. Commercial designs must have the top bit set. Boot code lets only the hart whose `mhartid` is 0 continue initialization. The fact that these five registers must be readable but may be 0 matters in design: you can comply at almost no cost. As for reset values, on RV64 `mstatus` was `0x0000000a00000000` (`UXL`=`SXL`=2), and `mie`, `medeleg`, `mcounteren` and `satp` were 0. The spec defines only some of the state right after reset (such as `mstatus.MIE` and `MPRV`), and the rest of the hart state is UNSPECIFIED (3.4 Reset). What value to start with is up to the platform and the implementation.

## 6. Map of the main CSRs

Only the CSRs you meet most often, from the spec's CSR listing tables, with the part of this series that covers each.

| Block | CSR (address) | Part |
|---|---|---|
| M information | `mvendorid` F11, `marchid` F12, `mimpid` F13, `mhartid` F14, `mconfigptr` F15 | 23 |
| M setup | `mstatus` 300, `misa` 301, `medeleg` 302, `mideleg` 303, `mie` 304, `mtvec` 305, `mcounteren` 306 | 23, 25, 26 |
| M trap handling | `mscratch` 340, `mepc` 341, `mcause` 342, `mtval` 343, `mip` 344 | 25 |
| M counters | `mcycle` B00, `minstret` B02, `mhpmcounter3`.. B03.., `mcountinhibit` 320 | 24 |
| M memory protection | `pmpcfg0`.. 3A0.., `pmpaddr0`.. 3B0.. | 29 |
| S setup and handling | `sstatus` 100, `sie` 104, `stvec` 105, `scounteren` 106, `sscratch` 140, `sepc` 141, `scause` 142, `stval` 143, `sip` 144 | 26 |
| S translation | `satp` 180 | 30, 31 |
| Unprivileged | `fflags` 001, `frm` 002, `fcsr` 003, `cycle` C00, `time` C01, `instret` C02 | 10, 12 |

One structure stands out. `sstatus`, `sie` and `sip` in S are **restricted views** of `mstatus`, `mie` and `mip` in M: they point to the same bits but show only the fields S may see (spec 3.1.6). `cycle` is a read-only **shadow** of `mcycle`. It is read-only yet must be readable from lower modes, so it sits at a separate address (`0xC00`), and the spec notes that counters are currently the only such shadows. These two become the requirement "several addresses see one piece of register state" in CSR file design (part 24).

## 7. Design checklist: privilege modes and CSRs

| Item | Rule |
|---|---|
| Mandatory mode | only M. One, two or three modes (M, M+U, M+S+U) |
| Mode indicator | no CSR reads the current mode; the implementation keeps 2 internal bits |
| `MPP` | WARL; holds only implemented modes; does not hold reserved 2 |
| Address check | illegal unless current mode >= `csr[9:8]`; illegal if a write is attempted with `csr[11:10]`==11 |
| Non-existent CSR | read and write are both illegal instructions |
| Extra blocking | counters add `mcounteren` and `scounteren` on top of the address rule |
| WARL illegal value | no exception; read gives a legal value; deterministic from the written value |
| WLRL illegal value | may raise an exception; the read value may be anything if deterministic |
| WPRI | read-only 0 if not implemented; software preserves it |
| Multi-field WARL | if one CSR changes another CSR's legal range, that field takes an UNSPECIFIED value in the new range (2.4) |
| `misa` | read-only 0 and a read-only value are both allowed; check the alignment rule when clearing `C` |
| Five ID CSRs | must be readable but may be 0; one `mhartid` must be 0 |
| Shadows and views | `sstatus` is a partial view of `mstatus`; `cycle` shadows `mcycle` |
| Privileged instruction violation | `mret`, `sret` and `sfence.vma` in a lower mode are illegal instructions (cause 2) |

## Summary

- There are three privilege modes, U, S and M, and only M is mandatory. On QEMU, `ecall` in each mode left `mcause` 8, 9 and 11, and writing and reading `MPP` told us which modes were implemented.
- The rule that `[11:10]` is read-only and `[9:8]` is the lowest privilege predicted every one of the 67 cells for existing CSRs on QEMU. Only 12 cells of non-existent CSRs and 6 cells of counter-enable bits fell outside the rule.
- A WARL field reads back a legal value without an exception, and what to keep is up to the implementation. QEMU ignored the whole write for `mtvec.MODE` 2 and 3 and for unsupported `satp.MODE`.
- Identification CSRs must be readable but may be 0. A structure where several addresses see one state, like `sstatus` and `cycle`, becomes a requirement of CSR file design.

**Next:** Part 24 builds the CSR file in hardware: how to handle CSRs whose reads and writes have side effects, how to build the permission check circuit from the address rule above, and how counters and `mcountinhibit` fit together.

## References

- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. 1.2 Privilege levels, 2.1 CSR address mapping conventions, 2.3 Field specifications (WPRI, WLRL, WARL), 3.1.1 `misa`, 3.1.2 to 3.1.5 `mvendorid`, `marchid`, `mimpid`, `mhartid`, 3.1.6 `mstatus`, 3.1.7 `mtvec`, 3.1.11 `mcounteren`, 3.1.15 `mcause`, 3.4 Reset, chapter 4 `satp`. https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 7 Zicsr.
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 10 (privileged architecture)
- QEMU documentation: `virt` board and RISC-V CPU options. https://www.qemu.org/docs/master/system/target-riscv.html
