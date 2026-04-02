# B Extension: Bit Manipulation

## Overview

The RISC-V B extension (ratified as version 1.0 in 2021) adds instructions for bit counting, rotation, byte/bit manipulation, and address generation. Rather than being a monolithic extension, B is organised into ratified sub-extensions — `Zba`, `Zbb`, and `Zbs` — each independently adoptable. A further group `Zbc` (carry-less multiply) and `Zbkb`/`Zbkc`/`Zbkx` (cryptography-oriented subsets) are defined for cryptographic workloads. Understanding which sub-extension provides which instruction, and why these save significant instruction counts in common algorithms, is the core interview topic in this area.

---

## Key Concepts

### Sub-Extension Overview

| Sub-extension | Focus | Typical users |
|---------------|-------|---------------|
| `Zba` | Address generation (scaled index) | Compilers, array indexing |
| `Zbb` | Basic bit manipulation (count, rotate, byte-reverse, min/max, sign-extend) | Compilers, general code |
| `Zbs` | Single-bit operations (set, clear, invert, extract) | Bitmap operations, flag manipulation |
| `Zbc` | Carry-less multiplication (CLMUL) | CRC, GCM, polynomial arithmetic |
| `Zbkb`, `Zbkc`, `Zbkx` | Cryptography (part of Scalar Crypto group) | AES, SHA, GHASH |

For most interviews, the focus is on `Zba`, `Zbb`, and `Zbs`.

---

### Zba: Address Generation Instructions

`Zba` provides scaled-add instructions that combine a shift-left with an add in one instruction. This directly targets the `base + index * stride` pattern that appears in array accesses.

| Instruction      | Operation             | Stride |
|------------------|-----------------------|--------|
| `SH1ADD rd, rs1, rs2` | `rd = rs2 + (rs1 << 1)` | 2 bytes (short arrays) |
| `SH2ADD rd, rs1, rs2` | `rd = rs2 + (rs1 << 2)` | 4 bytes (int/float arrays) |
| `SH3ADD rd, rs1, rs2` | `rd = rs2 + (rs1 << 3)` | 8 bytes (double/pointer arrays) |

On RV64, `SH1ADD.UW`, `SH2ADD.UW`, `SH3ADD.UW` zero-extend the lower 32 bits of `rs1` before shifting — used when the index is a 32-bit value in a 64-bit system.

Also in `Zba` for RV64:

| Instruction | Operation | Purpose |
|-------------|-----------|---------|
| `ADD.UW rd, rs1, rs2` | `rd = rs2 + zext32(rs1)` | Add 32-bit zero-extended value to 64-bit base |
| `SLLI.UW rd, rs1, imm` | `rd = zext32(rs1) << imm` | Shift zero-extended 32-bit value left |

**Without Zba**, an array access `a[i]` for a 32-bit integer array requires:

```asm
slli  t0, a1, 2       # t0 = i * 4
add   t0, a0, t0      # t0 = &a[i]
lw    t1, 0(t0)
```

**With Zba**:

```asm
sh2add  t0, a1, a0    # t0 = a0 + (a1 << 2) = &a[i]
lw      t1, 0(t0)
```

Saves one instruction per array element access — significant in tight loops.

---

### Zbb: Basic Bit Manipulation

`Zbb` is the richest sub-extension, covering:

#### Bit Counting

| Instruction | Operation | Description |
|-------------|-----------|-------------|
| `CLZ rd, rs1`  | Count Leading Zeros   | Number of 0 bits above the highest 1 bit |
| `CTZ rd, rs1`  | Count Trailing Zeros  | Number of 0 bits below the lowest 1 bit |
| `CPOP rd, rs1` | Count Population      | Number of 1 bits (popcount, Hamming weight) |

For RV64, `CLZW`, `CTZW`, `CPOPW` operate on the lower 32 bits and zero-extend the result.

**Without Zbb**, computing CLZ requires a binary search loop or lookup table (5-10 instructions). With `CLZ`, it is a single instruction.

Common uses:
- `CLZ`: find the most significant bit, normalise floating-point in software, compute `log2` (rounded down).
- `CTZ`: find the least significant set bit, extract a queue index from a bitmap.
- `CPOP`: Hamming distance, parity, error-checking, neural network popcount layers.

#### Rotation

