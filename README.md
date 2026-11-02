# Ten RTX 3090s on a Workstation Board

### How I built a 240 GB local LLM box with a SlimSAS card, two SlimSAS adapters, and a lot of patience

---

Most "10× RTX 3090" builds you see online are either mining frames with ribbon risers or
datacenter chassis with backplanes. I wanted neither. I wanted one machine, in one box,
on a normal workstation motherboard, that could hold ten 24 GB cards and serve models on
Ollama, vLLM and Project Maya without catching fire.

It works. Here is the whole thing — the lane budget, the SlimSAS wiring, the firmware
settings, the 24-inch drum fan that keeps ten consumer cards cool, and the exact commands
I run on it today.

## At a glance

| | |
| --- | --- |
| **Motherboard** | ASUS WS X299 SAGE (LGA 2066, X299) — 7 × PCIe 3.0 x16 slots, dual onboard PLX PEX 8747 switches, 2 × U.2 |
| **CPU** | Intel Core i9-7900X — 10 cores / 20 threads, **44 PCIe 3.0 lanes** |
| **RAM** | 128 GB DDR4 (125 GiB usable), quad channel |
| **GPUs** | 10 × NVIDIA GeForce RTX 3090, 24 GB each — **240 GiB of VRAM** |
| **GPU fabric** | Dual onboard PEX 8747 + PLX 8749 SlimSAS card (4 × 8i out) + 2 × dual-8i → x16 adapters + 2 × SFF-8643 → x4 adapters |
| **Cooling** | 10 × open-air three-fan 3090s + a 24-inch industrial drum fan (8,900 CFM, 320 W) |
| **Storage** | 2 TB Samsung PM9A1 NVMe (XFS), ~750 GB free after models |
| **OS** | Fedora 44 Server, kernel 7.2.11 |
| **Driver / CUDA** | NVIDIA 580.178.04, CUDA toolkit 13.4 |
| **Memory pool** | ~365 GiB (240 GiB VRAM + 125 GiB RAM) |

The headline number is the one that matters for models like DeepSeek V4.1: **240 GB of
VRAM plus 125 GB of RAM is a ~365 GB pool**. That is enough to hold a 339.5 GB
DeepSeek-V4.1-Flash-1017 Maya-S pack in memory, with room left for a 256K-token KV cache.

---

## 1. The real constraint is lanes, not slots

Here is the trap. The WS X299 SAGE has seven physical x16 slots. Ten cards need ten
slots, so at first glance you are three short. But slots are not the problem — **lanes are**.

Ten cards at x16 is 160 lanes. The i9-7900X has **44**. That gap is not solved by slot
count — it is solved by PCIe switches, and a switch multiplies *ports*, not *bandwidth*:
every card behind one shares that switch's upstream link.

The WS X299 SAGE does half of that work for you. ASUS puts **two PLX PEX 8747 chips** on
the board, which is how 44 lanes turn into seven x16 slots at x16/x8/x8/x8/x8/x8/x8.
That covers six GPUs. The remaining cards are what the SlimSAS hardware is for.

So the design decision is made for you:

> **Plan for x8 per card, not x16.** Inference does not need x16. It needs enough
> bandwidth for the all-reduce traffic between GPUs — and it needs that bandwidth to be
> *predictable*.

That is exactly what a SlimSAS fabric gives you. SFF-8654 ("SlimSAS") is a thin, latching,
shielded PCIe cable:

- **SFF-8654 4i** carries **4 lanes** (PCIe x4);
- **SFF-8654 8i** carries **8 lanes** (PCIe x8).

A card at Gen3 x8 has ~7.9 GB/s per direction raw, ~7 GB/s usable. Ten of those is a
fine fabric for tensor-parallel inference and a terrible one for training. I only ever
serve models on this box, so the trade is correct.

Two other things SlimSAS buys me that ribbon risers do not: **latched connectors** (they
do not walk out of the socket when the box warms up and cools down) and **thin cables**
that route around ten 2.5-slot cards without blocking airflow.

