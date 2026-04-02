# Problem 03: PMP Configuration for Secure Boot

**Topic:** Privilege Architecture — Physical Memory Protection  
**Difficulty:** Advanced  
**Estimated time:** 30 minutes

---

## Problem Statement

You are implementing the secure boot stage for an RV32 embedded SoC. The physical memory map is:

```
0x0000_0000 - 0x0000_FFFF   Boot ROM (64 KiB): read-only, execute
0x0001_0000 - 0x0001_FFFF   Reserved (16 KiB): no access permitted to any mode
0x2000_0000 - 0x2000_0FFF   UART0 MMIO (4 KiB): R/W for kernel, no execute
0x2000_1000 - 0x2000_1FFF   Timer MMIO (4 KiB): M-mode only, no S/U access
0x8000_0000 - 0x8001_FFFF   Secure firmware (128 KiB): M-mode only, locked
0x8002_0000 - 0x8003_FFFF   Kernel image (128 KiB): S-mode R/W/X
0x8004_0000 - 0x800F_FFFF   Application RAM (768 KiB): S/U-mode R/W, no execute
```

All other physical addresses not listed above are inaccessible to S/U-mode.

The SoC implements 16 PMP entries (pmpaddr0–pmpaddr15, pmpcfg0–pmpcfg3 on RV32).

**Part A:** For each memory region, determine the optimal PMP address matching mode (OFF, TOR, NA4, NAPOT) and calculate the `pmpaddr` register value. Justify your choice of NAPOT vs TOR.

**Part B:** Determine the `pmpcfg` byte for each active PMP entry. Show the bit-field breakdown.

**Part C:** Write the complete assembly initialisation sequence. Entries protecting firmware must be locked. Be precise about register layout: state which byte of which `pmpcfg` CSR holds each entry's configuration.

**Part D:** After your PMP is configured and the secure firmware locks itself, a U-mode application attempts to read from `0x8001_0000` (inside the locked firmware region). Trace the access attempt and identify the fault type, cause code, and which CSR holds the faulting address.

**Part E:** The kernel needs to add a new 256-byte I2C MMIO region at `0x2000_2100` dynamically, after the initial PMP setup. Which PMP entry is available and how do you configure it? What constraint does the dynamic addition face given the existing locked entry?

---

## Solution

### Part A: Address Matching Mode Selection

**Region 1: Boot ROM `[0x0000_0000, 0x0001_0000)` — 64 KiB**
```
Size = 64 KiB = 2^16 bytes. Base = 0x0000_0000 (naturally aligned to 2^16).
=> NAPOT (one entry, naturally aligned power-of-two)

pmpaddr = (0x0000_0000 >> 2) | (2^(16-3) - 1)
        = 0 | (8192 - 1)
        = 0x1FFF
```

**Region 2: Reserved `[0x0001_0000, 0x0002_0000)` — 16 KiB**
```
Size = 16 KiB = 2^14 bytes. Base = 0x0001_0000 (naturally aligned to 2^14).
=> NAPOT

pmpaddr = (0x0001_0000 >> 2) | (2^(14-3) - 1)
        = 0x4000 | (2048 - 1)
        = 0x4000 | 0x7FF
        = 0x47FF
```

**Region 3: UART0 MMIO `[0x2000_0000, 0x2000_1000)` — 4 KiB**
```
Size = 4 KiB = 2^12 bytes. Base = 0x2000_0000 (naturally aligned to 4 KiB).
=> NAPOT (minimum NAPOT size is 8 bytes; 4 KiB qualifies)

pmpaddr = (0x2000_0000 >> 2) | (2^(12-3) - 1)
        = 0x0800_0000 | (512 - 1)
        = 0x0800_0000 | 0x1FF
        = 0x0800_01FF
```

