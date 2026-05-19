# llama.cpp TQ + LimitBreaker

本项目是关于本地大模型性能与质量测评的官方成果发布库，专注于 **Qwen 3.6** 系列模型在 **TurboQuant (TQ)**、**DFlash** 以及 **MTP** 等加速技术下的实战优化。

## 🚀 快速入门教程

为了方便用户快速上手，我们提供了针对不同加速方案的详细“小白级”教程：

1.  **[Qwen 3.6 27B (TQ + DFlash) 跑通教程](docs/QWEN_TQ_DF_TUTORIAL.md)**
    *   **核心特性**：使用 `spiritbuun/buun-llama-cpp` 后端，支持投机采样 (DFlash) 与 KV 缓存压缩 (TQ)。
    *   **适用场景**：在有限显存下追求极致的生成速度与响应。

2.  **[Qwen 3.6 27B (TQ + MTP) 跑通教程](docs/QWEN_TQ_MTP_TUTORIAL.md)**
    *   **核心特性**：使用 `AtomicBot-ai/atomic-llama-cpp-turboquant` 后端，支持多 Token 预测 (MTP) 架构。
    *   **适用场景**：追求最高吞吐量（Tokens per second）的批处理或长文本场景。

## 📂 目录结构

*   `docs/`：包含详细的配置、安装与运行教程。
*   `weights/`：(待发布) 优化的量化权重或配置文件。
*   `scripts/`：(待发布) 用于一键部署或压测的自动化脚本。

---
*本仓库内容经过严格测试，建议在 Ubuntu / Debian 环境下搭配 NVIDIA GPU 使用。*
