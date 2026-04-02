# A Extension: Atomic Instructions

## Overview

The RISC-V A extension provides the minimum set of atomic primitives needed to implement all standard synchronisation algorithms on multiprocessor systems. It offers two complementary mechanisms: Load-Reserved/Store-Conditional (LR/SC), which supports arbitrary read-modify-write sequences, and Atomic Memory Operations (AMOs), which perform specific read-modify-write operations in a single instruction. The `aq` (acquire) and `rl` (release) ordering bits allow fine-grained control over memory ordering without requiring a full fence. Together these primitives form the foundation of the RISC-V memory model (RVWMO) and are the atomic backbone of every RISC-V Linux and RTOS kernel.

---

## Key Concepts

### The RISC-V Memory Model (RVWMO)

RISC-V uses a relaxed memory model called RVWMO (RISC-V Weak Memory Order). Key properties:

- Loads and stores to different addresses may be observed out of program order by other harts.
- A hart always observes its own operations in program order.
- The memory model is weaker than TSO (x86) but stronger than pure PowerPC/ARM weakly ordered models.
- Atomics with `aq`/`rl` bits introduce ordering constraints.

A `FENCE` instruction provides an explicit ordering barrier between memory operations. The A extension's ordering bits allow targeted ordering without a full fence.

### Load-Reserved / Store-Conditional

LR/SC provides a primitive for implementing compare-and-swap and other arbitrary atomic read-modify-write sequences:

```
LR.W  rd, (rs1)      # Load-Reserved: load word from rs1, mark address as reserved
SC.W  rd, rs2, (rs1) # Store-Conditional: store rs2 to rs1 if reservation still valid
                     # rd = 0 on success, rd = non-zero on failure
```

`LR.W` loads a word from memory and records a *reservation* on that memory address. `SC.W` attempts to store to the reserved address. The store succeeds (and `rd` = 0) only if:

1. No other hart has written to the reserved address since the `LR.W`.
2. No exception has occurred between `LR.W` and `SC.W`.
3. No other `LR.W` has been executed by this hart since the pairing `LR.W`.

If any condition is violated, the store does not occur and `rd` is set to a non-zero value (failure). Software must loop on failure.

**Architectural constraints on LR/SC sequences:**

The spec defines a *constrained LR/SC sequence* that implementations must guarantee will eventually succeed:

- The sequence contains only `LR.W`, `SC.W`, and branch instructions between LR and SC.
- The sequence contains at most 16 instructions total.
- The sequence contains no memory operations other than the LR and SC.
- The sequence contains no backward branches (no loops within the LR/SC critical section).

Implementations are only required to guarantee eventual success for sequences meeting these constraints. Arbitrary code between LR and SC may cause the SC to always fail on some implementations.

**RV64A adds** `LR.D`/`SC.D` for 64-bit granularity.

### Atomic Memory Operations (AMOs)

AMOs perform a read-modify-write atomically: they load from a memory address, perform an operation, and write the result back, all without any other hart being able to observe an intermediate state.

```
AMO<op>.W  rd, rs2, (rs1)
```

Loads word from `rs1` into `rd` (original value), then atomically stores `rd <op> rs2` back to `rs1`.

| Instruction    | Operation stored back   | Notes |
|----------------|------------------------|-------|
| `AMOSWAP.W`    | `rs2`                  | Atomic exchange |
| `AMOADD.W`     | `rd_orig + rs2`        | Atomic increment |
| `AMOAND.W`     | `rd_orig & rs2`        | Atomic bit-clear |
| `AMOOR.W`      | `rd_orig | rs2`        | Atomic bit-set |
| `AMOXOR.W`     | `rd_orig ^ rs2`        | Atomic bit-flip |
| `AMOMAX.W`     | `max(rd_orig, rs2)`    | Signed maximum |
| `AMOMAXU.W`    | `maxu(rd_orig, rs2)`   | Unsigned maximum |
| `AMOMIN.W`     | `min(rd_orig, rs2)`    | Signed minimum |
| `AMOMINU.W`    | `minu(rd_orig, rs2)`   | Unsigned minimum |

All AMOs are available in `.W` (32-bit) and `.D` (64-bit, RV64A) variants.

