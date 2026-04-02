# Toolchain: GCC and LLVM

## Prerequisites
- Familiarity with the C compilation pipeline: preprocessing, compilation, assembly, linking
- Basic RISC-V instruction formats and register file (rv32i / rv64i)
- Understanding of ELF object file structure at a high level

---

## Concept Reference

### The riscv-gnu-toolchain Component Stack

The riscv-gnu-toolchain project (hosted at github.com/riscv-collab/riscv-gnu-toolchain) assembles the following components into a coherent cross-compilation environment:

```
Source files (.c / .cpp / .S)
          |
   [ GCC front-end / cc1 ]         -- C/C++/Fortran front-end, produces GIMPLE IR
          |
   [ GCC middle-end ]              -- architecture-independent optimisations (O1/O2/O3)
          |
   [ GCC back-end: riscv.md ]      -- machine description, RTL -> RISC-V assembly
          |
   [ GNU Assembler (gas, as) ]     -- .s -> .o, part of binutils
          |
   [ GNU Linker (ld) ]             -- links .o + libc -> ELF executable, uses linker script
          |
   [ ELF binary ]
          |
   [ objcopy / objdump / nm ]      -- inspection and conversion utilities (binutils)
```

**Key packages:**

| Package       | Role                                                             |
|---------------|------------------------------------------------------------------|
| gcc           | Compiler driver and back-end code generator                      |
| binutils      | Assembler (as), linker (ld), objcopy, objdump, nm, readelf, strip|
| glibc / newlib| C standard library (glibc for Linux targets, newlib for bare-metal)|
| libgcc        | Low-level runtime: soft-float, 64-bit arithmetic on RV32, divide |
| gdb           | Source-level debugger, RISC-V remote target support              |
| gdbserver     | Runs on target, communicates with host GDB over RSP protocol     |

### Target Tuple Naming Convention

The toolchain prefix encodes the target completely:

```
riscv64-unknown-linux-gnu-gcc
  |       |       |     |
  |       |       |     +-- ABI / OS layer (gnu = glibc + Linux syscalls)
  |       |       +-------- Operating system (linux)
  |       +---------------- Vendor (unknown = no specific vendor)
  +------------------------ Architecture (riscv64)

Common prefixes:
  riscv64-unknown-linux-gnu-    Linux application (glibc, dynamic linking)
  riscv32-unknown-elf-          Bare-metal / RTOS (newlib, static linking)
  riscv64-unknown-elf-          Bare-metal 64-bit
  riscv32-unknown-linux-gnu-    32-bit Linux (rare; most Linux targets are rv64)
```

### The -march Flag: Architecture String

The `-march` flag specifies exactly which ISA extensions the compiler may emit. It is the primary mechanism for controlling code generation.

```
Syntax:  -march=rv{XLEN}{extensions}

XLEN:    32 or 64 (base word size)

Base ISA:
  i      RV32I / RV64I (integer base, always present)
  e      RV32E (embedded, 16 registers — rare)

Standard single-letter extensions:
  m      Integer multiply/divide
  a      Atomics (LR/SC, AMO)
  f      Single-precision floating-point
  d      Double-precision floating-point (implies f)
  c      Compressed 16-bit instructions
  b      Bit manipulation (Zba, Zbb, Zbc, Zbs sub-extensions)
  v      Vector (RISC-V V spec 1.0)

Sub-extensions (Zxxx notation, underscore-separated after main string):
  _zicsr       CSR instructions (implicit in most profiles)
  _zifencei    FENCE.I instruction
  _zba         Address generation bit manipulation
  _zbb         Basic bit manipulation
  _zbc         Carry-less multiplication
  _zbs         Single-bit manipulation
  _zbkb        Bit manipulation for cryptography
  _zkn         NIST cryptography suite

Examples:
  -march=rv32imac          RV32 with M, A, C — typical embedded MCU
  -march=rv32imafc         Add single-precision FPU
  -march=rv64gc            RV64 G (=IMAFD) + C — standard Linux target
  -march=rv64gcv           Add V extension (vector)
  -march=rv32imac_zba_zbb  Add bit manipulation sub-extensions
```

**Common mistake:** Specifying `-march=rv64gc` without `-mabi=lp64d` causes a mismatch — the compiler generates FPU instructions but the ABI is wrong for passing/returning floating-point values in FPU registers.

### The -mabi Flag: Application Binary Interface

