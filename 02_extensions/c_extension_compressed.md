# C Extension: Compressed Instructions

## Overview

The RISC-V C extension adds 16-bit compressed encodings for the most common instruction patterns. On typical embedded workloads, roughly 50-60% of static instructions can be compressed, yielding 25-30% code size reduction. This translates directly to lower instruction cache footprint, reduced Flash usage on microcontrollers, and improved icache hit rates. The C extension is so widely used that `RV32IMC` and `RV64GC` are the standard profiles for Linux-capable and embedded targets respectively.

---

## Key Concepts

### Why 16-bit Instructions?

Every standard RISC-V instruction is exactly 32 bits. This regularity simplifies decode but wastes encoding space when:
- An operand is a small immediate (e.g., add 4 to `sp`).
- One register is always a fixed register (e.g., `sp`, the return address register `ra`).
- The operation is a simple move or zero-register clear.

The C extension exploits these patterns: each 16-bit instruction is defined to be exactly equivalent to a specific 32-bit instruction. There are no new architectural effects — compressed instructions are purely an encoding shorthand.

### Identifying Compressed Instructions

The bottom two bits of every RISC-V instruction identify its width:

| `inst[1:0]` | Instruction Width |
|-------------|-------------------|
| `00`        | 16-bit (C extension) |
| `01`        | 16-bit (C extension) |
| `10`        | 16-bit (C extension) |
| `11`        | 32-bit (base ISA) |

A fetch unit must inspect the bottom two bits to determine how many bytes to consume. This means code can freely mix 16-bit and 32-bit instructions without alignment overhead — any 2-byte aligned address is a valid instruction start.

**Important:** 32-bit instructions are still required to be 4-byte aligned in the base ISA, but with C enabled they need only be 2-byte aligned. This is because the C extension changes the minimum instruction alignment to 2 bytes.

### Instruction Alignment with C Enabled

Without C: every instruction starts on a 4-byte boundary.
With C: every instruction starts on a 2-byte boundary.

This is directly visible in branch and jump immediate ranges: B-type and J-type instructions encode a byte offset with the lowest bit implicit. Without C, the lowest implicit bit is bit 0 (the PC always 4-byte aligned, so bit 1 is the LSB of the offset). With C, bit 0 is still implicit, but the effective range is unchanged — the encoding already handles 2-byte alignment.

### The CIW, CL, CS, CA, CB, CJ, CI, CSS Format Families

Compressed instructions use eight distinct 16-bit formats, all sharing the property that `inst[1:0] != 11`:

| Format | Key Fields | Typical Use |
|--------|------------|-------------|
| CR   | `funct4 | rd/rs1 | rs2 | op`            | Register ops, jumps    |
| CI   | `funct3 | imm | rd/rs1 | imm | op`       | Immediate ops          |
| CSS  | `funct3 | imm | rs2 | op`               | Stack-relative stores  |
| CIW  | `funct3 | imm | rd' | op`              | Stack pointer add      |
| CL   | `funct3 | imm | rs1' | imm | rd' | op` | Load                   |
| CS   | `funct3 | imm | rs1' | imm | rs2' | op`| Store                  |
| CA   | `funct6 | rd'/rs1' | funct2 | rs2' | op`| ALU ops               |
| CB   | `funct3 | offset | rs1' | offset | op` | Branches               |
| CJ   | `funct3 | jump target | op`             | Jumps                  |

The `op` field is `inst[1:0]` — quadrants 0, 1, 2.

### The RVC Register Subset (CIW/CL/CS/CA/CB Formats)

Many compressed formats use 3-bit register specifiers instead of the full 5-bit specifier, limiting them to 8 registers. The mapping is:

| 3-bit field | Register |
|-------------|----------|
| 000         | x8  (`s0` / `fp`) |
| 001         | x9  (`s1`) |
| 010         | x10 (`a0`) |
| 011         | x11 (`a1`) |
| 100         | x12 (`a2`) |
| 101         | x13 (`a3`) |
| 110         | x14 (`a4`) |
| 111         | x15 (`a5`) |

These are registers `x8`–`x15`, chosen because they are the most frequently used in typical code (argument registers `a0`-`a5`, and saved registers `s0`-`s1`). The full-register CR/CI/CSS formats still use 5-bit specifiers and can address any register.

