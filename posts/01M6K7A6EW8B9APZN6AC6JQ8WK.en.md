---
id: "01M6K7A6EW8B9APZN6AC6JQ8WK"
title_en: "Assembly and the ABI: Registers, Calling Convention, Stack Frames, ELF"
summary_en: "Part 6 of the RISC-V series. I follow the psABI calling convention and stack frame through real C functions. The work() frame was 80 bytes, and arguments from the 9th onward went on the stack. I checked argument locations per ABI and relocations, and linker relaxation shrank .text from 800 bytes to 712 bytes."
---
:::note
**Key points in three lines**
- The hardware treats `x1`~`x31` identically, but software assigns names and roles through a convention called the psABI. Arguments go in `a0`~`a7`, return values in `a0`·`a1`, `s0`~`s11` must be preserved by the function, and `sp` is always 16-byte aligned. The frame of a real C function, `work()`, was 80 bytes, and `sp` was `0x800043b0`.
- Arguments from the 9th onward go on the stack at `0(sp)`·`4(sp)`, and a 16-byte struct comes back through a hidden pointer (`a0`) on RV32. `ilp32`·`ilp32f`·`ilp32d`·`lp64d` pass the same `double` argument differently, as an integer register pair, a single integer register, or an FP register, and the linker refused to link a mix.
- `call`·`la` in an object file are `auipc` pairs with blanks (relocations). The linker fills in the distance and shrinks the pair when the target is close (relaxation). `.text` of the same program shrank from 800 bytes to 712 bytes. I cross-checked 18 execution and disassembly results in Python, and all matched.
:::

Parts 2 to 5 looked at instructions one by one. But when verifying a core, you more often need to read things like "why does this function lower `sp` by 80 here" or "what is this value in `a0`" than any single instruction. To chase a bug with the simulation waveform and the disassembly side by side, you have to know how the compiler uses registers and the stack. Part 6 covers that convention: assembly syntax, the ABI (application binary interface), and how ELF files get linked.

The experiments use RV32I (`-march=rv32i -mabi=ilp32 -O1`) by default, with RV32IMAFD and RV64IMAFD for the ABI comparison. Everything runs on QEMU 11.1.3 `virt`, and the tools are `riscv64-elf-gcc`·`objdump`·`readelf` 16.2.

## 1. Register Names and Roles

The ABI names seen in part 2 take on meaning here. The integer calling convention of the RISC-V psABI (the ELF psABI specification) is as follows.

| Register | Name | Preserved across calls? | Use |
|---|---|---|---|
| `x0` | `zero` | — | Always 0 |
| `x1` | `ra` | No | Return address |
| `x2` | `sp` | Yes | Stack pointer |
| `x3`, `x4` | `gp`, `tp` | — | Global and thread pointers |
| `x5`~`x7`, `x28`~`x31` | `t0`~`t6` | No | Temporaries |
| `x8` | `s0` / `fp` | Yes | Saved value, frame pointer |
| `x9`, `x18`~`x27` | `s1`~`s11` | Yes | Saved values |
| `x10`~`x17` | `a0`~`a7` | No | Arguments; `a0`·`a1` are return values |

"Preserved (callee-saved)" means a function that wants to use the register must save its original value and restore it before returning. "Not preserved (caller-saved)" means the called function may overwrite it freely, so the caller must save it if it needs it. Note that `ra` is not preserved. The moment a function makes a call, `jal` overwrites `ra`, so a function that calls other functions saves its own `ra` on the stack.

`gp` and `tp` are not available for general allocation. `gp` is the global pointer used for linker relaxation (Section 5), and `tp` points to thread-local storage. The register numbering has a reason too. The argument and saved registers are clustered in `x8`~`x15`, and those eight are exactly the registers that 16-bit C-extension instructions can name with 3 bits (part 11).

## 2. Stack Frames: Following a Real Function

Start with the rules. The stack grows from high addresses to low addresses. On function entry, `sp` must be **16-byte aligned** (psABI). So the frame size a function allocates is also a multiple of 16.

Here is the test function. It calls another function inside a loop, so `x` and `i` must survive across calls; it has a local array; and it calls a function with 10 arguments.

