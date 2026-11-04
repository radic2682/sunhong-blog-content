---
id: "01M6K7F9D0SH3ANM7K1VD4QJ0S"
title_en: "V Extension 2: Masks, Load/Store Types, Reductions, Permutes, Tail/Mask Agnostic"
summary_en: "Part 14 of the RISC-V series. I ran v0 masks, the four kinds of loads and stores, a first-fault strlen, reductions and permutations, and ta/ma in QEMU. Four kernels matched Python on all 78 items, and at VLEN 1024 they executed 4–7% as many instructions as scalar code."
---
:::note
**Key points in three lines**
- In vector code, an `if` is not a branch but a mask. A compare instruction writes 1 bit per element into `v0`, and the following instruction uses `v0.t` to update only the elements whose bit is 1. Whether to preserve the masked-off elements (inactive) and the elements beyond vl (tail) is decided by the ma and ta bits of `vtype`.
- Memory accesses come in four shapes: contiguous (unit-stride), fixed spacing (strided), a list of addresses (indexed), and struct fields (segment). A first-fault load shrinks vl instead of faulting unless the fault is on element 0. On a string right before the end of RAM, a normal load raised an access fault (vstart=40), while a first-fault load shrank vl to 40 and moved on.
- I ran four kernels (masked add, gather, sum, and a `vcompress` filter) on 1,000 elements in QEMU at VLEN 128–1024, and all 78 items matched the Python reference. The executed instruction count was 31–55% of scalar at VLEN 128 and 4–7% at 1024.
:::

In Part 13 we covered vector registers, `vtype` and `vl`, and wrote vector-length-agnostic loops with `vsetvli`. There, every element got the same treatment and memory was read only contiguously. Real loops have conditionals, look up values in tables, reduce an array to a single value, and reorder elements. Part 14 is the other half of the V extension that handles these. We go through masks, the four kinds of loads and stores plus first-fault loads, reductions, permutations, and tail/mask agnostic in turn, and finish by collecting the rules the decoder and register file need to handle. The source is the same as Part 13: Chapter 31 (V 1.0) of the unprivileged specification, version 20240411.

## 1. Masks: `v0` and Conditional Execution

Scalar code turns `if (a[i] > 0)` into a branch. A single vector instruction processes many elements together, so each element cannot take a different path. The V extension therefore turns conditions into **masks**. The comparison result is stored as 1 bit per element, the operation runs on every element, and only the results for elements whose bit is 1 are written.

The mask layout is simple. Bit i of the mask register is element i, always, regardless of SEW or LMUL. Even in the largest VLMAX case (e8, m8: 128 elements at VLEN 128), it fits in the 128 bits of a single register. So a mask is always a single register.

Whether an instruction uses a mask is decided by the **vm bit** at inst[25]. vm=0 means `v0.t` (v0 as the mask); vm=1 means unmasked, all elements. `v0` is the only register that can be used as a mask. Instead of a field for a mask register number, it was reduced to 1 bit. The tradeoff is that code using several masks needs extra instructions to move them into `v0`.

Let's trace one by hand. At VLEN 128, e32, vl=4, compute `c[i] = a[i] > 0 ? a[i] + b[i] : a[i]`.

```text
v1 = a = { 5, -3,  0,  8}     (elements 0, 1, 2, 3)
v2 = b = { 1,  1,  1,  1}
vmsgt.vi v0, v1, 0            → v0 = 0b1001
vadd.vv  v1, v1, v2, v0.t     → v1 = { 6, -3,  0,  9}
```

`vmsgt.vi` compares `a > 0` per element and writes 1 for elements 0 and 3. In binary, bits 3 and 0 are 1, giving `0b1001`. `vadd` adds only those two elements, and elements 1 and 2 keep v1's original values (a). This "keeping" is the mask undisturbed policy of Section 2. GCC 16.2 compiled the intrinsic `__riscv_vadd_vv_i32m1_mu` into exactly this shape, adding `mu` to the `vsetvli`.

The instructions that create and manipulate masks are as follows.

| Kind | Instructions | What they do |
|---|---|---|
| Compare | `vmseq` `vmsne` `vmslt` `vmsle` `vmsgt` … | Per-element compare, 1-bit result |
| Mask logic | `vmand.mm` `vmor.mm` `vmxor.mm` `vmandn.mm` … | Bitwise operations between masks |
| Count/find | `vcpop.m`, `vfirst.m` | Number of 1s, position of the first 1 (−1 if none) |
| Prefix | `vmsbf` `vmsif` `vmsof` | 1 before / up to / only at the first 1 |
| Index | `viota.m`, `vid.v` | Count of preceding 1s, element index |

