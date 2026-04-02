# Problem 01: Pipeline Hazard Identification and Resolution

## Difficulty
Intermediate (covers Tier 1 + Tier 2 material)

## Prerequisites
- 5-stage RISC-V pipeline (IF, ID, EX, MEM, WB)
- RAW, WAR, WAW hazard definitions
- Forwarding and load-use stall concepts

---

## Problem Statement

The following RISC-V instruction sequence is to execute on a classic 5-stage pipeline with:
- **Full forwarding:** EX/MEM -> EX and MEM/WB -> EX forwarding paths are implemented.
- **Load-use stall:** A one-cycle hardware stall is inserted when a load is immediately followed by a dependent instruction.
- **Branch resolution in EX stage:** 2-cycle penalty on a taken branch; predict-not-taken.

```
Address  Instruction
0x00:    LW   x1,  0(x10)      # x1  = mem[x10]
0x04:    ADD  x2,  x1,  x11    # x2  = x1 + x11
0x08:    LW   x3,  4(x10)      # x3  = mem[x10+4]
0x0C:    MUL  x4,  x2,  x3     # x4  = x2 * x3
0x10:    BEQ  x4,  x12,  0x30  # if x4 == x12, branch to 0x30 (TAKEN)
0x14:    ADD  x5,  x4,  x13    # wrong-path instruction (branch taken)
0x18:    SUB  x6,  x5,  x14    # wrong-path instruction (branch taken)
0x30:    OR   x7,  x4,  x15    # correct-path instruction at branch target
```

**Tasks:**

1. Identify every data hazard in the sequence. For each, state: the type (RAW/WAR/WAW), the producer instruction, the consumer instruction, and the register involved.
2. Determine which hazards are resolved by forwarding and which require a stall cycle.
3. Draw the full pipeline execution diagram including stall cycles, forward paths, and the branch squash.
4. Calculate the total execution time in cycles and the effective CPI for the 6 architecturally-executed instructions (0x00 through 0x10 and 0x30).

---

## Solution

### Part 1: Hazard Identification

**Data hazards (RAW only — no WAR or WAW in this in-order sequence):**

| # | Producer       | Consumer       | Register | Distance | Notes                          |
|---|----------------|----------------|----------|----------|--------------------------------|
| H1 | 0x00 LW x1    | 0x04 ADD x2    | x1       | 1 instr  | Load-use: forwarding cannot help |
| H2 | 0x04 ADD x2   | 0x0C MUL x4    | x2       | 2 instrs | MEM/WB forward resolves this   |
| H3 | 0x08 LW x3    | 0x0C MUL x4    | x3       | 1 instr  | Load-use: stall required       |
| H4 | 0x0C MUL x4   | 0x10 BEQ x4    | x4       | 1 instr  | Depends on MUL latency (see note)|
| H5 | 0x0C MUL x4   | 0x30 OR x7     | x4       | depends  | Resolved by time OR executes   |

**Note on H4 (MUL latency):** In a basic 5-stage pipeline, MUL is treated as a single-cycle EX instruction (as in RV32IM without a multi-cycle multiplier). If MUL is pipelined with 1-cycle latency, EX/MEM forwarding resolves H4 with no stall. This solution assumes a 1-cycle MUL for the standard 5-stage model. A multi-cycle MUL unit would introduce additional stalls and is noted in the Discussion.

**Control hazard:**
- Branch at 0x10 is TAKEN. With predict-not-taken and EX-stage resolution: 2-cycle penalty.

### Part 2: Hazard Resolution

| Hazard | Resolution | Penalty |
|--------|-----------|---------|
| H1 (LW->ADD, x1, distance 1) | Hardware stall: 1 cycle | +1 cycle |
| H2 (ADD->MUL, x2, distance 2) | MEM/WB -> EX forward | 0 cycles |
| H3 (LW->MUL, x3, distance 1) | Hardware stall: 1 cycle | +1 cycle |
| H4 (MUL->BEQ, x4, distance 1) | EX/MEM -> EX forward | 0 cycles |
| Branch taken penalty | 2 instructions squashed (0x14, 0x18) | +2 cycles |

