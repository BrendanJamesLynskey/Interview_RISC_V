# Problem 01: Instruction Decoding

**Topic:** RISC-V instruction format identification and field extraction  
**Difficulty:** Fundamentals (Parts A-C), Intermediate (Parts D-E), Advanced (Part F)  
**Prerequisites:** `instruction_formats.md`, `rv32i_base_instructions.md`

---

## Problem Statement

Each part gives you either a 32-bit hex encoding or an assembly instruction. You must perform the opposite conversion: hex-to-assembly or assembly-to-hex. Show all working.

---

### Part A (Fundamentals)

Decode the following hex word to an assembly instruction. Show every field.

```
0x003100B3
```

---

### Part B (Fundamentals)

Decode to assembly:

```
0xFE010113
```

---

### Part C (Fundamentals)

Encode the following instruction to a 32-bit hex word:

```assembly
SW x14, -8(x2)
```

---

### Part D (Intermediate)

Decode to assembly. Be precise about the immediate value and its sign.

```
0xFFC4A303
```

---

### Part E (Intermediate)

Encode to hex. Show the split immediate carefully.

```assembly
BNE x5, x6, -20
```

---

### Part F (Advanced)

The following four hex words are consecutive instructions starting at address `0x1000`. Decode all four and describe what the sequence as a whole accomplishes.

```
Address 0x1000:  0x00001537
Address 0x1004:  0xABC50513
Address 0x1008:  0x00050463
Address 0x100C:  0x00150513
```

---

## Solutions

### Part A Solution

**Input:** `0x003100B3`

Step 1 — write out the binary:
```
0x003100B3 = 0000 0000 0011 0001 0000 0000 1011 0011
```

Step 2 — extract opcode (bits [6:0]):
```
bits [6:0] = 011 0011 = 0x33 = 51
opcode 0110011 => OP (integer register-register, R-type)
```

Step 3 — extract remaining R-type fields:
```
bits [11:7]  = 0 0001 = 1         => rd = x1
bits [14:12] = 000               => funct3 = 0 (ADD/SUB)
bits [19:15] = 0 0010 = 2        => rs1 = x2
bits [24:20] = 0 0011 = 3        => rs2 = x3
bits [31:25] = 000 0000          => funct7 = 0 (ADD, not SUB)
```

Step 4 — identify operation:
```
opcode=0110011, funct3=000, funct7=0000000 => ADD
```

Step 5 — write assembly:
```assembly
ADD x1, x2, x3
```

Verification:
```
funct7=0000000, rs2=00011, rs1=00010, funct3=000, rd=00001, opcode=0110011
Binary: 0000000_00011_00010_000_00001_0110011
= 0000 0000 0011 0001 0000 0000 1011 0011
= 0x003100B3  (matches)
```

---

### Part B Solution

**Input:** `0xFE010113`

Step 1 — binary:
```
0xFE010113 = 1111 1110 0000 0001 0000 0001 0001 0011
```

Step 2 — opcode (bits [6:0]):
```
bits [6:0] = 001 0011 = 0x13
opcode 0010011 => OP-IMM (I-type)
```

Step 3 — extract I-type fields:
```
bits [11:7]  = 0 0010 = 2        => rd = x2
bits [14:12] = 000               => funct3 = 0 (ADDI)
bits [19:15] = 0 0010 = 2        => rs1 = x2
bits [31:20] = 1111 1110 0000    => imm[11:0] = 111111100000
```

Step 4 — sign-extend the 12-bit immediate:
```
imm[11:0] = 111111100000
MSB (bit 11) = 1 => negative

Value: two's complement of 111111100000
= -(~111111100000 + 1)
= -(000000011111 + 1)
= -(000000100000)
= -(32)
= -32

Verification: 0xFE0 = 1111 1110 0000
As signed 12-bit: 0xFE0 - 0x1000 = -32  (correct)
```

Step 5 — assembly:
```assembly
ADDI x2, x2, -32
```

Using ABI names:
```assembly
addi sp, sp, -32
```

This is a canonical stack frame allocation: the prologue of any function that needs 32 bytes of stack space. `sp -= 32`.