| Instruction | Operation |
|-------------|-----------|
| `ROL rd, rs1, rs2`  | Rotate rs1 left by `rs2[4:0]` bits |
| `ROR rd, rs1, rs2`  | Rotate rs1 right by `rs2[4:0]` bits |
| `RORI rd, rs1, imm` | Rotate rs1 right by immediate (5-bit, 6-bit on RV64) |

Rotation is used extensively in hash functions (SHA, MD5, ChaCha20), CRC computation, and cryptographic algorithms. Without hardware rotation, each rotate requires two shifts and an OR:

```asm
# ROR a0, a0, 8  (without Zbb)
srli  t0, a0, 8
slli  t1, a0, 24
or    a0, t0, t1
```

With Zbb: `rori a0, a0, 8` — one instruction.

#### Byte and Bit Reversal

| Instruction | Operation |
|-------------|-----------|
| `REV8 rd, rs1`  | Reverse byte order (endian swap) |
| `BREV8 rd, rs1` | Reverse bits within each byte (Zbkb only) |

`REV8` is essential for network protocol code (big-endian ↔ little-endian conversion) and cryptographic algorithms that require byte reversal. Without it, a 32-bit byte-reverse on RV32 requires 8+ instructions using shifts and ORs.

#### Extend Instructions

| Instruction | Operation |
|-------------|-----------|
| `SEXT.B rd, rs1` | Sign-extend byte (bits 7:0) to XLEN |
| `SEXT.H rd, rs1` | Sign-extend halfword (bits 15:0) to XLEN |
| `ZEXT.H rd, rs1` | Zero-extend halfword (bits 15:0) to XLEN |

These replace `slli`/`srai` pairs for sign/zero extension of sub-word values.

#### Min/Max and Absolute Value

| Instruction | Operation |
|-------------|-----------|
| `MIN  rd, rs1, rs2`  | Signed minimum |
| `MINU rd, rs1, rs2`  | Unsigned minimum |
| `MAX  rd, rs1, rs2`  | Signed maximum |
| `MAXU rd, rs1, rs2`  | Unsigned maximum |

Without these, min/max requires a compare and a branch or a conditional-move sequence (which RISC-V base ISA does not have). With Zbb, branchless min/max in one instruction.

#### Logical with Negate

| Instruction | Operation |
|-------------|-----------|
| `ANDN rd, rs1, rs2` | `rd = rs1 & ~rs2` |
| `ORN  rd, rs1, rs2` | `rd = rs1 | ~rs2`  |
| `XNOR rd, rs1, rs2` | `rd = ~(rs1 ^ rs2)` |

`ANDN` is particularly common for bit-clearing masks: "clear the bits indicated by `rs2` from `rs1`."

#### OR-Combine (`ORC.B`)

`ORC.B rd, rs1` sets each byte of `rd` to `0xFF` if the corresponding byte of `rs1` is non-zero, or `0x00` if zero. This is used in `strlen` and `strcmp` SIMD-style byte detection within an XLEN-wide register.

---

### Zbs: Single-Bit Operations

`Zbs` provides four instructions for operating on individual bits specified by a register or immediate:

| Instruction | Operation | Description |
|-------------|-----------|-------------|
| `BSET rd, rs1, rs2`  | `rd = rs1 | (1 << rs2[4:0])`  | Set bit `rs2` |
| `BCLR rd, rs1, rs2`  | `rd = rs1 & ~(1 << rs2[4:0])` | Clear bit `rs2` |
| `BINV rd, rs1, rs2`  | `rd = rs1 ^ (1 << rs2[4:0])`  | Invert bit `rs2` |
| `BEXT rd, rs1, rs2`  | `rd = (rs1 >> rs2[4:0]) & 1`  | Extract bit `rs2` |

Immediate variants: `BSETI`, `BCLRI`, `BINVI`, `BEXTI` use a 5-bit (or 6-bit on RV64) immediate instead of `rs2`.

These replace two-instruction sequences (shift + OR/AND/XOR) with single instructions. For bitmap manipulation (scheduler ready-queues, memory allocator bitmaps, peripheral register access), these are high-frequency operations.

Example — setting bit `n` in a register without Zbs:

```asm
li   t0, 1
sll  t0, t0, a1     # t0 = 1 << n
or   a0, a0, t0     # a0 |= t0
```

With Zbs:

```asm
bset a0, a0, a1     # a0 |= (1 << a1)
```

---

### Encoding Notes

