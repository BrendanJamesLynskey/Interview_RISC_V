# RV32I Base Instructions

## Prerequisites
- RISC-V instruction formats (R/I/S/B/U/J) — see `instruction_formats.md`
- Two's complement arithmetic
- Binary shifts (logical vs. arithmetic)
- Basic memory addressing

---

## Concept Reference

### RV32I Instruction Groups

RV32I defines 47 distinct instructions. They divide into eight functional groups:

```
Group                  Count  Opcodes / Instructions
---------------------  -----  -----------------------------------------------------------
Integer computational   21    ADD ADDI SUB AND ANDI OR ORI XOR XORI SLL SLLI SRL SRLI
                               SRA SRAI SLT SLTI SLTU SLTIU LUI AUIPC
Load                     5    LB LH LW LBU LHU
Store                    3    SB SH SW
Branch                   6    BEQ BNE BLT BGE BLTU BGEU
Unconditional jump        2    JAL JALR
Memory ordering          2    FENCE FENCE.I
System / privileged       4    ECALL EBREAK  (+ CSRRW CSRRS CSRRC CSRRWI CSRRSI CSRRCI
                               but those are in the Zicsr extension)
```

### Integer Computational Instructions

#### Arithmetic: ADD, ADDI, SUB

```
ADD  rd, rs1, rs2    rd = rs1 + rs2          R-type  opcode=0110011  funct3=000  funct7=0000000
SUB  rd, rs1, rs2    rd = rs1 - rs2          R-type  opcode=0110011  funct3=000  funct7=0100000
ADDI rd, rs1, imm    rd = rs1 + sext(imm12)  I-type  opcode=0010011  funct3=000

Notes:
  - Arithmetic wraps silently on overflow (modulo 2^32). No overflow exception.
  - There is no SUBI: use ADDI with a negative immediate.
  - ADDI x0, x0, 0  is the canonical NOP (no operation).
  - MV rd, rs  =>  ADDI rd, rs, 0  (pseudo-instruction)
  - LI rd, imm =>  ADDI rd, x0, imm  (for small immediates fitting in 12 bits)
```

Example encodings:
```
ADD  x3, x1, x2   Binary: 0000000_00010_00001_000_00011_0110011  Hex: 0x002081B3
ADDI x5, x6, 100  Binary: 000001100100_00110_000_00101_0010011   Hex: 0x06430293
SUB  x4, x8, x9   Binary: 0100000_01001_01000_000_00100_0110011  Hex: 0x40940233
```

#### Logical: AND, ANDI, OR, ORI, XOR, XORI

```
AND   rd, rs1, rs2   rd = rs1 & rs2          R-type  funct3=111  funct7=0000000
ANDI  rd, rs1, imm   rd = rs1 & sext(imm12)  I-type  funct3=111
OR    rd, rs1, rs2   rd = rs1 | rs2          R-type  funct3=110  funct7=0000000
ORI   rd, rs1, imm   rd = rs1 | sext(imm12)  I-type  funct3=110
XOR   rd, rs1, rs2   rd = rs1 ^ rs2          R-type  funct3=100  funct7=0000000
XORI  rd, rs1, imm   rd = rs1 ^ sext(imm12)  I-type  funct3=100

Key uses:
  - ANDI: mask bits (AND with a bitmask immediate)
  - ORI:  set bits
  - XORI: flip bits; XORI rd, rs1, -1  inverts all 32 bits (NOT pseudo-instruction)
  - XOR rd, rs1, rs1  clears rd to zero (but ADDI x0 is preferred)

Sign extension of immediate matters:
  ANDI x1, x2, 0xFF  -- imm = 0x0FF, positive, zero-fills upper; acts as byte mask
  ANDI x1, x2, 0xF00 -- imm = 0xF00 = -256 signed; sign-extends to 0xFFFFF00
                         This does NOT isolate the nibble at bits [11:8]!
```

#### Shift: SLL, SLLI, SRL, SRLI, SRA, SRAI

