# Problem 02: Sv39 Page Table Walk

**Topic:** Privilege Architecture — Sv39 virtual memory, page table walks  
**Difficulty:** Advanced  
**Estimated time:** 35 minutes

---

## Problem Statement

A Linux RISC-V process is running on an RV64 processor configured with Sv39 virtual memory. The `satp` CSR is:

```
satp = 0x8000_0000_0008_0001
```

The page tables for this process have been set up in physical memory as follows. All values are given as 64-bit physical memory contents at the stated addresses.

**Root page table (level-2) at physical address `0x8000_1000`:**

| Byte offset | PTE value |
|-------------|-----------|
| 0x000       | `0x0000_0000_2000_2001` |
| 0x008       | `0x0000_0000_0000_0000` |
| ...         | (all other entries are 0x0) |
| 0xFF8       | `0x0000_0000_2000_4001` |

**Level-1 page table A at physical address `0x8000_8000`:**

| Byte offset | PTE value |
|-------------|-----------|
| 0x000       | `0x0000_0000_2001_0001` |
| 0x008       | `0x0000_0000_0000_0000` |
| ...         | (all other entries are 0x0) |

**Level-1 page table B at physical address `0x8001_0000`:**

| Byte offset | PTE value |
|-------------|-----------|
| 0x000       | `0x0000_0000_0000_0000` |
| ...         | (all other entries are 0x0) |
| 0xFF8       | `0x0020_0000_2003_00CF` |

**Level-0 (leaf) page table at physical address `0x8004_0000`:**

| Byte offset | PTE value |
|-------------|-----------|
| 0x000       | `0x0000_0000_2005_00C7` |
| 0x008       | `0x0000_0000_2005_10C7` |
| 0x010       | `0x0000_0000_2005_20CB` |
| ...         | (all other entries are 0x0) |

**Part A:** Decode `satp` to determine the Sv39 mode, ASID, and root page table physical address.

**Part B:** Walk the Sv39 page table for virtual address `0xFFFF_FFC0_0000_0000`. Show every PTE address computed, every PTE value loaded, and the final physical address or fault.

**Part C:** Walk the Sv39 page table for virtual address `0x0000_0000_0000_1008`. Show all steps.

**Part D:** Walk the Sv39 page table for virtual address `0x0000_0000_0000_3FF8`. Identify the type of fault (if any) and the value of `stval` that would be reported.

**Part E:** A U-mode process attempts a store to virtual address `0x0000_0000_0000_0008`. Determine whether the access is permitted and identify the relevant PTE permission bits.

---

## Solution

### Part A: Decode satp

```
satp = 0x8000_0000_0008_0001

Binary: 1000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_1000_0000_0000_0000_0001

Bits [63:60] = MODE:
  satp >> 60 = 0x8 = 8 => Sv39 mode confirmed

Bits [59:44] = ASID (16 bits):
  (satp >> 44) & 0xFFFF = (0x8000_0000_0008_0001 >> 44) & 0xFFFF
  = 0x80000 & 0xFFFF = 0x0000
  ASID = 0

Bits [43:0] = PPN (44 bits):
  satp & 0x0FFF_FFFF_FFFF = 0x0000_0000_0008_0001
  PPN = 0x80001

Root page table physical address:
  PPN * 4096 = 0x80001 * 0x1000 = 0x8000_1000
```

### Part B: Walk for VA `0xFFFF_FFC0_0000_0000`

**Step 0: Validity check**

Sv39 requires bits [63:39] to all equal bit [38]:
```
VA = 0xFFFF_FFC0_0000_0000
Bit 38 = (VA >> 38) & 1 = (0xFFFF_FFC0_0000_0000 >> 38) & 1

0xFFFF_FFC0_0000_0000 >> 38 = 0x3FFFFF (keeping lower bits):
More precisely: 0xFFFF_FFC0_0000_0000 in binary:
  1111_1111_1111_1111_1111_1111_1100_0000_0000_0000_0000_0000_0000_0000_0000_0000

Bit 38 = 1 (the 39th bit from LSB, 0-indexed)
Bits [63:39]: all 1s (FFFFFF... shifted)
Bits [63:39] = 0x1FFFFFF (25 bits all ones)
Bit 38 = 1

All bits [63:39] equal bit [38] = 1. VALID Sv39 address.
```

