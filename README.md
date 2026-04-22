[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](i18n/README_cn.md) · [ 🇯🇵 日本語](i18n/README_ja.md) · [ 🇰🇷 한국어](i18n/README_ko.md) · [ 🇻🇳 Tiếng Việt](i18n/README_vi.md) · [ 🇵🇭 Tagalog](i18n/README_tl.md) · [ 🇪🇸 Español](i18n/README_es.md) · [ 🇵🇹 Português](i18n/README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **CRITICAL: VERSION MISMATCH WILL CAUSE COMPILATION FAILURE OR RUNTIME CRASH.**
> This patch is strictly designed for **llama.cpp commit `ca7f7b7b`**. Applying it to any other version (newer or older) will likely result in manual conflict resolution failures or memory alignment errors.
> **DO NOT** attempt to apply this patch without first running `git checkout ca7f7b7b`.

> **Breaking the Physical Laws of 24GB VRAM: Run Gemma-4-31B / Qwopus-27B with 100% GPU full offload on a single RTX 3090, while maintaining 96K - 256K massive context.**

## 🔍 Why this project?

As of April 2026, the local deployment of the open-source AI community faces a "deadlock":
1. **Official Lag**: The official `llama.cpp` mainline has not natively integrated TurboQuant operators (`tbqp3_0`) into CUDA, causing 30B+ models to instantly exhaust 24GB VRAM during long-text prefill.
2. **Community Fork Compatibility Issues**: Existing community TurboQuant branches are slow to update and cannot compatible with the latest SWA (Sliding Window) / ISWA hybrid cache architecture introduced in Gemma 4, leading to `GGML_ASSERT` or segmentation faults.
3. **Agent Scheduling Death Loop**: When paired with AI IDEs like Claude Code, ZeroClaw, or Google Antigravity, the prefill latency of ultra-long prompts easily triggers client timeout retries, causing the server to fall into a high-power idle death loop of frantic `cancel task`.

**This project provides a purified limit patch (`llama_turboquant.patch`).** It precisely injects the core TurboQuant rotation compression operators into the latest official mainline code, manually resolving C++ source conflicts and memory alignment errors, perfectly bypassing the crash red lines.

---

## 📊 Extreme Performance Benchmarks

- **Hardware**: Single NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Models**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM Optimization**: 
  - 98,304 (96K) context initialized, VRAM usage locked at **23,644 MiB / 24,576 MiB**.
  - KV Cache (K+V) compressed from >6GB to **~1.48 GB**.
- **Generation Speed (Decode)**: Full `-ngl 99` acceleration, stable output at **~16.5 tokens/s**.

---

## 🛠️ Build & Injection Guide

To ensure the patch is applied perfectly and to avoid compilation errors from upstream API changes, **please strictly lock your repository to the tested specific commit version**.

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

### 2. Run Example
```bash
./build/bin/llama-server \
    -m /path/to/gemma-4-31B-it-UD-IQ3_XXS.gguf \
    -c 98304 \
    -b 1024 -ub 512 \
    -ngl 99 \
    -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --parallel 1 \
    --no-prompt-cache \
    --timeout 3600 \
    --port 1337
```
