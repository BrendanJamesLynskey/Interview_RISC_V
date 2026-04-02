# Problem 03: Function Call Stack Trace

**Topic:** Stack frame evolution through nested function calls  
**Difficulty:** Fundamentals (Parts A-B), Intermediate (Parts C-D), Advanced (Parts E-F)  
**Prerequisites:** `abi_and_calling_convention.md`, `rv32i_base_instructions.md`

---

## Problem Statement

The following C program runs on an RV32I processor using the standard `ilp32` ABI. The stack starts at `0xBFFC` (initial value of `sp`) and grows downward. The M extension is available.

```c
int multiply(int a, int b) {
    return a * b;
}

int dot_product(int x0, int y0, int x1, int y1) {
    int p = multiply(x0, x1);
    int q = multiply(y0, y1);
    return p + q;
}

int main(void) {
    int result = dot_product(3, 4, 5, 6);
    return result;
}
```

---

### Part A (Fundamentals)

Write the RISC-V assembly for `multiply()`. Justify why it needs no prologue or epilogue.

---

### Part B (Fundamentals)

At the moment `multiply(x0, x1)` is called for the first time (from inside `dot_product`), list the values in the following registers and memory locations:

- `sp`, `ra`, `a0`, `a1`
- The stack contents from `sp` to `0xBFFB`

---

### Part C (Intermediate)

Write the complete annotated assembly for `dot_product()`, including all stack saves and restores, register choices, and a cycle-by-cycle trace of the key register values.

---

### Part D (Intermediate)

Write the complete annotated assembly for `main()`. Then draw the full stack layout at the deepest point of the call chain (when `multiply` is executing for the first time), showing every word on the stack and its address.

---

### Part E (Advanced)

A second version of `dot_product` is written that avoids saving `y0`/`y1` by re-computing them from a pointer:

```c
int dot_product_v2(int *vec_a, int *vec_b) {
    int p = multiply(vec_a[0], vec_b[0]);
    int q = multiply(vec_a[1], vec_b[1]);
    return p + q;
}
```

Write the RISC-V assembly for this version. Identify which registers must be saved and explain the saving strategy.

---

### Part F (Advanced)

A bug is introduced: the programmer forgets to save `ra` in `dot_product`. Show the exact assembly that results, trace the execution of `dot_product(3, 4, 5, 6)`, and explain precisely where and why the program crashes.

---

## Solutions

### Part A Solution

```assembly
# int multiply(int a, int b)
# Arguments: a in a0 (x10), b in a1 (x11)
# Return:    result in a0
# Uses M extension: MUL rd, rs1, rs2

multiply:
    mul  a0, a0, a1    # a0 = a * b
    ret                # JALR x0, ra, 0
```

**Why no prologue or epilogue:**

`multiply` is a **leaf function** — it does not call any other function. Therefore:

1. **`ra` (return address) is never overwritten.** Only a `JAL` or `JALR` that writes to `ra` would overwrite it. `MUL` does not touch `ra`. Since `ra` is not overwritten, it does not need to be saved.

2. **No callee-saved registers are used.** The function uses `a0` and `a1`, which are both caller-saved (argument/return registers). There are no `s0`-`s11` registers used, so nothing needs preserving.

3. **No local storage needed.** The computation fits entirely in registers.

Consequently, no stack space is allocated and `sp` is not modified. The function is 2 instructions (8 bytes).

---

### Part B Solution

**Context:** We are inside `dot_product`, which was called from `main`. The call to `multiply(x0, x1)` is being made.

First, trace the setup to reach this point.

`main` calls `dot_product(3, 4, 5, 6)`:
```
a0 = 3  (x0)
a1 = 4  (y0)
a2 = 5  (x1)
a3 = 6  (y1)
```

`dot_product` runs its prologue (see Part C for full details):
```
sp decremented to save ra, s0, s1, s2 (16 bytes: sp = 0xBFFC - 16 = 0xBFEC)
ra = return address to main
s1 = a1 = 4  (y0, saved before first call)
s2 = a3 = 6  (y1, saved before first call)
a0 = a0 = 3  (x0, already in a0 for first call)
a1 = a2 = 5  (x1, moved to a1 for first call)
```

At the point `JAL ra, multiply` executes (first call):
- `a0 = 3` (argument: x0)
- `a1 = 5` (argument: x1)
- `ra` is about to be set to `PC_of_jal + 4` (the instruction after JAL in dot_product)
- `sp = 0xBFEC`