**Important:** the destination register `rd` always receives the *original* value loaded from memory (before the operation). This allows the caller to observe what value was present before the modification.

### Memory Ordering: the `aq` and `rl` Bits

Every LR, SC, and AMO instruction has two 1-bit ordering qualifiers in the encoding:

- **`aq` (acquire):** No memory operation after this instruction (in program order) may be observed before this instruction completes. This instruction acts as an acquire barrier.
- **`rl` (release):** No memory operation before this instruction (in program order) may be observed after this instruction completes. This instruction acts as a release barrier.

Setting both `aq=1, rl=1` gives sequentially consistent behaviour for that instruction.

| `aq` | `rl` | Ordering semantic |
|------|------|-------------------|
| 0    | 0    | Relaxed — no ordering guarantees |
| 1    | 0    | Acquire — subsequent loads/stores cannot be reordered before this |
| 0    | 1    | Release — previous loads/stores cannot be reordered after this |
| 1    | 1    | Sequentially consistent (strongest) |

**Acquire-release pattern for a mutex lock/unlock:**

```asm
# Lock: acquire semantics on the LR/SC that claims the lock
.lock_loop:
    lr.w.aq   t0, (a0)        # load-reserved with acquire
    bnez      t0, .lock_loop  # spin if lock is not 0 (already taken)
    li        t1, 1
    sc.w.rl   t0, t1, (a0)    # store-conditional with release
    bnez      t0, .lock_loop  # retry if SC failed

# Unlock:
    amoswap.w.rl  x0, x0, (a0)  # store 0 with release semantics
```

The acquire on the load prevents critical section operations from being reordered before the lock acquisition. The release on the store/unlock prevents critical section operations from being reordered after the unlock.

### Encoding

LR/SC and AMO instructions use a specialised R-type encoding:

```
[31:27] funct5 | [26] aq | [25] rl | [24:20] rs2 | [19:15] rs1 | [14:12] funct3 | [11:7] rd | [6:0] opcode
```

For `LR`, `rs2 = x0` (unused). The `opcode = 0101111` (AMO). `funct3` selects word (`010`) or doubleword (`011`).

| Instruction  | funct5  |
|--------------|---------|
| `LR`         | 00010   |
| `SC`         | 00011   |
| `AMOSWAP`    | 00001   |
| `AMOADD`     | 00000   |
| `AMOXOR`     | 00100   |
| `AMOAND`     | 01100   |
| `AMOOR`      | 01000   |
| `AMOMIN`     | 10000   |
| `AMOMAX`     | 10100   |
| `AMOMINU`    | 11000   |
| `AMOMAXU`    | 11100   |

### Alignment Requirements

LR/SC and AMO instructions require **naturally aligned** addresses. An `AMO.W` requires 4-byte alignment; `AMO.D` requires 8-byte alignment. An unaligned address raises a store/AMO address-misaligned exception (or store/AMO access fault for hardware-enforced alignment).

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What is the difference between LR/SC and AMO instructions? When would you use each?**

**Answer:**

**LR/SC** (Load-Reserved/Store-Conditional) is a pair of instructions that together create an atomic read-modify-write critical section. `LR` loads a value and marks the address as reserved; `SC` conditionally stores a new value only if the reservation is still valid. The critical section between LR and SC can contain arbitrary computation (within constraints) — this makes LR/SC the more general primitive.

Use LR/SC when:
- Implementing compare-and-swap (CAS) semantics.
- The new value depends on a complex function of the old value that cannot be expressed as a single AMO operation.
- Building higher-level primitives like mutexes, condition variables, or lock-free queues.

**AMO** instructions perform one of a fixed set of operations (swap, add, and, or, xor, min, max) atomically in hardware. They are non-looping — a single instruction, guaranteed to succeed.

Use AMO when:
- The operation is one of the supported operations (especially `AMOADD` for counters, `AMOSWAP` for locks).
- Performance matters and the fixed-operation limitation is acceptable.
- Implementing simple reference counting, statistics counters, or flag setting.

Trade-off: AMOs have lower overhead (one instruction vs. a loop) but are limited to specific operations. LR/SC can express any CAS-based algorithm but may need to retry on contention.

---

**Q2. What does `AMOSWAP.W rd, rs2, (rs1)` do? What value ends up in `rd` after the instruction?**

