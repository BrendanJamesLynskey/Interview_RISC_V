# Virtual Memory: Sv32, Sv39, Sv48

## Prerequisites
- RISC-V privilege modes, particularly S-mode and the `satp` CSR
- Physical vs virtual address concepts
- Basic understanding of page tables and TLBs
- Bitfield extraction and alignment arithmetic

---

## Concept Reference

### Virtual Memory Overview

Virtual memory in RISC-V is controlled by the `satp` CSR. When `satp.MODE != 0`, the MMU translates every S-mode and U-mode virtual address through a multi-level page table walk before issuing the physical memory access. M-mode code always uses physical addresses directly unless `mstatus.MPRV = 1`.

RISC-V defines three virtual address schemes:

| Scheme | Architecture | VA bits | PA bits | Page table levels | Page size |
|--------|-------------|---------|---------|-------------------|-----------|
| Sv32   | RV32        | 32      | 34      | 2                 | 4 KiB     |
| Sv39   | RV64        | 39      | 56      | 3                 | 4 KiB     |
| Sv48   | RV64        | 48      | 56      | 4                 | 4 KiB     |
| Sv57   | RV64        | 57      | 56      | 5                 | 4 KiB     |

All schemes also support **megapages** (2 MiB in Sv39) and **gigapages** (1 GiB in Sv39) as leaf PTEs at non-leaf levels, enabling huge page mappings.

### Sv32: Two-Level Page Tables (RV32)

Sv32 divides the 32-bit virtual address into three fields:

```
Virtual Address (32 bits):
  [31:22]  VPN[1]   10 bits  Index into root (level-1) page table
  [21:12]  VPN[0]   10 bits  Index into leaf (level-0) page table
  [11:0]   offset   12 bits  Byte offset within the 4 KiB page

Physical Address (34 bits):
  [33:12]  PPN      22 bits  Physical page number
  [11:0]   offset   12 bits  Same as virtual offset (unchanged)

satp (RV32):
  [31]     MODE     1 bit    0=Bare, 1=Sv32
  [30:22]  ASID     9 bits   Address Space ID
  [21:0]   PPN      22 bits  PPN of the root page table
```

Sv32 Page Table Entry (PTE) format (32 bits, each PTE is 4 bytes):

```
Bit(s)  Field   Description
31:10   PPN     Physical Page Number of next-level PT or leaf page
9:8     RSW     Reserved for supervisor software (ignored by hardware)
7       D       Dirty: set by hardware on store to page; must be 1 to store
6       A       Accessed: set by hardware on any access; must be 1 to access
5       G       Global: page mapped in all address spaces (ignores ASID)
4       U       User: page accessible in U-mode; if 0, S-mode can access only
                if sstatus.SUM=1
3       X       Execute permission
2       W       Write permission
1       R       Read permission
0       V       Valid: if 0, this PTE is invalid (page not present)
```

Leaf vs pointer PTE:
```
R=0, W=0, X=0  =>  pointer PTE (PPN points to next-level page table)
Any of R, W, X = 1  =>  leaf PTE (PPN points to the actual page)
W=1 and R=0 is RESERVED (invalid combination)
```

### Sv39: Three-Level Page Tables (RV64)

Sv39 is the most common scheme for 64-bit Linux. It gives a 512 GiB virtual address space (enough for most current workloads).

```
Virtual Address (64 bits, only 39 bits used):
  [63:39]  Must equal bit 38 (sign-extended); violation = page fault
  [38:30]  VPN[2]   9 bits   Index into root (level-2) page table
  [29:21]  VPN[1]   9 bits   Index into level-1 page table
  [20:12]  VPN[0]   9 bits   Index into leaf (level-0) page table
  [11:0]   offset   12 bits  Byte offset within the 4 KiB page

Physical Address (56 bits):
  [55:12]  PPN[2:0]  44 bits  Physical page number (split as 26+9+9)
  [11:0]   offset    12 bits  Same as virtual offset

satp (RV64, Sv39):
  [63:60]  MODE     4 bits   8 = Sv39
  [59:44]  ASID    16 bits   Address Space ID
  [43:0]   PPN     44 bits   PPN of root page table
```

