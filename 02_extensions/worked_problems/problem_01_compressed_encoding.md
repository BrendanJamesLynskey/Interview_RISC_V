# Problem 01: Compressed Instruction Encoding and Decoding

## Problem Statement

**Context:** You are a hardware engineer reviewing a RISC-V instruction fetch unit design. Your colleague hands you two 16-bit half-words pulled from an instruction stream and asks you to decode them manually. You must also encode a given 32-bit instruction as its compressed equivalent.

---

### Part A: Decode the Following 16-bit Values

Decode each 16-bit value into its compressed instruction mnemonic and its 32-bit equivalent. Show your bit-field extraction at each step.

1. `0x4505`
2. `0xC24C`
3. `0x8082`

---

### Part B: Encode a 32-bit Instruction as Compressed

Encode the following 32-bit instruction as its 16-bit compressed equivalent, showing all bit fields:

```
ADDI sp, sp, -48
```

---

### Part C: Determine if a Compressed Form Exists

For each instruction below, state whether a compressed form exists in standard RVC. If it does, name it. If it does not, explain which constraint prevents it.

1. `ADDI x3 (gp), x3, 8`
2. `LW x10, 28(x2)`
3. `SW x6, 4(x1)`
4. `ADD x10, x0, x11`

---

## Solution

### Part A: Decoding

---

#### Value 1: `0x4505`

Binary: `0100 0101 0000 0101`

```
[15:13] = 010   → funct3
[12]    = 0
[11:7]  = 01010 → rd/rs1 = x10 (a0)
[6:2]   = 00001 → imm[4:0]
[1:0]   = 01    → quadrant 1
```

Quadrant 1, `funct3 = 010`: this is **C.LI** (load immediate).

C.LI format: `funct3 | imm[5] | rd | imm[4:0] | op`

- `rd = 01010` = x10 (a0)
- `imm[5] = inst[12] = 0`
- `imm[4:0] = inst[6:2] = 00001`
- Immediate = `{0, 00001}` = `0b000001` = 1 (6-bit signed = positive)

**Decoded:** `C.LI a0, 1`

**32-bit equivalent:** `ADDI a0, x0, 1`

Verification: the instruction loads the small constant 1 into `a0`. This is the first argument register — a very common instruction pattern in function entry or loop setup.

---

#### Value 2: `0xC24C`

Binary: `1100 0010 0100 1100`

```
[1:0] = 00  → quadrant 0
```

Quadrant 0, `funct3 = inst[15:13]`:

```
[15:13] = 110   → funct3 = 110
[12:10] = 000   → imm part
[9:7]   = 100   → rs1' (3-bit register field)
[6:5]   = 10    → imm part
[4:2]   = 011   → rs2' (3-bit register field)
[1:0]   = 00    → quadrant 0
```

Quadrant 0, `funct3 = 110`: **C.SW** (stack-relative store, but actually CL/CS format for `rs1'` addressing).

C.SW format (CS format): `funct3 | imm[5:3] | rs1' | imm[2|6] | rs2' | op`

Re-extracting carefully:

```
Binary: 1100 0010 0100 1100
Bit:    15..                ..0

[15:13] = 110
[12:10] = 000        → imm[5:3] = 000
[9:7]   = 100        → rs1' = 100 = x12 (a2)
[6]     = 1          → imm[2] = 1
[5]     = 0          → imm[6] = 0  (for C.SD on RV64; for C.SW this is imm[2])
```

For C.SW: the immediate encoding is `{imm[5:3], imm[2], imm[6]}` — but C.SW has a 5-bit unsigned offset (word-aligned):

C.SW immediate: `uimm[5:3] = inst[12:10]`, `uimm[2] = inst[6]`, `uimm[6]` is not present for C.SW (C.SW max offset = 124 bytes). Let me use the correct C.SW encoding:

**C.SW encoding (CS format):**
```
[15:13] = 110          funct3
[12:10] = uimm[5:3]
[9:7]   = rs1'         base register
[6]     = uimm[2]
[5]     = uimm[6]      (only for C.SD; for C.SW this is also uimm[6] if present, but C.SW offset is 7 bits total: uimm[6:2])
```

Re-checking: C.SW uses a 5-bit offset scaled by 4 (covering 0-124 bytes):
- `uimm[5:3]` from `inst[12:10]`
- `uimm[2]` from `inst[6]`
- `uimm[6]` from `inst[5]`

```
[12:10] = 000  → uimm[5:3] = 000
[9:7]   = 100  → rs1' = x12 (a2)
[6]     = 1    → uimm[2] = 1
[5]     = 0    → uimm[6] = 0
[4:2]   = 011  → rs2' = x11 (a1)
```