This is a high (kernel) virtual address — negative in signed representation. Kernel mappings in Linux RISC-V start at `0xFFFF_FFC0_0000_0000`.

**Step 1: Extract VPN fields**

```
VA = 0xFFFF_FFC0_0000_0000

VPN[2] = va[38:30] = (VA >> 30) & 0x1FF
  0xFFFF_FFC0_0000_0000 >> 30 = 0x3FFFFFFFC00 >> 0 ... let's compute:
  VA >> 30 = 0xFFFF_FFC0_0000_0000 / 0x4000_0000
           = 0x3FFFF_FF00 (approximately)
  More precisely: shift right 30:
  0xFFFF_FFC0_0000_0000 >> 30 = 0xFFFF_FFF0_0000 (64-bit)
  Masking low 9 bits: & 0x1FF = 0x1FF = 511

VPN[1] = va[29:21] = (VA >> 21) & 0x1FF
  VA >> 21 = 0x7FFFF_FE000 (approximately)
  Lower 9 bits = 0x000 = 0

VPN[0] = va[20:12] = (VA >> 12) & 0x1FF
  VA >> 12 = 0xFFFF_FFC0_0000
  Lower 9 bits = 0x000 = 0

offset = VA & 0xFFF = 0x000
```

Summary:
```
VPN[2] = 511 = 0x1FF
VPN[1] = 0
VPN[0] = 0
offset = 0x000
```

**Step 2: Level-2 (root) walk**

```
PTE address = root_PT + VPN[2] * 8
            = 0x8000_1000 + 511 * 8
            = 0x8000_1000 + 0xFF8
            = 0x8000_1FF8

PTE value at 0x8000_1FF8 = 0x0000_0000_2000_4001

Decode PTE:
  V = bit 0 = 1 (valid)
  R = bit 1 = 0
  W = bit 2 = 0
  X = bit 3 = 0
  => R=W=X=0: this is a POINTER PTE (points to next-level PT)

  PPN = PTE[53:10] = 0x0000_0000_2000_4001 >> 10
      = 0x0000_0000_0008_0010
  PPN = 0x80010 (after masking to 44 bits)
```

**Step 3: Level-1 walk**

```
Level-1 PT physical address = PPN * 4096 = 0x80010 * 0x1000 = 0x8001_0000

PTE address = 0x8001_0000 + VPN[1] * 8
            = 0x8001_0000 + 0 * 8
            = 0x8001_0000

PTE value at 0x8001_0000 = 0x0000_0000_0000_0000

V = bit 0 = 0 (INVALID!)
=> PAGE FAULT
```

Re-examining: the entry at offset 0xFF8 is the last entry (index 511), not offset 0. Offset 0 is zero.

```
PTE at 0x8001_0000 (offset 0, index 0) = 0x0000_0000_0000_0000
V = 0 => INVALID PTE => instruction/load/store PAGE FAULT

scause = 12 (instruction page fault) or 13 (load) or 15 (store)
  depending on the access type
stval  = 0xFFFF_FFC0_0000_0000 (faulting virtual address)
```

This virtual address is NOT mapped — Linux would need to handle this as a kernel page fault, which typically indicates a kernel bug (NULL pointer dereference in kernel context) and results in a kernel oops.

### Part C: Walk for VA `0x0000_0000_0000_1008`

**Step 0: Validity check**

```
VA = 0x0000_0000_0000_1008
Bit 38 = 0; bits [63:39] = all zeros. VALID (positive user-space address).
```

**Step 1: Extract VPN fields**

