[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus Ready)

打破 24G 显存的物理法则：在单张 RTX 3090 上，以 100% GPU 满血卸载运行 Gemma-4-31B / Qwopus-27B，并维持 96K - 256K 超大上下文。

## 🔍 为什么会有这个项目？

截止 2026 年 4 月，开源 AI 社区的本地部署面临一个“死局”：
1. **官方进度滞后**：官方 `llama.cpp` 主线迟迟未将 TurboQuant 算子 (`tbqp3_0`) 原生并入 CUDA，导致 30B+ 级别的模型在长文本预填充时瞬间撑爆 24G 显存。
2. **社区 Fork 兼容性断裂**：现有的民间 TurboQuant 魔改分支更新缓慢，无法兼容 Gemma 4 最新引入的 SWA (滑动窗口) / ISWA 混合缓存架构，运行即报 `GGML_ASSERT` 甚至段错误。
3. **Agent 调度死亡循环**：当搭配 Claude Code、ZeroClaw 或 Google Antigravity 等 AI IDE 使用时，超长 Prompt 的预填充延迟极易触发客户端超时重试，导致服务器陷入疯狂 `cancel task` 的高功耗空转死循环。

**本项目提供了一套提纯后的极限补丁 (`llama_turboquant.patch`)。** 它将最核心的 TurboQuant 旋转压缩算子精准注入到官方最新主干代码中，手动解决了 C++ 源码冲突与内存对齐报错，完美绕过崩溃红线。

---

## 📊 极限性能实测数据

- **测试硬件**: 单卡 NVIDIA GeForce RTX 3090 (24GB VRAM)
- **测试模型**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **极致显存压榨**: 
  - 启动 98,304 (96K) 上下文，总显存占用精确锁定在 **23,644 MiB / 24,576 MiB**。
  - KV Cache (K+V) 从原本的 >6GB 暴力压缩至 **~1.48 GB**。
- **生成速度 (Decode)**: 纯血 `-ngl 99` 加速下，稳定输出 **~16.5 tokens/s**。

---

## 🛠️ 构建与注入指南 (Build Instructions)

为了保证补丁能 100% 完美打入，避免官方底层 API 突变带来的编译错误，**请务必严格锁定到经过极限测试的特定 Commit 版本**。

### 1. 准备代码库与锁定版本
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# 必须锁定到此安全版本 (Build b8728)
git checkout 5e9c63546

# 愉快地编译和运行
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
