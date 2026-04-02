# Problem 03: Branch Penalty and CPI Impact

## Difficulty
Intermediate (covers Tier 1 + Tier 2 material)

## Prerequisites
- Branch prediction strategies: predict-not-taken, 2-bit saturating counter, BTFNT
- CPI analysis framework
- Misprediction penalty calculation for a given pipeline depth

---

## Problem Statement

A RISC-V processor has a 5-stage pipeline where branch conditions are evaluated at the **end of the EX stage** (stage 3). You are evaluating the branch prediction strategy for an embedded control application with the following characteristics:

**Workload profile:**
- 24% of instructions are conditional branches
- 70% of all conditional branches are taken
- There are two distinct branch patterns in the code:
  - **Pattern A (loop back-edges):** 60% of all branches; taken 95% of the time
  - **Pattern B (forward conditionals):** 40% of all branches; taken 37.5% of the time

**Pipeline parameters:**
- Branch resolved at end of EX: 2-cycle misprediction penalty
- All other CPI contributions (load-use, cache misses) total 0.15 cycles/instruction

**Prediction strategies under evaluation:**
1. Predict-not-taken (static)
2. Predict-taken (static)
3. BTFNT (Backward-Taken, Forward-Not-Taken)
4. 2-bit saturating counter (global, 85% overall accuracy)
5. 2-bit saturating counter (per-branch, with warm-up: Pattern A accuracy 97%, Pattern B accuracy 72%)

**Tasks:**

1. Verify the overall "70% taken" figure from the Pattern A and Pattern B breakdown.
2. For each of the five prediction strategies, calculate: misprediction rate, CPI contribution from branches, and total CPI.
3. Rank the strategies by total CPI. Identify the best strategy for this workload.
4. For the per-branch 2-bit predictor (strategy 5), calculate the number of mispredictions per 1000 conditional branches and the total wasted cycles per 1000 instructions.
5. A deeper pipeline alternative is proposed: 10 stages, branch resolved at end of stage 5 (4-cycle misprediction penalty). Using strategy 4 (2-bit global, 85% accuracy), calculate the new CPI and compare with the 5-stage result.

---

## Solution

### Part 1: Verify Overall Taken Rate

```
Total branches split:
  Pattern A: 60% of branches, 95% taken
  Pattern B: 40% of branches, 37.5% taken

Overall taken rate:
  = (0.60 * 0.95) + (0.40 * 0.375)
  = 0.570 + 0.150
  = 0.720

Note: the problem states 70% taken, but the precise per-pattern values give 72%.
The "70%" in the problem statement is an approximation; all subsequent calculations
use the 72% precise figure derived from the per-pattern data.
```

**Verification:** 72% taken is consistent with a mix of frequently-taken loops (Pattern A) and less-frequently-taken forward branches (Pattern B). This is a realistic profile for embedded control code with several loops and moderate conditional logic.

### Part 2: CPI for Each Strategy

**Notation:**
- `f_b = 0.24` (branch fraction)
- `p = 2` (misprediction penalty, cycles)
- `CPI_other = 0.15` (all other stalls)
- `CPI = 1 + CPI_branch + CPI_other`

**CPI_branch formula:**
```
CPI_branch = f_b * misprediction_rate * penalty
           = 0.24 * misprediction_rate * 2
```

---

**Strategy 1: Predict-Not-Taken**
```
Misprediction = any taken branch
Misprediction rate = 72% (overall taken rate)

CPI_branch = 0.24 * 0.72 * 2 = 0.346
Total CPI  = 1 + 0.346 + 0.15 = 1.496
```

**Strategy 2: Predict-Taken**
```
Misprediction = any not-taken branch
Misprediction rate = 1 - 0.72 = 28%

CPI_branch = 0.24 * 0.28 * 2 = 0.134
Total CPI  = 1 + 0.134 + 0.15 = 1.284
```

Note: predict-taken requires the branch target to be known at IF time (BTB or early decode). Without a BTB, there is still a penalty on taken branches even with predict-taken, because the target is not computed until ID or EX. For this analysis, we assume a BTB is present for strategy 2.

