# F and D Extensions: Floating-Point

## Overview

The F extension adds 32 single-precision floating-point registers and the instructions to operate on them. The D extension builds on F to add 64-bit double-precision support, sharing the same register file but widening each register to 64 bits. Together, F and D implement IEEE 754-2008 binary32 and binary64 arithmetic with five software-selectable rounding modes and a complete set of exception flags. Understanding NaN boxing, the FCSR register, and the subtleties of rounding-mode semantics distinguishes candidates who have worked with floating-point hardware from those who have only used it from C.

---

## Key Concepts

### Register File

The F/D extensions add 32 floating-point registers `f0`–`f31`, each 64 bits wide when D is implemented (32 bits wide when only F is present). These are completely separate from the integer register file `x0`–`x31`. The ABI names and calling convention assignments are:

| Registers      | ABI Names        | Role                       | Saved by  |
|----------------|------------------|----------------------------|-----------|
| `f0`–`f7`      | `ft0`–`ft7`      | Temporaries                | Caller    |
| `f8`–`f9`      | `fs0`–`fs1`      | Saved                      | Callee    |
| `f10`–`f11`    | `fa0`–`fa1`      | Arguments / return values  | Caller    |
| `f12`–`f17`    | `fa2`–`fa7`      | Arguments                  | Caller    |
| `f18`–`f27`    | `fs2`–`fs11`     | Saved                      | Callee    |
| `f28`–`f31`    | `ft8`–`ft11`     | Temporaries                | Caller    |

### Instruction Categories

| Category              | Mnemonics (F/D prefixed)                         |
|-----------------------|--------------------------------------------------|
| Load/Store            | `FLW`, `FLD`, `FSW`, `FSD`                       |
| Arithmetic            | `FADD`, `FSUB`, `FMUL`, `FDIV`, `FSQRT`         |
| Fused Multiply-Add    | `FMADD`, `FMSUB`, `FNMADD`, `FNMSUB`            |
| Comparison            | `FEQ`, `FLT`, `FLE`                              |
| Classification        | `FCLASS`                                         |
| Conversion            | `FCVT.W.S`, `FCVT.S.W`, `FCVT.D.S`, etc.        |
| Sign Injection        | `FSGNJ`, `FSGNJN`, `FSGNJX`                     |
| Move (FP ↔ Int)       | `FMV.X.W`, `FMV.W.X` (F); `FMV.X.D`, `FMV.D.X` (D, RV64 only) |
| Min/Max               | `FMIN`, `FMAX`                                   |

### Rounding Modes

All arithmetic instructions include a 3-bit `rm` field that selects the IEEE 754 rounding mode for that instruction. The special encoding `111` (DYN) means "use the current dynamic rounding mode from `fcsr.frm`".

| `rm` | Name | Abbreviation | Description |
|------|------|--------------|-------------|
| 000  | Round to Nearest, ties to Even | RNE | Default IEEE 754 mode |
| 001  | Round Toward Zero              | RTZ | Truncation (C cast behaviour) |
| 010  | Round Down (toward -inf)       | RDN | Floor |
| 011  | Round Up (toward +inf)         | RUP | Ceil |
| 100  | Round to Nearest, ties to Max Magnitude | RMM | Rounds halfway away from zero |
| 101  | Reserved | — | Illegal |
| 110  | Reserved | — | Illegal |
| 111  | Dynamic (use `fcsr.frm`)       | DYN | Per-instruction override |

Using an illegal rounding mode (`rm = 101` or `110`) raises an illegal instruction exception.

### Floating-Point Control and Status Register (FCSR)

`fcsr` is a 32-bit CSR (address `0x003`) that contains two sub-fields:

```
 31                    8  7   5  4  3  2  1  0
 +----------------------+-------+-------------+
 |       reserved       |  frm  |   fflags    |
 +----------------------+-------+-------------+
                          [7:5]   [4:0]
```

**`fflags` (bits 4:0) — Accrued Exception Flags:**

| Bit | Flag | Meaning                               |
|-----|------|---------------------------------------|
| 4   | NV   | Invalid Operation (e.g., 0/0, inf-inf)|
| 3   | DZ   | Divide by Zero                        |
| 2   | OF   | Overflow (result too large)           |
| 1   | UF   | Underflow (result too small)          |
| 0   | NX   | Inexact (result was rounded)          |

