---
id: "01M6K7VC2EEK6PWVQVRNAH0480"
title_en: "F and D Extensions: FP Registers, Rounding, fcsr Flags, NaN Boxing"
summary_en: "Part 10 of the RISC-V series. I ran 229 instructions on QEMU covering the F and D extensions' rounding modes and exception flags, the canonical NaN, NaN boxing, and saturation rules for out-of-range integer conversion, and confirmed that all of them matched an exact-fraction reference model."
---
:::note
**Key points in three lines**
- The F and D extensions add the floating-point registers `f0`–`f31` and one control register, `fcsr`. `fcsr` holds a 3-bit rounding mode (`frm`) and 5 exception flag bits (`fflags`); division by zero or overflow just sets a flag, with no trap.
- RISC-V pins down the parts IEEE 754 leaves to the implementation. A NaN result is always the canonical NaN `0x7fc00000`, out-of-range integer conversions produce saturated values, and a single-precision value held in a 64-bit register must have all upper 32 bits set to 1 (NaN boxing).
- I executed 229 instructions on QEMU, printed the result bits and flags, and compared them against a Python reference model I wrote myself using exact rationals (Fraction). All 229 matched.
:::

The earlier posts covered the integer ISA (RV32I and RV64I), the M and A extensions, and the memory model. Part 10 covers the last two letters in G: the F (single-precision) and D (double-precision) extensions. The arithmetic itself follows IEEE 754, so instead of re-explaining the standard, this post covers only the rules RISC-V adds on top of it. These are the details that will fail compliance tests if you get even one wrong when building an FPU in RTL.

The experiments used the same QEMU 11.1.3 `virt` board as part 3, with `rv64imafd`. The machine was a Mac M1 Max, and the compiler was `riscv64-elf-gcc` 16.2.

## 1. Registers and Enabling the FPU

The F extension adds 32 floating-point registers (`f0`–`f31`) of 32 bits each, and the D extension widens them to 64 bits. The register width is called FLEN: FLEN=32 with F alone, 64 with D. It is set independently of XLEN (the integer register width), so adding D to RV32 gives 32-bit integer registers but 64-bit `f` registers.

Unlike `x0` on the integer side, `f0` is an ordinary register.

| Part | Contents |
|---|---|
| `f0`–`f31` | 32 registers of FLEN bits, no special registers |
| `fcsr` (CSR `0x003`) | `frm`[7:5] + `fflags`[4:0] |
| `frm` (CSR `0x002`) | Dynamic rounding mode |
| `fflags` (CSR `0x001`) | 5 accrued exception flags |

On bare metal, the FPU starts out disabled. If `mstatus.FS` (bits 14:13) from the privileged specification is 0 (Off), every floating-point instruction and every `fcsr` access raises an illegal instruction exception. So I copied the startup code and added two lines.

```asm
    li   t0, 0x2000      # mstatus.FS = 01 (Initial)
    csrs mstatus, t0
    csrwi fcsr, 0        # frm = RNE, fflags = 0
```

FS has four states: Off, Initial, Clean, and Dirty. Changing an `f` register or `fcsr` makes the hardware raise it to Dirty. The operating system looks at this to decide whether to save the `f` registers on a context switch (parts 23 and 24).

## 2. fcsr: Rounding Modes and Exception Flags

![fcsr consists of 3 frm bits and 5 flag bits (NV, DZ, OF, UF, NX), and a single-precision value in an FLEN=64 register is read as a valid input only if its upper 32 bits are all 1|full](/img/01M6K7VC2EEK6PWVQVRNAH0480/8869339357dd.webp)

Instructions that need rounding have a 3-bit rm field (in the funct3 position of integer instructions). Values 0–4 are the five rounding modes, and 111 (DYN) means "use the mode written in `frm`." 101 and 110 are reserved. If you omit rm, the assembler inserts `dyn`.

- **RNE** (Round to Nearest, ties to Even): to the nearest value; on an exact tie, toward the side whose last bit is 0 (even). The IEEE 754 default.
- **RTZ** (toward Zero): truncation. C's `(int)x` conversion does this.
- **RDN** (toward −∞), **RUP** (toward +∞): the lower and upper bounds in interval arithmetic.
- **RMM** (to Nearest, ties to Max Magnitude): to the nearest value; on a tie, away from zero. The rounding taught in school.

