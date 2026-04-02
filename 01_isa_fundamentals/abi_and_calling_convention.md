# ABI and Calling Convention

## Prerequisites
- RV32I base instructions — see `rv32i_base_instructions.md`
- Stack pointer concepts: push, pop, frame
- Function call mechanics: call, return, arguments, return values

---

## Concept Reference

### What the ABI Defines

The Application Binary Interface (ABI) is the contract between separately compiled units of code. For RISC-V, the primary ABI specification is the **RISC-V psABI** (Processor-Specific ABI Supplement). It defines:

1. **Register names and roles** — which architectural registers (x0-x31) serve which purpose
2. **Calling convention** — how arguments are passed and results returned
3. **Caller-saved vs. callee-saved** — which registers a function may destroy vs. must preserve
4. **Stack frame layout** — how local variables, saved registers, and the frame pointer are arranged

The integer ABI names (used in assembly and compiler output) are the standard across all RISC-V platforms. The most common ABI variants are:
- `ilp32` — 32-bit integers, longs, and pointers (used with RV32I/RV32G)
- `lp64` — 64-bit longs and pointers (used with RV64I/RV64G)

---

### Register Names and Roles (RV32I Integer Registers)

```
Reg    ABI Name   Role                                  Saver
-----  ---------  ------------------------------------  ----------
x0     zero       Hardwired zero (reads always 0,       N/A
                  writes discarded)
x1     ra         Return address                        Caller
x2     sp         Stack pointer                         Callee
x3     gp         Global pointer (SDA base)             N/A
x4     tp         Thread pointer                        N/A
x5     t0         Temporary / alternate link register   Caller
x6     t1         Temporary                             Caller
x7     t2         Temporary                             Caller
x8     s0 / fp    Saved register 0 / Frame pointer      Callee
x9     s1         Saved register 1                      Callee
x10    a0         Argument 0 / Return value 0           Caller
x11    a1         Argument 1 / Return value 1           Caller
x12    a2         Argument 2                            Caller
x13    a3         Argument 3                            Caller
x14    a4         Argument 4                            Caller
x15    a5         Argument 5                            Caller
x16    a6         Argument 6                            Caller
x17    a7         Argument 7                            Caller
x18    s2         Saved register 2                      Callee
x19    s3         Saved register 3                      Callee
x20    s4         Saved register 4                      Callee
x21    s5         Saved register 5                      Callee
x22    s6         Saved register 6                      Callee
x23    s7         Saved register 7                      Callee
x24    s8         Saved register 8                      Callee
x25    s9         Saved register 9                      Callee
x26    s10        Saved register 10                     Callee
x27    s11        Saved register 11                     Callee
x28    t3         Temporary                             Caller
x29    t4         Temporary                             Caller
x30    t5         Temporary                             Caller
x31    t6         Temporary                             Caller
```

Memory aid for groupings:
- **zero** (x0): 1 register
- **ra** (x1): 1 return address
- **sp, gp, tp** (x2-x4): 3 pointer registers
- **t0-t2** (x5-x7): 3 temporaries (first group)
- **s0-s1** (x8-x9): 2 saved (first group); s0 doubles as fp
- **a0-a7** (x10-x17): 8 argument/return registers
- **s2-s11** (x18-x27): 10 saved (second group)
- **t3-t6** (x28-x31): 4 temporaries (second group)

---

### Calling Convention: Argument Passing

**Integer and pointer arguments:**

