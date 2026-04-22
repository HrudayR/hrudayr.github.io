---
layout: page
title: Real-Time Posterization Filter on FPGA
description:
img: assets/img/video_thumbnail.png
importance: 2
category: work
---

**Tools:** Vitis HLS · Vivado · Python · OpenCV · Jupyter (PYNQ)  
**Focus:** HLS design, streaming pipeline architecture, line buffer/sliding window memory, AXI integration  
**Context:** CESE4090 Reconfigurable Computing Design, TU Delft (Q2 2025/2026)

---

<!-- ============================================================ -->
<!-- 📷 HERO IMAGE — insert Fig. 1 (Before / After posterization) -->
<!--    Suggested caption: "Posterization filter: smooth colour   -->
<!--    gradients become flat bands with cartoon-like outlines."   -->
<!-- ============================================================ -->
{% include figure.liquid path="assets/img/before_after.png" title="Architecture diagram" class="img-fluid rounded z-depth-1" %}

### The Problem

The goal: drop a real-time video processing block into an HDMI pipeline on a **PYNQ-Z2 (Xilinx Zynq-7000)** board and keep up with a **1280 × 720 stream at 60 fps**. The pipeline moves one pixel per clock, so whatever algorithm goes in has to finish its per-pixel work inside a single clock period — otherwise the stream stalls.

Posterization made for a strong candidate. It's a non-photorealistic effect that flattens smooth colour gradients into discrete bands and overlays dark edges, giving a cartoon-like look. From an engineering standpoint, it's a good fit for FPGA acceleration because it combines several things that pure software handles badly:

- Local spatial filtering on 3×3 neighbourhoods (Gaussian blur, box blur, Sobel)
- Edge detection followed by thresholding
- Per-channel colour quantization
- Two independent processing paths that can run in parallel on the same pixel stream

A quick Python prototype on the ARM Cortex-A9 confirmed why this wasn't a software problem: **1831 ms per frame** — roughly 110× too slow for 60 fps. The bottleneck was pure compute, not memory (peak RAM usage was only 63 MB). That framed the whole design brief: push the per-pixel work into programmable logic and let the CPU just handle configuration.

---

### Algorithm Overview

Posterization breaks down into five stages:

1. **Colour conversion** — convert the RGB input to grayscale, which separates structure from colour for cleaner edge detection.
2. **Edge detection** — denoise the grayscale with a Gaussian low-pass, then apply Sobel operators and threshold the gradient magnitude to build a binary edge map.
3. **Smoothing** — apply a 3×3 box filter to each RGB channel to suppress small colour fluctuations before quantizing.
4. **Quantization** — map each channel from 256 levels down to a small set of discrete levels, producing the flat cartoon-style regions.
5. **Overlay** — combine the edge map with the quantized colours: edge pixels render as black, everything else keeps its quantized colour.

<!-- ============================================================ -->
<!-- 📷 IMAGE — insert Fig. 2 (Posterization workflow diagram)    -->
<!--    Suggested caption: "End-to-end posterization workflow:    -->
<!--    two parallel paths combined by the overlay stage."        -->
<!-- ============================================================ -->
{% include figure.liquid path="assets/img/video_pipeline.png" title="Architecture diagram" class="img-fluid rounded z-depth-1" %}
---

### Software Prototype

Before building anything in hardware, the algorithm ran first in Python with OpenCV to verify it worked end-to-end — grayscale conversion via `cv2.cvtColor`, Gaussian denoising, Sobel derivatives, magnitude thresholding, optional `cv2.dilate` for thicker outlines, box blur per channel, uniform quantization, and a final masking step. Straightforward, but sequential and slow.

Benchmarking 200 frames on the Cortex-A9 gave a **mean latency of 1831 ms** per frame (throughput ≈ 0.55 fps), with P95 and max latencies tight against the mean (1939 ms and 1965 ms). That tight spread was useful diagnostic information: the runtime was stable and consistent, which ruled out OS jitter or background tasks and confirmed the work itself was the bottleneck. Combined with the lean 63.4 MB peak memory, the picture was unambiguous — **compute-bound, not memory-bound**. Exactly the kind of workload hardware offload is designed for.

---

### Hardware Implementation

The FPGA block was built as a Vitis HLS IP core with **AXI4-Stream** for pixel data and **AXI4-Lite** for runtime parameters. Three control registers were exposed — `edge_threshold`, `bright_factor`, and `quant_levels` — so the effect could be tuned live from the Python wrapper without re-synthesizing.

To keep up with the 60 fps HDMI pipeline, the design had to hit an **Initiation Interval of 1** — one completed pixel per clock, every clock.

<!-- ============================================================ -->
<!-- 📷 IMAGE — insert Fig. 5 (Hardware pipeline block diagram)  -->
<!--    Suggested caption: "Hardware pipeline: two parallel paths -->
<!--    (edge detection + colour quantization) merged by a final  -->
<!--    multiplexer. Line buffers in BRAM feed 3×3 windows."       -->
<!-- ============================================================ -->

