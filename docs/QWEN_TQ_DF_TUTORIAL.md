# 小白必看：从零在本地跑通 Qwen 3.6 27B (TQ + DFlash 加速版)

本教程将引导你在 Ubuntu 环境下，从零开始配置、下载、编译并运行 Qwen 3.6 27B 模型。我们将使用 **TurboQuant (TQ)** 和 **DFlash (投机采样)** 两种核心加速技术。

## 1. 前置准备 (Environment Setup)

在开始之前，请确保你已经安装了 NVIDIA 驱动程序。接着，运行以下命令安装必要的构建工具：

```bash
sudo apt update
sudo apt install -y git cmake g++ gcc python3-pip
```

安装完成后，建议确认下你的 GCC 版本（推荐 GCC 12 或更高）：
```bash
gcc --version
```

## 2. 下载模型 (Downloading Models from HF)

我们将使用 `huggingface-cli` 来下载特定的模型量化版本。

首先安装下载工具：
```bash
pip install -U "huggingface_hub[cli]"
```

创建一个存放模型的文件夹：
```bash
mkdir -p /root/models/qw36/vanilla
cd /root/models/qw36/vanilla
```

下载 **Qwen 3.6 27B 主模型** (Q4_K_M 量化):
```bash
huggingface-cli download unsloth/Qwen3.6-27B-GGUF Qwen3.6-27B-Q4_K_M.gguf --local-dir /root/models/qw36/vanilla --local-dir-use-symlinks False
```

下载 **DFlash 草稿模型** (q8_0 量化):
```bash
huggingface-cli download spiritbuun/Qwen3.6-27B-DFlash-GGUF dflash-draft-3.6-q8_0.gguf --local-dir /root/models/qw36/vanilla --local-dir-use-symlinks False
```

*注意：下载可能需要一些时间，具体取决于你的网络带宽。*

## 3. 克隆并编译 Buun 分支 (Cloning & Building)

我们需要支持 TurboQuant 和 DFlash 的特殊后端代码。

```bash
# 返回你的工作根目录
cd ~

# 克隆支持 TQ/DF 的专用代码仓库
git clone https://github.com/spiritbuun/buun-llama-cpp.git buun-llama-cpp
cd buun-llama-cpp

# 配置编译参数 (开启 CUDA 和 Flash Attention)
cmake -B build \
    -DGGML_CUDA=ON \
    -DGGML_NATIVE=ON \
    -DGGML_CUDA_FA=ON \
    -DGGML_CUDA_FA_ALL_QUANTS=ON

# 开始编译 (这一步可能需要几分钟)
cmake --build build --config Release -j $(nproc) --target llama-server
```

## 4. 运行魔法命令：启动 TQ+DF 加速服务

现在，你可以用下面的命令直接启动加速服务了。

```bash
./build/bin/llama-server \
    -m /root/models/qw36/vanilla/Qwen3.6-27B-Q4_K_M.gguf \
    -md /root/models/qw36/vanilla/dflash-draft-3.6-q8_0.gguf \
    --spec-type dflash \
    -ngld 99 \
    -ctk turbo4 \
    -ctv turbo4 \
    -ctkd q4_0 \
    -ctvd q4_0 \
    -ngl 99 \
    -fa on \
    -c 16384 \
    --port 8080
```

### 核心参数小白翻译：
- `-m` 和 `-md`: 分别指定主模型和“草稿”模型的路径。
- `--spec-type dflash`: 开启 **DFlash** 投机采样加速，能显著提升生成速度。
- `-ctk turbo4 -ctv turbo4`: 开启 **TurboQuant** KV 缓存，大幅节省显存占用，让你在更小的显存上跑更大的模型。
- `-ngl 99`: 将所有模型层都放到 GPU 上（如果你显存够的话）。
- `-fa on`: 开启 Flash Attention，进一步提速。
- `-c 16384`: 设置上下文长度。

## 5. 发送测试请求 (Test it out)

服务启动后，新开一个终端，运行以下命令测试：

```bash
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "messages": [{"role": "user", "content": "你好，请自我介绍一下。"}],
        "max_tokens": 128,
        "temperature": 0
    }'
```

如果看到模型正常返回了 JSON 格式的回复，恭喜你，你已经成功跑通了 Qwen 3.6 27B 加速版！
