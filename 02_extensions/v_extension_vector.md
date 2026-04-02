# V Extension: Vector

## Overview

The RISC-V V extension (ratified as version 1.0 in 2021) introduces a scalable vector architecture that fundamentally differs from fixed-width SIMD approaches like SSE/AVX on x86 or NEON on ARM. Rather than encoding a specific vector width in the ISA, RISC-V V uses a vector-length agnostic (VLA) programming model: the application specifies how many elements it wants to process, the hardware decides how many it can handle per operation based on its physical vector length (`VLEN`), and `vsetvli` negotiates the actual number processed per iteration. This design allows the same binary to run correctly and efficiently on hardware ranging from a microcontroller with 128-bit vectors to a server processor with 512-bit or wider vectors, without recompilation.

---

## Key Concepts

### Vector Registers

The V extension adds 32 vector registers `v0`–`v31`. Each register contains `VLEN` bits, where `VLEN` is an implementation parameter (a power of two, minimum 128 bits). `VLEN` is not an architectural constant — it is discovered at runtime.

The registers are used as:
- Individual vector registers: each holds `VLEN / SEW` elements (where `SEW` is the selected element width).
- Register groups (LMUL > 1): multiple adjacent registers treated as one wide vector.
- Mask register: `v0` is special — it serves as the mask register for masked operations.

### Element Width (SEW) and Vector Length (VLEN, VLMAX, AVL)

Three related parameters govern vector operation sizing:

| Parameter | Meaning |
|-----------|---------|
| `VLEN`    | Physical vector register length in bits (hardware constant, e.g., 256) |
| `SEW`     | Selected Element Width in bits (8, 16, 32, or 64) — set by `vsetvli` |
| `LMUL`    | Length MULtiplier — number of physical registers grouped into one logical vector (1/8, 1/4, 1/2, 1, 2, 4, 8) |
| `VLMAX`   | Maximum number of elements per operation = `LMUL * VLEN / SEW` |
| `AVL`     | Application Vector Length — how many elements the application wants to process |
| `vl`      | Active vector length — how many elements will actually be processed (set by `vsetvli`) |

The relationship: `vl = min(AVL, VLMAX)`.

### The `vsetvli` Instruction and Vector-Length Agnostic Programming

`vsetvli rd, rs1, vtypei` is the central instruction of the V extension:

```
vsetvli  vl_out, avl_in, e<SEW>, m<LMUL>, ta/tu, ma/mu
```

- `rd` receives the actual `vl` (number of elements that will be processed).
- `rs1` provides `AVL` (how many elements the application needs to process).
- `vtypei` encodes `SEW`, `LMUL`, tail policy (`ta`/`tu`), and mask policy (`ma`/`mu`).

**Vectorised loop idiom:**

```asm
# Process n elements of float array (SEW=32, LMUL=1)
# a0 = pointer to array, a1 = n (element count)
.loop:
    vsetvli  t0, a1, e32, m1, ta, ma   # t0 = vl (elements this iteration)
    vle32.v  v1, (a0)                   # load t0 elements
    # ... process ...
    vse32.v  v1, (a0)                   # store t0 elements
    slli     t1, t0, 2                  # bytes processed = vl * 4
    add      a0, a0, t1                 # advance pointer
    sub      a1, a1, t0                 # remaining = n - vl
    bnez     a1, .loop                  # continue if more remain
```

This loop is **vector-length agnostic**: the same code runs on any VLEN. On hardware with `VLEN=128` and `SEW=32`, `VLMAX=4`, so each iteration processes 4 elements. On hardware with `VLEN=512`, `VLMAX=16`, so each iteration processes 16 elements. The loop count adjusts automatically.

### LMUL: Length Multiplier

LMUL groups multiple physical registers into a single logical vector register, increasing the number of elements per operation:

| LMUL | Physical registers per logical | Effective width | Use case |
|------|-------------------------------|-----------------|----------|
| 1/8  | 1/8 of one                   | VLEN/8 bits     | Short vectors, mixed-precision |
| 1/4  | 1/4 of one                   | VLEN/4 bits     | |
| 1/2  | 1/2 of one                   | VLEN/2 bits     | |
| 1    | 1                            | VLEN bits       | Standard (default) |
| 2    | 2                            | 2*VLEN bits     | Wider logical vectors |
| 4    | 4                            | 4*VLEN bits     | |
| 8    | 8                            | 8*VLEN bits     | Maximum width |

With `LMUL=8` and `VLEN=256`, `SEW=32`: `VLMAX = 8*256/32 = 64` elements per operation, using all 32 vector registers as 4 groups of 8.

