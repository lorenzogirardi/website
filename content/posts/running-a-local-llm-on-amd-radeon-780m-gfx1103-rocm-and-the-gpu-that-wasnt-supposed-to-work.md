---
title: Running a Local LLM on AMD Radeon 780M — gfx1103, ROCm, and the GPU That
  Wasn't Supposed to Work
date: 2026-06-07
draft: false
description: "A full walkthrough of setting up Gemma 4 12B QAT on a Lenovo M75Q
  Gen5 mini-PC with AMD Radeon 780M iGPU. ROCm kernel mapping, GTT memory, GPU
  clock optimization, and a surprising benchmark result: CPU beats GPU on
  generation speed."
tags:
  - llm
  - amd
  - rocm
  - vulkan
  - ollama
  - homelab
  - radeon
  - inference
  - monitoring
featuredImage: /images/Screenshot 2026-06-07 at 13.10.19.png
images:
  - "/images/Screenshot 2026-06-07 at 13.10.19.png"
---
### Table of Contents

- The Machine
- The Problem: gfx1103 Doesn't Exist
- GTT Memory — 24 GB for Free
- The ROCm Stack
- Getting GPU Inference Working
- Optimizing: The Hidden GPU Clock Problem
- Benchmarks — Every Configuration Tested
- The Surprising Finding: CPU Beats GPU on Generation
- The Real Bottleneck: Single-Channel RAM
- The Breakthrough: MoE on CPU
- Monitoring with Collectd and Grafana
- What the Dashboard Actually Shows
- Lessons Learned

---

I wanted a local AI box. Not a cloud API with latency and per-token billing. Not a GPU workstation that sounds like a jet engine. A quiet mini-PC that runs a capable model at home, on my desk, forever, for free.

After some research I picked up a **Lenovo ThinkCentre M75Q Gen5** — 8-core AMD Ryzen 7 PRO 8700GE APU, 32 GB DDR5-5600, Radeon 780M iGPU. Around 400€. Fits in a hand. Silent.

What followed was three days of ROCm archaeology, kernel parameter tuning, and benchmarking everything I could think of. This is the full account.

---

## The Machine

The M75Q Gen5 is an APU (Accelerated Processing Unit) — CPU and GPU on the same die, sharing system RAM. No discrete GPU. No separate VRAM. Just 32 GB DDR5-5600 doing triple duty as CPU memory, GPU memory, and swap space for inference workloads.

```
Hardware
├── CPU: AMD Ryzen 7 PRO 8700GE — 8 cores / 16 threads, RDNA3
├── iGPU: AMD Radeon 780M — 12 Compute Units, gfx1103
├── RAM: 32 GB DDR5-5200 — SINGLE module (only one of two channels populated!)
├── Storage: 512 GB NVMe → LVM 466 GB at /
└── OS: Ubuntu 26.04 LTS, kernel 7.0.0-22
```

The GPU has no dedicated VRAM. Instead, it uses two memory pools:

- **UMA VRAM**: a slice of RAM carved out by the BIOS (typically 512 MB – 2 GB)
- **GTT (Graphics Translation Table)**: the GPU's window into the rest of system RAM, managed by the `amdgpu` kernel driver

For LLM inference, GTT is what matters. With the right kernel parameter, you can expose 24 GB of system RAM to the GPU. That's enough to load Gemma 4 12B Q4_0 (7.6 GB) with room to spare.

---

## The Problem: gfx1103 Doesn't Exist

The Radeon 780M uses GPU architecture version **gfx1103**. In AMD's naming scheme, that's `11.0.3` — RDNA3, Hawk Point generation.

When you install Ollama and try to run a model, this happens:

```
WARN dropping ROCm device — no rocblas support for gfx target
device=ROCm0 gfx_target=gfx1103
supported="map[gfx1030:true gfx1100:true gfx1101:true gfx1102:true
           gfx1150:true gfx1151:true gfx1200:true gfx1201:true
           gfx908:true gfx90a:true gfx942:true gfx950:true]"
hint="set HSA_OVERRIDE_GFX_VERSION to map to a supported target"
```

gfx1103 is not in Ollama's supported list. The GPU is rejected. Inference falls back to CPU.

