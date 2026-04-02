# Immediate Encoding

## Prerequisites
- RISC-V instruction formats (R/I/S/B/U/J) — see `instruction_formats.md`
- Two's complement signed integers
- Binary bit manipulation: shifts, masks, concatenation

---

## Concept Reference

### Why Immediates Are Embedded in Instructions

RISC-V, like all load-store RISC architectures, encodes small constants directly inside the instruction word. This avoids a memory access to fetch the constant and is faster in almost every context. The cost is that the constant must fit in fewer bits than a full register — 12 bits for most arithmetic and memory instructions, 20 bits for upper-immediate instructions, 13 or 21 bits for branches and jumps.

The five distinct immediate shapes in RV32I are:

```
Format  Width   Signed range               Used by
------  ------  -------------------------  ----------------------------------------
I-imm   12-bit  -2048      to +2047        ADDI, LW, LB, JALR, and all OP-IMM
S-imm   12-bit  -2048      to +2047        SW, SH, SB
B-imm   13-bit  -4096      to +4094        BEQ, BNE, BLT, BGE, BLTU, BGEU
U-imm   20-bit  forms upper 20 bits        LUI, AUIPC
J-imm   21-bit  -1,048,576 to +1,048,574  JAL
```

All immediate values are **sign-extended** to 32 bits before being used in any computation.

---

### I-type Immediate: Contiguous 12-bit Field

The simplest encoding. The 12-bit immediate occupies a single contiguous field at bits [31:20] of the instruction word.

```
 31          20  19   15  14  12  11    7   6      0
+--------------+-------+------+--------+---------+
|   imm[11:0]  |  rs1  |funct3|   rd   |  opcode |
+--------------+-------+------+--------+---------+

Extraction: imm = { {20{inst[31]}}, inst[31:20] }
```

The sign bit of the immediate is always `inst[31]` — this invariant holds across all five immediate forms and is a deliberate design choice (see Question I1 below).

Example: `ADDI x3, x1, -5`
```
-5 in 12-bit two's complement = 111111111011

 31       20  19  15  14  12  11   7  6     0
+------------+------+------+-------+--------+
|111111111011|00001 | 000  | 00011 |0010011 |
+------------+------+------+-------+--------+
  imm=-5      rs1=x1 ADD    rd=x3   OP-IMM

Binary: 1111_1111_1011_0000_1000_0001_1001_0011
Hex:    0xFFB08193
```

Sign extension to 32 bits:
```
12-bit:  1111_1111_1011                      (-5)
32-bit:  1111_1111_1111_1111_1111_1111_1111_1011  (still -5, correct)
```

---

### S-type Immediate: Split Across Two Fields

Store instructions have no destination register. The 5 bits normally used for `rd` are repurposed for the lower portion of the store offset. The immediate is split into two non-contiguous pieces that must be reassembled.

```
 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
| imm[11:5]|  rs2  |  rs1  |funct3|imm[4:0]|  opcode |
+----------+-------+-------+------+--------+---------+

Extraction: imm = { {20{inst[31]}}, inst[31:25], inst[11:7] }

Note: inst[31:25] provides imm[11:5]
      inst[11:7]  provides imm[4:0]
```

Why split? Because `rs1` (the base address register) and `rs2` (the register to be stored) must stay at their canonical positions [19:15] and [24:20] — the same positions used in R-type and B-type. Keeping register fields fixed lets the decoder read the register file before it knows which format the instruction uses.

Example: `SW x10, 20(x2)` — store word from x10 to address (x2 + 20)
```
20 = 0b0000_0001_0100
imm[11:5] = 0000000
imm[4:0]  = 10100

 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
| 0000000  | 01010 | 00010 | 010  | 10100  | 0100011 |
+----------+-------+-------+------+--------+---------+
 imm[11:5]  rs2=x10 rs1=x2  SW    imm[4:0]  STORE

Binary: 0000_0000_1010_0001_0010_1010_0010_0011
Hex:    0x00A12A23
```

Reconstruction:
```
inst[31:25] = 0000000  => imm[11:5]
inst[11:7]  = 10100    => imm[4:0]
Concatenated: 000000010100 = 20  (correct)
```

---

### B-type Immediate: Scrambled for Branch Offsets

Branch instructions encode a 13-bit PC-relative offset (the LSB is implicitly zero because instructions are always at least 2-byte aligned). The 13 bits occupy four non-contiguous sub-fields.