Fractional LMUL (`m1/2`, `m1/4`, `m1/8`) uses a fraction of a register. This allows mixing different element widths in the same code without exhausting register space — e.g., using `e8, m1` for bytes and `e32, m4` for the widened results in the same routine.

### Memory Operations

| Category | Instructions |
|----------|-------------|
| Unit-stride load/store | `vle8.v`, `vle16.v`, `vle32.v`, `vle64.v` / `vse*` |
| Strided load/store | `vlse32.v rs1, rs2` — stride in bytes in `rs2` |
| Indexed (gather/scatter) | `vluxei32.v` (unordered), `vloxei32.v` (ordered) |
| Whole-register | `vl1re32.v`, `vs1r.v` — load/store whole vector register |
| Fault-only-first | `vle32ff.v` — load with fault-only-first semantics |
| Segment loads/stores | `vlseg2e32.v` etc. — for interleaved structure-of-arrays ↔ array-of-structures |

**Fault-only-first loads** (`vle32ff.v`) are a key feature for string/null-terminated operations: the load proceeds until either all `vl` elements are loaded or a fault occurs on any element past the first. On a fault, `vl` is reduced to the number of elements successfully loaded before the fault. This allows safe speculative loading past string boundaries into potentially unmapped memory.

### Arithmetic and Reduction Operations

The V extension provides a complete set of arithmetic instructions parameterised by element type and width:

| Category | Example mnemonics |
|----------|-------------------|
| Integer add/sub | `vadd.vv`, `vadd.vx`, `vadd.vi` |
| Integer mul | `vmul.vv`, `vmulh.vv` |
| Integer div | `vdiv.vv`, `vdivu.vv` |
| Widening ops | `vwmul.vv` (result is 2× SEW), `vwadd.vv` |
| Narrowing ops | `vnsrl.wi`, `vnsra.wi` (truncate to SEW/2) |
| FP arithmetic | `vfadd.vv`, `vfmul.vv`, `vfmacc.vv` |
| FP reduction | `vfredusum.vs`, `vfredmax.vs` |
| Integer reduction | `vredsum.vs`, `vredmax.vs`, `vredmin.vs` |
| Comparison | `vmseq.vv`, `vmslt.vv` → writes mask to `vd` |
| Mask operations | `vmand.mm`, `vmor.mm`, `vmnot.m` |
| Permutation | `vrgather.vv` (scatter), `vcompress.vm` |
| Slide | `vslideup.vx`, `vslidedown.vx` |

Instruction suffixes:
- `.vv`: vector op vector (element-wise)
- `.vx`: vector op scalar (broadcast scalar from integer register)
- `.vi`: vector op immediate (small signed immediate broadcast)
- `.vs`: vector op scalar vector (for reductions: result in first element of `vd`)

### Masking

Every vector instruction can be optionally masked using `v0.t` (when bit `vm=0` in the instruction encoding). Masked elements either keep their previous value (mask-undisturbed) or are set to all-ones (mask-agnostic), depending on the mask policy set in `vsetvli`.

```asm
# Zero elements where a0[i] < 0 (keep others unchanged)
vmslt.vx  v0, v1, x0       # v0 = mask where v1[i] < 0
vmv.v.i   v2, 0            # v2 = 0 (will be merged)
vmerge.vvm v1, v1, v2, v0  # v1[i] = (v0[i] ? 0 : v1[i])
```

### The `vtype` CSR and `vl` CSR

`vtype` (CSR `0xC21`) holds the current vector configuration: SEW, LMUL, tail policy, mask policy. It is set by `vsetvli`/`vsetivli`/`vsetvl`.

`vl` (CSR `0xC20`) holds the current active vector length. Always read the `rd` output of `vsetvli` rather than CSR-reading `vl` — it is faster and avoids a CSR read dependency.

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What is vector-length agnostic (VLA) programming, and why is it superior to a fixed-width SIMD approach like SSE/AVX?**

**Answer:**

In fixed-width SIMD (SSE/AVX/NEON), the vector width is baked into the ISA: `ADDPS` always adds 4 floats (128-bit SSE), `VADDPS` always adds 8 floats (256-bit AVX), or 16 floats (512-bit AVX-512). Code written for AVX2 does not automatically benefit from AVX-512 hardware — it must be rewritten with wider intrinsics, requiring a new binary.

In VLA programming (RISC-V V extension), the application specifies how many elements it needs to process (AVL), and the hardware specifies how many it can handle per operation (VLMAX). The `vsetvli` instruction negotiates `vl = min(AVL, VLMAX)`. The application loops until all elements are processed, with `vl` automatically being VLMAX for full-width iterations and the remainder for the last partial iteration.

**Advantages of VLA:**

1. **Binary compatibility:** one binary runs efficiently on a 128-bit implementation (e.g., a microcontroller) and a 512-bit implementation (e.g., a server CPU). No recompilation needed.