```
SLL   rd, rs1, rs2   rd = rs1 << rs2[4:0]   R-type  funct3=001  funct7=0000000
SLLI  rd, rs1, shamt rd = rs1 << shamt       I-type  funct3=001  imm[11:5]=0000000
SRL   rd, rs1, rs2   rd = rs1 >> rs2[4:0]   R-type  funct3=101  funct7=0000000  (logical)
SRLI  rd, rs1, shamt rd = rs1 >> shamt       I-type  funct3=101  imm[11:5]=0000000
SRA   rd, rs1, rs2   rd = rs1 >> rs2[4:0]   R-type  funct3=101  funct7=0100000  (arithmetic)
SRAI  rd, rs1, shamt rd = rs1 >> shamt       I-type  funct3=101  imm[11:5]=0100000

  - Only the lower 5 bits of rs2 are used as shift amount (range 0-31).
  - SRL fills vacated high bits with 0 (unsigned right shift).
  - SRA fills vacated high bits with the sign bit (arithmetic/signed right shift).
  - SLLI/SRLI/SRAI: shamt field is imm[4:0]; imm[11:5] serves as funct7.

Encoding example — SRAI x1, x1, 3:
  opcode = 0010011, funct3 = 101, imm[11:5] = 0100000, shamt = 00011
  Binary: 0100000_00011_00001_101_00001_0010011
  Hex:    0x4030D093
```

#### Compare: SLT, SLTI, SLTU, SLTIU

```
SLT   rd, rs1, rs2   rd = (rs1 <  rs2) ? 1 : 0  R-type  funct3=010  (signed compare)
SLTI  rd, rs1, imm   rd = (rs1 <  imm) ? 1 : 0  I-type  funct3=010
SLTU  rd, rs1, rs2   rd = (rs1 <u rs2) ? 1 : 0  R-type  funct3=011  (unsigned compare)
SLTIU rd, rs1, imm   rd = (rs1 <u imm) ? 1 : 0  I-type  funct3=011

  - Result is 0 or 1 (not a flag/condition code — written to rd as a normal value).
  - These are the only integer comparison instructions that produce a boolean.
  - SEQZ rd, rs  =>  SLTIU rd, rs, 1  (set if rs == 0; only value < unsigned 1 is 0)
  - SNEZ rd, rs  =>  SLTU  rd, x0, rs (set if rs != 0; x0=0 < rs only if rs != 0)

  Note on SLTIU sign extension: the 12-bit immediate is sign-extended to 32 bits,
  then treated as an unsigned 32-bit value for the comparison.
  SLTIU rd, rs1, 1  =>  imm = sign_ext(1) = 0x00000001 (unsigned), checks rs1 < 1.
  SLTIU rd, rs1, -1 =>  imm = sign_ext(-1) = 0xFFFFFFFF (unsigned), checks rs1 < 0xFFFFFFFF,
                         which is true for all values except 0xFFFFFFFF itself.
```

#### Upper Immediate: LUI, AUIPC

```
LUI   rd, imm20   rd = {imm20, 12'b0}          U-type  opcode=0110111
AUIPC rd, imm20   rd = PC + {imm20, 12'b0}     U-type  opcode=0010111

LUI (Load Upper Immediate):
  - Places a 20-bit immediate into bits [31:12] of rd; bits [11:0] are zeroed.
  - Used in pairs with ADDI to synthesise any 32-bit constant:
      LUI  x1, 0x12345     ; x1 = 0x12345000
      ADDI x1, x1, 0x678   ; x1 = 0x12345678
  - Pitfall: if the lower 12 bits intended for ADDI are >= 0x800 (bit 11 set),
    the sign-extended ADDI subtracts from the intended value. Compensate:
      To load 0x12345FFF:
        ADDI imm = 0xFFF = -1 (signed).
        So pre-add 1 to the upper immediate:
        LUI  x1, 0x12346     ; x1 = 0x12346000
        ADDI x1, x1, -1      ; x1 = 0x12345FFF  ✓

AUIPC (Add Upper Immediate to PC):
  - Result is PC-relative: enables position-independent code (PIC).
  - Used with JALR to construct PC-relative calls to functions far away:
      AUIPC x1, %hi(target)    ; x1 = PC + (upper 20 bits of offset)
      JALR  x0, x1, %lo(target) ; jump to x1 + lower 12 bits of offset
  - Also used with ADDI to compute addresses of global variables:
      AUIPC x1, %hi(global_var)
      ADDI  x1, x1, %lo(global_var)  ; x1 = address of global_var
```