The difference shows up most clearly in integer conversion. I ran `fcvt.w.s` under all five modes on QEMU.

![Running fcvt.w.s under five modes: 2.5 becomes 2 under RNE and 3 under RMM and RUP, and -0.5 becomes -1 only under RDN and RMM|xl](/img/01M6K7VC2EEK6PWVQVRNAH0480/2c3692dd9de2.webp)

RNE and RMM differ only on ties (x.5). 2.5 gives 2 (even) under RNE and 3 under RMM, while 1.5 gives 2 under both.

In ordinary arithmetic, the mode changes the last bit of the result. Near 1, single-precision spacing is 2⁻²³, so 1 + 2⁻²⁴ sits exactly halfway between 1 (`0x3f800000`) and 1 + 2⁻²³ (`0x3f800001`). RNE picks `0x3f800000`, whose last bit is 0, and RMM picks `0x3f800001`, which is farther from zero.

| Operation | RNE | RTZ/RDN | RUP/RMM |
|---|---|---|---|
| 1 / 3 | `3eaaaaab` | `3eaaaaaa` | `3eaaaaab` |
| 1 + 2⁻²⁴ (tie) | `3f800000` | `3f800000` | `3f800001` |
| Max value × 2 | +∞ | Max value | +∞ |

The last row (adding the largest finite value to itself) is easy to miss. Overflow does not always produce infinity. RTZ, and RDN for positive values, never round toward infinity, so the result is the largest finite value `0x7f7fffff`; the flags were OF and NX under all five modes. Running `fdiv.s ...,dyn` while changing `frm` with `fsrm` gave the same results as the static modes.

## 3. Exceptions Are Flags, Not Traps

IEEE 754 defines five exceptions and allows implementations to trap on them. RISC-V does not trap on floating-point exceptions. It only sets flags and uses the defined default result. The flags accrue: once set, they stay until software clears `fflags`.

| Operation (single precision, RNE) | Result | Flags |
|---|---|---|
| 1 / 0 | +∞ `7f800000` | DZ |
| 0 / 0, √−1, ∞ − ∞ | `7fc00000` | NV |
| 2¹²⁷ × 10 | +∞ | OF NX |
| 10⁻³⁰ × 10⁻³⁰ | +0 | UF NX |
| 2⁻¹²⁶ × 0.5 | `00400000` | None |
| 1 + 2⁻³⁰, √2 | Rounded value | NX |

The result in the fifth row, 2⁻¹²⁷, is a subnormal, but it is represented exactly, so UF is not set. UF is set only when the result is tiny **and** inexact. Doing the same multiplication with `0x00800001`, whose last bit is 1, causes rounding, and UF and NX were set. Tininess is judged **after** rounding (tininess after rounding): if the result, rounded as though the exponent range were unbounded, is at least 2⁻¹²⁶, it is not tiny.

The specification leaves it to software to check the flags directly when needed, and it also left out instructions that branch on flags to keep the ISA simple. On the design side, the FPU needs no precise exception path; it only has to update the flags in order.

After `fsrmi 3` (RUP), I ran 1/0 and read `fcsr` with `frcsr`: `0x68` (`frm`=011, DZ).

## 4. Canonical NaN: The Payload Is Not Propagated

A NaN has an all-ones exponent and a nonzero mantissa. If the top mantissa bit is 1 it is a quiet NaN (qNaN); if 0, a signaling NaN (sNaN). The remaining bits are the payload. IEEE 754 only recommends propagating the payload into the result; it does not require it.

RISC-V does not propagate it. If an operation's result is NaN, it is always the **canonical NaN**: `0x7fc00000` for single precision and `0x7ff8000000000000` for double. Sign 0, quiet, payload 0.

| Input | Operation | Result | Flags |
|---|---|---|---|
| qNaN `7fc12345` | `fadd.s` +1 | `7fc00000` | None |
| sNaN `7f800001` | `fadd.s` +1 | `7fc00000` | NV |
| Negative qNaN `ffc00000` | `fadd.s` +1 | `7fc00000` | None |
| qNaN `7fc12345` | `fsgnjn.s` (negate) | `ffc12345` | None |