---

### Part C Solution

**Target:** `SW x14, -8(x2)`

Identify format: SW is a store instruction — **S-type**.

```
opcode = 0100011  (STORE)
funct3 = 010      (SW = 32-bit store)
rs1    = x2  = 00010  (base address register)
rs2    = x14 = 01110  (register to store)
offset = -8
```

Split the offset:
```
-8 in 12-bit two's complement:
  8  = 0000_0000_1000
 -8  = 1111_1111_1000  (12-bit two's complement)

imm[11:5] = upper 7 bits = 1111111  (all ones)
imm[4:0]  = lower 5 bits = 11000    (wait: 1111_1111_1000, lower 5 bits = 11000? No)

Let us be precise:
-8 in 12-bit binary: 1111_1111_1000
                     ^bit11     ^bit0

imm[11:5] = bits 11 down to 5 = 1111_111  (7 bits) = 1111111
imm[4:0]  = bits 4 down to 0  = 1_1000    (5 bits) = 11000

Wait: 1111_1111_1000
  bit 11 = 1
  bit 10 = 1
  bit  9 = 1
  bit  8 = 1
  bit  7 = 1
  bit  6 = 1
  bit  5 = 1
  bit  4 = 1
  bit  3 = 1
  bit  2 = 0
  bit  1 = 0
  bit  0 = 0

imm[11:5] = bits[11:5] = 1111111
imm[4:0]  = bits[4:0]  = 11000
```

Assemble:
```
inst[31:25] = imm[11:5] = 1111111
inst[24:20] = rs2       = 01110   (x14)
inst[19:15] = rs1       = 00010   (x2)
inst[14:12] = funct3    = 010     (SW)
inst[11:7]  = imm[4:0]  = 11000
inst[6:0]   = opcode    = 0100011

Binary: 1111111_01110_00010_010_11000_0100011
```

Convert to hex:
```
1111 1110 1110 0001 0010 1100 0010 0011
= 0xFEE12C23
```

Verification — reconstruct offset from encoding:
```
inst[31:25] = 1111111 => imm[11:5]
inst[11:7]  = 11000   => imm[4:0]
Full 12-bit: 1111111_11000 = 1111_1111_1000 = 0xFF8
0xFF8 as signed 12-bit = 0xFF8 - 0x1000 = -8  (correct)
```

---

### Part D Solution

**Input:** `0xFFC4A303`

Step 1 — binary:
```
0xFFC4A303 = 1111 1111 1100 0100 1010 0011 0000 0011
```

Step 2 — opcode:
```
bits [6:0] = 000 0011 = 0x03
opcode 0000011 => LOAD (I-type)
```

Step 3 — I-type fields:
```
bits [11:7]  = 0 0110 = 6        => rd = x6
bits [14:12] = 010               => funct3 = 2 => LW (32-bit load, signed)
bits [19:15] = 0 1001 = 9        => rs1 = x9
bits [31:20] = 1111 1111 1100    => imm[11:0] = 111111111100
```

Step 4 — sign-extend immediate:
```
imm[11:0] = 111111111100
MSB = 1 (bit 11 = 1) => negative value

Two's complement:
  Invert:  000000000011
  Add 1:   000000000100
  = 4

So the value is -4.

Sign-extended to 32 bits: 0xFFFFFFFC = -4
```

Step 5 — assembly:
```assembly
LW x6, -4(x9)
```

Using ABI names: `lw a2, -4(s1)`

This loads a 32-bit word from address `(x9 - 4)` into x6. A common pattern for reading the last element before the current pointer, or for loading a value from just below the stack pointer.

---

### Part E Solution

**Target:** `BNE x5, x6, -20`

Identify format: BNE is a branch — **B-type**.

```
opcode = 1100011  (BRANCH)
funct3 = 001      (BNE — branch if not equal)
rs1    = x5 = 00101
rs2    = x6 = 00110
offset = -20
```

