---
id: "01M75HAZ9MMPAAWKECZKWMQGZ0"
title_en: "RISC-V CSR File Design: Permission Checks, Views and Counters Checked Against QEMU"
summary_en: "Part 24 of the RISC-V series. We built the CSR file's permission check, views, WARL cleanup and counters as a cycle model and checked it against QEMU. Only 19 of 349 RV64 cells differed, all counters: QEMU read 1 higher right after a write."
---
:::note
**Key points of this post**
- A CSR file is one path: "address decode and permission check, then the read mux and write-value computation, then WARL cleanup, then storage". Whether an instruction attempts a write must come from the instruction fields (is `rs1` `x0`?), not from the value. For the read-only `mhartid`, `csrrs rd, mhartid, t0` with t0=0 raised an illegal-instruction exception on QEMU too, and only `rs1=x0` or `uimm=0` could read it.
- We built this as a cycle-level Python model and checked it against QEMU (349 cells on RV64, 351 on RV32). The permission matrix, the views (`sstatus`, `sie`) and WARL cleanup all agreed. Only the counters differed. The spec-order model was off by one in 19 RV64 cells, and a model changed to "apply the write first, then count that instruction too" matched all 349.
- The spec says a value written to `minstret` is what the next instruction reads, but QEMU (`-icount shift=0`) read back 1001 after writing 1000. On RV32, `mcycleh` did not advance when the low 32 bits of `mcycle` wrapped (`mcycle` passed `0xffffffff` and became 1 while `mcycleh` stayed 0). The model with spec behavior predicts that reading the two halves separately gives `0x1ffffffff` in the same sequence.
:::

[Part 23](post:01M70HYVSMKS8FPAYNWXMWSDHT) covered what CSRs exist, how their addresses are laid out, and which fields do not read back what you wrote. This post turns those rules into hardware. We follow one CSR instruction through the file, see where the read and write rules attach to the circuit, and see how the counters and `mcountinhibit` interlock every clock. How traps are actually entered and how `mepc` and `mcause` get filled is part 25. Here `mcause` appears only as evidence that an illegal instruction (cause 2) happened.

To check the design choices we wrote a cycle-level model of the CSR file (`csrfile.py`, 193 lines): one hart with M+S+U and F but no H or V, one cycle per instruction. We fed the same scenarios to the model and to QEMU 11.1.3 `virt` (Mac M1 Max, `riscv64-elf-gcc` 16.2) and compared cell by cell. The specification is the unprivileged and privileged manuals, version 20240411. Section 7 has the method and the full results.

## 1. The skeleton of a CSR file

![The path of one CSR instruction. The address, instruction fields and current mode are checked; if they pass, the read mux and the write-value computation read and write the same storage. A failed check raises an illegal-instruction exception|full](/img/01M75HAZ9MMPAAWKECZKWMQGZ0/41a29b655c76.webp)

A CSR instruction reads, modifies and writes in one step (Zicsr, unprivileged spec chapter 7). Inside the file that is six parts.

| Part | What it does |
|---|---|
| Decode | picks the CSR from `csr[11:0]` and flags addresses that do not exist |
| Check | looks at mode, the read-only block and extra conditions to produce "illegal" |
| Read mux | sends the selected CSR's read value (views and shadows included) to `rd` |
| Write value | `rw` takes `rs1`, `rs` takes read value OR `rs1`, `rc` takes read value AND NOT `rs1` |
| WARL cleanup | turns an illegal value into a legal one, or ignores the write |
| Storage | flip-flops; several addresses look at the same storage |

One point first. The "read value" the write computation uses is the value **as seen at that address**. `csrrs t0, sstatus, t1` ORs `t1` into the `sstatus` view, not into all of `mstatus`, and writes that back through `sstatus`. Section 3 shows why this matters.

### Does it read, does it write

Only the instruction fields decide this. The Zicsr table, condensed:

| Instruction | Reads | Writes |
|---|---|---|
| `csrrw`, `csrrwi` | only if `rd`≠`x0` | always |
| `csrrs`, `csrrc` (and i forms) | always | only if `rs1`≠`x0` (`uimm`≠0) |