**Register values when `multiply(x0, x1)` starts executing:**

```
Register   Value      Explanation
---------  ---------  --------------------------------------------------
sp         0xBFEC     dot_product's frame is allocated; multiply is leaf
ra         0x????     Just set by JAL to (address of JAL instruction + 4)
                      i.e., the return address pointing back into dot_product
a0         3          x0 argument (first call: multiply(3, 5))

Wait: multiply(x0, x1) = multiply(3, 5):
  x0 = a0 = 3
  x1 = a2 = 5, but we moved a2 into a1 for the call
a0         3          first argument to multiply
a1         5          second argument to multiply
```

**Stack contents from `sp` (0xBFEC) to 0xBFFB:**

```
Address    Value      Contents
---------  ---------  --------------------------------------------------
0xBFEB     (n/a)      below sp, unused
0xBFEC     s2 saved   6  (y1 = a3 from main's call, saved by dot_product)
0xBFF0     s1 saved   4  (y0 = a1 from main's call, saved by dot_product)
0xBFF4     s0 saved   (previous s0 value from main/startup — irrelevant)
0xBFF8     ra saved   return address back to main (dot_product's saved ra)
0xBFFC     (empty)    above dot_product's frame: main's frame territory
```

Note: `multiply` does not touch `sp`, so the stack during `multiply`'s execution is identical to the stack at the point of the call. The 4 saved values are from `dot_product`'s prologue.

---

### Part C Solution

```assembly
# int dot_product(int x0, int y0, int x1, int y1)
# Arguments on entry:
#   a0 = x0 (3)
#   a1 = y0 (4)
#   a2 = x1 (5)
#   a3 = y1 (6)
# Return: a0 = p + q

dot_product:
    #---------------------------------------------------------
    # PROLOGUE
    # We call multiply() twice, so ra will be overwritten.
    # We need to save: ra, and any values needed across calls.
    #
    # What we need after the first call:
    #   y0 (a1): needed as argument to second call => save in s1
    #   y1 (a3): needed as argument to second call => save in s2
    #   result of first call (a0 after first multiply) => save in s0
    # x0 (a0) and x1 (a2): consumed by first call, not needed after
    #---------------------------------------------------------

    addi sp, sp, -16       # allocate 16-byte frame
    sw   ra, 12(sp)        # save ra (will be overwritten by JAL)
    sw   s0,  8(sp)        # save s0 (callee-saved: must preserve)
    sw   s1,  4(sp)        # save s1 (callee-saved: must preserve)
    sw   s2,  0(sp)        # save s2 (callee-saved: must preserve)

    #---------------------------------------------------------
    # SAVE ARGUMENTS NEEDED ACROSS FIRST CALL
    # a0 (x0) and a2 (x1) are used in the first call.
    # a1 (y0) and a3 (y1) would be clobbered — save in s1, s2.
    #---------------------------------------------------------
    mv   s1, a1            # s1 = y0 (save before it is clobbered)
    mv   s2, a3            # s2 = y1 (save before it is clobbered)

    #---------------------------------------------------------
    # FIRST CALL: p = multiply(x0, x1) = multiply(a0, a2)
    # a0 is already x0. Move x1 (a2) into a1.
    #---------------------------------------------------------
    mv   a1, a2            # a1 = x1 (second argument for multiply)
    call multiply          # a0 = x0 * x1; ra overwritten; a1 clobbered
    mv   s0, a0            # s0 = p = multiply(x0, x1) result

    #---------------------------------------------------------
    # SECOND CALL: q = multiply(y0, y1)
    # Restore y0 and y1 from saved registers.
    #---------------------------------------------------------
    mv   a0, s1            # a0 = y0
    mv   a1, s2            # a1 = y1
    call multiply          # a0 = y0 * y1; ra overwritten again

    #---------------------------------------------------------
    # COMPUTE RETURN VALUE: p + q
    # a0 = q (result of second call)
    # s0 = p
    #---------------------------------------------------------
    add  a0, s0, a0        # a0 = p + q

    #---------------------------------------------------------
    # EPILOGUE
    #---------------------------------------------------------
    lw   s2,  0(sp)        # restore s2
    lw   s1,  4(sp)        # restore s1
    lw   s0,  8(sp)        # restore s0
    lw   ra, 12(sp)        # restore ra
    addi sp, sp, 16        # deallocate frame
    ret                    # return to caller
```