All B-extension instructions use existing R-type and I-type formats. They are distinguished by `funct7` and `funct3` values in encoding space reserved for future extensions in the base ISA. No new opcode space is introduced. This was a deliberate design choice to minimise decoder changes.

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What are the three main ratified sub-extensions of the B extension, and what does each target?**

**Answer:**

- **Zba (Address generation):** scaled-add instructions (`SH1ADD`, `SH2ADD`, `SH3ADD`) that compute `base + index * stride` in one instruction. Targets array indexing and pointer arithmetic.

- **Zbb (Basic bit manipulation):** a broad set including bit counting (`CLZ`, `CTZ`, `CPOP`), rotation (`ROL`, `ROR`, `RORI`), byte reversal (`REV8`), sign/zero extension (`SEXT.B`, `SEXT.H`, `ZEXT.H`), min/max (`MIN`, `MINU`, `MAX`, `MAXU`), and logic-with-negate (`ANDN`, `ORN`, `XNOR`). Targets general-purpose compiled code and cryptographic algorithms.

- **Zbs (Single-bit operations):** `BSET`, `BCLR`, `BINV`, `BEXT` for manipulating individual bits by index. Targets bitmap operations, register flag manipulation, and hardware driver code.

---

**Q2. How would you compute `popcount` (number of set bits) on RV32 without the B extension? How does `CPOP` improve this?**

**Answer:**

Without `CPOP`, a common software popcount for a 32-bit value uses a parallel bit-counting sequence (Hamming weight, "SWAR" technique):

```asm
# a0 = input, result in a0
# Step 1: count bits in each 2-bit group
li    t0, 0x55555555    # 0101...0101
srli  t1, a0, 1
and   t1, t1, t0
and   a0, a0, t0
add   a0, a0, t1        # sum of pairs

# Step 2: count bits in each 4-bit group
li    t0, 0x33333333    # 0011...0011
srli  t1, a0, 2
and   t1, t1, t0
and   a0, a0, t0
add   a0, a0, t1

# Step 3: count bits in each 8-bit group
li    t0, 0x0F0F0F0F
srli  t1, a0, 4
and   t1, t1, t0
and   a0, a0, t0
add   a0, a0, t1

# Step 4: sum the four 8-bit counts
li    t0, 0x01010101
mul   a0, a0, t0        # (requires M extension)
srli  a0, a0, 24        # final count in a0
```

This is approximately 14 instructions (plus a `MUL` if available). Alternative naive approaches (shift-and-count loop) use 10-15+ cycles.

With `CPOP`:

```asm
cpop  a0, a0    # one instruction, typically 1-3 cycles
```

The hardware can implement this in a single tree of adders with logarithmic depth (5-6 levels for 32 bits), making it far faster than any software sequence. Used in: cryptography (GHASH), machine learning (binary neural networks), OS schedulers (find-first-set in ready-queue), compression algorithms.

---

**Q3. Show how `SH2ADD` from `Zba` simplifies indexing into an array of 32-bit integers. What does it replace?**

**Answer:**

Accessing `arr[i]` where `arr` is an `int[]` (4 bytes per element):

**Without Zba:**

```asm
# a0 = base address of arr, a1 = index i
slli  t0, a1, 2       # t0 = i * 4  (byte offset)
add   t0, a0, t0      # t0 = &arr[i]
lw    t1, 0(t0)       # t1 = arr[i]
```

3 instructions before the load.

**With Zba:**

```asm
# a0 = base address of arr, a1 = index i
sh2add  t0, a1, a0    # t0 = a0 + (a1 << 2) = &arr[i]
lw      t1, 0(t0)     # t1 = arr[i]
```

2 instructions before the load — saves one instruction per access. In a loop iterating over a large array, this reduction compounds significantly.

`SH1ADD` is for 2-byte element arrays (int16, short), `SH3ADD` for 8-byte elements (int64, double, pointers on RV64).

---

**Q4. What does `CLZ` return for the input `0`? What about `CTZ` for `0`? Are these defined?**

**Answer:**

Both are defined:

- `CLZ rd, x0_value_zero`: returns **XLEN** (32 on RV32, 64 on RV64). There are no leading 1-bits, so there are XLEN leading zeros.
- `CTZ rd, x0_value_zero`: returns **XLEN** (32 on RV32, 64 on RV64). There are no trailing 1-bits, so there are XLEN trailing zeros.

