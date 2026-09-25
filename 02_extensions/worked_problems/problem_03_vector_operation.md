# Problem 03: Vectorised Array Operations Using the V Extension

## Problem Statement

**Context:** You are optimising an embedded signal-processing library for a RISC-V processor with the V extension (`VLEN = 256`, RV32GCV). All inputs are in DRAM; no SIMD registers are pre-loaded. You must write vector-length-agnostic code that runs correctly on any compliant RISC-V V implementation.

---

### Part A: Element-wise Integer Absolute Value

Write a vectorised function that computes the absolute value of each element in a signed 32-bit integer array and writes the results to an output array.

```c
void abs_array(int32_t *dst, const int32_t *src, int n);
```

Constraints:
- `n` may be any non-negative integer including 0.
- `dst` and `src` may overlap.
- Do not use any M-extension instructions.

---

### Part B: Threshold Clipping

Write a vectorised function that clips each element of a 16-bit signed integer array to the range `[lo, hi]`:

```c
void clip_array(int16_t *dst, const int16_t *src, int n, int16_t lo, int16_t hi);
```

Constraints:
- The clip must be correct for the boundary case `lo == hi`.
- Use masked or min/max vector instructions — do not use branches inside the element loop.

---

### Part C: Vectorised Dot Product with Widening Accumulation

Write a vectorised function computing the dot product of two arrays of 8-bit unsigned integers, accumulating into a 32-bit result:

```c
uint32_t dot_u8(const uint8_t *a, const uint8_t *b, int n);
```

Constraints:
- Overflow: the sum of products must not wrap. Assume `n <= 65535` and element values are in `[0, 255]`, so the maximum exact result is `255 * 255 * 65535 = 4,261,413,375`, which fits (just) within the 32-bit unsigned range (4,294,967,295). Show that 32-bit accumulation is therefore exact (document this).
- Use widening multiply-add to avoid multiple precision steps per iteration.

---

### Part D: Performance Analysis

Given `VLEN = 256`, `SEW = 32`, `LMUL = 1`:

1. What is `VLMAX`?
2. How many loop iterations does the `abs_array` function take for `n = 100`?
3. If the vector FP add has 4-cycle throughput-1 latency on this implementation, and the loop body contains one `vle32.v` (3-cycle latency), one `vsra.vi` (1-cycle), one `vadd.vv` (1-cycle), one `vmax.vv` (1-cycle), and one `vse32.v` (3-cycle), is the loop throughput-bound or latency-bound? Identify the bottleneck.

---

## Solution

### Part A: Vectorised Absolute Value

**Algorithm:** For a signed integer `x`, `|x| = max(x, -x)`. Compute `-x` as `0 - x` (using vector subtract from zero or negation), then take element-wise maximum.

Alternatively: `|x| = (x < 0) ? -x : x`, implemented using arithmetic right shift for the sign mask and XOR/ADD, but `vmax.vv` is cleaner and directly available in the base V extension integer instructions.

```asm
# abs_array(int32_t *dst, const int32_t *src, int n)
# a0 = dst, a1 = src, a2 = n

abs_array:
    beqz    a2, .done          # early exit for n=0

.loop:
    vsetvli  t0, a2, e32, m1, ta, ma   # t0 = vl = min(n, VLMAX)

    vle32.v  v1, (a1)          # load vl elements from src
    vrsub.vi v2, v1, 0         # v2 = 0 - v1 = -v1  (vector reverse-subtract from immediate 0)
    vmax.vv  v1, v1, v2        # v1 = max(v1, -v1) = |v1|

    vse32.v  v1, (a0)          # store result to dst

    sub      a2, a2, t0        # remaining elements -= vl
    slli     t1, t0, 2         # bytes per iteration = vl * 4
    add      a0, a0, t1        # advance dst pointer
    add      a1, a1, t1        # advance src pointer

    bnez     a2, .loop

.done:
    ret
```

**Notes on `vrsub.vi`:**

`vrsub.vi vd, vs2, imm` computes `vd[i] = imm - vs2[i]`. With `imm = 0`, this is vector negation: `vd[i] = -vs2[i]`. There is no dedicated `vneg` instruction in the V extension base; `vrsub.vi vd, vs, 0` is the idiomatic negation.

**Correctness for INT_MIN:**