**Register trace through key points:**

```
Point                    a0    a1    s0    s1    s2    ra          sp
-----------------------  ----  ----  ----  ----  ----  ----------  ------
dot_product entry        3     4     ?     ?     ?     ret_main    0xBFFC
after prologue           3     4     saved saved saved ret_main    0xBFEC
after mv s1,a1; mv s2,a3 3     4     saved 4     6     ret_main    0xBFEC
after mv a1,a2           3     5     saved 4     6     ret_main    0xBFEC
--- call multiply ---     3     5     saved 4     6     -> dp+4     0xBFEC
multiply executes        15    5     saved 4     6     dp+4        0xBFEC
--- return from multiply  15    5     saved 4     6     dp+4        0xBFEC
after mv s0,a0           15    5     15    4     6     dp+4        0xBFEC
after mv a0,s1; mv a1,s2 4     6     15    4     6     dp+4        0xBFEC
--- call multiply (2nd) ---
multiply executes        24    6     15    4     6     dp+?        0xBFEC
--- return from multiply  24    6     15    4     6     dp+?        0xBFEC
after add a0,s0,a0       39    6     15    4     6     dp+?        0xBFEC
after epilogue           39    6     s0    s1    s2    ret_main    0xBFFC
```

Final return value: `a0 = 39 = 3*5 + 4*6 = 15 + 24`.

---

### Part D Solution

```assembly
# int main(void)
# dot_product(3, 4, 5, 6) -> result in a0
# return result (a0 is already the return value)

main:
    #---------------------------------------------------------
    # PROLOGUE
    # main calls dot_product, so ra must be saved.
    # No callee-saved registers needed (result just goes in a0 for return).
    # Minimum frame: 16 bytes (for 16-byte alignment; only ra needed = 4 bytes,
    # padded to 16).
    #---------------------------------------------------------
    addi sp, sp, -16       # allocate 16-byte frame (sp: 0xBFFC -> 0xBFEC)
    sw   ra, 12(sp)        # save ra at 0xBFEC + 12 = 0xBFF8

    #---------------------------------------------------------
    # SET UP ARGUMENTS: dot_product(3, 4, 5, 6)
    #---------------------------------------------------------
    li   a0, 3             # a0 = x0 = 3
    li   a1, 4             # a1 = y0 = 4
    li   a2, 5             # a2 = x1 = 5
    li   a3, 6             # a3 = y1 = 6
    call dot_product       # a0 = dot_product(3,4,5,6) = 39

    #---------------------------------------------------------
    # EPILOGUE AND RETURN
    # a0 already contains the return value (39).
    #---------------------------------------------------------
    lw   ra, 12(sp)        # restore ra
    addi sp, sp, 16        # deallocate frame (sp: 0xBFEC -> 0xBFFC)
    ret                    # return to OS/startup
```

**Full stack layout at the deepest point (during first `multiply` execution):**

At this moment, three frames are on the stack: `main`'s, `dot_product`'s, and `multiply` (which is leaf and adds nothing to the stack).