Step 1 — represent -20 as a 13-bit signed value:
```
20  = 0001_0100_0 (13 bits, with bit 0 = 0)
-20 two's complement:
  ~20:  1110_1011_1111 + 1... let me work in 13 bits:
  20 in 13 bits = 0b00000_0001_0100 (bits 12:0, bit 0 = 0)
  = 0_0000_0001_0100

  Invert all 13 bits: 1_1111_1110_1011
  Add 1:              1_1111_1110_1100
  = -20

  Bit layout of -20 (13-bit):
    bit 12 = 1  (sign)
    bit 11 = 1
    bit 10 = 1
    bit  9 = 1
    bit  8 = 1
    bit  7 = 1
    bit  6 = 1
    bit  5 = 1
    bit  4 = 1
    bit  3 = 0
    bit  2 = 1
    bit  1 = 1 ... wait, let me recompute:

20 = 16 + 4 = 0b0000_0001_0100
  bit 4 = 1 (16? No: 16 = 2^4, bit 4 = 1)
  bit 2 = 1 (4 = 2^2)
  others = 0

So 20 in bits: bit4=1, bit2=1 (bit0=0 implicit for branch offsets)

Full 13-bit: 0_0000_0001_0100
              ^bit12      ^bit0

Invert: 1_1111_1110_1011
Add 1:  1_1111_1110_1100

-20 in 13-bit:
  bit 12 = 1
  bit 11 = 1
  bit 10 = 1
  bit  9 = 1
  bit  8 = 1
  bit  7 = 1
  bit  6 = 1
  bit  5 = 1
  bit  4 = 0
  bit  3 = 1
  bit  2 = 1
  bit  1 = 0
  bit  0 = 0 (implicit, not stored)
```

Step 2 — extract sub-fields for B-type encoding:
```
imm[12]   = bit 12 = 1
imm[11]   = bit 11 = 1
imm[10:5] = bits 10:5 = 111111
imm[4:1]  = bits 4:1  = 0110   (bit4=0, bit3=1, bit2=1, bit1=0)
imm[0]    = 0 (not stored)
```

Step 3 — place into B-type instruction word:
```
inst[31]   = imm[12] = 1
inst[30:25]= imm[10:5] = 111111
inst[24:20]= rs2 = 00110  (x6)
inst[19:15]= rs1 = 00101  (x5)
inst[14:12]= funct3 = 001  (BNE)
inst[11:8] = imm[4:1] = 0110
inst[7]    = imm[11] = 1
inst[6:0]  = opcode = 1100011

Binary: 1_111111_00110_00101_001_0110_1_1100011

Assembling 32 bits:
bit 31:    1
bits 30:25: 111111
bits 24:20: 00110
bits 19:15: 00101
bits 14:12: 001
bits 11:8:  0110
bit 7:      1
bits 6:0:   1100011

= 1111 1110 0110 0010 1001 0110 1110 0011
```

Convert to hex:
```
1111 1110 0110 0010 1001 0110 1110 0011
= 0xFE6296E3
```

Verification — reconstruct offset:
```
inst[31]   = 1   => imm[12] = 1  (sign: negative)
inst[7]    = 1   => imm[11] = 1
inst[30:25]= 111111 => imm[10:5] = 111111
inst[11:8] = 0110   => imm[4:1] = 0110

Full 13-bit: {1, 1, 111111, 0110, 0}
= 1_1111_1110_1100_0  -- wait, let me reconstruct carefully:
{imm[12], imm[11], imm[10:5], imm[4:1], imm[0]}
= {1, 1, 111111, 0110, 0}
= 1_1_111111_0110_0
= 1 1111 1101 1000 ... hmm

Let me lay out the 13 bits:
  Position: [12][11][10][9][8][7][6][5][4][3][2][1][0]
  Value:      1   1   1  1  1  1  1  1  0  1  1  0  0

= 1_1111_1110_1100 (interpreting bit 0 as implicit 0)
= as unsigned: 0x1FEC
= as signed 13-bit: 0x1FEC - 0x2000 = -20  (correct, since 0x2000 = 2^13)
```

---

### Part F Solution

**Input:** Four consecutive instructions at 0x1000-0x100C

**Decode each instruction:**

