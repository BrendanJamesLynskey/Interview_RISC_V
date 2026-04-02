# Problem 02: Lock-Free Data Structures Using LR/SC

## Problem Statement

**Context:** You are implementing a concurrent runtime library for a RISC-V multiprocessor embedded system running four harts. The system has the A extension but no operating system — all synchronisation must be implemented in bare-metal assembly. You must implement two lock-free data structures and analyse their correctness under RVWMO.

---

### Part A: Lock-Free Stack (Push and Pop)

Implement a lock-free stack using LR/SC. The stack is a singly-linked list with a shared `top` pointer. Each node has the layout:

```c
struct Node {
    int    value;   // offset 0
    struct Node *next;  // offset 4 (RV32)
};

struct Stack {
    struct Node *top;   // offset 0, initially NULL
};
```

Implement `push(Stack *s, Node *node)` and `pop(Stack *s) -> Node*` in RISC-V assembly. Specify the memory ordering required and justify your choice.

---

### Part B: Bounded Sequence Number Generator

Implement an atomic sequence number generator with overflow detection. The generator must:

1. Atomically increment a shared counter and return the new value.
2. Return a special sentinel value (`-1`) if the counter has already reached `MAX_SEQ = 1000`.
3. Never skip a sequence number (no gaps).
4. Operate correctly under simultaneous calls from all four harts.

The counter variable is a 32-bit integer at address `counter_addr`.

---

### Part C: Correctness Analysis

A colleague proposes the following lock-free flag-setting routine:

```asm
# Set bit n in a flag word at address a0, but only if it is currently clear.
# a1 = bit index (0-31)
# Returns a0 = 1 if we set the bit, 0 if it was already set.
try_set_bit:
    li    t0, 1
    sll   t0, t0, a1       # t0 = 1 << n (bitmask)
.retry:
    lw    t1, 0(a0)        # load current flags
    and   t2, t1, t0       # check if bit already set
    bnez  t2, .already_set
    or    t2, t1, t0       # compute new value with bit set
    sw    t2, 0(a0)        # store back   <--- PROBLEM HERE
    li    a0, 1
    ret
.already_set:
    li    a0, 0
    ret
```

Identify every correctness issue with this code and provide a correct replacement using LR/SC with appropriate ordering.

---

## Solution

### Part A: Lock-Free Stack

#### Push

`push` must atomically update `s->top` to point to `node` while setting `node->next` to the old `top`. The critical atomicity requirement: no other hart must modify `s->top` between the time we read it (to set `node->next`) and the time we write back.

```asm
# push(Stack *s, Node *node)
# a0 = &s->top  (pointer to the top pointer)
# a1 = node to push

# step 1: set node->next = current top (can be done non-atomically before the LR/SC;
# if the LR/SC fails, we will overwrite node->next on the next iteration)
push:
.retry:
    lr.w.aq   t0, (a0)          # t0 = current top, acquire
    sw        t0, 4(a1)         # node->next = old top  (4 = offset of next pointer)
    sc.w.rl   t1, a1, (a0)      # try: top = node, release
    bnez      t1, .retry        # SC failed, retry
    ret
```

**Ordering analysis:**

- `LR.W.AQ`: acquire semantics. Subsequent operations (the `SW` into node->next and the `SC`) cannot be observed before this load. This ensures that any thread that reads the pushed node via `top` after the `SC` is guaranteed to also see the correct `node->next` value — the store to `node->next` is ordered before the store to `top`.

- `SC.W.RL`: release semantics. The `SW node->next = old_top` is guaranteed to be visible before the `SC` that publishes the node. This is the release barrier protecting the node initialisation.

**Why the `SW` before the `SC` is safe:** the `sw t0, 4(a1)` writes to `node->next` within the LR/SC pair. This write is to the *private* node being pushed — it is not the shared variable protected by LR/SC. Writing to it does not invalidate the reservation on `s->top`. However, the spec requires that the LR/SC pair not contain stores to addresses other than the SC address if the implementation is to guarantee forward progress. Strictly, this means for spec-guaranteed success we should set `node->next` before the `LR`:

```asm
push_correct:
.retry:
    lw        t0, 0(a0)         # non-atomic read of top (for setting next)
    sw        t0, 4(a1)         # node->next = observed top
    lr.w.aq   t0, (a0)          # re-read top atomically (may differ from first read)
    sw        t0, 4(a1)         # update node->next with reservation-read value
    sc.w.rl   t1, a1, (a0)      # CAS: top = node
    bnez      t1, .retry
    ret
```

The extra `sw` inside the LR/SC is necessary to use the value read by the `LR` (which is the canonical atomic value). This remains within the constrained LR/SC sequence (fewer than 16 instructions, no backward branches within).