`csrrw x0, ...` does not read, so no read side effect may happen. `csrrs rd, csr, x0` does not write, so there is no write side effect and no illegal-instruction exception for read-only CSRs. These two rules are why `csrr` (=`csrrs rd, csr, x0`) can read a read-only counter. The spec notes that standard CSRs have no read side effects, so the "`rd=x0` means no read" rule is invisible for standard CSRs today. It matters for custom CSRs, or a device where reading pops a FIFO.

The write side has one trap. What if `rs1` is not `x0` but holds 0? The spec says per-field side effects do not happen, but side effects of writing the whole CSR do. For a read-only CSR the instruction is illegal. A write-attempt signal built from "is the value to write nonzero" would be wrong. This is the "write attempt comes from the instruction field, not the register value" point from part 12.

## 2. The permission check circuit

Turning part 23's address rules into a circuit gives four checks.

1. Does the address exist? Reads and writes of an unimplemented CSR are illegal.
2. mode ≥ `csr[9:8]`? U=00, S=01, M=11, and 10 (the H level) is unused without the H extension, so one 2-bit unsigned comparator does it.
3. A write attempt with `csr[11:10]`=11 is illegal.
4. Extra per-CSR conditions: counter enable bits (`mcounteren`, `scounteren`), `mstatus.TVM` (S-mode `satp` access), `mstatus.FS`=Off (`fcsr` family).

```systemverilog
// Illustrative, not simulated. priv: U=00, S=01, M=11
wire wr_try  = (funct3[1:0] == 2'b01) | (rs1_f != 5'd0);
wire rd_try  = !(funct3[1:0] == 2'b01 && rd_f == 5'd0);
wire priv_ok = (priv >= csr[9:8]);
wire ro_ok   = !(wr_try && csr[11:10] == 2'b11);
wire illegal = csr_en && !(exists && priv_ok && ro_ok && extra_ok);
```

`funct3[1:0]==01` is `csrrw` and `csrrwi`, and `rs1_f` is the 5-bit `rs1` field. In the i forms that field is `uimm`, so one expression covers both. These signals can be built in decode, and when the check says "illegal" the write-enable is never sent to the CSR file, so an illegal instruction cannot change a CSR.

### Checking it: the permission matrix

On QEMU we read (`csrrs`, `rs1=x0`) and wrote 0 to 15 CSRs from M, S and U mode. First with `mcounteren` and `scounteren` both 0.

| CSR | M | S | U |
|---|---|---|---|
| `mstatus`, `mscratch`, `mcycle` | read/write | illegal | illegal |
| `sstatus`, `satp`, `sscratch` | read/write | read/write | illegal |
| `cycle`, `instret`, `time`, `hpmcounter3`, `mhartid` | read | illegal | illegal |
| `hpmcounter31`, `0x600`, `0x7c0`, `0x8ff` | illegal | illegal | illegal |

The last row is the existence check. `hpmcounter31` because this QEMU implements only 16 counters (3 to 18), `0x600` (`hstatus`) because H is off, and `0x7c0`, `0x8ff` because they are in the custom blocks but nothing is there, so even M-mode gets an exception. Reading `cycle` from M-mode works, but writing is illegal because `csr[11:10]`=11.

Counter shadows also look at `mcounteren` and `scounteren`. We repeated the same read with different settings.

| Setup | `cycle`, `instret` | `time` | `hpmcounter3` |
|---|---|---|---|
| S, `mcounteren`=0 | illegal | illegal | illegal |
| S, `mcounteren`=5 | read | illegal | illegal |
| U, `mcounteren`=5, `scounteren`=0 | illegal | illegal | illegal |
| U, `mcounteren`=5, `scounteren`=7 | read | illegal | illegal |

5 is CY (bit 0) and IR (bit 2). In the last row `scounteren`=7 also has TM (bit 1) set, yet `time` is illegal. U-mode may read a counter only when the bits in **both** registers are set (spec 4.1.5 note), and `mcounteren.TM` was 0. As a circuit: `ctr_ok = (priv==M) | (mcounteren[n] & (priv==S | scounteren[n]))`.