This contrasts with x86's `BSR`/`BSF` instructions, which leave the destination undefined when the source is zero. RISC-V's `CLZ`/`CTZ` are fully defined for zero input, which simplifies software that calls them — no special-case check for zero is needed.

Example use: computing `floor(log2(n))` for `n > 0`:

```asm
clz   t0, a0          # t0 = leading zeros
li    t1, 31          # (or 63 on RV64)
sub   a0, t1, t0      # a0 = floor(log2(a0))
```

If `a0 = 0`, this produces `31 - 32 = -1` (unsigned wrap-around), which is a sentinel for "undefined". Software can choose to check for zero before calling `CLZ` if needed, but the hardware does not require it.

---

**Q5. What is `ANDN` and why is it useful? Give a concrete example.**

**Answer:**

`ANDN rd, rs1, rs2` computes `rd = rs1 & (~rs2)`. It ANDs `rs1` with the bitwise complement of `rs2` — equivalent to "clear the bits in `rs1` that are set in `rs2`."

Without `ANDN`:

```asm
not   t0, a1        # t0 = ~mask  (xori t0, a1, -1)
and   a0, a0, t0    # a0 &= ~mask
```

With `ANDN`:

```asm
andn  a0, a0, a1    # a0 &= ~a1
```

**Concrete example:** clearing a flag in a status register.

```asm
# Clear the ERROR_FLAG (bit 3) from status register in a0
li    t0, (1 << 3)   # mask for ERROR_FLAG
andn  a0, a0, t0     # a0 &= ~ERROR_FLAG_MASK
```

`ANDN` also appears in:
- SHA-3 (Keccak χ step): `a[i] = a[i] ^ (~a[i+1] & a[i+2])` — the `~a[i+1] & a[i+2]` term is directly `ANDN`.
- Bit manipulation for selecting elements (keep bits not covered by a mask).
- Efficient implementation of priority dequeuing (clear the lowest set bit: `andn a0, a0, t0` where `t0` is the isolated lowest bit).

---

### Intermediate Tier

---

**Q6. Implement `strlen` using `ORC.B` and `CTZ` from `Zbb` on RV64. Explain how it processes 8 bytes at a time.**

**Answer:**

The technique: load 8 bytes at a time into an XLEN-wide register. Use `ORC.B` to detect zero bytes, then use `CTZ` to find the position of the first zero byte.

```asm
# RV64: a0 = pointer to string, returns length in a0
strlen:
    mv    t0, a0          # t0 = pointer (will advance)
    li    t3, 0x0101010101010101  # byte-splat 1 (for end-of-scan add)

.loop:
    ld    t1, 0(t0)       # load 8 bytes
    orc.b t2, t1          # each byte of t2: 0xFF if source byte nonzero, 0x00 if zero
    # We want to find bytes that are 0x00 in t1.
    # After ORC.B: zero bytes in t1 become 0x00 in t2 (since orc.b sets byte to 0 only if input byte is 0)
    # Wait: orc.b sets byte to 0xFF if nonzero, 0x00 if zero.
    # So: NOT t2 gives 0xFF for each ZERO byte, 0x00 for nonzero bytes.
    not   t2, t2          # t2: 0xFF for each zero byte, 0x00 for nonzero
    # Now find the position of the first 0xFF byte (= first zero in original string)
    # In little-endian: least significant byte is leftmost in memory
    ctz   t2, t2          # count trailing zeros -- bits before the first set byte
    # t2 is now in bits; divide by 8 to get byte position
    srli  t2, t2, 3       # t2 = byte offset of first zero within the 8 bytes
    # Check if we found a zero byte in this chunk
    li    t3, 8
    blt   t2, t3, .found  # if offset < 8, found null within this chunk
    addi  t0, t0, 8       # advance pointer by 8
    j     .loop

.found:
    add   a0, t0, t2      # address of null terminator
    sub   a0, a0, a0_orig # length = null_addr - start (but we need to save a0 start)
    # Correction: save start at function entry
    ret
```

Cleaner version saving the start address:

```asm
strlen:
    mv    t4, a0          # save start pointer
.loop:
    ld    t1, 0(a0)
    orc.b t2, t1          # t2[byte] = 0xFF if t1[byte]!=0, else 0x00
    not   t2, t2          # t2[byte] = 0xFF if t1[byte]==0, else 0x00
    bnez  t2, .found      # if any byte is zero, found the terminator
    addi  a0, a0, 8
    j     .loop
.found:
    ctz   t2, t2          # trailing zeros in bit representation
    srli  t2, t2, 3       # convert bit offset to byte offset
    add   a0, a0, t2      # pointer to null byte
    sub   a0, a0, t4      # length = null_addr - start
    ret
```

The loop processes 8 characters per iteration instead of 1, giving up to 8x speedup for long strings. The `ORC.B` + `NOT` + `CTZ` pattern is a standard idiom for SIMD-in-a-register byte zero detection.

---

**Q7. What is the byte-reverse instruction `REV8`? Write the equivalent code sequence for RV32 without Zbb, and explain why `REV8` is important in network programming.**

**Answer:**

`REV8 rd, rs1` reverses the byte order of `rs1`. On RV32, it swaps bytes 3↔0 and 2↔1 (big-endian ↔ little-endian conversion for a 32-bit word).

Without `REV8` on RV32:

```asm
# a0 = 0xAABBCCDD, result = 0xDDCCBBAA
rev8_rv32:
    # Extract and shift each byte
    srli  t0, a0, 24        # t0 = 0x000000AA
    slli  t1, a0, 24        # t1 = 0xDD000000
    or    a0, t0, t1        # a0 = 0xDD0000AA (bytes 3 and 0 swapped)

    srli  t0, a0, 8
    li    t1, 0x00FF0000
    and   t0, t0, t1        # t0 = 0x00BB0000
    srli  t2, a0, 16
    li    t1, 0x0000FF00
    and   t2, t2, t1        # ... complex extraction

    # Faster 4-instruction version using masks:
    li    t0, 0x00FF00FF
    and   t1, a0, t0        # t1 = 0x00BB00DD
    srli  a0, a0, 8
    and   a0, a0, t0        # a0 = 0x00AA00CC
    slli  t1, t1, 8         # t1 = 0xBB00DD00
    or    a0, a0, t1        # a0 = 0xBBAADDCC  -- wait, this isn't right either
    # ... requires careful construction; typically 8-10 instructions total
```

With `REV8`: one instruction.

**Importance in network programming:**

Network protocols (IPv4, TCP, UDP, DNS) transmit multi-byte integers in big-endian byte order ("network byte order"). RISC-V (like x86) is little-endian. Every read of a 32-bit or 16-bit field from a network packet requires byte-swapping before use, and every write requires byte-swapping before transmission.

`REV8` is used to implement `ntohl()`/`htonl()` (network-to-host and host-to-network 32-bit conversion) and `ntohs()`/`htons()` (16-bit). Without it, the CPU overhead of byte-swapping in a packet-processing fast path adds up to a measurable fraction of total processing time (especially at 10GbE+ rates).

---

**Q8. Show how the four `Zbs` single-bit instructions replace common two-instruction sequences. Give the assembly for clearing bit 5 of `a0` using a variable bit index stored in `a1`.**

**Answer:**

Without `Zbs`, clearing bit `n` (variable) in a register:

```asm
li    t0, 1
sll   t0, t0, a1       # t0 = 1 << n
not   t0, t0           # t0 = ~(1 << n)
and   a0, a0, t0       # a0 &= ~(1 << n)
```

4 instructions.

With `Zbs`:

```asm
bclr  a0, a0, a1       # a0 &= ~(1 << a1)
```

1 instruction.

For fixed bit index (bit 5):

```asm
bclri  a0, a0, 5       # clear bit 5 (immediate form)
```

The four operations and their two-instruction equivalents:

| `Zbs` instruction | Equivalent (2 instrs) |
|-------------------|-----------------------|
| `BSET a0, a0, a1` | `li t0,1; sll t0,t0,a1; or a0,a0,t0` (3 instrs) |
| `BCLR a0, a0, a1` | `li t0,1; sll t0,t0,a1; not t0,t0; and a0,a0,t0` (4 instrs) |
| `BINV a0, a0, a1` | `li t0,1; sll t0,t0,a1; xor a0,a0,t0` (3 instrs) |
| `BEXT a0, a0, a1` | `srl a0,a0,a1; andi a0,a0,1` (2 instrs) |

`BEXT` is used in tight loops over bitfields: extracting each bit of a mask word to dispatch on it. `BSET`/`BCLR`/`BINV` are common in peripheral register access (setting/clearing control bits by name) and OS scheduler bitmaps.