---

#### Pop

`pop` must atomically: load the current top, load `top->next`, and store `top->next` into `s->top`. If any other hart modifies `s->top` between our `LR` and `SC`, we retry.

```asm
# pop(Stack *s) -> Node* (or NULL if empty)
# a0 = &s->top
# Returns: a0 = popped node pointer, or NULL

pop:
.retry:
    lr.w.aq   t0, (a0)          # t0 = current top, acquire
    beqz      t0, .empty        # stack is empty (top == NULL)
    lw        t1, 4(t0)         # t1 = top->next  (load next pointer from node)
    sc.w.rl   t2, t1, (a0)      # try: s->top = top->next, release
    bnez      t2, .retry        # SC failed (another hart modified top), retry
    mv        a0, t0            # return the popped node
    ret
.empty:
    li        a0, 0             # return NULL
    ret
```

**Why this is ABA-immune:**

The LR/SC combination tracks whether `s->top` was written by any hart since the `LR`. Even if another hart pops a node and pushes back a node with the same address (ABA), the `SC` will fail — any write to `s->top` since the `LR` invalidates the reservation, regardless of whether the written value is the same as what was read.

**The `lw t1, 4(t0)` inside the LR/SC pair:** this load is to `top->next`, a different address from the reservation (`s->top`). The spec permits this within a constrained LR/SC pair. If the `SC` fails, we retry and reload `t0` with a fresh `LR`, then load `top->next` again from the (potentially different) node.

**Potential hazard: use-after-free.** If another hart pops `t0` from the stack and frees the node between our `LW t1, 4(t0)` and the `SC`, we will read from freed memory. This is a fundamental hazard of lock-free lists in environments with manual memory management. Solutions include hazard pointers, epoch-based reclamation, or a node pool that never frees nodes to the OS allocator.

---

### Part B: Bounded Sequence Number Generator

```asm
# Atomically increment counter if < MAX_SEQ, return new value or -1
# a0 = &counter
# Returns: a0 = new sequence number, or -1 if counter >= 1000

.equ MAX_SEQ, 1000

next_seq:
.retry:
    lr.w.aq   t0, (a0)           # t0 = current counter value, acquire
    li        t1, MAX_SEQ
    bge       t0, t1, .overflow  # if counter >= 1000, cannot increment

    addi      t2, t0, 1          # t2 = new value = current + 1
    sc.w.rl   t3, t2, (a0)       # try: counter = counter + 1, release
    bnez      t3, .retry         # SC failed, retry

    mv        a0, t2             # return new value (counter + 1)
    ret

.overflow:
    # Release the reservation (SC with x0 discards the store and signals failure)
    # Actually, we just don't issue SC; the reservation will be abandoned.
    # But we must be careful: we should issue a dummy SC or just let the LR expire.
    # Safest: just return without SC (reservation will time out).
    li        a0, -1
    ret
```

**Discussion of the overflow path:**

When the counter is already at `MAX_SEQ`, we return -1 without executing an `SC`. This abandons the reservation established by `LR`. The spec permits this — it simply means the reservation expires without a store. Other harts waiting to do their own `LR/SC` on this address are not affected (the LR reservation is local to our hart).

**Proof of no skipped sequence numbers:**

Each successful `SC` increments the counter from exactly the value read by the paired `LR`. If the counter is 7 at the `LR`, the `SC` stores 8 and returns 0 (success). No other hart can atomically change the counter between our `LR` and our `SC` without making our `SC` fail (by invalidating the reservation). Therefore, every new sequence number is exactly `previous + 1`, and no number is ever skipped.

**Starvation analysis:**

On a system with four harts all calling `next_seq` simultaneously:
- All four `LR`s read the same value (e.g., 42).
- All four attempt `SC`. Only one succeeds; the other three fail and retry.
- On retry, the value has advanced to 43. Three more `LR`s, one more succeeds.
- Pattern: each round of N harts produces exactly one successful increment.

The spec guarantees that a constrained LR/SC sequence will eventually succeed if no other hart is also in an LR/SC sequence on the same address continuously. With four harts, eventually each gets a turn. On highly contended counters, performance degrades to one increment per N rounds — this is acceptable for a sequence generator used at modest rates.

For high-throughput counters under contention, `AMOADD` is superior (one instruction, hardware-guaranteed atomic, no retry loop).

---

### Part C: Correcting the Flag-Setting Routine

#### Issues in the Original Code

**Issue 1: Non-atomic read-modify-write (critical bug).**

The original sequence is:
1. `LW t1, 0(a0)` — load flags.
2. Check if bit set, compute new value.
3. `SW t2, 0(a0)` — store new flags.