For another extra condition, set `mstatus.TVM` (written in M-mode, then drop to S-mode). Reads and writes of `satp` from S-mode became illegal instructions while `sstatus` was still readable in the same mode. Clearing TVM made `satp` readable again (privileged spec 3.1.6). This is where "extra per-CSR conditions" enter the check.

### Checking it: the `rs1` field versus the value

We touched the read-only `mhartid` (0xF14) from M-mode in several forms.

| Instruction | Result |
|---|---|
| `csrr rd, mhartid` (`csrrs`, `rs1=x0`) | reads (0) |
| `csrrs rd, mhartid, t0` (t0=0) | illegal |
| `csrrc rd, mhartid, t0` (t0=0) | illegal |
| `csrrs rd, mhartid, t0` (t0=1) | illegal |
| `csrrsi`, `csrrci rd, mhartid, 0` | reads (0) |
| `csrrsi rd, mhartid, 1` | illegal |
| `csrw`, `csrrw`, `csrrwi` (value 0) | illegal |

Even with t0=0 it was a write attempt because `rs1` was not `x0`. `csrrw` always writes, even a zero, so on a read-only CSR it is illegal. The model and QEMU agreed on every row.

## 3. Views and shadows: several addresses, one state

![Four ways to see the same storage. sstatus is the S fields of mstatus, sie is mie filtered by mideleg, cycle is a read-only shadow of mcycle, fflags is the low 5 bits of fcsr|full](/img/01M75HAZ9MMPAAWKECZKWMQGZ0/ce34f4a9f27e.webp)

A "view" is an address that sees one storage through a mask. The implementation is two masks. A read is `storage & read_mask`. A write is `storage = (storage & ~write_mask) | (wdata & write_mask)`. There are no separate flip-flops for the view's bits. Storing them separately opens a path to bugs where two addresses disagree.

### `sstatus` is part of `mstatus`

We wrote −1 and 0 alternately to `mstatus` and `sstatus` and read both (RV64).

```text
csrw mstatus,0    mstatus 0x0000000a00000000
csrw sstatus,-1   mstatus 0x8000000a000c6122
                  sstatus 0x80000002000c6122
csrw mstatus,-1   mstatus 0x8000000a007e79aa
                  sstatus 0x80000002000c6122
csrw sstatus,0    mstatus 0x0000000a00721888
                  sstatus 0x0000000200000000
```

Writing −1 to `sstatus` set only `SIE`, `SPIE`, `SPP`, `FS`, `SUM` and `MXR`; `MIE`, `MPIE` and `MPP` stayed 0. Writing −1 to `mstatus` also turns on `MIE`, `MPIE`, `MPP` (=3), `MPRV`, `TVM`, `TW` and `TSR`, yet `sstatus` is the same `0x000c6122`. Finally, writing 0 to `sstatus` clears only the S fields, leaving `0x00721888` (the M fields) in `mstatus`. The top bit `SD` (bit 63) is a read-only summary bit. In an implementation it is simplest not to store it but to build it on the read path from `FS==3` (or `VS`/`XS` dirty). Clearing `FS` through `sstatus` cleared `SD` too, which fits that design. `UXL` (bits 33:32) is visible in `sstatus` but read-only 2, and `SXL` is not visible there.

`csrrs` and `csrrc` also compute on the view. With `mstatus` at 0, `csrrs sstatus, (SPP|SUM)` returned the old `sstatus` value `0x200000000` (only `UXL`) and left `mstatus` at `0xa00040100`. Then `csrrc sstatus, SPP` left `0xa00040000`.

### `sie` and `sip` are `mie` and `mip` filtered by `mideleg`

The spec says: if an interrupt is delegated to S-mode through `mideleg`, it becomes visible in `sip` and maskable in `sie`; otherwise those bits of `sip` and `sie` are read-only zero (privileged spec 3.1.9, last paragraph). The body of section 4.1.3 in 20240411 says only that `sip` and `sie` are subsets of `mip` and `mie`, so the delegation condition has to be read in 3.1.9. In hardware it is one AND gate: the read and write mask of `sie` is `mideleg & S-interrupt bits`.

Values measured on QEMU (accessed from M-mode):