The last row is the exception. Sign injection (`fsgnj`, `fsgnjn`, `fsgnjx`, and the pseudo-instructions `fmv.s`, `fneg.s`, `fabs.s`) is bit manipulation, so it leaves a NaN untouched and only changes the sign, with no NV even for an sNaN.

Thanks to the canonical NaN rule, there is no need for a mux to pick and forward an input NaN; the hardware just outputs a single constant.

Comparison instructions come in quiet and signaling varieties. `feq.s` returns 0 for a qNaN input without setting a flag (NV only for an sNaN). `flt.s` and `fle.s` set NV even for a qNaN. On QEMU too, `feq.s` on two qNaNs gave 0 with no flag, and `flt.s` on a qNaN and 1 gave 0 with NV. `fclass.s` sets one bit out of 10 classes (I checked sNaN `0x100`, qNaN `0x200`, and negative subnormal `0x4`).

## 5. NaN Boxing: Single-Precision Values in 64-Bit Registers

With D, the `f` registers are 64 bits. Which part do single-precision instructions use? RISC-V puts the single-precision value in the lower 32 bits and fills the upper 32 bits with **all 1s**. This is called NaN boxing. Read as a double, this value is a NaN, so even if it is misread it does not turn into a bogus number.

There are two rules. When writing a single-precision result, fill the upper bits with 1s. When a single-precision operation reads an input, it checks the upper 32 bits, and if they are not all 1s, it treats that input as the canonical NaN. The lower half of the figure above shows this, and the QEMU results were as follows.

| Input register (64-bit) | Instruction | Result |
|---|---|---|
| `ffffffff40000000` | `fadd.s` with itself | `ffffffff40800000` |
| `0000000040000000` | `fadd.s` with itself | `ffffffff7fc00000` |
| `3ff0000000000000` | `fadd.s` with itself | `ffffffff7fc00000` |
| `0000000040000000` | `fsgnjn.s` | `ffffffffffc00000` |

In the second row, the lower 32 bits are 2.0, yet the result is the canonical NaN. The third row reads a double-precision 1.0 with a single-precision instruction. The fourth row is interesting. I said sign injection doesn't touch NaNs, but an input with broken boxing is first turned into the canonical NaN `7fc00000` and then negated, giving `ffc00000`. Widening it with `fcvt.d.s` produced the double-precision canonical NaN. The canonical NaN is quiet, so none of these set flags.

Some instructions don't check. The specification classifies loads and stores (`flw`, `fsw`) and bit moves (`fmv.w.x`, `fmv.x.w`) as "transfer" instructions. Transfers going in produce a boxed value, and transfers going out ignore the upper 32 bits and take the lower 32 bits as is. `fmv.x.w` on `12345678c0000000` gave `ffffffffc0000000`, which is the lower 32 bits `c0000000` sign-extended into an RV64 integer register.

## 6. Integer Conversion: Out-of-Range Values Saturate

`fcvt.w.s` (single precision → 32-bit integer) cannot represent NaN, infinity, or numbers that are too large. In that case RISC-V produces the nearest endpoint (the saturated value) and sets NV. NaN goes to the positive endpoint.

| Input | `fcvt.w.s` | `fcvt.wu.s` |
|---|---|---|
| NaN, +∞ | `7fffffff` NV | `ffffffff` NV |
| −∞ | `80000000` NV | `0` NV |
| 3×10⁹ | `7fffffff` NV | `b2d05e00` |
| 2³¹ | `7fffffff` NV | `80000000` |
| −2³¹ | `80000000` | `0` NV |
| −1.0 | `ffffffff` | `0` NV |
| −0.5 | `0` NX | `0` NX |

The table is written in 32 bits. Let's look at more boundary values.

- `fcvt.w.s` of 2,147,483,520 (the largest single below 2³¹) gives `7fffff80` with no flags. The single just below −2³¹ (`0xcf000001`) gives `80000000` and NV.
- `fcvt.wu.s` of −0.5 rounds to −0 under RNE and RTZ, which is in range, so only 0 and NX are set. Under RDN it becomes −1, which is out of range: 0 and NV. Whether a value is in range is judged **after rounding**.
- `fcvt.w.d` of 2,147,483,647.0 gives `7fffffff` with no flags; 2,147,483,647.5 rounds to 2³¹ and gives the same bits with NV.
- On RV64, 32-bit results are sign-extended to 64 bits. `fcvt.wu.s` is no exception: the NaN result `ffffffff` went into the register as `ffffffffffffffff`, and 4,294,967,040 as `ffffffffffffff00`.