Sv39 PTE format (64 bits, each PTE is 8 bytes):

```
Bit(s)  Field
63:54   Reserved (must be zero; non-zero triggers page fault)
53:10   PPN[2:0]  44-bit physical page number
9:8     RSW
7       D
6       A
5       G
4       U
3       X
2       W
1       R
0       V
```

Each page table in Sv39 contains 512 entries (2^9), occupying exactly one 4 KiB page.

### Sv48: Four-Level Page Tables (RV64)

Sv48 extends Sv39 with an additional level, expanding the virtual address space to 256 TiB:

```
Virtual Address (64 bits, only 48 bits used):
  [63:48]  Must equal bit 47 (sign-extended)
  [47:39]  VPN[3]   9 bits   Root (level-3) page table index
  [38:30]  VPN[2]   9 bits   Level-2 index
  [29:21]  VPN[1]   9 bits   Level-1 index
  [20:12]  VPN[0]   9 bits   Leaf index
  [11:0]   offset   12 bits
```

PTE format is identical to Sv39. The hardware simply performs four walks instead of three.

### The Page Table Walk Algorithm

The RISC-V specification defines the page table walk precisely. All hardware implementations must follow this algorithm:

```
Input:  va (virtual address), satp, access_type (read/write/execute)
Output: pa (physical address) or page fault

i = LEVELS - 1   (e.g., 2 for Sv39, starting from the root)
a = satp.PPN * PAGE_SIZE   (physical address of root page table)

loop:
  pte_addr = a + VPN[i] * PTE_SIZE
  pte = memory_load(pte_addr, PTE_SIZE)  # physical memory load

  if pte.V == 0 or (pte.W == 1 and pte.R == 0):
    raise PAGE_FAULT              # invalid PTE

  if pte.R == 1 or pte.X == 1:   # leaf PTE found
    goto leaf_check

  # Non-leaf: follow pointer to next level
  i = i - 1
  if i < 0:
    raise PAGE_FAULT              # too many levels, malformed table
  a = pte.PPN * PAGE_SIZE
  goto loop

leaf_check:
  # Permission checks
  if access_type == EXECUTE and pte.X == 0: raise INSTRUCTION_PAGE_FAULT
  if access_type == READ and pte.R == 0:
    if pte.X == 0 or mstatus.MXR == 0: raise LOAD_PAGE_FAULT
  if access_type == WRITE and pte.W == 0: raise STORE_PAGE_FAULT
  if current_privilege == U and pte.U == 0: raise PAGE_FAULT
  if current_privilege == S and pte.U == 1 and sstatus.SUM == 0:
    raise PAGE_FAULT

  # Superpage alignment check
  if i > 0:  # this is a superpage (leaf at non-leaf level)
    for j in 0..i-1:
      if pte.PPN[j] != 0: raise PAGE_FAULT  # misaligned superpage

  # A/D bit update
  if pte.A == 0 or (access_type == WRITE and pte.D == 0):
    # Hardware may: raise a page fault (to let OS update A/D)
    # OR atomically set A/D bits in the PTE

  # Construct physical address
  pa.offset = va.offset
  pa.PPN[LEVELS-1:i+1] = pte.PPN[LEVELS-1:i+1]
  pa.PPN[i:0] = va.VPN[i:0]   # superpage: low PPN bits from VA
  return pa
```

The superpage physical address construction at the last step is important: for a 2 MiB superpage (i=1 in Sv39), `pa.PPN[0]` (the 9-bit leaf PPN) comes from `va.VPN[0]`, not from the PTE. This is what makes a single PTE map a contiguous 2 MiB region.

### Page Table Entry A and D Bits

The **Accessed** (A) and **Dirty** (D) bits support OS page reclamation algorithms:

- `A` is set by hardware on any access (read, write, or execute) to the page.
- `D` is set by hardware on any **write** to the page.
- The OS can clear these bits periodically; if a page has `A=0` for a long period, it is a candidate for swapping out.

