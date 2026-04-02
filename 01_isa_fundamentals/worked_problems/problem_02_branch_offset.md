# Problem 02: Branch Offset Calculation

**Topic:** B-type and J-type PC-relative offset encoding and decoding  
**Difficulty:** Fundamentals (Parts A-B), Intermediate (Parts C-D), Advanced (Parts E-F)  
**Prerequisites:** `instruction_formats.md`, `immediate_encoding.md`, `rv32i_base_instructions.md`

---

## Problem Statement

### Part A (Fundamentals)

The following code sequence is placed in memory starting at address `0x0400`. Calculate the binary encoding of the BEQ instruction, paying careful attention to the PC-relative offset.

```assembly
0x0400:  addi  t0, zero, 5      # t0 = 5
0x0404:  addi  t1, zero, 5      # t1 = 5
0x0408:  beq   t0, t1, equal    # branch if t0 == t1
0x040C:  addi  a0, zero, 0      # a0 = 0 (not-equal path)
0x0410:  jal   zero, done
equal:
0x0414:  addi  a0, zero, 1      # a0 = 1 (equal path)
done:
0x0418:  ...
```

What is the branch offset? Give the full 32-bit encoding of the BEQ instruction in hex.

---

### Part B (Fundamentals)

A BEQ instruction at address `0x2000` has the hex encoding `0xFE208EE3`.

1. What is the branch offset?
2. What is the branch target address?
3. Is the branch forward or backward?

---

### Part C (Intermediate)

The following loop is placed starting at `0x5000`. Derive the hex encoding of the BNEZ instruction.

```assembly
0x5000:  lw    a1, 0(a0)       # load current element
0x5004:  add   s0, s0, a1      # accumulate sum
0x5008:  addi  a0, a0, 4       # advance pointer
0x500C:  addi  t0, t0, -1      # decrement counter
0x5010:  bnez  t0, loop        # loop: branch back to 0x5000 if t0 != 0
0x5014:  ...
```

The label `loop` is at `0x5000`. Show the complete B-type encoding of `BNEZ t0, loop`.

---

### Part D (Intermediate)

A JAL instruction at address `0x3000` must call a function at `0x4200`. 

1. Calculate the J-type immediate (the offset to encode).
2. Show the complete bit-scrambling of the J-type immediate.
3. Give the final 32-bit hex encoding. Assume the return address is saved in `ra` (x1).

---

### Part E (Advanced)

A compiler generates the following branch sequence to implement `if (x != y) { z = 1; }` where x is in `a0`, y is in `a1`, and z is in `a2`. The function starts at `0x8000` and the branch instruction is at `0x8010`.

```assembly
0x8010:  beq   a0, a1, skip
0x8014:  addi  a2, zero, 1
skip:
0x8018:  ...
```

Now suppose the "true" block is expanded to 8192 bytes of code (8 KB), so `skip` would be at `0x8010 + 4 + 8192 = 0xA014`. This is beyond the BEQ range.

Show the complete branch sequence a compiler would emit to handle this far branch, using the minimum number of instructions.

---

### Part F (Advanced)

Given the hex encoding `0xF61FF06F` at address `0x1080`, fully decode the instruction, recover the jump offset, and compute the absolute target address. Verify your answer by re-encoding and checking the hex.

---

## Solutions

### Part A Solution

**BEQ t0, t1, equal** at address `0x0408`.

Target label `equal` is at `0x0414`.

Step 1 — compute PC-relative offset:
```
offset = target - PC_of_branch
       = 0x0414 - 0x0408
       = 0x000C
       = 12 (decimal)
```

Step 2 — verify offset is in range and even:
```
12 is even (bit 0 = 0): valid branch offset
12 is within ±4094 bytes: valid B-type range
```

Step 3 — represent 12 as a 13-bit signed value and extract sub-fields:
```
12 = 0b0000_0000_1100 (12 bits stored, bit 0 implicit 0)

Bit layout of 12:
  bit 0  = 0  (implicit)
  bit 1  = 0
  bit 2  = 1
  bit 3  = 1
  bit 4  = 0
  bit 5  = 0
  ...
  bit 12 = 0  (sign bit)

imm[12]  = 0
imm[11]  = 0
imm[10:5]= 000000
imm[4:1] = 0110  (bits 4:1 of 12: bit4=0, bit3=1, bit2=1, bit1=0 => 0110)
```