```c
int work(int x) {
  int buf[4];
  for (int i = 0; i < 4; i++)
    buf[i] = mkpair(x + i).b;
  return sum10(buf[0], buf[1], buf[2], buf[3],
               x, x, x, x, 1, 2);
}
```

Here are the prologue and epilogue GCC generated.

```text
work:
  addi sp,sp,-80      # 80-byte frame
  sw   ra,76(sp)      # return address
  sw   s0,72(sp)      # 5 saved registers to be used
  sw   s1,68(sp)
  sw   s2,64(sp)
  sw   s3,60(sp)
  sw   s4,56(sp)
  ...                 # body
  lw   ra,76(sp)      # epilogue: restore in reverse
  lw   s0,72(sp)
  ...
  addi sp,sp,80
  ret
```

![The stack frame of work() is 80 bytes: from the top, saved ra and s0~s4 (24 bytes), buf[4], and the 9th and 10th arguments at the bottom; sp was 0x800043b0|full](/img/01M6K7A6EW8B9APZN6AC6JQ8WK/09a513274598.webp)

I ran it on QEMU and printed the real addresses. When `main` called `work`, `sp` was `0x80004400`; inside `work` it was `0x800043b0` (80 lower). Both are multiples of 16. The address of `buf` was `0x800043c8`, which is `sp + 24`. The compiler kept `x` and the loop variable in `s1`·`s2`, and the start and end addresses of `buf` in `s3`·`s4`. These values survive even when the `mkpair` call overwrites the `a` and `t` registers. The price is saving five registers in the prologue.

With the frame pointer enabled (`-fno-omit-frame-pointer`), the prologue gains an `addi s0,sp,80`. `s0` (=`fp`) then points to the `sp` at function entry, so a debugger can walk back up the call chain by following `fp`. By default (`-O1` and above), `fp` is not used, and `s0` is just another saved register.

## 3. Many Arguments, and Returning Structs

**Nine or more arguments.** Only 8 integer arguments, `a0` through `a7`, go in registers. The rest are placed by the caller at the bottom of its own frame, in order starting at `0(sp)`. Those are the two bottom slots in the figure above. `work` did `sw a5,0(sp)` (i=1) and `sw a5,4(sp)` (j=2) and then called `sum10`. `sum10` has no frame, so its `sp` is the caller's `sp`, and it read them with `lw a5,0(sp)`·`lw a5,4(sp)`. The result of `sum10(1, …, 10)`, `0x000a0924`, matched 1+…+8 + (9≪8) + (10≪16) = 657,700 computed in Python.

**Returning structs.** The psABI rule depends on size.

| Return value | RV32 (ilp32) | RV64 (lp64) |
|---|---|---|
| `struct {int a, b;}` 8 bytes | `a0`·`a1` | Packed into `a0` alone |
| `struct {int v[4];}` 16 bytes | Hidden pointer | `a0`·`a1` |
| `struct {float x, y;}` | `a0`·`a1` | Packed into `a0` alone |
| Same, with `ilp32f`·`lp64d` | `fa0`·`fa1` | `fa0`·`fa1` |

Values up to 2×XLEN come back in two registers. Anything larger is handled by having **the caller set aside memory and pass its address in `a0` as a hidden first argument.** On RV32, 16 bytes exceeds 2×XLEN (8 bytes). So `mkquad(100)` writes to where `a0` points, as in `sw a1,0(a0)`, `sw a4,4(a0)` …, and the original first argument `x` is pushed to `a1`. The caller, `main`, passed a slot inside its own frame with `addi a0,sp,24`. On RV64, the same 16 bytes is exactly 2×XLEN, so it came back in `a0`·`a1`, two fields packed into each (combining two 32-bit values with `slli`·`or`). At run time, `mkpair(40)` returned (40, 41) and `mkquad(100).v[3]` was 103.

## 4. ABI Names: `ilp32`, `ilp32f`, `lp64d`

A `-mabi` name has two parts. The first part is the size of C types. `ilp32` means int, long, and pointer are 32 bits; `lp64` means long and pointer are 64 bits (int is 32 bits). Checking with the compiler, `sizeof(long)` and `sizeof(void *)` were 4 under `ilp32` and 8 under `lp64`. The trailing letter says where floating-point arguments are passed. With none, they go in integer registers (soft-float ABI); `f` puts up to `float` in FP registers; `d` puts up to `double` in FP registers.