Compare instructions that produce a mask write one mask into one register (even if LMUL is 8). There is one rule between masked operations and the destination register. The destination register group of a masked (vm=0) instruction must not overlap `v0`. The only exceptions are when the result is a mask (compares) or the scalar result of a reduction. The rule prevents an instruction from overwriting its own mask while writing elements.

## 2. Body and Tail: Tail/Mask Agnostic

The elements an instruction sees fall into three parts.

- **prestart**: elements with index less than `vstart`. These are the parts already finished when restarting after an exception (Section 5). They are not touched.
- **body**: `vstart` to vl−1. Active if the mask bit is 1, inactive if 0.
- **tail**: vl to VLMAX−1. Elements outside the current vl.

Active elements get the result. The question is the inactive and tail elements. The vma and vta bits of `vtype` set the policy for these two.

- **undisturbed (mu/tu)**: the previous value must be preserved.
- **agnostic (ma/ta)**: the previous value may be preserved, or every bit may be overwritten with 1.

Why is agnostic needed? Consider an out-of-order core with register renaming. The result of `vadd.vv v8, …` is written to a new physical register. If the tail had to be preserved, the old v8 value would have to be read and copied into the new register. That needs one more read port, and the instruction has to wait for the instruction that produced the old v8 (a data dependency appears). With agnostic, that copy can be skipped. Conversely, a simple in-order core can just skip writing, so undisturbed is actually easier there. The specification explains that the overwrite value was chosen as all 1s rather than 0 so that software does not write code that relies on that value.

I checked this in QEMU. I filled v8 with 7, executed `vadd.vv v8, v1, v2, v0.t` with vl=3 and mask `0b0101` (elements 0 and 2 active), then read all four elements with `vs1r.v` (stores the whole register regardless of vl).

![With vl=3 and mask 0b0101, vadd under tu/mu preserves the 7 in the inactive and tail elements; under ta/ma, QEMU preserves 7 by default and fills with 0xffffffff when rvv_ta_all_1s and rvv_ma_all_1s are enabled|full](/img/01M6K7F9D0SH3ANM7K1VD4QJ0S/03d8719d0978.webp)

By default, QEMU preserved the previous values even under ta/ma, one of the two behaviors the specification allows. With the CPU properties `rvv_ta_all_1s=true,rvv_ma_all_1s=true`, the same machine code turned elements 1 and 3 into `0xffffffff`. In software verification, running with this setting catches bugs of the "agnostic but relies on the old value" kind. One more thing: the tail of an instruction that writes a mask result (compares and so on) is always agnostic, regardless of vta.

## 3. Four Kinds of Loads and Stores

The `vle32.v` from Part 13 was a unit-stride load that reads contiguous elements. There are three more ways to form addresses.

![Unit-stride reads 4 contiguous elements, strided reads at a fixed spacing, indexed reads the slots pointed to by an offset vector, and segment splits the fields of an array of structs into separate registers|full](/img/01M6K7F9D0SH3ANM7K1VD4QJ0S/62d151b08fed.webp)

| Kind | Load / Store | Address |
|---|---|---|
| Unit-stride | `vle32.v` / `vse32.v` | base + i × 4 |
| Strided | `vlse32.v` / `vsse32.v` | base + i × rs2 (bytes) |
| Indexed | `vluxei32.v` / `vsuxei32.v` | base + vs2[i] (bytes) |
| Segment | `vlseg3e8.v` / `vsseg3e8.v` | 3 fields into v, v+1, v+2 |

A **strided** load gathers elements at a fixed spacing, like a column of a matrix. The stride is given in bytes in an integer register, and can be 0 or negative. In the experiment, reading column 3 of a 16×16 `int32` matrix with a 64-byte stride gave all 16 values correctly (3, 103, 203, …, 1503).

An **indexed** load is a gather with a different address per element; the store is a scatter. The offset vector vs2 holds **byte** offsets, so for subscripts into an `int32` array they must be multiplied by 4 (the `vsll` in the experiment code). The offset width is the EEW in the instruction name (32 bits for `ei32`), and the data element width follows SEW — the opposite of a normal load. The `u` in the name means unordered, and `o` means ordered. Ordering matters where access order is meaningful, such as device memory. In a scatter where several elements write the same address, only ordered guarantees that the last element's value remains.