The `-mabi` flag controls the calling convention and the handling of floating-point values across function call boundaries.

```
-mabi=ilp32      RV32, integer registers only for args/return values
                 Soft-float: float/double passed in integer registers
                 Required if -march has no F or D extension

-mabi=ilp32f     RV32, single-precision FP args in f registers
                 Hard-float single; requires -march with 'f'

-mabi=ilp32d     RV32, double-precision FP args in f registers
                 Hard-float double; requires -march with 'd'

-mabi=lp64       RV64, integer registers only for FP args (soft-float)
-mabi=lp64f      RV64, single-precision FP in FP registers
-mabi=lp64d      RV64, double-precision FP in FP registers (standard Linux)

Naming breakdown (ilp32d):
  i   = int     is 32 bits
  l   = long    is 32 bits (ilp32) or 64 bits (lp64)
  p   = pointer is 32 bits (ilp32) or 64 bits (lp64)
  32  = 32-bit
  d   = doubles in FP registers
```

**Critical rule:** All object files and libraries linked together must use the same `-mabi`. Mixing ABIs produces a linker error:

```
ld: error: cannot link object files with incompatible ABIs:
    cannot mix ilp32 with ilp32d
```

### Multilib: Handling Multiple ABI/ISA Combinations

A toolchain pre-builds the standard library (libc, libgcc, libstdc++) for several common `-march`/`-mabi` combinations. This is called **multilib** support.

```bash
# List available multilib variants in the installed toolchain:
riscv32-unknown-elf-gcc -print-multi-lib

# Example output (riscv-gnu-toolchain newlib build):
.;
rv32e/ilp32e;@march=rv32e@mabi=ilp32e
rv32em/ilp32e;@march=rv32em@mabi=ilp32e
rv32ema/ilp32e;@march=rv32ema@mabi=ilp32e
rv32i/ilp32;@march=rv32i@mabi=ilp32
rv32im/ilp32;@march=rv32im@mabi=ilp32
rv32iac/ilp32;@march=rv32iac@mabi=ilp32
rv32imac/ilp32;@march=rv32imac@mabi=ilp32
rv32imafc/ilp32f;@march=rv32imafc@mabi=ilp32f
rv32imafdc/ilp32d;@march=rv32imafdc@mabi=ilp32d
rv64imac/lp64;@march=rv64imac@mabi=lp64
rv64imafdc/lp64d;@march=rv64imafdc@mabi=lp64d
```

When you compile with a specific `-march`/`-mabi`, GCC searches this list (longest prefix match) and links with the appropriate pre-built library variant. If your combination is not in the multilib list, GCC falls back to the default, which may produce mismatched-ABI link errors.

**Practical consequence:** If you add a custom extension (e.g., `-march=rv32imac_xcustom`), you must either:
1. Ensure the extension does not change the calling convention (it usually does not), so GCC falls back to the `rv32imac` multilib, or
2. Build and register a new multilib variant for your target.

### LLVM/Clang for RISC-V

LLVM has first-class RISC-V support. The LLVM RISC-V backend targets the same `-march`/`-mabi` flags as GCC for compatibility.

```
Compiler:    clang (replaces gcc)
Assembler:   llvm-as or integrated assembler (replaces gas)
Linker:      lld (replaces ld; faster, better error messages)
Disassembler: llvm-objdump (replaces objdump)
Archiver:    llvm-ar

Key clang flags for RISC-V cross-compilation:
  --target=riscv32-unknown-elf    Target triple (replaces gcc prefix)
  -march=rv32imac                 Same as GCC
  -mabi=ilp32                     Same as GCC
  --sysroot=/path/to/sysroot      Root for headers and libraries
  -fuse-ld=lld                    Use lld as linker

Example:
  clang --target=riscv32-unknown-elf \
        -march=rv32imac \
        -mabi=ilp32 \
        -nostdlib \
        -T link.ld \
        main.c crt0.S -o firmware.elf
```

**LLVM advantages over GCC for RISC-V:**

| Property              | GCC                            | LLVM / Clang                        |
|-----------------------|--------------------------------|--------------------------------------|
| Custom extension support | GCC plugins, complex to add | TableGen (.td files), well-documented|
| LTO quality           | Moderate (GCC LTO)             | High (LLVM bitcode-based LTO)        |
| Build times           | Faster for single files        | Faster for whole-program LTO         |
| Intrinsic model       | `__builtin_` functions         | `__builtin_` + LLVM intrinsic IR     |
| Inline assembly       | AT&T-style constraints         | Same AT&T-style, compatible          |
| Error messages        | Improving but variable         | Generally clearer                    |
| Custom target         | Requires GCC backend fork      | LLVM out-of-tree target supported    |