2. **Future-proofing:** when wider hardware ships, old binaries automatically use the wider vectors without modification.

3. **Loop tail handling:** the last iteration with fewer-than-VLMAX elements is handled naturally — `vsetvli` returns `vl = remaining elements`. No special scalar cleanup loop needed.

4. **Reduced ISA footprint:** instead of separate instructions for 128-bit, 256-bit, and 512-bit widths, one instruction set covers all widths.

**Disadvantage:** slightly more complex loop structure (the `vsetvli` negotiation). Hardware implementations are also more complex than fixed-width SIMD.

---

**Q2. What does `vsetvli t0, a1, e32, m1, ta, ma` do? What is stored in `t0` after this instruction?**

**Answer:**

This instruction configures the vector unit and returns the active vector length:

- `e32`: element width (SEW) = 32 bits (each element is a 32-bit value).
- `m1`: LMUL = 1 (one physical register per logical vector).
- `ta`: tail agnostic — tail elements (beyond `vl`) may be overwritten with any value.
- `ma`: mask agnostic — masked-off elements may be overwritten with any value.
- `a1`: AVL — the application is requesting to process `a1` elements.
- `t0`: receives the actual `vl` — the number of elements that will be processed.

`t0 = min(a1, VLMAX)` where `VLMAX = VLEN / 32` (with `m1` and `e32`).

Example: if hardware has `VLEN = 256`:
- `VLMAX = 256 / 32 = 8` elements per vector register.
- If `a1 = 20` (20 elements remain), `t0 = 8`.
- If `a1 = 5` (last partial iteration), `t0 = 5`.

After `vsetvli`, subsequent vector loads, stores, and arithmetic instructions process exactly `t0` elements.

---

**Q3. What is the difference between LMUL=1, LMUL=2, and LMUL=8? When would you use LMUL > 1?**

**Answer:**

`LMUL` (Length Multiplier) controls how many physical vector registers are grouped into one logical vector:

- **LMUL=1:** each logical vector register is one physical register. `VLMAX = VLEN / SEW`. You have 32 independent logical vector registers.
- **LMUL=2:** each logical vector is two consecutive physical registers. `VLMAX = 2 * VLEN / SEW`. But you only have 16 distinct logical registers (v0, v2, v4, ... v30 — even-numbered only).
- **LMUL=8:** each logical vector is 8 consecutive physical registers. `VLMAX = 8 * VLEN / SEW`. You have only 4 logical registers (v0, v8, v16, v24).

**When to use LMUL > 1:**

1. **Low VLEN implementations:** a microcontroller with `VLEN=128` processes only 4 floats per operation at `LMUL=1, SEW=32`. Using `LMUL=4` groups four 128-bit registers into one 512-bit logical vector, enabling 16-float operations at the cost of register count.

2. **Long loops with wide data:** when the application's inner loop has few registers but long vectors, LMUL > 1 increases effective vector width and reduces loop overhead.

3. **Widening operations:** `vwmul.vv` (widening multiply) takes two `SEW`-wide inputs and produces `2*SEW`-wide results. The result needs twice as many bits, requiring the next LMUL tier. For example, multiplying `e8, m1` inputs (8-bit) and storing `e16, m2` results (16-bit, 2× registers).

**When to use fractional LMUL:**

When mixing different element widths in a computation, fractional LMUL allows sharing register space. Using `e8, m1/2` and `e32, m2` in the same routine allows both to occupy the same 32-register file space, with the narrow-element registers using half of each physical register.

---

**Q4. Describe the vectorised loop structure for processing an array of `n` 32-bit integers. Show the complete assembly.**

**Answer:**

```asm
# Multiply each element of array by 2 (shift left by 1)
# a0 = pointer to int32 array, a1 = number of elements
vector_double:
.loop:
    vsetvli  t0, a1, e32, m1, ta, ma  # t0 = vl = min(a1, VLMAX)
    beqz     t0, .done                 # if vl == 0, nothing to do (a1 was 0)
    vle32.v  v1, (a0)                  # load vl elements from a0
    vsll.vi  v1, v1, 1                 # v1 = v1 << 1 (multiply by 2)
    vse32.v  v1, (a0)                  # store vl elements back to a0
    sub      a1, a1, t0                # remaining elements -= vl
    slli     t1, t0, 2                 # bytes advanced = vl * sizeof(int32) = vl * 4
    add      a0, a0, t1                # advance pointer
    bnez     a1, .loop                 # loop while elements remain
.done:
    ret
```

The key insight: the loop body is identical for every iteration. The first `VLMAX - 1` iterations process VLMAX elements each; the final iteration processes `n % VLMAX` elements (or VLMAX if perfectly divisible). `vsetvli` handles this transparently — it returns `vl = remaining` for the last partial iteration, and subsequent instructions automatically apply only to those elements.