### Load Instructions

```
LB  rd, offset(rs1)  rd = sext(mem[rs1+offset][ 7:0])  8-bit  signed extend
LH  rd, offset(rs1)  rd = sext(mem[rs1+offset][15:0]) 16-bit  signed extend
LW  rd, offset(rs1)  rd =      mem[rs1+offset][31:0]  32-bit  (no extension needed)
LBU rd, offset(rs1)  rd = zext(mem[rs1+offset][ 7:0])  8-bit  zero extend
LHU rd, offset(rs1)  rd = zext(mem[rs1+offset][15:0]) 16-bit  zero extend

All loads: I-type, opcode=0000011
  LB:  funct3=000
  LH:  funct3=001
  LW:  funct3=010
  LBU: funct3=100
  LHU: funct3=101

  - offset is a signed 12-bit immediate: range -2048 to +2047 bytes.
  - Address must be aligned for LH/LW in most implementations; misaligned access
    generates an exception unless the implementation supports misaligned loads.
  - There is no LWU in RV32I; the 32-bit register already holds the full word.
    LWU exists in RV64I to zero-extend a 32-bit value into a 64-bit register.

Example: LW x10, -4(x2)  -- load word from address (sp - 4)
  opcode   = 0000011
  rd       = 01010  (x10)
  funct3   = 010    (LW)
  rs1      = 00010  (x2 = sp)
  imm[11:0]= 111111111100  (-4 in two's complement)
  Binary: 111111111100_00010_010_01010_0000011
  Hex:    0xFFC12503
```

### Store Instructions

```
SB  rs2, offset(rs1)  mem[rs1+offset][ 7:0] = rs2[ 7:0]   S-type  funct3=000
SH  rs2, offset(rs1)  mem[rs1+offset][15:0] = rs2[15:0]   S-type  funct3=001
SW  rs2, offset(rs1)  mem[rs1+offset][31:0] = rs2[31:0]   S-type  funct3=010

All stores: opcode=0100011

  - Only the low bytes of rs2 are written; upper bits are ignored for SB/SH.
  - The 12-bit signed offset is split: imm[11:5] at bits [31:25], imm[4:0] at bits [11:7].
  - No destination register (no rd field). The rd bits are reused for imm[4:0].

Example: SW x11, 16(x2)  -- store word x11 to address (sp + 16)
  offset = 16 = 0x10 = 0b00000010000
  imm[11:5] = 0000000
  imm[4:0]  = 10000
  opcode   = 0100011
  funct3   = 010    (SW)
  rs1      = 00010  (x2 = sp)
  rs2      = 01011  (x11)
  Binary: 0000000_01011_00010_010_10000_0100011
  Hex:    0x00B12823
```

### Branch Instructions

```
BEQ  rs1, rs2, offset  if (rs1 == rs2)  PC += sext(offset)  funct3=000
BNE  rs1, rs2, offset  if (rs1 != rs2)  PC += sext(offset)  funct3=001
BLT  rs1, rs2, offset  if (rs1 <  rs2)  PC += sext(offset)  funct3=100  (signed)
BGE  rs1, rs2, offset  if (rs1 >= rs2)  PC += sext(offset)  funct3=101  (signed)
BLTU rs1, rs2, offset  if (rs1 <u rs2)  PC += sext(offset)  funct3=110  (unsigned)
BGEU rs1, rs2, offset  if (rs1 >=u rs2) PC += sext(offset)  funct3=111  (unsigned)

All branches: B-type, opcode=1100011

  - 13-bit signed PC-relative offset (bit 0 implicit zero).
  - Range: -4096 to +4094 bytes from the branch instruction's address.
  - No BGT/BLE/BGTU/BLEU: these are pseudo-instructions that swap rs1/rs2 and use BLT/BGE.
  - BEQZ rs, offset => BEQ rs, x0, offset
  - BNEZ rs, offset => BNE rs, x0, offset
  - BLTZ rs, offset => BLT rs, x0, offset
  - BGEZ rs, offset => BGE rs, x0, offset
  - On not-taken path: PC = PC + 4 (next sequential instruction).
```