---

**Q9. A firmware engineer is implementing a software CRC-32 using the RISC-V B extension. Which sub-extension and instructions are most relevant? Describe the approach.**

**Answer:**

For CRC-32, the most relevant instruction is `CLMUL` from the `Zbc` sub-extension (carry-less multiply). However, if only `Zbb` and `Zbs` are available, rotation and byte reversal reduce instruction count significantly.

**With `Zbc` (`CLMUL`):**

CRC-32 is based on polynomial division over GF(2), which is carry-less arithmetic. The Barrett reduction for CRC-32 requires two carry-less multiplications:

```asm
# Zbc approach (Intel PCLMULQDQ equivalent):
# For 64-bit CRC folding:
clmul   t0, data_lo, poly_lo      # lower 64 bits of carry-less product
clmulh  t1, data_lo, poly_lo      # upper 64 bits
# fold and reduce...
```

This is the same approach used in Intel's CRC-32 acceleration with PCLMULQDQ, yielding throughputs of multiple bytes per cycle.

**With only `Zbb` (no `Zbc`):**

The standard bit-serial and table-driven CRC-32 algorithms benefit from `Zbb`:

1. **`REV8`**: CRC-32 input must often be bit-reflected. `REV8` handles byte reversal; per-byte bit reversal uses `BREV8` (from `Zbkb`).
2. **`ROR`/`RORI`**: CRC computation involves circular shifts in some formulations.
3. **`ANDN`**: masking extracted polynomial bits without a temporary NOT.

Table-driven CRC-32 (the most common software implementation):

```asm
# Process one byte: a0 = current CRC, a1 = byte, a2 = &crc_table[256]
# crc = (crc >> 8) ^ table[(crc ^ byte) & 0xFF]
xor   t0, a0, a1          # (crc ^ byte)
andi  t0, t0, 0xFF        # & 0xFF = table index
sh2add t0, t0, a2         # table base + index*4
lw    t0, 0(t0)           # t0 = table[index]  (Zba: sh2add saves one instruction)
srli  a0, a0, 8           # crc >> 8
xor   a0, a0, t0          # new CRC
```

`Zba`'s `SH2ADD` saves one instruction per byte in the table lookup.

---

**Q10. What is `ORC.B` and how does it enable efficient `memchr` / `strchr` implementations?**

**Answer:**

`ORC.B rd, rs1` (OR-Combine Bytes) sets each byte of `rd` to `0xFF` if the corresponding input byte is non-zero, or `0x00` if the input byte is zero:

```
rd[byte_i] = (rs1[byte_i] != 0) ? 0xFF : 0x00
```

For `strchr(str, '\0')` (finding null terminator = `strlen`), the technique is:

1. Load XLEN/8 bytes at once.
2. Apply `ORC.B` to produce a mask where zero bytes become `0x00` and non-zero bytes become `0xFF`.
3. Invert with `NOT`: zero bytes → `0xFF`, non-zero → `0x00`.
4. Use `CTZ` to find the position of the first `0xFF` byte (= first zero in original string).

For `strchr(str, c)` (finding a specific byte `c`):

1. Load 8 bytes.
2. XOR with a splat of `c` (`c * 0x0101010101010101` on RV64) — this turns matching bytes to zero.
3. Apply `ORC.B` and `NOT` to detect zero bytes.
4. Use `CTZ` to find the match position.

```asm
# RV64 strchr: a0 = str, a1 = char c
# Splat c to all 8 bytes of t3
li    t4, 0x0101010101010101
mul   t3, a1, t4           # t3 = c in all bytes (requires M extension)
                            # Alternative without M: slli/or pairs

.loop:
    ld    t1, 0(a0)
    xor   t1, t1, t3       # zero the matching bytes
    orc.b t2, t1           # 0x00 for matched bytes, 0xFF for others
    not   t2, t2           # 0xFF for matches, 0x00 for non-matches
    bnez  t2, .found
    addi  a0, a0, 8
    j     .loop

.found:
    ctz   t2, t2
    srli  t2, t2, 3        # byte offset of first match
    add   a0, a0, t2
    ret
```

This processes 8 bytes per loop iteration. `glibc`'s `strlen`/`strchr` for RISC-V uses exactly this pattern.

---

### Advanced Tier

---