**Region 4: Timer MMIO `[0x2000_1000, 0x2000_2000)` — 4 KiB**
```
Size = 4 KiB = 2^12 bytes. Base = 0x2000_1000 (naturally aligned to 4 KiB).
=> NAPOT

pmpaddr = (0x2000_1000 >> 2) | (2^(12-3) - 1)
        = 0x0800_0400 | 0x1FF
        = 0x0800_05FF
```

**Region 5: Secure firmware `[0x8000_0000, 0x8002_0000)` — 128 KiB**
```
Size = 128 KiB = 2^17 bytes. Base = 0x8000_0000 (naturally aligned to 2^17).
=> NAPOT (must be locked)

pmpaddr = (0x8000_0000 >> 2) | (2^(17-3) - 1)
        = 0x2000_0000 | (2^14 - 1)
        = 0x2000_0000 | 0x3FFF
        = 0x2000_3FFF
```

**Region 6: Kernel image `[0x8002_0000, 0x8004_0000)` — 128 KiB**
```
Size = 128 KiB = 2^17 bytes. Base = 0x8002_0000 (naturally aligned to 2^17).
=> NAPOT

pmpaddr = (0x8002_0000 >> 2) | (2^(17-3) - 1)
        = 0x2000_8000 | 0x3FFF
        = 0x2000_BFFF
```

**Region 7: Application RAM `[0x8004_0000, 0x8010_0000)` — 768 KiB**
```
Size = 768 KiB. Is this a power of two? 768 = 512 + 256 = not a power of two.
=> Cannot use single NAPOT entry.
=> Options:
   (a) TOR: two entries, covers exactly [0x8004_0000, 0x8010_0000)
   (b) Two NAPOT entries: 512 KiB at 0x8004_0000 + 256 KiB at 0x800C_0000

Option (a) — TOR uses 2 entries, exact coverage.
Option (b) — NAPOT uses 2 entries, also exact coverage.

Check (b) alignment:
  512 KiB at 0x8004_0000: 2^19, base must be 2^19-aligned.
    0x8004_0000 / 0x80000 = 0x1001 — not a multiple of 512 KiB boundary.
    0x8004_0000 mod 0x80000 = 0x4000 != 0. NOT naturally aligned for 512 KiB.
  Try 256 KiB at 0x8004_0000 + 256 KiB at 0x8008_0000 + 256 KiB at 0x800C_0000:
    That uses 3 entries and 0x800C_0000 only goes to 0x8010_0000 (256 KiB). That covers it.
    But 3 entries for one region is wasteful.

Best choice: TOR with 2 entries (lower/upper bound), exact coverage regardless of alignment.
  pmpaddr_base = 0x8004_0000 >> 2 = 0x2001_0000  (lower bound, entry with A=OFF)
  pmpaddr_top  = 0x8010_0000 >> 2 = 0x2004_0000  (upper bound exclusive, TOR entry)
```

### Part B: pmpcfg Byte Values

| Entry | Region | A mode | L | R | W | X | pmpcfg byte |
|---|---|---|---|---|---|---|---|
| 0 | Boot ROM | NAPOT | 0 | 1 | 0 | 1 | `0b0_00_11_101` = `0x1D` |
| 1 | Reserved | NAPOT | 0 | 0 | 0 | 0 | `0b0_00_11_000` = `0x18` |
| 2 | UART0 | NAPOT | 0 | 1 | 1 | 0 | `0b0_00_11_011` = `0x1B` |
| 3 | Timer MMIO | NAPOT | 0 | 0 | 0 | 0 | `0b0_00_11_000` = `0x18` |
| 4 | Secure FW | NAPOT | 1 | 0 | 0 | 0 | `0b1_00_11_000` = `0x98` |
| 5 | Kernel image | NAPOT | 0 | 1 | 1 | 1 | `0b0_00_11_111` = `0x1F` |
| 6 | App RAM base | OFF | 0 | 0 | 0 | 0 | `0b0_00_00_000` = `0x00` |
| 7 | App RAM TOR | TOR | 0 | 1 | 1 | 0 | `0b0_00_01_011` = `0x0B` |