### Jump Instructions: JAL and JALR

```
JAL  rd, offset    rd = PC+4;  PC = PC + sext(offset)   J-type  opcode=1101111
JALR rd, rs1, imm  rd = PC+4;  PC = (rs1 + sext(imm)) & ~1  I-type  opcode=1100111  funct3=000

JAL (Jump And Link):
  - PC-relative jump. offset is a 21-bit signed value (bit 0 implicit zero).
  - Range: -1 MB to +1 MB - 2 bytes.
  - rd receives the address of the instruction following JAL (return address).
  - J rd, offset  =>  JAL x0, offset  (discard return address)
  - CALL offset   =>  JAL ra, offset  (subroutine call; ra = x1)

JALR (Jump And Link Register):
  - Register-indirect jump. Target address = (rs1 + imm) with bit 0 forced to 0.
  - The forced-zero of bit 0 handles the case where the LSB of rs1 is set,
    ensuring the target is always instruction-aligned (to 2-byte for compressed,
    or the hardware will exception if trying to jump to a misaligned 32-bit instruction).
  - RET pseudo-instruction => JALR x0, ra, 0  (return from subroutine)
  - Used together with AUIPC for far calls (>1 MB range):
      AUIPC ra, %hi(far_func)
      JALR  ra, ra, %lo(far_func)

Return address convention:
  - rd = x0: discard link (plain jump)
  - rd = x1 (ra): standard subroutine call
  - rd = x5 (t0): alternate link register (for linker veneer stubs)
```

### Memory Ordering: FENCE

```
FENCE  pred, succ   I-type  opcode=0001111  funct3=000
FENCE.I             I-type  opcode=0001111  funct3=001

FENCE:
  - Ensures that memory operations from before the FENCE (in program order) in the
    predecessor set (pred) are visible to other harts before any memory operations
    after the FENCE in the successor set (succ).
  - pred and succ are 4-bit fields: I (device input), O (device output), R (read), W (write).
  - FENCE RW, RW  is the full barrier (all reads and writes before, all after).
  - On single-hart systems with no I/O ordering requirements, FENCE is a NOP.

FENCE.I:
  - Synchronises instruction fetch with data stores. Required before executing
    freshly-written code (self-modifying code, JIT compilation, dynamic linker).
  - After writing to memory with SW and before executing from that address, a
    FENCE.I ensures the instruction cache is coherent with the data cache.
```

### System Instructions: ECALL, EBREAK

```
ECALL   opcode=1110011  funct3=000  imm=000000000000  -- environment call
EBREAK  opcode=1110011  funct3=000  imm=000000000001  -- breakpoint exception

ECALL:
  - Raises a synchronous exception, transferring control to the trap handler.
  - Used by U-mode software to invoke OS services (equivalent to a syscall).
  - The system call number and arguments are passed in registers (a0-a7 by convention).

EBREAK:
  - Raises a breakpoint exception (mcause = 3 in M-mode, dcsr in Debug mode).
  - Used by debuggers to set software breakpoints and by C runtime panic handlers.
  - In the debug specification, EBREAK entered while in debug mode halts the hart
    (re-entering the debugger) rather than taking a trap.
```

---

## Tier 1 — Fundamentals

### Question F1
**What is the result of `ADDI x5, x0, -1`? What is the bit pattern in x5?**

**Answer:**

`x0` is the hardwired zero register, permanently 0. `ADDI x5, x0, -1` computes:

```
x5 = x0 + sign_extend(-1, 12 bits)
x5 = 0 + 0xFFFFFFFF
x5 = 0xFFFFFFFF
```

The 12-bit immediate `-1` is `111111111111` in binary. Sign-extended to 32 bits, this is `0xFFFFFFFF`.

In x5: `1111_1111_1111_1111_1111_1111_1111_1111`