Step 4 — identify register encodings:
```
BEQ:   funct3 = 000, opcode = 1100011
t0 = x5 = 00101  (rs1)
t1 = x6 = 00110  (rs2)
```

Step 5 — assemble the 32-bit instruction:
```
inst[31]   = imm[12] = 0
inst[30:25]= imm[10:5] = 000000
inst[24:20]= rs2 = 00110  (t1)
inst[19:15]= rs1 = 00101  (t0)
inst[14:12]= funct3 = 000  (BEQ)
inst[11:8] = imm[4:1] = 0110
inst[7]    = imm[11] = 0
inst[6:0]  = opcode = 1100011

Binary: 0_000000_00110_00101_000_0110_0_1100011
= 0000 0000 0110 0010 1000 0110 0110 0011
= 0x00628663
```

Verification by tracing execution:
```
Instruction at 0x0408 = 0x00628663
When t0 == t1 (both = 5):
  PC = 0x0408 + 12 = 0x0414  => jumps to `equal` label  (correct)
When t0 != t1:
  PC = 0x0408 + 4 = 0x040C   => falls through to the not-equal path  (correct)
```

---

### Part B Solution

**Input:** `0xFE208EE3` at address `0x2000`

Step 1 — binary:
```
0xFE208EE3 = 1111 1110 0010 0000 1000 1110 1110 0011
```

Step 2 — opcode and format:
```
bits [6:0] = 110 0011 => BRANCH (B-type) confirmed
bits [14:12]= 000 => funct3 = 0 => BEQ
bits [19:15]= 00001 = 1 => rs1 = x1
bits [24:20]= 00010 = 2 => rs2 = x2
```

Step 3 — extract B-type immediate:
```
inst[31]   = 1  => imm[12] = 1  (sign bit: negative offset)
inst[7]    = 1  => imm[11] = 1
inst[30:25]= 111111 => imm[10:5] = 111111
inst[11:8] = 1110   => imm[4:1] = 1110

Reconstruct:
imm = {imm[12], imm[11], imm[10:5], imm[4:1], 1'b0}
    = {1, 1, 111111, 1110, 0}

Bit positions:
  bit 12 = 1
  bit 11 = 1
  bits 10:5 = 111111
  bits 4:1  = 1110
  bit 0     = 0

Full 13-bit value:
  1_1111_1111_1100  (reading from bit 12 down to bit 0)

  = 1 1111 1111 1100 (binary)

Sign-extend to 32 bits (sign bit = 1):
  0xFFFF_FFFC  minus...

  Actually: interpret as signed 13-bit:
  Magnitude = ~(1111111111100) + 1 = 0000000000011 + 1 = 0000000000100 = 4
  So value = -4? Let me double-check:

  The 13-bit value 1_1111_1111_1100:
  As unsigned: 0x1FFC = 8188
  As signed 13-bit: 8188 - 8192 = -4

  So offset = -4.
```

Step 4 — compute branch target:
```
target = PC + offset = 0x2000 + (-4) = 0x1FFC
```

Step 5 — answer:
1. Branch offset: **-4 bytes**
2. Branch target address: **0x1FFC**
3. The branch is **backward** (target < PC)

This pattern — `BEQ ra, sp, -4` (branches to 4 bytes before itself) — is unusual and in normal code would be a tight 2-instruction infinite loop or an error. In this case the instruction is a loop-back to an instruction 4 bytes before the branch itself (the instruction immediately before it).

---

### Part C Solution

**Target:** `BNEZ t0, loop` at address `0x5010`, target `0x5000`

Note: `BNEZ t0, loop` is a pseudo-instruction. The real instruction is:
```assembly
BNE t0, zero, loop     # funct3 = 001, rs2 = x0
```

Step 1 — compute offset:
```
offset = target - PC = 0x5000 - 0x5010 = -16
```