```
Address  Value              Owner           Contents
-------  -----------------  --------------  ----------------------------------
0xBFFC   (above main frame) OS / startup    (previous sp, not our concern)

------ main's stack frame (16 bytes, 0xBFEC to 0xBFFB) ------
0xBFF8   <ra: ret to OS>    main            saved return address (back to startup)
0xBFF4   (padding)          main            unused padding (to maintain 16-byte frame)
0xBFF0   (padding)          main            unused padding
0xBFEC   (padding)          main            unused padding

------ dot_product's stack frame (16 bytes, 0xBFDC to 0xBFEB) ------

Wait: main's frame is 16 bytes, starting at sp=0xBFEC, occupying 0xBFEC to 0xBFFB.
  ra saved at 0xBFF8 (sp+12 = 0xBFEC+12 = 0xBFF8)
  
Then dot_product allocates another 16 bytes:
  sp goes from 0xBFEC to 0xBFDC (0xBFEC - 16 = 0xBFDC)

dot_product saves:
  ra at sp+12 = 0xBFDC+12 = 0xBFE8
  s0 at sp+ 8 = 0xBFDC+ 8 = 0xBFE4
  s1 at sp+ 4 = 0xBFDC+ 4 = 0xBFE0
  s2 at sp+ 0 = 0xBFDC+ 0 = 0xBFDC

Full stack picture:

Address  Value                    Owner          Description
-------  -----------------------  -------------  --------------------------------
0xBFFC   (os frame / above main)  startup        initial sp value (not allocated)
0xBFFB                                            |
0xBFFA                                            | main's frame (16 bytes)
0xBFF9                                            |
0xBFF8   <return addr to OS>      main           saved ra
0xBFF7                                            |
0xBFF6                                            |
0xBFF5                                            |
0xBFF4   0x00000000 (padding)     main           unused
0xBFF3                                            |
0xBFF2                                            |
0xBFF1                                            |
0xBFF0   0x00000000 (padding)     main           unused
0xBFEF                                            |
0xBFEE                                            |
0xBFED                                            |
0xBFEC   0x00000000 (padding)     main           unused (bottom of main's frame)
                                                  <- sp after main's prologue
0xBFEB                                            |
0xBFEA                                            |
0xBFE9                                            |
0xBFE8   <return addr into main>  dot_product    saved ra (points into main after call)
0xBFE7                                            |
0xBFE6                                            |
0xBFE5                                            |
0xBFE4   <prev value of s0>       dot_product    saved s0 (value from before dot_product ran)
0xBFE3                                            |
0xBFE2                                            |
0xBFE1                                            |
0xBFE0   4  (= y0)                dot_product    saved s1 (= 4, y0 argument)
0xBFDF                                            |
0xBFDE                                            |
0xBFDD                                            |
0xBFDC   6  (= y1)                dot_product    saved s2 (= 6, y1 argument)
         ^                                        <- sp = 0xBFDC during multiply
```

`multiply` is a leaf function: it does not modify `sp`. The stack pointer remains at `0xBFDC` throughout `multiply`'s execution.

**Summary of sp values:**
```
Before main prologue:             sp = 0xBFFC
After main prologue:              sp = 0xBFEC
After dot_product prologue:       sp = 0xBFDC  (deepest point)
After dot_product epilogue:       sp = 0xBFEC
After main epilogue:              sp = 0xBFFC
```

---

### Part E Solution

```assembly
# int dot_product_v2(int *vec_a, int *vec_b)
# Arguments: vec_a in a0 (x10), vec_b in a1 (x11)
# Access: vec_a[0] at 0(a0), vec_a[1] at 4(a0), etc.

dot_product_v2:
    #---------------------------------------------------------
    # ANALYSIS: What do we need across the two calls?
    # First call:  multiply(vec_a[0], vec_b[0])
    # Second call: multiply(vec_a[1], vec_b[1])
    #
    # After first call, a0 holds result p.
    # We need: vec_a (to load vec_a[1]) and vec_b (to load vec_b[1]).
    # We also need p after the first call.
    #
    # Registers to save: ra, s0 (for p), s1 (for vec_a), s2 (for vec_b)
    # That's 4 registers = 16 bytes. 16-byte frame (already aligned).
    #---------------------------------------------------------

    addi sp, sp, -16
    sw   ra, 12(sp)
    sw   s0,  8(sp)
    sw   s1,  4(sp)
    sw   s2,  0(sp)

    # Save the pointers (they will be clobbered by the call: a0 and a1
    # are argument/return registers, and the calling convention does not
    # guarantee they survive across a call).
    mv   s1, a0            # s1 = vec_a pointer
    mv   s2, a1            # s2 = vec_b pointer

    #---------------------------------------------------------
    # FIRST CALL: p = multiply(vec_a[0], vec_b[0])
    #---------------------------------------------------------
    lw   a0, 0(s1)         # a0 = vec_a[0]
    lw   a1, 0(s2)         # a1 = vec_b[0]
    call multiply          # a0 = vec_a[0] * vec_b[0]
    mv   s0, a0            # s0 = p

    #---------------------------------------------------------
    # SECOND CALL: q = multiply(vec_a[1], vec_b[1])
    #---------------------------------------------------------
    lw   a0, 4(s1)         # a0 = vec_a[1]  (offset 4 bytes = next int)
    lw   a1, 4(s2)         # a1 = vec_b[1]
    call multiply          # a0 = vec_a[1] * vec_b[1]

    #---------------------------------------------------------
    # RESULT: p + q
    #---------------------------------------------------------
    add  a0, s0, a0        # a0 = p + q

    lw   s2,  0(sp)
    lw   s1,  4(sp)
    lw   s0,  8(sp)
    lw   ra, 12(sp)
    addi sp, sp, 16
    ret
```

