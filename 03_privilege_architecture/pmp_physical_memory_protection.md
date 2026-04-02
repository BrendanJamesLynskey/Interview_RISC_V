# PMP: Physical Memory Protection

## Prerequisites
- RISC-V privilege modes: M, S, U and their memory access rules
- CSR register access mechanics (CSRRW, CSRRS, CSRRC)
- Binary representation: powers of two, alignment, bitmasking

---

## Concept Reference

### What PMP Does

Physical Memory Protection (PMP) is a hardware mechanism that allows M-mode to restrict the physical memory regions accessible to S-mode and U-mode code. It operates on **physical addresses** — below the MMU — which makes it effective even when virtual memory is disabled (e.g., during early boot, in a bare-metal RTOS, or in the secure boot phase before the OS starts).

PMP rules:
- Up to 64 PMP regions (implementations must provide at least 0; typical embedded cores provide 8 or 16).
- Each region specifies a physical address range and read/write/execute permissions.
- If a S-mode or U-mode access does not match any PMP region, the access raises an **access fault** (not a page fault).
- M-mode bypasses PMP by default, **unless** the PMP entry has its **lock** (L) bit set, in which case M-mode is also subject to the restriction.

### PMP CSR Organisation

PMP uses two types of CSRs:

**PMP configuration registers (pmpcfg0–pmpcfg15):**
Each `pmpcfg` register holds the configuration for 4 consecutive PMP entries (8-bit config per entry). On RV32, there are 16 pmpcfg CSRs (pmpcfg0–pmpcfg15) holding configs for entries 0–63. On RV64, 8 pmpcfg CSRs (pmpcfg0, pmpcfg2, pmpcfg4...pmpcfg14, even numbers only) each hold 8 configs.

Each 8-bit PMP configuration byte:

```
Bit  Field  Description
7    L      Lock: if 1, this entry also applies to M-mode and cannot
            be modified until reset
6    0      Reserved, must be 0
5    0      Reserved, must be 0
4    A[1]   Address matching mode (high bit)
3    A[0]   Address matching mode (low bit)
2    X      Execute permission
1    W      Write permission
0    R      Read permission

Address matching modes (A field):
  00  OFF    Region disabled; PMP entry is inactive
  01  TOR    Top Of Range: region is [pmpaddr[i-1], pmpaddr[i])
             (pmpaddr[i] is the exclusive upper bound)
  10  NA4    Naturally Aligned 4-byte region
             Address is pmpaddr[i]; region is pmpaddr[i]..pmpaddr[i]+3
  11  NAPOT  Naturally Aligned Power-Of-Two region
             (see encoding below)
```

**PMP address registers (pmpaddr0–pmpaddr63):**
Each `pmpaddr` CSR holds a **right-shifted** physical address. The address stored is `physical_address >> 2` (i.e., bits [33:2] for RV32 with 34-bit PA, or bits [55:2] for RV64 with 56-bit PA).

### NAPOT Address Encoding

NAPOT (Naturally Aligned Power-Of-Two) encodes both the base address and the size in the `pmpaddr` register using a trailing-ones encoding:

```
pmpaddr encoding    Region size    Base address (byte-aligned)
-----------------   -----------   --------------------------------
aaaa...aaaa         4 bytes        (with NA4, not NAPOT)
aaaa...aaa0         8 bytes        base = {pmpaddr[N-1:1], 3'b000}
aaaa...aa01         16 bytes       base = {pmpaddr[N-1:2], 4'b0000}
aaaa...a011         32 bytes       base = {pmpaddr[N-1:3], 5'b00000}
aaaa...0111         64 bytes       ...
...
aaaa0111...1111     2^(k+3) bytes  k trailing ones encodes size 2^(k+3)
0111...1111         max region     2^(PADDR_WIDTH-1) bytes from address 0
1111...1111         full address   entire physical address space (4 GiB on RV32)
```