| Step | Value read |
|---|---|
| write `0xffff` to `mie` | `mie` = `0xaaa` |
| with `mideleg`=0 | `sie` = 0, and `mie` unchanged after writing `sie` |
| write `0x222` to `mideleg` | `mideleg` = `0x222`, `sie` = `0x222` |
| write 0 to `sie` | `mie` = `0x888` |
| write `0xffff` to `sie` | `mie` = `0xaaa` |

Writing `0xffff` to `mie` left only six bits (1, 3, 5, 7, 9, 11: SSI, MSI, STI, MTI, SEI, MEI). That is the rule that the `mie` bit of an unimplemented interrupt is read-only zero. Writing 0 to `sie` cleared only the delegated S bits (`0x222`), leaving `0x888`. `mip` is similar. Writing `0xaaa` from M-mode read back `0x2a2`: the writable bits SSIP, STIP and SEIP (1, 5, 9), plus bit 7 (MTIP), which we never wrote but read as 1. MTIP is an input driven by a device, independent of writes (we did not check why it was raised). `sip` showed `0x222`, and writing 0 to `sip` left `mip` at `0x2a0`. S-mode can write only SSIP, so STIP and SEIP stay. Writing `0xffff` to `sip` set only SSIP again, back to `0x2a2`.

`mip` also has a bit whose read path and write path differ. The spec says reading SEIP returns "the software-writable bit OR the external interrupt signal", but only the software bit takes part in the read-modify-write of `csrrs` and `csrrc` (3.1.9). The input of the read mux and the input of the write computation differ. We could not test this because we did not drive an external signal on QEMU `virt`.

Writing `mideleg` changes whether `sie` is writable. The spec says that when an alias field goes from read-only to writable its value must immediately reflect the alias's current value, not be UNSPECIFIED (2.4). A view design satisfies this automatically. If you store the bits separately you need a circuit that copies them when `mideleg` is written.

### `fflags`, `frm`, `fcsr`: aliases, and a write that changes another CSR

`fcsr` (0x003) is 8 bits, with `fflags` (0x001) at `[4:0]` and `frm` (0x002) at `[7:5]`: two aliases, one storage. More interesting is the link to `mstatus.FS`. On QEMU:

| Situation | Result |
|---|---|
| read `fcsr`, `fflags` with `FS`=Off | illegal instruction (cause 2) |
| read `fflags` with `FS`=Initial (1) | 0, `mstatus` unchanged |
| write `0x1f` to `fflags` | `fcsr`=`0x1f`, `mstatus.FS`=3 (Dirty), `SD`=1 |
| read `frm` with `FS`=Clean (2) | 0, `FS` stays 2 |
| write `0xff` to `frm` | `frm`=7 (only 3 bits kept), `fcsr`=`0xff`, `FS`=3 |

A write that changes FP state sets `mstatus.FS` to Dirty. A read does not touch FS. So the `mstatus` storage has two write sources: `csrw mstatus` and FP writes (`fcsr`, FP register updates). Two instructions cannot collide in one instruction, so merging (OR-like) instead of priority may be enough, but it becomes a problem in a pipeline where different instructions write from different stages (section 6).

### `cycle`, `instret`, `time` are read-only shadows

`cycle` and `instret` are read-only shadows of `mcycle` and `minstret`, and `time` is a shadow of the memory-mapped `mtime` (spec 3.1.11). That is a view whose read mask is everything and write mask is 0, plus the enable-bit check from section 2. `time` may be implemented by turning the read into a read of the `mtime` register (spec note), and this model does not deal with its value.

## 4. WARL cleanup circuits

Part 23 said "how to handle `mtvec.MODE` is chosen in part 24". There are three options.

| Option | Circuit | Writing 3 gives |
|---|---|---|
| (a) ignore the whole write | write enable = `we & ~wdata[1]` | old value kept |
| (b) force MODE[1] to 0 | store `wdata[1]` as 0 | MODE=1 (vectored) is stored |
| (c) fix MODE | field is constant 0 (direct) | MODE=0 |