Implementations may handle `A/D` updates in two ways:
1. **Hardware update**: The hardware atomically sets the bit in the PTE in memory. This is fast but requires the hardware to perform a store to the page table on every access.
2. **Fault-and-update**: When `A=0` (or `D=0` on a write), the hardware raises a page fault. The OS sets the bit and re-executes the instruction. This simplifies hardware but adds OS overhead.

Most high-performance implementations use hardware update. Simple embedded implementations may use fault-and-update to avoid the hardware complexity.

### TLB: Translation Lookaside Buffer

Walking a three-level page table for every memory access would be prohibitively slow (3 extra memory accesses per access). The **TLB** caches recent virtual-to-physical translations to avoid repeated walks.

**TLB organization:**
- Typically split into instruction TLB (ITLB) and data TLB (DTLB).
- May have separate entries for 4 KiB pages and 2 MiB/1 GiB superpages.
- Tagged with the ASID from `satp` to allow multiple address spaces to coexist.

**TLB invalidation (SFENCE.VMA):**
The `SFENCE.VMA rs1, rs2` instruction flushes TLB entries. Its operands refine the flush:

```
SFENCE.VMA x0, x0   =>  Flush all TLB entries for all ASIDs
SFENCE.VMA rs1, x0  =>  Flush all TLB entries for all ASIDs at VA = rs1
SFENCE.VMA x0, rs2  =>  Flush all TLB entries for ASID = rs2
SFENCE.VMA rs1, rs2 =>  Flush TLB entry for VA = rs1, ASID = rs2
```

`SFENCE.VMA` is a fence: all prior stores to page table memory are visible to subsequent page table walks.

When to use `SFENCE.VMA`:
- After modifying a PTE (adding, removing, or changing a mapping): flush entries for the affected virtual address and ASID.
- After changing `satp` (switching address spaces): flush the entire TLB or use ASID tagging to avoid a full flush.

**ASID tagging** is a critical optimization: if every context switch does a full TLB flush, the TLB is cold after every switch. By tagging TLB entries with the ASID, the hardware can retain entries from other address spaces. Only entries with the old ASID need to be flushed when a mapping is modified, not the entire TLB.

### G (Global) Bit

A PTE with `G = 1` is a global mapping, shared across all address spaces. Global pages are not flushed by `SFENCE.VMA` unless the ASID operand is x0 (flush all ASIDs).

Kernel mappings (which are the same in every address space) are typically marked global, ensuring the kernel's TLB entries are never evicted by user-space context switches.

---

## Tier 1 — Fundamentals

### Question F1
**What does the `satp` CSR control, and what must the OS do after writing it?**

**Answer:**

`satp` (Supervisor Address Translation and Protection, CSR `0x180`) controls virtual memory for S-mode and U-mode:

- `MODE` field: selects the translation scheme (Bare = no translation, Sv32, Sv39, Sv48).
- `ASID` field: identifies the current address space. TLB entries tagged with this ASID are considered valid for the current process.
- `PPN` field: physical page number of the root page table. The hardware multiplies this by 4096 to get the root page table's physical address.

After writing `satp`, the OS **must** execute `SFENCE.VMA` to ensure:
1. All prior stores to page table memory are visible to subsequent hardware page table walks.
2. Stale TLB entries from the previous address space (if the ASID changed) are flushed.

```asm
# Switch to a new process's address space
csrw  satp, a0        # a0 = new satp value (mode | asid | ppn)
sfence.vma x0, x0     # full TLB flush (or use ASID-specific flush)
```

Omitting `SFENCE.VMA` is a common bug that causes intermittent faults: the TLB may hold stale mappings from the previous address space that happen to match a virtual address in the new process.

---

### Question F2
**In Sv39, how many bytes does a complete page table occupy? How many physical memory accesses does a full page table walk require?**

**Answer:**

A single Sv39 page table has 2^9 = 512 entries, each 8 bytes (64-bit PTE). Total size: 512 × 8 = **4096 bytes = one 4 KiB page**.