Notes on permission choices:
- **Boot ROM**: R=1, X=1, W=0. Read-only executable. Not locked (M-mode already bypasses PMP; if locked with W=0, M-mode also cannot write — acceptable for ROM since it is physically read-only anyway).
- **Reserved**: R=0, W=0, X=0, A=NAPOT. Any S/U access raises an access fault. M-mode is not blocked (L=0).
- **Timer MMIO**: R=0, W=0, X=0 — denies all S/U access to the timer registers. The M-mode SBI handles timer via `sbi_set_timer`.
- **Secure FW**: L=1, R=0, W=0, X=0. Locked, all permissions denied. Because L=1, this also applies to M-mode — preventing even compromised M-mode code from reading or writing firmware after lockdown. Firmware copies itself into RAM and executes from there before locking.
- **Kernel image**: R=1, W=1, X=1. The kernel needs to write itself (BSS zeroing, relocation), so W=1 during boot; ideally W would be cleared post-relocation but that requires an additional PMP update.
- **App RAM**: TOR, R=1, W=1, X=0. User processes read/write application data. Execute is denied to prevent code injection into data pages (W^X policy). Note: for full W^X, kernel image should also have W cleared, but that complicates initial loading.

### Part C: Assembly Initialisation Sequence

On RV32, `pmpcfg0` covers entries 0–3, `pmpcfg1` covers entries 4–7.

```asm
.section .text.pmp_init
.global pmp_init

pmp_init:
    # =========================================================
    # Step 1: Configure all pmpaddr registers BEFORE pmpcfg.
    # Locked entries (entry 4) cannot be changed after pmpcfg
    # is written with L=1. Set pmpaddr first.
    # =========================================================

    # Entry 0: Boot ROM NAPOT (64 KiB at 0x0000_0000)
    li    t0, 0x00001FFF
    csrw  pmpaddr0, t0

    # Entry 1: Reserved NAPOT (16 KiB at 0x0001_0000)
    li    t0, 0x000047FF
    csrw  pmpaddr1, t0

    # Entry 2: UART0 NAPOT (4 KiB at 0x2000_0000)
    li    t0, 0x080001FF
    csrw  pmpaddr2, t0

    # Entry 3: Timer MMIO NAPOT (4 KiB at 0x2000_1000)
    li    t0, 0x080005FF
    csrw  pmpaddr3, t0

    # Entry 4: Secure firmware NAPOT (128 KiB at 0x8000_0000)
    # MUST be written before pmpcfg1 locks it!
    li    t0, 0x20003FFF
    csrw  pmpaddr4, t0

    # Entry 5: Kernel image NAPOT (128 KiB at 0x8002_0000)
    li    t0, 0x2000BFFF
    csrw  pmpaddr5, t0

    # Entry 6: App RAM TOR lower bound (OFF entry, base = 0x8004_0000)
    li    t0, 0x20010000
    csrw  pmpaddr6, t0

    # Entry 7: App RAM TOR upper bound (exclusive = 0x8010_0000)
    li    t0, 0x20040000
    csrw  pmpaddr7, t0

    # Entries 8-15: unused (will be left as reset state, A=OFF)

    # =========================================================
    # Step 2: Write pmpcfg registers.
    # pmpcfg0 = entries 3,2,1,0 (byte 3 = entry 3, byte 0 = entry 0)
    # pmpcfg1 = entries 7,6,5,4
    # Layout (little-endian byte packing, RV32):
    #   pmpcfg0[7:0]   = entry 0 config
    #   pmpcfg0[15:8]  = entry 1 config
    #   pmpcfg0[23:16] = entry 2 config
    #   pmpcfg0[31:24] = entry 3 config
    # =========================================================

    # pmpcfg0:
    #   byte 0 (entry 0): 0x1D  Boot ROM NAPOT, R/X
    #   byte 1 (entry 1): 0x18  Reserved NAPOT, no permissions
    #   byte 2 (entry 2): 0x1B  UART0 NAPOT, R/W
    #   byte 3 (entry 3): 0x18  Timer NAPOT, no permissions (M-mode only)
    # pmpcfg0 = 0x181B_181D
    li    t0, 0x181B181D
    csrw  pmpcfg0, t0

    # pmpcfg1:
    #   byte 0 (entry 4): 0x98  Secure FW NAPOT, LOCKED, no S/U permissions
    #   byte 1 (entry 5): 0x1F  Kernel NAPOT, R/W/X
    #   byte 2 (entry 6): 0x00  App RAM TOR base, OFF
    #   byte 3 (entry 7): 0x0B  App RAM TOR, R/W
    # pmpcfg1 = 0x0B001F98
    li    t0, 0x0B001F98
    csrw  pmpcfg1, t0

    # Entries 8-15 default to OFF (A=0), so pmpcfg2 and pmpcfg3
    # can be left at their reset values (or explicitly cleared):
    csrw  pmpcfg2, zero
    csrw  pmpcfg3, zero

    # =========================================================
    # Step 3: Verification — read back and check key entries.
    # Entry 4 (secure FW) should now be locked and immutable.
    # =========================================================

    csrr  t0, pmpcfg1
    li    t1, 0x0B001F98
    bne   t0, t1, pmp_config_error

    csrr  t0, pmpaddr4
    li    t1, 0x20003FFF
    bne   t0, t1, pmp_config_error

    # Attempt to overwrite locked entry 4 (should be silently ignored)
    li    t0, 0xDEADBEEF
    csrw  pmpaddr4, t0        # silently ignored (locked)
    csrr  t1, pmpaddr4
    li    t2, 0x20003FFF
    bne   t1, t2, pmp_lock_verification_failed  # lock NOT working if we reach here

    ret

pmp_config_error:
    # Handle configuration mismatch (e.g., log error, halt)
    j     halt

pmp_lock_verification_failed:
    # The lock bit did not take effect — hardware bug or platform issue
    j     halt
```