A warning while you shop: SlimSAS is **PCIe-only**. It is not SATA, not SAS, and not U.2.
A "SlimSAS to 4× SATA" cable will not help you. You want SFF-8654 to SFF-8654, and an
adapter that terminates in a physical x16 slot on the GPU end.

---

## 2. The fabric as it actually enumerates

This is the part most build logs skip. Here is the real `lspci -tv` tree from my box,
with the GPUs mapped to their CUDA indices:

```
CPU  i9-7900X  (44 PCIe 3.0 lanes)
│
├── root port 16:00.0            (board slot 4)
│   └── PEX 8747 #1 (onboard)   17:00.0
│       ├── x8 → GPU 0   19:00.0
│       ├── x8 → GPU 1   1a:00.0
│       ├── x8 → GPU 2   1b:00.0
│       └── x8 → GPU 3   1c:00.0
│
├── root port 64:00.0            (board slot 6)
│   └── PEX 8747 #2 (onboard)   65:00.0
│       ├── x8 → GPU 4   67:00.0
│       ├── x8 → GPU 5   68:00.0
│       ├── x8 → PEX 8749 add-in card  69:00.0
│       │        ├── x8 → GPU 6   6d:00.0   ← dual-8i adapter
│       │        └── x8 → GPU 7   6e:00.0   ← dual-8i adapter
│       └── x8 → free
│
└── root port b2:00.0            (the two U.2 ports)
    ├── x4 → GPU 8   b3:00.0   ← SFF-8643 → x4 adapter
    └── x2 → GPU 9   b4:00.0   ← same path, trained narrow
```

Three PLX switches appear, and they play two completely different roles:

- **Two PEX 8747** (48-lane, 5-port) are **soldered to the motherboard**. ASUS advertises
  "dual PLX chips" on this board for exactly this reason: 44 CPU lanes cannot feed seven
  x16 slots on their own. The PLX pair turns them into seven slots at
  x16/x8/x8/x8/x8/x8/x8. Each chip takes one x16 upstream link and breaks it into four x8
  downstream ports.
- **One PEX 8749** (48-lane, 18-port, multi-root) is the **add-in SlimSAS card**: one x16
  slot in, four SFF-8654 8i ports out.

Read the tree bottom-up and the whole build resolves into four layers:

1. **Six cards in the board's own slots**, split across the two onboard 8747s — x8 each.
2. **The PLX 8749 card** in the seventh slot, fanning out to four 8i ports.
3. **Two dual-8i → x16 adapters** on that card, one 8i cable each → two more cards at x8.
4. **The board's two U.2 ports** (SFF-8639, cabled with SFF-8643) → passive
   SFF-8643 → PCIe x4 adapters → the last two cards, at x4 and x2.

That is 6 + 2 + 2 = **10 GPUs**, and all seven of the board's x16 slots are occupied —
six by GPUs, one by the SlimSAS card.

**Why do the last two show up on CPU root ports with no switch in between?** Because the
U.2 adapter is *passive*. It is copper and a slot, nothing else. `lspci` has no idea it
exists, so the GPU appears directly behind the root port, at whatever width the link
carries. On this board the U.2 ports are x4 — which is why one card trains at **x4**. The
second one training at **x2** is a cable or port symptom worth chasing, not a driver bug.

| GPU | PCI address | Link | Upstream | Notes |
| ---: | --- | --- | --- | --- |
| 0–3 | `19`, `1a`, `1b`, `1c` | Gen3 x8 | PEX 8747 #1 (onboard) | four of the board's slots |
| 4–5 | `67`, `68` | Gen3 x8 | PEX 8747 #2 (onboard) | two of the board's slots |
| 6–7 | `6d`, `6e` | Gen3 x8 | PEX 8749 add-in card | via the two dual-8i adapters |
| 8 | `b3` | Gen3 **x4** | U.2 → SFF-8643 | passive x4 adapter |
| 9 | `b4` | Gen3 **x2** | U.2 → SFF-8643 | same path, narrowed |

Every card is a Gen4-capable 3090 running at Gen3, because X299 is a Gen3 platform. That
is expected and fine.

---

## 3. Physical build

Ten 3090s is a mechanical project before it is an electrical one.