No special scalar cleanup loop is needed for the tail. The `ta` (tail agnostic) policy means we do not care about elements beyond `vl` in the destination register.

---

**Q5. What is `v0` special for in the V extension? How is a masked operation written?**

**Answer:**

`v0` is the dedicated **mask register**. Every vector instruction that supports masking reads its per-element enable bits from `v0`. Each bit of `v0` corresponds to one element of the current active vector length; if bit `i` is 1, element `i` is active (processed); if bit `i` is 0, element `i` is masked off.

Masking is indicated by the `vm` bit in the instruction encoding:
- `vm=1`: unmasked (all elements active).
- `vm=0`: masked by `v0.t` (only elements where `v0[i]=1` are processed).

Example — masked addition: add scalar `a1` to elements of `v2` only where `v3[i] > 0`:

```asm
# a0 = base pointer, a1 = scalar addend
# First, create mask in v0: v0[i] = (v2[i] > 0)
vmsgt.vi  v0, v2, 0           # v0[i] = (v2[i] > 0)  (vmset.vi with immediate 0)

# Add a1 to v2 only where mask is set (masked undisturbed)
vsetvli  t0, a1_count, e32, m1, tu, mu    # tu=tail undisturbed, mu=mask undisturbed
vadd.vx  v2, v2, a1, v0.t               # v2[i] += a1 where v0[i]==1; v2[i] unchanged otherwise
```

The `v0.t` suffix in the instruction denotes use of `v0` as the mask. Mask undisturbed (`mu`) means masked-off elements retain their original value — important for partial update semantics.

`v0` can be loaded/modified like any other vector register using `vle8.v v0, (addr)` or computed using mask arithmetic instructions (`vmand.mm`, `vmor.mm`, `vmxor.mm`, `vmnot.m`).

---

### Intermediate Tier

---

**Q6. Explain the tail policy (`ta`/`tu`) and mask policy (`ma`/`mu`) set by `vsetvli`. When does the distinction matter?**

**Answer:**

**Tail policy** controls what happens to elements in the destination register beyond the active `vl`:

- `ta` (tail agnostic): the implementation may write any value to tail elements. Software must not rely on their content.
- `tu` (tail undisturbed): tail elements retain their previous value in the destination register.

**Mask policy** controls what happens to masked-off elements (where `v0[i]=0` in a masked operation):

- `ma` (mask agnostic): masked-off elements may be overwritten with any value.
- `mu` (mask undisturbed): masked-off elements retain their previous value.

**When the distinction matters:**

1. **`tu` for partial register accumulation:** if you are building a result into a vector register and only want to update the first `vl` elements, leaving the rest intact (e.g., accumulating a histogram), use `tu`. With `ta`, the hardware might corrupt elements beyond `vl`.

2. **`mu` for conditional assignment:** when using masking to conditionally update some elements while keeping others — e.g., `if (a[i] > 0) b[i] = f(a[i])` — use `mu` to ensure the unmasked elements of `b` are not overwritten.

3. **`ta`/`ma` for performance:** `ta`/`ma` allows the hardware to skip clearing or preserving tail/mask-off values. On implementations where this requires extra work, using `ta`/`ma` can improve throughput. For most computations where you will overwrite the full vector on each iteration, `ta`/`ma` is correct and potentially faster.

4. **ABI considerations:** C intrinsics for the V extension expose these policies. `vuint32m1_t` vectors carry implicit `ta`/`tu` semantics depending on the intrinsic variant. Getting the policy wrong causes subtle bugs where stale data leaks into computations.

---

**Q7. Implement a vectorised SAXPY operation (`y[i] = a * x[i] + y[i]`) using the V extension. Show the assembly for both RV32 and RV64 base with V.**

**Answer:**

SAXPY (Single-precision A times X Plus Y) is the Level-1 BLAS reference operation. On RISC-V V:

```asm
# saxpy: y[i] = a * x[i] + y[i] for i in [0, n)
# Arguments: a0=n, fa0=a (float scalar), a1=&x[0], a2=&y[0]
# Works on both RV32 and RV64 (same V extension instructions)

saxpy:
    fmv.x.w  t0, fa0         # move float scalar 'a' to integer register for vfmv
    # Actually, use vfmv.v.f to broadcast scalar from FP register to vector
.loop:
    vsetvli  t0, a0, e32, m8, ta, ma   # LMUL=8 for maximum throughput
    beqz     t0, .done
    vle32.v  v0, (a1)                   # v0 = x[0..vl-1]
    vle32.v  v8, (a2)                   # v8 = y[0..vl-1]  (m8: v8 is next group)
    vfmacc.vf v8, fa0, v0               # v8 = fa0 * v0 + v8  (fused multiply-accumulate)
    vse32.v  v8, (a2)                   # store y[0..vl-1]
    sub      a0, a0, t0                 # remaining elements
    slli     t1, t0, 2                  # bytes = vl * 4
    add      a1, a1, t1                 # advance x pointer
    add      a2, a2, t1                 # advance y pointer
    bnez     a0, .loop
.done:
    ret
```