```
VPN[2] = (0x0000_0000_0000_1008 >> 30) & 0x1FF = 0
VPN[1] = (0x0000_0000_0000_1008 >> 21) & 0x1FF = 0
VPN[0] = (0x0000_0000_0000_1008 >> 12) & 0x1FF = 1
offset = 0x0000_0000_0000_1008 & 0xFFF = 0x008
```

**Step 2: Level-2 walk**

```
PTE address = 0x8000_1000 + 0 * 8 = 0x8000_1000
PTE value = 0x0000_0000_2000_2001

V = 1, R=0, W=0, X=0 => pointer PTE
PPN = 0x0000_0000_2000_2001 >> 10 = 0x80008_0 >> ... 
PPN field [53:10]:
  0x0000_0000_2000_2001 in binary, bits 53 down to 10:
  The value is 0x2000_2001. Bits [53:10]:
  0x2000_2001 = 0b 0010_0000_0000_0000_0010_0000_0000_0001
  Bits [53:10] of a 64-bit value:
  0x2000_2001 fits in 32 bits, so bits [53:32] are 0.
  Bits [31:10] = 0x2000_2001 >> 10 = 0x80008
  PPN = 0x80008
```

**Step 3: Level-1 walk**

```
Level-1 PT physical address = 0x80008 * 0x1000 = 0x8000_8000

PTE address = 0x8000_8000 + 0 * 8 = 0x8000_8000
PTE value = 0x0000_0000_2001_0001

V = 1, R=0, W=0, X=0 => pointer PTE
PPN = 0x0000_0000_2001_0001 >> 10 = 0x80040 (bits [31:10] of 0x2001_0001)
  0x2001_0001 >> 10 = 0x80040
  PPN = 0x80040
```

**Step 4: Level-0 leaf walk**

```
Level-0 PT physical address = 0x80040 * 0x1000 = 0x8004_0000

PTE address = 0x8004_0000 + 1 * 8 = 0x8004_0008
PTE value = 0x0000_0000_2005_10C7

Decode:
  V = bit 0 = 1 (valid)
  R = bit 1 = 1
  W = bit 2 = 1
  X = bit 3 = 0
  U = bit 4 = 1 (user accessible)
  G = bit 5 = 0
  A = bit 6 = 1
  D = bit 7 = 1
  => R=1, W=1, X=0: read-write user page (no execute)
  => LEAF PTE

  PPN = 0x0000_0000_2005_10C7 >> 10 = 0x80014
  (0x2005_10C7 >> 10 = 0x80014 after masking)
```

**Step 5: Compute physical address**

```
PA = PPN * 4096 + offset
   = 0x80014 * 0x1000 + 0x008
   = 0x8001_4000 + 0x008
   = 0x8001_4008
```

Virtual address `0x0000_0000_0000_1008` maps to physical address **`0x8001_4008`**.  
Permissions: R=1, W=1, X=0, U=1 (user read/write, no execute).

### Part D: Walk for VA `0x0000_0000_0000_3FF8`

**Step 1: Extract VPN fields**

```
VA = 0x0000_0000_0000_3FF8
VPN[2] = (VA >> 30) & 0x1FF = 0
VPN[1] = (VA >> 21) & 0x1FF = 0
VPN[0] = (VA >> 12) & 0x1FF
  0x3FF8 >> 12 = 0x3 = 3
  VPN[0] = 3
offset = 0x3FF8 & 0xFFF = 0xFF8
```

**Step 2 and 3: Same as Part C**

VPN[2]=0, VPN[1]=0 => same root PTE and level-1 PTE as Part C.
Level-0 PT at `0x8004_0000`.

**Step 4: Level-0 leaf walk**

```
PTE address = 0x8004_0000 + 3 * 8 = 0x8004_0018

PTE value at 0x8004_0018 = 0x0000_0000_0000_0000

V = 0 (INVALID PTE)
```

**Result: Page fault**

```
Fault type: depends on access:
  Load  => scause = 13 (load page fault)
  Store => scause = 15 (store/AMO page fault)
  Fetch => scause = 12 (instruction page fault)

stval = 0x0000_0000_0000_3FF8   (faulting virtual address)
sepc  = PC of the faulting instruction
```