**Saving strategy analysis:**

| Register | Why saved | Saved as |
|----------|-----------|----------|
| `ra` | `call multiply` overwrites `ra` with the return address into this function | Saved to stack at `sp+12` |
| `s0` | Holds `p` (first multiply result) across the second call; must survive the call | Callee-saved register, saved to stack at `sp+8` |
| `s1` | Holds `vec_a` pointer; both `a0` and `a1` are caller-saved and will be destroyed | Callee-saved register, saved to stack at `sp+4` |
| `s2` | Holds `vec_b` pointer, same reason as `s1` | Callee-saved register, saved to stack at `sp+0` |

The original `dot_product` received its arguments by value and had to save 4 scalar arguments. This version receives pointers and reloads the values from memory when needed, but still must preserve the pointers themselves across the calls — requiring the same number of saves (ra + 3 registers = 16 bytes).

**Could we save fewer registers?**

Yes, if we loaded all four values before the first call and stored them in saved registers:
```assembly
# Alternative: pre-load all 4 values
lw   s0, 0(a0)   # s0 = vec_a[0]
lw   s1, 4(a0)   # s1 = vec_a[1]
lw   s2, 0(a1)   # s2 = vec_b[0]
lw   s3, 4(a1)   # s3 = vec_b[1]  (requires saving s3 too => larger frame)
# Then calls don't need pointer registers
```
But this requires 4 saved registers plus `ra` = 5 saves = 20 bytes padded to 32 bytes — larger than the 16-byte frame of the original approach. The pointer-preservation approach is more efficient.

---

### Part F Solution

**Bug:** `dot_product` omits saving `ra`.

```assembly
# dot_product WITH THE BUG (ra not saved):

dot_product_buggy:
    addi sp, sp, -12       # allocate 12 bytes (only s0, s1, s2 saved)
    # BUG: ra is NOT saved here
    sw   s0,  8(sp)
    sw   s1,  4(sp)
    sw   s2,  0(sp)

    mv   s1, a1
    mv   s2, a3
    mv   a1, a2
    call multiply          # BUG: ra is OVERWRITTEN here with return address
                           # into dot_product_buggy (just past this JAL)
                           # The original ra (pointing back to main) is LOST

    mv   s0, a0
    mv   a0, s1
    mv   a1, s2
    call multiply          # ra is OVERWRITTEN again with another return address

    add  a0, s0, a0

    lw   s2,  0(sp)
    lw   s1,  4(sp)
    lw   s0,  8(sp)
    addi sp, sp, 12
    ret                    # BUG: jumps to whatever is in ra NOW
                           # ra = address inside dot_product_buggy (from last JAL)
```

**Execution trace showing the crash:**

```
Step 1: main calls dot_product_buggy(3, 4, 5, 6)
  - JAL ra, dot_product_buggy
  - ra = address of instruction AFTER this JAL in main  (call this addr_main_ret)

Step 2: dot_product_buggy prologue
  - sp -= 12
  - s0, s1, s2 saved to stack
  - ra is NOT saved: ra still = addr_main_ret

Step 3: first call to multiply
  - Before: ra = addr_main_ret
  - JAL ra, multiply
  - ra is OVERWRITTEN = addr_after_first_jal (the mv s0, a0 instruction)

  After this JAL, ra = addr_after_first_jal. addr_main_ret is LOST.

Step 4: multiply executes and returns
  - ret in multiply: JALR x0, ra, 0
  - ra = addr_after_first_jal (points back into dot_product_buggy, correct)
  - Execution continues at addr_after_first_jal in dot_product_buggy

Step 5: second call to multiply
  - Before: ra = addr_after_first_jal
  - JAL ra, multiply
  - ra is OVERWRITTEN = addr_after_second_jal (the add a0, s0, a0 instruction)

Step 6: multiply executes and returns
  - ret: jumps to addr_after_second_jal
  - Execution continues at add instruction in dot_product_buggy

Step 7: dot_product_buggy epilogue and ret
  - Epilogue restores s0, s1, s2 and adjusts sp
  - ret: JALR x0, ra, 0
  - ra = addr_after_second_jal (the address INSIDE dot_product_buggy)
  - THIS IS WRONG. ra should be addr_main_ret.

Step 8: CRASH / undefined behaviour
  - PC = addr_after_second_jal (which is the 'add' instruction inside dot_product_buggy)
  - Execution re-enters the middle of dot_product_buggy with corrupted state:
    - a0 has the return value (39)
    - s0, s1, s2 were just restored but are now being used again
    - sp has been incremented back (+12) so the stack is corrupt
    - The second 'add' executes: a0 = s0_restored + a0 = ?_old + 39 = garbage
    - Hits the second 'epilogue' which subtracts a different stack offset
    - ret again: jumps to addr_after_second_jal again
    - This is an INFINITE LOOP or leads to a stack underflow crash
```