**Card choice: I run consumer cards, not blowers.** The usual advice is "blower 3090s or
water blocks". I ignored half of it — all ten cards in this build are open-air, three-fan
consumer 3090s. They are cheaper, easier to find, and quieter per card, but they dump heat
sideways into the card next to them, and in a ten-card stack "the next card" is already at
70 °C. What makes it work is not the cards. It is the air.

**Cooling: a 24-inch industrial drum fan.** Instead of ten water blocks, I put a
[LifePlus 24-inch industrial drum fan][fan] in front of the rack — **8,900 CFM** on three
speeds, 320 W, on casters so it can be rolled out of the way. It is aimed at the open side
of the stack so that room air is pushed *through* the gaps between the cards and the
exhaust leaves the rig instead of recirculating into the next card. That is the whole
trick: a three-fan card in still air cooks its neighbour, and the same card in 8,900 CFM
of forced air does not.

Two honest costs. The fan is loud — it is a warehouse fan, and at full speed you will hear
it through a wall — and it draws 320 W, which belongs in the power budget below. But it
turns "ten consumer cards will throttle themselves into uselessness" into "ten consumer
cards at a stable temperature", for a fraction of the price of ten water blocks.

**Spacing.** Even with forced air, do not stack three-fan cards flush against each other.
The open-air coolers are thicker than the board's two-slot spacing, and that is one more
reason the SlimSAS fabric matters: it lets me place cards where they physically fit instead
of where the slot spacing forces them. The two narrow-link cards hang off the U.2 ports
rather than the x16 slots, so they sit where the airflow is best.

**Cables.** I use 0.5 m SFF-8654 → SFF-8654 runs between the PLX 8749 card and the two
dual-8i adapters, and 60 cm SFF-8643 runs from the board's two U.2 ports to the x4
adapters. Keep every run short, do not kink them, do not zip-tie them into a hard right
angle, and reseat both ends after moving the machine. A cable that has been pulled
sideways shows up in `lspci` as a link that trained at x2 instead of x4 or x8. That is not
a subtle failure; it is a diagnostic gift.

Two build details that bite people. The dual-8i adapters have a **12 V 6-pin PCIe power
connector** because the x16 slot they create has to supply the card's 75 W — plug it in.
And the dual-8i adapter wants **two** 8i inputs to present a full x16 slot; I run one
cable per adapter, which is why those cards train at x8 and why two ports on the 8749 are
left empty.

**Power.** This is the part that has no clever trick.

- 10 cards capped at **280 W** each = **2.8 kW** just for GPUs (uncapped, ten of them
  can spike past 4 kW and knock cards off the bus)
- CPU + board + RAM + case fans: ~250–350 W
- The drum fan: **320 W**
- NVMe and NICs: trivial by comparison
- **Plan for ~3.4–3.5 kW continuous, with transient headroom above that**

At 240 V that is about 15 A — one decent circuit, but not a shared one. At 120 V it is
~29 A, which means two circuits, not one. Each 3090 wants 2–3 × 8-pin PCIe connectors; at
280 W, two 8-pin cables (150 W each) are genuinely enough, and that is 20–30 connectors
across the rig. Use PSU-native cables or a proper breakout board. Do not chain four cards
onto one molex-style daisy chain and then act surprised.

---

## 4. BIOS and boot parameters

Five minutes in firmware saves a week of debugging. The slot rules below all come from
ASUS's own [WS X299 SAGE user manual][manual] — read the expansion-slot table there before
you plan anything.

| Setting | Value | Why |
| --- | --- | --- |
| **Above 4G Decoding** | **Enabled** | Non-negotiable. Without it the firmware cannot assign 64-bit BARs to ten 24 GB cards, and you will see cards with no MMIO space. |
| **Resizable BAR** | Enabled | Lets the driver map the full BAR; helps with ten large devices. |
| **PCIe AER / DPC** | Leave to firmware, don't expect it | This board reports `_OSC: platform does not support [SHPCHotplug AER LTR DPC]` — the firmware will not contain a link error for you. |
| **ASPM** | Disabled in the OS | `pcie_aspm=off` on the kernel command line. ASPM L1 across ten endpoints behind switches caused link drops for me. |
| **IOMMU** | On | Default on this platform; keep it for isolation. |
| **Boot display** | The card you want as console | The 7900X has no iGPU, so one 3090 owns the framebuffer. Pick it deliberately. |