To configure a NAPOT region of size `2^k` bytes starting at base address `base`:
```
pmpaddr = (base >> 2) | ((2^(k-3)) - 1)
        = (base | (2^(k-1) - 1)) >> 2

Example: 4 KiB region at 0x8000_0000
  size = 4096 = 2^12, k = 12
  pmpaddr = (0x80000000 >> 2) | ((2^(12-3)) - 1)
          = 0x20000000 | (512 - 1)
          = 0x20000000 | 0x1FF
          = 0x200001FF

Example: 256 MiB region at 0x8000_0000
  size = 2^28, k = 28
  pmpaddr = (0x80000000 >> 2) | (2^(28-3) - 1)
          = 0x20000000 | (2^25 - 1)
          = 0x20000000 | 0x1FFFFFF
          = 0x21FFFFFF
```

### TOR: Top Of Range

TOR uses two consecutive entries: entry i-1's `pmpaddr` is the base (inclusive), and entry i's `pmpaddr` is the top (exclusive). The protected range is `[pmpaddr[i-1] << 2, pmpaddr[i] << 2)`.

Entry 0 with TOR uses address 0 as the implicit lower bound.

TOR is useful for regions that are not power-of-two aligned or sized. Two PMP entries are consumed for one region, so it is less efficient than NAPOT for naturally aligned regions.

```
Example: Protect UART MMIO at 0x1000_0000 to 0x1000_0100 (256 bytes)
  pmpaddr0 = 0x1000_0000 >> 2 = 0x0400_0000   (base, entry 0)
  pmpaddr1 = 0x1000_0100 >> 2 = 0x0400_0040   (top exclusive, entry 1)
  pmpcfg1.A = TOR (01), permissions = R|W (no execute)
  pmpcfg0.A = OFF (entry 0 is only a base marker, not an active region)
```

### PMP Match Priority and Default Behaviour

PMP entries are checked in **priority order from 0 to N-1**. The first matching entry determines the permission. If no entry matches:
- S-mode and U-mode accesses raise an **access fault** (loads: cause 5; stores: cause 7; fetches: cause 1).
- M-mode accesses are **permitted** (M-mode bypasses PMP for unmatched accesses).

This default-deny behaviour for S/U-mode means: a system with zero configured PMP regions denies all S-mode and U-mode memory accesses. Any system that runs an OS must configure PMP entries that grant at least the minimal required access.

### The Lock Bit

Setting the L bit in a PMP entry has two effects:
1. The entry **also applies to M-mode** accesses. M-mode is no longer exempt from that entry's restrictions.
2. The `pmpcfg` and `pmpaddr` registers for the locked entry become **read-only** until the next reset. A write to a locked entry is silently ignored.

Locking is used in secure boot to protect firmware after it has finished initialisation:

```asm
# Lock OpenSBI firmware region [0x80000000, 0x80200000)
# after setup, preventing any mode from modifying firmware code/data
li    t0, (0x80000000 >> 2)
csrw  pmpaddr0, t0         # base = 0x80000000
li    t0, (0x80200000 >> 2)
csrw  pmpaddr1, t0         # top = 0x80200000

# Entry 1: TOR, locked, read+execute only (no write)
# pmpcfg0 contains config for entries 0-3 (RV32) or 0-7 (RV64)
# byte 0 = entry 0 config, byte 1 = entry 1 config
li    t0, (0x8D << 8)      # byte 1: L=1, A=TOR(01), X=1, W=0, R=1 = 0b1_00_01_101 = 0x8D
                            # byte 0: OFF (entry 0 is TOR base only) = 0x00
csrw  pmpcfg0, t0
# After this write, pmpaddr0 and pmpaddr1 and pmpcfg0 bytes 0-1 are LOCKED.
# Even M-mode cannot write firmware memory now.
```

### PMP and Page Faults: Ordering

Physical memory protection checks and virtual memory translation are separate pipeline stages. The order is:

```
Virtual Address
     |
     v
  MMU Walk (if satp.MODE != Bare)
  -> Page fault if PTE invalid/permissions denied
     |
     v
  Physical Address
     |
     v
  PMP Check
  -> Access fault if PMP denies access
     |
     v
  Physical Memory / MMIO
```

A page fault fires **before** the PMP check. This means:
- A U-mode access to an unmapped VA raises a page fault (not an access fault), regardless of PMP.
- A U-mode access to a mapped VA that is blocked by PMP raises an access fault.
- For M-mode (no MMU walk), the PMP check fires directly on the physical address.

---

## Tier 1 — Fundamentals

### Question F1
**What is the purpose of PMP? How does it differ from the MMU-based virtual memory protection?**

**Answer:**

PMP (Physical Memory Protection) enforces access control at the **physical address** level, before data reaches DRAM or MMIO. It is configured and controlled by M-mode, and its primary purpose is to restrict what S-mode and U-mode code can access.

Key differences from MMU-based virtual memory:

| Property | PMP | MMU (Sv39, etc.) |
|---|---|---|
| Address type | Physical | Virtual |
| Configured by | M-mode only | S-mode (via satp, page tables) |
| Granularity | Region-based (4 bytes to full space) | Page-based (4 KiB minimum) |
| Effective when | Always (even bare/no-MMU) | Only when satp.MODE != Bare |
| Overhead | Parallel hardware compare | Multi-level page table walk |
| Purpose | Isolate M-mode firmware from OS | Isolate processes from each other |

PMP is essential for systems that run without virtual memory (bare-metal RTOS, secure enclaves, early boot) and as a second layer of protection underneath the MMU. A compromised OS kernel cannot bypass PMP to read M-mode firmware memory — the hardware enforces the boundary regardless of page table contents.

---

### Question F2
**Describe the four PMP address matching modes (OFF, TOR, NA4, NAPOT) and when each is used.**

**Answer:**

**OFF (A=00):**
The PMP entry is disabled and does not match any address. Use when a PMP slot is unused or when temporarily disabling a region (note: locked entries cannot be changed).

**TOR — Top Of Range (A=01):**
Uses two PMP entries to define an arbitrary range `[pmpaddr[i-1]<<2, pmpaddr[i]<<2)`. The entry configured as TOR provides the upper bound; the previous entry provides the lower bound (its A field may be OFF). Useful for regions that are not power-of-two sized or not naturally aligned.

**NA4 — Naturally Aligned 4-byte (A=10):**
Protects exactly 4 consecutive bytes at the address `pmpaddr[i] << 2`. Rarely used in practice (only 4 bytes), but covers the case where NAPOT's minimum of 8 bytes is too coarse.

**NAPOT — Naturally Aligned Power-Of-Two (A=11):**
Encodes both base and size in a single `pmpaddr` register using trailing-ones encoding. The region size is 2^k bytes (minimum 8 bytes with k=3). This is the most efficient mode — one PMP entry covers one naturally aligned power-of-two region. Used for DRAM regions, flash regions, and MMIO blocks.

Choosing between TOR and NAPOT:
- NAPOT: use when the region is power-of-two sized and aligned. Costs one entry.
- TOR: use when the region is arbitrarily sized or aligned. Costs two entries (one for base, one for top).

---

### Question F3
**A RISC-V system has no PMP entries configured. A kernel running in S-mode tries to read physical address `0x8000_0000`. What happens?**

**Answer:**

The S-mode read fails with an **access fault** (load access fault, `mcause/scause = 5`).

When no PMP entries match an S-mode or U-mode access, RISC-V applies a **default deny** policy: the access is rejected. This is the opposite of M-mode, which has default allow (M-mode can access any physical address that is not covered by a locked PMP entry).

The access fault propagates as a trap. Depending on `medeleg`:
- If `medeleg[5] = 1` (load access fault delegated): the trap goes to S-mode. But since S-mode is already the privilege level of the faulting code, this would send the trap back to the S-mode trap handler — which would likely panic the kernel.
- If `medeleg[5] = 0`: the trap goes to M-mode firmware, which sees an unexpected access fault from S-mode and would typically reset the system or log a fatal error.

