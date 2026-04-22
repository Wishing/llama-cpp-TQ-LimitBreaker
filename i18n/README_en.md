[ 🇺🇸 English](README_en.md) · [ 🇨🇳 简体中文](../README.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **CRITICAL: VERSION MISMATCH WILL CAUSE COMPILATION FAILURE OR RUNTIME CRASH.**
> This patch is strictly designed for **llama.cpp commit `ca7f7b7b`**. Applying it to any other version (newer or older) will likely result in manual conflict resolution failures or memory alignment errors.
> **DO NOT** attempt to apply this patch without first running `git checkout ca7f7b7b`.

> **Breaking the Physical Laws of 24GB VRAM: Run Gemma-4-31B / Qwopus-27B with 100% GPU full offload on a single RTX 3090, while maintaining 96K - 256K massive context.**

## 🔍 Why this project?

As of April 2026, the local deployment of the open-source AI community faces a "deadlock":
1. **Official Lag**: The official `llama.cpp` mainline has not natively integrated TurboQuant operators (`tbqp3_0`) into CUDA.
2. **Advanced Hybrid Support**: This patch provides deep support for the **Gemma 4** series and latest **Qwopus** variants (e.g., **Qwopus-GLM-Heretic-27B-dare-ties**), which utilize the latest SWA (Sliding Window) / ISWA hybrid cache architecture.
3. **Optimized for Agentic Workflows**: Prevents prefill latency timeouts and server "death loops" when used with modern AI IDEs.

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
   git checkout ca7f7b7b
   ```
3. Copy the `llama_turboquant.patch` file from this project into the `llama.cpp` root directory.

4. Apply the patch and build:
   ```bash
   git apply llama_turboquant.patch
   cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
   cmake --build build --config Release -j $(nproc) --target llama-server
   ```

### 2. Run Examples

#### A. Running Gemma-4-31B (96K Context on Single 3090)
```bash
./build/bin/llama-server \
    -m /path/to/gemma-4-31B-it-UD-IQ3_XXS.gguf \
    -c 98304 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --port 1337
```

#### B. Running Qwopus-27B Heretic (Ultra-Long Context)
```bash
./build/bin/llama-server \
    -m /path/to/Qwopus-GLM-Heretic-27B-dare-ties.gguf \
    -c 131072 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --port 1337
```