**Answer:**

`AMOSWAP.W` atomically:
1. Loads the 32-bit word at address `rs1` into `rd`.
2. Stores the value of `rs2` to address `rs1`.

Steps 1 and 2 are atomic — no other hart can observe the memory between the load and the store.

`rd` contains the **original value** that was in memory before the swap. `rs2` (unchanged by this instruction) is what gets written to memory.

Example use — acquiring a spinlock:

```asm
# a0 = address of lock variable (0 = free, 1 = held)
li      t0, 1                  # value to write (lock = taken)
amoswap.w.aq  t1, t0, (a0)    # t1 = old lock value; write 1 to lock
bnez    t1, .spin_fail         # if old value was non-zero, lock was already taken
# Lock acquired
```

If `t1 == 0`, the lock was free and we have now set it to 1 — lock acquired. If `t1 != 0`, the lock was already taken and we must retry or wait.

---

**Q3. Write a RISC-V assembly sequence to atomically increment a counter at address `a0` and return the new value in `a0`.**

**Answer:**

```asm
# Increment counter at address a0, return new value in a0
li      t0, 1
amoadd.w  t1, t0, (a0)   # t1 = old value; memory[a0] += 1

# t1 holds the value BEFORE the increment
# new value = old value + 1
addi    a0, t1, 1        # a0 = new value (original + 1)
```

`AMOADD.W` writes `old + rs2` to memory and returns `old` in `rd`. Adding 1 to the returned old value gives the new value.

If only the old value is needed (common for release-sequence patterns):

```asm
# Return old value in a0, increment memory
li      t0, 1
amoadd.w  a0, t0, (a0)   # a0 = value before increment
```

For a sequentially consistent atomic increment (e.g., for a global reference counter):

```asm
li      t0, 1
amoadd.w.aqrl  t1, t0, (a0)   # aq+rl = sequentially consistent
addi    a0, t1, 1
```

---

**Q4. What does the `aq` bit do on an `LR.W` instruction? Why is it important for implementing a mutex lock?**

**Answer:**

The `aq` (acquire) bit on `LR.W.AQ` establishes a memory ordering constraint: **no memory operation that follows this `LR` in program order can be observed by any other hart before this `LR` is observed.** Informally, the acquire barrier prevents subsequent memory accesses from "leaking" before the lock acquisition.

For a mutex:

```asm
.try_lock:
    lr.w.aq  t0, (a0)      # load lock state; ACQUIRE
    bnez     t0, .try_lock # already locked, retry
    li       t1, 1
    sc.w     t1, t1, (a0)  # attempt to claim lock
    bnez     t1, .try_lock # SC failed (another hart got there first), retry
# Critical section starts here
```

Without `.aq`, the processor or memory system could reorder loads and stores from inside the critical section to execute *before* the lock acquisition is visible to other harts. This would allow another hart to simultaneously believe it also holds the lock, and both would operate on the supposedly protected resource together — a data race.

With `.aq`, all subsequent memory operations (the critical section) are guaranteed to be observed after the lock acquisition, preserving mutual exclusion.

---

**Q5. What is the return value of `SC.W` on success, and on failure? What does "failure" mean architecturally?**

**Answer:**

- **Success:** `rd = 0`. The store was performed atomically.
- **Failure:** `rd = non-zero` (typically 1, but any non-zero value is valid). The store was **not** performed; memory was not modified.

Failure occurs when the reservation established by the matching `LR.W` is no longer valid. Architectural causes of reservation loss:

1. **Another hart wrote to the reserved address** (the most common case — contention on a shared variable).
2. **An exception or interrupt occurred** between the `LR` and `SC` — the reservation is abandoned.
3. **Another `LR.W` was executed by this hart** — each hart holds at most one reservation at a time; a new `LR` cancels the previous one.
4. **Implementation-specific events** — e.g., a cache eviction of the reserved line, a TLB shootdown, or a spurious invalidation from a write-combining buffer. The spec permits spurious SC failures (implementations may be conservative), so software must always loop on failure.

The key property: if `SC` succeeds, the write happened atomically relative to all other harts. If it fails, the write did not happen, and the loop retries safely.

---

### Intermediate Tier

---

