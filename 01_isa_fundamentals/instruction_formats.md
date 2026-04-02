# Instruction Formats

## Prerequisites
- Binary and hexadecimal number systems
- Basic computer architecture: registers, memory, program counter
- Two's complement signed integer representation

---

## Concept Reference

### The 32-bit Fixed-Width Philosophy

Every standard RV32I instruction is exactly 32 bits wide. This is a deliberate design choice that simplifies:
- **Decode hardware:** A single 32-bit fetch is always one instruction. No variable-length prefixes to track.
- **Alignment:** Instructions must be 4-byte aligned (bits [1:0] are always `11` for 32-bit instructions, reserved for the C extension which uses `00`, `01`, `10` in bits [1:0] to indicate 16-bit encodings).
- **Pipeline design:** Instruction fetch, decode, and issue stages all operate on a fixed-width unit.

The ISA spec reserves bits [1:0] as the instruction width indicator:
```
bits[1:0] = 11  =>  32-bit instruction (all standard RV32I/RV64I/RV128I)
bits[1:0] = 00  =>  16-bit (C extension, quadrant 0)
bits[1:0] = 01  =>  16-bit (C extension, quadrant 1)
bits[1:0] = 10  =>  16-bit (C extension, quadrant 2)
```

### The Six Instruction Formats

RISC-V defines six base instruction formats. All share the same opcode field location (bits [6:0]) and, critically, the same register field positions wherever registers appear.

```
Bit positions:  31       25 24   20 19   15 14  12 11      7 6       0
                +---------+-------+-------+------+---------+---------+
R-type          | funct7  |  rs2  |  rs1  |funct3|   rd    | opcode  |
                +---------+-------+-------+------+---------+---------+
I-type          |   imm[11:0]     |  rs1  |funct3|   rd    | opcode  |
                +-----------------+-------+------+---------+---------+
S-type          |imm[11:5]|  rs2  |  rs1  |funct3|imm[4:0] | opcode  |
                +---------+-------+-------+------+---------+---------+
B-type          |i[12|10:5]| rs2  |  rs1  |funct3|i[4:1|11]| opcode  |
                +---------+-------+-------+------+---------+---------+
U-type          |         imm[31:12]             |   rd    | opcode  |
                +--------------------------------+---------+---------+
J-type          |     imm[20|10:1|11|19:12]      |   rd    | opcode  |
                +--------------------------------+---------+---------+
```

### R-type (Register-Register)

Used for arithmetic, logical, and shift operations where both operands come from registers.

```
 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
|  funct7  |  rs2  |  rs1  |funct3|   rd   |  opcode |
|  [31:25] | [24:20]|[19:15]|[14:12]|[11:7] |  [6:0]  |
+----------+-------+-------+------+--------+---------+
   7 bits    5 bits  5 bits  3 bits  5 bits    7 bits

Field meanings:
  opcode  [6:0]   : operation class (e.g., 0110011 = OP, integer register-register)
  rd      [11:7]  : destination register
  funct3  [14:12] : sub-operation selector
  rs1     [19:15] : source register 1
  rs2     [24:20] : source register 2
  funct7  [31:25] : further sub-operation selector (e.g., distinguishes ADD from SUB)

Example: ADD x3, x1, x2
  opcode = 0110011  (OP)
  rd     = 00011    (x3)
  funct3 = 000      (ADD/SUB)
  rs1    = 00001    (x1)
  rs2    = 00010    (x2)
  funct7 = 0000000  (ADD; 0100000 would be SUB)

Binary: 0000000 00010 00001 000 00011 0110011
Hex:    0x00208133
```

### I-type (Immediate)

Used for arithmetic/logical operations with a 12-bit immediate, loads, JALR, CSR instructions, and FENCE.

```
 31          20  19   15  14  12  11    7   6      0
+--------------+-------+------+--------+---------+
|   imm[11:0]  |  rs1  |funct3|   rd   |  opcode |
|   [31:20]    |[19:15]|[14:12]|[11:7] |  [6:0]  |
+--------------+-------+------+--------+---------+
    12 bits      5 bits  3 bits  5 bits    7 bits

The 12-bit immediate is sign-extended to 32 bits before use.
Range of immediate: -2048 to +2047 (signed 12-bit).

Example: ADDI x5, x6, -1
  opcode = 0010011  (OP-IMM)
  rd     = 00101    (x5)
  funct3 = 000      (ADDI)
  rs1    = 00110    (x6)
  imm    = 111111111111  (-1 in 12-bit two's complement)

Binary: 111111111111 00110 000 00101 0010011
Hex:    0xFFF30293

Example: LW x8, 12(x9)   -- load word, offset +12 from x9
  opcode = 0000011  (LOAD)
  rd     = 01000    (x8)
  funct3 = 010      (LW)
  rs1    = 01001    (x9)
  imm    = 000000001100  (+12)

Binary: 000000001100 01001 010 01000 0000011
Hex:    0x00C4A403
```