Key points:

1. `LMUL=8` doubles the effective vector width, keeping the register file fully utilised when `VLEN >= 128`. On `VLEN=512`, this processes 128 floats per iteration.

2. `vfmacc.vf v8, fa0, v0` is `v8[i] += fa0 * v0[i]` — the fused multiply-accumulate variant that accumulates into `v8`. This is one instruction for what would be `vfmul` + `vfadd` (two instructions, two roundings).

3. The scalar `fa0` is broadcast to all lanes by the `.vf` (vector-float) variant of `vfmacc`.

4. With `LMUL=8` and two vector register groups, only `v0`–`v7` and `v8`–`v15` are consumed, leaving `v16`–`v31` free for other uses.

---

**Q8. What is the fault-only-first load (`vle32ff.v`)? Give a use case where it is essential.**

**Answer:**

`vle32ff.v vd, (rs1)` is a unit-stride vector load that behaves like `vle32.v` but with one difference: if any element past the first encounters a memory fault (page fault, access fault), instead of raising an exception, the instruction sets `vl` to the index of the first faulting element and loads only the elements before it. Only if the **first** element faults does the instruction raise a normal exception.

**Semantics:**

```
Load elements 0..vl-1. If element 0 faults: raise exception (normal).
If element k > 0 faults: set vl = k, do not raise exception.
Elements 0..k-1 are loaded. Elements k..old_vl-1 are undefined.
```

**Use case: null-terminated string processing.**

A string of unknown length may end near a page boundary. A naive vector load of 16 bytes near the end of a page would fault when it crosses into the next unmapped page — even if the null terminator was within the current page.

```asm
# strlen using fault-only-first: safely load up to 16 bytes without knowing length
# a0 = string pointer
strlen_vff:
    mv    t4, a0            # save start
.loop:
    vsetvli  t0, zero, e8, m1, ta, ma   # max elements
    vle8ff.v v1, (a0)                   # load up to vl bytes, stopping before page fault
    csrr  t0, vl                        # read actual vl (may have been reduced by fault)
    # Check for null byte in loaded data
    vmseq.vi  v0, v1, 0                 # v0[i] = (v1[i] == 0)
    vfirst.m  t1, v0                    # t1 = index of first set bit in v0 (first null), -1 if none
    bgez  t1, .found                    # found null byte
    add   a0, a0, t0                    # advance by actual vl (may be < VLMAX if fault)
    j     .loop

.found:
    add   a0, a0, t1        # a0 = address of null byte
    sub   a0, a0, t4        # length = null_addr - start
    ret
```

Without fault-only-first, the loop must carefully avoid loading past the last page boundary of the string, requiring complex alignment checks or a scalar fallback for the tail. With fault-only-first, the vector loop safely speculates past string bounds.

---

**Q9. Explain widening and narrowing operations in the V extension. When would you use `vwmul.vv` instead of `vmul.vv`?**

**Answer:**

**Widening operations** produce a result that is twice the width of the inputs:

- `vwmul.vv vd, vs2, vs1` (widening signed multiply): each element of `vs1` and `vs2` is `SEW` bits; the result in `vd` is `2*SEW` bits per element. `vd` must use the next LMUL tier (e.g., `vs1/vs2` are `e8, m1`; `vd` is `e16, m2`).

**Narrowing operations** produce a result that is half the width of the input:

- `vnsrl.wi vd, vs2, imm`: narrow shift-right logical — shifts each `2*SEW`-wide element right by `imm` bits and truncates to `SEW` bits.

**When to use `vwmul.vv`:**

Use `vwmul.vv` when the product of two `SEW`-wide values may overflow `SEW` bits — i.e., when you need the full `2*SEW` result.

Example: multiplying 8-bit pixel values. Two 8-bit values (0–255) multiply to give a result up to 65025, which requires 16 bits. Using `vmul.vv` at `SEW=8` would overflow and lose the high bits.