These flags are **sticky** — they are set when an exception occurs and remain set until explicitly cleared by software. They accumulate across multiple instructions. Use `CSRC fflags, t0` to clear specific bits.

**`frm` (bits 7:5) — Dynamic Rounding Mode:**

Holds the default rounding mode used when an instruction's `rm` field is `DYN (111)`. Can be set with `CSRW frm, t0`.

The sub-registers `fflags` (CSR `0x001`) and `frm` (CSR `0x002`) are independently accessible aliases into `fcsr`.

### NaN Boxing

When F is implemented alongside D (or Zfh for half-precision), a value stored in a narrower format occupies the low bits of a wider register. The **NaN boxing** rule states: the upper bits must all be 1. If an `FLW` loads a 32-bit value into an `f` register in a D-capable system, the upper 32 bits of the 64-bit register are filled with all-ones (`0xFFFFFFFF`), making the 64-bit pattern a quiet NaN when interpreted as a double.

The rule works in reverse: any instruction expecting a single-precision operand from an `f` register checks whether the upper 32 bits are all-ones. If they are not, the operand is treated as the canonical NaN (`0x7FC00000`) regardless of the lower bits. This prevents accidental reinterpretation of a double-precision value as a valid single-precision value.

**Practical impact:**
- An `FLD` followed by a `FMUL.S` on the same register will treat the loaded double as a NaN — you cannot load a double into a float register and then use it as a float.
- A freshly loaded `FLW` value has NaN-boxed upper bits (all-ones) set by the load itself. The first subsequent F-extension operation sees a valid 32-bit float.

### IEEE 754 Special Values

| Value        | Float (binary32)         | Notes |
|--------------|--------------------------|-------|
| +0           | `0 00000000 00...0`      | Positive zero |
| -0           | `1 00000000 00...0`      | Negative zero; `+0 == -0` is true |
| +Inf         | `0 11111111 00...0`      | |
| -Inf         | `1 11111111 00...0`      | |
| sNaN         | `x 11111111 0 nonzero`   | Signalling NaN; raises NV on use |
| qNaN         | `x 11111111 1 xxx`       | Quiet NaN; propagates silently |
| Subnormal    | `x 00000000 nonzero`     | Gradual underflow |

In RISC-V, the canonical quiet NaN is `0x7FC00000` (float) and `0x7FF8000000000000` (double). Operations that produce a NaN result return this canonical value.

**RISC-V deviation from IEEE 754:** `FMIN` and `FMAX` with a quiet NaN input return the other operand (not NaN). This is the IEEE 754-2008 `minNum`/`maxNum` behaviour, not the IEEE 754-2019 `minimum`/`maximum` behaviour (which propagates NaN). Signalling NaN inputs always raise the NV flag.

### Fused Multiply-Add

The four FMA instructions compute a fused operation in a single rounding step:

| Instruction | Operation        |
|-------------|------------------|
| `FMADD`     | `+(rs1 * rs2) + rs3` |
| `FMSUB`     | `+(rs1 * rs2) - rs3` |
| `FNMSUB`    | `-(rs1 * rs2) + rs3` |
| `FNMADD`    | `-(rs1 * rs2) - rs3` |

Using R4-type encoding (four register operands), these compute the full product internally at double precision before the single addition and round once at the end — more accurate than a separate `FMUL` followed by `FADD`.

### The `FCLASS` Instruction

`FCLASS.S rd, rs1` and `FCLASS.D rd, rs1` write a 10-bit bitmask to the integer register `rd` identifying the class of the floating-point value in `rs1`:

| Bit | Meaning               |
|-----|-----------------------|
| 0   | rs1 is -infinity      |
| 1   | rs1 is negative normal |
| 2   | rs1 is negative subnormal |
| 3   | rs1 is -0             |
| 4   | rs1 is +0             |
| 5   | rs1 is positive subnormal |
| 6   | rs1 is positive normal |
| 7   | rs1 is +infinity      |
| 8   | rs1 is a signalling NaN |
| 9   | rs1 is a quiet NaN    |

Exactly one bit is set. This replaces a sequence of comparisons that would otherwise require multiple instructions and careful NaN handling.

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What is the difference between the F and D extensions? Can you implement D without F?**