### Part D: U-mode Read from Locked Firmware Region `0x8001_0000`

Physical address `0x8001_0000` falls within `[0x8000_0000, 0x8002_0000)` — this is PMP entry 4 (secure firmware, NAPOT, L=1, R=0, W=0, X=0).

**PMP matching and priority:**

Entries are checked from 0 to 15. Entries 0–3 cover different address ranges that do not include `0x8001_0000`. Entry 4 covers `[0x8000_0000, 0x8002_0000)`.

```
Entry 4 matches: 0x8000_0000 <= 0x8001_0000 < 0x8002_0000 ✓
L=1, R=0, W=0, X=0
Current privilege: U-mode

L=1 means this entry applies to ALL modes.
R=0 => read not permitted.
```

**Result: Load access fault**

```
Fault type:  Load access fault (physical address blocked by PMP)
scause:      5  (load access fault — if medeleg[5]=1, else mcause=5 in M-mode)
stval:       0x8001_0000  (faulting physical address)
sepc:        PC of the load instruction
```

Note: this is an **access fault**, NOT a page fault. Page faults occur during MMU virtual address translation. Access faults occur when the physical address is denied by PMP (or a bus error). The distinction matters because:
- The kernel handles page faults by potentially mapping the page (demand paging).
- The kernel handles access faults as unrecoverable errors (the physical address is hardwired to be inaccessible). A U-mode access fault results in SIGSEGV being sent to the process.

Also: because `L=1` on entry 4, even if an attacker somehow escalated to M-mode, reading `0x8001_0000` would still raise an M-mode access fault. The physical protection is absolute until the next reset.

### Part E: Adding I2C MMIO Dynamically

**Available entries:**
Entries 8–15 are unused (A=OFF). Entry 8 is the lowest available.

**I2C region:** `[0x2000_2100, 0x2000_2200)` — 256 bytes

