# Problem 01: Assembly Debugging with GDB and Spike

## Difficulty
Intermediate (covers Tier 1 + Tier 2 material)

## Prerequisites
- RISC-V calling convention (ABI): a0-a7 (function args/return), ra (return address), sp (stack pointer)
- RISC-V RV32I instructions: LW, SW, ADDI, ADD, JAL, JALR, BLT, BEQ
- Basic GDB commands: `break`, `stepi`, `info registers`, `x/Ni $pc`
- Spike interactive debugger (`spike -d pk`)

---

## Problem Statement

The following RISC-V assembly function is intended to compute the **sum of an integer array**. It has been compiled for RV32I (`-march=rv32i -mabi=ilp32`) and loaded into Spike.

```asm
# Function: int array_sum(int *arr, int n)
# Arguments: a0 = pointer to array, a1 = n (array length)
# Returns:   a0 = sum of arr[0..n-1]

array_sum:
    addi  sp, sp, -16
    sw    ra, 12(sp)
    sw    s0, 8(sp)
    sw    s1, 4(sp)

    mv    s0, a0          # s0 = arr pointer
    mv    s1, a1          # s1 = n

    li    a0, 0           # a0 = sum = 0
    li    t0, 0           # t0 = i = 0

.loop:
    bge   t0, s1, .done   # if i >= n, exit loop

    lw    t1, 0(s0)       # t1 = arr[i]    <-- BUG: should be arr[i], but s0 is not updated
    add   a0, a0, t1      # sum += arr[i]

    addi  t0, t0, 1       # i++
    j     .loop           # (pseudo: jal x0, .loop)

.done:
    lw    ra, 12(sp)
    lw    s0, 8(sp)
    lw    s1, 4(sp)
    addi  sp, sp, 16
    ret
```

**Test call:**
```c
int arr[] = {1, 2, 3, 4, 5};
int result = array_sum(arr, 5);  // Expected: 15; Actual: wrong value
```

**Tasks:**

1. Identify the bug in the assembly code without running it (static analysis).
2. Describe the exact sequence of Spike debug commands and GDB commands you would use to observe the bug dynamically.
3. Write the corrected assembly code.
4. The corrected code is then compiled and tested with `n=0` (empty array). Predict the result and identify any additional edge case bugs.

---

## Solution

### Part 1: Static Analysis — Identify the Bug

**Reading the loop body carefully:**

```asm
.loop:
    bge   t0, s1, .done   # if i >= n: exit
    lw    t1, 0(s0)       # t1 = mem[s0 + 0]   <- s0 is FIXED, never updated
    add   a0, a0, t1      # sum += t1
    addi  t0, t0, 1       # i++
    j     .loop
```

`s0` is set to `arr` (the base pointer) before the loop and is never modified inside the loop. Every iteration executes `lw t1, 0(s0)` — loading from the same fixed address (`arr[0]`) on every iteration.

**The bug:** The array pointer `s0` is not incremented by 4 bytes (one `int` = 4 bytes on RV32I ILP32 ABI) after each element access. The loop computes `n * arr[0]` instead of `sum(arr[0..n-1])`.

**Expected behaviour:**
- `arr = {1, 2, 3, 4, 5}`, `n = 5`
- Correct: `1 + 2 + 3 + 4 + 5 = 15`
- Buggy: `5 * arr[0] = 5 * 1 = 5`

### Part 2: Dynamic Debugging with Spike and GDB

**Method A: Spike interactive debugger**

```bash
# Compile with debug info and no optimisation
riscv32-unknown-elf-gcc -march=rv32i -mabi=ilp32 -g -O0 -o sum_test sum_test.c

# Launch Spike in debug mode
spike -d --isa=rv32i pk sum_test
```

