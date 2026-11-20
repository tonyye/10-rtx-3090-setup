# High-Density GPU Workstation: System Configuration & Bandwidth Benchmark Report

**Date:** 2026-11-12
**Type:** Technical configuration and performance-characterization article

---

## 1. System Overview

This machine is a purpose-built, high-capacity GPU workstation/server designed for
compute-heavy workloads such as large language model (LLM) inference and high-performance
computing. It pairs a 12-GPU configuration with a dual AMD EPYC platform that provides ample
PCIe lanes and a large memory footprint.

### Key hardware
| Component | Detail |
|-----------|--------|
| **Motherboard** | GIGABYTE **MZF2-AC0-00** (server-class EPYC board), BIOS **R20** |
| **CPU** | **AMD EPYC 7542** — 32-core Zen2 "Rome", dual-socket IOD with 8 PCIe root complexes |
| **GPUs** | **12 total** — 10× NVIDIA RTX 3090 + 2× AMD Radeon RX 7900 XTX |
| **Platform** | PCIe Gen4 throughout, one dedicated root-port link per GPU |
| **PCIe slots 7 & 8** | Dedicated x16 slots, each fitted with an x16 → 2× x8 bifurcation kit |
| **Power** | Five ATX 3.1 / PCIe 5.1 power supplies, ~5.25 kW combined |
| **Serving** | Project Maya on the 10× RTX 3090, Ollama on the 2× RX 7900 XTX — both at once |

### PCIe slot layout and bifurcation

