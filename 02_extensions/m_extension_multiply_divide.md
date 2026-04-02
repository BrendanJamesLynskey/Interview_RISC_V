# M Extension: Multiply and Divide

## Overview

The RISC-V M extension adds eight integer multiply/divide instructions to the base ISA. All M-extension instructions use the R-type encoding and operate on the same integer register file as RV32I/RV64I. Understanding MUL vs MULH, the exact semantics of division by zero, and the signed/unsigned variants is a recurring interview topic because these details expose whether a candidate has actually worked with the hardware rather than just read the name.

---

## Key Concepts

### Instruction Summary

| Mnemonic   | Operation                                    | Result Width  |
|------------|----------------------------------------------|---------------|
| `MUL`      | `rs1 * rs2` (signed x signed), lower half    | XLEN bits     |
| `MULH`     | `rs1 * rs2` (signed x signed), upper half    | XLEN bits     |
| `MULHSU`   | `rs1 * rs2` (signed x unsigned), upper half  | XLEN bits     |
| `MULHU`    | `rs1 * rs2` (unsigned x unsigned), upper half| XLEN bits     |
| `DIV`      | `rs1 / rs2` (signed)                         | XLEN bits     |
| `DIVU`     | `rs1 / rs2` (unsigned)                       | XLEN bits     |
| `REM`      | `rs1 % rs2` (signed)                         | XLEN bits     |
| `REMU`     | `rs1 % rs2` (unsigned)                       | XLEN bits     |

For RV64M, word-width variants (`MULW`, `DIVW`, `DIVUW`, `REMW`, `REMUW`) operate on the lower 32 bits and sign-extend the result to 64 bits.

### MUL and the MULH Family

A 32x32 multiply produces a 64-bit result. The base `MUL` instruction returns the lower 32 bits — sufficient when overflow does not matter or when the programmer has verified it cannot occur. To obtain the full 64-bit product you pair `MUL` with one of the upper-half variants:

```
full_product[63:0] = MULH(rs1, rs2) : MUL(rs1, rs2)
```

The three upper-half variants exist because the sign-extension treatment of each operand changes the result:

- `MULH`   — both operands treated as signed (two's complement sign-extended to 2*XLEN bits before multiply).
- `MULHU`  — both operands treated as unsigned (zero-extended).
- `MULHSU` — `rs1` signed, `rs2` unsigned. This covers the common case of multiplying a signed number by an unsigned magnitude (e.g., scaling a signed sensor reading).

**Fused MULH/MUL hint:** The spec permits implementations to detect a consecutive `MULH[SU|U] rdh, rs1, rs2` / `MUL rdl, rs1, rs2` pair (same source registers, different destinations) and fuse them into a single 2*XLEN multiply. Implementations that do not fuse must still produce correct results.

### Division and Remainder Semantics

#### Division by Zero

Rather than raising an exception, RISC-V defines specific deterministic results for division by zero. This was a conscious design decision to simplify hardware (no mandatory divide-by-zero trap) while still giving software a predictable value to check.

| Operation    | Dividend | Divisor | Result (defined) |
|--------------|----------|---------|------------------|
| `DIV`        | any      | 0       | -1 (all ones)    |
| `DIVU`       | any      | 0       | 2^XLEN - 1 (all ones) |
| `REM`        | any      | 0       | dividend (unchanged) |
| `REMU`       | any      | 0       | dividend (unchanged) |

The pattern is: unsigned division by zero returns the maximum unsigned value; signed division by zero returns -1 (0xFFFF...F); remainder by zero returns the original dividend.

Software that needs to detect division by zero must check the divisor explicitly before issuing the instruction — there is no trap to catch it.

#### Signed Overflow: The Minimum Integer Case

Signed integer division has one additional edge case: `INT_MIN / -1` overflows (the mathematical result, +2^(XLEN-1), is not representable). RISC-V defines:

| Operation | Dividend   | Divisor | Result    |
|-----------|------------|---------|-----------|
| `DIV`     | INT_MIN    | -1      | INT_MIN   |
| `REM`     | INT_MIN    | -1      | 0         |

This mirrors the x86 behaviour with `IDIV` (except x86 actually raises a #DE fault in this case, while RISC-V silently saturates). Compilers must emit an explicit check before `DIV` to avoid producing wrong results.

#### Truncation Direction

RISC-V division truncates toward zero (same as C99 `/` and `%`):
- `7 / 2 = 3`, `7 % 2 = 1`
- `-7 / 2 = -3`, `-7 % 2 = -1`
- `7 / -2 = -3`, `7 % -2 = 1`

The identity `(a/b)*b + (a%b) == a` always holds for non-zero divisors.

### Encoding

All M-extension instructions are R-type with `opcode = 0110011` (OP) for RV32M, `funct7 = 0000001`, and `funct3` selecting the specific operation:

| funct3 | Instruction |
|--------|-------------|
| 000    | MUL         |
| 001    | MULH        |
| 010    | MULHSU      |
| 011    | MULHU       |
| 100    | DIV         |
| 101    | DIVU        |
| 110    | REM         |
| 111    | REMU        |

### Hardware Implementation Notes

- **Multiply latency** on real implementations (e.g., SiFive U74, Rocket): typically 1-3 cycles for lower 32-bit result, more for full 64-bit.
- **Divide latency**: typically 8-35 cycles depending on implementation (iterative restoring vs non-restoring dividers).
- The spec deliberately does not mandate pipelining or latency, allowing ultra-low area implementations to use multi-cycle state machines.

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What are the eight instructions added by the M extension, and why are there three variants of the upper-half multiply?**

**Answer:**

The M extension adds: `MUL`, `MULH`, `MULHSU`, `MULHU`, `DIV`, `DIVU`, `REM`, `REMU`.

Three upper-half variants are needed because a 2*XLEN multiply result differs depending on how each operand is sign-extended before the multiplication:

- `MULH`: `rs1` and `rs2` are both sign-extended (signed × signed).
- `MULHU`: both are zero-extended (unsigned × unsigned).
- `MULHSU`: `rs1` is sign-extended, `rs2` is zero-extended (signed × unsigned).

`MUL` alone gives the lower XLEN bits, which are the same regardless of sign treatment. The upper half differs, which is why three separate instructions are needed to cover the different combinations.

---

**Q2. What does RISC-V specify as the result of `DIV x1, x2, x0` (i.e., dividing by zero)?**

**Answer:**

RISC-V does not trap on division by zero. The result is architecturally defined:

- `DIV` with divisor = 0 → result = -1 (all bits set, 0xFFFFFFFF on RV32).
- `DIVU` with divisor = 0 → result = 2^XLEN − 1 (maximum unsigned value).
- `REM` / `REMU` with divisor = 0 → result = dividend (unchanged).

Software is responsible for checking the divisor before dividing if a zero divisor needs to be detected. The motivation for this design is hardware simplicity: no exception handler, no trap infrastructure required for the divide unit.

---

**Q3. Which instruction would you use to compute the upper 32 bits of the product of two unsigned 32-bit values on RV32?**

**Answer:**

`MULHU rd, rs1, rs2`.

`MULHU` zero-extends both `rs1` and `rs2` to 64 bits before multiplying and places the upper 32 bits in `rd`. This is the correct instruction when both operands are unsigned. Using `MULH` (signed) on unsigned values would produce an incorrect upper half if either operand has bit 31 set, because the sign extension would change the mathematical value of the operand.

---

**Q4. What is the result of `DIV x1, x1, x1` where x1 = `0x80000000` (INT_MIN) on RV32? What about `REM x1, x1, x1`?**

**Answer:**

The divisor here is `x1 = 0x80000000 = INT_MIN`, not `-1`. Dividing INT_MIN by INT_MIN:

- Mathematical result: 1 (INT_MIN / INT_MIN = 1).
- 1 is representable as a signed 32-bit integer.
- `DIV` result = 1.
- `REM` result = 0 (INT_MIN - 1 * INT_MIN = 0).

The overflow case only arises for `DIV INT_MIN, -1`. For any other divisor, the result is representable.

---

**Q5. How is a 64-bit multiply result obtained on RV32M?**

**Answer:**

Use a `MUL`/`MULH` pair (choosing the appropriate MULH variant based on signedness):

```asm
# Unsigned 64-bit product of a0 and a1, result in (a1:a0) [a1=high, a0=low]
mulhu  a1, a0, a1   # upper 32 bits: a1 = upper(a0 * a1) unsigned
mul    a0, a0, a1   # lower 32 bits: a0 = lower(a0 * a1)
```

Note: care is needed with register overlap — the source registers must be read before any destination is written. Many assemblers will warn if the destination of the first instruction overlaps a source used by the second. The canonical safe approach uses a temporary register:

```asm
# Unsigned: rs1=a0, rs2=a1 -> result high in t0, low in a0
mulhu  t0, a0, a1   # t0 = upper 32 bits
mul    a0, a0, a1   # a0 = lower 32 bits
mv     a1, t0       # a1 = upper 32 bits
```

---

### Intermediate Tier

---

**Q6. A compiler encounters `int c = a / b` where `b` is a variable. What code must it emit to ensure correctness on RISC-V, considering the division by zero and INT_MIN/-1 cases?**

**Answer:**

```asm
# Assume a in a0, b in a1, result to a0
# Step 1: Check for divisor = 0
beqz  a1, .div_by_zero

# Step 2: Check for INT_MIN / -1 overflow
li    t0, -1
bne   a1, t0, .safe_div     # if b != -1, safe
li    t0, 0x80000000         # INT_MIN
beq   a0, t0, .overflow      # if a == INT_MIN and b == -1, overflow

.safe_div:
div   a0, a0, a1
j     .done

.div_by_zero:
li    a0, -1                 # spec-defined result
j     .done

.overflow:
li    a0, 0x80000000         # INT_MIN (spec-defined result for this case)

.done:
```

In practice, most compilers emit the divisor-zero check only when required by language semantics (C undefined behaviour allows the compiler to assume `b != 0` if the programmer has not added a check). The INT_MIN/-1 check is likewise often omitted in optimised builds because hitting it is UB in C. However, safety-critical code (e.g., MISRA C) must emit these guards.

---

**Q7. What is the `MULHSU` instruction for? Give a concrete use case.**

**Answer:**

`MULHSU rd, rs1, rs2` computes the upper XLEN bits of the product `rs1 * rs2` where `rs1` is treated as a signed integer and `rs2` as an unsigned integer.

**Use case: multi-precision arithmetic with mixed signs.**

When implementing 64-bit signed arithmetic on RV32I (before having RV64), a 64-bit number is stored as a signed high word and an unsigned low word. Multiplying two such numbers requires four 32×32 partial products:

```
(a_hi:a_lo) * (b_hi:b_lo)

= a_hi * b_hi * 2^64       (signed × signed  -> MULH)
+ a_hi * b_lo * 2^32       (signed × unsigned -> MULHSU)
+ a_lo * b_hi * 2^32       (unsigned × signed -> MULHSU with operands swapped)
+ a_lo * b_lo              (unsigned × unsigned -> MULHU / MUL)
```

Without `MULHSU`, the mixed-sign cross-product terms would require emulation via multiple instructions. `MULHSU` makes this one instruction.

---

**Q8. Describe the difference in the truncation behaviour between C99 integer division and floor division. Why does RISC-V choose truncation toward zero?**

**Answer:**

- **Truncation toward zero (C99, RISC-V `DIV`/`REM`):** the quotient is rounded toward zero. Remainder has the same sign as the dividend.
  - `-7 / 2 = -3`, `-7 % 2 = -1`
- **Floor division (Python `//`, `%`):** the quotient is rounded toward negative infinity. Remainder has the same sign as the divisor.
  - `-7 // 2 = -4`, `-7 % 2 = 1`

RISC-V chose truncation toward zero for two reasons:

1. **C compatibility**: the most important system programming language specifies truncation toward zero for `/` and `%` since C99. Hardware that matches C semantics directly requires no fixup code from the compiler.
2. **Simpler hardware**: truncation toward zero is naturally what binary division hardware produces — it is the raw result of a non-restoring divider. Floor division requires a post-correction step when the operands have different signs.

---

**Q9. On an in-order RISC-V core where `DIV` has 32-cycle latency, how would you schedule code that needs `a = x / y` and independently `b = p + q`?**

**Answer:**

Interleave the independent computation with the long-latency divide to hide its latency:

```asm
# Issue divide early
div   a0, s0, s1        # start x/y — 32 cycles latency

# Independent work fills the pipeline while divide completes
add   a1, s2, s3        # b = p + q   (1 cycle, no dependency on div)
# ... other independent instructions ...

# By now the divide result is ready (if enough instructions filled the gap)
mv    a2, a0            # use the divide result
```

On in-order cores (which cannot reorder instructions themselves), the programmer or compiler must schedule long-latency instructions as early as possible and fill the gap with independent work. If there is insufficient independent work, the core will stall. For a 32-cycle latency, you need ~31 independent instructions to fully hide the latency — often impractical, so a stall of several cycles is typical.

Out-of-order cores issue the divide to the execution unit and continue issuing independent instructions from the reorder buffer, hiding most of the latency automatically.

---

**Q10. Write a RISC-V assembly routine that computes `n % 8` for a non-negative integer `n` using only a single M-extension instruction. Then show how to do it without the M extension at all.**

**Answer:**

With M extension:
```asm
# a0 = n (non-negative), result in a0
li    t0, 8
remu  a0, a0, t0    # unsigned remainder: a0 = a0 % 8
```

Without M extension — since 8 is a power of 2, use a bitmask:
```asm
# a0 = n (non-negative), result in a0
andi  a0, a0, 7     # a0 & (8-1) == a0 % 8  (works because n >= 0 and 8 is power of 2)
```

The `ANDI` version is one instruction and executes in a single cycle with no latency. This illustrates why compilers replace `% (power of two)` with a bitmask for unsigned values. For signed values the substitution is more complex (`% 8` on a negative number requires a sign-fixup when using masking).

---

### Advanced Tier

---

**Q11. Explain how a compiler can replace signed division by a compile-time constant with a multiply-shift sequence. Walk through the specific case of `x / 7` on RV32M.**

**Answer:**

For a compile-time constant divisor `d`, the compiler computes a magic multiplier `M` and shift `s` such that:

```
floor(x / d)  ≈  (x * M) >> (XLEN + s)   for x >= 0
```

For signed division toward zero, the formula adjusts for the sign of x.

**Derivation for d = 7, XLEN = 32:**

We seek M (33 bits) and s (shift) such that `M = ceil(2^(32+s) / 7)`.

Using the standard Hacker's Delight algorithm:
- M = `0x24924925` (the "magic number" for dividing by 7)
- s = 2

The generated code for `int x / 7`:

```asm
# a0 = x (signed)
li     t0, 0x24924925   # magic multiplier
mulh   t1, a0, t0       # upper 32 bits of signed multiply (t1 = high half)
add    t1, t1, a0       # adjustment: add original x to compensate for truncation
srai   t1, t1, 2        # arithmetic right shift by s=2
srli   t2, a0, 31       # extract sign bit of original x
add    a0, t1, t2       # add sign bit to correct truncation direction
```

This sequence (5-6 instructions) is faster than a single `DIV` instruction when `DIV` has high latency (which it typically does — 20-32 cycles). Compilers universally apply this transformation for constant divisors.

The key insight: multiplication is cheap (1-3 cycles), division is expensive (20-35 cycles). Trading one `DIV` for several `MUL`+shifts is almost always a win.

---

**Q12. In a multi-hart RISC-V system, two harts each execute `DIV` with the same dividend and divisor simultaneously. Is there any architectural interaction or race condition possible?**

**Answer:**

No. The M-extension integer instructions (`MUL`, `DIV`, `REM`, etc.) are purely computational — they read source registers and write a destination register, with no memory access and no shared architectural state. Each hart has its own independent register file.

Possible implementation concerns (not architectural races):

1. **Shared divide unit:** In some microarchitectures, multiple harts share a single divide execution unit. A second `DIV` arriving while the first is in progress may be stalled until the unit is free. This is a microarchitectural scheduling problem, not a correctness issue.

2. **Energy/thermal:** Simultaneous long-latency divides from multiple harts can cause a power spike. This is a physical design concern, not a correctness concern.

3. **CSR-affecting instructions:** Floating-point divide (`FDIV`) writes flags to `fflags` in `fcsr` — that CSR is per-hart in RISC-V, so even floating-point divides on separate harts do not race.

The question tests whether a candidate conflates instruction latency with architectural state sharing.

---

**Q13. A candidate claims that `MUL a0, a0, a1` followed immediately by `MULH a0, a0, a1` gives the 64-bit product in `(a0:a0)`. What is wrong with this sequence?**

**Answer:**

Two bugs:

1. **Source register overwritten before second use:** The first instruction writes `a0 = lower(a0_orig * a1)`. The second instruction then reads `a0` (which now holds the lower product, not `a0_orig`) and computes `upper(lower(a0_orig * a1) * a1)` — which is not the upper half of the original product at all.

2. **Same destination for both halves:** Even if the source issue were fixed, both instructions write to `a0`. After both execute, `a0` holds the upper half and the lower half is lost.

The correct sequence, with distinct destinations and sources:

```asm
# a0 = multiplicand, a1 = multiplier
# Result: a2 (high 32 bits), a0 (low 32 bits)
mulh  a2, a0, a1    # upper half first, uses original a0
mul   a0, a0, a1    # lower half, uses original a0 (still unmodified)
```

Writing `MULH` before `MUL` ensures both instructions see the original `a0`. Alternatively, use `t0` as a temporary.

---

**Q14. Why does RISC-V define the result of `REM x, x, x0` (remainder by zero) as the dividend rather than zero or an exception? What does this enable?**

**Answer:**

Defining `REM rd, rs1, x0` (zero divisor) = `rs1` (the dividend) enables branchless divisor-zero safe remainder in scenarios where the programmer wants a no-op when the divisor is zero — the original value passes through unchanged.

More importantly, it is consistent with the mathematical identity:

```
dividend = quotient * divisor + remainder
```

If we accept the `DIV` by zero result of -1, then:
```
remainder = dividend - (-1) * 0 = dividend - 0 = dividend
```

The identity holds! This arithmetic consistency is the primary justification. It also means a combined `DIV`/`REM` pair is consistent without special-casing.

From a hardware perspective, returning the dividend on zero requires no extra hardware — the remainder datapath already computes `dividend - quotient * divisor`; when the quotient is the defined value and divisor is zero, the subtraction yields the dividend.

---

**Q15. A firmware engineer writes the following to check for both division-by-zero and overflow before a signed divide. Is this correct? Can it be improved?**

```c
if (b == 0 || (a == INT_MIN && b == -1)) {
    return handle_error();
}
return a / b;
```

**Answer:**

The logic is correct — it guards the two cases where `DIV` produces a defined-but-possibly-wrong result (or UB in C). However, it has a few practical issues:

**Issue 1: Compiler may elide the check.** Because `a / b` with `b == 0` or `a == INT_MIN && b == -1` is undefined behaviour in C, the compiler is allowed to assume those conditions never occur and may eliminate the guard. In GCC/Clang with `-O2`, this can actually happen via UB-based dead code elimination.

**Correct approach:** use `__builtin_expect` or `volatile`, or rearrange to use `DIVU` / `REMU` after manual sign handling to avoid signed-integer UB:

```c
#include <limits.h>
#include <stdint.h>

int safe_div(int a, int b) {
    if (b == 0) return handle_error();
    /* Cast to unsigned to avoid UB; INT_MIN/-1 will produce UINT_MAX+1
       which wraps to 0 in unsigned — still detectable */
    if ((unsigned int)a == (unsigned int)INT_MIN && b == -1) return handle_error();
    return a / b;
}
```

**Issue 2: `INT_MIN` portability.** Assumes `int` is 32 bits. Use `INT_MIN` from `<limits.h>` (which is correct) rather than the literal `0x80000000`.

**Issue 3: Performance.** Two branches on every division. For hot paths, consider structuring the algorithm to ensure the divisor is never zero by construction rather than adding runtime guards.
