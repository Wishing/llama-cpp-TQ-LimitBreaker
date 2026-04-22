[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

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

#### A. 运行 Gemma-4-31B (单卡 3090 运行 96K 上下文)
```bash
./build/bin/llama-server \
    -m /path/to/gemma-4-31B-it-UD-IQ3_XXS.gguf \
    -c 98304 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --port 1337
```

#### B. 运行 Qwopus-27B Heretic (超长上下文支持)
```bash
./build/bin/llama-server \
    -m /path/to/Qwopus-GLM-Heretic-27B-dare-ties.gguf \
    -c 131072 -b 1024 -ub 512 -ngl 99 -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --port 1337
```