**Practical implication:** Any RISC-V system that boots an OS must configure PMP before jumping to S-mode. OpenSBI sets up at least one PMP entry granting S-mode and U-mode access to the full permitted physical address range (excluding firmware memory):

```asm
# Grant S/U mode access to all physical memory not protected by firmware
li    t0, -1                 # all ones
csrw  pmpaddr3, t0           # pmpaddr3 = 0xFFFFFFFF... (all-ones NAPOT = full space)
li    t0, 0x1F               # pmpcfg byte: L=0, A=NAPOT(11), X=1, W=1, R=1 = 0b0_00_11_111 = 0x1F
csrw  pmpcfg0, t0            # (shifted appropriately for the correct byte)
```

---

### Question F4
**What does the lock (L) bit in a PMP configuration byte do? Why is it irreversible until reset?**

**Answer:**

The L bit (bit 7 of the PMP configuration byte) has two effects when set:

1. **Extends protection to M-mode:** The PMP entry's address range and permissions apply to M-mode memory accesses as well as S/U-mode. Without L, M-mode bypasses PMP entirely for regions not marked locked.

2. **Makes the entry immutable:** Any write to the corresponding `pmpcfg` byte or `pmpaddr` register is **silently ignored**. The entry cannot be changed, disabled, or overridden.

The only way to unlock a locked PMP entry is a system reset.

Irreversibility is deliberate: it provides a hardware-enforced security guarantee. If M-mode firmware protects its own memory with a locked PMP entry, even a subsequently compromised M-mode runtime cannot remove that protection. This is critical for:

- **Secure boot**: Firmware locks its code/data region before transferring control to the OS. Even if the OS is malicious, it cannot modify firmware.
- **TEE (Trusted Execution Environment)**: A secure enclave locks its memory region and then transfers to normal-world software. The normal world cannot access enclave memory — guaranteed by hardware, not by normal-world code running at M-mode.
- **Trusted Platform Module (TPM) emulation**: M-mode locks the TPM state storage area so the OS cannot tamper with it.

---

## Tier 2 — Intermediate

### Question I1
**Write the PMP configuration to protect two regions: (a) a 64 KiB firmware ROM at `0x0000_0000` (execute-only, locked) and (b) a 512 KiB RAM region at `0x2000_0000` (read/write for S/U-mode, no execute). Show the pmpaddr and pmpcfg values.**

**Answer:**

Region (a): 64 KiB execute-only locked firmware at `0x0000_0000`
```
Size = 64 KiB = 2^16 bytes => NAPOT with k=16
pmpaddr = (0x0000_0000 >> 2) | (2^(16-3) - 1)
        = 0 | (2^13 - 1)
        = 0x1FFF
pmpcfg byte: L=1, A=NAPOT(11), X=1, W=0, R=0
           = 0b1_00_11_100 = 0x9C
```

Region (b): 512 KiB read/write RAM at `0x2000_0000`
```
Size = 512 KiB = 2^19 bytes => NAPOT with k=19
pmpaddr = (0x20000000 >> 2) | (2^(19-3) - 1)
        = 0x08000000 | (2^16 - 1)
        = 0x08000000 | 0xFFFF
        = 0x0800FFFF
pmpcfg byte: L=0, A=NAPOT(11), X=0, W=1, R=1
           = 0b0_00_11_011 = 0x1B
```