### Linker Scripts and Memory Layout

For bare-metal RISC-V targets, the linker script defines where code and data land in physical memory.

```ld
/* link.ld — bare-metal RV32 linker script */
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY {
    /* Flash: 512 KB starting at 0x20000000 (XIP) */
    FLASH (rx)  : ORIGIN = 0x20000000, LENGTH = 512K
    /* SRAM: 128 KB starting at 0x80000000 */
    SRAM  (rwx) : ORIGIN = 0x80000000, LENGTH = 128K
}

SECTIONS {
    /* Interrupt vector table must be at the start of Flash */
    .vectors : {
        KEEP(*(.vectors))
    } > FLASH

    /* Read-only code and constants */
    .text : {
        *(.text .text.*)
        *(.rodata .rodata.*)
    } > FLASH

    /* Initialised data: stored in Flash, copied to SRAM at startup */
    .data : {
        _data_start = .;
        *(.data .data.*)
        _data_end = .;
    } > SRAM AT > FLASH       /* VMA in SRAM, LMA in Flash */

    _data_load = LOADADDR(.data);   /* Flash source address for startup copy */

    /* Zero-initialised data: only in SRAM; startup code clears it */
    .bss : {
        _bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        _bss_end = .;
    } > SRAM

    /* Stack: grows downward from top of SRAM */
    _stack_top = ORIGIN(SRAM) + LENGTH(SRAM);
}
```

### Compiler Intrinsics for RISC-V Extensions

The compiler exposes hardware instructions as C functions (intrinsics), avoiding inline assembly for common operations:

```c
#include <riscv_vector.h>   /* RISC-V Vector intrinsics (RVV) */

/* Example: vectorised integer addition using RVV intrinsics */
void vec_add(int32_t *dst, const int32_t *a, const int32_t *b, size_t n) {
    for (size_t vl; n > 0; n -= vl, a += vl, b += vl, dst += vl) {
        vl = __riscv_vsetvl_e32m1(n);   /* set vector length for e32, lmul=1 */
        vint32m1_t va  = __riscv_vle32_v_i32m1(a, vl);   /* vector load */
        vint32m1_t vb  = __riscv_vle32_v_i32m1(b, vl);   /* vector load */
        vint32m1_t vr  = __riscv_vadd_vv_i32m1(va, vb, vl); /* vector add */
        __riscv_vse32_v_i32m1(dst, vr, vl);               /* vector store */
    }
}
```

```c
/* Zbb bit-manipulation intrinsics (available in GCC 12+ / Clang 14+) */
#include <stdint.h>

uint32_t leading_zeros(uint32_t x) {
    return __builtin_clz(x);   /* maps to CLZ (Zbb) if -march includes _zbb */
}

uint32_t count_bits(uint32_t x) {
    return __builtin_popcount(x); /* maps to CPOP (Zbb) */
}
```

---

## Tier 1 — Fundamentals

### Question F1
**What is the difference between a native compiler and a cross-compiler? Why is a cross-compiler necessary for RISC-V embedded development?**

**Answer:**

A **native compiler** runs on the same architecture and operating system as the code it produces. A host `gcc` on an x86-64 Linux workstation is a native compiler for x86-64 Linux programs.

A **cross-compiler** runs on a host machine but produces binaries for a different target architecture or operating system. `riscv32-unknown-elf-gcc` runs on x86-64 but generates RISC-V machine code.

Cross-compilation is necessary for embedded RISC-V work for three reasons:

1. **Resource constraints:** The target hardware (MCU, FPGA prototype, evaluation board) typically lacks storage, RAM, and processing power to host a full compiler toolchain. A Cortex-M class RISC-V MCU with 128 KB Flash cannot self-host GCC.

2. **No operating system:** Bare-metal targets have no OS services (file system, dynamic linker, process management) that a native compiler and its tools depend on.

3. **Development workflow:** Building on a fast x86-64 workstation with full IDE, version control, and CI infrastructure is far more productive than building natively on embedded hardware even if the hardware were capable.

**Common mistake:** Confusing the compiler's host triplet with its target triplet. `riscv64-unknown-linux-gnu-gcc` always runs on the host (e.g., x86-64), even though its name starts with `riscv64`.