The criterion is how software gets fooled. With (b), writing MODE=3 leaves 1, so "writing a reserved value turns on vectored mode", which is odd. (c) cannot use vectored mode. This post picks (a). Code that writes BASE and MODE together does not get a half-changed BASE when MODE is illegal, and the difference between written and read values reveals the failure. QEMU is (a) too. Changing `mtvec` in M-mode during the test would break the trap handler, so we checked `stvec`, which has the same rule.

| Value written | Value read |
|---|---|
| `0x80001000` (MODE 0) | `0x80001000` |
| `0x80002001` (MODE 1) | `0x80002001` |
| `0x80003003` (MODE 3) | `0x80002001` (old value) |
| `0x80004002` (MODE 2) | `0x80002001` (old value) |

We checked other fields cleaned the same way on QEMU.

| Field | Written | Read |
|---|---|---|
| `mstatus.MPP` (M+S+U) | 1, 2, 3, 2 in turn | 1, 1, 3, 3 |
| `satp.MODE` (RV64) | 0, 1, 5, 8, 9, 10, 11, 15 | only 0, 8, 9, 10 accepted |
| `mcounteren`, `scounteren` | −1 | `0x7ffff` (bits 0 to 18) |
| `mcountinhibit` | −1 | `0x7fffd` (no TM bit 1) |

`MPP` kept the old value per field. Writing the reserved 2 leaves the 1 or 3 just written. As a circuit: `mpp_d = (wdata[12:11]==2'b10) ? mpp_q : wdata[12:11]`. For `satp`, the whole write is ignored when MODE is not legal (spec 4.1.11), so `we_satp = we & mode_legal`. On RV32 MODE is the single bit 31 and both 0 and 1 were accepted. The counter-enable registers have bits only for counters that exist (CY, TM, IR, HPM 3 to 18). `mcountinhibit` has no TM bit; the spec's reason is that `time` may be shared through `mtime` and cannot be inhibited (3.1.12 note).

Every legalization choice so far is a model parameter (`mpp`, `tvec`, `satp_modes`). Putting the choices the spec left to the implementation in one place in the code means verification does not have to ask which way you went.

## 5. Counters and `mcountinhibit`

Counters are 64 bits wide. `mcycle` advances every clock and `minstret` advances when an instruction completes. On RV32 the low half `mcycle` (0xB00) and the high half `mcycleh` (0xB80) are read separately (2.7). When CY (bit 0) or IR (bit 2) of `mcountinhibit` is set, that counter stops; the register does not affect accessibility (3.1.12). In a circuit, if the write enable **replaces** the increment, you match the spec.

```systemverilog
// Illustrative, not simulated. _q is the register value at the start of the cycle
mcycle_d   = wr_mcycle   ? wdata
           : inhibit_q[0] ? mcycle_q : mcycle_q + 1;
minstret_d = wr_minstret ? wdata
           : inhibit_q[2] ? minstret_q : minstret_q + retire;
```

This gives two rules for free. An instruction that writes a counter loses its own increment (`wdata` replaces it). An instruction that writes `mcountinhibit` sees `inhibit_q` (the old value), so its own increment follows the old setting. The spec basis is the Zicsr sentence (a CSR write replaces the increment; in particular a value written to `instret` is the value the next instruction reads) and 3.1.10 ("a CSR write takes effect after the writing instruction has otherwise completed"). The reading for `mcycle` and `mcountinhibit` is derived from these two sentences.

![Reads after writing 1000 to mcycle are 1000, 1001, 1003, 1004 in spec order and 1001, 1002, 1004, 1005 on QEMU. After mcountinhibit is cleared, QEMU is again higher by 1|full](/img/01M75HAZ9MMPAAWKECZKWMQGZ0/7461d59d5253.webp)

### Checking it: a write and the increment in the same cycle

By default QEMU counts `mcycle` with the host clock. Running the window "write 1000 to `mcycle`, then read" four times, the first read was 3000 or 4000 and the step between reads was not 1 (one run: 4000, 7000, 9000, 12000, 14000). It is not a cycle-accurate simulator. With `-icount shift=0` one instruction is one tick and the values are deterministic. Everything below uses that setting. So QEMU's "cycle" is the number of executed instructions, and CPI is 1.