A **segment** load splits an array of structs with interleaved fields, such as RGB pixels, into one register per field. `vlseg3e8.v v8, (a0)` puts R in v8, G in v9, and B in v10. In the experiment I split 64 pixels; vl was 16 at VLEN 128, so it took 4 iterations, 2 at 256, and 1 at 512 and above. The number of fields (NFIELDS) is 2–8, and NFIELDS × EMUL ≤ 8 must hold. A register number going past v31 is also reserved.

In terms of encoding, vector loads and stores do not use new opcodes. They share the opcodes of FP loads (LOAD-FP, `0000111`) and FP stores (STORE-FP, `0100111`) and are distinguished by the width field (funct3).

```text
LOAD-FP width: 000 e8, 101 e16, 110 e32, 111 e64 → vector
               001 flh, 010 flw, 011 fld, 100 flq → FP
[31:29] nf    number of segment fields − 1
[28]    mew   0 (1 is reserved for elements of 128 bits or more)
[27:26] mop   00 unit, 01 indexed (u), 10 strided, 11 indexed (o)
[25]    vm    mask
[24:20] for unit-stride, lumop: 00000 normal,
        01000 whole register, 01011 mask, 10000 ff
```

Cores without V also decode LOAD-FP, so a core with only F must send width 000, 101, 110, or 111 to illegal instruction.

## 4. First-Fault Loads and `strlen`

Measuring the length of a C string with vectors has a problem. You do not know where the string ends, so reading 16 bytes at a time ends up reading memory past the end (NUL). If that memory is an unmapped page, you get a fault that the original program never had.

**First-fault loads** (`vle8ff.v` and others) solve this. There are two rules.

- If an exception occurs on element 0, it traps like a normal load. vl is unchanged.
- If an exception occurs on element 1 or later, it does not trap and instead reduces vl to that element's index, as if it had read only up to there.

If element 0 can be read, at least one element makes progress, so the loop never stalls, and a genuinely bad address becomes element 0 on the next iteration and traps. The specification allows an implementation to reduce vl even without an exception (if vstart=0 and vl>0, it must process at least one element). So the code has to reread the reduced vl every time. GCC inserted `csrr a5, vl` right after `vle8ff.v`.

QEMU `virt`'s RAM is 128MiB starting at `0x80000000`, so it ends at `0x88000000`, with no device beyond it. I placed a 30-character string (NUL at index 30, `z` at 31–39) 40 bytes before the end of RAM and read it in chunks of the e8/m1 VLMAX.

| VLEN | VLMAX | `vle8ff` vl sequence | Same task with `vle8.v` |
|---|---|---|---|
| 128 | 16 | 16, 16 | No problem |
| 256 | 32 | 32 | No problem |
| 512 | 64 | 40 | Access fault, vstart 40 |
| 1024 | 128 | 40 | Access fault, vstart 40 |

From VLEN 512, trying to read 64 bytes at once gets blocked at the 41st byte (`0x88000000`). The first-fault load reduced vl to 40, found the NUL at index 30 within that, and returned 30. The normal load trapped with mcause 5 (load access fault), mtval `0x88000000`, and **vstart 40**. Per the precise exception rule in the next section, vstart means "restart from element 40." The same machine code is fine on a narrow machine and faults only on a wide one. That is why vector-length-agnostic code needs first-fault loads.

## 5. Reductions and Permutations

A **reduction** reduces a vector to a single scalar. `vredsum.vs vd, vs2, vs1` adds element 0 of vs1 and all active elements of vs2, and writes the result to element 0 of vd. The remaining elements of vd are treated as tail. Because it takes element 0 of vs1 as the initial value, a strip-mining loop can carry the running total in element 0.

```text
vsetivli zero, 1, e32, m1, ta, ma
vmv.s.x  v1, zero          # accumulator = 0
loop:
  vsetvli    a5, a1, e32, m1, ta, ma
  vle32.v    v2, (a0)
  vredsum.vs v1, v2, v1    # v1[0] += sum(v2[0..vl-1])
  ...
vmv.x.s  a0, v1            # to an integer register
```

Integer reductions such as `vredmax` and `vredand` give the same answer in any order. FP sums round differently depending on order, so they come in two forms. `vfredosum` adds in element order, and `vfredusum` leaves the order to the implementation. On the unordered side, hardware is free to build any adder tree. In the experiment, the sum of 1,000 random integers in −1000 to 1000 was −22,951 at all four VLENs, matching Python.

A **permutation** moves elements to different positions.