---

### Question F2
**What do the flags `-march=rv32imac` and `-mabi=ilp32` each control? What happens if you compile one file with `-mabi=ilp32` and another with `-mabi=ilp32d` and then link them?**

**Answer:**

`-march=rv32imac` controls which **instructions** the compiler is allowed to emit:
- `rv32` — 32-bit base integer ISA
- `i` — the base RV32I instruction set (always required)
- `m` — integer multiply and divide instructions (MUL, MULH, DIV, REM)
- `a` — atomic instructions (LR.W, SC.W, AMOADD.W, etc.)
- `c` — compressed 16-bit instructions (C extension)

`-mabi=ilp32` controls the **calling convention**: how function arguments, return values, and stack frames are laid out. `ilp32` means:
- `int` is 32 bits, `long` is 32 bits, `pointer` is 32 bits
- Floating-point arguments are passed in integer registers (soft-float)

Mixing `-mabi=ilp32` and `-mabi=ilp32d` produces an **incompatible object file error** at link time:

```
ld: error: /tmp/foo.o: cannot link object files with incompatible ABIs
```

This happens because `ilp32d` passes `float` and `double` arguments in floating-point registers `fa0`–`fa7`, while `ilp32` passes them in integer registers `a0`–`a7`. A caller compiled with one ABI and a callee compiled with the other will look in different registers for the same argument, causing silent data corruption or a crash.

**The linker catches this** because GCC embeds the ABI in the ELF file's `.riscv.attributes` section. The linker reads these attributes and refuses to link object files with mismatched ABIs.

---

### Question F3
**Explain what a linker script does and why bare-metal RISC-V programs typically require a custom one.**

**Answer:**

A linker script directs `ld` (the linker) to:
1. **Map input sections** (`.text`, `.data`, `.bss` from object files) to **output sections** in the final ELF.
2. **Assign Virtual Memory Addresses (VMAs)** — the addresses the code uses at runtime.
3. **Assign Load Memory Addresses (LMAs)** — the addresses where the data is stored in Flash (which may differ from VMA for initialised data).
4. **Define symbols** (e.g., `_stack_top`, `_bss_start`) that startup code uses to initialise memory.
5. **Set the entry point** — the symbol the processor jumps to on reset.

Bare-metal targets require a custom linker script because:

- **No OS loader:** On Linux, the OS loader places segments at the ELF-specified virtual addresses. Bare-metal has no loader; the binary is programmed directly to Flash at a fixed physical address that must be specified in the linker script.
- **Flash vs. SRAM layout:** Code executes from Flash (XIP), but initialised global variables must reside in SRAM (write access required). The linker script specifies that `.data` has VMA in SRAM but LMA in Flash, enabling the startup code to copy it at boot.
- **Memory region sizes vary:** Each target board has specific Flash base addresses, sizes, and SRAM locations. These differ between MCUs and cannot be inferred generically.
- **Vector table placement:** The interrupt vector table must appear at a specific address (usually the start of Flash). The linker script uses `KEEP(*(.vectors))` to prevent dead-code elimination from discarding it.

---

### Question F4
**What is `libgcc` and why is it linked with almost every RISC-V bare-metal program?**

**Answer:**

`libgcc` is GCC's low-level runtime support library. It provides software implementations of operations that the target hardware cannot perform directly or that the selected ISA subset omits.

For a RISC-V RV32I target without the M extension (`-march=rv32i`):

```
Operation in C          libgcc function called       Reason
-----------------------  --------------------------  --------------------------------
int a = b * c;           __mulsi3                    RV32I has no MUL instruction
int a = b / c;           __divsi3                    RV32I has no DIV instruction
long long a = b * c;     __muldi3                    64-bit multiply on 32-bit core
float a = b + c;         __addsf3                    Soft-float: no F extension
double a = b + c;        __adddf3                    Soft-float: no D extension
float a = (float)i;      __floatsisf                 Integer to float conversion
```

Even with `-march=rv32imac` (which includes M for multiply/divide), `libgcc` is still needed for:
- 64-bit arithmetic operations on a 32-bit core
- Soft-float operations if `-mabi=ilp32` (no hardware FP even if F extension present in hardware but not in march)
- Stack unwinding support for C++ exceptions
- `__clzsi2` (count leading zeros) on cores without Zbb