This is by design: each page table level occupies exactly one page, so the OS can allocate page tables using the same page allocator used for data pages.

A full 3-level Sv39 walk for a 4 KiB page requires **3 physical memory accesses**:
```
Access 1: Load PTE from root (level-2) page table
          Address: satp.PPN * 4096 + va[38:30] * 8

Access 2: Load PTE from level-1 page table
          Address: PTE1.PPN * 4096 + va[29:21] * 8

Access 3: Load PTE from leaf (level-0) page table
          Address: PTE2.PPN * 4096 + va[20:12] * 8

Final PA: PTE3.PPN * 4096 + va[11:0]
```

Without a TLB, every 4-byte data load would require 3 + 1 = **4 physical memory accesses**. For an instruction fetch, the same applies. The TLB eliminates the 3 walk accesses for cached translations, making effective memory access overhead negligible.

For a 2 MiB superpage (leaf at level 1), only 2 walk accesses are required.

---

### Question F3
**What is the purpose of the `U` bit in a Sv39 PTE? How does `sstatus.SUM` modify its behaviour?**

**Answer:**

The `U` (User) bit controls whether the page is accessible in U-mode:

```
PTE.U = 1:  Page is accessible from U-mode.
            S-mode can only access this page if sstatus.SUM = 1.
            (Default S-mode behaviour: S-mode cannot read/write U-mode pages.)

PTE.U = 0:  Page is NOT accessible from U-mode.
            U-mode access raises a page fault.
            S-mode can always access this page.
```

`sstatus.SUM` (Supervisor User Memory access) is a one-bit flag that, when set, temporarily allows S-mode to access pages with `U=1`. This is needed for kernel routines that copy data between kernel memory and user buffers (`copy_to_user`, `copy_from_user`).

Security implication: without the `U` bit restriction on S-mode, a kernel vulnerability that corrupts the instruction pointer could redirect kernel execution to user-controlled code. With `SUM=0` (the default), executing user-page code from S-mode would raise a page fault (X permission on a U-page is checked). The kernel sets `SUM=1` only for the duration of the specific copy operation, minimising the attack window.

---

### Question F4
**What causes a "page fault" in RISC-V? Name three different `mcause`/`scause` codes for page faults and explain when each fires.**

**Answer:**

A page fault is raised during a hardware page table walk when the walk cannot produce a valid physical address with the required permissions. There are three page fault cause codes:

```
Cause 12  Instruction page fault:
  Fired when the MMU cannot translate the PC during an instruction fetch.
  Triggers: PTE.V=0, PTE.X=0, PTE.U=0 (in U-mode), reserved PTE bits set.
  mtval/stval = faulting PC.

Cause 13  Load page fault:
  Fired when a load instruction's effective address cannot be translated.
  Triggers: PTE.V=0, PTE.R=0 (and MXR=0 prevents X-only fallback),
            PTE.U=0 (in U-mode), SUM rules, reserved bits.
  mtval/stval = faulting virtual address.

Cause 15  Store/AMO page fault:
  Fired when a store or atomic instruction's address cannot be translated.
  Triggers: PTE.V=0, PTE.W=0, PTE.D=0 (on implementations using
            fault-and-update for D bit), PTE.U=0, etc.
  mtval/stval = faulting virtual address.
```

Note there is no cause 14: that code is reserved (the sequence goes 12, 13, 15, skipping 14 which is reserved for a future use).

---

## Tier 2 — Intermediate

### Question I1
**Explain Sv39 superpage mappings. For a 2 MiB superpage, which PTE level is the leaf, and how is the physical address constructed from the PTE and the virtual address?**

**Answer:**

In Sv39, a **superpage** is created by placing a leaf PTE at a non-leaf page table level. A leaf PTE is identified by having at least one of R, W, or X set.

For a **2 MiB superpage** (gigapage in the level-1 table):
- The walk reaches the level-1 page table (after consulting the level-2 root).
- The level-1 PTE has `R=1` or `X=1` (leaf PTE at level i=1).
- The walk terminates here.

Physical address construction at level i=1:

```
va = virtual address
pte = leaf PTE at level 1

pa[55:30] = pte.PPN[2:1]   # 26 bits from PTE: gigapage region
pa[29:21] = pte.PPN[0]     # 9 bits from PTE: 2 MiB page region within giga
            (actually: pa[29:21] = pte.PPN[0])
pa[20:12] = va[20:12]      # va.VPN[0] (9 bits) from virtual address
pa[11:0]  = va[11:0]       # page offset from virtual address

Wait -- correction for Sv39 2 MiB superpage (leaf at level i=1):
  pa[55:21] = pte.PPN[2:0] (all 44 bits of PPN, but pte.PPN[0] must be 0)
  pa[20:12] = va.VPN[0]    (9 bits from VA, selects 4 KiB within the 2 MiB)
  pa[11:0]  = va.offset    (12 bits, byte within the 4 KiB page)
```

Alignment requirement: for a valid 2 MiB superpage, `pte.PPN[0]` must be 0. If it is non-zero, the hardware raises a page fault (misaligned superpage). This ensures the superpage is naturally aligned to a 2 MiB boundary.

Similarly, for a **1 GiB superpage** (leaf at level i=2, the root):
```
pa[55:30] = pte.PPN[2]     (PPN[1] and PPN[0] must be 0)
pa[29:21] = va.VPN[1]
pa[20:12] = va.VPN[0]
pa[11:0]  = va.offset
```

Superpages are valuable for mapping large regions (kernel text, MMIO regions, large contiguous allocations) with a single TLB entry, reducing TLB pressure and walk overhead.

---

### Question I2
**A Linux kernel performs a context switch between two processes. Describe the complete sequence of `satp` and `SFENCE.VMA` operations, and explain why ASID management matters for performance.**

**Answer:**

**Without ASID optimisation (simple implementation):**

```asm
# context_switch(prev_mm, next_mm)
# 1. Load next process's satp value
ld    t0, next_mm_satp(a1)   # precomputed satp: MODE|ASID=0|PPN

# 2. Switch address space
csrw  satp, t0

# 3. Full TLB flush (required because ASID=0 for all processes)
sfence.vma x0, x0
```

Every context switch flushes the entire TLB. The next process starts with a cold TLB and will incur page table walk overhead for every unique page it accesses.

**With ASID optimisation:**

The kernel allocates a unique 16-bit ASID (in Sv39) to each process from a pool of available ASIDs. The `satp` value for each process embeds its ASID:

```asm
# context_switch(prev_mm, next_mm)
# 1. Load next process's satp (includes its ASID)
ld    t0, next_mm_satp(a1)   # satp: MODE|ASID=next_asid|PPN

# 2. Switch address space
csrw  satp, t0

# 3. No SFENCE.VMA needed if:
#    - next_asid has not been recycled since this process last ran
#    - no page table modifications were made to this process since last run
# Only flush if the process's mappings changed:
# sfence.vma x0, rs2   (where rs2 = next_asid -- flush only this ASID)
```

TLB entries tagged with `next_asid` that were deposited during the process's previous timeslice are **still valid** — no flush needed.

**ASID exhaustion:** The hardware supports at most 2^ASIDLEN ASIDs (up to 65536 for 16-bit). When all are allocated, the kernel performs a **TLB generation rollover**:
1. Flush all TLB entries (`sfence.vma x0, x0`).
2. Reset the ASID counter to 0.
3. Reallocate ASIDs for all active processes.

This rollover is infrequent (requires more than 65536 unique processes to have been created since last rollover) but causes a full TLB flush when it occurs.

---

### Question I3
**What is `mstatus.MXR` and when would a kernel set it?**

**Answer:**

`MXR` (Make eXecutable Readable, bit 19 of `mstatus`) relaxes the page permission check for loads: when `MXR = 1`, a load from a page that has `X=1` succeeds even if `R=0`.

Normal behaviour (`MXR = 0`): a page with `R=0, X=1` is execute-only. A load raises a load page fault.

With `MXR = 1`: a page with `R=0, X=1` can be read. The execute permission acts as an implicit read permission.

