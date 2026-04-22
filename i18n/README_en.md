[ 🇺🇸 English](README_en.md) · [ 🇨🇳 简体中文](../README.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **CRITICAL: VERSION MISMATCH WILL CAUSE COMPILATION FAILURE OR RUNTIME CRASH.**
> This patch is strictly designed for **llama.cpp commit `82d3f4d3`**. Applying it to any other version (newer or older) will likely result in manual conflict resolution failures or memory alignment errors.
> **DO NOT** attempt to apply this patch without first running `git checkout 82d3f4d3`.

> **Breaking the Physical Laws of 24GB VRAM: Run Gemma-4-31B / Qwopus-27B with 100% GPU full offload on a single RTX 3090, while maintaining 96K - 256K massive context.**

## 🔍 Why this project? (The Motivation)

In the era of local LLMs, **Gemma 4** and **Qwopus** represent a breakthrough where 30B-class models outperform previous 70B giants. However, developers face a significant "technical gap" in official support:

1.  **Missing TurboQuant Kernels**: While discussed in the community, the core CUDA kernels for TurboQuant (especially the `tbqp` series for Key cache) have not been merged into the official `llama.cpp` mainline. This means official builds cannot benefit from the ~75% KV VRAM savings.
2.  **Gemma 4 Hybrid Cache Support**: Gemma 4 utilizes a sophisticated **SWA (Sliding Window) + ISWA (Independent SWA)** hybrid cache mechanism. The official mainline currently cannot apply TurboQuant to these hybrid layers correctly, leading to massive perplexity degradation or runtime crashes.
3.  **Necessity of DFlash**: At extreme context lengths (100K+), traditional speculative engines like Eagle3 incur significant VRAM overhead, often causing "OOM for the sake of speed." **DFlash** is a lightweight engine specifically tuned for this patch, deeply integrated with TurboQuant operators to provide acceleration without bloated VRAM usage.

---

## 📊 Supported & Tested Models

This patch is heavily optimized and tested for:
- **Gemma 4 Series**: Fully supports SWA/ISWA hybrid cache alignment. 
  - *Tested: Gemma-4-31B-It-UD-IQ3_XXS.gguf*
- **Qwopus Variants**: Optimized for Qwen2-based architectures.
  - *Tested: Qwopus-GLM-Heretic-27B-dare-ties.gguf*
  - *Tested: Qwopus3.5-27B-v3*
- **Extreme VRAM Savings**: 
  - **Gemma-4-31B** + **96K Context** = **~23.6 GB VRAM** (Single 3090/4090).
  - KV Cache reduction: **75% Compression Ratio**.

---

## 🛠️ Build & Injection Guide

To ensure the patch is applied perfectly, **please strictly lock your repository to the tested specific commit version**.

### 1. Prepare Repository & Lock Version
1. Clone the official llama.cpp repository:
   ```bash
   git clone https://github.com/ggml-org/llama.cpp.git
   cd llama.cpp
   ```
2. **MUST** lock to this safe version (Validated Commit):
   ```bash
   git checkout 82d3f4d3
   ```
3. Copy the `llama_turboquant.patch` file from this project into the `llama.cpp` root directory.

4. Apply the patch and build:
   ```bash
   git apply llama_turboquant.patch
   cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
   cmake --build build --config Release -j $(nproc) --target llama-server
   ```

### 2. Run Examples

This patch introduces two core technologies: **TurboQuant (KV Compression)** and **DFlash (Speculative Acceleration)**.

#### 🚀 Core Parameters Explained

| Parameter | Recommended | Description & Advantages |
| :--- | :--- | :--- |
| **`-ctk`** | `tbqp3` / `tbqp4` | **TurboQuant Key Cache Quantization**. `tbqp` includes QJL optimization. 3-bit saves ~75% KV VRAM, essential for 100K+ context lengths. |
| **`-ctv`** | `tbq3` / `tbq4` | **TurboQuant Value Cache Quantization**. Used with `-ctk` to drastically reduce VRAM pressure during long-context inference. |
| **`--dflash`** | (Toggle) | **Enable DFlash Speculative Engine**. Predicts tokens using a dedicated Draft model, significantly boosting TPS without losing accuracy. |
| **`-md`** | `draft.gguf` | **Specify Draft Model**. Required for `--dflash`. Usually a specialized 0.1B~1B parameter model for speed. |

---

#### 💡 Ultimate Geek Configurations

##### 🌟【The Final Solution】Gemma-4-31B Full Offload: 256K Context + DFlash
The raison d'être of this project: Run a 256K window on a single RTX 3090/4090, a workload that normally requires 4x A100s, while maintaining smooth typing speeds.
- **VRAM Compression**: 3-bit TurboQuant compresses KV cache from ~120GB to ~28GB (enabled via unified memory sharing).
- **Inference Speed**: DFlash speculative sampling offsets prefill/quantization latency.

```bash
./build/bin/llama-server \
    -m models/gemma-4-31b-it-UD-IQ3_XXS.gguf \
    -md models/gemma-4-dflash-draft.gguf \
    --dflash \
    -c 262144 -b 2048 -ub 1024 -ngl 99 -fa \
    -ctk tbqp3 -ctv tbq3 \
    --port 1337
```

##### ⚡【Maximum Speed】Qwopus-27B: Instant Response at 100K Context
Ultra-throughput configuration optimized for Qwen-based architectures.

```bash
./build/bin/llama-server \
    -m models/Qwopus-GLM-Heretic-27B-dare-ties.gguf \
    -md models/qwopus-27b-dflash-draft.gguf \
    --dflash \
    -c 98304 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp4 -ctv tbq4 \
    --port 1337
```