The hint tells you exactly what to do. `HSA_OVERRIDE_GFX_VERSION` is an AMD ROCm environment variable that tells the hardware identification layer to present the GPU as a different version. Set it to `11.0.2` and the GPU announces itself as gfx1102 — which IS in the supported list.

```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.2
```

That's it. One environment variable. GPU inference works.

The reason `11.0.2` (not `11.0.3`) is the mapping: Ollama's bundled ROCm 7.2 includes highly optimized Tensile kernel libraries for gfx1102 (RDNA3 chips like the RX 6650 XT, a common gaming card with large community usage). Those kernels run on gfx1103 hardware because the architectures are close relatives — same RDNA3 generation, similar CU layout.

{{< mermaid >}}
flowchart TD
    A[Ollama starts] --> B{GPU detection}
    B -->|gfx1103 found| C[Check supported list]
    C -->|gfx1103 NOT in list| D[Drop GPU → CPU fallback]
    C -->|HSA_OVERRIDE=11.0.2| E[GPU presents as gfx1102]
    E --> F[gfx1102 IS in list ✓]
    F --> G[Load ROCm 7.2 Tensile kernels for gfx1102]
    G --> H[GPU inference active]
    D --> I[CPU-only inference ~5 tok/s]
    H --> I2[GPU inference ~4.5 tok/s gen / 54 tok/s prefill]
{{< /mermaid >}}

---

## GTT Memory — 24 GB for Free

Before ROCm can use GPU memory, the driver needs to know how much GTT to expose. By default, `amdgpu` limits GTT to a fraction of system RAM. For a 32 GB machine that might be 4–8 GB — not enough for a 7.6 GB model.

The fix is a kernel boot parameter:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="amdgpu.gttsize=24576"
# 24576 MB = 24 GB GTT pool
```

After `sudo update-grub && sudo reboot`, verify:

```bash
cat /sys/class/drm/card0/device/mem_info_gtt_total
# 25769803776 = 24 GB ✓