I compiled the same function, `double mix(double x, float y, int z)`, for the same hardware with an FPU (`rv32imafd`, `rv64imafd`), changing only the ABI.

| ABI | `x` (double) | `y` (float) | `z` (int) |
|---|---|---|---|
| `ilp32` | `a0`·`a1` | `a2` | `a3` |
| `ilp32f` | `a0`·`a1` | `fa0` | `a2` |
| `ilp32d` | `fa0` | `fa1` | `a0` |
| `lp64` | `a0` | `a1` | `a2` |
| `lp64d` | `fa0` | `fa1` | `a0` |

The instruction count of `mix` was 13 for `ilp32`, 12 for `ilp32f`, 4 for `ilp32d`, 7 for `lp64`, and 4 for `lp64d`.

Under `ilp32`, a `double` arrives in two integer registers even when there is an FPU. The function writes it to the stack, reads it back with `fld` to move it into an FP register, and returns the result the same way. The computation is a single `fmadd.d`, yet it took 13 instructions. `ilp32d` and `lp64d` take 4. Integer and FP registers are numbered separately: under `ilp32d`, `z` is the first integer argument, so it goes in `a0`.

The ABI is recorded in the object file header (`e_flags` in the ELF header). `readelf -h` showed `0x0` for `ilp32`·`lp64`, `0x2, single-float ABI` for `ilp32f`, and `0x4, double-float ABI` for `ilp32d`·`lp64d`. The linker will not mix files with different values. Linking an `ilp32` object together with an `ilp32d` object stopped with this error.

```text
can't link double-float modules
with soft-float modules
```

If one side believes the same `double` is in `a0`·`a1` and the other believes it is in `fa0`, the value gets passed incorrectly. The standard combination used by Linux distributions is `rv64gc` + `lp64d`.

## 5. Pseudo-Instructions and ELF Relocations

### How Pseudo-Instructions Expand

I collected pseudo-instructions in an assembly file and looked at the real instructions with alias-free disassembly (`objdump -M no-aliases`).

```text
nop            addi zero,zero,0
li  a0,0x12345678
               lui a0,0x12345 + addi a0,a0,1656
la  a1,buf     auipc a1,0 + addi a1,a1,0  (relocation)
mv  a3,a0      addi a3,a0,0
not a4,a0      xori a4,a0,-1
neg a5,a0      sub a5,zero,a0
seqz a6,a0     sltiu a6,a0,1
beqz a0,L      beq a0,zero,L
bgt a0,a1,L    blt a1,a0,L
ble a0,a1,L    bge a1,a0,L
j   L          jal zero,L
call g         auipc ra,0 + jalr ra,0(ra)
tail g         auipc t1,0 + jalr zero,0(t1)
jr  a0         jalr zero,0(a0)
jalr a0        jalr ra,0(a0)
ret            jalr zero,0(ra)
```