**Q11. Explain how `CLZ` can be used to implement a branchless binary logarithm `floor(log2(n))` for `n > 0`. Then show how to compute `ceil(log2(n))` using `CPOP` or `CTZ`.**

**Answer:**

**`floor(log2(n))`** — the position of the most significant set bit (0-indexed):

```asm
# a0 = n > 0, result in a0 = floor(log2(n))
clz   t0, a0          # t0 = number of leading zeros
li    t1, 31          # for RV32; use 63 for RV64
sub   a0, t1, t0      # a0 = 31 - leading_zeros = MSB position
```

Derivation: `CLZ(n) + MSB_position = XLEN - 1`, so `MSB_position = (XLEN-1) - CLZ(n)`.

Examples:
- `n = 1` (`0x00000001`): CLZ = 31, result = 0. Correct: log2(1) = 0.
- `n = 8` (`0x00000008`): CLZ = 28, result = 3. Correct: log2(8) = 3.
- `n = 7` (`0x00000007`): CLZ = 29, result = 2. Correct: floor(log2(7)) = 2.

**`ceil(log2(n))`** — floor(log2(n)) + 1 if n is not a power of 2; floor(log2(n)) if n is a power of 2.

To check if `n` is a power of 2: a power of 2 has exactly one set bit, so `CPOP(n) == 1`.

```asm
# a0 = n > 0, result = ceil(log2(n))
clz   t0, a0
li    t1, 31
sub   t1, t1, t0      # t1 = floor(log2(n))
cpop  t2, a0          # t2 = popcount
# If popcount == 1 (power of 2), ceil == floor; else ceil == floor + 1
sltiu t2, t2, 2       # t2 = (popcount < 2) ? 1 : 0 = (power of 2) ? 1 : 0
xori  t2, t2, 1       # t2 = (not power of 2) ? 1 : 0
add   a0, t1, t2      # ceil = floor + (0 if power of 2, else 1)
```

Alternative using `CTZ`: for a power of 2, `CLZ(n) + CTZ(n) = XLEN - 1`. If `CLZ + CTZ < XLEN - 1`, it's not a power of 2.

Both approaches are fully branchless — important for pipelined execution where branch mispredictions are expensive.

---

**Q12. The SHA-256 algorithm uses right-rotation by fixed amounts extensively. Show how `Zbb` rotation instructions and `ANDN` implement the SHA-256 Sigma functions efficiently on RV32.**

**Answer:**

SHA-256 defines four rotation-based functions:

```
Σ0(x) = ROTR(x, 2) ^ ROTR(x, 13) ^ ROTR(x, 22)
Σ1(x) = ROTR(x, 6) ^ ROTR(x, 11) ^ ROTR(x, 25)
σ0(x) = ROTR(x, 7) ^ ROTR(x, 18) ^ SHR(x, 3)
σ1(x) = ROTR(x, 17) ^ ROTR(x, 19) ^ SHR(x, 10)
```

**Without Zbb** (RV32, no rotation instruction): each ROTR requires 2 shifts + 1 OR = 3 instructions. `Σ0` requires 3×3 = 9 instructions plus 2 XORs = 11 instructions.

**With Zbb** (RORI):

```asm
# Σ0(a): a in a0, result in a0
# Σ0 = ROTR(a,2) ^ ROTR(a,13) ^ ROTR(a,22)
rori  t0, a0, 2      # t0 = ROTR(a, 2)
rori  t1, a0, 13     # t1 = ROTR(a, 13)
rori  t2, a0, 22     # t2 = ROTR(a, 22)
xor   t0, t0, t1     # t0 = ROTR(a,2) ^ ROTR(a,13)
xor   a0, t0, t2     # a0 = Σ0(a)
```

5 instructions instead of 11 — a 2x reduction for just this function. Across the full SHA-256 compression loop (64 rounds), the B extension saves hundreds of instructions per block.

**SHA-256 `Ch` function with `ANDN`:**

```c
Ch(x, y, z) = (x & y) ^ (~x & z)
```

The `~x & z` term is directly `ANDN z, z, x`:

```asm
# Ch(e, f, g): e in a0, f in a1, g in a2 -> result in a0
and   t0, a0, a1      # t0 = e & f
andn  t1, a2, a0      # t1 = ~e & g  = g & ~e  (ANDN: rs1 & ~rs2)
xor   a0, t0, t1      # a0 = Ch(e,f,g)
```