### Key Compressed Instruction Mappings

Below are the most important compressed instructions and their 32-bit equivalents:

```
C.LW   rd', rs1', imm  <=>  LW  rd, imm(rs1)       (offset: 5-bit, 4-byte aligned)
C.SW   rs1', rs2', imm <=>  SW  rs2, imm(rs1)       (offset: 5-bit, 4-byte aligned)
C.ADDI rd, imm         <=>  ADDI rd, rd, imm         (imm: 6-bit signed, imm!=0)
C.LI   rd, imm         <=>  ADDI rd, x0, imm         (load small constant)
C.LUI  rd, imm         <=>  LUI  rd, imm              (upper 20 bits)
C.ADDI16SP imm         <=>  ADDI sp, sp, imm          (imm: 10-bit multiple of 16)
C.ADDI4SPN rd', imm    <=>  ADDI rd, sp, imm          (imm: 8-bit multiple of 4)
C.ADD  rd, rs2         <=>  ADD  rd, rd, rs2
C.SUB  rd', rs2'       <=>  SUB  rd', rd', rs2'
C.AND  rd', rs2'       <=>  AND  rd', rd', rs2'
C.OR   rd', rs2'       <=>  OR   rd', rd', rs2'
C.XOR  rd', rs2'       <=>  XOR  rd', rd', rs2'
C.MV   rd, rs2         <=>  ADD  rd, x0, rs2          (rs2 != x0)
C.JR   rs1             <=>  JALR x0, 0(rs1)
C.JALR rs1             <=>  JALR ra, 0(rs1)            (calls via register)
C.J    imm             <=>  JAL  x0, imm               (unconditional jump)
C.JAL  imm             <=>  JAL  ra, imm               (RV32 only)
C.BEQZ rs1', imm       <=>  BEQ  rs1, x0, imm
C.BNEZ rs1', imm       <=>  BNE  rs1, x0, imm
C.NOP               <=>  ADDI x0, x0, 0
C.EBREAK            <=>  EBREAK
```

On RV64C, `C.LD`/`C.SD` replace `C.JAL` (32-bit jump-and-link is less common in 64-bit code).

### Illegal Encodings and Hints

Some encodings in the C extension are architecturally illegal:
- `C.ADDI` with `imm = 0` is a `C.NOP` only when `rd = x0`; for `rd != x0` and `imm = 0` the encoding is a HINT (legal no-op).
- `C.MV` with `rs2 = x0` is illegal.
- `C.LUI` with `rd = x0` or `rd = x2 (sp)` is illegal.
- `C.ADDI16SP` with `imm = 0` is illegal.

Implementations encountering illegal 16-bit encodings raise an illegal instruction exception, just as they do for illegal 32-bit encodings.

### Code Density Impact

On the SPEC CPU2006 benchmark suite, the C extension reduces average code size by approximately 26% on RV32 and 20% on RV64. The reduced static footprint has compounding effects:
- Fewer instruction cache misses.
- Better branch predictor coverage per BTB entry.
- Reduced DMA transfer sizes for firmware.
- Smaller application binaries — important for microcontrollers with limited Flash.

---

## Interview Questions

### Fundamentals Tier

---

**Q1. How does the processor distinguish a 16-bit compressed instruction from a 32-bit instruction in a mixed instruction stream?**

**Answer:**

By examining `inst[1:0]`, the two least-significant bits of the first halfword fetched:

- If `inst[1:0] != 11` (i.e., `00`, `01`, or `10`): the instruction is 16 bits. The processor consumes 2 bytes.
- If `inst[1:0] == 11`: the instruction is 32 bits. The processor consumes 4 bytes.

This encoding is designed so that a processor without the C extension will always see `inst[1:0] == 11` for valid instructions (since all base ISA encodings have `11` in these bits). A processor with the C extension checks these bits on every fetch to decide the instruction length. No special mode bit is needed — the instruction stream self-describes its encoding width.

---