Note that `tail` uses `t1`. A tail call must not touch `ra` (it has to pass along the caller's return address unchanged), so it builds the address in a temporary register. Outside position-independent code (`-fPIC`), `la` is the same `auipc` + `addi` as `lla`. Under PIC, it becomes `auipc` + `lw`, which reads the address from the global offset table (GOT).

### Relocations: Blanks Left by the Assembler

The immediate of `auipc a1,0` is 0 because the assembler does not know the final address of `buf`. Instead it leaves a **relocation** record that says "fill in this slot this way at link time." I counted the relocations in the `abi.c` object file by type.

| Relocation | medlow | medany | What it fills |
|---|---|---|---|
| `R_RISCV_HI20` / `LO12_I` | 14 / 15 | — | Upper/lower parts of an absolute address (`lui` + `addi`) |
| `R_RISCV_PCREL_HI20` / `LO12_I` | — | 15 / 15 | PC-relative distance (`auipc` + `addi`) |
| `R_RISCV_CALL_PLT` | 19 | 19 | `auipc` + `jalr` call pair |
| `R_RISCV_BRANCH` | 4 | 4 | 13-bit branch distance |
| `R_RISCV_RELAX` | 48 | 49 | "May be shortened" marker |

The code model (`-mcmodel`) chooses how addresses are built. `medlow` builds absolute addresses within ±2 GiB of address 0 with `lui`; `medany` builds addresses within ±2 GiB of the current pc with `auipc`. The linker reads these records and does the `%hi`·`%lo` and `%pcrel_hi`·`%pcrel_lo` arithmetic from part 3 for you.

`PCREL_LO12_I` has a trap. This relocation points not at `g_small` but at the location of its paired `auipc` (the local label `.L0`). The low 12 bits are the lower part of "the distance from the `auipc`'s pc to the target," so the reference must be the address of the `auipc`, not the `addi`.

### Linker Relaxation: Shrink When in Reach

A pair marked with `R_RISCV_RELAX` may be rewritten shorter by the linker. Once it knows the actual distance, the linker turns a `call` into a single `jal` if the target is within ±1 MiB, and if an address is within ±2 KiB of `gp`, it deletes the `auipc` and turns the pair into a single `gp`-relative access.

![The auipc+jalr call in the object file became a single jal after linking, and the auipc+lw global variable read became a single lw a1,-1791(gp), shrinking .text from 800 bytes to 712 bytes|full](/img/01M6K7A6EW8B9APZN6AC6JQ8WK/1b28151ac353.webp)

To see `gp` relaxation, the linker script must define `__global_pointer$` and the startup code must set `gp`. The shared startup code lacks this, so in this part's folder I wrote a separate linker script, adding `__global_pointer$` the same way as the GNU ld default script, plus startup code. The `la gp, __global_pointer$` in the startup code is wrapped in `.option norelax`, because relaxing the instruction that sets up `gp` into a `gp`-relative access would make it reference itself.

I built with `medany` and compared `--no-relax` against the default (relaxed). `.text` shrank from 800 bytes to 712 bytes, `auipc` went from 37 to 15, and `jal ra` went from 0 to 20.

All 20 calls became `jal`, and the read and the address computation of the small global variable `g_small` became `lw a1,-1791(gp)`·`addi a1,gp,-1791`. The 88 bytes saved are exactly 22 instructions (4 bytes each). Building with `medlow` gave a similar reduction, from 804 bytes to 720 bytes. The 256-byte `g_big` was 2,047 bytes from `gp`, numerically within range, yet its `auipc` remained. The linker appears to leave a margin for alignment when deciding (my guess).

Relaxation matters to hardware designers for a reason: the disassembly of the source or object file differs from the instructions in the ELF actually loaded into memory. When matching against a simulation log, always disassemble the **linked ELF**.

### Sections

The sections of the linked ELF were `.text` (code, `0x80000000`), `.rodata` (string constants), `.data` (256 bytes of initialized globals), `.sdata` (4 bytes of small globals), plus `.riscv.attributes`·`.symtab` and others that are not loaded into memory. `.riscv.attributes` holds the canonical ISA name seen in part 1 (`Tag_RISCV_arch`). `.sdata`·`.sbss` exist to gather small globals near `gp` so the relaxation above works well.

## 6. Hands-On: Cross-Checking Execution and Disassembly

`check.py` does the build, the QEMU run, and the objdump/readelf collection in one go. The key check reconstructs addresses directly from the machine code of the linked ELF. It finds an `auipc` followed immediately by an `addi`·`lw`, computes `pc + (imm20 ≪ 12) + imm12`, computes `gp + imm12` for `gp`-relative accesses, and checks that they match the addresses in the symbol table.

```python
for (a, w, _), (a2, w2, _) in zip(I, I[1:]):
    # auipc rd + (addi|lw) rd: reconstruct pc-relative address
    if w & 0x7f == 0x17 and a2 == a + 4 \
       and (w2 >> 15) & 31 == (w >> 7) & 31:
        got.append(a + sx(w & 0xfffff000, 32)
                   + sx(w2 >> 20, 12))
    # rs1 == gp(x3): reconstruct gp-relative address
    if w & 0x7f in (0x13, 0x03) and (w >> 15) & 31 == 3:
        got.append(gp + sx(w >> 20, 12))
```

| Test | Count | Matched |
|---|---|---|
| Run-time values (10 arguments, struct return, `work`) | 5 | 5 |
| 16-byte alignment of `sp` and frames, location of `buf` | 4 | 4 |
| Global addresses printed by QEMU = symbol table | 2 | 2 |
| Addresses reconstructed from machine code (relaxation on/off) | 2 | 2 |
| ABI: type sizes, hidden pointer, link refusal, `fp` | 5 | 5 |

18 in total, all matched. The machine-code reconstruction check also means the linker correctly applied the +1 adjustment for `%pcrel_hi`·`%pcrel_lo` (part 3) and the `gp` relaxation.

## 7. Design Checklist: Using the ABI from the Hardware Side

The ABI is a software convention, but people who verify and debug cores read it every day.

| Item | Rule |
|---|---|
| Register file | The hardware treats `x1`~`x31` identically. ABI names are just the disassembler's notation |
| Return address | `ra` is the pc+4 written by `jal`/`jalr`. The predictor's RAS hints are `x1`·`x5` (part 4) |
| Stack alignment | `sp` is 16-byte aligned on function entry. A misalignment in simulation signals stack corruption |
| Trap handler | Interrupts can arrive anywhere, so even caller-saved registers must all be saved |
| `gp` setup | Boot code sets `gp` first under `.option norelax`. Otherwise relaxed global accesses go to the wrong address |
| Linked ELF | Relaxation makes the instructions differ from the `.o`. Compare logs against the linked ELF |
| ABI check | Flags from `readelf -h` (`0x0`·`0x2`·`0x4`, plus `0x1` for RVC) and `Tag_RISCV_arch` from `-A` |
| Aliases off | `objdump -M no-aliases` shows the instructions the decoder actually receives |
| Environment calls | SBI puts the extension ID in `a7`, the function ID in `a6`, arguments in `a0`~`a5`, and returns in `a0`·`a1` (part 36) |
| `tp`·`gp` | The compiler does not use them for ordinary values. Only runtime code changes these two |

The trap handler row connects to the hands-on in part 4. The handler in part 4 was built with GCC's `interrupt("machine")` attribute, and its disassembly showed the prologue saving 17 registers: `ra`, `t0`~`t6`, `a0`~`a7`, and `s0`. An ordinary function does not save the `t` and `a` registers. A trap cuts in somewhere other than a function call boundary, and the interrupted code may be holding values in those registers.

## Summary

- `a0`~`a7` are arguments (`a0`·`a1` are return values), `s0`~`s11` and `sp` are preserved, and `t`, `a`, and `ra` are not. The real `work()` allocated an 80-byte frame that saved `ra` and `s0`~`s4`, and `sp` was a multiple of 16.
- Arguments from the 9th onward go at the caller's `0(sp)` onward, and structs larger than 2×XLEN come back through memory provided by the caller (`a0`). The same 16-byte struct was returned in memory on RV32 and in `a0`·`a1` on RV64.
- An ABI name encodes type sizes (`ilp32`·`lp64`) and FP argument registers (`f`·`d`). The same `mix()` took 13 instructions under `ilp32` and 4 under `ilp32d`, and linking different FP ABIs was refused.
- `call`·`la` in an object file are `auipc` pairs with relocations. The linker fills in the distance and, when in reach, shrinks them to `jal` or a `gp`-relative access. `.text` went from 800 bytes to 712 bytes.

**Next:** Part 7 looks at the M extension. Running on QEMU, I check what `mul`·`mulh`·`mulhu`·`mulhsu` and the `div`·`rem` family compute, what value division by zero returns without raising an exception, and how the overflow of dividing the most negative number by −1 is defined. I also revisit how `utoa10` calling `__udivsi3` in part 1 changes with the M extension.

## References

- RISC-V ELF psABI Specification (integer and FP calling conventions, struct passing, stack alignment, ELF e_flags, relocations and relaxation). https://github.com/riscv-non-isa/riscv-elf-psabi-doc
- RISC-V Assembly Programmer's Manual (pseudo-instructions, `.option norelax`, `%pcrel_hi`·`%pcrel_lo`). https://github.com/riscv-non-isa/riscv-asm-manual
- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 2.1 Programmers' Model, 2.5 Control Transfer Instructions (`x1`·`x5` link registers)
- RISC-V Supervisor Binary Interface Specification (SBI calling convention). https://github.com/riscv-non-isa/riscv-sbi-doc
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 3, RISC-V Assembly Language
- GCC RISC-V options (`-mabi`, `-mcmodel`, `-mno-relax`). https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html