```
 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
|i[12|10:5]|  rs2  |  rs1  |funct3|i[4:1|11]| opcode |
+----------+-------+-------+------+--------+---------+

Bit extraction:
  imm[12]   = inst[31]     -- sign bit, as always
  imm[11]   = inst[7]      -- this is the "scrambled" bit
  imm[10:5] = inst[30:25]
  imm[4:1]  = inst[11:8]
  imm[0]    = 0 (implicit)

Reconstruction:
  imm = { {19{inst[31]}}, inst[31], inst[7], inst[30:25], inst[11:8], 1'b0 }
```

The placement of `imm[11]` at `inst[7]` (rather than `inst[31]`) is the only deviation from what would otherwise be a straightforward field assignment. This was specifically chosen so that the S-type and B-type encodings differ in only two bit positions (bit 7 and bit 31), allowing a very small hardware difference between store and branch address generation circuits.

Comparison of S-type and B-type bit usage:
```
Bit 31:  S = imm[11]    B = imm[12]   (both are MSB of their respective immediates)
Bit 30:  S = imm[10]    B = imm[10]   (identical)
...
Bit 25:  S = imm[5]     B = imm[5]    (identical)
Bit 11:  S = imm[4]     B = imm[4]    (identical)
...
Bit 8:   S = imm[1]     B = imm[1]    (identical)
Bit 7:   S = imm[0]     B = imm[11]   (differ: S stores imm[0], B "rotates" imm[11] here)
```

Example: `BEQ x5, x6, +16` — branch 16 bytes forward if x5 == x6
```
offset = 16 = 0b0000_0001_0000
  imm[12]  = 0
  imm[11]  = 0
  imm[10:5]= 000000
  imm[4:1] = 1000   (16 >> 1 = 8 = 0b1000)
  imm[0]   = 0 (implicit)

inst[31]   = imm[12]  = 0
inst[30:25]= imm[10:5]= 000000
inst[24:20]= rs2=x6   = 00110
inst[19:15]= rs1=x5   = 00101
inst[14:12]= funct3   = 000 (BEQ)
inst[11:8] = imm[4:1] = 1000
inst[7]    = imm[11]  = 0
inst[6:0]  = opcode   = 1100011

Binary: 0000000_00110_00101_000_10000_1100011
Hex:    0x00628863
```

Verify reconstruction:
```
inst[31]   = 0  => imm[12] = 0
inst[7]    = 0  => imm[11] = 0
inst[30:25]= 000000 => imm[10:5] = 0
inst[11:8] = 1000   => imm[4:1] = 8
imm = {0, 0, 000000, 1000, 0} = 0b0000_0001_0000 = 16 bytes  (correct)
```

---

### U-type Immediate: Upper 20 Bits, No Scrambling

U-type is the simplest of all: the 20-bit immediate occupies bits [31:12] directly. No sign extension is needed because the value is placed in the upper 20 bits of the register; bits [11:0] of the result are forced to zero.

```
 31                          12  11    7   6      0
+--------------------------------+--------+---------+
|          imm[31:12]            |   rd   |  opcode |
+--------------------------------+--------+---------+

Result in rd = { inst[31:12], 12'b000000000000 }
```

Note that despite having 20 bits in the instruction word, the value placed into the register is 32 bits wide (the lower 12 bits are zeroed). This is not sign extension — U-type always produces a 32-bit result with zero in the lower 12 bits.

Example: `LUI x15, 0xABCDE` — load 0xABCDE000 into x15
```
imm[31:12] = 0xABCDE = 1010_1011_1100_1101_1110

inst[31:12]= 10101011110011011110
inst[11:7] = 01111  (x15)
inst[6:0]  = 0110111  (LUI opcode)

Binary: 10101011110011011110_01111_0110111
Hex:    0xABCDE7B7

Result: x15 = 0xABCDE000
```