cat /sys/class/drm/card0/device/mem_info_gtt_used
# 7845441536 during inference = ~7.8 GB used for model
```

The model loads entirely into GTT. The GPU can access it directly via DMA without copying.

{{< mermaid >}}
graph LR
    subgraph DDR5["DDR5-5600 32 GB (89.6 GB/s)"]
        UMA["UMA VRAM\n~512 MB\n(BIOS carved)"]
        GTT["GTT Pool\n24 GB\n(amdgpu.gttsize=24576)"]
        SYSRAM["System RAM\n~7.5 GB\nOS + processes"]
    end

    subgraph GPU["Radeon 780M — 12 CU gfx1103"]
        ROCm["ROCm runtime"]
    end

    MODEL["/Storage/models/\ngemma4-qat:12b\n7.6 GB GGUF"] -->|loaded into| GTT
    ROCm <-->|DMA access| GTT
    ROCm <-->|direct| UMA
{{< /mermaid >}}

---

## The ROCm Stack

Here's what's actually running on the machine after setup:

{{< mermaid >}}
graph TB
    CLIENT["Client HTTP / OpenAI-compat\n:11434"]
    OLLAMA["Ollama 0.30.6 official\n/usr/local/bin/ollama"]
    ROCM["ROCm 7.2 bundled\n/usr/local/lib/ollama/rocm_v7_2/"]
    ROCBLAS["librocblas.so.5.2\n1792 Tensile kernels\ngfx1102 optimized"]
    HIP["libggml-hip.so\nGGML HIP backend"]
    GPU["AMD Radeon 780M\ngfx1103 → gfx1102 via HSA_OVERRIDE"]
    GTT["GTT Pool 24 GB\nModel: gemma4-qat:12b 7.6 GB"]
    COLLECTD["collectd exec\ngpu_metrics.sh\ncollect_ollama_metrics.sh"]
    DEVA["deva 192.168.1.28\nInfluxDB → Grafana"]

    CLIENT --> OLLAMA
    OLLAMA --> ROCM
    ROCM --> ROCBLAS
    ROCM --> HIP
    HIP --> GPU
    GPU <--> GTT
    COLLECTD -->|UDP :25826| DEVA
    OLLAMA -.->|metrics via journald| COLLECTD
    GPU -.->|sysfs hwmon| COLLECTD
{{< /mermaid >}}

Key environment variables in `/etc/systemd/system/ollama.service`:

```ini
[Service]
User=lgirardi
Environment=HSA_OVERRIDE_GFX_VERSION=11.0.2
Environment=ROC_ENABLE_PRE_VEGA=1
Environment=AMDGPU_TARGETS=gfx1103
Environment=OLLAMA_MODELS=/Storage/models
Environment=OLLAMA_IGPU_ENABLE=1
Environment=OLLAMA_FLASH_ATTENTION=1
Environment=OLLAMA_NUM_PARALLEL=1
Environment=LD_LIBRARY_PATH=/usr/local/lib/ollama
ExecStartPre=+/bin/sh -c 'echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level'
ExecStart=/usr/local/bin/ollama serve
ExecStopPost=+/bin/sh -c 'echo auto > /sys/class/drm/card0/device/power_dpm_force_performance_level'
```

Two things worth calling out:

**`OLLAMA_IGPU_ENABLE=1`**: Ollama drops integrated GPUs by default. This tells it to keep them.

**`ExecStartPre`**: Forces the GPU to maximum clock speed (2700 MHz) at service start. This turns out to be critical — see the next section.

---

## Optimizing: The Hidden GPU Clock Problem

After getting GPU inference working, I benchmarked it and got 4.4 tok/s generation speed. Then I ran it on CPU and got 5.3 tok/s. The CPU was faster.

Something was wrong.

I checked the GPU clock:

```bash
cat /sys/class/drm/card0/device/pp_dpm_sclk
# 0: 800Mhz *      ← running at 800 MHz!
# 1: 1100Mhz
# 2: 2700Mhz
```

The GPU was running at **800 MHz** — the lowest power state. The `amdgpu` driver defaults to `auto` power management, and with an iGPU doing inference (which doesn't look like a gaming workload to the driver), it chose to downclock.

Force it to maximum:

```bash
echo high | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
```

After the change:

```bash
cat /sys/class/drm/card0/device/pp_dpm_sclk
# 0: 2700Mhz
# 1: 1100Mhz
# 2: 2700Mhz *     ← now at 2700 MHz
```

This is why the `ExecStartPre` line in the service unit matters. Without it, every service restart returns the GPU to 800 MHz.

---

## Benchmarks — Every Configuration Tested

I tested every meaningful combination I could think of:

- **Ollama + ROCm** (the main path, gfx1102 mapping)
- **llama.cpp + Vulkan** (Mesa RADV, native gfx1103 support)
- **llama.cpp + HIP native gfx1103** (compiled from source with `GPU_TARGETS=gfx1103`)
- **llama.cpp + CPU only** (8 threads, 16 threads)
- **llama.cpp split** (24 GPU layers + 24 CPU layers)

Model: Gemma 4 12B QAT Q4_0 (`google/gemma-4-12B-it-qat-q4_0-gguf`), 7.6 GB. Prompt: "write numbers 1 to 30", 80 tokens generated. All GPU tests at 2700 MHz.

| Setup | Prefill (tok/s) | Generation (tok/s) |
|---|---|---|
| **Ollama + ROCm 7.2 gfx1102 + FA** | **54** | 4.5 |
| Vulkan RADV PHOENIX (Mesa 26.0.3) | 39 | 4.7 |
| llama.cpp HIP native gfx1103 (ROCm 7.1) | 22 | 4.5 |
| llama.cpp split 24GPU + 24CPU | 18.5 | 4.7 |
| **llama.cpp CPU-only 8 threads** | 14 | **5.3** |
| llama.cpp CPU-only 16 threads | 16 | 5.1 |
| Ollama + ROCm gfx1102 (800 MHz, no fix) | 37 | 4.5 |

{{< mermaid >}}
xychart-beta
    title "Prefill Speed (tok/s) — higher is better"
    x-axis ["Ollama ROCm 2700MHz+FA", "Vulkan RADV", "llama.cpp HIP native", "Split 24/24", "CPU 8t", "CPU 16t", "Ollama ROCm 800MHz"]
    y-axis "tok/s" 0 --> 60
    bar [54, 39, 22, 18.5, 14, 16, 37]
{{< /mermaid >}}

{{< mermaid >}}
xychart-beta
    title "Generation Speed (tok/s) — higher is better"
    x-axis ["Ollama ROCm 2700MHz+FA", "Vulkan RADV", "llama.cpp HIP native", "Split 24/24", "CPU 8t", "CPU 16t", "Ollama ROCm 800MHz"]
    y-axis "tok/s" 0 --> 6
    bar [4.5, 4.7, 4.5, 4.7, 5.3, 5.1, 4.5]
{{< /mermaid >}}

---

## The Surprising Finding: CPU Beats GPU on Generation

Look at the generation column. **CPU (5.3 tok/s) beats every GPU configuration (4.5–4.7 tok/s).**

This is not intuitive. GPUs are supposed to be faster. For LLM inference on a discrete GPU with fast GDDR6 or HBM, they are. But on an APU with unified memory, the math is different.

Here's why:

{{< mermaid >}}
graph LR
    subgraph CPU["CPU Path"]
        CPU_CORE["Ryzen 7 PRO 8700GE\n8 cores / AVX-512 BF16"] -->|direct DDR5 access\nno overhead| DDR5_CPU["DDR5 ~31 GB/s\n(single-channel 5200)"]
    end

    subgraph GPU["GPU Path (GTT)"]
        GPU_CU["Radeon 780M\n12 CUs"] -->|UMA / GTT DMA\n+ ROCm dispatch overhead| DDR5_GPU["DDR5 ~29 GB/s\n(single-channel + overhead)"]
    end
{{< /mermaid >}}

Token generation is **memory bandwidth bound**, not compute bound. Each forward pass reads the full ~7 GB of model weights from RAM. The CPU does this with direct DDR5 access and AVX-512 BF16 SIMD instructions. The iGPU has to go through the GTT DMA path, which adds latency and reduces effective bandwidth.

The result: CPU achieves ~5.3 tok/s, GPU achieves ~4.5 tok/s. The GPU's overhead eats its raw bandwidth advantage. (Both numbers are low for a reason I hadn't found yet — see *The Real Bottleneck* below.)

**Prefill is the opposite.** Processing an input prompt (the "prefill" phase) is compute-bound — you're doing batch matrix multiply across all input tokens simultaneously. Here the GPU's 12 CUs at 2700 MHz dominate: 54 tok/s vs 14 tok/s on CPU. The clock speed matters too: the same GPU at 800 MHz only manages 37 tok/s.

{{< mermaid >}}
quadrantChart
    title "Prefill vs Generation — Where Each Backend Wins"
    x-axis "Low Generation Speed" --> "High Generation Speed"
    y-axis "Low Prefill Speed" --> "High Prefill Speed"
    quadrant-1 "Best for RAG / long prompts"
    quadrant-2 "Best overall"
    quadrant-3 "Avoid"
    quadrant-4 "Best for chat"
    Ollama ROCm 2700MHz: [0.43, 0.97]
    Vulkan RADV: [0.49, 0.67]
    llama.cpp native: [0.43, 0.36]
    CPU 8 threads: [0.96, 0.18]
    Split 24-24: [0.49, 0.30]
{{< /mermaid >}}

**Practical strategy:**
- **Chat (short prompts, long responses):** CPU-only `llama.cpp` — smoother output at 5.3 tok/s
- **RAG / document analysis (long prompts):** Ollama GPU — 54 tok/s prefill makes a real difference
- **Current setup:** Ollama GPU (best prefill, slightly slower generation but acceptable)

---

## Why llama.cpp Native gfx1103 Lost

I compiled llama.cpp from source with native gfx1103 HIP support:

```bash
cmake -B build -DGGML_HIP=ON -DGPU_TARGETS=gfx1103 -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc) --target llama-server
```

This produces a 73 MB `libggml-hip.so` with GGML HIP kernels compiled natively for gfx1103. No HSA_OVERRIDE needed — the HIP runtime accepts the GPU directly.

Yet it was slower than Ollama's gfx1102 mapping (22 vs 54 tok/s prefill).

The reason: **ROCm version**. Ubuntu 26.04 ships ROCm 7.1 in apt. Ollama bundles ROCm 7.2 in its own library directory (`/usr/local/lib/ollama/rocm_v7_2/`). The 7.2 gfx1102 Tensile kernels are more aggressively optimized than the 7.1 gfx1103 native kernels. A newer version of a close-relative kernel beats an older version of a native one.

The system ROCm 7.1 does include native gfx1103 Tensile kernels:
```bash
ls /usr/lib/x86_64-linux-gnu/rocblas/5.1.0/library/ | grep gfx1103
# Kernels.so-000-gfx1103.hsaco  ← exists!
```

But they're slower. For now, the mapping approach wins.

---

## Why Vulkan Lost

Mesa RADV supports gfx1103 natively. It's a well-maintained Vulkan driver for AMD hardware, and after installing `mesa-vulkan-drivers`, Ollama automatically detected it:

```
library=Vulkan compute=0.0 name=Vulkan0
description="AMD Radeon 780M Graphics (RADV PHOENIX)"
type=iGPU total="24.5 GiB"
```

Note that Vulkan sees 24.5 GiB (essentially the full GTT), while ROCm sees only 15 GiB. More memory visibility didn't translate to better performance: 39 tok/s prefill vs 54 tok/s for ROCm.

GGML's Vulkan shader kernels are less optimized for batch GEMM operations than the Tensile ROCm path. Vulkan is a viable fallback if ROCm refuses to work, but it's not the fastest path on this hardware.

---

## The Real Bottleneck: Single-Channel RAM

The benchmarks above kept hitting a wall: 4.5 tok/s generation, no matter the backend. I'd written it off as "memory bandwidth bound, that's just the hardware." Then a benchmark from someone else nagged at me — a 780M running a 7B model at **19.5 tok/s**. Same chip. How?

The math gives it away. Generation reads the model weights from RAM once per token:

- 7B Q4 ≈ 4 GB/token. At 19.5 tok/s → **78 GB/s** effective.
- My 12B Q4 ≈ 7 GB/token. At 4.5 tok/s → **31 GB/s** effective.

A 2.5× gap. That's not "GPU overhead" — that's a different memory subsystem. So I actually looked:

```bash
sudo dmidecode -t 17 | grep -E "Size|Locator|Speed"
# Size: 32 GB        Bank Locator: P0 CHANNEL A   Speed: 5200 MT/s
# Size: No Module Installed   Bank Locator: P0 CHANNEL B
```

**One 32 GB module. Channel B empty. Single-channel.**

I'd assumed dual-channel the whole time (the spec sheet says "up to," and a single big DIMM looks identical to two small ones in `free -h`). Single-channel DDR5-5200 is one 64-bit bus at 5200 MT/s ≈ **41.6 GB/s theoretical → ~31 GB/s real** — exactly my number. The other guy's 780M had two sticks: double the bus, ~78 GB/s, 19.5 tok/s on a 7B.

**The bottleneck was never the GPU, ROCm, or gfx1103. It was a missing stick of RAM.** Every hour spent on kernel mappings and Tensile libraries was optimizing the wrong layer. The M75q Gen5 has two SO-DIMM slots; populating Channel B with a matching module would roughly double generation speed across the board.

> Lesson: before you blame the exotic part (the iGPU, the kernel), verify the boring part (how many RAM sticks are actually in the machine).

---

## The Breakthrough: MoE on CPU

Single-channel RAM caps a **dense** model: every token must read every weight. But there's an architecture that sidesteps this — **Mixture of Experts (MoE)**. A MoE model has many "expert" sub-networks but activates only a few per token. Gemma 4 26B-A4B has 26B total parameters but only ~4B *active* per token — so each token reads ~2.5 GB instead of the full file.

Less data per token = less bandwidth needed. And on this single-channel box, bandwidth is the whole game. The catch: 17 GB total doesn't fit ROCm's 15 GB cap, so it **crashes on GPU**. But the CPU has no such cap, and direct DDR5 access. So I ran it on CPU:

```bash
# gemma4-26b-cpu = Modelfile FROM gemma4:26b-a4b-it-q4_K_M + PARAMETER num_gpu 0
curl -s http://localhost:11434/api/chat -d '{
  "model":"gemma4-26b-cpu",
  "messages":[{"role":"user","content":"Write Fibonacci with memoization"}],
  "think":false, "stream":false
}'
```

**13–14 tok/s.** Three times the dense 12B on GPU — from a *bigger, smarter* model.

| Model | Type | Engine | Generation | Output |
|---|---|---|---|---|
| Gemma 4 12B QAT | dense | GPU gfx1102 | 4.5 tok/s | ✅ |
| Gemma 4 12B QAT | dense | CPU | 5.5 tok/s | ✅ |
| Gemma 4 E4B (eff. 4B) | MatFormer | CPU | 12.3 tok/s | ✅ |
| **Gemma 4 26B-A4B** | **MoE** | **CPU** | **13–14 tok/s** | ✅ |

{{< mermaid >}}
graph LR
    subgraph DENSE["Dense 12B — reads everything"]
        D1["7 GB / token"] -->|31 GB/s single-channel| D2["4.5 tok/s"]
    end
    subgraph MOE["MoE 26B-A4B — reads active experts only"]
        M1["~2.5 GB / token\n4B of 26B active"] -->|31 GB/s single-channel| M2["13-14 tok/s"]
    end
{{< /mermaid >}}

The sparse model turns the bandwidth bottleneck into a non-issue: by reading a third of the data per token, it gets a 3× speedup on the *same* memory bus — while carrying the knowledge of a 26B model. **On a bandwidth-starved APU, sparse architecture beats dense optimization.**

### One trap: Gemma 4 is a reasoning model

First runs returned empty responses with `done_reason: length`. The models weren't broken — Gemma 4 *reasons* before answering, and with a low `num_predict` it spent the entire token budget thinking, leaving nothing for the answer. The fix is `"think": false` for direct replies (or a generous token budget if you want the reasoning):

```bash
# empty content, all budget burned on hidden reasoning:
curl ... -d '{"...","options":{"num_predict":80}}'        # → ""
# direct answer:
curl ... -d '{"...","think":false}'                        # → "Fib(n) = ..."
```

---

## Monitoring with Collectd and Grafana

The machine ships metrics to my central monitoring stack via collectd UDP to `deva` (192.168.1.28:25826 → InfluxDB → Grafana).

{{< mermaid >}}
sequenceDiagram
    participant GPU as GPU sysfs/hwmon
    participant Script as gpu_metrics.sh
    participant Ollama as Ollama API/journald
    participant OScript as collect_ollama_metrics.sh
    participant Collectd as collectd exec plugin
    participant InfluxDB as InfluxDB (deva)
    participant Grafana as Grafana

    loop Every 60 seconds
        Script->>GPU: read mem_info_gtt_used/total
        Script->>GPU: read hwmon temp1_input (amdgpu hwmon)
        Script->>Collectd: PUTVAL gpu/bytes-gtt_used
        Script->>Collectd: PUTVAL gpu/bytes-vram_used
        Script->>Collectd: PUTVAL gpu/temperature-gpu

        OScript->>Ollama: GET /api/ps (model loaded?)
        OScript->>Ollama: journalctl parse (request count, duration)
        OScript->>Collectd: PUTVAL ollama/gauge-models_loaded
        OScript->>Collectd: PUTVAL ollama/counter-requests

        Collectd->>InfluxDB: UDP :25826
        Grafana->>InfluxDB: query
    end
{{< /mermaid >}}

One gotcha: the GPU temperature sensor. The machine has multiple hwmon entries — NVMe thermal sensor on `hwmon0`, CPU on `hwmon1`, and `amdgpu` on `hwmon3`. A naive `hwmon*/temp1_input | head -1` grabs the NVMe (always ~45°C), not the GPU. The correct approach is to look up the hwmon entry by name:

```bash
AMDGPU_HWMON=$(grep -rl "^amdgpu$" /sys/class/hwmon/hwmon*/name | head -1 | xargs dirname)
GPU_TEMP_RAW=$(cat "$AMDGPU_HWMON/temp1_input")
GPU_TEMP_C=$(awk "BEGIN {printf \"%.1f\", $GPU_TEMP_RAW/1000}")
```

Another gotcha: `bc` (the calculator) is not installed on Ubuntu 26.04. Use `awk` for arithmetic instead.

The Ollama metrics situation is more awkward. Ollama 0.30.6 returns `404` for `GET /metrics` regardless of the `OLLAMA_METRICS=true` environment variable — the endpoint isn't implemented in this version. The workaround is to parse the GIN access logs from journald:

```bash
LINES=$(journalctl -u ollama.service --since "65 seconds ago" --no-pager -o cat | \
  grep -E '\[GIN\].*\| 200 \|.*POST.*(api/generate|api/chat|v1/completions)')