The other direction (integer → floating point) cannot overflow, but it can run out of precision. 2²⁴ + 1 = 16,777,217 does not fit in the 24-bit single-precision mantissa. `fcvt.s.w` produced 16,777,216 (`4b800000`) with NX under RNE, and 16,777,218 (`4b800001`) under RUP.

## 7. fmin/fmax and the Four fmadd Variants

The rules for `fmin` and `fmax` changed in version 2.2 of F. They now follow IEEE 754-2019's minimumNumber and maximumNumber. If only one input is NaN, the non-NaN one is returned; if both are NaN, the canonical NaN. An sNaN sets NV. And −0 is treated as less than +0.

| Input | `fmin.s` | Flags |
|---|---|---|
| qNaN, 1.0 (either order) | 1.0 | None |
| qNaN, qNaN | `7fc00000` | None |
| −0, +0 | −0 (`fmax.s` gives +0) | None |
| sNaN, 1.0 | 1.0 | NV |

There are four fused multiply-add instructions. They use four opcodes of their own and the R4 format with three source register fields (rs3 in [31:27]).

| Instruction | Computation | Run with 2, 3, 1 |
|---|---|---|
| `fmadd` | a×b + c | 7 |
| `fmsub` | a×b − c | 5 |
| `fnmsub` | −(a×b) + c | −5 |
| `fnmadd` | −(a×b) − c | −7 |

The name `fnmsub` makes sense if you read it as "−(a×b − c)."

The key point of fusion is that rounding happens **only once**. Try a = b = 1 + 2⁻¹² (`0x3f800800`) and c = −(1 + 2⁻¹¹). The exact product is 1 + 2⁻¹¹ + 2⁻²⁴. Multiplying separately with `fmul.s`, the 2⁻²⁴ disappears in rounding (it is halfway to 1 + 2⁻²³, so it goes to the even side), leaving 1 + 2⁻¹¹, and adding c gives 0. `fmadd.s` adds without rounding the product, so it produced exactly 2⁻²⁴ (`0x33800000`). The separate version set NX, and the fused version had an exact result and no flags.

If the product is ∞ × 0, NV must be set even when c is a qNaN. On QEMU too, `fmadd.s`(∞, 0, qNaN) gave the canonical NaN and NV.

## 8. Experiment: QEMU Results Checked Against a Fraction Reference Model

The test program wraps a single instruction in inline assembly, clears `fflags` before running it, and afterwards prints the full 64 bits of the result register and `fflags` over UART.

```c
#define FF(s, a, b, c) do { /* 3 single-precision inputs */  \
  __asm__ volatile("fsflags x0\n\t"               \
    "fmv.w.x ft0,%2\n\tfmv.w.x ft1,%3\n\t"         \
    "fmv.w.x ft2,%4\n\t" s "\n\t"                  \
    "fmv.x.d %0,ft3\n\tfrflags %1"                 \
    : "=&r"(r_), "=&r"(f_)                         \
    : "r"(a_), "r"(b_), "r"(c_)                    \
    : "ft0", "ft1", "ft2", "ft3");                 \
  show(s, a_, b_, c_, r_, f_); } while (0)

FF("fadd.s ft3,ft0,ft1,rmm", ONE, 0x33800000, 0);
FF("fmadd.s ft3,ft0,ft1,ft2", 0x7f800000, 0, QNAN);
```

Because the result is pulled out in full 64 bits with `fmv.x.d`, the boxing of every single-precision result gets checked as well.

I wrote the reference model myself in Python. It decodes the input bits into exact rationals with `fractions.Fraction`, computes the operation exactly, and rounds once per mode (square roots are computed to 400 bits with `math.isqrt`, with the remainder as the sticky bit). The canonical NaN, boxing checks, saturating conversions, and tininess were transcribed straight from the specification's wording. I first validated the model itself against Python `float` (double precision, RNE) on 10,000 random pairs.