**Strategy 3: BTFNT (Backward-Taken, Forward-Not-Taken)**
```
Pattern A (backward, loop back-edges): predict TAKEN
  Misprediction = not-taken predictions for Pattern A
  Pattern A not-taken rate = 1 - 0.95 = 5%
  Pattern A misprediction rate = 5%

Pattern B (forward conditionals): predict NOT TAKEN
  Misprediction = taken predictions for Pattern B
  Pattern B taken rate = 37.5%
  Pattern B misprediction rate = 37.5%

Overall misprediction rate:
  = (0.60 * 0.05) + (0.40 * 0.375)
  = 0.030 + 0.150
  = 0.180 = 18%

CPI_branch = 0.24 * 0.18 * 2 = 0.086
Total CPI  = 1 + 0.086 + 0.15 = 1.236
```

**Strategy 4: 2-Bit Global Saturating Counter (85% accuracy)**
```
Misprediction rate = 1 - 0.85 = 15%

CPI_branch = 0.24 * 0.15 * 2 = 0.072
Total CPI  = 1 + 0.072 + 0.15 = 1.222
```

**Strategy 5: Per-Branch 2-Bit Counter (Pattern A: 97%, Pattern B: 72%)**
```
Pattern A accuracy: 97%, misprediction rate: 3%
Pattern B accuracy: 72%, misprediction rate: 28%

Overall misprediction rate:
  = (0.60 * 0.03) + (0.40 * 0.28)
  = 0.018 + 0.112
  = 0.130 = 13%

CPI_branch = 0.24 * 0.13 * 2 = 0.062
Total CPI  = 1 + 0.062 + 0.15 = 1.212
```

### Part 3: Strategy Ranking

| Strategy | Misprediction Rate | CPI_branch | Total CPI | Rank |
|----------|-------------------|------------|-----------|------|
| 1: Predict-NT    | 72%   | 0.346 | 1.496 | 5 (worst) |
| 2: Predict-Taken | 28%   | 0.134 | 1.284 | 4          |
| 3: BTFNT         | 18%   | 0.086 | 1.236 | 3          |
| 4: 2-bit global  | 15%   | 0.072 | 1.222 | 2          |
| 5: Per-branch 2-bit | 13% | 0.062 | 1.212 | 1 (best)  |

**Best strategy for this workload: Strategy 5 (per-branch 2-bit counter, 13% misprediction rate, CPI = 1.212).**

**Important observations:**

The strong performance of BTFNT (Strategy 3) is noteworthy. It achieves 18% misprediction using only a single sign-bit decode in the IF stage — far cheaper than maintaining a 2-bit table. For a low-power embedded core, BTFNT may be the preferred solution because:
- It requires no SRAM for prediction state.
- It has zero prediction latency (combinational from instruction bits).
- Its misprediction rate (18%) is only marginally worse than the 2-bit global predictor (15%).

The per-branch predictor (Strategy 5) has 13% misprediction vs. 15% for the global predictor. The improvement stems from pattern-specific training: Pattern A achieves 97% accuracy (2-bit counter quickly reaches "strongly taken" and stays there), while Pattern B is harder at 72% (forward branches are less predictable).

Pattern B's 28% misprediction rate with per-branch counters reveals the fundamental limit of 2-bit counters for irregular control flow. A more sophisticated predictor (TAGE) would use longer history to capture patterns in Pattern B branches.

### Part 4: Mispredictions and Wasted Cycles per 1000 Instructions

**Using Strategy 5 (per-branch 2-bit counter):**

```
Per 1000 instructions:
  Number of conditional branches: 1000 * 0.24 = 240 branches

  Pattern A branches: 240 * 0.60 = 144
  Pattern B branches: 240 * 0.40 = 96

  Pattern A mispredictions: 144 * 0.03 = 4.32 ≈ 4 mispredictions
  Pattern B mispredictions: 96  * 0.28 = 26.88 ≈ 27 mispredictions

  Total mispredictions: 4 + 27 = 31 mispredictions per 1000 branches
  (Equivalently: 31 per 1000 * 0.24 = 31 per 240 branches = 12.9% -- confirms 13%)

Wasted cycles per 1000 instructions:
  = mispredictions * penalty
  = 31 * 2
  = 62 wasted cycles per 1000 instructions

CPI contribution check:
  62 wasted cycles / 1000 instructions = 0.062 cycles/instruction
  Matches CPI_branch = 0.062 calculated in Part 2.
```

