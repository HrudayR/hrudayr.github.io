---
layout: page
title: Energy-Efficient FFT Accelerator
description:
img: assets/img/fft_layout.png
importance: 1
category: work
---

**Tools:** SystemVerilog · Cadence Genus · Cadence Innovus  
**Focus:** RTL architecture, synthesis flow, clock gating, physical optimization  

---

{% include figure.liquid path="assets/img/fft_layout.png" title="FSM comparison" class="img-fluid rounded z-depth-1" %}


### The Problem

The starting point was a working 32-point Cooley-Tukey FFT accelerator that took **732 clock cycles** at 12 MHz (61 µs) and burned **24.6 nJ** per chunk. The challenge: cut energy as deeply as possible without raising the clock frequency.

A quick profile of where the cycles were going made the bottleneck obvious. **8 out of every 9 cycles per butterfly were memory transactions.** Every complex sample needed two separate 32-bit reads and two separate 32-bit writes because the real and imaginary parts lived in consecutive words. The FSM had 13 states — one per memory transaction — and the datapath sat idle through almost all of them. Memory bandwidth, not compute, was the real bottleneck. That insight shaped every decision that followed.

<div class="row justify-content-center">
    <div class="col-md-6">
        {% include figure.liquid path="assets/img/FSM.png" title="FSM comparison" class="img-fluid rounded z-depth-1" %}
        <div class="caption">Redesigned FSM</div>
    </div>
</div>

---

### Step 1 — Stop Wasting Power When Idle

The baseline left the multipliers clock-enabled across all 13 FSM states even though only one state actually computed anything. The cheapest possible win came first: gate the multiplier enable so it only fires during `COMPUTE`.

That single change dropped energy from **24.6 nJ → 8.5 nJ** — a 2.9× reduction before touching the architecture at all. It also set the tone for the rest of the project: aggressively hunt down anything switching that doesn't need to be.

---

### Step 2 — Fix the Memory Bandwidth Bottleneck

With idle switching dealt with, the structural problem was still in the way: 4 reads and 4 writes per butterfly, each on its own cycle.

The next move was to redesign the memory layout — packing each complex sample (real + imaginary) into a single **64-bit word** instead of two 32-bit words. One read now pulls both halves at once, halving memory traffic per butterfly from `4R + 4W` to `2R + 2W`. The FSM collapsed from 13 to 7 states and energy fell from **8.5 nJ → 3.11 nJ**.

---

### Step 3 — Eliminate Twiddle Factor Multiplications

The baseline computed twiddle factors at runtime with an iterative recurrence: at each butterfly, the accumulator updated as `w ← (w · wm) >> 12`. Across a full 32-point FFT, this generated **320 multiplications** that had nothing to do with the actual signal — pure overhead spent tracking a fully predictable mathematical pattern.

The key observation: twiddle factors only depend on the stage and butterfly index, never on input data — which means they can be precomputed. Even better, cosine and sine have **quarter-wave symmetry**, so the entire twiddle table for every stage compresses down to just 5 base values at angles 0, π/16, π/8, 3π/16, π/4. The correct factor for any (stage, k) pair is recovered combinationally from those 5 entries — no multiplications, no memory reads at runtime.

Killing 320 multiplications — one of the most power-hungry operations in the datapath — brought energy from **3.11 nJ → 1.37 nJ**.

---

### Step 4 — Pipeline the Remaining Stages

With memory wider and twiddle multiplications gone, the last bottleneck was sequential execution: the FSM still chewed through one butterfly at a time — read, compute, write — each waiting on the last to finish.

The fix was a full restructure into a **3-stage pipeline** (READ → COMPUTE → WRITE) running continuously, with the memory split into **two independent 64-bit banks**. The trick that made conflict-free dual access possible came from looking carefully at the Cooley-Tukey butterfly itself: the two operand addresses `base + k` and `base + k + half` always differ by `half`, which is a power of two. That guarantees they always land on opposite banks under an XOR bank selector — so both operands can be read (and written) on the same cycle with zero arbitration logic. No stalls, no contention, ever.

{% include figure.liquid path="assets/img/dual_mem.png" title="Dual memory bank" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    Dual memory bank
</div>

The FSM collapsed from 7 states down to **3: INIT, RUN, FINISH**, with all the butterfly work happening inside the single RUN state — sustaining one completed butterfly per clock cycle once the pipeline fills.

Cycle count dropped from 246 → **84 cycles** and energy from **1.37 nJ → 0.269 nJ**.

---

### Step 5 — Physical Optimizations (Synthesis & P&R)

With the architecture locked in, attention shifted to tuning the synthesis and place-and-route flow specifically for power:

- **Integrated Clock Gating:** ICG cells were inserted to disable clock toggling on idle registers. The flow gated **4,481 out of 4,970 flip-flops (90%)**, cutting dynamic power from the clock network directly.
- **High-Vt Cell Mapping:** Non-critical paths were pushed onto HVT library cells, which trade switching speed for lower leakage. With 33 ns of setup slack at 12 MHz, there was massive headroom to absorb the slower cells without breaking timing.
- **Power-Driven Innovus Flow:** `optPower` was run pre-CTS, post-CTS, and post-route with `leakageToDynamicRatio 0.5`, letting the tool trade area for power across the full implementation.

---

### Final Results

The final design runs at the same 12 MHz as the baseline but completes a chunk in just **84 cycles (7 µs)** instead of 732 cycles (61 µs) — **8.7× faster**. Accelerator power dropped from **0.403 mW to 0.038 mW** (10.5× lower), and energy per chunk fell from **24.6 nJ to 0.269 nJ** — a **91× overall improvement**.

Tracking each step's contribution: operand isolation took the design from 24.6 nJ → 8.5 nJ, the 64-bit memory interface to 3.11 nJ, the twiddle ROM to 1.37 nJ, and dual-port pipelining brought the final figure to **0.269 nJ**. Every step compounded on the last.

The design passed all physical signoff checks — DRC, connectivity, geometry, antenna — at the 596.4 µm × 596.4 µm core area, with positive setup slack of 33.6 ns and hold slack of 0.010 ns. Power numbers were measured with 100% activity annotation coverage, so they reflect real switching behavior rather than estimated averages.