**Q6. Implement a compare-and-swap (CAS) operation using LR/SC. The function signature is `int cas(int *addr, int expected, int new_val)` — return 1 on success, 0 on failure.**

**Answer:**

```asm
# a0 = addr, a1 = expected, a2 = new_val
# Returns: a0 = 1 (success), a0 = 0 (failure)
cas:
    lr.w.aq   t0, (a0)         # load current value with acquire
    bne       t0, a1, .fail    # if current != expected, CAS fails
    sc.w.rl   t1, a2, (a0)    # try to store new_val with release
    bnez      t1, .retry       # SC failed (reservation lost), retry
    li        a0, 1            # success
    ret

.retry:
    # Reservation was lost; reload and check again
    lr.w.aq   t0, (a0)
    bne       t0, a1, .fail
    sc.w.rl   t1, a2, (a0)
    bnez      t1, .retry
    li        a0, 1
    ret

.fail:
    li        a0, 0            # failure: value changed before we could swap
    ret
```

Simplified loop version (cleaner but same semantics):

```asm
cas:
.loop:
    lr.w.aq   t0, (a0)        # load with acquire
    bne       t0, a1, .fail   # not expected value
    sc.w.rl   t1, a2, (a0)   # try swap with release
    bnez      t1, .loop       # SC failed, retry from scratch
    li        a0, 1           # success
    ret
.fail:
    li        a0, 0
    ret
```

Note: the re-check after `.loop` ensures we retry only when the value still equals `expected`. If another hart changed the value between the LR and SC, the `bne` catches it on the next iteration.

This is a **strong CAS** — it fails only if the current value does not equal `expected`. A **weak CAS** could additionally fail spuriously (SC failure not due to value mismatch), but the loop above effectively makes this strong by retrying SC failures.

---

**Q7. Explain the ABA problem. Can it occur with LR/SC on RISC-V? Can it occur with AMOs?**

**Answer:**

The **ABA problem** arises in lock-free algorithms when:
1. Thread 1 reads value A from a shared location.
2. Thread 2 changes the value A → B → A.
3. Thread 1 performs a CAS, sees A, and succeeds — but the memory has been through two state transitions that Thread 1 missed.

**LR/SC on RISC-V:** LR/SC is naturally immune to ABA. The `LR` places a reservation on the *memory location*, not just the value. If any other hart writes to the address between `LR` and `SC` — even if it writes back the same value — the reservation is invalidated and `SC` fails. Thread 1's `SC` would fail after the A→B→A sequence because Thread 2's write of B (and then A) both invalidated the reservation.

This makes LR/SC superior to standard CAS for many lock-free algorithms. A CAS-based implementation would have to add a version counter (double-width CAS / DCAS) to detect ABA.

**AMO instructions:** AMOs cannot exhibit ABA because they are single instructions — there is no window between load and store for another hart to intervene. The read-modify-write happens atomically. However, AMOs do not check what value was present; they unconditionally perform their operation. ABA does not apply to single AMO operations because they do not have a "check and then act" semantics.

The ABA problem only applies to multi-step algorithms where you read, compute, then conditionally write based on the read value.

---

**Q8. Why must addresses used with LR/SC and AMO instructions be naturally aligned? What exception is raised on a misaligned access?**

**Answer:**

**Natural alignment requirement:**

An `AMO.W` or `SC.W` operates on a 32-bit (4-byte) word. It must be 4-byte aligned. An `AMO.D` or `SC.D` operates on a 64-bit (8-byte) doubleword and must be 8-byte aligned.

**Why alignment is required:**

1. **Hardware implementation simplicity:** AMO operations must complete atomically. An unaligned word may straddle two cache lines (e.g., bytes at offset 0x1E and 0x20). Atomically accessing two cache lines simultaneously requires coordinating two cache controllers — a significant hardware cost. Natural alignment guarantees the access fits within one cache line and one coherence domain.

2. **Cache coherence protocol:** the reservation for LR/SC is typically tracked at cache-line granularity. An aligned word lives entirely within one line; an unaligned word could span two lines with different MESI states.

3. **Atomicity guarantee:** the fundamental guarantee that no intervening observation is possible depends on the operation being confined to a single coherence unit.

**Exception raised:**