**Answer:**

The F extension adds single-precision (32-bit, IEEE 754 binary32) floating-point support with 32 registers `f0`–`f31`, each 32 bits wide. The D extension extends these registers to 64 bits and adds double-precision (IEEE 754 binary64) instructions.

D requires F — you cannot implement D without first implementing F. The D extension is defined as a superset of F: a system with D has F by definition. Registers widen from 32 to 64 bits when D is added, and the NaN boxing mechanism manages the relationship between single and double precision values in the same register file.

The rationale: sharing one register file avoids the cost of two separate register files and simplifies calling conventions (both F and D parameters use the same `fa0`–`fa7` argument registers, distinguished only by whether a 32-bit or 64-bit load/store is used).

---

**Q2. What are the five IEEE 754 exception flags in `fflags`, and what condition raises each one?**

**Answer:**

| Flag | Condition |
|------|-----------|
| NV (Invalid Operation) | Operations with undefined results: `0.0/0.0`, `inf - inf`, `sqrt(-1.0)`, unordered comparison involving NaN |
| DZ (Divide by Zero)    | A finite non-zero dividend divided by zero: `1.0 / 0.0` |
| OF (Overflow)          | The rounded result exceeds the representable range and rounds to infinity |
| UF (Underflow)         | The result is a non-zero value smaller than the smallest normal number (subnormal or rounds to zero) |
| NX (Inexact)           | The rounded result differs from the exact mathematical result |

The flags are sticky and must be cleared explicitly. A typical pattern for detecting FP errors:

```asm
csrwi  fflags, 0        # clear all flags
fadd.s fa0, fa1, fa2    # operation of interest
csrr   t0, fflags       # read accumulated flags
andi   t0, t0, 0x10     # test NV (bit 4)
bnez   t0, .fp_error
```

---

**Q3. What rounding mode does `FCVT.W.S` (convert float to signed 32-bit integer) use by default, and what result is returned if the floating-point value is 2.7?**

**Answer:**

`FCVT.W.S rd, rs1, rm` uses the `rm` field in the instruction encoding. The default when writing the mnemonic without an explicit rounding mode in assembly is `RNE` (round to nearest, ties to even), but the programmer should specify explicitly. For conversion to integer, the most common need is truncation, which uses `RTZ` (round toward zero) — this matches C cast semantics `(int)f`.

With `RTZ`: `(int)2.7 = 2`. The fractional part is discarded toward zero.
With `RNE`: 2.7 rounds to 3 (nearest integer).

The NX (inexact) flag is always raised when the conversion is not exact (i.e., when the fractional part is non-zero), since rounding occurred.

Special cases:
- Infinity or NaN input → result = `INT_MAX` or `INT_MIN` (implementation-defined maximum), NV flag set.
- Value out of `int` range → clamped to `INT_MAX`/`INT_MIN`, NV flag set.

---

**Q4. Explain NaN boxing. What happens when `FLW fa0, 0(a0)` is executed on a system with both F and D extensions?**

**Answer:**

On a D-capable system, `f` registers are 64 bits wide. `FLW` loads a 32-bit value from memory and places it in the lower 32 bits of the destination register. The upper 32 bits are filled with all-ones (`0xFFFFFFFF`). This is NaN boxing.

The resulting 64-bit pattern has the upper 32 bits all set to 1. When this 64-bit value is interpreted as a double-precision float, the exponent field (bits 62:52) is all-ones and the mantissa is non-zero (the lower 32 bits), making it a quiet NaN. This is intentional: the register now holds a "NaN-boxed" 32-bit float.

When an F-extension instruction subsequently reads this register:
- It checks that the upper 32 bits are all-ones.
- If yes: the lower 32 bits are a valid single-precision value and are used normally.
- If no: the operand is treated as the canonical quiet NaN (`0x7FC00000`), regardless of the actual value.

This mechanism prevents accidental use of double-precision values as single-precision inputs, which would produce nonsensical results.

---

**Q5. What is the canonical quiet NaN in RISC-V for single precision, and why does RISC-V define one?**

**Answer:**