`libgcc` is linked by default. You can suppress it with `-nostdlib`, but then any operation requiring a `libgcc` helper will produce an undefined symbol error at link time.

---

## Tier 2 — Intermediate

### Question I1
**Describe the complete set of compiler flags needed to build a bare-metal RISC-V binary for a target with the following specification: RV32IMC core, no FPU, 256 KB Flash at 0x08000000, 64 KB SRAM at 0x20000000, newlib C library. Explain each flag's purpose.**

**Answer:**

```makefile
CC      = riscv32-unknown-elf-gcc
CFLAGS  = -march=rv32imc          \  # Emit RV32 + M + C instructions only
          -mabi=ilp32             \  # Soft-float calling convention (no FPU)
          -mcmodel=medlow         \  # PC-relative addressing for code below 2 GB
          -O2                     \  # Optimise; bare-metal code needs small size
          -Wall -Wextra           \  # Enable warnings
          -ffunction-sections     \  # Place each function in its own .text section
          -fdata-sections         \  # Place each variable in its own .data section
          -ffreestanding          \  # Don't assume hosted environment (no main as entry)
          -nostdlib               \  # Don't auto-link system libraries
          -specs=nano.specs       \  # Link newlib-nano (size-optimised C library)
          -T link.ld              \  # Use custom linker script for this board
          -Wl,--gc-sections          # Linker: garbage-collect unused sections
```

**Explanation of each flag:**

- `-march=rv32imc`: Without `a` (atomics), the compiler will not emit `lr.w`/`sc.w`. Without `f`/`d`, no FP instructions. Important to match hardware exactly.
- `-mabi=ilp32`: Matches the absence of FPU. Floats go through integer registers and `libgcc` soft-float routines.
- `-mcmodel=medlow`: Restricts program to the low 2 GB address range. All symbols addressable with a single `lui`+`addi` pair (32-bit absolute or PC-relative). Required for correct code generation on MCU targets.
- `-ffunction-sections` / `-fdata-sections` combined with `-Wl,--gc-sections`: Removes unreferenced functions and data from the final binary. Critical for embedded: linking the full newlib can add hundreds of KB; GC removes what is not called.
- `-ffreestanding`: Tells GCC not to assume the standard hosted environment. Without this, GCC may silently convert `printf` and similar calls to built-in equivalents with incorrect assumptions about memory layout.
- `-nostdlib`: Prevents auto-linking of `libc`, `libm`, and `crt0.o`. You supply your own startup code (`crt0.S`) and explicitly specify which libraries to include.
- `-specs=nano.specs`: Links newlib-nano, a size-optimised C library variant with reduced printf (no `%f` by default, saving ~20 KB).

---

### Question I2
**What is Link-Time Optimisation (LTO) and what RISC-V-specific considerations apply when enabling it?**

**Answer:**

**Link-Time Optimisation (LTO)** defers the final code-generation step from individual compilation units to the link step, where the entire program is visible as a single compilation unit.

**How it works:**

```
Without LTO:
  a.c --[compile]--> a.o (machine code, opaque to linker)
  b.c --[compile]--> b.o (machine code, opaque to linker)
             [link] --> final binary
             (cross-module optimisations impossible)

With LTO (GCC -flto):
  a.c --[compile]--> a.o (contains GIMPLE IR + machine code)
  b.c --[compile]--> b.o (contains GIMPLE IR + machine code)
             [lto-link] --> reads IR, performs whole-program analysis,
                            then generates machine code for all modules together
                            --> final binary
```

**Benefits for RISC-V embedded targets:**
- Inlining across compilation unit boundaries eliminates call overhead for small hot functions
- Interprocedural constant propagation can eliminate dead branches
- Whole-program dead-code elimination beyond what `-Wl,--gc-sections` alone achieves
- Smaller binary: often 5-15% reduction in code size for MCU firmware

**RISC-V-specific considerations:**

1. **Consistent `-march`/`-mabi` across all TUs:** LTO combines all IR before code generation. If one compilation unit was compiled with different flags, the merged code generation may silently use wrong instruction sets or ABIs. Use `CFLAGS` consistently in your Makefile.

2. **Linker script symbol references:** Symbols defined in the linker script (e.g., `_bss_start`) are not in any object file. LTO's dead-code elimination must not remove the code that references these. Use `__attribute__((used))` or `KEEP()` in the linker script to protect critical sections.