The kernel would attempt to handle this as a demand-paging fault: look up the VMA for address `0x3FF8`, allocate a physical page, install a PTE, and re-execute. If no VMA exists for that address, the fault is a segfault (SIGSEGV delivered to the process).

### Part E: U-mode Store to VA `0x0000_0000_0000_0008`

**Walk: same VPN[2]=0, VPN[1]=0, VPN[0]=0, offset=0x008**

Level-0 PTE at `0x8004_0000 + 0 * 8 = 0x8004_0000`:
```
PTE value = 0x0000_0000_2005_00C7

V = 1, R = 1, W = 1, X = 0, U = 1, G = 0, A = 1, D = 1
Leaf PTE: read/write/no-execute, user-accessible
```

Permission check for U-mode store:
```
W = 1  => write permitted
U = 1  => user-mode access permitted (U=0 would require S-mode or SUM=1)
D = 1  => dirty bit already set (no fault-and-update needed)
A = 1  => accessed bit set (same)
```

Result: Store is **permitted**.

Physical address: PPN = `0x80014` => PA = `0x8001_4000 + 0x008` = `0x8001_4008`.

VPN[0]=0 maps to PTE at offset 0, PPN = 0x80014 (from the first PTE `0x2005_00C7`):
```
0x2005_00C7 >> 10 = 0x80014
PA = 0x80014 * 0x1000 + 0x008 = 0x8001_4008
```

The store succeeds. Physical memory at `0x8001_4008` is written.

---

## Discussion

### PTE Decoding Reference

For any 64-bit Sv39 PTE, extract fields as follows:

```
Bit  Field  Extract as
0    V      (pte >> 0) & 1
1    R      (pte >> 1) & 1
2    W      (pte >> 2) & 1
3    X      (pte >> 3) & 1
4    U      (pte >> 4) & 1
5    G      (pte >> 5) & 1
6    A      (pte >> 6) & 1
7    D      (pte >> 7) & 1
53:10 PPN   (pte >> 10) & 0x3FFF_FFFF_FFFF
```

Leaf vs pointer:
```
(R | W | X) == 0  =>  pointer PTE (non-leaf)
(R | W | X) != 0  =>  leaf PTE
W=1, R=0          =>  RESERVED, raises page fault
```

### Common Mistakes in Page Table Walk Problems

1. **Wrong PTE size:** Sv39 uses 8-byte PTEs. Multiplying VPN by 4 (RV32 PTE size) instead of 8 gives the wrong PTE address.

2. **Forgetting the shift in PPN extraction:** The PPN in a PTE is stored in bits [53:10]. To get the physical page number, right-shift by 10 and mask to 44 bits. Forgetting the shift is the most common arithmetic error.

3. **Confusing PPN of satp with a PA:** `satp.PPN` is already the physical page number — multiply by 4096 to get the physical address of the root page table.

4. **Ignoring the Sv39 canonical address check:** Addresses where bits [63:39] != sign-extend(bit[38]) are invalid and immediately raise a page fault, without any table walk.

5. **Superpage identification:** A leaf PTE at level 1 (not level 0) is a 2 MiB superpage. In that case, `pa.VPN[0]` comes from the virtual address, not the PTE. Not checking for leaf PTEs at non-leaf levels (or forgetting this is possible) is a common oversight.

### Walk Summary Table

| VA | VPN[2] | VPN[1] | VPN[0] | Result |
|---|---|---|---|---|
| `0xFFFF_FFC0_0000_0000` | 511 | 0 | 0 | Page fault (level-1 PTE invalid) |
| `0x0000_0000_0000_1008` | 0 | 0 | 1 | PA `0x8001_4008`, R/W/U |
| `0x0000_0000_0000_3FF8` | 0 | 0 | 3 | Page fault (level-0 PTE invalid) |
| `0x0000_0000_0000_0008` | 0 | 0 | 0 | PA `0x8001_4008`, R/W/U, store OK |