**Breakdown by pattern:**
```
Pattern A: 4 mispredictions * 2 cycles = 8 wasted cycles per 1000 instructions
Pattern B: 27 mispredictions * 2 cycles = 54 wasted cycles per 1000 instructions

Pattern B accounts for 54/62 = 87% of branch misprediction overhead,
despite representing only 40% of branches.
```

**Actionable insight:** If the compiler can restructure Pattern B branches (e.g., by loop inversion — converting `if (cond) { loop body }` to `do { loop body } while (cond)` where the loop back-edge becomes a backward branch), Pattern B branches may be reclassified as Pattern A, dramatically improving prediction accuracy.

### Part 5: Deeper Pipeline (10-Stage, 4-Cycle Penalty)

**Using Strategy 4 (2-bit global, 85% accuracy) on the 10-stage pipeline:**

```
New penalty: 4 cycles (branch resolved at end of stage 5 in a 10-stage pipeline)
Misprediction rate: 15% (same predictor accuracy)

CPI_branch_new = 0.24 * 0.15 * 4 = 0.144

Other stalls (cache misses, load-use): assume proportional scaling.
In a 10-stage pipeline, load-use penalty is 2 cycles (one extra stage between
EX and WB). Assume CPI_other scales from 0.15 to 0.20 for the deeper pipeline.

Total CPI_10stage = 1 + 0.144 + 0.20 = 1.344
```

**Comparison:**

| Pipeline   | Strategy | Mispred penalty | CPI_branch | CPI_other | Total CPI |
|------------|----------|-----------------|------------|-----------|-----------|
| 5-stage    | 2-bit global | 2 cycles    | 0.072      | 0.15      | 1.222     |
| 10-stage   | 2-bit global | 4 cycles    | 0.144      | 0.20      | 1.344     |

**The 10-stage pipeline increases CPI by 0.122 (10%) despite presumably offering a higher clock frequency.** Whether the frequency gain justifies the CPI increase depends on the ratio of clock frequency improvement to CPI degradation.

```
Break-even analysis:
  Let f_5 = clock frequency of 5-stage, f_10 = clock frequency of 10-stage.
  
  Performance = f / CPI
  
  5-stage performance: f_5 / 1.222
  10-stage performance: f_10 / 1.344

  For the 10-stage pipeline to win:
    f_10 / 1.344 > f_5 / 1.222
    f_10 / f_5 > 1.344 / 1.222 = 1.10

  The 10-stage pipeline must achieve at least 10% higher clock frequency
  to break even on this workload.
```

**Conclusion:** The deeper pipeline requires both a better branch predictor (to limit the increased misprediction penalty) and a significant clock frequency improvement to justify its complexity cost. For embedded applications with the branch-heavy profile described here, a 5-stage pipeline with a per-branch 2-bit predictor (Strategy 5) offers the best CPI with minimal hardware overhead.

---

## Discussion

### Why the global 2-bit predictor underperforms the per-branch predictor

The global 2-bit predictor uses a single table indexed by (some bits of) the branch PC. If the BHT (Branch History Table) has limited entries, two branches may alias to the same table entry — each training the counter in opposite directions, reducing accuracy for both.

In our workload:
- Pattern A branches are strongly taken (95%) and would train a counter to "Strongly Taken."
- Pattern B branches are weakly and irregularly taken (37.5%) and would push toward "Weakly Not Taken."
- If a Pattern A branch and a Pattern B branch alias to the same BHT entry, they interfere, reducing both to ~72-73% accuracy (close to a coin flip for the mixed entry).

A per-branch predictor (Strategy 5) assigns each branch its own entry, preventing aliasing. Pattern A branches train to "Strongly Taken" and stay there; Pattern B branches have independent counters that track their own history. The result: 97% for Pattern A (close to the 95% theoretical maximum) and 72% for Pattern B (limited by the branch's inherent irregularity).

### Impact of warm-up on per-branch predictor accuracy

"Per-branch 2-bit counter with warm-up" means we are measuring steady-state accuracy after the counters have been trained. On first encounter, every branch starts cold (either 2'b00 or 2'b10, depending on initialisation). For a Pattern A loop executed 1000 times, the counter reaches "Strongly Taken" after the first taken iteration and stays there — warm-up cost is negligible. For Pattern B branches in short-lived conditional blocks, warm-up may consume a significant fraction of the execution budget, and the effective accuracy is lower than the 72% steady-state figure.