Step 2 — represent -16 as 13-bit signed:
```
16 = 0b0000_0001_0000  (13 bits including implicit bit 0)
-16 in 13-bit two's complement:
  ~16:  1111_1110_1111
  +1:   1111_1111_0000  (this is still only 12 stored bits; the sign bit at bit 12 = 1)

Let me work in 13 bits explicitly:
  +16  = 0_0000_0001_0000   (13 bits: bit4=1, rest 0, bit0=0)
  flip:  1_1111_1110_1111
  add 1: 1_1111_1111_0000

-16 bit layout (13-bit):
  bit 12 = 1 (sign)
  bit 11 = 1
  bit 10 = 1
  bit  9 = 1
  bit  8 = 1
  bit  7 = 1
  bit  6 = 1
  bit  5 = 1
  bit  4 = 1
  bit  3 = 0
  bit  2 = 0
  bit  1 = 0
  bit  0 = 0 (implicit)
```

Step 3 — extract B-type sub-fields from -16:
```
imm[12]  = bit 12 = 1
imm[11]  = bit 11 = 1
imm[10:5]= bits 10:5 = 111111
imm[4:1] = bits 4:1  = 1000  (bit4=1, bit3=0, bit2=0, bit1=0)
```

Step 4 — register encodings:
```
BNE: funct3 = 001, opcode = 1100011
t0  = x5 = 00101  (rs1)
x0  = x0 = 00000  (rs2)
```

Step 5 — assemble:
```
inst[31]   = imm[12] = 1
inst[30:25]= imm[10:5] = 111111
inst[24:20]= rs2 = 00000  (x0)
inst[19:15]= rs1 = 00101  (t0)
inst[14:12]= funct3 = 001  (BNE)
inst[11:8] = imm[4:1] = 1000
inst[7]    = imm[11] = 1
inst[6:0]  = opcode = 1100011

Binary: 1_111111_00000_00101_001_1000_1_1100011
= 1111 1110 0000 0010 1001 1000 1110 0011
= 0xFE029 8E3
```

Let me carefully re-lay out the bits:
```
bit 31:    1
bits 30:25: 111111
bits 24:20: 00000
bits 19:15: 00101
bits 14:12: 001
bits 11:8:  1000
bit 7:      1
bits 6:0:   1100011

Full 32-bit:
  31   30-25   24-20  19-15  14-12  11-8  7  6-0
  1   111111  00000  00101   001   1000  1  1100011

Binary grouping (4 bits each from MSB):
1111 | 1110 | 0000 | 0010 | 1001 | 1000 | 1110 | 0011
= 0xFE0298E3
```

Verification:
```
Reconstruct from 0xFE0298E3:
  Binary: 1111 1110 0000 0010 1001 1000 1110 0011
  inst[31]   = 1   => imm[12] = 1
  inst[7]    = 1   => imm[11] = 1
  inst[30:25]= 111111 => imm[10:5] = 111111
  inst[11:8] = 1000   => imm[4:1] = 1000
  imm = {1,1,111111,1000,0}
      13-bit: 1_1111_1111_0000
      signed: 8176 - 8192 = -16  (correct)
  target = 0x5010 - 16 = 0x5000  (correct, = loop label)
```

**Final answer:** `0xFE0298E3`

---

### Part D Solution

**JAL ra, 0x4200** (target) from address **0x3000** (JAL instruction address).

Step 1 — compute offset:
```
offset = target - PC = 0x4200 - 0x3000 = 0x1200 = 4608 (decimal)
```

Step 2 — verify J-type range:
```
4608 is within ±1,048,574: valid
4608 is even (bit 0 = 0): valid
```

Step 3 — represent 4608 as 21-bit signed value and extract fields:
```
4608 = 0x1200 = 0001_0010_0000_0000 (16 bits)
In 21 bits: 0_0000_0001_0010_0000_0000

Bit layout of 4608:
  bit 0  = 0 (implicit)
  bit 1  = 0
  bit 2  = 0
  bit 3  = 0
  bit 4  = 0
  bit 5  = 0
  bit 6  = 0
  bit 7  = 0
  bit 8  = 0
  bit 9  = 1  (512)
  bit 10 = 0
  bit 11 = 0
  bit 12 = 1  (4096)
  bit 13 = 0
  ...
  bit 20 = 0  (sign bit, positive)

Verify: 2^9 + 2^12 = 512 + 4096 = 4608  (correct)

Extract J-type sub-fields:
  imm[20]   = bit 20 = 0
  imm[19:12]= bits 19:12 = 00000001  (bit12=1, rest 0)
  imm[11]   = bit 11 = 0
  imm[10:1] = bits 10:1  = 0100000000 (bit9=1, rest 0)
```