Assembly to configure (RV32, pmpcfg0 covers entries 0-3):
```asm
# Configure pmpaddr registers
li    t0, 0x00001FFF        # entry 0: firmware NAPOT
csrw  pmpaddr0, t0
li    t0, 0x0800FFFF        # entry 1: RAM NAPOT
csrw  pmpaddr1, t0

# Configure pmpcfg0: byte 0 = entry 0 config, byte 1 = entry 1 config
# byte 0 = 0x9C (firmware, locked, execute-only)
# byte 1 = 0x1B (RAM, RW, no execute)
# pmpcfg0 = 0x00001B9C (little-endian: byte0=0x9C, byte1=0x1B, bytes 2-3=0)
li    t0, 0x00001B9C
csrw  pmpcfg0, t0

# Verification: read back and check
csrr  t1, pmpcfg0
li    t2, 0x00001B9C
bne   t1, t2, pmp_config_error
```

After this, attempting to write to the firmware region raises a store access fault in all modes (L bit set). S/U-mode can execute from firmware. M-mode can also read/execute firmware but not write (L + X=1, W=0). The RAM region is read/write accessible to S/U-mode but not executable (X=0).

---

### Question I2
**Explain the PMP priority model. Two overlapping PMP entries exist: entry 0 covers `[0x1000, 0x2000)` with R=1, W=0, X=0, and entry 2 covers `[0x0000, 0x4000)` with R=1, W=1, X=1. A U-mode store to `0x1500` — which entry matches and is the store permitted?**

**Answer:**

RISC-V PMP evaluates entries **from lowest index (0) to highest**, and the **first matching entry wins**. No priority merging or combining of permissions occurs.

For a U-mode store to physical address `0x1500`:
- **Entry 0** matches: `0x1000 <= 0x1500 < 0x2000`. Entry 0 has W=0.
- Entry 2 also matches (`0x0000 <= 0x1500 < 0x4000`), but it is never consulted.

Result: Entry 0 wins with W=0. The store raises a **store access fault** (mcause = 7).

This priority model means you can create **exception carve-outs**: configure a large permissive region at a high index, then overlay a restrictive entry at a lower index to deny access to a sub-range.

```
Entry 0: [0x1000, 0x2000) R=1, W=0, X=0  -- restricted sub-range
Entry 1: [0x0000, 0x4000) R=1, W=1, X=1  -- broad permission (lower priority)

Accesses to [0x1000, 0x2000): entry 0 matches first => restricted
Accesses to [0x0000, 0x1000): entry 0 does not match; entry 1 matches => full
Accesses to [0x2000, 0x4000): entry 0 does not match; entry 1 matches => full
```

**Common mistake:** Assuming that a higher-numbered entry with broader permissions can "override" a lower-numbered restrictive entry. It cannot — lower index always wins on first match.

---

### Question I3
**A platform uses TOR mode to protect a non-power-of-two region. Configure a PMP entry protecting the range `[0x1000_0400, 0x1000_0900)` for S/U-mode read-only access. How many PMP entries are consumed?**

**Answer:**

TOR requires two entries: the previous entry's `pmpaddr` provides the lower bound, the TOR-mode entry's `pmpaddr` provides the exclusive upper bound.

```
Lower bound: 0x1000_0400
Upper bound: 0x1000_0900 (exclusive)

pmpaddr[i-1] = 0x1000_0400 >> 2 = 0x0400_0100
pmpaddr[i]   = 0x1000_0900 >> 2 = 0x0400_0240

pmpcfg[i-1]: A=OFF (0b00), permissions don't matter = 0x00
pmpcfg[i]:   A=TOR (0b01), X=0, W=0, R=1 = 0b0_00_01_001 = 0x09
```

Assembly (using entries 2 and 3 as an example):
```asm
li    t0, 0x04000100    # lower bound (entry 2, used as TOR base)
csrw  pmpaddr2, t0
li    t0, 0x04000240    # upper bound (entry 3, the TOR entry)
csrw  pmpaddr3, t0

# pmpcfg0 on RV32 covers entries 0-3
# Need to set byte 2 = 0x00 (entry 2, OFF) and byte 3 = 0x09 (entry 3, TOR R-only)
# Read-modify-write to avoid disturbing entries 0 and 1
csrr  t0, pmpcfg0
li    t1, 0xFF00_0000   # mask bytes 2 and 3
not   t1, t1
and   t0, t0, t1        # clear bytes 2 and 3
li    t1, 0x0900_0000   # byte 3 = TOR R-only (0x09), byte 2 = OFF (0x00)
                        # Note: byte 3 is bits [31:24], byte 2 is bits [23:16]
                        # 0x09 << 24 = 0x0900_0000
or    t0, t0, t1
csrw  pmpcfg0, t0
```