- `vslideup.vx vd, vs2, k`: vd[i+k] = vs2[i]. Elements 0 to k−1 of vd are left as is.
- `vslidedown.vx vd, vs2, k`: vd[i] = vs2[i+k]. 0 if i+k is at least **VLMAX**.
- `vrgather.vv vd, vs2, vs1`: vd[i] = vs2[vs1[i]]. 0 if the index is at least VLMAX.
- `vcompress.vm vd, vs2, vs1`: packs only the elements whose mask bit in vs1 is 1 to the front.

Trying these on 8 elements {10, …, 17}: slideup by 3 gave {−1, −1, −1, 10, 11, 12, 13, 14} (the first 3 keep the pre-filled −1), and `vrgather` with {7, 6, …, 0} made by `vid` reversed the order. The interesting one is slidedown. With vl=8, e32/m2, VLMAX is 8 at VLEN 128, so the last 3 were 0. But at VLEN 256 and above, VLMAX is 16 or more, so elements 8–10 of vs2 beyond vl (values left over from a previous operation) came in. This is because the rule is based on VLMAX, not vl. In vector-length-agnostic code, never trust the values of elements beyond vl.

`vcompress` is used for filters. A loop that keeps only positive values looked like this.

```text
vle32.v      v1, (a1)
vmsgt.vi     v0, v1, 0     # positive mask
vcpop.m      a3, v0        # count
vcompress.vm v2, v1, v0    # positives to the front
vsetvli      zero, a3, e32, m1, ta, ma
vse32.v      v2, (a4)      # store only that many
```

A loop that would have had a branch per element in scalar code has only one branch, at the end of the loop. Of 1,000 elements, 478 remained, and even the order matched Python.

## 6. Experiment: Four Kernels, Four VLENs

Same method as Part 13. I built `vops.c` once with `-march=rv64gcv -O1` (first 16 characters of the `.text` sha256: `cd6ae9d556f31d59`) and ran it on QEMU 11.1.3 at VLEN 128, 256, 512, and 1024. As in Part 13, the startup code enables `mstatus.VS`, and with `-icount shift=0`, `minstret` served as the executed instruction count. Inputs come from a fixed-seed 64-bit LCG so that C and Python produce the same values. The machine is a Mac M1 Max with `riscv64-elf-gcc` 16.2.

```c
size_t vfilter(int32_t *c, const int32_t *a, size_t n) {
  size_t k = 0;
  for (size_t vl; n; n -= vl, a += vl) {
    vl = __riscv_vsetvl_e32m1(n);
    vint32m1_t x = __riscv_vle32_v_i32m1(a, vl);
    vbool32_t p = __riscv_vmsgt_vx_i32m1_b32(x, 0, vl);
    size_t cnt = __riscv_vcpop_m_b32(p, vl);
    __riscv_vse32_v_i32m1(c + k,
        __riscv_vcompress_vm_i32m1(x, p, vl), cnt);
    k += cnt;
  }
  return k;
}
```

Executed instruction counts for 1,000 elements. Scalar is the same loop compiled with auto-vectorization disabled.

| Kernel | Scalar | VLEN 128 | VLEN 1024 |
|---|---|---|---|
| Masked add | 8,448 | 3,010 | 394 |
| Gather | 8,013 | 2,510 | 330 |
| Sum | 4,011 | 1,760 | 234 |
| `vcompress` filter | 6,405 | 3,511 | 459 |

![Executed instruction counts of the four 1000-element kernels roughly halve each time VLEN doubles; the masked add went from 8448 scalar instructions to 394 at VLEN 1024|xl](/img/01M6K7F9D0SH3ANM7K1VD4QJ0S/b4e6edf3e68b.webp)

That is 31–55% of scalar at VLEN 128 and 4–7% at 1024. The filter has the highest ratio. The scalar filter skips negative values without storing them, so it needs fewer instructions per element, while the vector filter runs `vsetvli` once more to match the store count (14 instructions per iteration). As in Part 13, these are instruction counts, not time. A gather has a different address per element, so real hardware often processes only a few elements at a time (my guess). The more work one instruction does, the bigger the gap between instruction count and time.

For correctness, 19 items per VLEN (hashes and counts for the masked add, gather, and filter; 3 scalar results; the sum; the strided column; segment results and iteration count; 3 permutations; tu/mu and ta/ma values; the `strlen` result; the first-fault vl sequence; trap information from the normal load) plus 2 agnostic all-1s fills, 78 items in total, matched the Python reference. At first, QEMU did not finish at VLEN 1024. In the ta/ma test, I had stored the register into a 16-byte array with `vs1r.v`, but `vs1r.v` writes all VLEN bits (128 bytes at 1024) regardless of vl. It overwrote the stack and corrupted the return address. I learned the hard way that whole-register loads and stores do not look at `vtype`.