Is 256 bytes a power of two? Yes: 256 = 2^8. Is `0x2000_2100` aligned to 256 bytes?
```
0x2000_2100 mod 0x100 = 0x100 mod 0x100 = 0 ✓
(0x2000_2100 ends in 0x100, which is 256-byte aligned)
```

NAPOT is possible: size = 2^8, k=8.
```
pmpaddr8 = (0x20002100 >> 2) | (2^(8-3) - 1)
         = 0x08000840 | (32 - 1)
         = 0x08000840 | 0x1F
         = 0x0800085F
```

pmpcfg byte for entry 8: L=0, A=NAPOT, R=1, W=1, X=0 = `0x1B`

```asm
# Dynamic addition at runtime (S-mode or M-mode, depending on who manages PMP)
# Note: pmpcfg2 covers entries 8-11 on RV32

li    t0, 0x0800085F
csrw  pmpaddr8, t0

# Read-modify-write pmpcfg2 to set byte 0 (entry 8) without disturbing entries 9-11
csrr  t0, pmpcfg2
andi  t0, t0, ~0xFF         # clear byte 0 (entry 8 config)
ori   t0, t0, 0x1B          # set entry 8: NAPOT, R/W, no execute
csrw  pmpcfg2, t0
```

**Constraint due to locked entry:**

The locked entries (entry 4: pmpaddr4, pmpcfg1 byte 0) cannot be modified, but that does not affect the I2C configuration — locked entries do not prevent other entries from being written. The constraint here is:

1. **Priority ordering:** If I2C address `[0x2000_2100, 0x2000_2200)` should be inaccessible in some modes, the I2C entry must be placed at a lower index than any broader permissive entry. Since entries 0–7 handle other specific regions and entry 8 is the first free entry, and no broad "allow all" entry exists at a lower index, this is fine.

2. **No overlap with locked entry 4:** The I2C MMIO at `0x2000_2100` does not overlap `[0x8000_0000, 0x8002_0000)`. No conflict.

3. **Future locking:** If the I2C entry were later locked (`L=1` in pmpcfg2 byte 0), it would also become immutable. This matters if the I2C address might need to change (e.g., for dynamic bus probing). For peripherals with fixed MMIO addresses, locking is acceptable for security.

---

## Discussion

### Design Decisions and Trade-offs

**Why lock only the firmware entry?**
Locking all PMP entries would prevent the OS from dynamically managing MMIO regions (adding new peripherals, changing UART permissions). Only the security-critical firmware region (which must not be tampered with post-boot) is locked. Other entries remain modifiable by M-mode for runtime configuration.

**Why deny execute on MMIO and RAM?**
Setting X=0 on data regions enforces a hardware-level Write-XOR-Execute (W^X) policy — the processor cannot fetch instructions from MMIO registers or application data pages. This prevents code injection attacks where an attacker writes shellcode to a data buffer and redirects execution to it. PMP X bit enforcement occurs at the physical level, bypassing any software page table manipulation.

**Why does entry 6 (App RAM TOR lower bound) have A=OFF?**
In TOR mode, entry i-1 provides the lower bound and entry i provides the upper bound. The lower-bound entry does not itself define an active protected region — it merely establishes the base address. Its A=OFF means "this entry is not an active region, just a boundary marker for the next TOR entry." If entry 6 had non-OFF permissions, it would be checked independently as an active region for any address below its own `pmpaddr` value, which could cause unintended matches.

### NAPOT vs TOR Decision Matrix

| Scenario | Use NAPOT | Use TOR |
|---|---|---|
| Region is power-of-two sized AND naturally aligned | Yes | No |
| Region is power-of-two sized but NOT aligned | No | Yes |
| Region is not power-of-two sized | No | Yes |
| PMP entries are scarce | Prefer (1 entry) | Avoid (2 entries) |
| Address range has arbitrary bounds | No | Yes |