```
# Spike debugger session
(spike) until pc 0 0x<array_sum_addr>   # run until function entry
                                          # find address from: nm sum_test | grep array_sum

(spike) reg 0 a0    # verify a0 = &arr (should be a valid stack or .data address)
(spike) reg 0 a1    # verify a1 = 5
(spike) reg 0 s0    # s0 = ? (not yet set at function entry)

(spike) run 6       # execute 6 instructions (prologue: addi+4x sw + mv + mv + li + li)

(spike) reg 0 s0    # verify s0 = &arr[0]
(spike) reg 0 s1    # verify s1 = 5
(spike) reg 0 a0    # should be 0 (sum initialised)
(spike) reg 0 t0    # should be 0 (i initialised)

# Now single-step through the first loop iteration:
(spike) step        # execute: bge t0, s1, .done  (t0=0, s1=5: not taken)
(spike) step        # execute: lw t1, 0(s0)
(spike) reg 0 t1    # should be 1 (arr[0])
(spike) step        # execute: add a0, a0, t1
(spike) reg 0 a0    # should be 1

# Step through second iteration:
(spike) step        # addi t0, t0, 1
(spike) step        # j .loop
(spike) step        # bge (t0=1, s1=5: not taken)
(spike) step        # lw t1, 0(s0)   <- BUG: s0 unchanged, still points to arr[0]
(spike) reg 0 t1    # should be 2 (arr[1]), but is 1 (arr[0] again) -- BUG CONFIRMED
(spike) reg 0 s0    # compare to original value -- identical: BUG CONFIRMED
```

**Method B: GDB with QEMU**

```bash
# Launch QEMU user mode with GDB stub
qemu-riscv32 -g 1234 ./sum_test &

# Connect GDB
riscv32-unknown-elf-gdb -ex "target remote :1234" sum_test
(gdb) break array_sum
(gdb) continue

# At the function entry:
(gdb) info registers a0 a1
# a0 = 0x<arr_addr>, a1 = 0x5

(gdb) break *array_sum+24    # break at the lw instruction (offset depends on prologue size)
                               # find exact offset: disassemble array_sum
(gdb) disassemble array_sum  # view the compiled assembly

(gdb) continue               # run to the lw instruction (first iteration)
(gdb) stepi                  # execute lw t1, 0(s0)
(gdb) info registers t1 s0   # t1 = arr[0] = 1; s0 = &arr[0]

(gdb) continue               # run to second iteration's lw
(gdb) stepi
(gdb) info registers t1 s0   # t1 should be 2 but is 1; s0 is unchanged -- BUG
```

**Key observation from debugging:**

```
Iteration 1: s0 = 0x1000, t1 = mem[0x1000] = 1   (correct: arr[0])
Iteration 2: s0 = 0x1000, t1 = mem[0x1000] = 1   (wrong: should be arr[1] = 2)
Iteration 3: s0 = 0x1000, t1 = mem[0x1000] = 1   (wrong: should be arr[2] = 3)
...
Final sum: a0 = 5 (not 15)
```

The dynamic trace confirms the static analysis: `s0` is never advanced.

### Part 3: Corrected Assembly Code

```asm
# Function: int array_sum(int *arr, int n)
# Arguments: a0 = pointer to array, a1 = n (array length)
# Returns:   a0 = sum of arr[0..n-1]

array_sum:
    addi  sp, sp, -16
    sw    ra, 12(sp)
    sw    s0, 8(sp)
    sw    s1, 4(sp)

    mv    s0, a0          # s0 = arr pointer (current element pointer)
    mv    s1, a1          # s1 = n

    li    a0, 0           # a0 = sum = 0
    li    t0, 0           # t0 = i = 0

.loop:
    bge   t0, s1, .done   # if i >= n, exit loop

    lw    t1, 0(s0)       # t1 = *s0 = arr[i]
    add   a0, a0, t1      # sum += arr[i]

    addi  s0, s0, 4       # s0 += 4  (advance pointer by one int = 4 bytes) <-- FIX
    addi  t0, t0, 1       # i++
    j     .loop

.done:
    lw    ra, 12(sp)
    lw    s0, 8(sp)
    lw    s1, 4(sp)
    addi  sp, sp, 16
    ret
```