Kernel command line on my box:

```
pcie_aspm=off pcie_ports=native rd.driver.blacklist=nouveau,nova_core
```

`pcie_ports=native` matters because the firmware does not implement the PCIe port
services properly. Letting Linux drive the ports directly is more predictable than
trusting the board's ACPI.

---

## 5. Verify before you install a single model

Before installing anything, confirm the hardware is actually there and actually trained
at the width you designed. Two commands.

**Count and link width:**

```sh
nvidia-smi --query-gpu=index,pci.bus_id --format=csv,noheader |
while read -r i bus; do
  addr="${bus#00000000:}"
  printf 'gpu %-2s %-10s %s\n' "$i" "$addr" "$(lspci -s "$addr" -vv | grep -o 'LnkSta:.*')"
done
```

On a healthy build you want every line to say `Width x8` (or x16), never `x2`, and never
`downtrained`.

**Fabric map:**

```sh
nvidia-smi topo -m
```

The legend is the whole point:

- **PIX** — one PCIe bridge between the two GPUs. Cheapest all-reduce.
- **PXB** — several bridges, same host bridge. Fine.
- **PHB** — through the CPU's PCIe host bridge. Slower.
- **NODE** — through the host bridge and back. Slower still.
- **SYS** — across NUMA nodes. Do not put a tensor-parallel group here.

On my box the four cards behind switch #1 are `PIX` to each other, the two cards behind
switch #2 are `PIX` to each other and `PXB` to the 8749 pair, and the two narrow cards are
`PHB` to everything. **There is no `NV#` anywhere** — no NVLink bridges, no peer-to-peer.
Every collective goes over PCIe. That single fact drives every tuning decision later.

---

## 6. Ollama: the "does it even work" layer

Ollama is the first thing I install, because it answers a question no benchmark does:
*can this box load a model on ten GPUs and answer a prompt without falling over?*

```sh
curl -fsSL https://ollama.com/install.sh | sh
```

I run it as a systemd service with an explicit device pin. On a box where indices are PCI
bus order, pinning is not optional — otherwise a model lands on the two narrow-link cards
and everything is slower for no visible reason.

```ini
# /etc/systemd/system/ollama.service.d/20-gpus.conf
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_KEEP_ALIVE=24h"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_VULKAN=0"
Environment="CUDA_DEVICE_ORDER=PCI_BUS_ID"
Environment="CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7,8,9"
```

Then import a GGUF directly. For a multi-shard model, download with the HF CLI and point
a `Modelfile` at the first shard:

```sh
hf download <org>/<model>-GGUF --include "*.gguf" --local-dir ./models/<model>
```

```dockerfile
# Modelfile
FROM ./models/<model>/<model>-00001-of-00006.gguf
PARAMETER num_ctx 32768
PARAMETER temperature 0.7
SYSTEM """You are a careful technical assistant. Answer in English."""
```

```sh
ollama create ds4-flash -f Modelfile
ollama run ds4-flash
```

**Honest assessment:** Ollama is by far one of the easiest ways to serve an LLM model but 
Ollama's llama.cpp backend splits layers across all visible GPUs
in a pipeline, with no tensor parallelism and no expert parallelism. On a fabric like
this, throughput is set by the slowest link in the chain, and a prompt that has to cross
ten GPUs costs real time. It is excellent for single-user chat, quick model A/B tests, and
proving the hardware works. It is not how I serve the box.

---

## 7. vLLM: the serving layer

For real serving, vLLM. Version **0.31.0**, torch 2.13, on the 10 RTX 3090 cards.
It supports tensor parallelism which allows a single request to better utilize multiple GPUs.

The launch script, trimmed to what matters:

```bash
#!/usr/bin/env bash
set -euo pipefail
source .venv-vllm/bin/activate

export CUDA_DEVICE_ORDER=PCI_BUS_ID
export CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7,8,9

# system gcc is 16, which nvcc 13 refuses; skip the FlashInfer JIT sampler
export VLLM_USE_FLASHINFER_SAMPLER=0
export NVCC_APPEND_FLAGS="-allow-unsupported-compiler"

# opt-in low-latency NCCL for a single interactive user
[ "${LOW_LATENCY:-0}" = "1" ] && export NCCL_PROTO=LL

vllm serve models/Qwen3.8-Flash-Next-FP8 \
  --served-model-name Qwen3.8-Flash-Next-FP8 \
  --tensor-parallel-size 8 \
  --enable-expert-parallel \
  --gpu-memory-utilization 0.95 \
  --max-model-len 131072 \
  --max-num-seqs 16 \
  --enable-prefix-caching \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --reasoning-parser qwen3 \
  --host 0.0.0.0 --port 8000
```

> A note on indices: CUDA device order follows PCI bus order, so the list shifts every
> time you add, remove, or move a card. The `0–10` above is what the 10 RTX 3090 cards are on
> this ten-card layout.

What I learned tuning it:

- **Speculative decoding is the single biggest win.** Without MTP: 46 tok/s. With one
  draft token: 73. Two: 80. Three: ~86–92. Four: no better than three. Three is the
  setting.
- **`NCCL_PROTO=LL` trades prefill for decode.** Single-stream decode goes from ~82 to
  104–122 tok/s (+25–30%), but prefill drops ~42% and batched throughput ~25%. Great for
  one interactive user, wrong for a shared endpoint. Hence the `LOW_LATENCY` switch.
- **`--max-num-seqs` is a KV-cache decision, not a throughput decision.** At 32
  sequences the KV cache shrinks from 192K to 140K tokens and the model gets *slower*.
  Sixteen is the sweet spot for the FP8 build.
- **Bigger prefill chunks are not free.** Raising `--max-num-batched-tokens` to 4096
  is what knocked several cards off the bus. Prefill stayed capped around 2.1K tok/s —
  the limit was the collective, not the weights.

Measured on this box (8 × 3090, Gen3 x8, no P2P), 131K context:

| Workload | FP8 | W4A16 |
| --- | ---: | ---: |
| Short prompt, decode | 82 tok/s | 92 tok/s |
| 8K prompt, prefill / decode | 2,116 / 82 | 2,123 / 89 |
| 32K prompt, prefill / decode | 2,117 / 88 | 2,136 / 88 |
| 100K prompt, prefill / decode | 2,076 / 81 | 2,089 / 93 |
| 4 / 8 / 16 concurrent (aggregate) | 228 / 347 / 474 | 250 / 355 / 495 |
| 32 concurrent (aggregate) | — (KV too small) | 587 |
| Low-latency mode, single stream | 104–122 | 106–112 |

The W4A16 quant is 3–15% faster with ~3× the KV cache; FP8 is the safer quality choice.
If you only serve one person at a time, `LOW_LATENCY=1` is the best hundred tokens/s you
will get out of this rig.

---

## 8. Project Maya: the model that actually needs 240 GB

This is why the box exists.

**Project Maya** is an open-source engine for running very large mixtures-of-experts
models on your own hardware — GLM-5.3-Flash, DeepSeek-V4.1-Flash-1017 and
MiniMax-M3.1-Flash. It keeps the experts your conversation uses in VRAM, the next ones
in RAM, and the rest on NVMe, and moves them as you chat. It measures the machine and
tunes itself to it.

Setup is genuinely one command:

```sh
git clone https://github.com/mw00/project-maya.git
cd project-maya
./setup.sh
```

The setup checks the box, asks which GPUs to use, downloads the model, compiles the engine
once (10–30 minutes), builds the pack, and starts the dashboard at
`http://127.0.0.1:8080`.

My configuration is a 339.5 GB DeepSeek-V4.1-Flash-1017 **Maya-S** pack across all ten
cards. The parts that matter:

```json
{
  "args": [
    "--pack", "/home/aye/code/Maya-data/models/DeepSeek-V4.1-Flash-1017-GGUF/Maya-S/maya-ds4",
    "--max-context", "262144",
    "--tp", "2",
    "--dspark", "on"
  ],
  "gpu": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
  "port": 8080
}
```

- **All ten GPUs**, including the two narrow-link cards. Maya splits layers per GPU
  rather than forcing one collective group, so a x4 card still contributes useful VRAM
  instead of dragging everyone down.
- **`--dspark on`** — Maya's speculative drafter. The setup measures whether drafting
  actually helps on your hardware and keeps it only if it does. On this box it measured
  **51.98 tok/s with drafting vs 45.1 without**, so it stayed.
- **262K context**, which is what 240 GB of VRAM plus 125 GB of RAM buys you.

And as a user service, so it survives logout and restarts:

```ini
# ~/.config/systemd/user/maya.service
[Unit]
Description=Project Maya (DeepSeek V4.1 Flash · Maya-S)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/aye/code/project-maya
ExecStart=/home/aye/code/project-maya/maya.sh --config /home/aye/code/project-maya/maya-ds4.json --host 0.0.0.0
Restart=on-failure
RestartSec=30
TimeoutStartSec=3600
TimeoutStopSec=600

[Install]
WantedBy=default.target
```

```sh
systemctl --user enable --now maya
journalctl --user -fu maya
```

It exposes OpenAI- and Anthropic-compatible APIs on port 8080, so coding agents and
existing tools talk to it without changes.

### What ten cards actually deliver

From the live log on a warm engine:

| Prompt | Prefill | Decode |
| --- | ---: | ---: |
| 558 tokens (cold start) | 124 tok/s | 39.5 tok/s |
| 9,413 tokens | 589 tok/s | 68.0 tok/s |
| 13,327 tokens | 552 tok/s | 79.4 tok/s |
| 27,827 tokens | 1,084 tok/s | 62.1 tok/s |

**60–80 tok/s decode on a 748 B-parameter model, at 256K context, on ten used 3090s.**
The first, cold request is slower (39.5 tok/s while the expert cache fills); after that
the engine holds 60–80 for the rest of the session. The expert cache hit rate was 100%
throughout — every expert ended up resident in VRAM, which is the best case this hardware
can do.

For comparison: the same box does 82–93 tok/s on Qwen3.8-Flash-Next in vLLM — roughly
140 GiB of FP8 weights, so a ~150 B-class model. The difference is not the fabric; it is
that a 748 B MoE has far more weights to touch per token. That is the honest cost of
running a frontier-sized model at home.

---

## 9. Reference: bill of materials