The test is, in M-mode, `csrw mcycle, s1` (s1=1000) followed by read instructions with a `nop` in between. Values are stored after the window ends, so no measurement code intrudes.

| Values read (in order) | `mcycle` window | `minstret` window |
|---|---|---|
| spec-order model | 1000, 1001, 1003, 1004 | 1000, 1001, 1003, 1004 |
| QEMU | 1001, 1002, 1004, 1005 | 1001, 1002, 1004, 1005 |

After writing 1000 to `minstret`, the value read by the very next instruction was 1000 by the spec and 1001 on QEMU. The writing instruction also retires, but the spec says the written value replaces that increment, so by the spec `csrw minstret` is never counted in `minstret`. QEMU seems to apply the write first and then count the instruction too (presumably it uses the instruction count at the time of the write as its baseline). `mcountinhibit` follows the same order. Right after writing 5 (CY and IR), `mcycle` stopped at 1002 and `minstret` at 1001 (same after a `nop`), and both orders agreed. But after setting it back to 0, the spec order gave 1002, 1002, 1004 and QEMU gave 1003, 1003, 1005. In spec order the instruction that clears it follows the old setting (inhibited) and is not counted; QEMU applies the new setting (allowed) first and counts it.

With only CY inhibited (QEMU values), `mcycle` stayed at 1002 while `minstret` kept going from 1003 to 1005. The two inhibit bits are independent. Also note what happens when a `csrrs` modifies a running counter. After writing 1000 to `mcycle` we did `csrrs rd, mcycle, (1<<20)` and read twice. Spec order read the old value 1000, then `0x1003e8` and `0x1003e9` (continuing from 1000 OR 2^20). QEMU read 1001, `0x1003ea`, `0x1003eb`. A read-modify-write instruction on a running counter makes that cycle disappear instead of being counted. Software that tries to correct the counter live needs to know this.

### Hardware performance counters

`mhpmcounter3` to 18 were implemented (16 of them) and `hpmcounter31` was illegal. Writing 1000 to `mhpmcounter3` read back 1000, and it was still 1000 several instructions later. Writing 7 to `mhpmevent3` read back 7. That `mhpmcounter3` stays at 1000 suggests this QEMU does not count event 7 (our guess). The spec recommends implementing all counters, but making both the counter and the event selector read-only 0 is legal too (3.1.10). For a small core that is the cheapest choice; then the matching bits of `mcounteren` and `mcountinhibit` are read-only 0 too.

### RV32: reading in two halves

On RV32, reading a 64-bit counter in two steps lets a carry slip in between. This is why chapter 8 of the unprivileged spec gives code that reads `rdcycleh`, `rdcycle`, `rdcycleh` and repeats until the upper halves match. In the spec-setting model (carry on, write replaces the increment), we wrote `mcycleh`=0 and `mcycle`=`0xfffffffd` and read (low, high, low, high). The output was `fffffffd, 0` and then `ffffffff, 1`. The second pair is a torn value `0x1ffffffff`: the low half read before the carry and the high half after. The real value is near `0x100000000`. This is a model prediction.

QEMU did not reproduce it. After writing `0xfffffffe` to `mcycle`, the low half read `ffffffff`, then `1`, then `3`, while `mcycleh` and `cycleh` stayed 0. For a 64-bit counter `mcycleh` should have become 1. QEMU RV32 (11.1.3, `-icount shift=0`) appears not to pass the carry from the low 32 bits to the high half (our guess). The model's `carry='none'` parameter is this behavior. The QEMU behaviors that differ from the spec were this and the write order above, both limited to counters.

## 6. CSRs inside the pipeline

When attaching the CSR file to the 5-stage pipeline of [part 17](post:01M6K8E2MPEJ5G51N0JWQC7XQ0), the decision is "in which stage do we read and write". The spec says CSR accesses happen in program order, the effect of a CSR write must be observable by the very next instruction, and earlier instructions must not see it first (Zicsr, "CSR Access Ordering"). To keep that, a CSR write has to happen after all earlier instructions have finished without exceptions. If an earlier `lw` can take a page fault, `mstatus` must not already have changed. Common choices:

- **Read and write in the MEM or WB stage.** Finish the illegal-instruction check in ID (the equations of section 2) and do the write in a later stage. If the read is in MEM, there is a load-like hazard. If the next instruction uses the value, it is not in EX/MEM yet, so stall one cycle or forward from MEM/WB. Part 17's condition `RegWrite ∧ rd≠0 ∧ rd=rs` applies unchanged.
- **CSR instructions serialize the pipeline.** Some CSRs change how earlier stages behave. `satp` changes address translation for instruction fetch and loads and stores, `MPRV`, `SUM` and `MXR` in `mstatus` change load and store translation and permission, and `mstatus.FS` changes whether FP instructions are legal. After writing such a CSR you discard the already fetched and decoded instructions behind it and refetch from the next pc. A CSR like `mscratch` does not need this. Whether to apply it to every CSR write and accept the slowdown, or to make a list and trim it, is the design decision. Parts 30 and 31 cover `satp` and `SFENCE.VMA`, and part 27 covers traps and pipeline flush.
- **Count `minstret` in WB.** Add the number of retiring instructions; with single issue that is at most 1. If `csrw minstret` is in WB in the same cycle, the write wins over the increment as in the section 5 equation. `mcycle` advances every clock, and what happens in situations like a stopped clock during `wfi` is left to the execution environment, as chapter 8 of the unprivileged spec says.

We did not check how this model's "one cycle per instruction" assumption changes in this structure. In a 5-stage pipeline `mcycle` counts cycles, so it drifts from `minstret` by the CPI. Making the two counters advance 1:1 as in this model is an assumption matched to QEMU's icount.

## 7. Experiment: model versus QEMU

A scenario is a list of segments (mode, instruction list). From the same list we build two things. One is QEMU assembly: for each instruction it loads the value and a sentinel with `li`, runs the CSR instruction, and records `a0` (the read value) and the last trap's `mcause`. If a trap occurs, `a0` stays the sentinel, so it is distinguishable in the record. S and U mode are entered and left with part 23's `enter_mode`. In the counter tests any extra instruction between would change the values, so registers for the values are assigned beforehand and stored only after the window ends. The other is the feed to the model. The core of one model cycle:

```python
def step(self, mode, ins=None, retire=True):
    ...
    try:
        self._check(mode, a, writes)              # exists, mode, [11:10], extra
        old = self._read(a)                       # read value through views/shadows
        if writes:
            new = src if rw else old | src if rs else old & ~src
            self._write(a, new)                   # view mask, WARL cleanup
    except Illegal:
        cause = 2; retire = False
    inh = st['mcountinhibit'] if self.order == 'before' else inh_old
    bump(0, 0, True); bump(2, 2, retire)          # counter update
```

`order='after'` equals the register equations of section 5 (the write replaces the increment, and inhibit changes apply from the next instruction). `order='before'` applies the write first and then counts that instruction. The model was built from the spec first, not tuned to QEMU. Only after seeing the mismatched cells did we add `before` and `carry='none'`.

| Setup | Cells compared | Spec-order model | QEMU-order model |
|---|---|---|---|
| RV64 | 349 | 330 match, 19 differ | all 349 match |
| RV32 | 351 | 320 match, 31 differ | all 351 match |

Of the 349 RV64 cells, 187 compared only the cause (whether a trap occurred) and 162 compared the value too (reads of `cycle`, `time` and `mcycle` whose value differs between runs compare only the cause). RV32 has 187 and 164. Every mismatch is in a counter window. The 19 RV64 cells are 5 in the `mcycle` window, 4 in the `minstret` window, 3 + 4 after clearing the inhibit (`CY|IR`, and CY only), and 3 in the `csrrs`-modified counter. The extra 12 on RV32 are cells in the half-reading test where QEMU (no carry) and the spec model (carry) diverge. We also ran the same program twice to see whether the compared values are identical (deterministic); only the `cycle`, `time` and `mcycle` reads right after startup differed.

To reproduce: `uv run python check.py` (writes `results.json`).

## 8. Design checklist: the CSR file