Step 4 — place fields into J-type instruction word:
```
inst[31]   = imm[20]   = 0
inst[30:21]= imm[10:1] = 0100000000
inst[20]   = imm[11]   = 0
inst[19:12]= imm[19:12]= 00000001
inst[11:7] = rd = ra = x1 = 00001
inst[6:0]  = opcode = 1101111  (JAL)

Bit layout:
bit 31:    0
bits 30:21: 0100000000
bit 20:    0
bits 19:12: 00000001
bits 11:7:  00001
bits 6:0:   1101111

Full 32-bit:
  31   30-21       20  19-12     11-7    6-0
  0   0100000000   0  00000001  00001  1101111

Binary (4-bit groups):
  0010 | 0000 | 0000 | 0000 | 0001 | 0000 | 1110 | 1111

= 0x200010EF
```

Verification:
```
Reconstruct from 0x200010EF:
  Binary: 0010 0000 0000 0000 0001 0000 1110 1111
  inst[31]   = 0   => imm[20] = 0  (positive)
  inst[30:21]= 0100000000 => imm[10:1] = 0100000000 (bit9=1)
  inst[20]   = 0   => imm[11] = 0
  inst[19:12]= 00000001 => imm[19:12] (bit12=1)

  Full offset = {imm[20], imm[19:12], imm[11], imm[10:1], 1'b0}
  = {0, 00000001, 0, 0100000000, 0}
  = 0_0000_0001_0_0100000000_0
  Bit 12 = 1 (from imm[19:12] bit 0), bit 9 = 1 (from imm[10:1] bit 8)
  = 0b0000_0001_0010_0000_0000 = 0x1200 = 4608  (correct)

  target = 0x3000 + 4608 = 0x3000 + 0x1200 = 0x4200  (correct)
```

**Final answer:** `0x200010EF`

---

### Part E Solution

**Far branch problem:** BEQ a0, a1, skip where skip is at `0x8010 + 4 + 8192 = 0xA014`.

Offset to skip: `0xA014 - 0x8010 = 0x2004 = 8196 bytes`.

8196 > 4094 (maximum positive B-type offset). Direct BEQ is impossible.

**Compiler solution — inverted branch over a JAL:**

```assembly
0x8010:  bne   a0, a1, skip_near    # branch over the jump if a0 != a1
0x8014:  jal   zero, skip           # unconditional jump to far skip target
skip_near:
0x8018:  ...  (the 8 KB true block starts here)
...
0xA014:  ...  (skip label, where execution resumes after the if)
```

Note: `skip_near` is not `skip`. The condition is inverted:
- Original: `if (a0 == a1) goto skip` (skip the true block)
- Inverted: `if (a0 != a1) goto skip_near` (if not-taken, skip_near is the first instruction of the true block, only 4 bytes away from the BNE)
- The JAL at `0x8014` jumps to `skip` (the far target)

Check JAL range:
```
offset = 0xA014 - 0x8014 = 0x2000 = 8192 bytes
8192 < 1,048,574 (J-type max): JAL is sufficient
```

**Encoding of the BNE (at 0x8010):**

`BNE a0, a1, skip_near` where skip_near = `0x8018`:
```
offset = 0x8018 - 0x8010 = 8 bytes
imm[4:1] = 0100 (8>>1 = 4, in 4 bits = 0100)
all other imm bits = 0

BNE: funct3=001, opcode=1100011
a0 = x10 = 01010 (rs1)
a1 = x11 = 01011 (rs2)

inst[31]   = 0, inst[7]=0, inst[30:25]=000000
inst[11:8] = 0100, inst[24:20]=01011, inst[19:15]=01010
inst[14:12]=001, inst[6:0]=1100011

Binary: 0000000_01011_01010_001_01000_1100011
= 0x00B51463
```

**Encoding of the JAL (at 0x8014):**

