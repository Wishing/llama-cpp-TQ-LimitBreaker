[ 🇨🇳 简体中文](README.md) · [ 🇺🇸 English](i18n/README_en.md) · [ 🇯🇵 日本語](i18n/README_ja.md) · [ 🇰🇷 한국어](i18n/README_ko.md) · [ 🇻🇳 Tiếng Việt](i18n/README_vi.md) · [ 🇵🇭 Tagalog](i18n/README_tl.md) · [ 🇪🇸 Español](i18n/README_es.md) · [ 🇵🇹 Português](i18n/README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **重要提示：版本不匹配会导致编译失败或运行时崩溃。**
> 本补丁严格针对 **llama.cpp commit `518b5ffb`** 设计。将其应用于任何其他版本（无论新旧）都可能导致手动冲突解决失败或内存对齐错误。
> **在运行 `git checkout 518b5ffb` 之前，请勿尝试应用此补丁。**

打破 24G 显存的物理法则：在单张 RTX 3090 上，以 100% GPU 满血卸载运行 Gemma-4-31B / Qwopus-27B / Qwen-3.5-27B，并维持 96K - 256K 超大上下文。

## 🔍 为什么会有这个项目？ (The Motivation)

在本地大模型（LLM）领域，**Gemma 4** 和 **Qwopus** 的出现标志着 30B 级别模型已经拥有了超越以往 70B 模型的逻辑能力。然而，开发者面临着官方支持的“技术断层”：

1.  **官方 TurboQuant 缺失**：尽管 `llama.cpp` 社区有关于 TurboQuant 的讨论，但其核心 CUDA 算子（尤其是针对 Key 缓存的 `tbqp` 系列）在最新主线中仍未合并。这意味着官方版本无法在 CUDA 上享受 3-bit KV 压缩带来的显存红利。
2.  **Gemma 4 适配荒原**：Gemma 4 采用了极其复杂的 **SWA (滑动窗口) + ISWA (独立滑动窗口)** 混合缓存机制。目前官方主线在处理此类混合架构时，无法直接应用 TurboQuant 量化，会导致严重的精度崩坏或运行时内存对齐错误。
3.  **DFlash 的必然性**：在超长上下文（100K+）下，传统的投机采样（如 Eagle3）会产生巨大的额外 KV 显存占用，导致“为了加速而爆显存”。**DFlash** 是专为此补丁设计的轻量化投机引擎，它与 TurboQuant 算子深度融合，在维持极低显存占用的同时，通过预测机制抵消了量化带来的微小延迟增加。

---

## 🛠️ 构建与注入指南 (Build Instructions)

本项目提供三个补丁文件（dflash 及全功能补丁目前正在紧急适配中）：

| 补丁文件 | 包含功能 | 适用场景 | 状态 |
| :--- | :--- | :--- | :--- |
| **`llama_turboquant.patch`** | 仅 TurboQuant | 只需要 KV 缓存压缩（显存优化），不需要投机采样。 | ✅ **已发布 (稳定版)** |
| **`llama_dflash.patch`** | 仅 DFlash | 只需要投机采样加速，不需要 KV 缓存压缩。 | ✅ **已发布 (稳定版)** |
| **`llama_tq_df.patch`** | **TQ + DFlash** | 同时启用 KV 压缩与投机加速，实现显存与速度的双重极限。 | ✅ **已发布 (稳定版)** |

### 1. 准备代码库与锁定版本
1. 克隆官方 llama.cpp 代码库：
   ```bash
   git clone https://github.com/ggml-org/llama.cpp.git
   cd llama.cpp
   ```
2. **必须**锁定到此安全版本 (Validated Commit)：
   ```bash
   git checkout 518b5ffb
   ```

### 2. 应用补丁 (目前推荐方案 A)

*   **方案 A (推荐 - 全能型)**：同时应用 TurboQuant 与 DFlash 补丁
    ```bash
    git apply /path/to/llama_tq_df.patch
    ```
*   **方案 B (显存优化型)**：仅应用 TurboQuant 补丁
    ```bash
    git apply /path/to/llama_turboquant.patch
    ```
*   **方案 C (速度优化型)**：仅应用 DFlash 补丁
    ```bash
    git apply /path/to/llama_dflash.patch
    ```

### 3. 编译
```bash
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
cmake --build build --config Release -j $(nproc) --target llama-server
```

---

## 📊 已测试并支持的模型

本补丁针对以下架构进行了深度优化和测试：
- **Gemma 4 全系列**：完整支持其复杂的 SWA/ISWA 混合缓存对齐。
  - *测试模型：Gemma-4-31B-It-UD-IQ3_XXS.gguf*
- **Qwen 3.5 系列**：针对 Qwen 架构深度优化，包括：
  - `Qwen3.5-27B-It` (最新支持)
  - 所有基于 Qwen2/2.5/3.5 的 27B+ 稠密衍生模型。

---

## 🚀 核心参数深度解析

| 参数 | 建议值 | 说明与优势 |
| :--- | :--- | :--- |
| **`-ctk`** | `turbo3_tcq` | **TurboQuant Key 缓存量化**。`turbo3_tcq` 包含 QJL 优化，3-bit 可节省约 75% KV 显存。 |
| **`-ctv`** | `turbo3` | **TurboQuant Value 缓存量化**。配合 `-ctk` 使用，大幅降低显存压力。 |
| **`--spec-type`** | `dflash` | **启用 DFlash 投机引擎**。利用专用的 Draft 模型预测 Token，在不损失精度的前提下显著提升推理 TPS。 |
| **`-md`** | `draft.gguf` | **指定 Draft 模型**。需配合 `--spec-type dflash` 使用。 |

---

#### 💡 极客极限配置示例 (Ultimate Geek Examples)

##### 🌟【终极方案】Gemma-4-31B 满血全量：128K 极限上下文 + TQ 压缩
注意：Gemma 4 验证重点为 TurboQuant 显存节省，暂不建议配合 DFlash 使用。

```bash
./build/bin/llama-server \
    -m models/gemma-4-31b-it-UD-IQ3_XXS.gguf \
    -ngl 99 -fa on -c 128000 \
    -ctk turbo3_tcq -ctv turbo3 \
    --port 1337
```

##### 💎【全能加速】Qwen3.5-27B：DFlash + TurboQuant 混合模式
这是性能最均衡的配置，同时享受显存压缩与投机采样加速。

```bash
./build/bin/llama-server \
    -m models/Qwen3.5-27B-UD-IQ3_XXS.gguf \
    -md models/dflash-draft-q4_k_m.gguf \
    --spec-type dflash \
    -ngl 99 -fa on -c 32768 -b 512 \
    -ctk turbo3_tcq -ctv turbo3 \
    --port 1337
```