This is a common idiom to fill a register with all-ones, equivalent to -1 in two's complement or the unsigned maximum value 4,294,967,295.

**Follow-up:** Can you write to x0? No. x0 is a constant-zero register. Any instruction that specifies x0 as `rd` silently discards the result. This is used to achieve side-effecting operations with no result (e.g., `JALR x0, ra, 0` — jump to return address, discard the link).

---

### Question F2
**What is the difference between LB and LBU? When would you use each?**

**Answer:**

Both load a single byte from memory into a 32-bit register. They differ in how the byte is extended to fill the upper 24 bits:

```
LB  (Load Byte, signed):   result = sign_extend(mem_byte)
                           If mem_byte[7] = 1: upper 24 bits are all 1
                           If mem_byte[7] = 0: upper 24 bits are all 0
                           Range of result: -128 to +127

LBU (Load Byte Unsigned):  result = zero_extend(mem_byte)
                           Upper 24 bits are always 0
                           Range of result: 0 to 255
```

**Use LB** when the byte represents a signed value (e.g., reading a `int8_t` array).
**Use LBU** when the byte represents an unsigned value (e.g., reading a `uint8_t` array, ASCII character processing, reading a byte of a pixel value).

**Common mistake:** Using LB to read a byte value of 0xFF (255) and expecting to get 255. LB sign-extends 0xFF to 0xFFFFFFFF (-1 signed). You must use LBU to get 255.

---

### Question F3
**What pseudo-instructions do compilers use to: (a) copy a register, (b) load a 32-bit constant, (c) return from a function, (d) do nothing?**

**Answer:**

```
(a) Copy register — MV rd, rs
    Real instruction: ADDI rd, rs, 0
    Adds zero; result is rs unchanged.

(b) Load 32-bit constant — LI rd, imm32
    For small values (fits in 12 bits): ADDI rd, x0, imm12
    For large values: assembler expands to two instructions:
      LUI  rd, imm[31:12]+(imm[11] ? 1 : 0)  ; compensate for ADDI sign extension
      ADDI rd, rd, imm[11:0]
    Note: The compensation is needed because ADDI sign-extends the 12-bit value.
    If imm[11] = 1 (i.e., bit 11 set), ADDI will subtract, so add 1 to the upper part.

(c) Return from function — RET
    Real instruction: JALR x0, ra, 0  (= JALR x0, x1, 0)
    Jumps to address in ra (x1), discards the link by writing to x0.

(d) No operation — NOP
    Real instruction: ADDI x0, x0, 0
    Writes to x0 (discarded), computes 0+0=0, no side effects.
    Encoding: 0x00000013
```

---

### Question F4
**What is the maximum branch offset in RV32I? If a branch target is too far away, how does a compiler handle it?**

**Answer:**

The B-type immediate is a 13-bit signed value (bit 0 implicit zero), giving a range of:
```
Minimum: -4096 bytes (-0x1000)
Maximum: +4094 bytes (+0xFFE)
```

This covers approximately ±4 KB from the branch instruction.

If a branch target is outside this range (a **far branch**), the compiler generates a **branch inversion with a JAL or JALR**:

```assembly
# Original intent: BEQ x1, x2, far_target  (but far_target is > 4 KB away)

# Compiler emits:
  BNE x1, x2, skip    # invert condition: branch over the jump if NOT equal
  JAL x0, far_target  # JAL has ±1 MB range (J-type, 21-bit offset)
skip:
  # next instruction

# If far_target is also beyond JAL range (>1 MB away):
  BNE x1, x2, skip
  AUIPC x1, %hi(far_target)
  JALR  x0, x1, %lo(far_target)
skip:
```

The BNE takes the short (skip) path in the common case, adding only one extra instruction to the common path. This is called a **branch relaxation** or **long branch** sequence.

---

## Tier 2 — Intermediate

### Question I1
**Explain how to synthesise a 32-bit constant `0xDEADBEEF` into register x10 using RV32I instructions. Show the correct LUI/ADDI calculation.**

**Answer:**