`JAL x0, skip` where skip = `0xA014`:
```
offset = 0xA014 - 0x8014 = 0x2000 = 8192

8192 = 0x2000 = 0b0010_0000_0000_0000
  bit 13 = 1 (8192 = 2^13)
  others = 0

8192 = 0x2000, so bit 13 of the offset = 1, all others 0.

imm[20]   = bit20 = 0
imm[19:12]= bits 19 down to 12 = {bit19, bit18, ..., bit13, bit12}
           = {0, 0, 0, 0, 0, 0, 1, 0}
           = 00000010
imm[11]   = bit11 = 0
  imm[10:1] = bits 10:1 = 0000000000

inst[31]   = imm[20]   = 0
inst[30:21]= imm[10:1] = 0000000000
inst[20]   = imm[11]   = 0
inst[19:12]= imm[19:12]= 00100000
inst[11:7] = rd = x0   = 00000
inst[6:0]  = opcode    = 1101111

Binary:
bit 31:    0
bits 30:21: 0000000000
bit 20:    0
bits 19:12: 00100000
bits 11:7:  00000
bits 6:0:   1101111

= 0000 0000 0000 0010 0000 0000 0110 1111
= 0x0020006F
```

**Complete emitted sequence:**
```assembly
0x8010:  0x00B51463    # BNE a0, a1, +8  (skip_near at 0x8018)
0x8014:  0x0020006F    # JAL x0, +8192   (jump to skip at 0xA014)
0x8018:  ...           # start of 8 KB true block
...
0xA014:  ...           # skip (if-join point)
```

**This uses only 2 instructions total on the branch critical path** (1 BNE + 1 JAL), which is optimal. The BNE is taken in the common case (a0 != a1, skip the true block), so execution jumps directly from 0x8010 to 0x8018 — the JAL at 0x8014 is only reached when a0 == a1 (the branch-to-skip case).

---

### Part F Solution

**Input:** `0xF61FF06F` at address `0x1080`

Step 1 — binary:
```
0xF61FF06F = 1111 0110 0001 1111 1111 0000 0110 1111
```

Step 2 — opcode:
```
bits [6:0] = 110 1111 = 0x6F
opcode 1101111 => JAL (J-type)
```

Step 3 — extract rd:
```
bits [11:7] = 0 0000 = 0 => rd = x0 (link discarded)
```
This is a plain unconditional jump (`J` pseudo-instruction), not a subroutine call (which would use rd = x1 = ra).

Step 4 — extract J-type immediate bits:
```
inst[31]   = 1   => imm[20] = 1  (sign bit: negative offset)

Re-extracting from binary:
0xF61FF06F = 1111_0110_0001_1111_1111_0000_0110_1111

Bit assignments:
  bit 31:    1
  bits 30:21: 111_0110_000  -- read bits 30 down to 21:
              bit30=1, bit29=1, bit28=1, bit27=0, bit26=1, bit25=1, bit24=0,
              bit23=0, bit22=0, bit21=0
              => imm[10:1] = 1110110000

  bit 20:    1 => imm[11] = 1
  bits 19:12: 1111_1111 = 0xFF => imm[19:12] = 11111111
  bits 11:7:  0 0000 = 0 => rd = x0
  bits 6:0:   110 1111 => JAL opcode confirmed

Let me re-read the binary carefully:
1111 0110 0001 1111 1111 0000 0110 1111

Position: 31 30 29 28 27 26 25 24 | 23 22 21 20 19 18 17 16 | 15 14 13 12 11 10 9 8 | 7 6 5 4 3 2 1 0
Value:      1  1  1  1  0  1  1  0 |  0  0  0  1  1  1  1  1 |  1  1  1  1  0  0  0 0 | 0  1  1  0  1  1  1  1

bits[31]   = 1            => imm[20] = 1
bits[30:21]= 1 1 0 1 1 0 0 0 0 1  (reading 30 down to 21)
             bit30=1, bit29=1, bit28=0, bit27=1, bit26=1, bit25=0, bit24=0, bit23=0, bit22=0, bit21=1
             => imm[10:1] = 1101100001

bits[20]   = 1            => imm[11] = 1
bits[19:12]= 1 1 1 1 1 1 1 1 (reading 19 down to 12)
             bit19=1,18=1,17=1,16=1,15=1,14=1,13=1,12=1
             => imm[19:12] = 11111111

bits[11:7] = 0 0000 = 0  => rd = x0
bits[6:0]  = 1101111     => JAL (confirmed)
```

