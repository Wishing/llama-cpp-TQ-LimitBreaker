🌐 Languages: 🌍 English · 🌏 简体中文 · 🇯🇵 日本語 · 🇰🇷 한국어 · 🇻🇳 Tiếng Việt · 🇵🇭 Tagalog · 🇪🇸 Español · 🇵🇹 Português · 🇮🇹 Italiano · 🇩🇪 Deutsch · 🇫🇷 Français · 🇸🇦 العربية · 🇮🇳 हिन्दी · 🇷🇺 Русский · 🇧🇩 বাংলা · 🇮🇱 עברית · 🇵🇱 Polski · 🇨🇿 Čeština · 🇳🇱 Nederlands · 🇹🇷 Türkçe · 🇺🇦 Українська · 🇮🇩 Bahasa Indonesia · 🇹🇭 ไทย · 🇵🇰 اردو · 🇷🇴 Română · 🇸🇪 Svenska · 🇬🇷 Ελληνικά · 🇭🇺 Magyar · 🇫🇮 Suomi · 🇩🇰 Dansk · 🇳🇴 Norsk

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

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
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MUST lock to this safe version (Build b8728)
git checkout 5e9c63546

# Apply patch and build
git apply llama_turboquant.patch
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
cmake --build build --config Release -j $(nproc) --target llama-server
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