### S-type (Store)

Used exclusively for store instructions. There is no destination register, so the bits normally used for `rd` are repurposed for the upper portion of the 12-bit immediate.

```
 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
| imm[11:5]|  rs2  |  rs1  |funct3|imm[4:0]|  opcode |
|  [31:25] |[24:20]|[19:15]|[14:12]| [11:7]|  [6:0]  |
+----------+-------+-------+------+--------+---------+

The 12-bit store offset is reassembled as: {imm[11:5], imm[4:0]}
This split keeps rs1 and rs2 at their canonical bit positions.

Example: SW x10, 8(x11)  -- store word in x10 to address (x11 + 8)
  opcode = 0100011  (STORE)
  imm[4:0] = 01000  (bits 3..0 of 8 = 8 = 0b01000, lower 5 bits)
  funct3   = 010    (SW)
  rs1      = 01011  (x11)
  rs2      = 01010  (x10)
  imm[11:5]= 0000000  (upper 7 bits of 8)

Binary: 0000000 01010 01011 010 01000 0100011
Hex:    0x00A5A423
```

### B-type (Branch)

Derived from S-type but with a different immediate layout optimised for branch target addresses. Branches always target PC-relative, 2-byte-aligned addresses (the offset is a multiple of 2, and bit 0 is implicitly 0).

```
 31      25  24   20  19   15  14  12  11    7   6      0
+----------+-------+-------+------+--------+---------+
|i[12|10:5]|  rs2  |  rs1  |funct3|i[4:1|11]| opcode |
|  [31:25] |[24:20]|[19:15]|[14:12]| [11:7] |  [6:0]  |
+----------+-------+-------+------+--------+---------+

Immediate bit extraction (the 'scrambling'):
  inst[31]    => imm[12]  (sign bit)
  inst[30:25] => imm[10:5]
  inst[11:8]  => imm[4:1]
  inst[7]     => imm[11]

Reconstructed immediate: {imm[12], imm[11], imm[10:5], imm[4:1], 1'b0}
This is a 13-bit signed value; range: -4096 to +4094 bytes.

Why scramble bits? So rs1, rs2, and funct3 stay at constant positions in
the 32-bit word. Decoder hardware reads these fields the same way in both
S-type and B-type, saving gates and design complexity.

Example: BEQ x1, x2, +8   (branch 8 bytes forward if x1 == x2)
  offset = 8 = 0b0000_0000_1000, bit layout:
    imm[12]  = 0
    imm[11]  = 0
    imm[10:5]= 000000
    imm[4:1] = 0100   (8 >> 1 = 4 = 0b0100)

  opcode   = 1100011  (BRANCH)
  imm[4:1] = 0100, imm[11] = 0  => inst[11:7] = 00100 = 0x04
  funct3   = 000      (BEQ)
  rs1      = 00001    (x1)
  rs2      = 00010    (x2)
  imm[12]  = 0, imm[10:5] = 000000  => inst[31:25] = 0000000

Binary: 0000000 00010 00001 000 00100 1100011
Hex:    0x00208463
```

### U-type (Upper Immediate)

Used by LUI and AUIPC. Encodes a 20-bit immediate in the upper 20 bits of the instruction word. The lower 12 bits of the destination value come from a subsequent I-type instruction (ADDI or JALR).

```
 31                          12  11    7   6      0
+--------------------------------+--------+---------+
|          imm[31:12]            |   rd   |  opcode |
|          [31:12]               | [11:7] |  [6:0]  |
+--------------------------------+--------+---------+
    20 bits                         5 bits    7 bits

The 20-bit value is placed directly into bits [31:12] of the result register.
Bits [11:0] of the register are set to zero by LUI.
For AUIPC: result = PC + {imm[31:12], 12'b0}

Example: LUI x15, 0x12345  -- load upper immediate 0x12345000 into x15
  opcode    = 0110111  (LUI)
  rd        = 01111    (x15)
  imm[31:12]= 00010010001101000101  (0x12345)

Binary: 00010010001101000101 01111 0110111
Hex:    0x123457B7
```