Two PMP entries are consumed (entries 2 and 3). The region `[0x1000_0400, 0x1000_0900)` — 1280 bytes, non-power-of-two — is now read-only for S/U-mode. Any store or fetch raises an access fault.

---

## Tier 3 — Advanced

### Question A1
**Design a PMP configuration for a secure boot scenario with the following requirements: (1) ROM at `[0x0000_0000, 0x0001_0000)`: execute-only, locked. (2) OpenSBI firmware at `[0x8000_0000, 0x8020_0000)`: R/W/X for M-mode only (no S/U access), locked. (3) All remaining physical memory: R/W/X for S/U-mode. Explain your entry choices and the order in which they must be configured.**

**Answer:**

Region analysis:

**Region 1: ROM `[0x0000_0000, 0x0001_0000)`**
- Size: 64 KiB = 2^16 bytes, naturally aligned => NAPOT
- Permissions: X only, locked (applies to M-mode too)
- `pmpaddr0 = (0x00000000 >> 2) | (2^13 - 1) = 0x1FFF`
- `pmpcfg byte 0 = L(1) A(11) X(1) W(0) R(0) = 0b1_00_11_100 = 0x9C`

**Region 2: OpenSBI firmware `[0x8000_0000, 0x8020_0000)`**
- Size: 2 MiB = 2^21 bytes, naturally aligned => NAPOT
- Permissions: None for S/U-mode (deny all), locked
- When L=1 and all of R/W/X = 0: M-mode is also denied — this is wrong for firmware. We want M-mode to access it but deny S/U.
- Solution: `L=0, R=0, W=0, X=0` — this denies S/U-mode (zero permissions), and M-mode bypasses PMP (L=0, M-mode not checked).
- `pmpaddr1 = (0x80000000 >> 2) | (2^(21-3) - 1) = 0x20000000 | 0x3FFFF = 0x2003FFFF`
- `pmpcfg byte 1 = L(0) A(11) X(0) W(0) R(0) = 0b0_00_11_000 = 0x18`
- Note: L=0 here means M-mode access is not subject to this entry. S/U-mode matches and gets denied.

**Region 3: All remaining physical memory — grant S/U access**
- The entire address space (minus the above regions which match first).
- NAPOT with all ones encodes the full address space.
- `pmpaddr2 = 0xFFFFFFFF` (all ones = full address space on RV32)
- `pmpcfg byte 2 = L(0) A(11) X(1) W(1) R(1) = 0b0_00_11_111 = 0x1F`

Complete configuration (pmpcfg0 on RV32):
```asm
# Step 1: Configure address registers BEFORE pmpcfg
# (writing pmpaddr of a locked entry is silently ignored,
#  so locked entries must be written to pmpaddr first, then pmpcfg)

li    t0, 0x00001FFF       # ROM: NAPOT, 64 KiB at 0x0
csrw  pmpaddr0, t0
li    t0, 0x2003FFFF       # OpenSBI: NAPOT, 2 MiB at 0x8000_0000
csrw  pmpaddr1, t0
li    t0, 0xFFFFFFFF       # All memory: full address space
csrw  pmpaddr2, t0

# Step 2: Write pmpcfg atomically
# byte 0 = 0x9C (ROM: locked, NAPOT, execute-only)
# byte 1 = 0x18 (OpenSBI: NAPOT, no permissions -- blocks S/U, M bypasses)
# byte 2 = 0x1F (all memory: NAPOT, RWX for S/U)
# byte 3 = 0x00 (unused)
li    t0, 0x001F189C
csrw  pmpcfg0, t0

# After this write, entries 0 is locked (L=1 in byte 0).
# Entry 1 (OpenSBI) is NOT locked -- but the firmware code is protecting
# itself by being in the OpenSBI entry which denies S/U access.
# If higher security is needed, set L=1 for entry 1 also
# (at the cost of M-mode also being denied unless a higher-priority entry allows it).
```