```
Target: x10 = 0xDEADBEEF

Step 1: Split into upper 20 bits and lower 12 bits:
  Upper: 0xDEADB   (bits [31:12] = 0xDEADB000 >> 12 = 0xDEADB)
  Lower: 0xEEF     (bits [11:0]  = 0xEEF)

Step 2: Check if lower 12 bits have bit 11 set:
  0xEEF = 0b1110_1110_1111  -- bit 11 = 1  => ADDI will sign-extend to 0xFFFFFEEF
  ADDI will effectively subtract 0x111 = 273 from the LUI value.
  Compensate: add 1 to the upper immediate.

Step 3: Final calculation:
  LUI immediate  = 0xDEADB + 1 = 0xDEADC
  ADDI immediate = 0xEEF = -273 as signed 12-bit = 0xFFFFFEEF sign-extended

Verification:
  LUI x10, 0xDEADC    =>  x10 = 0xDEADC000
  ADDI x10, x10, 0xEEF (= -273 signed)
  x10 = 0xDEADC000 + 0xFFFFFEEF = 0xDEADBEEF  ✓

Assembly:
  lui  a0, 0xDEADC       # a0 = 0xDEADC000
  addi a0, a0, -273      # a0 = 0xDEADBEEF (assembler accepts hex: addi a0, a0, 0xEEF)

Encoding:
  LUI:  0xDEADC7B7  -- imm[31:12]=0xDEADC, rd=01111 (x15... wait, a0=x10)
  Let rd = x10 = 01010:
  LUI:  imm[31:12]=0xDEADC, rd=01010, opcode=0110111
        0xDEADC000 | 0x537 = ...
  Binary: 11011110101011011100 01010 0110111
  Hex: 0xDEADC537

  ADDI: imm=0xEEF, rs1=01010, funct3=000, rd=01010, opcode=0010011
        111011101111 01010 000 01010 0010011
  Hex: 0xEEF50513
```

**Alternative check using assembler output:**
```c
// In C:  int x = 0xDEADBEEF;
// GCC -O2 -march=rv32i output:
//   lui  a0, 0xDEADC
//   addi a0, a0, -273
```

---

### Question I2
**What is AUIPC used for? Give a concrete example showing how a compiler uses AUIPC + ADDI to compute the address of a global variable in position-independent code.**

**Answer:**

AUIPC (Add Upper Immediate to PC) places `PC + {imm20, 12'b0}` into the destination register. The key property is that the result is **PC-relative**: it does not depend on the absolute load address of the binary.

Without position-independent code (absolute addressing):
```assembly
# Non-PIC: loads the absolute address 0x10020 — breaks if binary is relocated
lui  a0, 0x10
addi a0, a0, 0x20   # a0 = 0x10020
lw   a1, 0(a0)      # load from fixed address
```

With AUIPC (PC-relative addressing):
```assembly
# Suppose the AUIPC instruction is at PC = 0x8000 and global_var is at 0x9024.
# PC-relative offset = 0x9024 - 0x8000 = 0x1024
# Split: upper = 0x1, lower = 0x024

auipc a0, 0x1        # a0 = PC + 0x1000 = 0x8000 + 0x1000 = 0x9000
addi  a0, a0, 0x24   # a0 = 0x9000 + 0x24 = 0x9024  (address of global_var)
lw    a1, 0(a0)      # load the variable
```

If the binary is mapped at a different base address (say 0x20008000), the AUIPC still produces the correct result because it adds relative to the current PC, which also moved by the same offset.

**Linker relocations:** In practice, the assembler emits AUIPC with the relocation `R_RISCV_PCREL_HI20` and ADDI with `R_RISCV_PCREL_LO12_I`. The linker fills in the correct immediate values when it knows the final layout.

---

### Question I3
**Describe the semantics of JALR. Why is bit 0 of the target address forced to zero? What subtle bug can this cause with JALR used for an indirect branch table?**

**Answer:**

JALR computes:
```
target = (rs1 + sign_extend(imm12)) & ~1   // bit 0 forced to 0
rd     = PC + 4                             // link address (next instruction)
PC     = target
```