**Alternative fix using offset addressing (avoids modifying s0):**

```asm
    # Replace:
    lw    t1, 0(s0)
    # With computed offset: offset = i * 4; t1 = arr[offset]
    slli  t2, t0, 2       # t2 = i * 4 (left shift by 2 = multiply by 4)
    add   t2, t2, s0      # t2 = &arr[i]
    lw    t1, 0(t2)       # t1 = arr[i]
```

This alternative does not require `addi s0, s0, 4` in the loop, preserving `s0` as the original base pointer. Both approaches are correct; the first (pointer increment) is slightly more efficient (one instruction vs. two).

### Part 4: Edge Case Analysis — n=0

**With the corrected code:**

```
array_sum(arr, 0):
  s0 = arr, s1 = 0
  a0 = 0 (sum initialised)
  t0 = 0 (i initialised)
  .loop:
    bge t0=0, s1=0, .done  -> 0 >= 0 is TRUE -> branch taken to .done
  .done:
    return a0 = 0
```

**Result: a0 = 0. Correct.** The `bge` check correctly handles the empty array without any loop iterations. No bug for `n=0`.

**Additional edge cases to check:**

```
n=1:
  arr = {42}
  Loop: i=0: t1 = arr[0] = 42; sum = 42; s0 += 4; i = 1
        bge 1, 1, .done -> taken
  Return: 42. Correct.

n=negative (n=-1, a1=0xFFFFFFFF as signed):
  bge t0=0, s1=-1, .done
  In RISC-V: BGE is a signed comparison.
  0 >= -1 is TRUE -> branch taken immediately.
  Return: a0 = 0.
  This may or may not be the desired behaviour (treating negative n as empty).
  If n should never be negative, this is acceptable; document it.

n=very large (potential integer overflow in sum):
  If the array contains large values and n is large, the 32-bit sum can overflow.
  The code does not check for this; the result silently wraps modulo 2^32.
  This is a semantic issue, not an assembly bug per se, but worth noting in a review.
```

**The `j .loop` pseudo-instruction:**

Note that `j .loop` is a pseudo-instruction that assembles to `jal x0, .loop`. It writes the return address to `x0` (discarding it) and jumps to `.loop`. If the loop offset exceeds 20 bits (2 MB), this instruction cannot reach the target — but for typical function sizes, this is never a problem.

---

## Discussion

### Why this bug is hard to find without stepping

The function computes a numerically plausible result: for the input `{1,2,3,4,5}`, the buggy result is `5`, which could be confused with a correct result if the first element happened to be 3 and `n=5` (giving `5*3=15` which matches!). This illustrates why systematic testing requires arrays with distinct, non-trivial values where `n * arr[0] != sum(arr)`.

### Systematic test design for array_sum

```c
// Test cases that distinguish the bug from correct behavior:
{1, 2, 3, 4, 5},  n=5  -> expected 15, buggy produces 5   (5 * 1 = 5 != 15)
{3, 1, 4, 1, 5},  n=5  -> expected 14, buggy produces 15  (5 * 3 = 15 != 14)
{0, 0, 0, 0, 1},  n=5  -> expected 1,  buggy produces 0   (5 * 0 = 0 != 1)
{},               n=0  -> expected 0,  both produce 0     (cannot distinguish)
{7},              n=1  -> expected 7,  both produce 7     (cannot distinguish)
```

A good test suite includes cases where `n * arr[0] != sum(arr)` to discriminate the correct from the buggy implementation.

### Using objdump for static inspection before running

```bash
riscv32-unknown-elf-objdump -d sum_test | grep -A 30 '<array_sum>'
```

This shows the disassembled code. Scanning the loop body for pointer advancement (`addi sX, sX, 4`) or indexed addressing (`slli tX, tX, 2; add tX, tX, base`) is faster than running the debugger for simple bugs like this one. The absence of any `addi s0, s0, 4` instruction in the loop body immediately flags the pointer-not-incremented bug.
