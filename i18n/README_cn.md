[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](i18n/README_it.md) · [ 🇩🇪 Deutsch](i18n/README_de.md) · [ 🇫🇷 Français](i18n/README_fr.md) · [ 🇸🇦 العربية](i18n/README_ar.md) · [ 🇮🇳 हिन्दी](i18n/README_hi.md) · [ 🇷🇺 Русский](i18n/README_ru.md) · [ 🇧🇩 বাংলা](i18n/README_bn.md) · [ 🇮🇱 עברית](i18n/README_he.md) · [ 🇵🇱 Polski](i18n/README_pl.md) · [ 🇨🇿 Čeština](i18n/README_cs.md) · [ 🇳🇱 Nederlands](i18n/README_nl.md) · [ 🇹🇷 Türkçe](i18n/README_tr.md) · [ 🇺🇦 Українська](i18n/README_uk.md) · [ 🇮🇩 Bahasa Indonesia](i18n/README_id.md) · [ 🇹🇭 ไทย](i18n/README_th.md) · [ 🇵🇰 اردو](i18n/README_ur.md) · [ 🇷🇴 Română](i18n/README_ro.md) · [ 🇸🇪 Svenska](i18n/README_sv.md) · [ 🇬🇷 Ελληνικά](i18n/README_el.md) · [ 🇭🇺 Magyar](i18n/README_hu.md) · [ 🇫🇮 Suomi](i18n/README_fi.md) · [ 🇩🇰 Dansk](i18n/README_da.md) · [ 🇳🇴 Norsk](i18n/README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

> [!CAUTION]
> **重要提示：版本不匹配会导致编译失败或运行时崩溃。**
> 本补丁严格针对 **llama.cpp commit `518b5ffb`** 设计。将其应用于任何其他版本（无论新旧）都可能导致手动冲突解决失败或内存对齐错误。
> **在运行 `git checkout 518b5ffb` 之前，请勿尝试应用此补丁。**

打破 24G 显存的物理法则：在单张 RTX 3090 上，以 100% GPU 满血卸载运行 Gemma-4-31B / Qwopus-27B / Qwen-3.5-27B，并维持 96K - 256K 超大上下文。

## 🔍 为什么会有这个项目？

1. **官方 TurboQuant 缺失**：官方 `llama.cpp` 主线目前尚未合并针对 CUDA 的 TurboQuant 核心算子。
2. **深度适配混合架构**：本补丁特别加强了对 **Gemma 4** 系列以及 **Qwen 3.5** 系列的支持，完美兼容其最新的 SWA (滑动窗口) / ISWA 混合缓存。
3. **DFlash 投机引擎**：专为超长上下文设计的轻量化投机采样，在维持低显存的同时大幅提升生成速度。

---

## 🛠️ 构建与注入指南 (Build Instructions)

本项目提供三个补丁文件：

| 补丁文件 | 包含功能 | 适用场景 | 状态 |
| :--- | :--- | :--- | :--- |
| **`llama_turboquant.patch`** | 仅 TurboQuant | 只需要 KV 缓存压缩（显存优化）。 | ✅ **已发布 (稳定版)** |
| **`llama_dflash.patch`** | 仅 DFlash | 只需要投机采样加速。 | ✅ **已发布 (稳定版)** |
| **`llama_tq_df.patch`** | **TQ + DFlash** | 同时启用 KV 压缩与投机加速。 | ✅ **已发布 (稳定版)** |

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

### 3. 编译
```bash
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
cmake --build build --config Release -j $(nproc) --target llama-server
```

---

## 📊 已测试并支持的模型

- **Gemma 4 全系列**：完整支持其复杂的 SWA/ISWA 混合缓存对齐。
- **Qwen 3.5 系列**：针对 Qwen 架构深度优化，支持稠密版本。

---

#### 💡 极客极限配置示例 (Ultimate Geek Examples)

##### 🌟【终极方案】Gemma-4-31B 满血全量：128K 极限上下文 + TQ 压缩
```bash
./build/bin/llama-server \
    -m models/gemma-4-31b-it-UD-IQ3_XXS.gguf \
    -ngl 99 -fa on -c 128000 \
    -ctk turbo3_tcq -ctv turbo3 \
    --port 1337
```

##### 💎【全能加速】Qwen3.5-27B：DFlash + TurboQuant 混合模式
```bash
./build/bin/llama-server \
    -m models/Qwen3.5-27B-UD-IQ3_XXS.gguf \
    -md models/dflash-draft-q4_k_m.gguf \
    --spec-type dflash \
    -ngl 99 -fa on -c 32768 -b 512 \
    -ctk turbo3_tcq -ctv turbo3 \
    --port 1337
```