### J-type (Jump)

Used only by JAL. Encodes a 21-bit PC-relative signed offset (multiples of 2). Like B-type, the immediate bits are scrambled to keep the opcode at a fixed position and to share decode logic with other formats.

```
 31                          12  11    7   6      0
+--------------------------------+--------+---------+
| imm[20|10:1|11|19:12]          |   rd   |  opcode |
|         [31:12]                | [11:7] |  [6:0]  |
+--------------------------------+--------+---------+

Immediate bit extraction:
  inst[31]     => imm[20]   (sign bit)
  inst[30:21]  => imm[10:1]
  inst[20]     => imm[11]
  inst[19:12]  => imm[19:12]

Reconstructed immediate: {imm[20], imm[19:12], imm[11], imm[10:1], 1'b0}
21-bit signed; range: -1 MB to +1 MB - 2 bytes.

Example: JAL x1, +100   (call subroutine 100 bytes ahead, save return address in x1/ra)
  offset = 100 = 0x64 = 0b0000_0110_0100
    imm[20]   = 0
    imm[19:12]= 00000000
    imm[11]   = 0
    imm[10:1] = 0001100100  (100 >> 1 = 50 = 0b0001100100... wait, 50 = 0b0110010)
               (100 in binary = 1100100, shift right 1 = 110010 = bits [10:1] = 0001100100)
    Corrected: 100 = 64h = 0110 0100b
    imm[10:1] = 0011 00100 0  => 0011001000 (note: bit0 is implicit 0)
                Actually: 100 / 2 = 50 = 0b00110010 => imm[10:1] = 0000110010

  opcode    = 1101111  (JAL)
  rd        = 00001    (x1 = ra)

Binary encoding breakdown:
  imm[20]    =  0         => bit 31
  imm[10:1]  = 0000110010 => bits [30:21]
  imm[11]    =  0         => bit 20
  imm[19:12] = 00000000   => bits [19:12]
  rd         = 00001      => bits [11:7]
  opcode     = 1101111    => bits [6:0]

Binary: 0 0000110010 0 00000000 00001 1101111
Hex:    0x064000EF
```

### Opcode Map Summary

```
opcode[6:0]   Instruction format   Group
-----------   ------------------   -------------------------
0110011       R-type               OP (integer reg-reg)
0010011       I-type               OP-IMM (integer reg-imm)
0000011       I-type               LOAD
0100011       S-type               STORE
1100011       B-type               BRANCH
0110111       U-type               LUI
0010111       U-type               AUIPC
1101111       J-type               JAL
1100111       I-type               JALR
1110011       I-type               SYSTEM (ECALL, EBREAK, CSR ops)
0001111       I-type               MISC-MEM (FENCE, FENCE.I)
```

Key invariants that simplify decode hardware:
- **opcode is always [6:0]** — first decision in every decode tree
- **rd is always [11:7]** — can be read before opcode is fully decoded
- **rs1 is always [19:15]** — can be sent to register file early
- **rs2 is always [24:20]** — same
- **funct3 is always [14:12]** — secondary ALU control
- **The sign bit of every immediate is always bit 31** — sign extension uses a single wire

---

## Tier 1 — Fundamentals

### Question F1
**Name the six RISC-V base instruction formats and state which instructions use each.**

**Answer:**

| Format | Instructions that use it |
|--------|--------------------------|
| R-type | ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, SLT, SLTU, and all M-extension (MUL, DIV, REM) |
| I-type | ADDI, ANDI, ORI, XORI, SLLI, SRLI, SRAI, SLTI, SLTIU, LB, LH, LW, LBU, LHU, JALR, FENCE, ECALL, EBREAK, CSR instructions |
| S-type | SB, SH, SW |
| B-type | BEQ, BNE, BLT, BGE, BLTU, BGEU |
| U-type | LUI, AUIPC |
| J-type | JAL |

**Common mistake:** Candidates frequently place JAL in I-type because it has an immediate. JAL is J-type — its immediate is 20 bits wide (plus an implicit 0 for bit 0), not 12, and uses a distinct scrambled encoding to maximise branch range. JALR is I-type with a 12-bit immediate.

---