**0x1000: `0x00001537`**
```
Binary: 0000 0000 0000 0000 0001 0101 0011 0111
bits [6:0]  = 011 0111 => LUI (U-type)
bits [11:7] = 01010 = 10 => rd = x10 (a0)
bits [31:12]= 0000 0000 0000 0000 0001 = 0x00001

Assembly: LUI a0, 0x1
Result:   a0 = 0x00001000
```

**0x1004: `0xABC50513`**
```
Binary: 1010 1011 1100 0101 0000 0101 0001 0011
bits [6:0]  = 001 0011 => OP-IMM (I-type)
bits [11:7] = 01010 = 10 => rd = x10 (a0)
bits [14:12]= 000 => funct3 = 0 => ADDI
bits [19:15]= 01010 = 10 => rs1 = x10 (a0)
bits [31:20]= 1010 1011 1100 => imm[11:0] = 101010111100

Sign-extend 12-bit value:
  0xABC = 1010 1011 1100
  MSB = 1 => negative
  0xABC - 0x1000 = -0x544 = -1348

Assembly: ADDI a0, a0, -1348
Result:   a0 = 0x00001000 + 0xFFFFFABC = 0x00000ABC
          (i.e., a0 = 0x00001ABC... let me recheck:
           0xABC as signed 12-bit = 0xABC - 0x1000 = -1348 = 0xFFFFFABC sign-extended
           a0 was 0x00001000
           0x00001000 + 0xFFFFFABC = 0x00000ABC)
```

Wait — rechecking: `0xABC = 2748` unsigned. As 12-bit signed: bit 11 of 0xABC is bit 11 of `1010 1011 1100` = 1 (the leading 1), so it is negative. `-1348 = 0xFFFFFABC`. Then `0x1000 + 0xFFFFFABC = 0x00000ABC`. So a0 = 0x00000ABC.

Actually, let me re-examine the intent. This is loading `0x1ABC` into a0 via LUI + ADDI:
- Target: suppose the programmer wanted a0 = some constant.
- LUI a0, 0x1 => a0 = 0x1000
- ADDI a0, a0, 0xABC... but 0xABC has bit 11 set, so it sign-extends to -1348.
- Result: 0x1000 - 1348 = 0x1000 - 0x544 = 0xABC.

So a0 = `0x00000ABC` after instruction 2.

```
Hmm. More likely the programmer intended a0 = 0x1ABC. To get 0x1ABC:
  0x1ABC bit [11] = 1010 ... bit 11 = 1 (0x1ABC = 0001_1010_1011_1100, bit 11 of lower 12 = A = 1010, bit 11 = 1)
  So they need to add 1 to the upper part:
  LUI a0, 0x2   (not 0x1) => a0 = 0x2000
  ADDI a0, a0, 0xABC(-1348) => a0 = 0x2000 - 1348 = 0x2000 - 0x544 = 0x1ABC

  But the encoding uses LUI 0x1, not 0x2. So a0 ends up as 0x0ABC, not 0x1ABC.
  The programmer has the LUI+ADDI pitfall bug! (or they intended 0x0ABC)

For this problem, let us take the encoding at face value: a0 = 0x00000ABC.
```

**0x1008: `0x00050463`**
```
Binary: 0000 0000 0000 0101 0000 0100 0110 0011
bits [6:0]  = 110 0011 => BRANCH (B-type)
bits [14:12]= 000 => funct3 = 0 => BEQ
bits [19:15]= 01010 = 10 => rs1 = x10 (a0)
bits [24:20]= 00000 = 0  => rs2 = x0 (zero)

Extract B-type immediate:
  inst[31]   = 0 => imm[12] = 0
  inst[7]    = 0 => imm[11] = 0
  inst[30:25]= 000000 => imm[10:5] = 0
  inst[11:8] = 0100   => imm[4:1] = 0100 = 4

  imm = {0, 0, 000000, 0100, 0} = 0b0000_0000_1000 = 8

Assembly: BEQ a0, zero, +8
Semantics: if a0 == 0, branch to (0x1008 + 8) = 0x1010
```