Use case — **execute-only code pages and dynamic linking:**
Some security-hardened kernels map executable code with `R=0, X=1` to prevent user-space data reads of code pages (e.g., to hinder JIT spraying or information disclosure of code layout). However, the dynamic linker must read the GOT (Global Offset Table) and PLT (Procedure Linkage Table) entries, which may reside in the same page or adjacent pages with mixed R/X permissions.

More practically, `MXR` is used in kernel contexts where the kernel needs to read data from a page that was mapped execute-only by user space:

```asm
# Kernel: temporarily allow reading execute-only user pages
li    t0, MSTATUS_MXR
csrs  mstatus, t0         # set MXR
lw    a0, 0(a1)           # read from user execute-only page
csrc  mstatus, t0         # clear MXR immediately after
```

In practice, Linux RISC-V uses `MXR` primarily for ELF binary loading: the loader reads the `.text` segment from an executable file whose page may have been mapped with `X=1` before `R=1` was set.

---

## Tier 3 — Advanced

### Question A1
**Describe the RISC-V page table walk in exact detail for a virtual address `0x3F_4ABC_1234` on an Sv39 system with `satp.PPN = 0x80001`. Show the address calculation for each level.**

**Answer:**

Virtual address: `0x3F_4ABC_1234`

First, verify the Sv39 validity constraint: bits [63:39] must all equal bit 38.
```
VA = 0x0000_003F_4ABC_1234
Bit 38 = 0 (VA[38] = bit 38 of 0x3F_4ABC_1234)

0x3F = 0b00111111, so VA[38:32] = 0b0011111 (bit 38 = 0)
Bits [63:39] are all 0, and bit 38 = 0. Valid Sv39 address.
```

Extract VA fields:
```
VA = 0x0000_003F_4ABC_1234 = 0b ...0_0011_1111_0100_1010_1011_1100_0001_0010_0011_0100

Bits [38:30] = VPN[2]: extract bits 38 down to 30
  VA >> 30 = 0x3F4A >> 3 ... let's compute directly:
  0x3F_4ABC_1234 in binary, bits 38-30:
  0x3F_4ABC_1234 = 0b 0_0011_1111_0100_1010_1011_1100_0001_0010_0011_0100
  bits 38-30 = 0_0011_1111_0  => 0b0_0011_1111_0 = wait, numbering:
  bit 38 = (VA >> 38) & 1 = (0x3F_4ABC_1234 >> 38) & 1

Let's use hex arithmetic:
  0x3F_4ABC_1234 >> 30 = 0x3F_4ABC_1234 / 0x4000_0000
  0x3F_4ABC_1234 = 272_070_099_508
  272_070_099_508 / 1_073_741_824 = 253 (0xFD)
  VPN[2] = 0xFD & 0x1FF = 0xFD = 253

  VPN[1] = (VA >> 21) & 0x1FF
  0x3F_4ABC_1234 >> 21 = 272_070_099_508 / 2_097_152 = 129_729 (approx)
  More precisely: (0x3F_4ABC_1234 >> 21) & 0x1FF
  0x3F_4ABC_1234 >> 21 = 0x1FA_2 (keeping only relevant bits)
  Actually: 0x3F_4ABC_1234 >> 21:
    shift 21: 0x1FA25E (top bits after shifting)
    VPN[1] = 0x1FA25E & 0x1FF = 0x05E = 94

  VPN[0] = (VA >> 12) & 0x1FF
  0x3F_4ABC_1234 >> 12 = 0x3F4ABC1 (drop bottom hex digit + partial)
  VPN[0] = 0x3F4ABC1 & 0x1FF = 0xBC1 & 0x1FF = 0x1C1 = 449

  offset = VA & 0xFFF = 0x234
```