An unaligned AMO/LR/SC raises a **store/AMO address-misaligned exception** (`mcause = 6`). If the platform implements a PMP or page table that rejects the access, it raises a **store/AMO access fault** (`mcause = 7`) or **store/AMO page fault** (`mcause = 15`).

Note: even on platforms that support misaligned regular loads/stores in hardware, AMO misalignment always traps — there is no "transparent misaligned AMO" emulation.

---

**Q9. Describe what happens when an SC.W instruction is executed without a preceding LR.W from the same hart. Is this architecturally defined?**

**Answer:**

Executing `SC.W` without a preceding `LR.W` from the same hart is architecturally defined: the `SC` simply **fails** (returns non-zero in `rd`) and does not modify memory. This is guaranteed by the spec.

Rationale: the reservation mechanism requires a prior `LR` to establish a valid reservation. Without one, no reservation exists for the `SC` to honour. The `SC` always checks for a valid reservation and always fails if one does not exist.

Implications:
- Software that accidentally executes `SC` without `LR` will see a failure code but will not corrupt memory — the guaranteed non-destructive failure is a safety property.
- This also means `SC` can be used as a "try-store" with guaranteed failure as a sentinel in some algorithms, though this is unusual.
- The address in `rs1` for the failed `SC` is not accessed; since no reservation exists, no memory operation occurs. No address-related exceptions can arise from the address itself (though a misaligned or faulting address in `rs1` may still raise an exception depending on implementation — the spec is slightly ambiguous here; most implementations check the address only on a successful path).

---

**Q10. What is the purpose of using `AMOSWAP.W rd, x0, (a0)` with `rl` set? Give a concrete use case.**

**Answer:**

`AMOSWAP.W.RL rd, x0, (a0)` atomically writes 0 to the memory location at `a0` and returns the original value in `rd`. The `.RL` (release) ordering ensures all memory operations preceding this instruction in program order are visible to other harts before this store is visible.

**Use case: releasing a spinlock.**

```asm
# Unlock: store 0 to lock variable with release semantics
# a0 = address of lock
amoswap.w.rl  x0, x0, (a0)   # write 0 (unlock), rd=x0 (discard old value), release
```

The release ordering is critical: it ensures that all stores to shared data within the critical section are visible to other harts before the lock is observed as released. Without the release, another hart could acquire the lock and observe stale shared data.

An alternative using `SC` (less efficient — requires a loop):
```asm
# Or with a plain store + fence:
fence  rw, w          # release fence: drain all prior reads/writes
sw     x0, 0(a0)      # store 0 to release lock
```

`AMOSWAP` with `rl` is preferred because it combines the fence and store into one instruction, reducing code size and potentially latency.

Using `x0` as `rs2` writes 0. Using `x0` as `rd` discards the original value (we do not need it when unlocking — we do not care what the old lock state was, since we hold it). This is the canonical RISC-V spinlock release idiom.

---

### Advanced Tier

---

**Q11. A lock-free stack requires an atomic pop operation. Implement `pop` using LR/SC. Discuss why AMOs alone cannot implement this.**

**Answer:**

A lock-free stack stores a pointer to the top node. Each node contains a value and a `next` pointer. Pop must atomically read the top pointer, load `next` from the top node, and update the top pointer to `next` — all as one atomic operation.

```c
struct Node { int value; struct Node *next; };
struct Node *stack_top;  // shared, atomic head pointer
```

```asm
# RV32 assembly: a0 = address of stack_top, returns popped value in a0
pop:
.retry:
    lr.w.aq   t0, (a0)         # t0 = current top pointer (acquire)
    beqz      t0, .empty       # stack is empty (top == NULL)
    lw        t1, 4(t0)        # t1 = top->next  (non-atomic read, safe because:
                                # we own the reservation; if t0 is freed/reused,
                                # our SC will fail and we retry -- still ABA-safe due to LR/SC)
    sc.w.rl   t2, t1, (a0)    # CAS: stack_top = top->next (release)
    bnez      t2, .retry       # SC failed (another hart modified stack_top), retry
    lw        a0, 0(t0)        # return t0->value (our popped node's value)
    ret
.empty:
    li        a0, -1           # sentinel: stack was empty
    ret
```

**Why AMOs alone cannot implement this:**