**0x100C: `0x00150513`**
```
Binary: 0000 0000 0001 0101 0000 0101 0001 0011
bits [6:0]  = 001 0011 => OP-IMM (I-type)
bits [11:7] = 01010 = 10 => rd = x10 (a0)
bits [14:12]= 000 => ADDI
bits [19:15]= 01010 = 10 => rs1 = x10 (a0)
bits [31:20]= 0000 0000 0001 => imm = 1

Assembly: ADDI a0, a0, 1
Semantics: a0 = a0 + 1
```

**Overall sequence interpretation:**

```
Address  Instruction            a0 value after
-------  ---------------------- ---------------
0x1000   LUI  a0, 0x1           0x00001000
0x1004   ADDI a0, a0, 0xABC     0x00000ABC   (NOTE: LUI+ADDI pitfall — intended 0x1ABC?)
0x1008   BEQ  a0, zero, +8      branches to 0x1010 IF a0 == 0
0x100C   ADDI a0, a0, 1         a0 = 0x00000ABD (only reached if a0 != 0)
0x1010   (next instruction, branch target)
```

**What this sequence accomplishes:**

1. Loads the constant `0x00000ABC` into `a0` (via LUI+ADDI).
2. Tests if `a0` is zero. Since `0xABC != 0`, the branch is **not taken**.
3. Increments `a0` by 1, yielding `0xABD`.
4. Falls through to `0x1010`.

The BEQ at 0x1008 tests the just-computed constant, which can never be zero (because `0xABC != 0`). The branch is therefore **never taken** — it is dead code or the result of a code generation error. A real use case would have the BEQ after a computation whose result could be zero (e.g., after a subtraction used as a loop counter).

**Key observation — the LUI/ADDI pitfall:**
If the programmer intended `a0 = 0x1ABC`:
- They needed `LUI a0, 0x2` (not `0x1`) to compensate for the sign extension of `0xABC`.
- With `LUI a0, 0x1`, `a0 = 0x00000ABC`, not `0x1ABC`.
- This is the classic sign-extension compensation error described in `immediate_encoding.md`.

---

## Discussion

### Common Errors in Instruction Decoding

**1. Misidentifying the format before extracting fields.**
Always decode the opcode field `[6:0]` first. The opcode determines the format, which determines all subsequent field boundaries. A common error is to assume I-type and extract a 12-bit immediate when the instruction is actually B-type (where bits [11:7] are partially an immediate, not `rd`).

**2. Forgetting sign extension on the immediate.**
The raw bit pattern of the immediate (e.g., `0xABC`) is not the value used in the computation. Always sign-extend the field to 32 bits. An immediate of `0x800` is **-2048**, not **+2048**.

**3. Confusing B-type immediate reconstruction.**
The two "scrambled" bits (`imm[11]` at `inst[7]` and `imm[12]` at `inst[31]`) are the most common source of errors. Always use the reconstruction formula:
```
b_imm = {imm[12], imm[11], imm[10:5], imm[4:1], 1'b0}
       = {inst[31], inst[7], inst[30:25], inst[11:8], 1'b0}
```

**4. Off-by-one on branch targets.**
The branch offset is **PC-relative**, not absolute. The target address is `PC_of_branch + offset`, not `0 + offset`. On a multiple-choice exam, distractors will often present the offset value as the target address.

**5. Forgetting that JAL and branch offsets have an implicit `1'b0`.**
The stored immediate bits represent `offset >> 1` (the offset divided by 2). Multiply by 2 (or append a `0` bit) when computing the actual byte offset.

### Decoding Strategy (Exam Technique)

When given a hex word and asked to decode:

```
1. Write the 32-bit binary.
2. Extract bits [6:0] => identify opcode => identify format.
3. Extract bits [11:7] => rd (R/I/U/J-type) or imm[4:0] (S/B-type).
4. Extract bits [14:12] => funct3.
5. Extract bits [19:15] => rs1 (if format uses it).
6. Extract bits [24:20] => rs2 (R/S/B-type) or upper imm (I-type).
7. Extract bits [31:25] => funct7 (R-type) or upper imm (I/S/B/J/U-type).
8. Reconstruct the immediate using the format-specific formula.
9. Sign-extend the immediate.
10. Name the instruction using opcode + funct3 (+ funct7 for R-type).
```