{% include figure.liquid path="assets/img/hardware_pipeline.png" title="Architecture diagram" class="img-fluid rounded z-depth-1" %}

#### Streaming memory: line buffers and sliding windows

The core memory challenge: Sobel, Gaussian, and box filters all need a 3×3 neighbourhood, but the stream only delivers one pixel per cycle. The fix was **line buffers in BRAM** plus **fully-partitioned 3×3 sliding windows in registers**. Each line buffer stores one full image row. When a new pixel arrives, it enters the buffers and older pixels shift through, effectively caching the last few rows of the stream.

On every clock cycle, the 3×3 window slides left — column 0 ← column 1, column 1 ← column 2, column 2 ← freshly-fetched pixels from the input and line buffers. That gives combinational access to a full 3×3 neighbourhood every cycle without stalling.

#### Two parallel paths

Splitting the design into two datapaths operating on the same incoming pixel was the key architectural win for meeting II=1:

**Path 1 — Edge detection.** RGB is converted to grayscale, stored in its own line buffers, denoised with the Gaussian kernel, then run through Sobel. The gradient magnitude is compared against `edge_threshold` to produce a binary edge signal. A small logical-OR between the current and previous edge output thickens the edges slightly for better visual continuity.

**Path 2 — Colour smoothing and quantization.** The original RGB is buffered separately, blurred through a 3×3 box filter, brightness-scaled, and quantized to `quant_levels` discrete bands per channel.

**Overlay.** A final multiplexer selects between paths: edge pixels output as black, non-edge pixels output as their quantized colour.

#### Latency matching between paths

Because each 3×3 filter introduces a one-line delay, the grayscale path (Gaussian → Sobel = two stacked filters) ends up **two lines delayed** relative to the input, while the RGB path (one box filter) is only one line delayed. To keep the edge overlay aligned with the correct colour pixel, the RGB path needed an **extra line buffer** — a small cost in BRAM to preserve correctness at the merge point.

#### Fixed-point arithmetic and LUT tricks

Several choices were made to avoid expensive hardware operations:

- **Grayscale coefficients** (`Y = 0.299R + 0.587G + 0.114B`) were approximated as `(77R + 150G + 29B) >> 8`, eliminating floating-point.
- **Gaussian kernel** used integer weights `[1 2 1; 2 4 2; 1 2 1]` with a 4-bit right-shift divisor.
- **Sobel gradient magnitude** was approximated as `|Gx| + |Gy|` instead of `sqrt(Gx² + Gy²)`, skipping the square-root block entirely.
- **Quantization** replaced runtime division with **precomputed lookup tables** for step sizes and inverse step factors, using multiply-and-shift with a small rounding correction.

---

### Integration Flow

After functional verification in the Vitis C simulator, the generated IP was exported into Vivado and dropped into the existing video pipeline. The resulting `.bit` and `.hwh` files got loaded onto the PYNQ-Z2 board as an overlay, where a Python wrapper in a Jupyter notebook could control parameters and run the filter live on HDMI input.

<!-- ============================================================ -->
<!-- 📷 IMAGE — insert Fig. 7 (Vivado block design diagram)       -->
<!--    Suggested caption: "Vivado block design: the posterization -->
<!--    IP dropped into the HDMI → VDMA pipeline."                 -->
<!-- ============================================================ -->

{% include figure.liquid path="assets/img/vivado_pipeline.png" title="Architecture diagram" class="img-fluid rounded z-depth-1" %}
---

### Results

The software baseline ran at **1831 ms per frame (0.55 fps)** — about 110× too slow for 60 fps real-time video. Offloading to FPGA dropped mean latency to **16.77 ms per frame** and pushed throughput to **59.64 fps**, essentially hitting the 16.67 ms real-time budget. That's a **~109× speedup**, closing the gap from "impossible in software" to "runs live on HDMI."

Peak memory dropped from 63.4 MB on the ARM side to just 13.4 MB, since the heavy lifting now happens in on-chip BRAM instead of DDR. Some frames occasionally push slightly past the 16.67 ms target because of VDMA memory-transfer jitter — an artefact of moving full frames through DDR over AXI rather than anything algorithmic — but the design sustains 60 fps playback.

### Resource Utilization

The final design fits comfortably within the PYNQ-Z2's Zynq-7020 fabric: roughly **14,016 LUTs** (~13,229 as logic, 787 as memory), **24,391 flip-flops**, **16.5 BRAM tiles**, and **66 DSP blocks**. BRAM usage is dominated by the line buffers (one for grayscale, one for denoised grayscale, three for RGB channels plus the latency-matching buffer), and the DSPs primarily handle the weighted-sum convolutions and the quantization multiply-shift paths.

---

### Takeaways

The interesting part of this kind of work is that **memory architecture decides whether an algorithm can stream**, not raw compute. The Sobel and blur math are trivial; the engineering was in the line buffers, the sliding windows, and making sure the two parallel paths stayed latency-matched at the merge point. Everything else — the fixed-point tricks, the LUT-based quantization, the OR-based edge thickening — came down to staying inside the one-pixel-per-clock budget without introducing stalls.