```asm
# Multiply two uint8 arrays and store uint16 results
# a0 = &src1, a1 = &src2, a2 = &dst (uint16), a3 = n
.loop:
    vsetvli  t0, a3, e8, m1, ta, ma    # SEW=8 for inputs
    vle8.v   v0, (a0)                   # load 8-bit elements
    vle8.v   v2, (a1)
    vwmulu.vv v4, v0, v2               # v4 = zero-extended 16-bit products (m2 group)
    # Now store with SEW=16
    vsetvli  t0, a3, e16, m2, ta, ma   # reconfigure for 16-bit output
    vse16.v  v4, (a2)
    sub      a3, a3, t0
    slli     t1, t0, 1                  # bytes: 16-bit elements
    add      a1_8, ..., t1  # adjust 8-bit source pointer differently...
    bnez     a3, .loop
```

The LMUL relationships for widening:

| Input LMUL | Output LMUL |
|------------|-------------|
| m1         | m2          |
| m2         | m4          |
| m4         | m8          |
| m1/2       | m1          |

---

**Q10. What is the `vrgather` instruction? Give a concrete example of when scatter/gather is needed.**

**Answer:**

`vrgather.vv vd, vs2, vs1` implements a **gather** operation: for each element `i`, `vd[i] = vs2[vs1[i]]`. The elements of `vs1` are indices into `vs2`. Elements of `vs1` that are out of range produce zero in `vd`.

`vrgather.vi vd, vs2, imm` gathers the same element (at fixed index `imm`) into all lanes.

**Scatter** (indexed store) is provided by `vsuxei<SEW>.v vd, (rs1), vs2` (unordered indexed store).

**Concrete example: byte-shuffle / table lookup (SBOX in AES).**

The AES S-Box substitution replaces each byte with a fixed lookup table value. With `vrgather`:

```asm
# Load the 256-byte AES S-Box into v16..v19 (m4, e8 = 256 bytes at VLEN=512)
# a0 = &sbox, a1 = &data, a2 = n bytes
la    t0, sbox_table
vl4re8.v  v16, (t0)          # load 256-byte S-Box into v16 (requires VLEN >= 512, m4)

.loop:
    vsetvli  t0, a2, e8, m1, ta, ma
    vle8.v   v0, (a1)         # load input bytes
    vrgather.vv v4, v16, v0   # v4[i] = sbox[v0[i]]  (table lookup)
    vse8.v   v4, (a1)
    sub      a2, a2, t0
    add      a1, a1, t0
    bnez     a2, .loop
```

Without `vrgather`, each byte would require an individual `LBU` from the S-Box table — 16 separate loads per 16-byte chunk. With `vrgather`, the entire chunk is substituted in one instruction.

Other gather use cases:
- Decompression: expand sparse data with known index map.
- Permutation networks: shuffle vector elements for radix-sort or FFT butterfly reordering.
- String operations: character-class lookup tables (e.g., `isalpha`, `toupper` on a full XLEN-wide chunk).

---

### Advanced Tier

---

**Q11. A developer writes a vectorised matrix transpose for a 4×4 float matrix. Describe the algorithm using RISC-V V instructions, paying attention to which V instructions implement the data rearrangement.**

**Answer:**

A 4×4 float matrix transpose requires swapping elements across the main diagonal. The standard SIMD approach uses an interleave/deinterleave pattern.

For `VLEN >= 128`, `SEW=32`, `LMUL=1`: each row fits in one vector register (4 elements).

```asm
# Transpose 4x4 float matrix stored row-major at a0
# Row 0: v0, Row 1: v1, Row 2: v2, Row 3: v3
vsetvli  zero, zero, e32, m1, ta, ma

# Load all four rows
vle32.v  v0, (a0)             # row 0: [a00 a01 a02 a03]
addi     t0, a0, 16
vle32.v  v1, (t0)             # row 1: [a10 a11 a12 a13]
addi     t0, a0, 32
vle32.v  v2, (t0)             # row 2: [a20 a21 a22 a23]
addi     t0, a0, 48
vle32.v  v3, (t0)             # row 3: [a30 a31 a32 a33]

# Step 1: interleave rows 0+1 and rows 2+3
# Use vlsseg2e32.v (strided segment load) for a cleaner approach,
# OR use indexed gather for the transpose permutation.

# Index vectors for transpose permutation:
# To transpose, col j of output = row j of input.
# Element (i,j) of output = element (j,i) of input.
# For a 4x4 matrix stored flat, element (r,c) is at index r*4+c.
# Output row r (which becomes input column r):
#   output[r*4+0] = input[0*4+r] = input[r]
#   output[r*4+1] = input[1*4+r] = input[4+r]
#   output[r*4+2] = input[2*4+r] = input[8+r]
#   output[r*4+3] = input[3*4+r] = input[12+r]

# For output row 0: gather indices [0, 4, 8, 12]
# For output row 1: gather indices [1, 5, 9, 13]
# For output row 2: gather indices [2, 6, 10, 14]
# For output row 3: gather indices [3, 7, 11, 15]

# Load all 16 elements flat into v8..v11 (m4 gives 16 elements at SEW=32, VLEN=128)
vsetvli  zero, zero, e32, m4, ta, ma
vle32.v  v8, (a0)              # all 16 elements: [a00..a33]

# Load index vectors
la       t0, transpose_indices
vle32.v  v4, (t0)              # [0, 4, 8, 12] for output row 0

# Gather transposed rows
vsetvli  zero, zero, e32, m1, ta, ma
# Output row 0 = gather from v8 at [0, 4, 8, 12]:
vrgather.vi  v0, v8_base, ...   # vrgather needs careful register management

# Practical approach for 4x4 on VLEN=512 (all 16 elements in one m1 register):
# Load all 16 elements into v0 (at VLEN=512, SEW=32, m1: VLMAX=16)
vsetvli  zero, zero, e32, m1, ta, ma  # VLEN=512: VLMAX=16
vle32.v  v0, (a0)
# Index vector for transpose: [0,4,8,12, 1,5,9,13, 2,6,10,14, 3,7,11,15]
la       t0, idx_transpose
vle32.v  v1, (t0)
vrgather.vv  v2, v0, v1        # v2 = transposed flat array
vse32.v  v2, (a0)              # store back
```