## 7. Design Checklist: Decoder and Register File

| Item | Rule |
|---|---|
| Group alignment | Illegal if a vd/vs1/vs2 number is not a multiple of its EMUL (Part 13). For indexed loads, compute the offset EMUL = EEW/SEW × LMUL separately |
| v0 overlap | Reserved if the vd group of a vm=0 instruction overlaps v0. Compares (mask results) and scalar results of reductions are exceptions |
| Permute overlap | For `vslideup`, `vrgather`, `vcompress`, reserved if vd overlaps a source (and `vcompress`'s mask) |
| Segment | NFIELDS × EMUL ≤ 8, last register ≤ v31. Otherwise reserved |
| vill | Instructions that depend on `vtype` are illegal. Whole-register loads/stores, `vmv<n>r`, and `vset*` are exceptions |
| vstart ≠ 0 | Reductions, `vcompress`, `vcpop`/`vfirst`, etc. raise an illegal instruction exception |
| vstart on exception | Write the trapping element index to vstart (QEMU: 40). Reset to 0 when the instruction completes |
| first-fault | Only element 0 traps; after that, reduce vl and continue. Adds a second path that writes vl besides `vset*` |
| LOAD-FP sharing | width 000/101/110/111 is vector. Illegal without V. mew=1 is reserved |
| Read ports | Masked operations read v0 one more time. Undisturbed also needs the old vd (can be skipped with agnostic) |
| vl = 0 | Changes no element of vd, regardless of vstart |

The second-to-last row sets the size of the register file. Executing `vadd.vv vd, vs1, vs2, v0.t` with mu may require reading four registers: vs1, vs2, v0, and the old vd. The indexed store in Section 3 reads two vectors (data and offsets) plus the mask. Many implementations are said to use ta/ma to skip reading the old vd and to keep a separate copy of the mask register (my guess).

On the decoder side, remember that "is it legal?" is not decided by the 32-bit instruction alone. `vtype.vlmul` and `vsew` jointly determine group alignment and segment limits, and `vstart` determines whether a reduction is legal. Adding these inputs to the scalar decoder table of Part 15 is what makes a vector decoder.

## Summary

- Conditions become masks. A compare writes 1 bit per element into `v0` (bit i = element i), and an instruction with vm=0 writes only the elements whose bit is 1. The only mask source is `v0`.
- Inactive and tail elements are preserved under mu/tu, and either preserved or filled with 1s under ma/ta. QEMU preserved them by default and filled them with `0xffffffff` when `rvv_*_all_1s` was enabled. Agnostic eliminates the old-value copy in renaming cores.
- There are four kinds of loads and stores (unit-stride, strided, indexed, segment), and vector loads share the LOAD-FP opcode. At the end of RAM, the first-fault load reduced vl to 40, while the normal load trapped with vstart 40.
- The four kernels (masked add, gather, sum, filter) matched Python on all 78 items, and the executed instruction count was 31–55% of scalar at VLEN 128 and 4–7% at 1024.

**Next post:** Part 1 (the ISA) ends here, and from Part 15 we actually build a RISC-V core. First up is the decoder. I will organize the format table from Part 2 and the instructions from Parts 3–14 into opcode/funct3/funct7 tables, build the logic that identifies the format, and build illegal instruction detection that distinguishes nonexistent opcodes, reserved funct values, HINTs, and disabled extensions.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 31 "V" Standard Extension for Vector Operations, Version 1.0: 31.3.4 `vtype` (vta, vma, vill), 31.3.7 `vstart`, 31.4.5 mask layout, 31.5.3 masking, 31.5.4 prestart, body, and tail, 31.7 loads and stores (31.7.5 strided, 31.7.6 indexed, 31.7.7 first-fault, 31.7.8 segment, 31.7.9 whole register), 31.14 reductions, 31.15 mask instructions, 31.16 permutations, 31.17 exception handling. https://github.com/riscv/riscv-isa-manual
- RISC-V Vector C Intrinsics specification v1.0. https://github.com/riscv-non-isa/rvv-intrinsic-doc
- QEMU RISC-V documentation (`virt` board memory map, CPU properties). https://www.qemu.org/docs/master/system/riscv/virt.html
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 8 RV32V (based on a pre-ratification draft, so for concepts only)
