---
layout: page
title: Energy-Efficient FFT Accelerator
description:
img: assets/img/fft_layout.png
importance: 1
category: work
---

**Course:** ET4351 Digital VLSI Systems on Chip — TU Delft (2025/2026)  
**Tools:** SystemVerilog · Cadence Genus · Cadence Innovus  
**Role:** RTL design, synthesis, clock gating (Energy-Efficient team)

---

### The Problem

The baseline FFT accelerator computes a 32-point Cooley-Tukey FFT in **732 clock cycles** at 12 MHz (61 µs), consuming **24.6 nJ** per chunk. The goal was to reduce energy below the 0.403 mW baseline power target without raising the clock frequency.

Profiling the baseline immediately revealed why it was slow: **8 out of every 9 cycles per butterfly were memory transactions.** Each complex sample required two separate 32-bit reads and two separate 32-bit writes because real and imaginary parts were stored in consecutive words. The FSM had 13 states — one per memory transaction — so the datapath sat idle waiting for each individual read or write to complete before moving on.

{% include figure.liquid path="assets/img/FSM.png" title="FSM comparison" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    redesigned FSM
</div>

---

### Step 1 — Stop Wasting Power When Idle

The multipliers in the baseline were clock-enabled for all 13 FSM states, even when only one state actually computed anything. The first fix was simple: gate the multiplier enable so it only fires during the `COMPUTE` state.

This alone dropped energy from **24.6 nJ → 8.5 nJ** — a 2.9× reduction without touching the architecture.

---

### Step 2 — Fix the Memory Bandwidth Bottleneck

With idle switching addressed, the structural problem remained: 4 reads and 4 writes per butterfly, each requiring a separate cycle.

The fix was to pack each complex sample (real + imaginary) into a single **64-bit word** instead of two 32-bit words. One read now fetches both components simultaneously, halving the memory transactions per butterfly from `4R + 4W` to `2R + 2W`.

<div class="row justify-content-center">
<div class="col-md-8">

| | Baseline | Optimised |
|---|---|---|
| Memory word width | 32-bit | 64-bit |
| Reads per butterfly | 4 | 2 |
| Writes per butterfly | 4 | 2 |
| FSM states | 13 | 7 |

</div>
</div>

This halved the cycle count per butterfly and collapsed the FSM from 13 to 7 states. Energy dropped from **8.5 nJ → 3.11 nJ**.

---

### Step 3 — Eliminate Twiddle Factor Multiplications

The baseline computed twiddle factors at runtime using an iterative recurrence: at each butterfly, the accumulator was updated as `w ← (w · wm) >> 12`. Across a full 32-point FFT, this generated **320 multiplications** that had nothing to do with the actual signal — they were pure overhead for tracking a predictable mathematical pattern.

Since twiddle factors depend only on stage and butterfly index (never on input data), they can be precomputed. Better still, the cosine and sine functions have **quarter-wave symmetry** — the entire twiddle table for all stages reduces to just 5 base values at angles 0, π/16, π/8, 3π/16, π/4. The correct factor for any (stage, k) pair is recovered combinationally from these 5 entries with no multiplications and no memory reads at runtime.

Removing 320 multiplications — one of the most power-hungry operations in the datapath — brought energy from **3.11 nJ → 1.37 nJ**.

---

### Step 4 — Pipeline the Remaining Stages

With the memory interface widened and twiddle factors gone, the remaining bottleneck was sequential execution: the FSM still processed one butterfly at a time — read, then compute, then write — with each stage waiting for the previous to finish.

The solution was to restructure into a **3-stage pipeline** (READ → COMPUTE → WRITE) running continuously, and split the memory into **two independent 64-bit banks**. The key insight enabling conflict-free dual access is that in the Cooley-Tukey butterfly, the two operand addresses `base + k` and `base + k + half` always differ by `half`, which is a power of two. This guarantees they always land on opposite banks when routed via an XOR bank selector — so both operands can be read (and written) simultaneously with zero arbitration.

{% include figure.liquid path="assets/img/dual_mem.png" title="Dual memory bank" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    Dual memory bank
</div>

The FSM collapsed from 7 states to just **3 states: INIT, RUN, FINISH**, with all butterfly work happening inside a single RUN state that sustains one completed butterfly per clock cycle once the pipeline is filled.

This brought the cycle count from 246 → **84 cycles** and energy from **1.37 nJ → 0.269 nJ**.

---

### Step 5 — Physical Optimizations (Synthesis & P&R)

With the architecture locked, the synthesis and place-and-route flow was tuned for power:

- **Integrated Clock Gating:** ICG cells were inserted to disable clock toggling on idle registers. This gated **4,481 out of 4,970 flip-flops (90%)**, directly cutting dynamic power from the clock network.
- **High-Vt Cell Mapping:** Non-critical paths were mapped to HVT library cells, which trade switching speed for lower leakage. Since the design had 33 ns of setup slack at 12 MHz, there was ample margin to absorb the slower cells.
- **Power-Driven Innovus Flow:** `optPower` was run pre-CTS, post-CTS, and post-route with `leakageToDynamicRatio 0.5`, allowing the tool to trade area for power reduction across the full implementation.

---

### Final Results

<div class="row justify-content-center">
<div class="col-md-9">

| Metric | Baseline | Final Design | Improvement |
|---|---|---|---|
| Frequency | 12 MHz | 12 MHz | — |
| Latency (cycles) | 732 | 84 | **8.7× faster** |
| Latency (µs) | 61.0 | 7.0 | — |
| Accelerator Power | 0.403 mW | 0.038 mW | **10.5× lower** |
| Energy per chunk | 24.6 nJ | 0.269 nJ | **91× lower** |
| DRC Violations | 0 | 0 | ✓ |
| DRV Violations | 0 | 0 | ✓ |
| Setup WNS | 33.8 ns | 33.6 ns | ✓ |
| Hold WNS | 0.032 ns | 0.010 ns | ✓ |
| Activity annotation | 100% | 100% | ✓ |

</div>
</div>

Each optimisation step built on the last, with the cumulative effect shown below:

<div class="row justify-content-center">
<div class="col-md-7">

| Optimisation Step | Energy |
|---|---|
| Baseline | 24.6 nJ |
| Operand isolation | 8.5 nJ |
| 64-bit read & write | 3.11 nJ |
| Twiddle ROM | 1.37 nJ |
| Dual-port + pipelining | **0.269 nJ** |

</div>
</div>

The design passed all physical signoff checks — DRC, connectivity, geometry, and antenna — at the 596.4 µm × 596.4 µm core area constraint.