REQS=$(echo "$LINES" | grep -c "\[GIN\]")
```

This gives request counts and durations. Token counts (prompt tokens, completion tokens) aren't available without the metrics endpoint.

---

## What the Dashboard Actually Shows

The Grafana dashboard has four sections:

**System Overview** (stat panels, always populated):
- Load Average, Memory %, CPU %, GPU GTT Used, GPU Temperature, Disk /

**Ollama LLM** (stat panels):
- Model Loaded (0 or 1, from `/api/ps`)
- Requests/s (from journald)
- Avg Request Duration (from journald)
- Request Duration sum

**Time Series** (graphs):
- Request Rate over time
- System Load (1m/5m/15m)
- GPU Memory — GTT Used, VRAM Used, VRAM Total
- GPU Temperature
- Memory (used/cached/buffered)
- Network (enp2s0f1 RX/TX)

**Token Metrics**:
- Cumulative Request Count
- CPU Detail (user/system/wait %)

The GPU memory panel shows the inference pattern clearly: a spike from ~14 MB (idle) to ~7.8 GB (model loaded) when a request comes in, then back down after the 5-minute keep-alive expires.

---

## Lessons Learned

**The GPU clock issue is the biggest surprise.** APU iGPUs don't behave like discrete GPUs. The power management driver doesn't associate "ROCm inference" with "needs maximum clock". Without explicitly forcing `power_dpm_force_performance_level=high`, you're running at a third of peak frequency and wondering why performance is bad.

**CPU beats GPU for generation on APUs.** If you're running a single-user chat assistant where generation fluency matters, CPU inference with llama.cpp may actually give a better experience. The unified memory architecture removes the GPU's bandwidth advantage. This flips the usual "always use GPU" assumption.

**Ollama's bundled ROCm beats system ROCm.** Don't try to build your own ROCm stack to get "native" support — you'll end up with an older version of the libraries that performs worse. The official Ollama installer bundles ROCm 7.2 with well-optimized kernels. Use it.

**HSA_OVERRIDE_GFX_VERSION works, but understand what it does.** You're telling the hardware to lie about what it is. The gfx1102 Tensile kernels run on gfx1103 hardware because the architectures are close enough. This isn't a hack — AMD maintains near-binary compatibility across RDNA3 variants by design. But it means you're not running kernel code compiled for your exact hardware. When AMD eventually ships gfx1103 support in a future ROCm release, removing the override should give better performance.

**For APU inference, the model size ceiling is real but generous.** 24 GB GTT gives you headroom for anything up to ~13B parameters at Q4 quantization — comfortably covering the best open-source models in the "fits in a living room" category. Gemma 4 12B QAT Q4_0 is an excellent choice: quantization-aware training means near-BF16 quality at Q4_0 file size.

**Check the boring hardware before the exotic one.** I spent three days on ROCm kernel archaeology when the actual bottleneck was a single stick of RAM in a dual-channel board. `dmidecode -t 17` would have told me on day one. The spec sheet's "up to dual-channel" and `free -h`'s single total number both hid it. Verify channel population physically.

**On a bandwidth-starved machine, sparse architecture beats every optimization.** The single biggest win wasn't a kernel, a clock, or a driver — it was switching from a dense 12B to a **MoE 26B-A4B running on CPU**: 13–14 tok/s, 3× faster, *and* a more capable model, because it reads only ~4B active parameters per token. If your generation is memory-bound (and on an APU it always is), reach for MoE before you reach for ROCm flags.

**Modern models think — budget for it.** Gemma 4 burns tokens on hidden reasoning before answering. A low `num_predict` returns an empty string with `done_reason: length`. Use `"think": false` for direct answers, or give it room to reason.

---

The machine runs 24/7. Response times are acceptable. It doesn't phone home. The electricity bill is a rounding error.

For anyone who wants a local AI assistant without a datacenter, an APU mini-PC is a surprisingly capable platform. The path there wasn't the one I expected: I went in chasing GPU drivers and came out having learned that the GPU was a red herring. The two things that actually mattered — a second stick of RAM, and a sparse model on the CPU — had nothing to do with the iGPU I spent three days fighting.