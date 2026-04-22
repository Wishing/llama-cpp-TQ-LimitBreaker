[ 🇨🇳 简体中文](README.md) · [ 🇺🇸 English](i18n/README_en.md) · [ 🇯🇵 日本語](i18n/README_ja.md) · [ 🇰🇷 한국어](i18n/README_ko.md) · [ 🇻🇳 Tiếng Việt](i18n/README_vi.md) · [ 🇵🇭 Tagalog](i18n/README_tl.md) · [ 🇪🇸 Español](i18n/README_es.md) · [ 🇵🇹 Português](i18n/README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **重要提示：版本不匹配会导致编译失败或运行时崩溃。**
> 本补丁严格针对 **llama.cpp commit `ca7f7b7b`** 设计。将其应用于任何其他版本（无论新旧）都可能导致手动冲突解决失败或内存对齐错误。
> **在运行 `git checkout ca7f7b7b` 之前，请勿尝试应用此补丁。**

打破 24G 显存的物理法则：在单张 RTX 3090 上，以 100% GPU 满血卸载运行 Gemma-4-31B / Qwopus-27B，并维持 96K - 256K 超大上下文。

## 🔍 为什么会有这个项目？ (The Motivation)

在本地大模型（LLM）领域，**Gemma 4** 和 **Qwopus** 的出现标志着 30B 级别模型已经拥有了超越以往 70B 模型的逻辑能力。然而，开发者面临着官方支持的“技术断层”：

1.  **官方 TurboQuant 缺失**：尽管 `llama.cpp` 社区有关于 TurboQuant 的讨论，但其核心 CUDA 算子（尤其是针对 Key 缓存的 `tbqp` 系列）在最新主线中仍未合并。这意味着官方版本无法在 CUDA 上享受 3-bit KV 压缩带来的显存红利。
2.  **Gemma 4 适配荒原**：Gemma 4 采用了极其复杂的 **SWA (滑动窗口) + ISWA (独立滑动窗口)** 混合缓存机制。目前官方主线在处理此类混合架构时，无法直接应用 TurboQuant 量化，会导致严重的精度崩坏或运行时内存对齐错误。
3.  **DFlash 的必然性**：在超长上下文（100K+）下，传统的投机采样（如 Eagle3）会产生巨大的额外 KV 显存占用，导致“为了加速而爆显存”。**DFlash** 是专为此补丁设计的轻量化投机引擎，它与 TurboQuant 算子深度融合，在维持极低显存占用的同时，通过预测机制抵消了量化带来的微小延迟增加。

---

## 📊 已测试并支持的模型

本补丁针对以下架构进行了深度优化和测试：
- **Gemma 4 全系列**：完整支持其复杂的 SWA/ISWA 混合缓存对齐。
  - *测试模型：Gemma-4-31B-It-UD-IQ3_XXS.gguf*
- **Qwopus 系列**：针对 Qwen 架构深度优化，包括：
  - `Qwopus-GLM-Heretic-27B-dare-ties`
  - `Qwopus3.5-27B-v3`
  - 所有基于 Qwen2 的 27B+ 衍生模型。
- **极致显存节省**：
  - **Gemma-4-31B** + **96K 上下文** = **~23.6 GB 显存占用** (单张 3090/4090)。
  - KV Cache 压缩率高达 **75%**。

---

## 🛠️ 构建与注入指南 (Build Instructions)

为了保证补丁能 100% 完美打入，**请务必严格锁定到经过极限测试的特定 Commit 版本**。

### 1. 准备代码库与锁定版本
1. 克隆官方 llama.cpp 代码库：
   ```bash
   git clone https://github.com/ggml-org/llama.cpp.git
   cd llama.cpp
   ```
2. **必须**锁定到此安全版本 (Validated Commit)：
   ```bash
   git checkout ca7f7b7b
   ```
3. 将本项目中的 `llama_turboquant.patch` 文件复制到 `llama.cpp` 的根目录下。

4. 应用补丁并编译：
   ```bash
   git apply llama_turboquant.patch
   cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
   cmake --build build --config Release -j $(nproc) --target llama-server
   ```

### 2. 运行示例

本补丁引入了两大核心技术：**TurboQuant (KV 压缩)** 与 **DFlash (投机加速)**。

#### 🚀 核心参数深度解析

| 参数 | 建议值 | 说明与优势 |
| :--- | :--- | :--- |
| **`-ctk`** | `tbqp3` / `tbqp4` | **TurboQuant Key 缓存量化**。`tbqp` 包含 QJL 优化，3-bit 可节省约 75% KV 显存，是实现 100K+ 上下文的核心。 |
| **`-ctv`** | `tbq3` / `tbq4` | **TurboQuant Value 缓存量化**。配合 `-ctk` 使用，大幅降低长文本下的显存压力。 |
| **`--dflash`** | (开关) | **启用 DFlash 投机引擎**。利用专用的 Draft 模型预测 Token，在不损失精度的前提下显著提升推理 TPS。 |
| **`-md`** | `draft.gguf` | **指定 Draft 模型**。需配合 `--dflash` 使用，通常使用 0.1B~1B 规模的专用加速模型。 |

---

#### 💡 极客极限配置示例 (Ultimate Geek Examples)

##### 🌟【终极方案】Gemma-4-31B 满血全量：256K 极限上下文 + DFlash 加速
这是本项目的“存在意义”：在单张 RTX 3090/4090 上，运行原本需要 4x A100 才能支撑的 256K 窗口，并保持丝滑的打字机速度。
- **显存压缩**：3-bit TurboQuant 将 KV 占用从 ~120GB 压缩至 ~28GB（配合系统内存共享）。
- **推理加速**：DFlash 投机采样抵消预填充延迟。

```bash
./build/bin/llama-server \
    -m models/gemma-4-31b-it-UD-IQ3_XXS.gguf \
    -md models/gemma-4-dflash-draft.gguf \
    --dflash \
    -c 262144 -b 2048 -ub 1024 -ngl 99 -fa \
    -ctk tbqp3 -ctv tbq3 \
    --port 1337
```

##### ⚡【极限速度】Qwopus-27B：100K 窗口下的“秒回”体验
针对 Qwen 架构优化的极致吞吐配置。

```bash
./build/bin/llama-server \
    -m models/Qwopus-GLM-Heretic-27B-dare-ties.gguf \
    -md models/qwopus-27b-dflash-draft.gguf \
    --dflash \
    -c 98304 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp4 -ctv tbq4 \
    --port 1337
```