| Item | Rule |
|---|---|
| Write attempt | from the instruction fields. `rw` always; `rs`, `rc` when `rs1`≠`x0` (`uimm`≠0). Even a value of 0 is a write if `rs1`≠`x0` |
| Read attempt | `csrrw`, `csrrwi` do not read if `rd`=`x0`. Do not attach read side effects |
| Permission check | exists; mode ≥ `csr[9:8]`; write attempt and `csr[11:10]`=11 is illegal; per-CSR extra conditions |
| When illegal | do not change the CSR. Apply the write enable after the check |
| Extra conditions | counters pass both `mcounteren` (S, U) and `scounteren` (U); S-mode `satp` needs `TVM`=0; `fcsr` family needs `FS`≠Off |
| Views | one storage with a read and write mask. Compute on the visible value when writing. Do not store separately |
| Derived bits | build `SD` on read from `FS`, `VS`, `XS` |
| Read path ≠ write path | `mip.SEIP` ORs in the external signal on read, but `csrrs`/`csrrc` writes use only the software bit |
| Write that changes another CSR | writes to `fflags`, `frm`, `fcsr` set `mstatus.FS` to Dirty. Merge the write ports |
| WARL cleanup | make it legal or ignore the write. Put the choices (MODE, `MPP`, `satp.MODE`) in one place in the code |
| Not implemented | read-only 0 (missing bits of `mie`, `mcounteren`, `mcountinhibit`, missing `hpm`) or illegal |
| Counters | the write replaces the increment. Inhibit changes apply from the next instruction. `minstret` does not count the instruction that writes it |
| `mcountinhibit` | CY, IR, HPM only. `time` cannot be inhibited |
| RV32 | reading halves tears at the carry. A 64-bit counter carries into the high half |
| Pipeline | write after earlier instructions finish. Flush behind CSR writes like `satp` or `MPRV` |

## Summary

- A CSR file is decode, check, read mux, write-value computation, WARL cleanup and storage. The write attempt comes from the `rs1` field (if `rs1`≠`x0`, a read-only CSR is illegal even when the value is 0), and a view points at one storage with two masks.
- In the model-versus-QEMU comparison the permission matrix, the `sstatus`, `sie` and `sip` views, the cleanup of `stvec`, `MPP` and `satp`, and the `FS` update by `fflags` all agreed. Of 349 RV64 cells and 351 RV32 cells, only counter windows differed.
- For counters the spec says "the write replaces the increment, and inhibit changes apply from the next instruction". QEMU applied the write first and read 1 higher, and RV32 `mcycleh` did not take the carry. QEMU is not cycle-accurate, so comparing counter values needs `-icount`.
- When putting CSRs in a pipeline, write in a late stage and flush after CSRs that affect earlier-stage behavior. How an illegal instruction actually enters a trap is the next part.

**Next:** Part 25 is traps. We look at exception and interrupt kinds, the direct and vectored modes of `mtvec`, how the hardware changes `mepc`, `mcause`, `mtval` and `mstatus` when it takes a trap, and what `mret` restores. How this post's illegal instruction ends up recorded as `mcause`=2 is completed there.

## References

- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. 2.1 CSR address rules, 2.3 field rules (WPRI, WLRL, WARL), 2.4 field modulation (aliases), 2.6 to 2.7 width modulation and CSRs wider than XLEN, 3.1.6 `mstatus` (TVM, FS and SD), 3.1.7 `mtvec`, 3.1.8 `medeleg` and `mideleg`, 3.1.9 `mip` and `mie`, 3.1.10 hardware performance monitor (`mcycle`, `minstret`, `mhpmcounter`), 3.1.11 `mcounteren`, 3.1.12 `mcountinhibit`, 4.1.1 `sstatus`, 4.1.3 `sip` and `sie`, 4.1.5 `scounteren`. https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 7 Zicsr (read and write conditions table, counter write rule, CSR access ordering), chapter 8 Zicntr and Zihpm (counters, 64-bit read code).
- D. Harris et al., *RISC-V System-on-Chip Design*, 2023. Privileged unit and CSR implementation (CORE-V Wally)
- QEMU documentation: RISC-V `virt` board, `-icount`. https://www.qemu.org/docs/master/