Steps 1 and 3 are not atomic. Between the load and store, another hart may write to `0(a0)`. If Hart 0 and Hart 1 both read `flags = 0`, both compute `new_flags = (1 << n0)` and `(1 << n1)` respectively, and both store — the second store overwrites the first. One bit-set is lost.

**Issue 2: No memory ordering.**

The plain `LW` and `SW` have no `aq`/`rl` semantics. If this flag is used to signal readiness of other data (e.g., "data at address X is ready when bit n is set"), other harts may observe the flag set before the underlying data is written. A release fence on the `SW` is required.

**Issue 3: No retry on concurrent modification.**

Even if the read-modify-write were somehow correct (it is not), the code has no mechanism to detect that another hart modified the flag word between the load and store.

#### Correct Implementation Using LR/SC

```asm
# try_set_bit: atomically set bit a1 in word at a0 if currently clear
# a0 = address of flag word
# a1 = bit index (0..31)
# Returns a0 = 1 (success: we set the bit), a0 = 0 (bit was already set)

try_set_bit:
    li    t0, 1
    sll   t0, t0, a1           # t0 = 1 << n

.retry:
    lr.w.aq   t1, (a0)         # t1 = current flags (acquire: subsequent ops ordered after)
    and       t2, t1, t0       # check if bit already set
    bnez      t2, .already_set # bit is set, return 0

    or        t2, t1, t0       # t2 = flags | (1 << n)
    sc.w.rl   t3, t2, (a0)    # try to store (release: prior writes visible to observers)
    bnez      t3, .retry       # SC failed (another hart changed flags), retry

    li        a0, 1            # success
    ret

.already_set:
    # We do not execute SC here. The LR reservation will expire.
    li        a0, 0
    ret
```

**Why the ordering bits are correct:**

- `LR.W.AQ`: ensures that if we read a clear bit and proceed to set it, all our subsequent operations (including anything we do after confirming we own the bit) are ordered after the bit read. Another hart that later reads this flag with an `LR.AQ` is guaranteed to see our `SC` store before it reads, if our `SC` completes before their `LR`.

- `SC.W.RL`: ensures that all stores we made before the `SC` (e.g., writing data that the flag "protects") are visible before the flag itself is written. This is the standard release-store pattern for publishing data to other threads.

**Memory model guarantee:**

If Hart 0 calls `try_set_bit` and succeeds, and Hart 1 then calls `try_set_bit` for the same bit and sees it already set: Hart 1's `LR.AQ` observes Hart 0's `SC.RL`. By the acquire-release pairing in RVWMO, Hart 1 is also guaranteed to observe all stores that Hart 0 performed before its `SC.RL`. This is the mutual exclusion invariant: whoever wins the race to set the bit is guaranteed that the other party sees the bit as already set.

---

## Discussion

### Patterns and Principles

**The LR/SC pair as a software CAS.** Every LR/SC loop implements compare-and-swap: "if the memory location still contains the value I read, swap in the new value; otherwise, retry." This is more expressive than a fixed-operation CAS because the "new value" can be computed from the old value via arbitrary instructions between LR and SC (within the 16-instruction constraint).

**Constrained sequences.** The spec only guarantees eventual forward progress for LR/SC sequences that:
- Contain 16 or fewer instructions.
- Contain no memory operations other than the LR/SC pair.
- Contain no backward branches.

Violating these constraints does not make the code incorrect — the LR/SC may still succeed — but the hardware is not obligated to guarantee it ever succeeds. On a heavily loaded cache-coherent bus, an SC that is repeatedly pre-empted by external writes could theoretically livelock. In practice, this is extremely rare, but real-time systems should keep LR/SC sequences as short as possible.

**Acquire on load, release on store.** The canonical pattern for building higher-level synchronisation primitives:
- Lock acquisition: `LR.AQ` or `AMO.AQ` — prevents critical section from starting before lock is read.
- Lock release: `SC.RL` or `AMO.RL` or `SW` preceded by `FENCE W, W` — prevents critical section stores from becoming visible after lock release.

**AMO vs. LR/SC trade-off.** For the sequence number generator in Part B, `AMOADD` would be simpler and faster:

```asm
# Using AMOADD (if we don't need the bounds check)
li      t0, 1
amoadd.w.aqrl  t1, t0, (a0)   # t1 = old value, counter incremented atomically
addi    a0, t1, 1             # a0 = new value
```

But adding the bounds check (`return -1 if >= MAX_SEQ`) requires a conditional operation that AMO cannot express — the operation must not increment if already at the limit. This is why LR/SC is necessary here: the decision to store (and what to store) depends on the loaded value.