3. **Inline assembly:** GCC cannot LTO-optimise or move code around inline assembly blocks. Assembly barriers (`asm volatile("" ::: "memory")`) are respected, but complex interactions between C and inline assembly may behave unexpectedly across module boundaries.

4. **Fat LTO objects:** GCC's `-ffat-lto-objects` embeds both machine code and LTO IR in the same `.o` file. Useful when the same object file is used with and without LTO (e.g., in a library).

5. **Build time:** LTO requires re-processing all IR at link time. For large firmware projects this adds significant build time. Use `make -j` parallelism; LLVM ThinLTO is faster than GCC LTO for large projects.

---

### Question I3
**Explain how GCC and Clang handle RISC-V custom (vendor-specific) instruction extensions. What are the two main approaches for exposing a new instruction to C code?**

**Answer:**

Custom RISC-V instruction extensions occupy the reserved custom-0 through custom-3 opcode spaces. The toolchain must be told about these instructions to emit them in code.

**Approach 1: GCC/Clang inline assembly with `.insn` directive**

The `.insn` directive allows encoding arbitrary instruction words in inline assembly without requiring the assembler to know the mnemonic:

```c
/* Custom accumulate instruction: custom-0 opcode, R-type encoding
   Instruction: CACC rd, rs1, rs2
   Encoding:    funct7=0x00, rs2, rs1, funct3=0x0, rd, opcode=0x0B (custom-0)
*/

static inline uint32_t custom_acc(uint32_t a, uint32_t b) {
    uint32_t result;
    asm volatile (
        /* .insn type, opcode, funct3, rd, rs1, rs2  */
        ".insn r 0x0B, 0x0, 0x00, %0, %1, %2"
        : "=r"(result)    /* output: rd mapped to result */
        : "r"(a), "r"(b)  /* inputs: rs1=a, rs2=b */
        :                 /* no clobbers */
    );
    return result;
}
```

Advantages: requires no toolchain modification, works immediately.
Disadvantage: opaque to the compiler — it cannot optimise around the instruction, spill/reload registers intelligently, or schedule it.

**Approach 2: Toolchain extension with custom -march string and new mnemonic**

For GCC, this requires modifying `gcc/config/riscv/riscv.md` (the machine description) and `gcc/config/riscv/riscv-builtins.cc`. For LLVM, modifications go into `llvm/lib/Target/RISCV/` using TableGen `.td` files.

```
LLVM approach (outline):
1. Add ISA extension flag in RISCVFeatures.td:
     def FeatureExtXCustom : SubtargetFeature<"xcustom", "HasExtXCustom",
         "true", "RISC-V custom accelerator extension">;

2. Define instruction in RISCVInstrInfoXCustom.td:
     def CACC : RVInstR<0b0000000, 0b000, OPC_CUSTOM_0,
         (outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2),
         "cacc", "$rd, $rs1, $rs2">;

3. Map to an intrinsic in RISCVInstrInfoXCustom.td:
     def int_riscv_cacc : Intrinsic<[llvm_i32_ty],
         [llvm_i32_ty, llvm_i32_ty], [IntrNoMem]>;

4. C code using the intrinsic:
     #include <riscv_xcustom.h>          /* generated header */
     uint32_t r = __builtin_riscv_cacc(a, b);
```

The compiler-aware approach allows:
- Register allocation: the compiler picks `rd`, `rs1`, `rs2` optimally
- Instruction scheduling: the compiler can reorder for pipeline efficiency
- Constant folding: if `a` and `b` are compile-time constants, the compiler can fold the result
- `-march=rv32imac_xcustom` enables the extension

**Practical guidance:** Use inline assembly with `.insn` for prototyping or when only a few instructions need exposing. Use the TableGen/machine description approach for production toolchains where performance and compiler integration matter.

---

### Question I4
**What does `objdump -d` show, and how do you use it to verify that a compiled binary uses the correct ISA extensions and nothing more?**

**Answer:**

`objdump -d` disassembles the `.text` section (and other executable sections) of an ELF binary, showing the instruction address, encoding, and mnemonic for every instruction:

```bash
riscv32-unknown-elf-objdump -d -M no-aliases firmware.elf | head -40

firmware.elf:     file format elf32-littleriscv

Disassembly of section .text:

20000000 <_start>:
20000000:   00001137    lui     sp,0x1            # load stack pointer
20000004:   00010113    addi    sp,sp,0           # (lui + addi = LI pseudo-instr)
20000008:   02c000ef    jal     ra,20000034 <main>

2000000c <_loop>:
2000000c:   0000006f    jal     zero,2000000c <_loop>  # infinite loop

20000010 <uart_putc>:
20000010:   fe010113    addi    sp,sp,-32
20000014:   00812e23    sw      s0,28(sp)
...
```