On implementations with `VLEN >= 512`, the entire 16-element transpose is a single `vrgather.vv`. On implementations with smaller `VLEN`, a multi-step approach using `vlsseg` (strided segment loads to load columns) is more practical:

```asm
# Using strided segment loads for VLEN-agnostic 4x4 transpose:
# Load columns as rows using stride=16 bytes (one float per row)
li       t1, 16
vlsseg4e32.v  v0, (a0), t1     # v0=col0, v1=col1, v2=col2, v3=col3
# Now v0 = [a00,a10,a20,a30], etc. — these ARE the output rows
# Store them as rows:
vsetvli  t0, zero, e32, m1, ta, ma
vss32.v  v0, (a0)              # This approach uses segment loads elegantly
```

---

**Q12. Describe the RVV vector register state context-switch requirements. Why are they more complex than the standard integer register save/restore?**

**Answer:**

The complexity arises from three factors: variable-width registers, the `vtype`/`vl` CSR state, and the fact that `VLEN` is implementation-defined.

**State that must be saved/restored:**

1. **32 vector registers v0–v31**, each `VLEN` bits wide. Since `VLEN` is unknown at compile time, the OS must discover it at boot and allocate `32 * VLEN/8` bytes per thread's vector context.

2. **`vtype` CSR (0xC21):** the current SEW, LMUL, tail policy, mask policy. The kernel must save this value and restore it so the resumed thread sees the same vector configuration.

3. **`vl` CSR (0xC20):** the current active vector length. Must be saved and restored. Simply restoring `vtype` is not enough — `vl` also changes.

4. **`vcsr` (0x00F):** contains `vxrm` (fixed-point rounding mode) and `vxsat` (saturation flag). Must be included in context.

**Challenges vs. integer registers:**

- **Unknown size:** `x0–x31` are always `XLEN` bytes each — fixed. `v0–v31` are `VLEN/8` bytes each, and `VLEN` varies by implementation. The OS context-switch code cannot be a fixed-size `sw`/`lw` sequence; it must adapt to `VLEN`.

- **Efficient save/store with whole-register instructions:** RISC-V provides `vs1r.v`, `vs2r.v`, `vs4r.v`, `vs8r.v` (whole-register vector stores) and matching loads. These are designed for efficient context save/restore without needing to configure `vtype` first.

- **Lazy context switch:** because vector state is large (up to 32 × 512 bits = 2 KB for `VLEN=512`), OSes often implement lazy save: do not save vector state on every context switch. Instead, mark the vector unit dirty and save only when another thread actually uses the vector unit or when the current thread blocks for a long time. Linux uses this approach (the `VS_DIRTY` / `VS_CLEAN` state in `sstatus.VS`).

- **The `vstart` CSR:** if a trap occurs in the middle of a vector instruction (e.g., a page fault during a vector load), `vstart` records the element index at which execution was interrupted. When resuming, the instruction restarts from that element. The OS must save and restore `vstart` and understand that the interrupted vector instruction may need to be re-executed from the saved element index.

---

**Q13. A signal-processing kernel computes a dot product of two float arrays using the V extension. Implement the kernel and explain how vector reduction (`vfredusum.vs`) differs from a scalar reduction loop.**

**Answer:**