The fields assemble as `uimm[6:2] = {uimm[6], uimm[5:3], uimm[2]} = {0, 000, 1}` = `00001` = 1, and since this is scaled by 4: offset = 4 bytes.

**Decoded:** `C.SW a1, 4(a2)`

**32-bit equivalent:** `SW a1, 4(a2)` — store word `a1` to address `a2 + 4`.

---

#### Value 3: `0x8082`

Binary: `1000 0000 1000 0010`

```
[1:0] = 10  → quadrant 2
[15:13] = 100  → funct3
[12] = 0
[11:7] = 00001  → rs1 (full 5-bit) = x1 (ra)
[6:2] = 00000   → rs2 = x0
```

Quadrant 2, `funct3 = 100`, `rs2 = x0`, `rs1 = ra`:

In the CR format (quadrant 2, `funct4 = 1000`, meaning `inst[12] = 0` with rs2 = x0):
This matches **C.JR** (jump register).

`C.JR rs1` where `rs1 != x0`: jump to address in `rs1`.

With `rs1 = x1 = ra`: this is a **function return**.

**Decoded:** `C.JR ra`

**32-bit equivalent:** `JALR x0, 0(ra)` — jump to address in `ra`, discard return address.

This is the most common 16-bit instruction in typical code: every function return is a `C.JR ra` (2 bytes) rather than `JALR x0, 0(ra)` (4 bytes).

---

### Part B: Encoding `ADDI sp, sp, -48`

This is a stack pointer adjustment: `sp = sp + (-48)`. On RISC-V, the frame size for typical functions is a multiple of 16 bytes (ABI alignment requirement for `sp`). The offset -48 is a multiple of 16, which matches **C.ADDI16SP**.

**C.ADDI16SP** encodes `ADDI sp, sp, imm` where `imm` is a non-zero multiple of 16 in the range ±496.

Format (CI variant, quadrant 1, `funct3 = 011`):
```
[15:13] = 011        funct3
[12]    = imm[9]
[11:7]  = 00010      rd = sp = x2 (signals C.ADDI16SP when rd = x2)
[6]     = imm[4]
[5]     = imm[6]
[4]     = imm[8]
[3]     = imm[7]
[2]     = imm[5]
[1:0]   = 01         quadrant 1
```

Immediate -48 in binary, scaled by 16 (divide by 16 first): `-48 / 16 = -3`.

The 6-bit non-zero signed field covering the nzimm[9:4] (steps of 16):
- -48 in 6-bit signed two's complement: 111101₂

```
nzimm[9] = 1
nzimm[8] = 1
nzimm[7] = 1
nzimm[6] = 1
nzimm[5] = 0
nzimm[4] = 1
```

Hmm — let me redo this. The immediate encoding for C.ADDI16SP uses the raw signed 10-bit value (in steps of 16, so bits [9:4] are the non-zero part):

-48 in 10-bit two's complement: `1111010000` (−48 = 512−48 = 464 = 0b111010000... let me compute):