**Slots 7 and 8** are the two dedicated x16 slots, and each one is subdivided rather than used
whole: it carries an **RGEEK PCIe 4.0 x16 → 2× x8 bifurcation kit**
([Amazon B0H5R9D1KZ](https://www.amazon.com/dp/B0H5R9D1KZ)):

| Attribute | Detail |
|-----------|--------|
| Upstream port | PCIe 4.0 x16 (256 GT/s), plugged into the slot |
| Downstream ports | 2× independent PCIe 4.0 x8 (128 GT/s) |
| Cabling | SlimSAS 4.0 **SFF-8654 8i** |
| Kit contents | 1 main card, 2 breakout boards, 2× 75 cm SFF-8654 8i cables |
| Breakout power | Each breakout board takes a **6-pin PCIe** feed; in x8+x8 mode both cables land on **Slot 1** of the breakout board |
| BIOS requirement | The slot's bifurcation preset must be set to **x8+x8** *before* the cards will enumerate |

The two kits therefore turn slots 7 and 8 into **four Gen4 x8 links**, which is how ten
RTX 3090s and two RX 7900 XTX cards fit on one board while every GPU keeps its own root-port
link instead of sharing a slot's bandwidth with another card.

> **Note on link widths:** the measured table in Section 5 was captured with the current BIOS
> preset and reports GPU 8 (`C1:00.0`) at **Gen4 x16**. A bifurcated slot presents only x8 to
> each half, so either that slot is running its x16 preset rather than x8+x8, or GPU 8 is not
> sitting behind one of the two bifurcation kits. Links are re-checked per slot when the
> bifurcation preset changes.

---

## 2. Power Delivery — Five Power Supplies

The 12-GPU load is split across **five separate ATX 3.1 / PCIe 5.1 supplies** rather than one
large unit. Every supply is fully modular and carries a native **12V-2x6** (12VHPWR successor)
connector, which keeps the 3090s on native cables instead of daisy-chained adapters.

| # | Supply | Wattage | Efficiency | ATX / PCIe | Native 12V-2x6 | Notes |
|---|--------|---------|-----------|-----------|----------------|-------|
| PSU-1 | [MONTECH Century II 1200W](https://www.amazon.com/dp/B0F3XV451X) | 1200 W | 80+ Gold | ATX 3.1 / PCIe 5.1 | Yes | Compact 140 mm body, Zero RPM mode, 100% Japanese capacitors, 10-yr warranty |
| PSU-2 | [be quiet! Pure Power 13 M 850W](https://www.amazon.com/dp/B0FBY3F1NT) | 850 W | 80+ Gold, Cybenetics Platinum | ATX 3.1 / PCIe 5.1 | Yes | LLC resonant topology, semi-passive fan, single rail, 10-yr warranty |
| PSU-3 | [MSI MAG A1000GL PCIE5](https://www.amazon.com/dp/B0CT3XNCZ9) | 1000 W | 80+ Gold | ATX 3.1 / PCIe 5.1 | Yes | Native dual-colour 12V-2x6 connector, compact fully modular body, 10-yr warranty |
| PSU-4 | [CORSAIR RM1000x](https://www.amazon.com/CORSAIR-RM1000x-Modular-Low-Noise-Supply/dp/B0DJ1M9C62) | 1000 W | Cybenetics Gold | ATX 3.1 / PCIe 5.1 | Yes | Low-noise fan curve, fully modular |
| PSU-5 | [CORSAIR HX1200i (2025)](https://www.amazon.com/CORSAIR-HX1200i-Modular-Ultra-Low-12V-2x6/dp/B0F1NF61BQ) | 1200 W | Cybenetics Platinum | ATX 3.1 / PCIe 5.1 | Yes | Ultra low-noise, fluid dynamic bearing fan, fully modular |

**Combined nominal capacity:** 1200 + 850 + 1000 + 1000 + 1200 = **5,250 W**.

The efficiency mix is broad: three units are Gold-tier (PSU-1, PSU-3, PSU-4) and two reach
Cybenetics Platinum (PSU-2 and PSU-5; PSU-2 is additionally rated 80+ Gold). All five share the
same ATX 3.1 / PCIe 5.1 baseline and the same native 12V-2x6 connector, so the 3090s are fed
from native cables on whichever supply they are attached to.

---

## 3. GPU Configuration (all 12)

### NVIDIA — 10× GeForce RTX 3090 (24 GB GDDR6X)
| idx | Model | PCIe BDF | VRAM | Link |
|-----|-------|----------|------|------|
| 0 | RTX 3090 | 21:00.0 | 24 GB | Gen4 x8 |
| 1 | RTX 3090 | 22:00.0 | 24 GB | Gen4 x8 |
| 2 | RTX 3090 | 41:00.0 | 24 GB | Gen4 x8 |
| 3 | RTX 3090 | 42:00.0 | 24 GB | Gen4 x8 |
| 4 | RTX 3090 | 81:00.0 | 24 GB | Gen4 x8 |
| 5 | RTX 3090 | 82:00.0 | 24 GB | Gen4 x8 |
| 6 | RTX 3090 | A1:00.0 | 24 GB | Gen4 x8 |
| 7 | RTX 3090 | A2:00.0 | 24 GB | Gen4 x8 |
| 8 | RTX 3090 | C1:00.0 | 24 GB | **Gen4 x16** |
| 9 | RTX 3090 | E3:00.0 | 24 GB | Gen4 x8 |

Driver version: **580.178.04**, CUDA **13.0**.

### AMD — 2× Radeon RX 7900 XTX (24 GB GDDR6)
| idx | Model | PCIe BDF | VRAM | Link |
|-----|-------|----------|------|------|
| AMD-0 | RX 7900 XTX | 03:00.0 | 24 GB | Gen4 x16 (via x8 bridge) |
| AMD-1 | RX 7900 XTX | 06:00.0 | 24 GB | Gen4 x16 (via x8 bridge) |

Architecture: **gfx1100** (RDNA 3), ROCm/HIP 7.1.

---

## 4. Device-Memory (VRAM) Bandwidth Results

This measures the speed of data movement *inside* each GPU between its compute cores and its
own video memory — the figure that fundamentally drives compute and LLM token throughput.
It is **unaffected** by PCIe link states.

| GPU | Memory type | Measured BW | Spec (approx) | % of peak |
|-----|-------------|-------------|---------------|-----------|
| 0 | GDDR6X | 891.9 GB/s | ~936 GB/s | ~95% |
| 1 | GDDR6X | 891.9 GB/s | ~936 GB/s | ~95% |
| 2 | GDDR6X | 891.6 GB/s | ~936 GB/s | ~95% |
| 3 | GDDR6X | 891.9 GB/s | ~936 GB/s | ~95% |
| 4 | GDDR6X | 892.2 GB/s | ~936 GB/s | ~95% |
| 5 | GDDR6X | 891.4 GB/s | ~936 GB/s | ~95% |
| 6 | GDDR6X | 892.2 GB/s | ~936 GB/s | ~95% |
| 7 | GDDR6X | 891.9 GB/s | ~936 GB/s | ~95% |
| 8 | GDDR6X | 891.6 GB/s | ~936 GB/s | ~95% |
| 9 | GDDR6X | 891.9 GB/s | ~936 GB/s | ~95% |
| AMD-0 | GDDR6 | 940.8 GB/s | ~960 GB/s | ~98% |
| AMD-1 | GDDR6 | 940.2 GB/s | ~960 GB/s | ~98% |

**Status:** All 12 GPUs deliver 95–98% of their rated memory bandwidth — fully healthy.

---

## 5. PCIe Host↔GPU Bandwidth Results (H2D / D2H)

This measures data transfer across the PCIe link between the host (system RAM/CPU) and each
GPU. **H2D** = Host-to-Device (system→GPU); **D2H** = Device-to-Host (GPU→system).

The measured numbers vs. the theoretical maximum for each physical link:

| Link | Theoretical (one-way) | Measured H2D | Measured D2H |
|------|----------------------|--------------|--------------|
| **Gen4 x8** | ~15.75 GB/s | ~13.4 GB/s | ~13.1 GB/s |
| **Gen4 x16** | ~31.5 GB/s | ~21.7 GB/s | ~15.1 GB/s |

### Per-GPU results (NVIDIA)
| GPU | Link | H2D | D2H |
|-----|------|-----|-----|
| 0 | Gen4 x8 | 13.40 | 13.15 |
| 1 | Gen4 x8 | 13.36 | 13.15 |
| 2 | Gen4 x8 | 13.37 | 12.98 |
| 3 | Gen4 x8 | 13.38 | 13.08 |
| 4 | Gen4 x8 | 13.40 | 13.16 |
| 5 | Gen4 x8 | 13.42 | 13.18 |
| 6 | Gen4 x8 | 13.46 | 13.21 |
| 7 | Gen4 x8 | 13.41 | 13.18 |
| 8 | **Gen4 x16** | **21.70** | **15.09** |
| 9 | Gen4 x8 | 13.47 | 13.22 |

### Per-GPU results (AMD)
| GPU | Link | H2D | D2H |
|-----|------|-----|-----|
| AMD-0 | Gen4 x16 (x8 bridge) | 14.18 | 14.31 |
| AMD-1 | Gen4 x16 (x8 bridge) | 14.16 | 14.31 |

### Notes on the PCIe results
- The x8 cards hit ~85% of the Gen4 x8 theoretical limit — normal, healthy efficiency.
- **GPU 8**, the single x16 card, achieves higher H2D (~21.7 GB/s) thanks to its wider link.
- The H2D > D2H asymmetry is a known characteristic of PCIe (device→host reads are less
  efficient than host→device writes); it is expected behavior, not a fault.
- All PCIe links were verified **negotiated at Gen4** during testing.

> **Known platform quirk (for reference):** All 10 NVIDIA GPUs exhibit an *idle*-time PCIe link
> down-train (Gen4→Gen1) after ~15–20 s of inactivity, which recovers instantly on load. This is
> NVIDIA-endpoint-specific (the AMD GPUs hold Gen4) and does not affect device-memory bandwidth
> or sustained throughput. It only adds a minor latency on a first request after an idle gap.

---

## 6. LLM Serving — Two Frameworks, Two Disjoint GPU Pools

The machine runs two inference stacks at the same time, each owning a separate set of cards:

| Stack | Framework | GPUs | Model | Endpoint |
|-------|-----------|------|-------|----------|
| Large-scale model | **Project Maya** (`build/maya-ds4`) | 10× RTX 3090 | DeepSeek-V4.1-Flash-1017, Maya-S quant | `:8080`, OpenAI + Anthropic compatible |
| Everyday models | **Ollama** 0.40.16 (bundled ROCm 7.2) | 2× RX 7900 XTX | Qwen3 coder and Qwen3.8 | `:11434` |

Because the two pools are physically disjoint (`CUDA_VISIBLE_DEVICES=-1` and
`ROCR_VISIBLE_DEVICES=0,1` for Ollama, `--gpu 0..9` for Maya), neither server can take VRAM from
the other and the DeepSeek session keeps running while the Ollama models are used. The only
shared resources are the EPYC cores and the 126 GB of system RAM.

### 6.1 Project Maya — DeepSeek-V4.1-Flash-1017 on the ten RTX 3090

The model is a 748 B-parameter mixture-of-experts transformer, served from Maya's own
**Maya-S** quant: six GGUF shards named IQ2_XXS, whose expert lanes the engine reports as a mix
of IQ2_S and IQ3_S types — i.e. roughly 2–3 bits per expert weight.

| Item | Detail |
|------|--------|
| Parameters | **748 B** total, 384 routed experts, **6 experts per token**, expert intermediate size 2304 |
| Quant on disk | Maya-S, 6 shards totalling **339.5 GB** (`340 GB` by the project's rounded figure) |
| Extra weights | **DSpark** drafter 7.97 GB, vision encoder 0.97 GB |
| Layers / width | 40 layers, hidden 5120, 64 attention heads × 512 head dim, q-LoRA rank 1280 |
| Context | configured `--max-context 262144`; the position table allows up to **1,048,576** (rope factor 16 on a 65,536 native window) |
| Vocabulary | 129,280 tokens |
| Launcher | `run-maya-ds4.sh` → `maya.py --config maya-ds4.json`, port 8080 |

Runtime flags that shape performance:

- `--tp 2` — tensor parallelism across the ten GPUs, i.e. five TP pairs each holding one slice
  of the expert weights.
- `--dspark on` — the 7.97 GB draft head proposes 5-token blocks that the main model verifies,
  which is what lifts decode from the ~49 tok/s baseline to ~72 tok/s.
- `--ablate .../DeepSeek-V4.1-Flash-refusal-direction.gguf --ablate-default off` — an ablation
  vector is loaded but switched off by default.
- Sampling temperature 1.0 / top-p 1.0, reasoning effort `high`, CUDA 13.5.

**Resident footprint.** A 340 GB model does not fit in 240 GB of VRAM, so Maya keeps the hot
experts on the cards and streams the rest from RAM and NVMe. Measured with a live session
loaded: **18.6–23.0 GB used on each of the ten 3090s** — about **196 GiB (211 GB)** of weights
and buffers resident in VRAM, leaving roughly 130 GB of colder experts to be pulled from the
126 GB system RAM and the NVMe volume (1.9 TB, 479 GB used). Expert-cache hit rate in the
sampled runs was **100.0%**.

**Measured throughput** (453 requests from `maya-ds4.log`, agent workload at ~32–36K-token
contexts, DSpark enabled):

| Metric | p50 | mean | p90 | max |
|--------|-----|------|-----|-----|
| Decode | **63.0 tok/s** | 65.8 | 85.4 | **104.3** |
| Prefill (new tokens) | **279 tok/s** | 355 | — | **2,135** |

- **DSpark calibration** (v1.1.65, 2026-11-17): **71.99 tok/s** with drafting vs **48.85
  tok/s** with it disabled — a **+47%** decode gain. Draft blocks are accepted at roughly
  **78%** (633 of 810 in one sampled request, 339 of 403 in another).
- **Per-workload bands** from the same calibration: chat 49.4–59.3 tok/s, code 49.6–79.4,
  tool calls 47.6–81.8.
- **Prompt reuse matters more than raw prefill:** a 33,788-token prompt was served as 32,401
  reused + 1,387 new tokens, so only the new part is processed at prefill speed and the rest
  arrives from cache.

### 6.2 Ollama — Qwen models on the two RX 7900 XTX

Ollama runs as a systemd service bound to the two AMD cards (BDF `03:00.0` and `06:00.0` — the
two `AMD-0`/`AMD-1` entries in the PCIe table), with Vulkan and the NVIDIA devices explicitly
disabled. It is tuned for this pair:

| Setting | Value | Effect |
|---------|-------|--------|
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` | halves the ~16 GB f16 KV cache across the pair to ~8 GB |
| `OLLAMA_CONTEXT_LENGTH` | `65536` | 64K context instead of the 256K model default — large prefill and VRAM savings |
| `OLLAMA_KEEP_ALIVE` | `24h` | models stay resident instead of unloading after 5 min |
| `OLLAMA_HOST` | `0.0.0.0:11434` | reachable from other hosts on the LAN |

| Model | Parameters | Quant | Size on disk | Native context | Capabilities |
|-------|-----------|-------|--------------|----------------|--------------|
| `qwen3-coder:30b-a3b-q8_0` | 30.5 B MoE (3 B active) | Q8_0 | 32 GB | 262,144 | completion, tool calls |
| `qwen3.8:27b` | 27.3 B (+ 460 M CLIP projector) | Q4_K_M | 17 GB | 262,144 | vision, tools, thinking (4 levels) |

Both models fit entirely on the pair individually, but not both at once: the coder model loads
49/49 layers as 15.8 GiB + 14.9 GiB of weights with 1.7 + 1.6 GiB of KV cache, and
`qwen3.8:27b` loads 66/66 layers as 7.3 + 8.1 GiB plus 682 MiB of CPU-mapped tensors. Since
together they exceed the 48 GB, Ollama swaps between them per request.

**Measured throughput** (today, 7.5K-token prompt, 150-token decode):

| Model | Prefill | Decode |
|-------|---------|--------|
| `qwen3-coder:30b-a3b-q8_0` | **4,661 tok/s** | **57.6 tok/s** |
| `qwen3.8:27b` | **911 tok/s** | **58.8 tok/s** |

The coder model is sparse (roughly 3 B active parameters), which is why its prefill is ~5× the
dense 27 B model's while decode is nearly identical; decode is memory-bandwidth-bound for both.
The service's own logs also recorded **33–35 tok/s** decode during earlier agent sessions, so
treat **35–60 tok/s** as the working range depending on context depth and model residency.

### 6.3 How the two stacks are used

`~/.ollama/config.json` maps each client to one of the small models — `qwen3-coder:30b-a3b-q8_0`
for the Droid agent and `qwen3.8:27b` for OpenCode — while Project Maya's larger DeepSeek engine
on the 3090s serves the long-context, tool-heavy work. The split keeps the 3090 pool exclusive to
the big model: the cards sit at ~19–23 GB per card under Maya, so nothing else can be scheduled
there without evicting experts and dropping the cache hit rate.

---

## 7. Summary

- **12 GPUs, all healthy:** 10× RTX 3090 + 2× RX 7900 XTX.
- **PCIe topology:** slots 7 and 8 carry RGEEK x16 → 2× x8 SlimSAS bifurcation kits
  (x8+x8 BIOS preset, 6-pin feed per breakout board), yielding four additional Gen4 x8 links.
- **Power:** five ATX 3.1 supplies totalling **5,250 W** nominal, all with native 12V-2x6.
- **Serving:** Project Maya runs DeepSeek-V4.1-Flash-1017 (748 B, Maya-S, 339.5 GB) on the ten
  3090s at ~63 tok/s median decode and ~72 tok/s with DSpark drafting; Ollama serves
  `qwen3-coder:30b-a3b-q8_0` and `qwen3.8:27b` on the two 7900 XTX at 58 tok/s decode each.
- **Device-memory bandwidth:** 891–892 GB/s (3090s), 940–941 GB/s (7900 XTX) — 95–98% of peak.
- **PCIe bandwidth:** x8 cards ~13.4 GB/s H2D / ~13.1 GB/s D2H; the x16 card ~21.7 / ~15.1 GB/s,
  all close to physical-link limits with expected write/read asymmetry.
- All links negotiated at Gen4; the platform is functioning as specified.