| Test | Count | Matched |
|---|---|---|
| Reference model = Python `float` (double-precision add and divide) | 10,000 | 10,000 |
| QEMU = reference model (64-bit result + `fflags`) | 229 | 229 |
| Reserved rm (101), DYN with `frm`=101 | 2 | 2 (illegal instruction, `mcause`=2) |

In the first comparison, a whole batch of single-precision input rows mismatched. That was my mistake: the inputs I printed were the 32-bit values before boxing, so the reference model read them as "broken boxing." I encoded the reserved rm directly with `.insn r OP_FP, 5, 0, ft3, ft0, ft1` and checked it with a small trap handler that records `mcause` and skips the instruction.

## 9. Design Checklist: What an FPU Must Get Right

| Item | Rule |
|---|---|
| FPU off | With `mstatus.FS`=Off, every F/D instruction and every `fcsr` access is an illegal instruction |
| FS update | Changing an `f` register or `fcsr` sets FS=Dirty. When to raise it (on every write, or only on an actual change) is an implementation choice |
| rm field | 101 and 110 are illegal. With DYN, look at `frm`; if `frm` is 101–111, illegal |
| Exceptions | No traps. Flags accrue by OR (software clears them) |
| Overflow result | RTZ, RDN (positive), and RUP (negative) give the largest finite value, not infinity |
| UF | Only when tiny (judged after rounding) and inexact |
| NaN result | Always the canonical NaN. The payload is not propagated |
| Sign injection | Leaves NaNs unchanged and sets no flags. But an input with broken boxing becomes the canonical NaN first |
| NaN boxing | Single-precision results fill the upper bits with 1s. If an input's upper bits are not all 1s, it is the canonical NaN. Transfer instructions are exempt |
| Comparisons | `feq` sets NV only for sNaN; `flt`/`fle` set NV for any NaN |
| `fmin`/`fmax` | The non-NaN input; if both are NaN, the canonical NaN; −0 < +0; NV for sNaN |
| Integer conversion | Range check after rounding. Out of range or NaN gives the saturated value + NV. RV64 sign-extends 32-bit results (both W and WU) |
| Fusion | One rounding. ∞×0 sets NV even if c is a qNaN |
| Subnormals | Must be handled in hardware (there is no flush-to-zero mode) |

The last row affects area. Some ISAs save hardware with a mode that flushes subnormals to zero (flush-to-zero), but RISC-V F and D require subnormals to be handled per IEEE 754, so even a small FPU needs normalization (leading-zero count and shift).

## Summary

- F and D add `f0`–`f31` and `fcsr` (3 `frm` bits + 5 `fflags` bits). The FPU is usable only after `mstatus.FS` is turned on.
- The rounding mode is either fixed in the instruction's rm field or taken from `frm` via DYN. 2.5 gave 2 under RNE and 3 under RMM, and depending on the mode, overflow produced the largest finite value instead of infinity.
- Exceptions only accrue flags, with no traps. A NaN result is always `0x7fc00000`, out-of-range integer conversions saturate, and a single-precision value with FLEN=64 must have all upper 32 bits set to 1.
- All 229 instructions run on QEMU matched the Fraction reference model. I used concrete numbers to pin down easy-to-get-wrong spots like the range check after rounding (`fcvt.wu.s` of −0.5) and the single rounding in fusion (2⁻²⁴ versus 0).

**Next post:** Part 11 is the C extension. It lays out, as a mapping table, the formats of the compressed instructions that shrink frequently used instructions to 16 bits, why the 3-bit register fields map to `x8`–`x15`, and which 32-bit instruction each compressed instruction expands to. I'll cross-check a decompressor I wrote against objdump at scale and measure how often 16-bit instructions are used in real programs.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 20, F extension version 2.2 (20.2 fcsr, 20.3 NaN generation and propagation, 20.4 subnormal arithmetic, 20.6 computational instructions, 20.7 conversion and move instructions, 20.8 compare instructions, 20.9 classify instruction), Chapter 21, D extension (NaN boxing). https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. 3.1.6, the FS field of `mstatus`.
- IEEE Std 754-2019, IEEE Standard for Floating-Point Arithmetic.
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 5 RV32F and RV32D
- QEMU `virt` board documentation. https://www.qemu.org/docs/master/system/riscv/virt.html