**Critical ordering constraint:** PMP address registers must be written **before** their corresponding pmpcfg bytes are written with A != OFF. Writing pmpcfg activates the entry, and from that moment the hardware enforces it. If you write pmpcfg first, the entry may activate with a stale/incorrect pmpaddr. More critically, once an entry is locked (L=1), subsequent writes to pmpaddr are silently dropped — so pmpaddr must be configured before the pmpcfg write that sets L=1.

**Security note on Region 2:** Using `L=0` for the OpenSBI region means a malicious OS kernel running in S-mode cannot access OpenSBI memory (S-mode is blocked by PMP entry 1 with R=W=X=0). However, if the OS kernel somehow escalates to M-mode (e.g., via a firmware vulnerability), it could modify entry 1's pmpcfg since L=0. For the highest security, set L=1 on entry 1 as well. This also blocks M-mode from the region — but that is acceptable if OpenSBI is finished with its own setup.

---

### Question A2
**A security researcher finds that on a certain RISC-V SoC, a U-mode process can read M-mode firmware memory despite PMP being configured. They suspect the PMP check is not applied to instruction fetch. Explain the RISC-V specification requirements for PMP fetch checking, and describe two microarchitectural scenarios where a buggy implementation could allow this bypass.**

**Answer:**

**Specification requirements:**
The RISC-V privileged specification (Volume II, Section 3.7) states that PMP checks apply to **all three access types**: instruction fetch, data load, and data store. There is no exception for instruction fetch. An instruction fetch from a physical address that is blocked by PMP for the current privilege level must raise an **instruction access fault** (mcause = 1).

This applies to:
- Any fetch from S-mode or U-mode that matches a PMP entry with X=0.
- Any fetch in any mode from a locked PMP entry with X=0.

**Microarchitectural bypass scenario 1: Split PMP check pipeline**

Some implementations implement a "fast path" where the instruction fetch unit checks the PMP **speculatively** before the privilege level is confirmed (e.g., because the privilege is determined by a register writeback several cycles earlier). A race condition in the control logic could cause:
- The fetch to the firmware address completes physically before the PMP check fires.
- The instruction data enters the pipeline's instruction buffer.
- Even if the PMP check eventually raises a fault, the instruction data may already be in a buffer readable via a side channel (cache timing, power analysis).

The fix: PMP checks must be evaluated on the **resolved** privilege level, and the instruction data must not be used (or made observable) before the PMP check passes.

**Microarchitectural bypass scenario 2: Cached TLB/PMP results with stale privilege**

In a processor with a micro-TLB or PMP result cache that caches (VA, privilege, PMP result) tuples, a context switch bug could leave stale PMP-pass results for M-mode accesses in the cache. A subsequent U-mode access to the same physical address might hit the cached "permitted" result without re-checking the current (lower) privilege level.

Correct implementation: PMP caches must be tagged with the privilege level, or must be flushed on any privilege level change.

**Scenario 3: MPRV bypass**

If `mstatus.MPRV=1` and `mstatus.MPP=M`, an M-mode instruction using a load effectively acts as M-mode for the data access (PMP bypass allowed). A buggy implementation might incorrectly apply MPRV semantics to instruction fetches as well, allowing M-mode to "fetch" from a locked region by setting MPP=M. The specification is clear: MPRV only affects load/store, never instruction fetch.

Detection methods include: running a U-mode test that attempts to fetch from a PMP-protected region and verifying an access fault is taken, rather than a successful instruction execution.