Step 5 — reconstruct the 21-bit signed offset:
```
imm = {imm[20], imm[19:12], imm[11], imm[10:1], 1'b0}
    = {1, 11111111, 1, 1101100001, 0}

Bit positions:
  bit 20 = 1
  bits 19:12 = 11111111
  bit 11 = 1
  bits 10:1 = 1101100001
  bit 0 = 0

Full 21-bit value:
  1_1111_1111_1_1101100001_0

Let me write as one number:
  bit20=1, bits19-12=11111111, bit11=1, bits10-1=1101100001, bit0=0

  = 1 11111111 1 1101100001 0
  
  Mapping to bit positions 20 down to 0:
  20:1, 19:1, 18:1, 17:1, 16:1, 15:1, 14:1, 13:1, 12:1, 11:1, 10:1, 9:1, 8:0, 7:1, 6:1, 5:0, 4:0, 3:0, 2:0, 1:1, 0:0

  As binary: 1_1111_1111_1110_1100_0010
  = 0x1FFEC2 (21-bit unsigned)

  As signed 21-bit: 0x1FFEC2 - 0x200000 = -0x13E = -318

Hmm, that doesn't look clean. Let me reconsider by re-reading the hex more carefully.

0xF61FF06F:
F = 1111
6 = 0110
1 = 0001
F = 1111
F = 1111
0 = 0000
6 = 0110
F = 1111

So: 1111 0110 0001 1111 1111 0000 0110 1111

Annotated:
bits [31:28] = 1111
bits [27:24] = 0110
bits [23:20] = 0001
bits [19:16] = 1111
bits [15:12] = 1111
bits [11:8]  = 0000
bits [7:4]   = 0110
bits [3:0]   = 1111

So individual bits:
31=1, 30=1, 29=1, 28=1, 27=0, 26=1, 25=1, 24=0, 23=0, 22=0, 21=0, 20=1,
19=1, 18=1, 17=1, 16=1, 15=1, 14=1, 13=1, 12=1, 11=0, 10=0, 9=0, 8=0,
7=0, 6=1, 5=1, 4=0, 3=1, 2=1, 1=1, 0=1

J-type extraction:
  inst[31]   = 1         => imm[20] = 1  (negative)
  inst[30:21]: bits 30,29,28,27,26,25,24,23,22,21
             = 1,1,1,0,1,1,0,0,0,0
             => imm[10:1] = 1110110000

  inst[20]   = 1         => imm[11] = 1

  inst[19:12]: bits 19,18,17,16,15,14,13,12
             = 1,1,1,1,1,1,1,1
             => imm[19:12] = 11111111

  inst[11:7]:  bits 11,10,9,8,7 = 0,0,0,0,0 => rd = x0

  inst[6:0]:   bits 6,5,4,3,2,1,0 = 1,1,0,1,1,1,1 = 1101111 => JAL (confirmed)
```