| Part | What I used | Note |
| --- | --- | --- |
| Motherboard | ASUS WS X299 SAGE | 7 × x16 slots, dual onboard PEX 8747, 2 × U.2 |
| Motherboard manual | [WS X299 SAGE user manual (PDF, 13 MB)][manual] | [ASUS support page][asus-ws-x299] if that signed link expires |
| CPU | Intel Core i9-7900X | 44 PCIe 3.0 lanes. A 28-lane -X chip will not carry this build. |
| RAM | 128 GB DDR4, quad channel | 125 GiB usable; matters for Maya's RAM tier |
| GPUs | 10 × RTX 3090 24 GB | open-air three-fan consumer cards, not blowers |
| Cooling | [LifePlus 24-inch industrial drum fan][fan] | 8,900 CFM, 320 W; pushes room air through the stack |
| **SlimSAS host card** | [PLX8749 PCIe 3.0 x16 → 4 × SFF-8654 8i adapter](https://www.amazon.com/dp/B0GF1CK356) | one x16 slot becomes four x8 ports |
| **SlimSAS adapters** | [Dual SlimSAS 8i → PCIe x16 slot adapter, Gen4, 6-pin power](https://www.amazon.com/dp/B0D5HRP311) × 2 | takes two 8i inputs, presents one x16 slot |
| **SlimSAS cables** | [10Gtek SFF-8654 ↔ SFF-8654, PCIe 4.0, 0.5 m](https://www.amazon.com/dp/B08FCBN8TD) | one run per adapter in my build |
| **U.2 cables** | [HighPoint SFF-8643 ↔ SFF-8643 NVMe cable, 60 cm](https://www.amazon.com/dp/B08H3ZCMT8) × 2 | from the board's U.2 ports |
| **U.2 adapters** | [SFF-8643 → PCIe x4 slot adapter](https://www.amazon.com/dp/B0D5QZSPZF) × 2 | carries the two narrow-link cards |
| Storage | 2 TB NVMe (XFS) | ~750 GB free with the DeepSeek pack installed |
| OS | Fedora 44 Server | kernel 7.2.11, `pcie_aspm=off pcie_ports=native` |
| Driver | NVIDIA 580.178.04 | CUDA toolkit 13.4 |
| Power | ~3.5 kW continuous capacity | 240 V preferred; 120 V needs two circuits |

*Links point at the exact fabric parts I bought — the five items that turn seven slots and
two U.2 ports into ten GPU slots. The motherboard, CPU, RAM, GPUs, PSU and chassis are the
rest of the build.*

### The verification script I run after every move

```sh
#!/bin/sh
# what the box should look like
echo "GPUs:      $(nvidia-smi -L | wc -l)"
nvidia-smi --query-gpu=index,pci.bus_id --format=csv,noheader |
while read -r i bus; do
  addr="${bus#00000000:}"
  printf 'gpu %-2s %-10s %s\n' "$i" "$addr" "$(lspci -s "$addr" -vv | grep -o 'LnkSta:.*')"
done
echo
nvidia-smi --query-gpu=index,power.limit,power.draw,temperature.memory --format=csv,noheader
echo
nvidia-smi topo -m
```

If every line says `Width x8`, every power limit says `280.00 W`, and every memory
junction is under control, the fabric survived the move. If one card dropped to x2,
reseat its cable before you blame the driver.

[manual]: https://dlcdnta.asus.com/pub/ASUS/mb/Socket2066/WS_X299_SAGE/Manual/E16044_WS_X299_SAGE_UM_V4_WEB.pdf?model=WS%20X299%20SAGE&Signature=wBbp5pRtjJNp5fH1f8iGaOcvUWJrv4CMvGVJiftC3Kt9KdkvNH39CRRNbELwFXSxDkEP6Em97li5saf7HGLTFRLt~f1L-N3m0uxbNZKT1qCZEOlNg36-TM0MkD6HvGhYcLbxfNGGT-b-MakInXPnK-9cBOj4dTcTGJShP1p6DCY5XFwOAf-MM61akZsVLPGDZ9kJvxhHVhU1PTzefhzWW0YQWpbrV38hPKmKpGRIb1b4vMFSutgGaNfiG0g7dYuVkucWxSbXEtUkUGOBX5YWYnPvEI9wtkDHGmN3VIqiuec4cLo7hRwGM3FV3dgdD-Un0Pj10SNCbgJec7KhLb9W-g__&Expires=1793578575&Key-Pair-Id=K2ITB7O97XKKCX
[asus-ws-x299]: https://www.asus.com/us/motherboards-components/motherboards/workstation/ws-x299-sage/
[fan]: https://www.amazon.com/dp/B0DXFCDTG1

<!--
BEFORE PUBLISHING — two things only you can fill in:
1. PSU make/model/wattage and how many units (the article says "size for ~3.5 kW").
2. Chassis / frame. (The article now says the cards are open-air three-fan consumer 3090s,
   but not which model — add it if you want the build to be fully reproducible.)
WATCH OUT: the manual link ([manual], defined just above) is an ASUS CloudFront signed
URL. It expired on 2026-11-02 00:16 UTC, so it will be dead by the time this is published.
Either grab a fresh signed link from the ASUS support page ([asus-ws-x299]) or host the
PDF yourself and point [manual] at your copy.
Also confirm two assumptions in the parts table: that the two narrow cards each use an
SFF-8643 → x4 adapter (so × 2 of each of those last two parts), and that the x2 card is
the second U.2 run rather than a cable that has gone bad.
Optionally: what the build cost, and a photo of the stack.
-->
