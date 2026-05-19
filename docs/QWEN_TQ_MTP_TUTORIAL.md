# 小白必看：从零在本地跑通 Qwen 3.6 27B (TQ + MTP 加速版)

本教程将引导你在 Ubuntu 环境下，使用 **Atomic (TQ + MTP)** 后端来运行 Qwen 3.6 27B 模型。MTP (Multi-Token Prediction) 允许模型在一次推理中预测多个 token，配合 **TurboQuant (TQ)** 的显存压缩技术，可以获得极高的吞吐量。

## 1. 前置准备 (Environment Setup)

请确保你已经安装了 NVIDIA 驱动程序。运行以下命令安装必要的构建工具：

```bash
sudo apt update
sudo apt install -y git cmake g++ gcc python3-pip
```

## 2. 下载 MTP 模型 (Downloading MTP Models)

MTP 模式需要专门为多 Token 预测训练的模型权重。我们将使用 `huggingface-cli` 下载 `unsloth` 提供的 MTP 版本。

首先安装下载工具：
```bash
pip install -U "huggingface_hub[cli]"
```

创建一个存放模型的文件夹：
```bash
mkdir -p /root/models/qw36/mtp
cd /root/models/qw36/mtp
```

下载 **Qwen 3.6 27B MTP 主模型** (Q4_K_M 量化):
```bash
huggingface-cli download unsloth/Qwen3.6-27B-MTP-GGUF Qwen3.6-27B-Q4_K_M.gguf --local-dir /root/models/qw36/mtp --local-dir-use-symlinks False
```

## 3. 克隆并编译 Atomic 分支 (Cloning & Building)

我们需要使用支持 Atomic/MTP 加速的专用后端代码。

```bash
# 返回你的工作根目录
cd ~

# 克隆支持 TQ/MTP 的专用代码仓库
git clone https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant.git atomic-llama-cpp
cd atomic-llama-cpp

# 配置编译参数 (开启 CUDA 和 Flash Attention)
cmake -B build \
    -DGGML_CUDA=ON \
    -DGGML_NATIVE=ON \
    -DGGML_CUDA_FA=ON \
    -DGGML_CUDA_FA_ALL_QUANTS=ON

# 开始编译
cmake --build build --config Release -j $(nproc) --target llama-server
```

## 4. 运行魔法命令：启动 TQ+MTP 加速服务

MTP 模式下，主模型同时充当草稿角色。

```bash
./build/bin/llama-server \
    -m /root/models/qw36/mtp/Qwen3.6-27B-Q4_K_M.gguf \
    -md /root/models/qw36/mtp/Qwen3.6-27B-Q4_K_M.gguf \
    --spec-type nextn \
    --draft-max 3 \
    -ctk turbo4 \
    -ctv turbo4 \
    -ngl 99 \
    -fa on \
    -c 16384 \
    --port 8081
```

### 核心参数解释：
- `--spec-type nextn`: 开启 **MTP (Multi-Token Prediction)** 加速。
- `--draft-max 3`: 设置每次预测的额外 token 数量（通常设置为 1-3）。
- `-ctk turbo4 -ctv turbo4`: 开启 **TurboQuant** KV 缓存压缩。
- `-ngl 99`: 层全部卸载到 GPU。
- `-fa on`: 开启 Flash Attention。

## 5. 发送测试请求 (Test it out)

服务启动后（默认端口 8081），运行以下命令测试：

```bash
curl http://localhost:8081/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "messages": [{"role": "user", "content": "你好，请用 MTP 模式简单介绍下你自己。"}],
        "max_tokens": 128,
        "temperature": 0
    }'
```

如果看到流畅的回复，说明你的 Qwen 3.6 27B MTP 版已经成功跑通！