Step 6 — reconstruct and sign-extend:
```
imm = {imm[20], imm[19:12], imm[11], imm[10:1], 1'b0}
    = {1, 11111111, 1, 1110110000, 0}

21-bit value (writing out all bits, position 20 down to 0):
  pos: 20 19 18 17 16 15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
  val:  1  1  1  1  1  1  1  1  1  1  1  1  1  0  1  1  0  0  0  0  0

  = 1_1111_1111_1111_1011_0000_0 (but this is 21 bits with bit0)

Let me write as hex:
  bits 20:0 = 1 11111111 1 1110110000 0
  
  Bit 20 = 1 (MSB)
  Bits 19:12 = 1111 1111
  Bit 11 = 1
  Bits 10:1 = 1110 1100 00
  Bit 0 = 0

Full value:
  bits 10:1 = 1110110000 (bit10=1,9=1,8=1,7=0,6=1,5=1,4=0,3=0,2=0,1=0)

  All 21 bits from 20 to 0:
  1,1,1,1,1,1,1,1,1,1,1,1,1,0,1,1,0,0,0,0,0

  = 1_1111_1111_1111_0110_0000 (grouping 20:0 in 4-bit groups plus leading bit)
  
  Computing the value:
  imm[20]=1, magnitude bits:
    imm[19:12] = 11111111 = contributes to magnitude
    imm[11] = 1
    imm[10:1] = 1110110000

  As unsigned 21-bit:
  We have {1, 11111111, 1, 1110110000, 0}
  
  Position 20: 1 * 2^20 = 1048576
  Positions 19:12 all 1: sum = 2^19+2^18+...+2^12 = 2^20 - 2^12 = 1048576 - 4096 = 1044480
  Position 11: 1 * 2^11 = 2048
  Positions 10:1 = 1110110000 (MSB to LSB):
    bit10=1: 1024
    bit9=1:  512
    bit8=1:  256
    bit7=0:  0
    bit6=1:  64
    bit5=1:  32
    bit4=0:  0
    bit3=0:  0
    bit2=0:  0
    bit1=0:  0
    Total: 1024+512+256+64+32 = 1888
  Position 0: 0

  Total unsigned = 1048576 + 1044480 + 2048 + 1888 = ?
  That's more than 2^21 - 1, which is impossible.

  I'm making an error. Let me restart the reconstruction cleanly.

  21-bit value with each bit:
  bit20=imm[20]=1
  bit19=imm[19]=1 (from imm[19:12]=11111111, MSB=bit19)
  bit18=imm[18]=1
  bit17=imm[17]=1
  bit16=imm[16]=1
  bit15=imm[15]=1
  bit14=imm[14]=1
  bit13=imm[13]=1
  bit12=imm[12]=1 (from imm[19:12]=11111111, LSB=bit12)
  bit11=imm[11]=1
  bit10=imm[10]=1 (from imm[10:1]=1110110000, MSB=bit10)
  bit9=imm[9]=1
  bit8=imm[8]=1
  bit7=imm[7]=0
  bit6=imm[6]=1
  bit5=imm[5]=1
  bit4=imm[4]=0
  bit3=imm[3]=0
  bit2=imm[2]=0
  bit1=imm[1]=0
  bit0=0 (implicit)

  21-bit binary: 1_1111_1111_1111_0110_0000

  imm[10:1] = 1110110000 (imm[10]=1, imm[9]=1, imm[8]=1, imm[7]=0, imm[6]=1, imm[5]=1, imm[4:1]=0000)

  Grouping from bit20 to bit0 in groups of 4:
  bits 20-17: 1111 = 0xF
  bits 16-13: 1111 = 0xF
  bits 12-9:  1111 = 0xF (bit12=1, bit11=1, bit10=1, bit9=1)
  bits 8-5:   1011 = 0xB (bit8=1, bit7=0, bit6=1, bit5=1)
  bits 4-1:   0000 = 0x0
  bit 0:      0

  As unsigned 21-bit:
  = 2^20 + 2^19 + 2^18 + 2^17 + 2^16 + 2^15 + 2^14 + 2^13 + 2^12 + 2^11 + 2^10 + 2^9 + 2^8 + 2^6 + 2^5
  = (2^21 - 1) - (2^7 + 2^4 + 2^3 + 2^2 + 2^1)  [all bits set minus the ones that are 0]
  Actually easier: bits that are 0 are: bit7, bit4, bit3, bit2, bit1, bit0
  All-ones 21-bit = 2^21 - 1 = 2097151
  Subtract the zero bits:
    bit7 = 128
    bit4 = 16
    bit3 = 8
    bit2 = 4
    bit1 = 2
    bit0 = 1
    Total zeros contribution = 159

  Unsigned 21-bit value = 2097151 - 159 = 2096992

  As signed 21-bit (MSB=1, negative):
  signed = unsigned - 2^21 = 2096992 - 2097152 = -160

So the offset is **-160 bytes**.
```

Step 7 — compute target address:
```
target = PC + offset = 0x1080 + (-160) = 0x1080 - 0xA0 = 0x0FE0
```

