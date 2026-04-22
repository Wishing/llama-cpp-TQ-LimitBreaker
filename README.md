[ 🇨🇳 简体中文](README.md) · [ 🇺🇸 English](i18n/README_en.md) · [ 🇯🇵 日本語](i18n/README_ja.md) · [ 🇰🇷 한국어](i18n/README_ko.md) · [ 🇻🇳 Tiếng Việt](i18n/README_vi.md) · [ 🇵🇭 Tagalog](i18n/README_tl.md) · [ 🇪🇸 Español](i18n/README_es.md) · [ 🇵🇹 Português](i18n/README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **重要提示：版本不匹配会导致编译失败或运行时崩溃。**
> 本补丁严格针对 **llama.cpp commit `ca7f7b7b`** 设计。将其应用于任何其他版本（无论新旧）都可能导致手动冲突解决失败或内存对齐错误。
> **在运行 `git checkout ca7f7b7b` 之前，请勿尝试应用此补丁。**

打破 24G 显存的物理法则：在单张 RTX 3090 上，以 100% GPU 满血卸载运行 Gemma-4-31B / Qwopus-27B，并维持 96K - 256K 超大上下文。

## 🔍 为什么会有这个项目？

截止 2026 年 4 月，开源 AI 社区的本地部署面临一个“死局”：
1. **官方进度滞后**：官方 `llama.cpp` 主线迟迟未将 TurboQuant 算子 (`tbqp3_0`) 原生并入 CUDA。
2. **深度适配混合架构**：本补丁特别加强了对 **Gemma 4** 系列以及 **Qwen/Qwopus** 变体（如 **Qwopus-GLM-Heretic-27B-dare-ties**）的支持，完美兼容其最新的 SWA (滑动窗口) / ISWA 混合缓存。
3. **Agent 工作流优化**：解决在使用 AI IDE 时超长 Prompt 导致的预填充超时和服务器“死亡循环”。

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

#### 💡 完整运行示例 (Full Examples)

##### A. 极限上下文模式：Gemma-4-31B (单卡 3090 运行 128K 上下文)
此配置利用 **TurboQuant 3-bit** 压缩，将原本需要 60G+ 显存的 128K KV Cache 压缩至约 15G，从而在 24G 显卡上全量运行。

```bash
./build/bin/llama-server \
    -m models/gemma-4-31b-it-UD-IQ3_XXS.gguf \
    -c 131072 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp3 -ctv tbq3 \
    --port 1337
```

##### B. 极速推理模式：Qwopus-27B + DFlash 投机加速
此配置同时开启了 **4-bit KV 压缩**（兼顾精度与显存）与 **DFlash 投机采样**，推理速度通常可提升 1.5x - 2x。

```bash
./build/bin/llama-server \
    -m models/Qwopus-GLM-Heretic-27B-dare-ties.gguf \
    -md models/qwopus-27b-dflash-draft.gguf \
    --dflash \
    -c 98304 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp4 -ctv tbq4 \
    --port 1337
```