```asm
# dot_product(float *a, float *b, int n) -> float in fa0
# a0 = &a, a1 = &b, a2 = n
dot_product:
    # Initialise accumulator vector to 0.0
    vsetvli  t0, a2, e32, m4, ta, ma   # LMUL=4 for throughput
    vmv.v.i  v0, 0                      # v0 = 0.0 (all elements)
    # Note: vmv.v.i uses integer 0; for float, better to use fmv.v.f:
    fmv.w.x  ft0, zero                  # ft0 = 0.0
    vfmv.v.f v0, ft0                    # v0[i] = 0.0 for all i (accumulator)

.loop:
    vsetvli  t0, a2, e32, m4, ta, ma
    beqz     t0, .reduce
    vle32.v  v8,  (a0)                  # v8  = a[0..vl-1]
    vle32.v  v12, (a1)                  # v12 = b[0..vl-1]
    vfmacc.vv v0, v8, v12              # v0  += a[i] * b[i] (element-wise FMA into accumulator)
    sub      a2, a2, t0
    slli     t1, t0, 2
    add      a0, a0, t1
    add      a1, a1, t1
    j        .loop

.reduce:
    # Horizontal sum of v0 into scalar
    vsetvli  t0, zero, e32, m4, ta, ma
    vfmv.v.f v16, ft0                   # v16[0] = 0.0 (initial value for reduction)
    vfredusum.vs v16, v0, v16           # v16[0] = ordered sum of all v0 elements
    vfmv.f.s fa0, v16                   # fa0 = v16[0] (extract scalar result)
    ret
```

**How `vfredusum.vs` differs from a scalar loop:**

A scalar reduction loop processes one element per cycle:
```c
float sum = 0.0f;
for (int i = 0; i < n; i++) sum += dot[i];  // n cycles, n-1 add latencies chained
```

This is latency-bound: each addition depends on the previous one. With 4-cycle FP add latency and n=1024 elements, the scalar loop takes roughly 4096 cycles.

`vfredusum.vs` sums all `vl` elements of `vs2` into the first element of `vd`, initialised from the scalar `vs1[0]`. Internally:

1. The VLMAX elements are summed using a reduction tree (logarithmic depth for unordered sum) or sequentially (for ordered sum).
2. The result is a single scalar in `vd[0]`.

With LMUL=4 and VLEN=256, each `vfredusum` reduces 32 floats. In the loop above, 32 elements are FMA'd into `v0` per iteration (element-wise), then `vfredusum` collapses `v0` into a scalar at the end. The loop has only `n/32` iterations instead of `n`, dramatically reducing loop overhead.

**Unordered vs. ordered reduction:** `vfredusum.vs` may sum in any order (faster, allows tree reduction). `vfredosum.vs` (ordered) sums in strict element order (same as the scalar loop), giving reproducible results but potentially lower throughput.

---

**Q14. Explain how the V extension interacts with the RISC-V memory model. Specifically, what ordering guarantees do vector memory operations have relative to scalar memory operations on other harts?**

**Answer:**

Vector memory operations (`vle*`, `vse*`, `vlse*`, etc.) participate in the RVWMO memory model the same way scalar loads and stores do — they are subject to the same ordering rules.

**Key properties:**

1. **Within a hart:** vector loads and stores are ordered relative to each other and to scalar loads/stores by the same rules as scalar operations. Specifically, a later instruction in program order cannot bypass an earlier vector store to the same address (within the same hart).

2. **Cross-hart visibility:** vector stores become globally visible to other harts in the same relaxed manner as scalar stores. No implicit fences are added by vector operations.

3. **`FENCE` applicability:** `FENCE RW, RW` orders vector loads/stores before and after the fence, just as it does for scalar operations.

4. **Atomic interactions:** vector operations are not atomic at the element level — they do not provide AMO-style atomicity. A vector store of multiple elements is not an atomic multi-element write; other harts may observe a partial store (some elements written but not others). This is the same as scalar stores: a `VSE32.V` storing 8 elements is not guaranteed to appear atomically as 8 stores to other harts.

5. **No implicit acquire/release:** unlike AMO instructions which carry `aq`/`rl` bits, vector load/store instructions have no memory ordering bits. Ordering must be enforced with explicit `FENCE` instructions.

**Practical implication:**

A producer hart writes a vector result and signals a consumer hart:
```asm
# Producer:
vse32.v  v0, (a0)          # write vector result to shared buffer
fence    w, w               # ensure all previous stores are visible before flag
sw       t0, 0(a1)          # write flag to indicate data is ready

# Consumer:
.poll:
    lw   t0, 0(a1)
    beqz t0, .poll          # wait for flag
fence  r, r                 # ensure flag load is ordered before data load
vle32.v v0, (a0)            # safe to read vector data now
```

The `FENCE W, W` on the producer and `FENCE R, R` on the consumer ensure the vector data is visible before the flag (producer) and the flag read is ordered before the vector load (consumer). Without these fences, the consumer might read stale vector data even after seeing the flag.