Page table walk:
```
satp.PPN = 0x80001, so root PT physical address = 0x80001 * 0x1000 = 0x80001000

Level 2 (root):
  PTE address = 0x80001000 + VPN[2] * 8
              = 0x80001000 + 253 * 8
              = 0x80001000 + 0x7E8
              = 0x800017E8
  Load 8-byte PTE from physical address 0x800017E8.
  Assume PTE = 0x0000_0000_2000_2001 (V=1, R=0, W=0, X=0 => pointer PTE)
  PPN = 0x0000_0000_2000_2001 >> 10 = 0x80008 (approx; PPN field is [53:10])
  PPN = (PTE >> 10) & 0x3FFF_FFFF_FFFF = 0x20000 >> ... let's say PPN = 0x80002

Level 1:
  PT physical address = 0x80002 * 0x1000 = 0x80002000
  PTE address = 0x80002000 + VPN[1] * 8
              = 0x80002000 + 94 * 8
              = 0x80002000 + 0x2F0
              = 0x800022F0
  Assume PTE = 0x0000_0000_2000_4001 (V=1, R=0, W=0, X=0 => pointer PTE)
  PPN = 0x80010

Level 0 (leaf):
  PT physical address = 0x80010 * 0x1000 = 0x80010000
  PTE address = 0x80010000 + VPN[0] * 8
              = 0x80010000 + 449 * 8
              = 0x80010000 + 0xE08
              = 0x80010E08
  Assume PTE = 0x0000_0020_0000_00C7 (V=1, R=1, W=0, X=1 => leaf, read+execute)
  PPN = (0x0000_0020_0000_00C7 >> 10) & 0xFFFFFFFFF = 0x8000000 >> ...
  PPN = 0x80000 (44-bit PPN)

Physical address:
  PA = PPN * 0x1000 + offset
     = 0x80000 * 0x1000 + 0x234
     = 0x80000000 + 0x234
     = 0x80000234
```

The result: virtual address `0x3F_4ABC_1234` maps to physical address `0x8000_0234`, with read and execute permissions. A store to this address would raise a store page fault (W=0).

---

### Question A2
**A kernel developer is debugging a page fault storm after a context switch. They find `SFENCE.VMA` was called with the correct ASID but the wrong virtual address argument (a kernel VA instead of the user VA that was modified). Explain precisely what TLB entries were flushed and why the page fault persists.**

**Answer:**

`SFENCE.VMA rs1, rs2` flushes TLB entries matching both the virtual address in `rs1` AND the ASID in `rs2`. The specification states that only entries with a matching ASID and matching virtual page number are flushed.

In this scenario:
- The developer modified a user-space PTE for user VA `0x4000_1000` (a heap page).
- They called `sfence.vma kernel_va, asid` where `kernel_va` maps to some kernel page (e.g., `0xFFFF_FFC0_0000_0000`).
- The ASID matches, but the virtual address does not match the modified user mapping.

Result: the TLB entry for `0x4000_1000` in `asid` is **not flushed**. It remains in the TLB with the old physical address. When the process accesses `0x4000_1000`, the TLB hit returns the stale translation, bypassing the updated PTE entirely.

Why the page fault persists (or does not persist, but gives wrong behaviour):
- If the old PTE pointed to a valid but **wrong** physical page, the access succeeds silently but reads from/writes to incorrect memory — a data corruption bug, not a visible fault.
- If the old PTE was invalid (V=0), the TLB would not hold a cached entry for it (TLBs only cache valid translations). In this case, the walk would occur correctly. The bug manifests only when the old mapping was valid.

**The fix:**
```asm
# After modifying PTE for user VA 0x4000_1000:
li    t0, 0x40001000      # the user virtual address that was modified
csrr  t1, satp
srli  t1, t1, 44          # extract ASID from satp[59:44]
andi  t1, t1, 0xFFFF
sfence.vma t0, t1         # flush exactly the correct VA+ASID combination
```

Or, if multiple PTEs were modified:
```asm
sfence.vma x0, t1         # flush all entries for this ASID
```

This class of bug is difficult to find because: (1) it is timing-dependent (the TLB entry must still be present), (2) it may cause silent corruption rather than a crash, and (3) it does not reproduce under QEMU (which often implements `SFENCE.VMA` as a full flush regardless of arguments).