The first 8 integer or pointer arguments are passed in `a0`-`a7` (x10-x17). Arguments beyond 8 are placed on the stack, pushed in right-to-left order (the 9th argument is at the lowest stack address, i.e., `0(sp)` in the callee's view after the callee prologue).

```
int foo(int a, int b, int c);
  a => a0
  b => a1
  c => a2

void bar(int a, int b, int c, int d, int e, int f, int g, int h, int i, int j);
  a-h => a0-a7
  i   => 0(sp)   (lowest address of stack arguments)
  j   => 4(sp)
```

**Structs and aggregates:**

- Structs that fit entirely in 2 registers (<=8 bytes on RV32, <=16 bytes on RV64) are passed in a pair of argument registers.
- Larger structs are passed **by reference**: the caller allocates space on its stack and passes a pointer in an argument register.
- Structs with mixed floating-point and integer fields have additional rules when the F or D extension is present (hardware floating-point registers a0-a7 / fa0-fa7).

**Stack argument alignment:**

The stack must be 16-byte aligned at the point of a function call (i.e., at the moment `JAL`/`JALR` executes). Arguments placed on the stack are 4-byte aligned for 32-bit types.

---

### Calling Convention: Return Values

```
Return type                        Register(s) used
---------------------------------  ----------------
int, char, short, pointer (32-bit) a0
int64_t (on RV32)                  a0 (low 32 bits), a1 (high 32 bits)
struct <= 8 bytes (RV32)           a0, a1
struct > 8 bytes (RV32)            caller provides pointer in a0; callee writes to it
void                               (no register used)
```

If a function returns a 64-bit value on RV32I, `a0` holds bits [31:0] and `a1` holds bits [63:32]. This is the **little-endian register pair** convention.

---

### Caller-Saved vs. Callee-Saved Registers

This distinction answers: "When I return from a function, which registers are guaranteed to hold the same value they had before the call?"

**Caller-saved (also called "volatile" or "call-clobbered"):**
The callee may freely overwrite these. The caller must save them before the call if it needs their values after the call.
```
ra, t0-t6, a0-a7
(x1, x5-x7, x10-x17, x28-x31)
```

**Callee-saved (also called "non-volatile" or "call-preserved"):**
If the callee uses these registers, it must save their original values on entry and restore them before returning.
```
sp, s0-s11
(x2, x8-x9, x18-x27)
```

**Neither (fixed-purpose, never saved by convention):**
```
zero (x0): always 0, no need to save
gp   (x3): global pointer, set once at program startup, never changed
tp   (x4): thread pointer, managed by threading runtime, not saved across calls
```

**Memory model:**
```
Before call:                    During callee:               After return:
  a0 = argument 0               a0 = can be overwritten       a0 = return value
  s0 = my_data (important)      s0 = must be preserved        s0 = my_data (intact)
  t0 = scratch                  t0 = can be overwritten       t0 = unknown
```

---

### Stack Frame Layout

The RISC-V ABI uses a **full descending stack**: the stack pointer `sp` points to the last used (valid) word, and the stack grows towards lower addresses (each push decrements sp).

A typical stack frame for a non-leaf function (one that calls other functions):

```
High address (caller's territory)
  ...
  +---------------------------+
  | incoming stack arguments  |  (if any, managed by caller)
  +---------------------------+  <-- sp before callee's prologue
  | ra (return address)       |  saved by callee (caller-saved, but callee saves before overwriting)
  +---------------------------+
  | s0/fp (frame pointer)     |  saved by callee if fp is used
  +---------------------------+
  | s1, s2, ...               |  any other callee-saved registers used
  +---------------------------+
  | local variables           |  allocated by callee
  +---------------------------+
  | (padding to 16-byte align)|  if needed
  +---------------------------+  <-- sp after prologue (sp points here during function body)
Low address
```

Stack pointer alignment rule: `sp` must be 16-byte aligned **at every public function entry point** and **when making any CALL (`JAL`/`JALR`)**. Within a function, `sp` can temporarily be non-aligned as long as it is re-aligned before any call.

**Typical prologue and epilogue:**

```assembly
# Function: int add_and_store(int a, int b, int *dst)
# a  => a0 (x10)
# b  => a1 (x11)
# dst => a2 (x12)
# This function calls another function, so ra must be saved.

add_and_store:
    # Prologue: allocate frame, save callee-saved and ra
    addi sp, sp, -16      # allocate 16 bytes (16-byte aligned frame)
    sw   ra, 12(sp)       # save return address at sp+12
    sw   s0,  8(sp)       # save s0 (if used) at sp+8

    # Function body
    add  t0, a0, a1       # t0 = a + b  (t0 is caller-saved, no need to save)
    sw   t0, 0(a2)        # *dst = t0

    # Call another function (example)
    # mv a0, t0            # argument to callee
    # call some_func       # ra will be overwritten by JAL — but we saved it above

    mv   a0, t0           # return value = a + b

    # Epilogue: restore and return
    lw   s0,  8(sp)       # restore s0
    lw   ra, 12(sp)       # restore return address
    addi sp, sp, 16       # deallocate frame
    ret                   # JALR x0, ra, 0 — return to caller
```

---

### Frame Pointer (fp / s0)

The frame pointer `fp` (also named `s0` / `x8`) is optional in RISC-V. When used, it points to a fixed location within the current frame, typically the top of the frame (the address of the first saved register). This provides a stable reference for debuggers even when `sp` is modified during the function body (e.g., by variable-length stack allocations with `alloca()`).

```
High address
  +---------------------------+  <-- fp points here (if fp is used)
  | ra                        |
  +---------------------------+
  | saved fp                  |
  +---------------------------+
  | local variables           |
  +---------------------------+  <-- sp
Low address

With fp: local var at fp - 8, fp - 12, etc. (stable addresses)
Without fp: local var at sp + 0, sp + 4, etc. (valid only if sp doesn't change)
```

GCC omits the frame pointer by default with `-fomit-frame-pointer` (which is on by default at `-O1` and higher), saving the overhead of saving, restoring, and updating `fp`. Most embedded code does not use `fp`.

---

### The Global Pointer (gp) and Relaxation

`gp` (x3) points into the middle of the `.sdata` / `.sbss` section — the "short data" area for small global variables. This allows ADDI, LW, and SW instructions to access global variables as a single instruction (using `gp`-relative addressing) rather than requiring an AUIPC + ADDI + LW sequence.

```assembly
# Without gp relaxation (3 instructions):
auipc t0, %hi(global_var)
addi  t0, t0, %lo(global_var)
lw    a0, 0(t0)

# With gp relaxation (1 instruction, if global_var is within ±2 KB of gp):
lw    a0, %gprel(global_var)(gp)
```

The linker performs `gp` relaxation: it replaces AUIPC + ADDI pairs that access `.sdata`-region symbols with single GP-relative instructions. This is a significant code-size saving for embedded binaries with many small global variables.

`gp` is set once by the C runtime startup code (`_start`) before `main()` is called, and never changed thereafter.

---

### The Thread Pointer (tp)

`tp` (x4) points to the thread-local storage (TLS) block for the currently running thread. Each thread has its own TLS block containing a copy of variables declared `__thread` (or `thread_local` in C++11). The threading runtime (pthreads, RTOS task context) sets `tp` when switching between threads.

For bare-metal single-threaded embedded code, `tp` is typically unused and treated as a free register (though the ABI reserves it).

---

## Tier 1 — Fundamentals

### Question F1
**Name all 32 RISC-V integer registers by both their architectural name (x0-x31) and their ABI name. Without looking at the table, state which group each ABI name belongs to (temporary, saved, argument, or special).**

**Answer:**

The categories from memory:
```
Special purpose (4 registers):
  x0  = zero   hardwired zero
  x1  = ra     return address
  x2  = sp     stack pointer
  x3  = gp     global pointer
  x4  = tp     thread pointer

Temporaries — caller-saved, freely clobberable (7 registers):
  x5  = t0     also the alternate link register
  x6  = t1
  x7  = t2
  x28 = t3
  x29 = t4
  x30 = t5
  x31 = t6

Saved — callee-saved, must be preserved across calls (12 registers):
  x8  = s0/fp  also used as frame pointer
  x9  = s1
  x18 = s2
  x19 = s3
  x20 = s4
  x21 = s5
  x22 = s6
  x23 = s7
  x24 = s8
  x25 = s9
  x26 = s10
  x27 = s11

Arguments / return values — caller-saved (8 registers):
  x10 = a0     first argument and first return value
  x11 = a1     second argument and second return value
  x12 = a2
  x13 = a3
  x14 = a4
  x15 = a5
  x16 = a6
  x17 = a7
```

**Memory aid:** The ABI names cluster by number: `t0-t2` are x5-x7, `s0-s1` are x8-x9, `a0-a7` are x10-x17, `s2-s11` are x18-x27, `t3-t6` are x28-x31. The jump in the temporary numbering (t0-t2, then t3-t6) exists because x8-x9 and x18-x27 were assigned to saved registers, breaking the temporary group into two blocks.

---

### Question F2
**What is the difference between caller-saved and callee-saved registers? If a function `foo()` calls `bar()`, who is responsible for saving `t3` and who is responsible for saving `s3`?**

**Answer:**

**Caller-saved (volatile):** The register may be overwritten by any called function. If the caller needs the value after the call, the caller must save it before calling and restore it after.

**Callee-saved (non-volatile):** Any function that uses this register must save the original value on entry and restore it before returning. The caller's value is guaranteed to be intact after the call.

For `foo()` calling `bar()`:

```
t3 (x28) — caller-saved:
  foo() is responsible for saving t3 (if foo needs t3's value after the call).
  bar() may freely overwrite t3 without saving it.
  If foo() does not need t3 after the call, no save is necessary.

s3 (x19) — callee-saved:
  bar() is responsible for saving s3 before using it and restoring it before returning.
  foo() can rely on s3 being unchanged when bar() returns.
  foo() does not need to save s3 around the call.
```

**Common mistake:** Thinking "callee-saved" means the callee saves the register so the caller can use it. It means the callee saves it so that **the caller's value** is preserved. The callee saves s3 to protect the caller's value in s3.

---

### Question F3
**Write the RISC-V assembly prologue and epilogue for a leaf function `int square(int x)` that returns `x * x`. Assume the M extension is available for MUL. Does this function need to save ra? Why or why not?**

**Answer:**

A **leaf function** is one that does not call any other function. It does not need to save `ra` because `ra` will not be overwritten — no `JAL`/`JALR` instruction is executed that would overwrite `ra`.

```assembly
# int square(int x)
# Argument: x in a0
# Return:   result in a0

square:
    # No prologue needed: leaf function, no callee-saved registers used
    # a0 contains x; a0 is also the return register, so we compute in place

    mul  a0, a0, a0    # a0 = x * x
    ret                # return (JALR x0, ra, 0)
```

This function has:
- No frame allocation (no `addi sp, sp, -N`)
- No register saves
- No `sw ra, ...` because ra is never modified
- Minimal code: two instructions

If the M extension is not available:
```assembly
square:
    # Software multiplication via __mulsi3 (libgcc) — now ra MUST be saved
    addi sp, sp, -16
    sw   ra, 12(sp)
    sw   a0,  8(sp)    # save x before the call overwrites a0 (argument register)
    # Note: a0 is also used as argument to __mulsi3, so we must pass x in both a0 and a1
    mv   a1, a0        # a1 = x (second argument)
                       # a0 = x (first argument, already there)
    call __mulsi3      # a0 = a0 * a1 = x * x; this call overwrites ra
    lw   ra, 12(sp)
    addi sp, sp, 16
    ret
```

Without M extension, the leaf property is lost and `ra` must be saved.

---

### Question F4
**How are the first three arguments and the return value passed for this C function? `int compute(int a, char b, int *ptr);`**

**Answer:**

```
Argument  C type   ABI register   Notes
--------  -------  -------------  -------------------------------------------
a         int      a0 (x10)       Full 32-bit value in a0
b         char     a1 (x11)       Promoted to int for passing (C integer promotion);
                                  upper 24 bits are zero-filled (unsigned char)
                                  or sign-filled (signed char) — but 32 bits in a1
ptr       int*     a2 (x12)       32-bit pointer on RV32; full address in a2

Return:   int      a0 (x10)       Callee writes result here before returning
```

On RV32I (ilp32 ABI), all types narrower than 32 bits are widened to 32 bits when passed in registers. The callee receives `b` as a full 32-bit value with the upper bits either zero-extended (`unsigned char`) or sign-extended (`signed char`).

The pointer `ptr` is passed as a plain 32-bit integer value (the address) in a2. RISC-V has no separate pointer-passing mechanism.

---

## Tier 2 — Intermediate

### Question I1
**Trace the stack pointer for the following C code on RV32I. Show each `addi sp, sp, -N` and `addi sp, sp, +N` that a compiler would emit, and draw the stack layout during the execution of `inner()`.**

```c
int inner(int x) { return x + 1; }
int outer(int a, int b) {
    int sum = inner(a) + inner(b);
    return sum;
}
```

**Answer:**

```assembly
# inner(int x) — leaf function, no frame needed
inner:
    addi a0, a0, 1     # return x + 1
    ret

# outer(int a, int b)
# Needs to call inner() twice, so ra must be saved.
# Also needs to save the result of first call (a0 is overwritten by second call).
# Uses callee-saved s0 and s1 to hold intermediate results safely.
outer:
    # Prologue
    addi sp, sp, -16   # allocate 16-byte frame
    sw   ra, 12(sp)    # save return address
    sw   s0,  8(sp)    # save s0 (will use to store first inner() result)
    sw   s1,  4(sp)    # save s1 (will use to hold b across the call)

    # Body
    mv   s1, a1        # s1 = b (save b before call clobbers a1)
    # a0 = a (already in a0, ready for first call)
    call inner         # a0 = inner(a) = a + 1; ra is overwritten
    mv   s0, a0        # s0 = inner(a) result (safe from second call)

    mv   a0, s1        # a0 = b (argument for second call)
    call inner         # a0 = inner(b) = b + 1
    add  a0, s0, a0    # a0 = inner(a) + inner(b)

    # Epilogue
    lw   s1,  4(sp)    # restore s1
    lw   s0,  8(sp)    # restore s0
    lw   ra, 12(sp)    # restore return address
    addi sp, sp, 16    # deallocate frame
    ret
```

Stack layout during execution of `inner()` (called from `outer()`):

```
High address
+------------------------+  <- original sp (caller's territory above outer's frame)
| (caller's frame)       |
+------------------------+  <- outer's sp on entry
| ra (return to caller)  |  sp + 12
| s0 (caller's s0)       |  sp + 8
| s1 (caller's s1)       |  sp + 4
| (unused padding)       |  sp + 0
+------------------------+  <- sp during outer's body (sp - 16 from outer's entry)

When outer calls inner:
  - inner is a leaf function; no further adjustment to sp
  - sp stays at (outer's sp after prologue) during inner's execution
  - ra is overwritten by JAL (outer's ra has already been saved to the stack)
```

The 16-byte frame allocation even though only 12 bytes are needed (ra, s0, s1 = 12 bytes) is to satisfy the 16-byte stack alignment requirement: `sp` must be 16-byte aligned at every call site.

---

### Question I2
**A function has the following signature: `void process(int a0, int a1, int a2, int a3, int a4, int a5, int a6, int a7, int overflow1, int overflow2)`. On RV32I, how are `overflow1` and `overflow2` passed? Where on the stack are they? Who allocates and deallocates the space for them?**

**Answer:**

`a0`-`a7` (the first 8 arguments) fill the 8 argument registers `a0`-`a7` (x10-x17).

`overflow1` (9th argument) and `overflow2` (10th argument) are placed on the **caller's stack** before the call instruction. The caller allocates space and writes the arguments; the callee reads them using positive offsets from its own `sp`.

**Caller's responsibility (before the call):**
```assembly
# Caller sets up stack arguments (sp must be 16-byte aligned at the JAL):
addi sp, sp, -16       # allocate space for 2 x 4-byte stack arguments (padded to 16)
sw   t0, 0(sp)         # overflow1 at sp+0 (t0 holds the value of overflow1)
sw   t1, 4(sp)         # overflow2 at sp+4 (t1 holds the value of overflow2)
# ... fill a0-a7 with arguments 1-8 ...
call process           # sp+0 = overflow1, sp+4 = overflow2 as seen by callee
addi sp, sp, 16        # caller deallocates the stack arguments after return
```

**Callee's view (inside `process`, after any prologue that adjusts sp):**
```
If process has a prologue that does: addi sp, sp, -N

Then inside process:
  overflow1 is at: sp + N + 0
  overflow2 is at: sp + N + 4

If process has no stack frame (hypothetically):
  overflow1 is at: sp + 0
  overflow2 is at: sp + 4
```

**Who deallocates:** The **caller** deallocates the stack argument space after the function returns. This is the standard **cdecl**-style convention (same as x86 cdecl). The callee does NOT adjust `sp` for incoming stack arguments.

**Memory layout immediately before the call (from the stack arguments perspective):**
```
High address
  +----------------+
  | overflow2      |  sp + 4  (second stack argument)
  +----------------+
  | overflow1      |  sp + 0  (first stack argument, lowest address)
  +----------------+  <-- sp at call site (16-byte aligned)
```

---

### Question I3
**What does the RISC-V ABI say about the `gp` register? What is "linker relaxation" and how does `gp` enable it? What must the startup code do before calling `main()`?**

**Answer:**

The `gp` register (x3, global pointer) is a dedicated pointer to the small-data area (`.sdata` section). Its value is set once at startup and never changed during normal program execution.

**Linker relaxation with `gp`:**

The linker performs a pass called **relaxation** where it replaces multi-instruction sequences with shorter equivalents. One of the most impactful relaxations is GP-relative addressing:

Without relaxation, accessing a global variable requires:
```assembly
auipc t0, %hi(my_global)     # 4 bytes: t0 = PC + upper(offset)
lw    a0, %lo(my_global)(t0) # 4 bytes: a0 = mem[t0 + lower(offset)]
# Total: 8 bytes, 2 instructions
```

With GP relaxation (if `my_global` is within ±2047 bytes of `gp`):
```assembly
lw    a0, %gprel(my_global)(gp)  # 4 bytes: a0 = mem[gp + small_offset]
# Total: 4 bytes, 1 instruction
```

The linker replaces the AUIPC + LW pair with a single GP-relative LW if the symbol is close enough to `gp`. This saves code size and one cycle of execution time per global access. For an embedded binary with dozens of global variables, this can reduce code size by several percent.

**What the startup code must do:**

Before calling `main()`, the C runtime startup (`_start` or `crt0.S`) must:

1. Initialise the stack pointer:
   ```assembly
   la sp, _stack_top      # load address of top of stack
   ```

2. Set the global pointer:
   ```assembly
   .option push
   .option norelax        # IMPORTANT: disable relaxation for this instruction
   la gp, __global_pointer$
   .option pop
   ```
   The `.option norelax` directive is required because `la gp, ...` would normally be relaxed to `addi gp, gp, ...` (using gp to compute itself) — which is nonsensical. Disabling relaxation forces the assembler to emit a full AUIPC + ADDI pair.

3. Optionally set the thread pointer `tp` if TLS is used.

4. Copy `.data` from flash to RAM (initialise mutable globals).
5. Zero `.bss` (zero-initialise uninitialised globals).
6. Call `main()`.

---

### Question I4
**Explain the function call sequence for `call foo` (a pseudo-instruction). What real instructions does the assembler/linker emit? How does the return from `foo` work?**

**Answer:**

The pseudo-instruction `call foo` is expanded differently depending on the distance to `foo`:

**Case 1: `foo` is within ±1 MB (JAL range):**
```assembly
# Assembler emits single instruction:
jal ra, foo          # ra = PC + 4; PC = foo
                     # Encoding: JAL x1, offset_to_foo
```

**Case 2: `foo` is beyond ±1 MB (out of JAL range):**
```assembly
# Assembler emits two-instruction sequence:
auipc ra, %hi(foo)   # ra = PC + upper_bits_of_offset_to_foo
jalr  ra, ra, %lo(foo) # ra = auipc_instr_addr + 4; PC = ra + lower_bits
```

The two-instruction form works because:
- AUIPC computes `PC + {upper 20 bits, 12'b0}`, covering all but the lower 12 bits
- JALR adds the remaining 12-bit signed offset and jumps, simultaneously saving the return address in `ra`

**Important detail:** The return address saved is the address of the **instruction after JALR** (not after AUIPC). The AUIPC address is an intermediate value; the link value is always `PC_of_JALR + 4`.

**Return from `foo`:**
```assembly
# Inside foo, before returning:
ret                  # Assembler pseudo: JALR x0, ra, 0
                     # PC = ra (the saved return address)
                     # x0 = discarded (link to nowhere)
```

`ret` jumps to the address in `ra`. It discards the new link by writing to `x0`. After `ret`, execution continues at the instruction immediately following the original `call foo`.

**Full sequence with stack context:**
```
Caller stack before call: [ra = caller's return address, ... other saved regs ...]

  call foo:
    JAL ra, foo       => ra = addr_after_JAL; PC = foo

  Inside foo (if non-leaf):
    sw ra, N(sp)      => save ra (now = addr_after_JAL in caller)
    ...
    lw ra, N(sp)      => restore ra
    ret               => PC = addr_after_JAL

  Execution resumes in caller at addr_after_JAL.
```

---

## Tier 3 — Advanced

### Question A1
**A RISC-V function receives a C `struct` argument by value: `void process(struct Point p)` where `struct Point { int x; int y; }`. Describe exactly how `p` is passed on RV32I ilp32 ABI. If the struct were `struct Large { int a, b, c; }` (12 bytes), how would it be passed instead?**

**Answer:**

**`struct Point { int x; int y; }` — 8 bytes, fits in two registers:**

The struct is passed in a **register pair** `a0:a1`:
```
a0 (x10) = p.x     (low-address field, matches little-endian)
a1 (x11) = p.y
```

The caller places `p.x` in `a0` and `p.y` in `a1` before the call. No stack allocation is needed for the struct itself. The callee reads `a0` for `x` and `a1` for `y`.

```assembly
# Caller code for:  process(p) where p = {x: 10, y: 20}
li   a0, 10        # p.x
li   a1, 20        # p.y
call process
```

**`struct Large { int a, b, c; }` — 12 bytes, does NOT fit in two registers:**

For the ilp32 ABI, structs larger than 2 × XLEN (2 × 4 = 8 bytes on RV32) are passed **by implicit reference**:

1. The caller allocates space for the struct on the stack.
2. The caller copies the struct into that space.
3. The caller passes a pointer to the copied struct in the next available argument register.

```assembly
# Caller code for:  process_large(big) where big = {a:1, b:2, c:3}

# Allocate space for a copy of struct Large on caller's stack
addi sp, sp, -16       # 12 bytes needed, rounded up to 16 for alignment
li   t0, 1
sw   t0, 0(sp)         # big.a at sp+0
li   t0, 2
sw   t0, 4(sp)         # big.b at sp+4
li   t0, 3
sw   t0, 8(sp)         # big.c at sp+8
mv   a0, sp            # a0 = pointer to the struct copy
call process_large     # callee receives pointer in a0
addi sp, sp, 16        # caller deallocates the copy
```

**Why the distinction at 2 × XLEN?**

Passing a struct in registers avoids a load-store round-trip. Two registers covers the most common small structs (pairs of coordinates, key-value pairs, two-field results). Three registers or more is less common and the overhead of saving additional argument registers in the callee increases, making by-reference passing more efficient.

**Practical consequence:** A function that returns `struct Point` also uses `a0:a1` for the return value. A function that returns `struct Large` passes a hidden pointer in `a0` and writes through it — the caller pre-allocates the storage and passes the address.

---

### Question A2
**What happens to the stack alignment when the compiler inserts an `alloca()` (variable-length stack allocation) inside a function? How does the compiler maintain the ±2047-byte ADDI addressing range for local variables when `sp` is no longer at a fixed offset from the frame?**

**Answer:**

`alloca(n)` (or a C99 variable-length array) allocates `n` bytes on the stack at runtime, decrementing `sp` by a runtime value. After `alloca()`, `sp` is at an unknown (to the compiler) static offset from the function's entry `sp`. This has two consequences:

**1. The frame pointer becomes mandatory.**

Without `alloca()`, the compiler can address all locals and saved registers as fixed offsets from `sp` (which is constant after the prologue). With `alloca()`, `sp` changes dynamically and the compiler cannot use it as a stable base for frame-relative addressing.

The compiler therefore:
- Sets `fp` (= `s0` / `x8`) to point to a fixed location in the frame at the top of the prologue.
- Addresses locals as `fp - offset` (stable, since fp never changes after being set).
- Addresses the dynamically allocated block through `sp` (which moved).

```assembly
alloca_example:
    addi sp, sp, -32      # static frame: saved regs, fp, etc.
    sw   ra, 28(sp)
    sw   fp, 24(sp)
    addi fp, sp, 32       # fp = old sp = top of this frame (FIXED)

    # alloca(n): n is in a0
    sub  sp, sp, a0       # sp moves downward by n (dynamic)
    andi sp, sp, -16      # re-align sp to 16 bytes
    mv   a0, sp           # return pointer to allocated block

    # ... use fp-relative addressing for all other locals ...
    # local_var at fp - 8: always stable regardless of alloca
    sw   t0, -8(fp)

    # Epilogue
    mv   sp, fp           # restore sp = fp (undoes ALL alloca allocations)
    addi sp, sp, -32      # ... wait, we saved fp at sp+24, so:
    lw   fp, -8(fp)       # restore saved fp: fp pointed 32 bytes above prologue sp,
                          # saved fp is at fp-8... (depends on frame layout)
    lw   ra, -4(fp)
    # (exact offsets depend on compiler's frame layout choice)
    ret
```

**2. The epilogue restores `sp` from `fp` rather than a fixed offset.**

`addi sp, fp, -32` (or equivalent) restores the pre-alloca stack pointer. This frees all alloca blocks at once without tracking their sizes.

**Addressing range concern:**

When fp-relative addressing is used, all locals are at `fp - offset` for small positive `offset`. The ADDI/LW/SW immediates are 12-bit signed, so the range is -2048 to +2047 bytes from fp. For functions with large frames (more than 2 KB of locals), the compiler must use temporary registers to form addresses beyond the 12-bit immediate range:

```assembly
# Accessing a local at fp - 4096 (beyond ±2047):
li   t0, -4096
add  t0, fp, t0
lw   a0, 0(t0)      # two-instruction address calculation
```

In practice, GCC and LLVM attempt to reorder the frame layout to keep frequently accessed variables close to `fp` within the 12-bit range, placing rarely accessed or large arrays at larger offsets where the two-instruction form is acceptable.