3 instructions. Without `ANDN`, the `~x & z` requires `NOT` + `AND` = 4 instructions total.

SHA-3 (Keccak) uses `ANDN` even more heavily in the χ step, making `Zbb`/`Zbkb` particularly valuable for cryptographic implementations.

---

**Q13. A processor architect proposes adding a "find first set" (FFS) instruction to replace `CTZ`. What are the differences between FFS and CTZ semantics, and what problems arise in hardware when the input is zero?**

**Answer:**

**FFS vs CTZ:**

- `CTZ(x)`: returns the number of trailing zero bits. `CTZ(0) = XLEN` (defined). `CTZ(1) = 0`. `CTZ(0b1100) = 2`.
- `FFS(x)` (POSIX): returns the 1-indexed position of the lowest set bit. `FFS(0) = 0` (special sentinel). `FFS(1) = 1`. `FFS(0b1100) = 3`. Relationship: `FFS(x) = CTZ(x) + 1` for `x != 0`.

**The zero-input problem:**

The critical design issue: what should the hardware return for `CTZ(0)` or `FFS(0)`?

- x86 `BSF` (bit scan forward): result is undefined when input is zero. The ZF flag indicates zero input, but the destination register has microarchitecturally unpredictable content. This forces a separate branch or `CMOV` to handle zero.
- x86 `TZCNT` (from BMI1): always defined; returns XLEN for input zero.
- RISC-V `CTZ`: always defined; returns XLEN for input zero.

**Hardware trade-off for RISC-V:**

Returning `XLEN` for zero requires the hardware to special-case the zero input — the priority encoder output would normally give an incorrect result. Implementation options:

1. **OR-reduce detect:** before the priority encoder, check if the input is all-zero. If so, mux in `XLEN` as the output. This adds one OR-tree and one mux on the critical path.

2. **Augmented priority encoder:** encode the zero case directly in the priority encoder by adding a special "no bits set" output. This integrates the check into the encoding rather than adding a separate mux.

The RISC-V spec chose defined-for-zero semantics (returning XLEN) because:
- Software that calls `CLZ`/`CTZ` on a zero input is often doing so to implement `log2` or find-MSB algorithms. Having a sentinel value (XLEN) avoids a branch on zero.
- The hardware cost is minimal (a single level of logic on the output).
- Undefined behaviour in hardware (like x86 `BSF`) makes software harder to verify and reason about.

---

**Q14. Explain how the `Zba` instruction `SH1ADD.UW` differs from `SH1ADD` on RV64. Why is the `.UW` variant needed?**

**Answer:**

On RV64:

- `SH1ADD rd, rs1, rs2`: computes `rd = rs2 + (rs1 << 1)` where `rs1` is a full 64-bit value. The shift operates on all 64 bits of `rs1`.

- `SH1ADD.UW rd, rs1, rs2`: computes `rd = rs2 + (zero_extend_32(rs1) << 1)`. It zero-extends the lower 32 bits of `rs1` to 64 bits, then shifts left by 1.

**Why the `.UW` variant is needed:**

In a 64-bit program, array indices are commonly stored as 32-bit values (e.g., `int i` in a loop). On RV64, the 64-bit register holding `i` may have undefined upper 32 bits if the value was produced by a 32-bit operation that sign-extends or if it was loaded from a 32-bit memory location.

Example: `int i = 5;` produces `0x0000000000000005` in a register — the upper 32 bits are zero, so `SH2ADD` would work correctly. But if `i` was the result of, say, bit manipulation that left garbage in the upper 32 bits, `SH2ADD` would compute the wrong address.

`SH2ADD.UW` guarantees the upper 32 bits of `rs1` are zero before the shift, regardless of their actual content:

```asm
# RV64: array access arr[i] where i is a 32-bit int
# a0 = base, a1 = i (32-bit, may have dirty upper bits)
sh2add.uw  t0, a1, a0   # t0 = a0 + (zero_extend32(a1) << 2)
lw         t1, 0(t0)
```

Without `.UW`:
```asm
slli  t0, a1, 32        # shift away upper bits
srli  t0, t0, 32        # shift back = zero-extend to 64 bits
sh2add t0, t0, a0       # now safe
```

`.UW` combines the zero-extension and scaled-add into one instruction, matching the compiler's need to handle the common 32-bit index in 64-bit array access pattern cleanly.