**Verifying ISA compliance:**

```bash
# Check for any multiply instructions (should be absent if -march=rv32ic, no M)
riscv32-unknown-elf-objdump -d firmware.elf | grep -E '\bmul\b|\bdiv\b|\brem\b'

# Check for floating-point instructions (should be absent if -mabi=ilp32)
riscv32-unknown-elf-objdump -d firmware.elf | grep -E '^[0-9a-f]+:\s+[0-9a-f]+\s+f[a-z]'

# Check ELF attributes for recorded -march and -mabi:
riscv32-unknown-elf-readelf -A firmware.elf

# Output example:
Attribute Section: riscv
File Attributes
  Tag_RISCV_stack_align: 16-bytes
  Tag_RISCV_arch: "rv32i2p1_m2p0_a2p1_c2p0"
```

The `Tag_RISCV_arch` attribute is written by the assembler into the `.riscv.attributes` ELF section and records the exact ISA that was compiled. The linker reads this attribute and refuses to link incompatible object files.

**Key `objdump` flags for RISC-V:**
- `-d`: disassemble executable sections
- `-D`: disassemble all sections (useful for inspecting `.data` initialised with code addresses)
- `-M no-aliases`: show true instruction encodings rather than pseudo-instructions (e.g., `jalr zero, ra, 0` instead of `ret`)
- `-S`: interleave C source with assembly (requires `-g` at compile time)
- `--visualize-jumps`: draw ASCII art jump arrows (useful for loop analysis)

---

## Tier 3 — Advanced

### Question A1
**A firmware build produces a binary that is 20% larger when compiled with `-O2` compared to `-Os`. Describe a systematic methodology for diagnosing and resolving the size increase, naming the specific tools and techniques involved.**

**Answer:**

**Step 1: Establish a baseline section map**

```bash
# Show all sections with sizes
riscv32-unknown-elf-size -A firmware_O2.elf
riscv32-unknown-elf-size -A firmware_Os.elf

# Detailed symbol sizes for comparison
riscv32-unknown-elf-nm --print-size --size-sort --radix=d firmware_O2.elf > syms_O2.txt
riscv32-unknown-elf-nm --print-size --size-sort --radix=d firmware_Os.elf > syms_Os.txt
diff syms_Os.txt syms_O2.txt | grep "^[<>]" | sort -k2 -rn | head -20
```

This immediately identifies which functions grew. Common culprits: hot loops that `-O2` unrolls, functions where `-O2` inlines callees aggressively, or vectorised code expansions.

**Step 2: Inspect loop unrolling decisions**

```bash
# Generate annotated assembly with optimisation remarks
riscv32-unknown-elf-gcc -march=rv32imac -mabi=ilp32 -O2 \
    -fopt-info-missed=missed.txt \
    -fopt-info-optimized=optimized.txt \
    bigfile.c -c -o bigfile.o

grep "loop unrolled" optimized.txt
```

If a loop is being unrolled 8x under `-O2` but not under `-Os`, add `#pragma GCC optimize("no-unroll-loops")` to the problematic function, or use `__attribute__((optimize("Os")))` on specific functions to apply `-Os` selectively:

```c
__attribute__((optimize("Os")))
void large_table_init(void) {
    /* This function is called once at boot; optimise for size, not speed */
}
```

**Step 3: Control inlining**

`-O2` inlines more aggressively than `-Os`. Check which functions were inlined:

```bash
# -Winline warns when a function marked inline was not inlined (inverse useful here)
# Use -fno-inline to disable inlining and measure the effect:
riscv32-unknown-elf-gcc -march=rv32imac -mabi=ilp32 -O2 -fno-inline -c bigfile.c
```

To prevent inlining of specific large functions called from multiple sites:

```c
__attribute__((noinline))
static void slow_helper(void) { /* ... */ }
```

**Step 4: Profile-Guided size optimization**

For critical code that must stay fast but with smaller footprint:

```bash
# Build with -Os everywhere except the measured hot path
# Use __attribute__((hot)) on hot functions (GCC places them early in text)
# Use __attribute__((cold)) on error-handling paths (GCC may size-optimise them)
```