### Question F2
**Why do all six formats keep rs1 at bits [19:15], rs2 at bits [24:20], and rd at bits [11:7]?**

**Answer:**

Keeping register specifiers at fixed bit positions allows the decoder to **read the register file before the instruction is fully decoded**.

In a pipelined processor, the register file read happens in the same cycle as (or immediately after) instruction decode. If rs1 were at different bit positions in different formats, the hardware would have to first identify the format, then extract the correct bits, then issue the register file read — adding a pipeline stage of latency.

With fixed positions:
1. The register file receives the candidate read addresses on the same cycle the instruction arrives in the decode stage.
2. The result of the decode (which format this is, whether rs1 or rs2 are actually used) is used to **select** the correct value from the register file outputs, not to decide when to read.

This technique is called **early register file read** or **speculative register read** and is used in essentially every modern RISC-V implementation. It costs nothing extra because unused register values are simply discarded.

---

### Question F3
**What is the width of each field in an R-type instruction? How many distinct R-type operations can the format express?**

**Answer:**

```
Field     Bits    Width   Range
-------   ------  ------  --------------------------------
opcode    [6:0]   7 bits  128 opcode classes
rd        [11:7]  5 bits  32 destination registers (x0-x31)
funct3    [14:12] 3 bits  8 sub-operations per opcode
rs1       [19:15] 5 bits  32 source registers
rs2       [24:20] 5 bits  32 source registers
funct7    [31:25] 7 bits  128 further sub-operations per funct3
```

Maximum distinct R-type operations expressible: 8 (funct3) x 128 (funct7) = 1024, per opcode class. In practice only a small fraction are defined; the rest are reserved for extensions. The base RV32I uses opcode `0110011` for integer register-register ops, defining only 10 operations across funct3 and the relevant funct7 bits.

---

### Question F4
**What is bits [1:0] in every standard 32-bit RISC-V instruction, and why?**

**Answer:**

Bits [1:0] are always `11` in every 32-bit standard RISC-V instruction. This is the length encoding specified by the ISA:

```
inst[1:0] = 00  =>  16-bit compressed instruction (RVC quadrant 0)
inst[1:0] = 01  =>  16-bit compressed instruction (RVC quadrant 1)
inst[1:0] = 10  =>  16-bit compressed instruction (RVC quadrant 2)
inst[1:0] = 11  =>  32-bit instruction (all base + standard extensions)
```

This scheme allows processors implementing both C (compressed) and I (32-bit base) extensions to determine the instruction length in a **single cycle** by examining just the lowest two bits of the fetched half-word. The fetch unit knows immediately whether to consume 2 or 4 bytes.

Instructions longer than 32 bits (48-bit and 64-bit are reserved encoding spaces in the spec) use additional bit patterns: `11111` in bits [4:2] indicates 48-bit, `011111` in bits [5:2] indicates 64-bit, but these are not in current standard use.

---

## Tier 2 — Intermediate

### Question I1
**Explain the design decision to place the sign bit of all immediates at bit 31. What hardware benefit does this provide?**

**Answer:**

In all six formats, whenever an immediate is present, its most-significant (sign) bit occupies bit 31 of the instruction word. This is true for:
- I-type: imm[11] is at bit 31
- S-type: imm[11] is at bit 31
- B-type: imm[12] is at bit 31
- U-type: imm[31] is at bit 31
- J-type: imm[20] is at bit 31

The hardware benefit is **sign extension with zero extra logic**. Sign-extending a value from N bits to 32 bits requires replicating bit N-1 into all higher positions. If the sign bit were at different positions depending on the format, the sign extension mux would need to select from multiple possible bit positions before extending.

Because bit 31 is always the sign bit, the sign extension hardware is simply:
```
imm_sign = inst[31]  -- always, regardless of format
sign_extended_imm = { {20{inst[31]}}, imm[11:0] }  -- for 12-bit immediates
```

This saves mux logic, avoids a decode-dependent critical path in the immediate generation block, and is one of the reasons RISC-V decode is unusually fast.

---

### Question I2
**Draw the bit-level encoding of `SLLI x4, x5, 3` and explain how the decoder distinguishes SLLI from SRLI and SRAI.**

**Answer:**

SLLI (Shift Left Logical Immediate) is an I-type instruction with special encoding:

```
Instruction: SLLI x4, x5, 3
  opcode = 0010011  (OP-IMM)
  rd     = 00100    (x4)
  funct3 = 001      (SLL shifts)
  rs1    = 00101    (x5)
  shamt  = 00011    (3, the shift amount, occupies imm[4:0])
  imm[11:5] = 0000000  (must be 0000000 for SLLI)

Binary: 0000000 00011 00101 001 00100 0010011
        ^^^^^^^                              
        funct7=0000000 distinguishes SLLI
Hex: 0x00329213
```

Decode disambiguation:
```
funct3 = 001:
  inst[31:25] = 0000000  =>  SLLI (shift left logical)
  inst[31:25] = 0000000  =>  (only SLLI uses funct3=001)

funct3 = 101:
  inst[31:25] = 0000000  =>  SRLI (shift right logical, zero fill)
  inst[31:25] = 0100000  =>  SRAI (shift right arithmetic, sign fill)
```

For shift instructions, bits [31:25] behave exactly like funct7 in R-type instructions, splitting what looks like a 12-bit immediate field into a 7-bit function selector and a 5-bit shift amount (`shamt`). The shift amount cannot exceed 31 for RV32I, so only 5 bits are needed.

**Common mistake:** Treating bits [31:25] of a shift immediate instruction as part of the immediate value. They are a function code, not a signed immediate. Treating `SRAI x1, x1, 1` (encoding 0x40105093) as having a large negative immediate would give the wrong result.

---

### Question I3
**An instruction has opcode `0000011`, funct3 `010`. What instruction is this? What are its source register, destination register, and immediate value if the full encoding is `0x0144A303`?**

**Answer:**

Step 1 — Identify the instruction:
```
opcode = 0000011  =>  LOAD group
funct3 = 010      =>  LW (Load Word, 32-bit signed)
```

Step 2 — Decode `0x0144A303`:
```
0x0144A303 = 0000 0001 0100 0100 1010 0011 0000 0011

Binary: 0000_0001_0100 | 0_1001 | 010 | 0_0110 | 0000011
         imm[11:0]       rs1     funct3   rd     opcode

imm[11:0] = 000000010100 = 0x014 = 20 (decimal)
rs1       = 01001        = x9
funct3    = 010          (LW confirmed)
rd        = 00110        = x6
opcode    = 0000011      (LOAD confirmed)
```

Step 3 — State the instruction:
```
LW x6, 20(x9)
```
This loads a 32-bit word from address `(x9 + 20)` into register `x6`. The 20-byte offset is sign-extended (here it is positive, so extension is zero-fill into upper 20 bits).

---

### Question I4
**Why does the B-type format split the immediate across non-contiguous fields? Describe exactly which bits of the 32-bit instruction word encode which bits of the branch offset.**

**Answer:**

B-type splits the immediate for the same fundamental reason as J-type: **register field compatibility**. B-type instructions still have rs1 and rs2 fields (they compare two registers), so those fields must stay at [19:15] and [24:20]. This constrains the immediate encoding to use all other available bit positions.

The specific bit mapping is:
```
Immediate bit  =>  Instruction bit
imm[12]           inst[31]      (sign bit, same position as all other formats)
imm[10:5]         inst[30:25]   (upper portion of lower half)
imm[4:1]          inst[11:8]    (lower portion, excluding bit 0)
imm[11]           inst[7]       (a 'rotated' bit — this is the key oddity)
imm[0]            (implicit 0)  (branches always target 2-byte-aligned addresses)
```

The unusual placement of imm[11] at inst[7] was chosen so that the S-type and B-type encodings differ minimally — only in bit 7 and bit 31. This means the lower 25 bits of S-type and B-type look nearly identical, which simplifies partial decode and allows shared logic between store and branch address generation.

Reconstruction in hardware:
```verilog
assign b_imm = { {19{inst[31]}},  // sign extension
                  inst[31],       // imm[12]
                  inst[7],        // imm[11]
                  inst[30:25],    // imm[10:5]
                  inst[11:8],     // imm[4:1]
                  1'b0 };         // imm[0] = 0
```

---

## Tier 3 — Advanced

### Question A1
**A hardware decode unit must simultaneously determine the instruction format, extract rs1/rs2/rd, extract the immediate, and produce ALU control signals — all in a single pipeline stage. Describe an efficient hardware implementation strategy and identify the critical path.**

**Answer:**

An efficient decode stage exploits all three RISC-V decode invariants simultaneously:

**1. Parallel field extraction (no format-dependent muxing for register fields):**
```
// These are unconditional assignments — no enable, no format check needed
rd   = instruction[11:7];
rs1  = instruction[19:15];
rs2  = instruction[24:20];
```
All three are valid as soon as the instruction word arrives. The register file can be addressed immediately.

**2. Format identification from opcode:**
```
opcode = instruction[6:0];

// Opcode bits [6:5] and [4:2] identify the format:
// [6:2] = 01100  => R-type  (OP)
// [6:2] = 00100  => I-type  (OP-IMM)
// [6:2] = 00000  => I-type  (LOAD)
// [6:2] = 01000  => S-type  (STORE)
// [6:2] = 11000  => B-type  (BRANCH)
// [6:2] = 01101  => U-type  (LUI)
// [6:2] = 00101  => U-type  (AUIPC)
// [6:2] = 11011  => J-type  (JAL)
// [6:2] = 11001  => I-type  (JALR)
// [6:2] = 11100  => I-type  (SYSTEM)
```

**3. Immediate generation with a single mux tree:**
```verilog
// Compute all five immediate forms in parallel, then select:
wire [31:0] imm_i = {{20{inst[31]}}, inst[31:20]};
wire [31:0] imm_s = {{20{inst[31]}}, inst[31:25], inst[11:7]};
wire [31:0] imm_b = {{19{inst[31]}}, inst[31], inst[7],
                      inst[30:25], inst[11:8], 1'b0};
wire [31:0] imm_u = {inst[31:12], 12'b0};
wire [31:0] imm_j = {{11{inst[31]}}, inst[31], inst[19:12],
                      inst[20], inst[30:21], 1'b0};

// 5-to-1 mux selects based on format (one-hot from opcode decode)
wire [31:0] imm_out = ({32{sel_i}} & imm_i) |
                      ({32{sel_s}} & imm_s) |
                      ({32{sel_b}} & imm_b) |
                      ({32{sel_u}} & imm_u) |
                      ({32{sel_j}} & imm_j);
```

**Critical path analysis:**
The critical path through decode is: instruction register output -> opcode decode -> format select signals -> immediate mux -> immediate register input.

On a 28nm process targeting 1 GHz (1 ns cycle time):
- Opcode decode (7-bit comparator): ~100 ps
- Format mux (5-to-1, 32-bit): ~150 ps
- Wire delay and setup time: ~200 ps
- Total decode stage: ~450 ps — well within a 1 ns cycle

The register file read (typically 200-400 ps for a 32-register, 2-read-port SRAM) runs in parallel on the same cycle and is often the stage's actual critical path, not the decode logic.

---

### Question A2
**RISC-V has no condition code register (flags). How does this affect instruction decoding complexity versus architectures like x86 or ARM AArch32 that do have flags?**

**Answer:**

The absence of condition codes is a deliberate RISC-V design choice with direct implications for decode hardware:

**In x86/AArch32 (with flags):**
- Many instructions implicitly write EFLAGS/NZCV as a side effect (e.g., `ADD` updates N, Z, C, V).
- Branch instructions read the flags register as an implicit input.
- Some instructions have a conditional execution qualifier (AArch32 `ADDEQ`, `SUBNE`) that adds 4-bit condition field overhead to every instruction encoding.
- The decode stage must track flag-producing instructions and flag-consuming instructions to handle dependencies — flags are a hidden register that must be tracked by the register rename machinery in an out-of-order processor.

**In RISC-V (no flags):**
- Every data dependency is explicit through named registers only.
- Branch instructions carry their own comparison (`BEQ x1, x2` — both registers are explicit operands, compared in the execute stage).
- No hidden state to track through the rename unit. Every RISC-V instruction either reads from and writes to named registers, or produces no result at all.
- The decode stage is simpler: there is no implicit source or destination to infer.

**Consequence for an out-of-order processor:**
In x86, the rename unit must maintain a mapping for EFLAGS in addition to the 16 integer registers — EFLAGS is a single architectural register written and read by many instructions, creating a dependency chain that limits instruction-level parallelism. Compiler optimisations like flag-setting suppression (`ADD` without `S` suffix in AArch32) exist specifically to break this chain. RISC-V never has this problem.

**Cost of no flags:** SLT/SLTU (set less than) plus conditional branch instructions slightly increases code size compared to an architecture where a single compare sets flags and a branch reads them. However, the hardware simplification and the elimination of the hidden dependency more than compensate at modern superscalar widths.