Step 8 — verify by re-encoding `JAL x0, -160` at `0x1080`:
```
offset = -160
-160 in 21-bit two's complement:
  160 = 128 + 32 = 2^7 + 2^5 (21-bit: bits 7 and 5 are set, all others 0)
  -160 in binary:
  -160 = 0xFFFFFF60 (32-bit)
  Lower 21 bits: 0xFFFFFF60 & 0x1FFFFF = 0x1FFFFC0?

  0xFFFFFF60:
  F=1111, F=1111, F=1111, F=1111, F=1111, F=1111, 6=0110, 0=0000
  Lower 21 bits (bits 20:0):
  bit20=1, ...down to bit8 all 1, bit7=1, bit6=0, bit5=1, bit4=1, bits3:0=0000

  Actually: 0xFF60 in 16 bits = 1111_1111_0110_0000
  0x1FF60 wouldn't be right either. Let me compute properly:
  -160 mod 2^21:
  2^21 - 160 = 2097152 - 160 = 2096992

  2096992 in binary:
  2096992 = 0x1FFFE0? Let me check: 0x1FFFE0 = 2097120, not 2096992
  2096992 = 2097152 - 160 = 0x200000 - 0xA0 = 0x1FFF60

  0x1FFF60 in binary (21 bits):
  = 1_1111_1111_1111_0110_0000

  0x1FFF60 in binary: 0x60 = 0110_0000, so bits 7:0 = 0110_0000
  (confirming -0xA0 = 0x60 in 8-bit two's complement, sign-extended to 21 bits)

  bit20=1, bit19=1,...bit8=1 (all ones from bit20 to bit8)
  bit7=0, bit6=1, bit5=1, bit4=0, bit3=0, bit2=0, bit1=0, bit0=0

imm[20:0] = 1_1111_1111_1111_0110_0000

Extract J-type sub-fields:
  imm[20]   = bit20 = 1
  imm[19:12]= bits 19:12 = 1111_1111
  imm[11]   = bit11 = 1
  imm[10:1] = bits 10:1  = 1110110000

  The 21-bit value from MSB to LSB (bit 20 to bit 0):
  1,1,1,1,1,1,1,1,1,1,1,1,1,0,1,1,0,0,0,0,0

  bit20=1, bit19=1, bit18=1, bit17=1, bit16=1
  bit15=1, bit14=1, bit13=1, bit12=1, bit11=1
  bit10=1, bit9=1, bit8=1
  bit7=0
  bit6=1
  bit5=1
  bit4=0
  bit3=0
  bit2=0
  bit1=0
  bit0=0 (implicit)

  imm[10:1] = {bit10,bit9,bit8,bit7,bit6,bit5,bit4,bit3,bit2,bit1}
            = {1,1,1,0,1,1,0,0,0,0}
            = 1110110000

This matches what I extracted from the encoding earlier (inst[30:21] = 1110110000 => imm[10:1] = 1110110000). The encoding is self-consistent.

The offset is -160 bytes. Let us verify:
  PC = 0x1080
  target = 0x1080 - 160 = 0x1080 - 0xA0 = 0x0FE0
```

**Final answers for Part F:**
- Instruction: `JAL x0, -160` (unconditional backward jump, no link)
- Jump offset: **-160 bytes**
- Target address: **0x0FE0**
- The re-encoding of `JAL x0, -160` at `0x1080` should yield `0xF61FF06F` (matches the input, confirming the decode is correct)

---

## Discussion

### Key Insights for Branch/Jump Offset Problems

**1. Always compute offset as `target - PC_of_instruction`, not `target - (PC_of_instruction + 4)`.**

RISC-V branch/jump offsets are added to the PC of the branch instruction itself (the address of the branch). The PC+4 form is used by some other architectures (ARM Thumb) and is a frequent source of off-by-4 errors.

**2. The implicit `0` for bit 0 doubles the effective range.**

A B-type instruction stores 12 bits but encodes a 13-bit signed offset by knowing bit 0 is always 0. This doubles the range: the 12 stored bits represent `offset / 2`. When computing offsets, work in bytes, not in units of 2 bytes — just be aware that the encoded value will have bit 0 = 0.

**3. Negative offsets and backward branches are encoded as large unsigned values.**

A branch offset of -4 (4 bytes backward) encodes as `0x1FFC` in 13-bit form, which looks large. The decode/decode tests for the MSB (bit 12 for B-type, bit 20 for J-type) to determine sign.

**4. For far branches: always use inverted branch + JAL, not just JAL.**

A lone JAL cannot reproduce a conditional branch. The pattern is always:
1. Invert the condition (BEQ becomes BNE, BLT becomes BGE, etc.)
2. Branch over the JAL with the inverted condition (offset = +8 for a 2-instruction skip)
3. JAL to the far target

**5. Verify B-type encodings by checking both scrambled bits: `imm[11]` at `inst[7]` and `imm[12]` at `inst[31]`.**

These are the most commonly confused bits. After encoding, always reconstruct and verify the offset equals the original.