**Step 5: Verify with bloaty (Google's binary size analysis tool)**

```bash
bloaty firmware_O2.elf -- firmware_Os.elf
# Shows diff of symbols, sections, compilation units contributing to size delta
```

**Resolution strategy summary:**

| Cause                      | Fix                                                      |
|----------------------------|----------------------------------------------------------|
| Loop unrolling             | `-fno-unroll-loops` globally or `__attribute__((optimize("Os")))` |
| Aggressive inlining        | `__attribute__((noinline))` on large inlined callees     |
| Vectorisation              | `-fno-tree-vectorize` or disable vector for that function|
| String constant duplication| `-fmerge-constants` (default at `-O2`; verify enabled)   |
| Dead code not removed      | Ensure `-ffunction-sections -fdata-sections -Wl,--gc-sections` |

---

### Question A2
**Explain the difference between the ELF VMA (Virtual Memory Address) and LMA (Load Memory Address) for the `.data` section in a bare-metal RISC-V system. Walk through the exact mechanism by which initialised global variables become correct at runtime.**

**Answer:**

In a bare-metal system, the `.data` section presents a fundamental challenge: it contains **initialised global variables** that must be **read-write at runtime** (so they live in SRAM) but must be **stored persistently** when the device is powered off (so they must be in Flash).

This is resolved through the VMA / LMA split:

```
VMA (Virtual Memory Address):  The address the code uses to access the variable at runtime.
                                 Must be in SRAM for read-write access.
LMA (Load Memory Address):      The address where the data is physically stored in the
                                 flash binary. Must be in Flash to survive power-off.
```

**Linker script mechanism:**

```ld
/* .data section: VMA in SRAM, LMA in Flash */
.data : {
    _data_vma_start = .;          /* SRAM start address */
    *(.data .data.*)
    _data_vma_end = .;            /* SRAM end address */
} > SRAM AT > FLASH               /* VMA in SRAM, LMA in Flash */

_data_lma_start = LOADADDR(.data); /* Flash address where data is stored */
```

**The linker produces:**
- An ELF binary where `.data` bytes are placed at the LMA (Flash address) in the binary file
- Symbols `_data_vma_start`, `_data_vma_end`, `_data_lma_start` that the startup code uses

**Startup code (crt0.S) copies Flash to SRAM before main():**

```asm
/* crt0.S — RISC-V bare-metal startup */
    .section .text
    .globl _start
_start:
    /* 1. Set the stack pointer */
    la   sp, _stack_top

    /* 2. Copy initialised data from Flash (LMA) to SRAM (VMA) */
    la   a0, _data_lma_start    /* source: Flash address */
    la   a1, _data_vma_start    /* destination: SRAM address */
    la   a2, _data_vma_end      /* end of destination */
copy_data:
    bge  a1, a2, copy_data_done
    lw   t0, 0(a0)
    sw   t0, 0(a1)
    addi a0, a0, 4
    addi a1, a1, 4
    j    copy_data
copy_data_done:

    /* 3. Zero the BSS section */
    la   a0, _bss_start
    la   a1, _bss_end
zero_bss:
    bge  a0, a1, zero_bss_done
    sw   zero, 0(a0)
    addi a0, a0, 4
    j    zero_bss
zero_bss_done:

    /* 4. Call main */
    call main
    /* 5. Hang if main returns */
hang: j hang
```

**Concrete example:**

```c
int global_counter = 42;  /* initialised global — goes in .data */
int uninit_val;           /* uninitialised global — goes in .bss (zero-initialised) */
const int table[] = {1, 2, 3}; /* const — goes in .rodata in Flash; no copy needed */
```

```
After linking:
  Flash at 0x08010000: [00 00 00 2A]  /* 42 = 0x2A, little-endian — LMA of global_counter */
  SRAM  at 0x20000000: [?? ?? ?? ??]  /* undefined before startup code runs */

After _start copies .data:
  SRAM  at 0x20000000: [00 00 00 2A]  /* global_counter == 42, correct */

After _start zeros .bss:
  SRAM  at 0x20000004: [00 00 00 00]  /* uninit_val == 0, per C standard */
```

**Common mistake:** Forgetting to implement the BSS zeroing. The C standard requires that global variables without explicit initialisation be zero-initialised. An OS loader does this. Bare-metal startup code must do it explicitly. Omitting BSS zeroing causes subtle, non-deterministic bugs where uninitialised globals contain whatever was in SRAM from a previous power cycle.