**Q2. Which eight registers does the 3-bit "register prime" (rd', rs1', rs2') field in compressed instructions refer to?**

**Answer:**

The 3-bit field maps to `x8` through `x15`:

| 3-bit encoding | Register | ABI name |
|----------------|----------|----------|
| 000 | x8  | s0 / fp |
| 001 | x9  | s1      |
| 010 | x10 | a0      |
| 011 | x11 | a1      |
| 100 | x12 | a2      |
| 101 | x13 | a3      |
| 110 | x14 | a4      |
| 111 | x15 | a5      |

These were chosen because they are the most frequently accessed registers in typical compiled code: the first six argument registers (`a0`-`a5`) and the two callee-saved registers (`s0`-`s1`). Instructions that need to access other registers (such as `x1/ra`, `x2/sp`, `x3/gp`) must use the CR or CI format variants that carry full 5-bit register specifiers.

---

**Q3. What is the 32-bit equivalent of `C.ADDI4SPN a0, sp, 12`?**

**Answer:**

`ADDI a0, sp, 12`

`C.ADDI4SPN` adds a scaled immediate (multiple of 4, non-zero) to `sp` and writes the result to a restricted register (rd'). The immediate is zero-extended and must be a multiple of 4. This instruction is specifically designed for computing stack-relative addresses of local variables, which is extremely common in function prologs and within loops accessing stack-allocated structures.

The encoding maps: `rd' = a0` (3-bit field `010`), `sp = x2`, `imm = 12`.

---

**Q4. Why does `C.JAL` exist only on RV32C and not RV64C?**

**Answer:**

In RV32 code, short-range function calls via `JAL ra, offset` are very common — library functions tend to be within a ±1 KB range of their callers in small programs. `C.JAL` covers a ±2 KB range (11-bit signed offset, 1-bit implicit) in 16 bits.

In RV64 code, programs tend to be larger and functions are spread across wider address ranges. Direct `JAL`-based calls into a ±2 KB window are rare; most calls use `JALR` via a register (through a PLT stub or function pointer). The `C.JAL` encoding space is therefore repurposed in RV64C for `C.LD` (64-bit load), which is far more useful in a 64-bit ISA. `C.JAL` for calls is replaced by `C.J` (jump-and-forget) and `C.JALR` (jump-and-link via register).

---

**Q5. If a function starts at address `0x1002` (not 4-byte aligned), is this valid in a system running with the C extension enabled?**

**Answer:**

Yes, it is valid with the C extension. With C enabled, the minimum instruction alignment is 2 bytes, so any even address is a legal instruction start. The address `0x1002` is 2-byte aligned and therefore perfectly legal.

Without the C extension, instructions must be 4-byte aligned, and `0x1002` would be an invalid instruction start address (a jump to it would raise an instruction-address-misaligned exception).

The practical implication for jump targets: `JAL`, `JALR`, and branch targets must be at least 2-byte aligned when C is enabled, versus 4-byte aligned otherwise. The assembler emits the correct encoding and the hardware enforces the appropriate alignment check.

---

### Intermediate Tier

---

**Q6. Encode the instruction `addi a0, a0, -4` as a compressed instruction. Show the bit fields.**

**Answer:**

This maps to `C.ADDI rd, imm` where `rd = a0 (x10)` and `imm = -4`.

`C.ADDI` format (CI format, quadrant 1):
```
[15:13] funct3 = 000
[12]    imm[5]
[11:7]  rd/rs1 (5-bit) = 01010 (x10 = a0)
[6:2]   imm[4:0]
[1:0]   op = 01 (quadrant 1)
```

Immediate -4 in 6-bit signed two's complement: `111100`
- `imm[5] = 1`
- `imm[4:0] = 11100`

Assembling the 16-bit word:
```
15  14  13 | 12 | 11  10  9  8  7 | 6  5  4  3  2 | 1  0
 0   0   0 |  1 |  0  1  0  1  0 | 1  1  1  0  0 | 0  1
```
`= 0b0001_0101_0111_0001 = 0x1571`

Verification: `C.ADDI a0, -4` expands to `ADDI a0, a0, -4`. Sign-extending `111100` gives -4. Correct.

---

**Q7. Explain the alignment implications for branch target offsets in code that mixes 16-bit and 32-bit instructions. Give an example where a 32-bit instruction is at an odd word boundary.**

**Answer:**

With C enabled, instruction addresses are 2-byte aligned. This means:

```
Address 0x1000:  C.LI a0, 5        (2 bytes)
Address 0x1002:  C.LI a1, 3        (2 bytes)
Address 0x1004:  ADD  a2, a0, a1   (4 bytes) -- starts at 4-byte boundary, but need not
Address 0x1008:  C.BEQZ a0, 0x1000 (2 bytes) -- branch target 0x1000 is valid (2-byte aligned)
```

A 32-bit instruction at a 4-byte non-aligned but 2-byte aligned address:

```
Address 0x1000:  C.NOP             (2 bytes)
Address 0x1002:  ADDI a0, a1, 100  (4 bytes) -- 2-byte aligned, NOT 4-byte aligned
Address 0x1006:  C.NOP             (2 bytes)
```

`0x1002` is 2-byte aligned but not 4-byte aligned. With the C extension enabled, this is valid. Without C, a jump to `0x1002` would raise an instruction-address-misaligned exception.

Branch offsets in B-type and J-type instructions: the offset is a signed byte count. The processor PC-adds this offset. The result must be at least 2-byte aligned (with C), which the hardware checks. If the result is misaligned (odd byte address), an instruction-address-misaligned exception is raised.

---

**Q8. What happens when a processor core that does NOT implement the C extension encounters a 16-bit C instruction in the instruction stream?**

**Answer:**

From the non-C core's perspective, `inst[1:0] = 00`, `01`, or `10` means the encoding does not match any valid 32-bit opcode (since all 32-bit encodings have `inst[1:0] = 11`). The core raises an **illegal instruction exception** (`mcause = 2`).

The mismatch causes:
1. The 16-bit half-word is loaded but misidentified as the first half of a 32-bit instruction.
2. The full 32-bit word (first 16-bit chunk + next 16-bit chunk) is decoded and fails the opcode check.
3. An illegal instruction trap fires.

This is why RISC-V toolchains must be told whether C is supported (`-march=rv32imc` vs `-march=rv32im`). Generating C-extension code for a non-C target will cause crashes.

---

**Q9. The `C.SWSP` instruction stores to a stack-relative address. What is its range and granularity, and what 32-bit instruction does it expand to?**

**Answer:**

`C.SWSP rs2, imm(sp)` expands to `SW rs2, imm(sp)`.

Format: CSS (stack-relative store), quadrant 2, `funct3 = 110`.

The immediate field is 6 bits, representing a byte offset scaled by 4 (word granularity):
- 6-bit field × 4 = range 0..252, in steps of 4.
- **Range:** 0 to 252 bytes from `sp`.
- **Granularity:** 4 bytes (word-aligned stores only).

The CSS format encodes `imm[5:2]` in `inst[12:9]` and `imm[7:6]` in `inst[8:7]`. Wait — for C.SWSP the encoding is:

```
[15:13] = 110 (funct3)
[12:9]  = uimm[5:2]
[8:7]   = uimm[7:6]
[6:2]   = rs2
[1:0]   = 10 (quadrant 2)
```

So the immediate is actually 8 bits (uimm[7:2] concatenated), giving range 0-252.

For RV64C, `C.SDSP` uses 9-bit immediate (steps of 8), range 0-504.

This range covers most local variable stacks. A function that pushes more than 252/504 bytes to the stack must use a standard `SW` instruction for out-of-range accesses.

---

**Q10. Explain why `C.MV rd, x0` is an illegal encoding even though `ADD rd, x0, x0` (which is `li rd, 0`) is valid. How should a programmer load zero into a register?**

**Answer:**

`C.MV rd, rs2` is defined to expand to `ADD rd, x0, rs2`. When `rs2 = x0`, this would expand to `ADD rd, x0, x0`, which loads zero into `rd`. However, the C extension explicitly marks `C.MV rd, x0` (i.e., `rs2 = x0`) as **reserved** (illegal encoding).

The reason: the encoding space for `rs2 = x0` in the CR format is repurposed. Specifically, `C.MV rd=0, rs2=0` encodes `C.EBREAK`, and other `rd=0` encodings with non-zero `rs2` are the HINT space. The ISA designers decided that loading zero via `C.MV` would require checking for `rs2 = x0` and issuing a different pseudo-instruction, which complicates the assembler and decoder.

The correct ways to load zero:
```asm
c.li  a0, 0      # C.LI rd, 0  (expands to ADDI rd, x0, 0) -- preferred
```
`C.LI` with immediate 0 is valid and is the idiomatic compressed "load zero" instruction. The assembler emits `C.LI rd, 0` for `li rd, 0` when C is enabled.

---

### Advanced Tier

---

**Q11. A hardware engineer is designing the instruction fetch unit for an RVC-capable processor. Describe the logic needed to handle the case where a 32-bit instruction straddles a cache line boundary, with the first half in one cache line and the second half in the next.**

**Answer:**

This is the "split instruction" or "instruction boundary crossing" problem, a real complexity introduced by 2-byte alignment.

**Scenario:** Cache line is 64 bytes. A 32-bit instruction starts at address `0x003E` (last 2 bytes of cache line ending at `0x003F`). The second 2 bytes are at `0x0040` (first bytes of the next cache line).

**Required fetch unit logic:**

1. **After each fetch, inspect `inst[1:0]`:**
   - If `00`, `01`, or `10`: 16-bit instruction. Increment PC by 2. No problem.
   - If `11`: 32-bit instruction. Need 4 bytes. Check if current 2-byte fetch spans a line.

2. **Cache line boundary detection:** compare `PC[5:1]` (or whatever the line-offset width is). If the 32-bit instruction straddles the boundary:
   - The fetch unit must issue a second cache request for the next line.
   - Stall or buffer the first halfword until the second arrives.

3. **Implementation options:**
   - **Two-halfword buffer:** fetch 2 bytes at a time, buffer in a 32-bit shift register. When a `inst[1:0] == 11` is seen, wait for the next 2 bytes.
   - **Aligned 32-bit fetch, partial-word mux:** fetch 32 bits at a time but allow the start address to be 2-byte aligned. Use a register to hold the "unaligned remainder" from the previous fetch.
   - **Pre-decode buffer:** many industrial cores (e.g., CVA6, Ibex) use a 32-bit or 64-bit fetch buffer and a pre-decode stage that identifies instruction boundaries before issuing to the decode stage.

4. **Additional complication:** if the cache line for `0x0040` is not yet in cache, the second request causes a miss stall. The fetch unit must correctly resume with both halves when the data returns.

This is a known real-world complexity. The Ibex core (lowRISC) documents this in its fetch buffer design. The SiFive U74 handles it with a 64-bit fetch and a pre-decode realignment buffer.

---

**Q12. Walk through the binary encoding of `C.BEQZ a3, -10` and verify it decodes correctly.**

**Answer:**

`C.BEQZ rs1', imm` format (CB format, quadrant 1, `funct3 = 110`):

Mapping: expands to `BEQ rs1, x0, imm`.

- `rs1' = a3 = x13` → 3-bit encoding: `x13 - x8 = 5` → `101`
- `imm = -10` (signed byte offset, must be even)

Immediate encoding in CB format uses a non-contiguous 9-bit field:
```
imm[8|4:3] in inst[12:10]
imm[7:6|2:1|5] in inst[6:2]
```

The offset -10 in 9-bit signed two's complement:
- -10 = 0b1_1111_0110 (9 bits, but only odd-bit positions used for 2-byte alignment)
- Actually: the immediate encodes a byte offset with bit 0 implicit (always 0, since minimum alignment is 2).
- So the 9-bit field encodes imm[8:1]: -10 >> 0 = -10, but bit 0 is always 0 for 2-byte alignment.
- -10 in binary (9 significant bits, bit 0 = 0): 111110110

Bit assignments:
```
imm[8] = 1
imm[7] = 1
imm[6] = 1
imm[5] = 1
imm[4] = 1
imm[3] = 0
imm[2] = 1
imm[1] = 1
imm[0] = 0 (implicit)
```

CB bit layout:
```
[15:13] = 110  (funct3 for C.BEQZ)
[12]    = imm[8]  = 1
[11:10] = imm[4:3] = 10
[9:7]   = rs1' = 101  (a3)
[6:5]   = imm[7:6] = 11
[4:3]   = imm[2:1] = 11
[2]     = imm[5]   = 1
[1:0]   = 01  (quadrant 1)
```

16-bit word: `1101 1010 1111 1101` = `0xDAFD`

Decoding back:
- `inst[1:0] = 01` → C extension, quadrant 1.
- `inst[15:13] = 110` → `C.BEQZ`.
- `inst[9:7] = 101` → rs1' = x13 = a3.
- Reconstruct imm: `{inst[12], inst[6:5], inst[2], inst[11:10], inst[4:3], 0}` = `{1, 11, 1, 10, 11, 0}` = `111_1_10_11_0` = `1_1111_0110` in 9 bits = -10. Correct.

---

**Q13. A linker is generating code for a shared library. How does the C extension affect position-independent code (PIC) generation, particularly for GOT-relative and PC-relative accesses?**

**Answer:**

The C extension affects PIC in several ways:

**PC-relative addressing range:**
- `C.J` and `C.JAL` have an 11-bit signed offset (±1 KB). This is smaller than `JAL`'s 20-bit offset (±1 MB). When generating calls to PLT stubs, the linker must ensure the PLT stubs are within ±1 KB of the call site to use `C.J`; otherwise it falls back to `JAL` (32-bit).
- `C.BEQZ`/`C.BNEZ` have a 9-bit offset (±256 bytes). Branches to labels more than 256 bytes away cannot use compressed branches.

**GOT access:**
- GOT (Global Offset Table) entries are addressed via `AUIPC + LW` (load upper immediate relative + load). There is no compressed form of `AUIPC`, so GOT accesses always use 32-bit instructions. The C extension does not help here.

**Thunk insertion:**
- When a branch target is out of the compressed range, the compiler/linker inserts a thunk: a nearby unconditional jump (possibly `C.J` itself if within ±1 KB) that redirects to the real target. This adds an indirection.

**Alignment padding:**
- With PIC, functions may require alignment to 4 or 8 bytes for performance (e.g., loop alignment). The linker inserts `C.NOP` (2 bytes) rather than `NOP` (4 bytes) when fine-grained padding is needed.

**Practical impact:**
- The GCC linker relaxation pass (`--relax`) rewrites `AUIPC + ADDI` sequences to `C.LI` or `C.ADDI` when the immediate fits after final address assignment. This is performed after initial linking.

---

**Q14. Explain the HINT encoding space in the C extension. Give two examples of defined HINTs and describe how a processor should handle an unrecognised HINT.**

**Answer:**

A HINT instruction is a legal encoded instruction that performs no architectural state change (it is a no-op), but may carry information for microarchitectural optimisation (e.g., prefetch hints, execution hints).

In the C extension, several encodings are designated as HINTs rather than being illegal:

- `C.NOP` (canonical): `C.ADDI x0, x0, 0` — the architectural NOP. Always safe, always a no-op.
- `C.ADDI rd, 0` for `rd != x0`: adding zero to a register is a no-op architecturally, so these encodings are HINTs. A microarchitecture could use the `rd` field to indicate a register that should be prefetched or flagged for branch-predictor warmup.
- `C.LUI x0, imm`: `rd = x0`, so the result is discarded — this is a HINT.
- `C.MV x0, rs2` for `rs2 != x0`: writes to x0 (discarded) — HINT space.

**Defined HINT examples:**
1. `C.NOP` with non-zero immediate: `C.ADDI x0, imm` where `imm != 0`. The spec says this is a HINT. Some custom cores use the immediate field to pass micro-architectural advice (e.g., loop count for prefetch).
2. `C.ADD x0, rs2`: result written to `x0` — a HINT. The `rs2` field could identify a register whose value should be cache-prefetched.

**Handling unrecognised HINTs:**
An implementation that does not recognise a HINT must treat it as a NOP — **not** as an illegal instruction. This is the critical distinction from illegal encodings. The implementation must:
1. Advance the PC correctly (by 2 bytes for a 16-bit HINT).
2. Make no architectural state change.
3. Not trap.

This forward-compatibility guarantee means new HINT definitions can be added in future extensions without breaking existing software running on older cores.