**Why bit 0 is forced to zero:** The RISC-V ISA guarantees that instruction fetch addresses are at minimum 2-byte aligned (to support the C extension compressed instructions). Bit 0 of a valid instruction address is always 0. Forcing bit 0 to zero in JALR prevents accidental jumps to odd addresses (which would generate an instruction-address-misaligned exception on implementations that don't support the C extension). It also means software can safely store a function pointer in a register that may have its LSB set for other purposes (e.g., tagging pointers) — the JALR will strip the tag before jumping.

**Branch table bug:** Consider a hand-written jump table:
```assembly
# Indirect branch table in .rodata:
# table: .word target_0, target_1, target_2 ...

  slli t1, a0, 2      # t1 = index * 4 (word offset)
  la   t0, table      # t0 = base address of table
  add  t0, t0, t1     # t0 = &table[index]
  lw   t1, 0(t0)      # t1 = table[index] (the target address)
  jalr x0, t1, 0      # jump to target
```

The bug: if `target_n` values are absolute addresses stored in the table, and one of them accidentally has bit 0 set (e.g., due to a linker script error or a 32-bit pointer stored as 1-byte-offset from a previous instruction), JALR silently clears bit 0 and jumps to the wrong (neighbouring) instruction. No exception is raised — the bug manifests as mysterious misbehaviour.

**Mitigation:** Always verify branch table entries are correctly aligned. In debug builds, assert `(table_entry & 1) == 0` before the JALR. The RVC `JALR` (in compressed instructions) has the same behaviour.

---

### Question I4
**A program reads a `uint8_t` value from memory using LBU and then wants to isolate the lower nibble (bits [3:0]). Write the sequence. What if you used LB instead — would the AND still work?**

**Answer:**

Using LBU (correct approach):
```assembly
lbu  a1, 0(a0)       # a1 = 0x000000XX  (zero-extended byte, range 0-255)
andi a1, a1, 0xF     # a1 = 0x0000000X  (isolate lower nibble, range 0-15)
```
This works correctly. ANDI sign-extends the 12-bit immediate `0xF` to `0x0000000F`, then ANDs with a1.

Using LB (potential issue):
```assembly
lb   a1, 0(a0)       # a1 = sign_extended(mem_byte)
                     # If byte >= 0x80: a1 = 0xFFFFFFxx (upper bits all 1)
andi a1, a1, 0xF     # a1 = 0x0000000X  (lower nibble isolated)
```

The AND **does still work** to isolate the lower nibble, because `0x0000000F` masks away all upper bits regardless of whether they are 0 or 1. The lower nibble of `0xFFFFFFxx` is the same as the lower nibble of `xx`.

However, if you needed to test whether the byte is less than 32 using SLTI:
```assembly
lb   a1, 0(a0)
slti a2, a1, 32      # a2 = (a1 < 32) ? 1 : 0
                     # If original byte was 0xFF (255), a1 = -1 after LB.
                     # -1 < 32 (signed comparison), so a2 = 1. WRONG for uint8_t.
```
Using LBU would give a1 = 255, and `SLTI a2, a1, 32` would give 0 (255 is not < 32). The correct answer depends on whether the byte is conceptually signed or unsigned.

---

## Tier 3 — Advanced

### Question A1
**The RISC-V ISA has no integer multiply or divide instructions in the base set. How does a bare-metal embedded toolchain implement `a * b` and `a / b` without the M extension? What is the performance impact and when does this matter?**

**Answer:**

Without the M extension, GCC/Clang emit software library calls for multiply and divide.

**Multiply (`a * b` where a and b are 32-bit):**
```assembly
# Compiler emits: call __mulsi3 (from libgcc or compiler-rt)
# __mulsi3 is a software multiply using shift-and-add:

# Pseudocode for unsigned 32x32->32 multiply:
# result = 0
# for i in range(32):
#     if b & 1:
#         result += a
#     a <<= 1
#     b >>= 1

# Optimised version (halves iterations using early exit):
__mulsi3:
  mv    t0, a0        # t0 = a (multiplicand)
  mv    a0, zero      # result = 0
.loop:
  andi  t1, a1, 1     # test bit 0 of b
  beqz  t1, .skip
  add   a0, a0, t0    # result += a if bit set
.skip:
  slli  t0, t0, 1     # a <<= 1
  srli  a1, a1, 1     # b >>= 1 (logical)
  bnez  a1, .loop     # continue while b != 0
  ret
```

**Performance:** Software multiply is 10-30 cycles on average versus 3-5 cycles for a hardware multiplier. The exact count depends on the number of set bits in the multiplier.

**Divide (`a / b`):**

Software division is significantly more expensive — typically 30-60 instructions for a non-restoring divider implementation, translating to 30-60+ cycles. The algorithm:
1. Shift the dividend into a double-width accumulator.
2. At each step, attempt subtraction of the divisor.
3. Record the quotient bit and shift.

Division is rarely needed on tight embedded paths. But fixed-point DSP code that relies on constant divisors can optimise using **multiply-by-reciprocal**:
```c
// a / 10  =>  (a * 0xCCCCCCCD) >> 35  (for unsigned 32-bit)
// Compiler with -O2 performs this transformation automatically when dividing by a constant.
```

**When it matters:**
- For MCU firmware without M extension (e.g., custom RV32E implementations): avoid runtime division in interrupt service routines.
- For FPGA soft processors using RV32I core without M: every divide is a function call costing 50+ cycles.
- For RV32I cores in area-constrained silicon: M extension adds ~10-15% gate count. Some designs omit it intentionally for ultraslow control paths.

---

### Question A2
**What is the purpose of the FENCE instruction and FENCE.I instruction? On a simple in-order single-core processor, are either of them ever required? Give a precise scenario where FENCE.I is necessary even on a single-core system.**

**Answer:**

**FENCE (memory ordering barrier):**
Ensures that memory operations from the predecessor set (before FENCE) are observable to other observers (other harts, DMA engines, peripherals) before operations from the successor set (after FENCE).

On a **simple in-order single-core processor with no write buffer, no reorder buffer, and no DMA:**
- FENCE is a NOP. There is only one hart, no out-of-order reordering, no write buffer that could reorder stores, and no other observer to synchronise with.

On a **single-core processor with a write buffer or store queue:**
- FENCE RW, RW ensures stores in the write buffer are committed to the memory system before proceeding. This matters when writing to a peripheral register (device memory) — the write must reach the peripheral, not sit in a write buffer, before reading back the status.
- Device-memory accesses should not go through a write buffer at all (use IORW ordering with the FENCE instruction or mark the region appropriately), but the FENCE is the ISA-level guarantee.

**FENCE.I (instruction-fetch ordering barrier):**
Ensures that all stores visible to the current hart before FENCE.I are observable by subsequent instruction fetches on the same hart.

**Precise scenario where FENCE.I is necessary on a single-core system:**

A JIT compiler or dynamic loader writes machine code to a memory buffer, then jumps to it:

```assembly
# Step 1: JIT compiler writes code to buffer at 0x20000:
  sw   t0, 0(a0)    # store instruction word 1 to 0x20000
  sw   t1, 4(a0)    # store instruction word 2 to 0x20004
  sw   t2, 8(a0)    # store instruction word 3 to 0x20008

# Without FENCE.I here:
# The instruction cache may still hold stale data at 0x20000-0x20008
# from a previous execution, or the I-cache may not have fetched these
# addresses yet and the store path has not updated the I-cache.

  fence.i           # REQUIRED: flush instruction cache / synchronise I$ with D$

# Step 2: Jump to the newly written code:
  jalr x0, a0, 0   # Execute from 0x20000
```

On most real implementations, the data cache and instruction cache are **separate** (Harvard-like cache organisation), and writes to the data cache are not automatically visible to the instruction cache. FENCE.I is the explicit synchronisation barrier that flushes or invalidates the instruction cache before the jump.

Without FENCE.I, the processor might fetch stale (old or garbage) instructions from the I-cache instead of the newly written code. The CPU spec does not require the hardware to snoop I-cache on D-cache writes — software must explicitly synchronise.

**Implementations:** Many simple in-order cores implement FENCE.I as a full pipeline flush with I-cache invalidation — this is expensive (potentially 10-50 cycles) but only occurs when actually needed for code patching or JIT.