**Total stall/penalty cycles = 1 + 1 + 2 = 4**

### Part 3: Pipeline Execution Diagram

```
Cycle:      1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
0x00 LW x1: IF   ID   EX  MEM   WB
0x04 ADD x2:     IF   ID [stl]  EX  MEM   WB         <- H1: stall after ID
0x08 LW x3:          IF [stl]   ID   EX  MEM   WB    <- H1: also stalls in IF
0x0C MUL x4:              [--]  IF   ID [stl]  EX  MEM  WB  <- H3: stall after ID
0x10 BEQ x4:                         IF [stl]  ID   EX [...]
0x14 ADD x5:                              [--]  IF [squash]  <- wrong path, squashed
0x18 SUB x6:                                    IF [squash]  <- wrong path, squashed
0x30 OR x7:                                          IF   ID   EX  MEM  WB

Legend:
  [stl] = stall bubble in ID/EX
  [--]  = instruction frozen in IF or IF stage re-presents same address
  [squash] = instruction flushed from IF (pipeline register zeroed)

Forwarding paths:
  Cycle 6:  0x00 LW x1 in MEM/WB, 0x04 ADD x2 in EX  -> MEM/WB forward for x1
            (This is the cycle after the stall; ADD's EX is cycle 6)
  Cycle 9:  0x08 LW x3 in MEM/WB, 0x0C MUL x4 in EX  -> MEM/WB forward for x3
  Cycle 9:  0x04 ADD x2 in MEM/WB (result), 0x0C MUL x4 in EX -> forward for x2

Branch squash at end of cycle 10 (BEQ resolves in EX):
  0x14 (in ID) and 0x18 (in IF) are flushed. PC redirected to 0x30.
```

**Detailed cycle accounting:**

| Cycle | IF        | ID        | EX        | MEM       | WB        |
|-------|-----------|-----------|-----------|-----------|-----------|
| 1     | LW x1     | -         | -         | -         | -         |
| 2     | ADD x2    | LW x1     | -         | -         | -         |
| 3     | LW x3     | ADD x2    | LW x1 EX  | -         | -         |
| 4     | LW x3(hold)| bubble   | LW x1 MEM | LW x1 MEM | -         |
| 5     | MUL x4    | LW x3     | ADD x2 EX | LW x1 WB  | -         |
| 6     | BEQ x4    | MUL x4    | LW x3 EX  | ADD x2 MEM| LW x1 WB  |
| 7     | 0x14 ADD  | BEQ x4    | bubble    | LW x3 MEM | ADD x2 WB |
| 8     | 0x18 SUB  | 0x14 ADD  | MUL x4 EX | bubble    | LW x3 WB  |

H3 must be re-examined. LW x3 is at 0x08, MUL x4 is at 0x0C. Rechecking the cycle numbering:

```
Corrected pipeline diagram (numbered by instruction entry into IF):

Cycle:  1    2    3    4    5    6    7    8    9   10   11   12   13
LW x1: IF   ID   EX  MEM   WB
ADD x2:     IF   ID  [st]   EX  MEM   WB
                      ^-- stall (H1: LW followed immediately by ADD that reads x1)
LW x3:          IF  [st]   ID   EX  MEM   WB
                     ^-- stall (LW x3 frozen in IF while ADD x2 stalls in ID)
MUL x4:              [--]   IF   ID  [st]  EX  MEM   WB
                              ^-- MUL enters IF in cycle 5
                                      ^-- stall (H3: LW x3 -> MUL x4)
BEQ x4:                           IF  [st]  ID   EX  MEM  WB
                                       ^-- BEQ also stalls while MUL stalls
0x14:                                  IF  [st]  ID [sq]
0x18:                                       [--] IF [sq]
0x30 OR:                                              IF   ID   EX  MEM  WB
                                               ^-- branch resolved in EX, cycle 11
                                                   0x14 in ID squashed
                                                   0x18 in IF squashed
                                                   PC -> 0x30 in cycle 12
```

### Part 4: CPI Calculation