The canonical quiet NaN for single precision is `0x7FC00000`:
- Sign bit = 0
- Exponent = all-ones (`0xFF`)
- Mantissa = `0x400000` (bit 22 set, which is the "quiet" bit in IEEE 754)

RISC-V defines a canonical NaN because IEEE 754 allows any value with a non-zero significand and all-one exponent to be a quiet NaN — there are millions of possible quiet NaN bit patterns. When operations produce a NaN result (e.g., `0.0 / 0.0`), RISC-V always returns this single canonical value rather than attempting to propagate specific NaN payloads.

Benefits:
1. Hardware simplicity: no need to route payload bits through the FPU result path.
2. Determinism: NaN results are reproducible regardless of operand order.
3. Software predictability: a single pattern to test for.

The trade-off is that NaN payload propagation (used in some languages to tag NaN values with debug information) is not supported in hardware.

---

### Intermediate Tier

---

**Q6. A programmer writes the following C code. What assembly does a RISC-V compiler emit for the multiplication, and why is it more accurate than two separate instructions?**

```c
float fma_result = a * b + c;  // compiled with -ffast-math disabled
```

**Answer:**

Without `-ffast-math`, the compiler must respect IEEE 754 semantics, which requires that `a * b + c` is computed as two separate rounded operations: first `FMUL.S` (rounding to nearest), then `FADD.S` (rounding to nearest again). This introduces two rounding errors.

However, C99 `<math.h>` provides `fmaf(a, b, c)` which maps to `FMADD.S`:

```asm
# fmaf(a, b, c): a in fa0, b in fa1, c in fa2
fmadd.s  fa0, fa0, fa1, fa2    # fa0 = (fa0 * fa1) + fa2, one rounding
```

`FMADD.S` is more accurate because:
1. The product `a * b` is computed at full internal precision (at least 2×24 = 48 mantissa bits for binary32).
2. The addition of `c` is performed on the full-precision intermediate result.
3. A single rounding step is applied at the end.

