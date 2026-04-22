[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](i18n/README_cn.md) · [ 🇯🇵 日本語](i18n/README_ja.md) · [ 🇰🇷 한국어](i18n/README_ko.md) · [ 🇻🇳 Tiếng Việt](i18n/README_vi.md) · [ 🇵🇭 Tagalog](i18n/README_tl.md) · [ 🇪🇸 Español](i18n/README_es.md) · [ 🇵🇹 Português](i18n/README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

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