For `x = INT_MIN = 0x80000000`, `-x` overflows: `-INT_MIN = INT_MIN` in two's complement. So `max(INT_MIN, INT_MIN) = INT_MIN`. The absolute value of `INT_MIN` is not representable as a signed 32-bit integer. This is the same behaviour as `abs(INT_MIN)` in C (undefined behaviour, typically returns `INT_MIN`). The code matches C library semantics.

**Overlap correctness:**

If `dst == src`, the `vle32.v` loads from the same address as `vse32.v` stores. Since the vector instructions process exactly `vl` elements and advance both pointers by the same amount, the in-place update is correct: we read `vl` elements, compute absolute values, and write them back before advancing. No element is read after it has been written.

---

### Part B: Threshold Clipping

**Algorithm:** Clipping to `[lo, hi]` is: `result = min(max(x, lo), hi)`. The V extension provides `vmax.vx` (max with scalar) and `vmin.vx` (min with scalar).

```asm
# clip_array(int16_t *dst, const int16_t *src, int n, int16_t lo, int16_t hi)
# RV32 calling convention: a0=dst, a1=src, a2=n, a3=lo, a4=hi

clip_array:
    beqz    a2, .done

    # Sign-extend lo and hi from 16-bit to 32-bit for use in .vx instructions
    sext.h  a3, a3             # Zbb: sign-extend; without Zbb: slli/srai pair
    sext.h  a4, a4

.loop:
    vsetvli  t0, a2, e16, m2, ta, ma   # SEW=16, LMUL=2 for throughput

    vle16.v  v0, (a1)          # load vl int16 elements
    vmax.vx  v0, v0, a3        # v0[i] = max(v0[i], lo)  -- floor at lo
    vmin.vx  v0, v0, a4        # v0[i] = min(v0[i], hi)  -- ceiling at hi
    vse16.v  v0, (a0)          # store result

    sub      a2, a2, t0
    slli     t1, t0, 1         # bytes = vl * sizeof(int16) = vl * 2
    add      a0, a0, t1
    add      a1, a1, t1
    bnez     a2, .loop

.done:
    ret
```

**Why `LMUL=2` for `SEW=16`?**

With `VLEN=256` and `SEW=16`, `VLMAX (LMUL=1) = 256/16 = 16` elements. Using `LMUL=2` doubles this to 32 elements per iteration, reducing loop overhead. We have plenty of registers available (32 vector registers / 2 per group = 16 logical registers — more than enough for this kernel).

**Correctness for `lo == hi`:**

When `lo == hi`, `vmax(x, lo)` produces `max(x, lo)`. If `x > lo`, result is `x`. Then `vmin(x, hi) = vmin(x, lo) = lo` (since `hi == lo` and `x > lo`). If `x <= lo`, `vmax` gives `lo`, then `vmin(lo, hi) = lo`. In all cases, the result is `lo = hi`. Correct.

**Without the `sext.h` instruction (no Zbb):**

```asm
slli  a3, a3, 16
srai  a3, a3, 16    # sign-extend lo from bits 15:0
slli  a4, a4, 16
srai  a4, a4, 16    # sign-extend hi from bits 15:0
```

When XLEN > SEW, a `.vx` instruction uses only the least-significant SEW bits of the scalar register, so for `SEW=16` the pre-extension is not strictly required; it is harmless and makes the register value match the C `int16_t` argument.

---

### Part C: Widening Dot Product

**Algorithm:**

The naïve approach is:
1. Load 8-bit elements.
2. Multiply pairs: `vwmulu.vv` produces 16-bit results from 8-bit inputs.
3. Widen to 32 bits: `vwaddu.wv` widens and accumulates into a 32-bit accumulator.
4. At the end, horizontal reduce to a scalar.

A cleaner single-pass approach uses `vwmaccu.vv` (widening multiply-accumulate): multiply 8-bit × 8-bit and add the 16-bit result into a 16-bit accumulator, then at the outer level, widen and sum the 16-bit partial sums into a 32-bit total.

However, with the constraint that intermediate results must not overflow 32 bits, we need to be careful. Using 32-bit per-lane accumulators from the start is safest:

```asm
# dot_u8(const uint8_t *a, const uint8_t *b, int n) -> uint32_t in a0
# a0 = &a, a1 = &b, a2 = n

dot_u8:
    beqz    a2, .zero_result

    # Initialise 32-bit per-lane accumulator to 0
    # Using e32, m4 for 32-bit accumulators with maximum throughput
    # (VLEN=256, SEW=32, LMUL=4 -> VLMAX = 4*256/32 = 32 lanes)
    vsetvli  t1, zero, e32, m4, ta, ma
    vmv.v.i  v8, 0             # v8..v11 = 0 (32-bit accumulators, m4 group)

.loop:
    # Process using SEW=8 for loads and widening multiply
    # vl for e8,m1 gives VLMAX = 256/8 = 32 elements (matches our 32 accumulator lanes)
    vsetvli  t0, a2, e8, m1, ta, ma    # SEW=8 for 8-bit inputs

    vle8.v   v0, (a0)          # load 8-bit elements from a
    vle8.v   v2, (a1)          # load 8-bit elements from b

    # Widening multiply: 8-bit × 8-bit -> 16-bit
    vwmulu.vv v4, v0, v2       # v4 (e16, m2) = zero-extended product

    # Widen 16-bit products to 32-bit and add to accumulator
    # vwaddu.wv widens vs2 (16-bit) to 32-bit and adds to vd (already 32-bit)
    # Note: vd must be m4, vs2 must be m2 (half the LMUL)
    vsetvli  t0, a2, e16, m2, tu, ma   # reconfigure for 16-bit; tu keeps accumulator tail
    vwaddu.wv v8, v8, v4        # v8 (e32,m4) += zero_extend(v4 (e16,m2))

    # Note: e16,m2 has the same VLMAX (32) as e8,m1, so t0 equals the e8 vl

    sub      a2, a2, t0        # remaining elements -= vl
    add      a0, a0, t0        # advance by bytes = vl (SEW=8, vl = bytes)
    add      a1, a1, t0        # advance b pointer
    bnez     a2, .loop

.reduce:
    # Horizontal sum of 32-bit per-lane accumulators in v8
    vsetvli  t0, zero, e32, m4, ta, ma
    vmv.v.i  v0, 0             # v0 = 0 (initial value for reduction)
    vredsum.vs v0, v8, v0      # v0[0] = sum of all lanes in v8
    vmv.x.s  a0, v0            # extract scalar result to a0
    ret

.zero_result:
    li       a0, 0
    ret
```

**Clarification on the vsetvli switching:**

The in-loop `vsetvli` switch is safe here: with `VLEN = 256`, `e8,m1` and `e16,m2` both have `VLMAX = 32`, so both calls return the same `vl` and the pointer/count updates are correct. The accumulator update uses `tu` (tail-undisturbed) so that, on the final short iteration, lanes beyond `vl` keep their partial sums instead of being overwritten by a tail-agnostic policy before the full-width reduction.

`vwmaccu.vv` is not a drop-in replacement: it widens only once (8-bit × 8-bit into a 16-bit accumulator), so it cannot accumulate directly into 32-bit lanes from 8-bit inputs. The two-step `vwmulu.vv` + `vwaddu.wv` sequence above is the correct approach for 32-bit accumulation.

**Overflow note (as required by problem statement):**

The maximum dot product value is `255 * 255 * 65535`: `255 * 255 = 65025`, and `65025 * 65535 = 4,261,413,375`, which is less than `UINT32_MAX (4,294,967,295)`. So in fact, for the given constraints (`n <= 65535`, values `[0, 255]`), the result fits in a `uint32_t`. The `vredsum.vs` into a 32-bit accumulator is exact.

However, the per-lane 32-bit accumulators accumulate at most `ceil(n / VLMAX) * 255 * 255` per lane. With `VLMAX = 32` and `n = 65535`: `ceil(65535/32) = 2048` iterations per lane, `2048 * 65025 = 133,171,200` per lane — well within `uint32_t`. Safe.

---

### Part D: Performance Analysis

**1. VLMAX for VLEN=256, SEW=32, LMUL=1:**

```
VLMAX = LMUL * VLEN / SEW = 1 * 256 / 32 = 8 elements per vector operation
```

**2. Loop iterations for `abs_array`, n=100:**

```
Full iterations:  floor(100 / 8) = 12   (each processes 8 elements, 12 * 8 = 96)
Last iteration:   1                     (processes 100 - 96 = 4 elements, vl=4)
Total iterations: 13
```

The 13th iteration: `vsetvli` is called with `a2 = 4`, returns `vl = 4`. All vector instructions process only 4 elements. No special tail handling code is needed — VLA handles it.

**3. Throughput vs. latency analysis:**

Loop body for `abs_array`:

| Instruction   | Latency | Throughput (IPC) | Data dependency |
|---------------|---------|-----------------|-----------------|
| `vle32.v`     | 3 cy    | 1/1 (pipelined) | None (loads from memory) |
| `vrsub.vi`    | 1 cy    | 1/1             | Depends on vle32 result |
| `vmax.vv`     | 1 cy    | 1/1             | Depends on vrsub result |
| `vse32.v`     | 3 cy    | 1/1             | Depends on vmax result |

**Dependency chain (critical path):**

```
vle32.v (3) -> vrsub.vi (1) -> vmax.vv (1) -> vse32.v (3)
Total latency chain: 3 + 1 + 1 + 3 = 8 cycles
```

Additionally, the loop overhead:
- `sub` (1 cy), `slli` (1 cy), `add` (1 cy), `add` (1 cy), `bnez` (1 cy) = 5 cycles

The throughput bound: if all instructions issue at 1 per cycle, 1 `vsetvli` + 4 vector instructions + 5 scalar instructions = 10 instructions per loop. If issue width is 1 instruction/cycle, throughput bound = 10 cycles.

**Bottleneck determination:**

- Latency bound: **8 cycles** (critical path through vector instructions).
- Throughput bound: 10 cycles (instruction count, single-issue assumed).

The loop is **throughput-bound** at 10 cycles per iteration on a single-issue in-order machine. The latency chain (8 cycles) is shorter than the instruction count bound (10 cycles), so the processor cannot hide the latency — it will issue the next `vle32.v` before the previous chain completes, but only if it has out-of-order capability. On an in-order machine, each `vle32.v` must wait for the previous `vse32.v` to complete (there is a structural dependency through the loop).

**In-order machine (typical for embedded V implementations):**

On an in-order core, the next loop iteration's `vle32.v` can overlap with the current iteration's `sub`/`slli`/`add`/`bnez` if the load has no dependency on those instructions. Since `vle32.v` in iteration N+1 is independent of the `vse32.v` in iteration N (different addresses), on a machine with a non-blocking vector memory unit, loads from iteration N+1 could begin during the stores of iteration N. Assuming no structural hazard:

Effective cycles per iteration ≈ max(latency of critical path, throughput_instruction_count) ≈ 8-10 cycles.

**At VLMAX=8 elements per iteration and 8-10 cycles per iteration:**

Throughput ≈ `8 / 10` = 0.8 elements per cycle, or about 1.25 cycles per element. A scalar implementation would take at least 3-4 instructions per element (load, negate, max, store), so the vector implementation provides roughly a 2.4-3.2x throughput improvement.

---

## Discussion

### Key Insights from These Problems

**VLA loop structure is non-negotiable.** Every production V extension loop must use the `vsetvli`-at-top-of-loop pattern. Placing `vsetvli` outside the loop and assuming a fixed `vl` will break on hardware with different `VLEN`, and will produce incorrect results for `n` not divisible by `VLMAX`. The overhead of `vsetvli` (typically 1-2 cycles) is negligible for loops with any non-trivial body.

**LMUL selection is a register-throughput trade-off.** For short kernels (few vector instructions), higher LMUL increases `VLMAX` and reduces loop iterations. For complex kernels needing many intermediate vector registers, lower LMUL provides more logical registers. The sweet spot for most kernels is `LMUL=1` to `LMUL=4`.

**Widening operations require matching LMUL pairs.** The rule: if widening multiply takes `SEW`-wide inputs with `LMUL=m`, the output is `2*SEW`-wide with `LMUL=2m`. Planning the register layout before coding — deciding which groups hold inputs at what width — prevents register conflicts. The `vtype` must be explicitly reconfigured when switching between element widths.

**Horizontal reduction is a separate phase.** Vector computation naturally produces per-lane results. Reducing to a scalar (dot product final sum, array maximum, etc.) uses `vredsum.vs`, `vredmax.vs`, etc. These are post-loop operations on the accumulated per-lane results. The `vfredosum.vs` (ordered) vs. `vfredusum.vs` (unordered) distinction matters for reproducibility — unordered reduction has higher throughput but may give different floating-point results on different `VLEN` hardware.

**Performance analysis requires identifying the critical path.** For vector code, the critical path is typically memory latency (load → first dependent operation) rather than arithmetic latency. Scheduling instructions to hide load latency — issuing the load before the data is needed — is the primary optimisation technique for memory-bound vector kernels.