The difference matters in numerical algorithms: e.g., dot product computation, polynomial evaluation (Horner's method), and compensated summation all benefit from FMA. The error bound drops from `2u` (two separate operations, each with error `u = 2^{-24}`) to `u` (one FMA).

For the plain `a * b + c` expression without explicit `fmaf()`, whether the compiler uses `FMADD` is implementation-defined — GCC requires `-ffp-contract=fast` or the `FP_CONTRACT` pragma to fuse automatically.

---

**Q7. Describe the sign-injection instructions `FSGNJ`, `FSGNJN`, and `FSGNJX`. What common operations can they implement without arithmetic?**

**Answer:**

The sign-injection instructions copy all bits of `rs1` (magnitude and exponent) to `rd` but replace the sign bit with a value derived from `rs2`:

| Instruction | Sign bit of result | Operation |
|-------------|-------------------|-----------|
| `FSGNJ.S  rd, rs1, rs2` | `rs2` sign bit as-is | Copy sign from rs2 |
| `FSGNJN.S rd, rs1, rs2` | Inverted `rs2` sign bit | Negate sign of rs2 |
| `FSGNJX.S rd, rs1, rs2` | XOR of `rs1` and `rs2` sign bits | Toggle sign based on rs2 |

Common operations they implement:

```asm
# Absolute value: |a| -- FSGNJ rd, rs1, rs1 copies the sign from rs1 to rd (same register),
# which is a NOP (not useful for abs). The correct idiom uses FSGNJX:
fsgnj.s  fa0, fa0, fa0   # Not useful; this is a NOP (sign from itself)

# Actually the idiomatic absolute value:
fsgnjx.s fa0, fa0, fa0   # XOR sign with itself = always 0 = |fa0|

# Negate: -a = FSGNJN rd, rs1, rs1
fsgnjn.s fa0, fa0, fa0   # negate: flip sign bit of fa0

# Copy sign from another value (copysign):
fsgnj.s  fa0, fa0, fa1   # fa0 = |fa0| with sign of fa1
```

These are preferred over arithmetic negation because:
- No rounding is possible (pure bit manipulation).
- They work correctly on NaN and infinity without raising exceptions.
- They are typically one cycle with no FPU pipeline needed — many implementations route them through the integer pipeline.

---

**Q8. What does `FCLASS.S` return for the value `0xFF800000`? Show your working.**

**Answer:**

`0xFF800000` in binary:

```
Sign: 1
Exponent: 11111111 (all ones = 255)
Mantissa: 00000000 00000000 0000000 (all zeros)
```

All-one exponent with all-zero mantissa = infinity. Sign bit = 1 → negative infinity.

`FCLASS.S` returns a 10-bit mask with bit 0 set for negative infinity:

```
Result = 0b00_0000_0001 = 0x001
```

Bit 0 of `rd` is set; all other bits are zero.

In code:
```asm
li      a0, 0xFF800000
fmv.w.x fa0, a0          # move bit pattern into fp register
fclass.s a1, fa0          # classify
# a1 = 1 (bit 0 set = negative infinity)
```

---

**Q9. A function performs floating-point operations and needs to detect whether any exception occurred. It must save and restore the FP state around a library call. Write the RISC-V assembly to do this correctly.**

**Answer:**

```asm
# Save fp state before library call
csrr    t0, fcsr          # read entire fcsr (frm + fflags)
sw      t0, -4(sp)        # save to stack

# Clear accumulated flags before our operations
csrwi   fflags, 0

# ... perform FP operations ...
fadd.s  fa0, fa1, fa2
fdiv.s  fa0, fa0, fa3

# Read flags for our operations
csrr    t1, fflags        # t1 = exception flags from our operations
sw      t1, -8(sp)        # save our flags separately

# Restore original fcsr before calling library
lw      t0, -4(sp)
csrw    fcsr, t0           # restores both frm and fflags to pre-call state

# Call library function (may modify fcsr)
call    some_library_func

# Restore our fcsr state if needed afterward
# ... application logic using t1 (our saved flags) ...
lw      t1, -8(sp)
andi    t1, t1, 0x10      # test NV flag (bit 4)
bnez    t1, .handle_invalid_op
```

Key points:
1. Save and restore the entire `fcsr` (both `frm` and `fflags`) around library calls whose FP behaviour is unknown.
2. Clear `fflags` before your own operations to get a clean baseline.
3. Read `fflags` after your operations before the restore to capture your exceptions.
4. Use `csrwi fflags, 0` (not `csrw fcsr, 0`) to avoid overwriting the rounding mode.

---

**Q10. Why are there no floating-point conditional branch instructions in RISC-V? How does the compiler implement `if (a > b)` for floats?**

**Answer:**

RISC-V deliberately has no FP conditional branches. All floating-point comparisons write a result (0 or 1) into an integer register, and then a standard integer branch instruction tests that result. This keeps the branch prediction logic entirely in the integer pipeline.

Implementation of `if (a > b)` for floats:

```asm
# a in fa0, b in fa1
flt.s   t0, fa1, fa0      # t0 = (fa1 < fa0) ? 1 : 0  (i.e., a > b)
# Note: FLT tests "strictly less than"; to get a > b, use FLT(b, a)
beqz    t0, .else_branch  # if t0 == 0, condition was false

# then branch
.else_branch:
```

Alternatively using `FLE`:
```asm
fle.s   t0, fa0, fa1      # t0 = (fa0 <= fa1) ? 1 : 0
bnez    t0, .else_branch  # a <= b means NOT a > b
```

The three comparison instructions:
- `FEQ.S rd, rs1, rs2`: `rd = (rs1 == rs2)` — quiet NaN input returns 0, no NV flag.
- `FLT.S rd, rs1, rs2`: `rd = (rs1 < rs2)` — NaN input sets NV flag and returns 0.
- `FLE.S rd, rs1, rs2`: `rd = (rs1 <= rs2)` — NaN input sets NV flag and returns 0.

The NaN handling difference between `FEQ` and `FLT`/`FLE` is significant: IEEE 754 specifies that equality comparison with NaN is quiet (does not signal), while ordered comparisons (`<`, `<=`) with NaN signal the invalid operation exception.

---

### Advanced Tier

---

**Q11. Explain the difference between FTZ (flush to zero) and DAZ (denormals are zero) modes. Does RISC-V support either in hardware?**

**Answer:**

**Flush to Zero (FTZ):** When the result of an FP operation would be a subnormal, it is instead flushed to zero (with the sign preserved). Subnormal results are never written; they become zero. The UF and NX flags may still be raised.

**Denormals Are Zero (DAZ):** Subnormal *inputs* are treated as zero before the operation begins. This complements FTZ by also flushing subnormal inputs.

x86 SSE supports both FTZ and DAZ via the MXCSR register. They are common in DSP and game engine code where the gradual underflow behaviour of IEEE 754 is not needed but the performance cost of subnormal handling (often 10-100x slower on microcode-assisted implementations) is unacceptable.

**RISC-V does not define FTZ or DAZ modes in the F/D extensions.** The base F/D extensions are fully IEEE 754-2008 compliant, including correct gradual underflow via subnormals.

Implementations that want non-IEEE behaviour must either:
1. Use the Zfinx extension (FP in integer registers) with custom microarchitectural modes.
2. Define custom CSR fields (vendor-specific, non-standard).
3. Rely on the compiler inserting explicit zero-testing for subnormals.

The absence of FTZ/DAZ is a deliberate standards-compliance decision. High-performance RISC-V implementations (e.g., the SiFive P670) handle subnormals in hardware without a microcode penalty, making FTZ less necessary.

---

**Q12. A RISC-V FPU produces the result `0x3F800001` for a single-precision addition. Is this a valid IEEE 754 result? What rounding mode could produce it?**

**Answer:**

`0x3F800001` in binary:

```
Sign:     0
Exponent: 0111 1111  = 127 (biased) = 0 (actual exponent)
Mantissa: 000 0000 0000 0000 0000 0001
```

Value: `1.000...001 × 2^0 = 1 + 2^{-23}`

This is the floating-point number immediately above 1.0. It is a valid IEEE 754 binary32 value (a normal number).

For this to be the result of an addition like `1.0 + x` where `x` is very small:

The ULP (unit in the last place) at 1.0 is `2^{-23}`. If `x = 2^{-24}` (exactly halfway between 1.0 and `1 + 2^{-23}`):
- `RNE` → rounds to `1.0` (ties to even: last bit of 1.0 is 0, which is even)
- `RUP` → rounds up to `1 + 2^{-23}` = `0x3F800001`
- `RTZ` → rounds to `1.0` (toward zero means toward smaller magnitude)
- `RDN` → rounds to `1.0` (toward negative infinity = smaller value for positive result)

So `0x3F800001` as the result of `1.0 + 2^{-24}` is produced by rounding mode `RUP` (round toward +infinity).

It could also be the exact result of `1.0 + 2^{-23}` (no rounding needed), in which case any rounding mode produces it and the NX flag is not set.

---

**Q13. The `Zfinx` extension stores floating-point values in the integer register file instead of dedicated FP registers. What are the trade-offs, and when would you choose it over the standard F extension?**

**Answer:**

`Zfinx` ("float in integer register x") is a RISC-V extension that implements FP operations but maps the operands to the integer register file `x0`–`x31`. There are no separate `f` registers.

**Advantages of Zfinx:**
- **Area savings:** eliminates 32 dedicated FP registers (128 bytes for F, 256 bytes for D). For microcontrollers, this can be 3-5% of total register file area.
- **No FP context switch cost:** integer context saves/restores also save FP state automatically, since it is the same file. Significant for real-time systems with many interrupt contexts.
- **Simpler ABI:** one register file means one save/restore convention.
- **No NaN boxing complexity:** integer and FP values coexist directly.

**Disadvantages of Zfinx:**
- **Register pressure:** FP and integer operations now compete for the same 32 registers. The compiler's register allocator must balance both.
- **ABI incompatibility:** Zfinx code is ABI-incompatible with standard F/D code. Libraries must be recompiled.
- **No hardware FMA optimisation path for mixed FP+integer:** instruction scheduling is constrained by the single-port register file.
- **Not suitable for high-performance FP:** the shared register file creates structural hazards in superscalar designs.

**When to choose Zfinx:**
- Deeply embedded microcontrollers (e.g., IoT sensor nodes) where die area is the primary constraint.
- Real-time systems with tight interrupt latency budgets where FP context save is a concern.
- Systems running custom RTOS where the entire software stack can be recompiled from source.
- Applications that rarely use FP (occasional sensor calibration, not inner-loop computation).

Choose standard F/D extension for any application with significant FP workloads, Linux-based systems (which use the standard ABI), or when third-party binary libraries are required.

---

**Q14. Describe the rounding behaviour of `FCVT.W.S` when converting `+infinity`, `-infinity`, and `NaN` to a 32-bit signed integer.**

**Answer:**

`FCVT.W.S rd, rs1, rm` converts a single-precision float in `rs1` to a signed 32-bit integer in `rd` using rounding mode `rm`.

For values not representable as a signed 32-bit integer, RISC-V defines:

| Input        | Result (`rd`) | Flag set |
|--------------|---------------|----------|
| `+Infinity`  | `INT_MAX` = `0x7FFFFFFF` = 2^31 − 1 | NV |
| `-Infinity`  | `INT_MIN` = `0x80000000` = −2^31    | NV |
| Any NaN      | `INT_MAX` = `0x7FFFFFFF`            | NV |
| Float > INT_MAX (e.g., 3e9) | `INT_MAX` | NV |
| Float < INT_MIN (e.g., -3e9) | `INT_MIN` | NV |

The NV (invalid operation) flag is always raised for these cases. The rounding mode `rm` has no effect because no representable integer result exists — the operation is inherently invalid.

In C, `(int)(1.0f / 0.0f)` is undefined behaviour. On RISC-V, the hardware produces `INT_MAX` and raises NV, but the C standard does not guarantee this result. Safety-critical code should test for infinity/NaN before conversion:

```c
#include <math.h>
int safe_float_to_int(float f) {
    if (!isfinite(f) || f >= (float)INT_MAX || f <= (float)INT_MIN)
        return handle_error();
    return (int)f;
}
```

---

**Q15. A compiler is performing instruction scheduling for a floating-point loop body. The loop contains a dependent chain: `FMUL.S` → `FADD.S` → `FSUB.S`. The FPU has 4-cycle multiply latency and 2-cycle add/sub latency. What is the minimum cycle count for one iteration, and how should the compiler unroll to achieve peak throughput?**

**Answer:**

**Latency-bound analysis for one iteration:**

```
Cycle  1:  FMUL.S issued
Cycle  5:  FMUL result ready  (4-cycle latency)
Cycle  5:  FADD.S issued (dependent on FMUL)
Cycle  7:  FADD result ready  (2-cycle latency)
Cycle  7:  FSUB.S issued (dependent on FADD)
Cycle  9:  FSUB result ready  (2-cycle latency)
```

Minimum cycles per iteration (latency bound): **8 cycles** (from issue of FMUL to availability of FSUB result; plus 1 for the FSUB issue cycle = the chain is 4+2+2 = 8 cycles of latency). With a throughput of 1 instruction per cycle, the throughput bound is 3 instructions per cycle — but the dependency chain means the loop cannot issue a new FMUL until 8 cycles after the previous FMUL.

**Unrolling strategy:**

If the FPU has 1 FMUL/cycle throughput and 1 FADD/cycle throughput, unrolling by 8 allows 8 independent chains to be interleaved:

```asm
# Unrolled x8: 8 independent accumulator chains (acc0...acc7)
fmul.s  ft0, fa0, fa8    # chain 0
fmul.s  ft1, fa1, fa9    # chain 1
fmul.s  ft2, fa2, fa10   # chain 2
fmul.s  ft3, fa3, fa11   # chain 3
fmul.s  ft4, fa4, fa12   # chain 4
fmul.s  ft5, fa5, fa13   # chain 5
fmul.s  ft6, fa6, fa14   # chain 6
fmul.s  ft7, fa7, fa15   # chain 7
# By the time ft0 is needed for fadd, 4 cycles have passed (ft1..ft3 filled the gap + 1 more)
fadd.s  ft0, ft0, fs0    # chain 0 FADD -- ft0 ready at cycle 5 (issued cycle 1+4=5)
...
```

With 8-way unrolling, the FMUL throughput is fully utilised (one new FMUL per cycle), and the 4-cycle latency is hidden by the 7 independent MULs issued before the first ADD. Theoretical throughput with full unrolling: 3 instructions per 8-cycle iteration = 0.375 instructions/cycle from the perspective of original loop iterations, but 3 FP operations completing per cycle continuously in the unrolled steady state.