Pop requires a *conditional* update: "change `stack_top` from old value `X` to `X->next`." AMOs perform unconditional operations:
- `AMOSWAP` would exchange the top pointer for a fixed value, not a value derived from the old top.
- `AMOADD` makes no sense for pointer arithmetic in this context.

No AMO can express "load the next-pointer from the *currently loaded top node* and swap it in" — that requires reading from two memory locations (the stack top, and then `top->next`) in a single atomic unit. LR/SC allows this because the arbitrary computation between LR and SC is precisely where the `lw t1, 4(t0)` (load next pointer) occurs.

The LR/SC is ABA-immune here: if `top` is popped by another hart and then a node with the same address is pushed back, the `SC` still fails because the write to `stack_top` since the `LR` invalidates the reservation.

---

**Q12. Explain the difference between `FENCE`, `FENCE.I`, and the `aq`/`rl` ordering bits. When would you use each?**

**Answer:**

**`FENCE iorw, iorw`** — a memory ordering barrier for the data memory system.

```
FENCE predecessor, successor
```

The predecessor and successor fields each contain flags `I` (device input), `O` (device output), `R` (memory read), `W` (memory write). `FENCE RW, RW` is the strongest fence: all reads and writes before the fence are ordered before all reads and writes after it. Common patterns:

- `FENCE W, R`: "store-load" fence — expensive, needed for Peterson's algorithm and dekker-style synchronisation. x86 `MFENCE` is equivalent.
- `FENCE RW, W`: release fence (stronger than `rl` bit alone).
- `FENCE R, RW`: acquire fence.

**`FENCE.I`** — an instruction fence, not a data fence. It ensures that stores to instruction memory are visible to subsequent instruction fetches on the same hart. Required after patching self-modifying code or JIT-compiling a function. `FENCE.I` is a `Zifencei` extension instruction; it is not guaranteed to maintain coherence between data stores on one hart and instruction fetches on *another* hart (that may require cross-hart IPI + `FENCE.I` on each hart).

**`aq`/`rl` bits on LR/SC/AMO** — attach acquire or release semantics to a single atomic instruction:
- `aq=1`: the atomic acts as an acquire fence for subsequent memory accesses.
- `rl=1`: the atomic acts as a release fence for preceding memory accesses.
- More efficient than a separate `FENCE` + AMO because the ordering is folded into the atomic operation itself.

**When to use each:**

| Need | Mechanism |
|------|-----------|
| Mutex lock | `LR.W.AQ` + `SC.W.RL` or `AMOSWAP.AQ` |
| Mutex unlock | `AMOSWAP.RL` or `SW` + `FENCE W, R` |
| Sequentially consistent atomic | `AMO.AQRL` |
| DMA buffer ready flag | `FENCE W, W` then write flag |
| JIT code generation | `FENCE.I` after writing instruction bytes |
| Full memory barrier (between threads without atomics) | `FENCE RW, RW` |

---

**Q13. On a weakly ordered RISC-V system, two harts execute the following code. What are the possible outcomes for `(r1, r2)` after both have completed?**

```
Hart 0:             Hart 1:
x = 1               y = 1
r1 = y              r2 = x
```

(`x` and `y` are separate shared memory locations, initially 0.)

**Answer:**

Under RVWMO (the RISC-V relaxed memory model), the stores (`x = 1`, `y = 1`) and loads (`r1 = y`, `r2 = x`) can be reordered within each hart's visible effects, and neither hart's stores need to become globally visible in program order to the other hart.

Possible outcomes for `(r1, r2)`:

| r1 | r2 | Explanation |
|----|----|-------------|
| 1  | 1  | Hart 0's store visible before Hart 1's load; Hart 1's store visible before Hart 0's load |
| 1  | 0  | Hart 0's y=1 visible to Hart 1's r1 load; Hart 1's x=1 not yet visible to Hart 0's r2 load |
| 0  | 1  | Hart 1's x=1 visible; Hart 0's y=1 not yet visible |
| **0**  | **0**  | Both loads see stale values — allowed under RVWMO but NOT under TSO (x86) |

`(r1=0, r2=0)` is the surprising outcome: both loads see the initial value 0 even though both stores have been "issued." This can happen if:
- Hart 0's store to `x` is buffered and not yet globally visible when Hart 1 reads `x`.
- Simultaneously, Hart 1's store to `y` is buffered and not yet globally visible when Hart 0 reads `y`.