-48 in binary (10-bit, two's complement): `1111010000`₂ = -48 check: `1111010000` → flip → `0000101111` = 47, +1 = 48. Yes.

So:
```
nzimm[9] = 1  (sign bit of 10-bit value)
nzimm[8] = 1
nzimm[7] = 1
nzimm[6] = 1
nzimm[5] = 0
nzimm[4] = 1
nzimm[3:0] = 0000  (always 0, multiple of 16)
```

Bit assignments to instruction fields:

```
[12]    = nzimm[9] = 1
[6]     = nzimm[4] = 1
[5]     = nzimm[6] = 1
[4]     = nzimm[8] = 1
[3]     = nzimm[7] = 1
[2]     = nzimm[5] = 0
```

Assembling the 16-bit encoding:

```
[15:13] = 011
[12]    = 1
[11:7]  = 00010   (x2 = sp)
[6]     = 1
[5]     = 1
[4]     = 1
[3]     = 1
[2]     = 0
[1:0]   = 01
```

Bit string: `011 1 00010 1 1 1 1 0 01`

```
15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
 0   1   1   1   0   0   0   1   0   1   1   1   1   0   0   1
```

= `0111 0001 0111 1001` = `0x7179`

**Verification:**

Decode `0x7179` = `0111 0001 0111 1001`:
- `[1:0] = 01` → quadrant 1
- `[15:13] = 011` → check C.ADDI16SP / C.LUI
- `[11:7] = 00010` → rd = x2 = sp → this is C.ADDI16SP
- Reconstruct immediate: `{inst[12], inst[4:3], inst[5], inst[6], 0000}` = `{1, 11, 1, 1, 0000}` = `1_11_1_1_0000` in nzimm positions `[9:8:7:6:5:4]` followed by zeros.
- = `1111010000` in 10-bit = -48. Correct.

**Encoded result:** `0x7179`

---

### Part C: Compressed Form Availability

---

**1. `ADDI x3 (gp), x3, 8`**

This is `ADDI gp, gp, 8`. The C.ADDI instruction (`ADDI rd, rd, imm`, `imm != 0`) uses a full 5-bit `rd` field in the CI format — it can address any of the 32 registers. `x3 (gp)` is encodable.

`imm = 8`, which is non-zero and fits in 6-bit signed range (-32 to +31). 8 ≤ 31, so it fits.

**Yes, a compressed form exists: C.ADDI x3, 8**

32-bit expansion: `ADDI x3, x3, 8`

---

**2. `LW x10, 28(x2)`**

`LW a0, 28(sp)` — loads from stack pointer + 28.

The compressed form for this is **C.LWSP** (load word, stack-pointer relative):
- `rd` = x10 (a0) — must be non-zero. x10 is non-zero. ✓
- Base register must be `sp (x2)`. ✓
- Offset: C.LWSP allows `uimm[7:2]` (6-bit, word-aligned), range 0–252.
  - 28 is within 0–252 and is word-aligned (28 % 4 = 0). ✓

**Yes, a compressed form exists: C.LWSP a0, 28(sp)**

32-bit expansion: `LW a0, 28(sp)`

---

**3. `SW x6, 4(x1)`**

`SW t1, 4(ra)` — stores to `ra + 4`. Not `sp`-relative.

For a non-sp-relative store, the applicable format is **C.SW** (CS format). C.SW requires:
- `rs1'` (3-bit): base register must be from {x8–x15}. `x1 (ra)` is NOT in this set (x1 < x8). ✗

There is no CSS (stack-relative) form because the base is `ra`, not `sp`. The CL/CS formats only allow `rs1'` from {x8–x15}.

**No compressed form exists.** `SW x6, 4(x1)` cannot be encoded as a 16-bit instruction because the base register `x1 (ra)` is outside the 3-bit RVC register range (x8–x15).

---

**4. `ADD x10, x0, x11`**

This is a register-to-register copy: `a0 = 0 + a1 = a1`.

The compressed form **C.MV** encodes `ADD rd, x0, rs2` where `rs2 != x0`:
- `rd` = x10 (a0) — must be non-zero. ✓
- `rs2` = x11 (a1) — must be non-zero. ✓
- `rs1` is implicitly x0 in C.MV. ✓
- Both `rd` and `rs2` use full 5-bit fields in the CR format — any register is accessible. ✓

**Yes, a compressed form exists: C.MV a0, a1**

32-bit expansion: `ADD a0, x0, a1`

---

## Discussion

### Key Takeaways

**Register constraints are the most common reason compression fails.** The CL/CS/CA/CB formats restrict source and destination registers to the 8-register subset `x8`–`x15`. The CR/CI/CSS formats use full 5-bit register specifiers and can address all 32 registers, but the instruction opcodes available in those formats are more limited (primarily stack-relative loads/stores and register arithmetic).

**The bit layout of compressed immediates is non-contiguous and non-obvious.** The C.BEQZ and C.ADDI16SP encodings scatter immediate bits across the instruction word in an irregular pattern. In hardware, this means the immediate decode path for 16-bit instructions is a collection of wires routing specific bits to their logical positions — more complex than the contiguous immediate fields of 32-bit formats. In an interview, drawing the bit field diagram before attempting to assemble the value avoids errors.

**C.JR ra is the single most common compressed instruction in typical code.** Every function return uses it. Measuring the frequency of specific compressed instructions reveals that five instructions — `C.LW`, `C.SW`, `C.ADDI`, `C.LI`, and `C.JR` — account for the majority of compressed instruction use in real programs.

**Encoding verification strategy.** A reliable approach for manual encoding:
1. Identify the instruction class (CR, CI, CSS, CIW, CL, CS, CA, CB, CJ).
2. Write out the empty bit field template for that format.
3. Fill in `funct3`, `op` (quadrant), register fields, and immediate fields in order.
4. Assemble the 16-bit value.
5. Decode it back independently to verify.

This two-pass approach catches most encoding errors before they propagate to hardware or simulation.