**Precise crash point:**

The `ret` at the end of `dot_product_buggy` jumps to `addr_after_second_jal`, which is the `add a0, s0, a0` instruction inside `dot_product_buggy` itself. Execution re-enters the function body at an arbitrary point with:
- `sp` already incremented (the frame was deallocated)
- Callee-saved registers already restored to their pre-call values
- `a0 = 39` (the correct return value, but about to be corrupted)

The function does not return to `main` at all. From `main`'s perspective, `dot_product_buggy` never returns — it loops inside itself and eventually causes a stack underflow (repeated `addi sp, sp, 12` without corresponding `addi sp, sp, -12`) or jumps to a garbage address.

**How to detect this bug:**

1. **GDB / debugger:** Stack unwind will show corrupted `ra` at the point of the `ret`.
2. **Address sanitiser (ASAN):** Does not detect this specific bug (no memory violation in the usual case).
3. **Compiler:** `-O0` with `-fno-omit-frame-pointer` plus stack canaries would detect the frame corruption if a canary were placed at `sp+12` (where `ra` should have been saved).
4. **Code review:** A function that calls other functions must always save `ra`. Any non-leaf function without `sw ra, N(sp)` in its prologue is incorrect.
5. **Static analysis:** Tools like `clang-tidy` or custom checkers can flag non-leaf functions without `ra` saves in their assembly output.

**Correct fix:**

Save `ra` in the prologue before the first call:
```assembly
addi sp, sp, -16       # 16 bytes: ra(4) + s0(4) + s1(4) + s2(4)
sw   ra, 12(sp)        # save ra FIRST
sw   s0,  8(sp)
sw   s1,  4(sp)
sw   s2,  0(sp)
...
lw   s2,  0(sp)
lw   s1,  4(sp)
lw   s0,  8(sp)
lw   ra, 12(sp)        # restore ra
addi sp, sp, 16
ret                    # now correctly jumps back to main
```

---

## Discussion

### Key Lessons from This Problem

**1. The "leaf function" optimisation is significant.**

`multiply` requires 0 instructions for prologue/epilogue because it is a leaf. In a tight loop calling `multiply` millions of times, eliminating even 4 instructions per call (allocate, save, restore, deallocate) saves considerable time. The compiler automatically detects leaf functions and omits frame generation.

**2. The caller-saved / callee-saved split protects the right things.**

- `a0`-`a7` are caller-saved because they serve double duty as arguments and temporaries. Requiring the callee to preserve them would defeat their purpose as a pass-through mechanism.
- `s0`-`s11` are callee-saved because they are intended for long-lived values across function calls. A function can use `s0` as a loop variable knowing it survives calls inside the loop.

**3. `ra` is caller-saved — this surprises many candidates.**

`ra` is listed as caller-saved in the ABI table, which means the callee is free to overwrite it. This is correct: if the callee calls another function, it must overwrite `ra`. The callee is responsible for saving `ra` before overwriting it (which is the same thing as saving it in the prologue). The "caller-saved" designation means: *do not expect ra to survive across a call if you are the caller*. This is accurate because the first thing the callee does is overwrite `ra`.

**4. Stack frame size rounding to 16 bytes.**

The 16-byte alignment requirement for `sp` at every call site means frame sizes are always multiples of 16. `main` only needed 4 bytes for `ra`, but allocated 16 bytes. The 12 unused bytes are simply padding. This wastes some stack space but ensures correct alignment.

**5. Register use order in prologues.**

Conventionally, `ra` is saved at the highest address in the frame (largest positive offset from `sp`), and other registers at descending addresses. This is a convention, not a requirement — but consistent ordering makes debugging easier. GCC and Clang both follow this convention.