Architecturally executed instructions (correct path): 6
- 0x00 LW x1
- 0x04 ADD x2
- 0x08 LW x3
- 0x0C MUL x4
- 0x10 BEQ x4 (branch taken)
- 0x30 OR x7

The last instruction (0x30 OR x7) completes at cycle 13 + 4 = 17 (IF at cycle 12, WB at cycle 16... let's count precisely):

```
Precise cycle count:
  LW x1:  IF=1,  ID=2,  EX=3,  MEM=4,  WB=5
  ADD x2: IF=2,  ID=3,  stall, EX=5,  MEM=6,  WB=7
  LW x3:  IF=3,  stall, ID=5,  EX=6,  MEM=7,  WB=8
  MUL x4: IF=5,  ID=6,  stall, EX=8,  MEM=9,  WB=10
  BEQ x4: IF=6,  ID=7 (stall), ID=8, EX=9
             (BEQ reads x4; MUL x4 is in EX in cycle 8, BEQ is in ID in cycle 8.
             EX/MEM forwarding covers BEQ's need: at cycle 9, EX/MEM contains MUL's
             result which is forwarded to BEQ in EX. No additional stall for BEQ.)
             BEQ: IF=7, ID=8, EX=9. MUL: EX=8, MEM=9.

  BEQ x4: IF=7, ID=8, EX=9 -> branch taken detected, MEM=10, WB=11
  0x14:   IF=8, ID=9 -> SQUASH at end of cycle 9 (BEQ resolves in EX cycle 9)
  0x18:   IF=9        -> SQUASH at end of cycle 9
  0x30:   IF=10, ID=11, EX=12, MEM=13, WB=14
```

**Total cycles from first instruction start to last instruction complete: 14**

```
CPI = total_cycles / architecturally_executed_instructions
    = 14 / 6
    = 2.33 cycles/instruction

Stall/penalty breakdown:
  H1 load-use stall:   +1 cycle
  H3 load-use stall:   +1 cycle
  Branch taken penalty: +2 cycles
  Total stalls: 4 cycles

CPI_ideal = 1.00 (no stalls)
CPI_actual = (6 + 4) / 6 = 10/6 = 1.67

Note: The 14-cycle total includes the pipeline fill latency (4 cycles for first instruction
to reach WB) and the 4 stall cycles. The "program execution" view excluding pipeline
fill gives CPI = (N_instructions + stall_cycles) / N_instructions = 10/6 = 1.67.
```

---

## Discussion

### Why load-use hazards are unavoidable with forwarding

Even with full forwarding, a LW result is not available until the end of the MEM stage. A directly dependent instruction needs the value at the start of its EX stage — one cycle earlier than the data arrives. No forwarding path can resolve this; only a stall inserts the necessary one-cycle delay.

### The compounding effect of two load-use hazards

Notice that the two load-use hazards (H1 and H3) cost 2 cycles total. Had the code been:

```
LW  x1, 0(x10)
LW  x3, 4(x10)        # independent of x1
ADD x2, x1,  x11      # x1 now one instruction later -> no stall
MUL x4, x2,  x3       # x3 now one instruction later -> no stall
BEQ x4, x12, 0x30
```

Both load-use hazards would be eliminated by instruction reordering — a technique compilers perform at `-O1` and above (`-fschedule-insns`). This restructuring saves 2 cycles (reducing the CPI from 1.67 to 1.33 for this sequence).

### MUL latency in real RISC-V implementations

Real multipliers are not single-cycle at high clock frequencies. A pipelined 3-cycle multiplier (as in the SiFive E31 and similar cores) would introduce an additional 2-cycle stall on the MUL->BEQ dependency (H4). This further increases CPI. When profiling code, MUL instructions preceding dependent loads or branches are a significant hazard source.

### Branch penalty and its interaction with stalls

The two-cycle branch penalty occurs at the end of the sequence. If the branch had been not-taken, 0x14 and 0x18 would execute normally (no squash) and would themselves suffer no additional hazards. The interaction of hazard stalls and branch penalties means that the worst-case scenario is a branch that is mispredicted AND whose condition register was produced by a load (load -> branch RAW through a forwarding chain), requiring stalls before even reaching the branch penalty.