Under x86 TSO, `(0, 0)` is forbidden because stores must be globally visible before any subsequent load by another thread. RVWMO allows it.

To prevent `(0, 0)`, insert `FENCE W, R` between the store and load on both harts:

```
Hart 0:              Hart 1:
x = 1                y = 1
FENCE W, R           FENCE W, R
r1 = y               r2 = x
```

Now `(0, 0)` is forbidden because each fence ensures the store is globally visible before the load is performed.

---

**Q14. A real-time kernel needs to implement a ticket lock (fair FIFO mutex) using RISC-V atomics. Implement it with the A extension.**

**Answer:**

A ticket lock uses two counters: `next_ticket` (atomically incremented to claim a slot) and `now_serving` (incremented to release the lock). A hart holds the lock when its ticket equals `now_serving`.

```c
struct TicketLock {
    volatile int next_ticket;  // at offset 0
    volatile int now_serving;  // at offset 4
};
```

```asm
# lock(a0 = &ticketlock)
ticket_lock:
    li      t0, 1
    amoadd.w.aq  t1, t0, (a0)    # t1 = my ticket; atomically next_ticket++
                                  # acquire: no subsequent ops move before this
.spin:
    lw      t2, 4(a0)             # load now_serving
    bne     t1, t2, .spin         # spin until now_serving == my ticket
    # Memory barrier to ensure critical section loads don't precede the spin
    fence   r, rw                  # acquire fence for the spin load
    ret

# unlock(a0 = &ticketlock)
ticket_unlock:
    li      t0, 1
    amoadd.w.rl  x0, t0, 4(a0)   # now_serving++, release semantics
    ret
```

Key design points:

1. `AMOADD.W.AQ` on `next_ticket`: the acquire ordering ensures that loads/stores within the critical section are not observed before the ticket is claimed.
2. The spin loop uses a plain `LW` (more efficient than `LR` since we are just polling, not attempting a CAS). A `WFI` or `PAUSE` hint (if Zihintpause is available) can reduce bus traffic.
3. `AMOADD.W.RL` on `now_serving`: the release ordering ensures all critical section stores are visible before the ticket is incremented and the next waiter proceeds.
4. **Fairness:** unlike a spinlock where any waiting hart might acquire next, the ticket lock serves in strict FIFO order — the hart with the lowest ticket number always proceeds next.
5. **Scalability concern:** the spinning `LW` generates cache coherence traffic. For high-contention locks, each waiter should spin on a per-thread flag rather than the shared `now_serving` (MCS lock pattern).

---

**Q15. On a system with a non-coherent scratchpad memory (no cache, no cache coherence protocol), can LR/SC be used? What does the spec say?**

**Answer:**

This is a subtle architectural question. The RISC-V spec states that LR/SC must be to memory that supports the LR/SC reservation mechanism. The spec does not mandate that LR/SC works on all memory types — it is implementation-defined whether LR/SC is supported on non-cacheable or I/O memory regions.

**On non-coherent scratchpad memory:**

The reservation mechanism in most implementations is tracked by the L1 cache coherence controller (watching for invalidations of the reserved cache line). A non-coherent scratchpad bypasses the cache entirely:

1. There is no cache line to invalidate.
2. There is no coherence protocol to signal when another hart writes the reserved address.
3. The reservation state machine has no way to detect a conflicting write from another hart.

**Spec position:** The RISC-V privileged spec (and the A extension spec) states that SC *may* always fail on regions where LR/SC is not naturally supported. Implementations are permitted to make SC always fail on non-cacheable regions. Software relying on LR/SC in non-cacheable regions has undefined (or implementation-defined) behaviour.

**Practical guidance:**
- Use LR/SC only on normal cacheable memory.
- For MMIO synchronisation, use `AMOSWAP` if the device supports it, or use a mutex in cacheable memory to protect MMIO access.
- Check the platform documentation: some SoCs extend coherence to specific SRAM regions (making LR/SC work there), while others do not.
- PMP/PMA (Physical Memory Attributes) can mark regions as non-atomic, causing LR/SC to always fail with defined semantics.