For AUIPC: result = PC + {imm[31:12], 12'b0}, the same immediate shape, but added to the program counter rather than placed directly.

---

### J-type Immediate: Scrambled 21-bit Jump Offset

JAL's immediate encodes a 21-bit PC-relative offset (bit 0 implicit). Like B-type, bits are scrambled — but with a different permutation chosen to maximise how much of the J-type encoding overlaps with the U-type encoding (they share the same opcode position and rd position).

```
 31                          12  11    7   6      0
+--------------------------------+--------+---------+
| imm[20|10:1|11|19:12]          |   rd   |  opcode |
+--------------------------------+--------+---------+

Bit extraction:
  imm[20]   = inst[31]      -- sign bit
  imm[19:12]= inst[19:12]   -- eight contiguous bits (matches U-type imm[19:12])
  imm[11]   = inst[20]
  imm[10:1] = inst[30:21]
  imm[0]    = 0 (implicit)

Reconstruction:
  imm = { {11{inst[31]}}, inst[31], inst[19:12], inst[20], inst[30:21], 1'b0 }
```

The overlap with U-type is deliberate: bits [19:12] of the immediate are in the same position in both U-type and J-type. This means a hardware implementation can share the decode path for the upper byte of the immediate between LUI, AUIPC, and JAL.

Example: `JAL x1, +256` — call subroutine 256 bytes forward, return address in x1 (ra)
```
offset = 256 = 0x100 = 0b0000_0001_0000_0000_0000_0 (21-bit, bit 0 implicit)
  imm[20]   = 0
  imm[19:12]= 00000000
  imm[11]   = 0
  imm[10:1] = 1000000000  (256 >> 1 = 128 = 0b10000000, in 10 bits: 0010000000)

Wait, let us work through carefully:
  256 = 0b0_0000_0001_0000_0000_0 (20 bits of offset, bit 0 = 0)
  imm[20]   = 0
  imm[19:12]= 0000_0000
  imm[11]   = 0
  imm[10:1] = 00_1000_0000  (bits 10 down to 1 of 256: 256 = 0x100,
                              bits[10:1] = 0b0010000000 wait:
                              256 in binary = 1_0000_0000
                              imm[8] = 1, all others 0
                              imm[10:1] = 0b0010000000 ... )

Precise bit layout of 256:
  bit 0  = 0  (implicit, always)
  bit 1  = 0
  bit 2  = 0
  bit 3  = 0
  bit 4  = 0
  bit 5  = 0
  bit 6  = 0
  bit 7  = 0
  bit 8  = 1   <-- 256 = 2^8
  bit 9  = 0
  bit 10 = 0
  bit 11 = 0
  bit 12 = 0
  ...
  bit 20 = 0  (sign bit, positive)

imm[10:1] = bits 10 down to 1 = 0b00_1000_0000 = 0b0010000000
imm[11]   = bit 11 = 0
imm[19:12]= bits 19 down to 12 = 0b0000_0000
imm[20]   = 0

Encoding into instruction word:
  inst[31]   = imm[20]   = 0
  inst[30:21]= imm[10:1] = 0010000000
  inst[20]   = imm[11]   = 0
  inst[19:12]= imm[19:12]= 00000000
  inst[11:7] = rd = x1   = 00001
  inst[6:0]  = opcode    = 1101111

Binary: 0_0010000000_0_00000000_00001_1101111
Hex:    0x100000EF
```

---

### The Critical Hardware Design Invariant: Sign Bit Always at Bit 31

Across all five immediate forms:

```
Format   Sign bit of immediate   Located at instruction bit
------   ---------------------   --------------------------
I-type   imm[11]                 inst[31]
S-type   imm[11]                 inst[31]
B-type   imm[12]                 inst[31]
U-type   imm[31]                 inst[31]  (also the MSB placed in register)
J-type   imm[20]                 inst[31]
```

This invariant means the sign-extension hardware requires only a single input wire, `inst[31]`, regardless of format:

```verilog
// Sign extension in Verilog — no format-dependent muxing for sign bit:
wire sign = instruction[31];   // always the same wire

wire [31:0] imm_i = {{20{sign}}, instruction[31:20]};
wire [31:0] imm_s = {{20{sign}}, instruction[31:25], instruction[11:7]};
wire [31:0] imm_b = {{19{sign}}, sign, instruction[7],
                     instruction[30:25], instruction[11:8], 1'b0};
wire [31:0] imm_u = {instruction[31:12], 12'b0};     // no sign extension needed
wire [31:0] imm_j = {{11{sign}}, sign, instruction[19:12],
                     instruction[20], instruction[30:21], 1'b0};
```

If, hypothetically, the sign bit were at a different position for each format, the sign extension hardware would need a 5-way mux on the sign bit input — adding gates and increasing the critical path delay through the decode stage.

---

### Sign Extension: Mechanics and Pitfalls

Sign extension converts a narrow signed integer to a wider one while preserving its value. The rule: replicate the MSB (sign bit) into all new high-order bit positions.

```
12-bit value:  0111_1111_1111  (+2047)  =>  32-bit: 0000_0000_0000_0000_0000_0111_1111_1111
12-bit value:  1000_0000_0000  (-2048)  =>  32-bit: 1111_1111_1111_1111_1111_1000_0000_0000
12-bit value:  1111_1111_1111  (-1)     =>  32-bit: 1111_1111_1111_1111_1111_1111_1111_1111
```

**The LUI + ADDI pitfall** is the most common sign-extension bug in RISC-V assembly:

To synthesise any 32-bit constant `C`, the naive approach is:
```assembly
lui  rd, C[31:12]      # place upper 20 bits
addi rd, rd, C[11:0]   # add lower 12 bits
```

This fails when `C[11]` (bit 11 of the constant) is 1, because ADDI sign-extends its 12-bit immediate. A 12-bit value with bit 11 set is interpreted as negative, so ADDI subtracts rather than adding.

Correction: if `C[11] = 1`, add 1 to the upper immediate to pre-compensate for the subtraction.

```
C = 0xABCD_E F00
    ^^^^^^^^^      ^--- lower 12 = 0xF00
    upper 20 = 0xABCDE

0xF00 = 1111_0000_0000  -- bit 11 = 1, so ADDI will sign-extend to 0xFFFFF F00 = -256

Without compensation:
  lui  rd, 0xABCDE     # rd = 0xABCDE000
  addi rd, rd, 0xF00   # rd = 0xABCDE000 + 0xFFFFF F00 = 0xABCDD F00   WRONG

With compensation (add 1 to upper immediate):
  lui  rd, 0xABCDF     # rd = 0xABCDF000
  addi rd, rd, 0xF00   # rd = 0xABCDF000 + 0xFFFFF F00 = 0xABCDE F00   CORRECT
```

General rule in C-like notation:
```c
uint32_t upper = (C + 0x800) >> 12;   // add 0x800 to round up if bit 11 set
int32_t  lower = C - (upper << 12);   // lower will be in [-2048, +2047]
// Emit: LUI rd, upper; ADDI rd, rd, lower
```

The GNU assembler's `%hi()` and `%lo()` macros perform this calculation automatically:
```assembly
lui  a0, %hi(0xABCDEF00)    # assembler computes correct upper
addi a0, a0, %lo(0xABCDEF00) # assembler computes correct lower
```

---

## Tier 1 — Fundamentals

### Question F1
**What is the range of the immediate in an ADDI instruction? Why is it not -4096 to +4095 (a 13-bit range)?**

**Answer:**

ADDI is an I-type instruction. The immediate field is 12 bits wide, located at bits [31:20] of the instruction word. A 12-bit two's complement value has:
- Minimum: `-2^11` = `-2048` = `0x800` (12-bit) => sign-extended to `0xFFFFF800`
- Maximum: `2^11 - 1` = `+2047` = `0x7FF` (12-bit) => sign-extended to `0x000007FF`

The range is **-2048 to +2047**, which is 12 bits (4096 total values).

It is not 13-bit because the I-type immediate field is only 12 bits. The 13-bit range (-4096 to +4094) belongs to B-type branch offsets, which encode bit 0 implicitly (always zero) and use a 13-bit signed value in 12 stored bits plus 1 implicit bit. Each step in a branch offset is a multiple of 2, so the effective range for instruction targets is the same 4096 values but spaced 2 bytes apart.

**Common mistake:** Confusing the 12-bit immediate range of arithmetic instructions with the 13-bit branch offset range. They use the same number of bits in the instruction word (12 stored) but mean different things — branch offsets have an implicit `0` appended for bit 0.

---

### Question F2
**What is the result of `LUI x5, 0` followed by `ADDI x5, x5, -1`? What is x5?**

**Answer:**

```
LUI x5, 0:
  rd = { 0x00000, 12'b0 } = 0x00000000
  x5 = 0

ADDI x5, x5, -1:
  imm = -1 (12-bit) = 0xFFF => sign-extended to 0xFFFFFFFF
  x5 = 0 + 0xFFFFFFFF = 0xFFFFFFFF
```

x5 = `0xFFFFFFFF` = all ones = -1 in two's complement = 4,294,967,295 unsigned.

A simpler way to achieve the same result is `ADDI x5, x0, -1` (a single instruction, since x0 is hardwired zero and -1 fits in 12 bits). The LUI + ADDI pattern is needed only when the value does not fit in a 12-bit immediate.

---

### Question F3
**A B-type branch encodes its offset in 13 bits but only 12 bits are stored in the instruction word. Where is the missing bit, and why is it safe to omit it?**

**Answer:**

The missing bit is **bit 0** of the branch offset. It is not stored because it is always zero.

RISC-V instructions must be at minimum 2-byte aligned (to support the compressed C extension). All instruction addresses therefore have bit 0 equal to 0. Since branch targets are instruction addresses, the offset to a valid target is always a multiple of 2, so bit 0 of any branch offset is always 0 and never needs to be encoded — it is implicitly zero.

The hardware reconstructs the full 13-bit offset by concatenating the stored 12 bits with `1'b0` appended at the bottom:
```
imm = { imm[12], imm[11], imm[10:5], imm[4:1], 1'b0 }
```

By not storing bit 0, the 12 available bits instead encode bits [12:1], which doubles the effective range compared to if bit 0 were stored. This gives a range of ±4 KB rather than ±2 KB.

---

### Question F4
**What happens during sign extension of the 12-bit immediate `0x800`? Why does this matter for ADDI?**

**Answer:**

`0x800` in 12-bit binary is `1000_0000_0000`.

The MSB (bit 11) is `1`, so sign extension fills the upper 20 bits with `1`:
```
12-bit: 1000_0000_0000
32-bit: 1111_1111_1111_1111_1111_1000_0000_0000 = 0xFFFFF800
```

As a signed 32-bit integer this is `-2048`.

**Why it matters for ADDI:** The immediate `0x800` is the most negative value a 12-bit signed immediate can express. If a programmer writes `ADDI x1, x2, 0x800` intending to add 2048, the sign extension converts this to -2048 and the instruction **subtracts** 2048 instead.

This is a particularly insidious bug because:
1. In assembly source, `0x800` looks positive.
2. The assembler accepts it without warning (it fits in 12 bits).
3. The result is wrong by 4096 (the constant is negated).

The correct way to add 2048 is:
```assembly
addi x1, x2, 2047    # x1 = x2 + 2047
addi x1, x1, 1       # x1 = x2 + 2048
```
Or use LUI + ADDI for larger constants.

---

## Tier 2 — Intermediate

### Question I1
**Explain the hardware motivation for placing the sign bit of every immediate at bit 31 of the instruction word. How many gates does this save in a minimal decode implementation?**

**Answer:**

The motivation is to eliminate a mux on the sign-extension path.

Sign extension replicates the MSB of the narrow value into all higher-order bit positions. If the sign bit were at different positions depending on the immediate format, the hardware would require a multiplexer to select the correct source bit before replicating it 20 times (for a 12-bit immediate going to 32 bits).

With all sign bits at `inst[31]`:
```verilog
wire sign = instruction[31];  // one wire, no mux
wire [31:0] imm_i = {{20{sign}}, instruction[31:20]};
```

Without this invariant (hypothetical, sign bit at varying positions):
```verilog
// Three-way mux needed to select the sign bit:
wire sign = (is_i_type) ? instruction[31] :
            (is_s_type) ? instruction[31] :  // also 31 in RISC-V, but imagine if not
            (is_b_type) ? instruction[7]  :  // hypothetical bad encoding
            ...
```

In terms of gate count: a 2-to-1 mux in a standard cell library costs approximately 4-6 equivalent inverter gates (gate equivalents, GE). A 5-to-1 mux on a single bit costs roughly 12-16 GE. Sign-extending to 32 bits requires 20 copies of this mux (one per replicated bit), so removing the mux saves approximately 240-320 GE. For a simple 32-bit core with a total area of 50,000-200,000 GE, this is a measurable saving.

More importantly, the mux adds gate delay to the critical path through the decode stage. Removing it shortens the minimum clock cycle time.

**Interview tip:** The deeper answer is that this invariant was identified during the original ISA design by studying what made MIPS and SPARC decode hardware complex, and deliberately inverting those choices. The cost was accepting some "scrambled" looking immediate bit orderings in B-type and J-type, which confuses humans reading binary but is irrelevant to hardware.

---

### Question I2
**An assembler directive requests `ADDI x1, x2, 0xFFF`. What does this instruction actually compute? Show both the 12-bit encoding and the sign-extended value used in the addition.**

**Answer:**

`0xFFF` in 12-bit binary is `1111_1111_1111`. The MSB (bit 11) is `1`, so this is a **negative** value in 12-bit two's complement:

```
12-bit value: 1111_1111_1111
Interpretation: -(~1111_1111_1111 + 1) = -(0000_0000_0000 + 1) = -1
```

Sign-extended to 32 bits:
```
0xFFFFFFFF = -1 (signed 32-bit)
```

So `ADDI x1, x2, 0xFFF` computes:
```
x1 = x2 + (-1) = x2 - 1
```

The instruction encoding:
```
opcode = 0010011  (OP-IMM)
rd     = 00001    (x1)
funct3 = 000      (ADDI)
rs1    = 00010    (x2)
imm    = 111111111111  (0xFFF)

Binary: 1111_1111_1111_0001_0000_0000_1001_0011
Hex:    0xFFF10093
```

**Common mistake:** Believing `0xFFF` represents +4095. In a 12-bit field, this bit pattern represents -1. The value +4095 cannot be expressed as a 12-bit two's complement immediate (it would require 13 bits). To add 4095, you need two instructions: `ADDI x1, x2, 2047` followed by `ADDI x1, x1, 2048` — except 2048 also cannot fit (it sign-extends to -2048), so in practice you would use `LUI` for larger additions.

---

### Question I3
**Walk through the complete bit scrambling of a J-type immediate. Given `JAL x1, -8` (jump 8 bytes backward), derive the 32-bit instruction encoding step by step.**

**Answer:**

Target offset: `-8`

Step 1 — represent -8 as a 21-bit two's complement value (bit 0 implicit zero):
```
-8 in 21 bits (including implicit bit 0):
  8 = 0b0000_0000_0000_0000_1000 (21 bits, with bit 0 = 0)
 -8 = two's complement of 8 = flip and add 1
  binary of 8: 0_0000_0000_0000_0000_1000
  ~8:          1_1111_1111_1111_1111_0111
  ~8 + 1:      1_1111_1111_1111_1111_1000  => -8

So the 21-bit signed offset = 1_1111_1111_1111_1111_1000
(bit 20 = 1, meaning negative; bits 19:0 = 1111_1111_1111_1111_1000)
```

Step 2 — extract each sub-field from the 21-bit value:
```
Bit positions of the 21-bit immediate:
  imm[20]   = 1        (sign bit)
  imm[19:12]= 11111111
  imm[11]   = 1
  imm[10:1] = 1111111100   (bits 10 down to 1 of the offset:
               offset = ...1111_1111_1000, bits 10:1 = 1111_1111_00 -- careful:
               -8 binary = 1_1111_1111_1111_1111_1000
               bit 1 = 0, bit 2 = 0, bit 3 = 1, bits 4-10 = all 1... let me redo:

-8 = 0xFFFF8 in 20 bits? No. Let us be very precise:

-8 as a signed 32-bit = 0xFFFFFFF8
Lower 21 bits of -8 (since offset must fit in 21-bit signed):
  0xFFFFFFF8 & 0x1FFFFF = 0x1FFFF8

0x1FFFF8 = 0001_1111_1111_1111_1111_1000

bit 20 = 1
bit 19 = 1
...
bit 4 = 1
bit 3 = 1
bit 2 = 0
bit 1 = 0
bit 0 = 0 (implicit, not stored)

imm[20]   = bit 20 = 1
imm[19:12]= bits 19 down to 12 = 1111_1111
imm[11]   = bit 11 = 1
imm[10:1] = bits 10 down to 1 = 1111_1111_00
```

Step 3 — place sub-fields into instruction bit positions:
```
inst[31]   = imm[20]   = 1
inst[30:21]= imm[10:1] = 1111_1111_00
inst[20]   = imm[11]   = 1
inst[19:12]= imm[19:12]= 1111_1111
inst[11:7] = rd = x1   = 00001
inst[6:0]  = opcode    = 1101111

Assembling:
bit 31    : 1
bits 30:21: 1111111100
bit 20    : 1
bits 19:12: 11111111
bits 11:7 : 00001
bits 6:0  : 1101111

32-bit word: 1_1111111100_1_11111111_00001_1101111

Binary: 1111_1111_1001_1111_1111_0000_1110_1111
Hex:    0xFF9FF0EF
```

Step 4 — verify by reconstruction:
```
inst[31]   = 1  => imm[20] = 1   (sign: negative)
inst[7]    is not used in J-type
inst[30:21]= 1111111100 => imm[10:1] = 1111111100
inst[20]   = 1  => imm[11] = 1
inst[19:12]= 11111111 => imm[19:12] = 11111111

Full offset = {imm[20], imm[19:12], imm[11], imm[10:1], 1'b0}
            = {1, 11111111, 1, 1111111100, 0}
            = 1_1111_1111_1111_1111_1000
Sign-extending 21-bit to 32-bit:
  = 1111_1111_1111_1111_1111_1111_1111_1000
  = 0xFFFFFFF8
  = -8 in signed 32-bit  (correct)
```

---

### Question I4
**Why does RISC-V have both B-type (for branches) and J-type (for JAL) rather than using the same encoding for both? What does the extra bit width of J-type buy?**

**Answer:**

Branch instructions (B-type) use a **13-bit** signed offset giving ±4 KB range. JAL (J-type) uses a **21-bit** signed offset giving ±1 MB range. The different ranges reflect different usage patterns:

**Branches (B-type):**
Conditional branches almost always target nearby instructions — the body of an if-statement, a loop, or a short error path. Empirical measurements of real-world binaries show that >99% of conditional branch targets are within ±4 KB of the branch. A 13-bit offset is therefore sufficient for the vast majority of branches. Using a wider encoding would waste bits that could be used for register specifiers or the comparison function.

B-type needs two register specifiers (rs1 and rs2 to compare) plus funct3 (6 branch conditions). After allocating those fields, only 13 bits remain for the offset (including the implicit 0 at bit 0). This is the maximum the format can accommodate.

**JAL (J-type):**
Unconditional jumps are used primarily for function calls. Functions can be located anywhere in the binary, and in a large program (OS kernel, application with many libraries), a 4 KB range would be far too restrictive — most function calls would need multi-instruction sequences. JAL has no comparison registers (it does not compare anything), freeing 10 bits compared to B-type. Those 10 bits are added to the offset, giving 21 bits and ±1 MB range.

**Summary table:**
```
Format  Stored offset bits  Implicit bit  Total range
------  ------------------  ------------  -----------
B-type  12                  bit 0 = 0     ±4,096 bytes
J-type  20                  bit 0 = 0     ±1,048,576 bytes (1 MB)
```

For targets beyond ±1 MB, the compiler uses AUIPC + JALR (a two-instruction sequence), which gives a full 32-bit PC-relative range.

---

## Tier 3 — Advanced

### Question A1
**A processor is implementing a direct-mapped instruction cache with 64-byte cache lines. When a B-type branch is taken to a target that is exactly +4094 bytes away, describe all the activities that occur in the front-end pipeline: fetch, cache lookup, immediate reconstruction, and target calculation. Identify all points where sign extension occurs.**

**Answer:**

This question requires tracing the path of a near-miss branch through the front end.

**Instruction fetch and cache lookup:**
```
Assume branch instruction is at PC = 0x1000.
Target = 0x1000 + 4094 = 0x1FFE.

Fetch stage:
  Cache line size = 64 bytes = 0x40 bytes.
  Cache line containing branch: 0x1000 & ~0x3F = 0x1000 (line-aligned)
  Cache line containing target: 0x1FFE & ~0x3F = 0x1FC0

  These are different cache lines (0x1000 vs 0x1FC0).
  Target line tag = 0x1FC0 >> log2(num_lines) — depends on cache size.
  In a 4 KB direct-mapped cache: index = (0x1FC0 >> 6) & 0x3F = 0x3F (line 63).
  Branch instruction's line: index = (0x1000 >> 6) & 0x3F = 0x40 & 0x3F = 0 (line 0).

  The two addresses map to different cache lines, so the target fetch does not
  evict the branch instruction's line. However, the target line must be present;
  if not, a cache miss occurs at 0x1FC0.
```

**Immediate reconstruction (B-type):**
```
4094 = 0xFFE = 0b1111_1111_1110

Bit layout:
  imm[12]  = 0
  imm[11]  = 1   (bit 11 of 4094: 0xFFE has bit 11 = 1)
  imm[10:5]= 111111
  imm[4:1] = 1110  (bits 4:1 of 0xFFE)
  imm[0]   = 0 (implicit)

In the instruction word:
  inst[31]   = imm[12] = 0
  inst[7]    = imm[11] = 1
  inst[30:25]= imm[10:5] = 111111
  inst[11:8] = imm[4:1] = 1110

Immediate generation hardware:
  sign = inst[31] = 0
  imm = { {19{0}}, 0, 1, 111111, 1110, 0 }
      = { 0000_0000_0000_0000_0000, 0, 1, 111111, 1110, 0 }
      = 0x00000FFE = +4094

Sign extension occurs once, in the immediate generation block. Because sign = 0,
the 19 high bits are zeroed (zero extension of a positive value).
```

**Target address calculation:**
```
Sign extension has already produced imm = 0x00000FFE (32-bit).
Target = PC + imm = 0x1000 + 0x0FFE = 0x1FFE.

This add occurs in the branch target adder (often a dedicated adder in the
execute stage, separate from the main ALU).

No further sign extension occurs in the add — both operands are 32-bit.
```

**Points where sign extension occurs:**
1. Immediate generation: 13 stored bits (12 in instruction + 1 implicit zero) extended to 32 bits. In this case the value is positive so it is zero-filled.
2. Nowhere else — once extended to 32 bits, all subsequent arithmetic uses the full 32-bit representation.

**Additional note on the ±4094 maximum:** The maximum is +4094, not +4095, because the offset must be even (bit 0 = 0) and `4094 = 0xFFE` is the largest even number expressible in a 13-bit signed field. `4096 = 0x1000` would require imm[12] = 1, which would be negative, so it is not a valid positive offset.

---

### Question A2
**A linker is performing relaxation on a RISC-V binary. It finds an AUIPC + ADDI pair that computes a PC-relative address offset of exactly `0x800` (2048). It wants to relax this to a single ADDI if possible. Can it? What constraint does the single-instruction form run into, and what happens at the boundary of this constraint?**

**Answer:**

This question probes deep understanding of 12-bit immediate sign extension interacting with linker optimisations.

**Can the AUIPC + ADDI pair be relaxed to a single ADDI?**

Only if the PC-relative offset fits in a signed 12-bit immediate. The ADDI immediate range is -2048 to +2047. The offset `0x800 = 2048` is **outside this range** (it equals `+2048`, which is one more than the maximum `+2047`). Relaxation to a single ADDI is not possible.

**Why `0x800` specifically fails:**

`0x800` in 12 bits is `1000_0000_0000`. The MSB is 1, so this is interpreted as -2048, not +2048. There is no 12-bit encoding for +2048.

```
+2047 = 0x7FF = 0111_1111_1111  -- fits, MSB = 0
+2048 = 0x800 = 1000_0000_0000  -- does NOT fit; interpreted as -2048
```

**The boundary case and its asymmetry:**

The 12-bit signed immediate range is asymmetric: -2048 to +2047. The positive half has one fewer value than the negative half (due to two's complement). This means:
- An offset of +2047 relaxes to one ADDI.
- An offset of +2048 does **not** relax and requires two instructions.
- An offset of -2048 relaxes to one ADDI (it is the minimum of the range).
- An offset of -2049 does not relax.

**What the linker emits instead:**

For a PC-relative offset of `0x800`:
```assembly
# Cannot relax: keep the two-instruction sequence
auipc a0, 1          # a0 = PC + 0x1000  (upper 20 bits of next power-of-2 above 0x800)
addi  a0, a0, -2048  # a0 = PC + 0x1000 - 0x800 = PC + 0x800
```

Note the compensation: `%hi(0x800)` = 1 (because bit 11 of `0x800` is set), and `%lo(0x800)` = -2048. The linker's `R_RISCV_PCREL_HI20` / `R_RISCV_PCREL_LO12_I` relocation pair handles this correctly via the same `+0x800` rounding that the LUI/ADDI pair uses.

**Implication for linker-relaxation pass:** The RISC-V psABI and linker (ld/lld) implement a relaxation pass that shrinks AUIPC + ADDI pairs to a single ADDI when `abs(offset) <= 2047`. The linker must be careful not to relax pairs where the offset is exactly `-2048 <= offset <= +2047` without also verifying that the resulting single-instruction encoding is bit-exact, since -2048 relaxes but +2048 does not.
