---
title: "使用ktransformers docker部署deepseek-r1的笔记"
date: 2025-02-26T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - LLM
  - deepseek
  - deepseek-r1
  - ktransformers
---

## 编译和安装

这里我们使用一个 Docker 镜像来执行编译生成 whl 包，可以供后续其他镜像使用．

1. 下载源码：

```bash
git clone git@github.com:kvcache-ai/ktransformers.git
cd ktransformers
git submodule update --init --recursive
```

2. 然后使用一个 Docker 镜像来编译：

```bash
docker run -it --rm --gpus all -v ./:/data --shm-size 8G pytorch/pytorch:2.5.1-cuda12.1-cudnn8-devel bash
cd /data
# 安装依赖
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
pip install build cpufeature
# 编译生成 whl 文件，这里的 TORCH_CUDA_ARCH_LIST 根据自己的显卡来
# 可以参考 https://arnon.dk/matching-sm-architectures-arch-and-gencode-for-various-nvidia-cards 来确定
LANG=en_US.UTF-8 \
TORCH_CUDA_ARCH_LIST="8.6" \
KTRANSFORMERS_FORCE_BUILD=TRUE \
python -m build --wheel --no-isolation
# 编译完毕之后会在 dist 目录下得到一个 whl 文件
```

## 构建 Docker 镜像

这里构建 Docker 镜像可以使用 `Dockerfile.local`．替换其中的 `<WHL-URL>` 为实际的 whl url．这里我们可以使用以下命令来快速启动一个 HTTP server:

```bash
# 在上一步生成的 dist 目录下执行
python -m http.server 8083
```

然后打开浏览器复制拿到 whl 文件的 URL 即可．构建 Docker 镜像的命令为：

```bash
docker build -t ktransformers:$(date -uI) --progress plain -f Dockerfile.local .
```

这里编译和构建步骤分开，且写得这么啰嗦，是想要让最终生成的 Docker 镜像尽可能的小．如果不太介意这个的话，可以直接将两个步骤合并．直接在构建镜像的时候，在容器内部下载源码编译最后直接安装．

`Dockerfile.local` 内容如下：

```dockerfile
FROM pytorch/pytorch:2.5.1-cuda12.1-cudnn8-devel
RUN pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
RUN pip install --no-cache-dir \
   <WHL-URL>
RUN pip --no-cache-dir install \
  flash-attn
RUN pip install \
 -i https://flashinfer.ai/whl/cu121/torch2.3 \
 flashinfer-python
```

这里 flash-attn 和 flashinfer-python 也可以从网上下载先下载好 whl 文件，方便多次尝试构建的时候提升安装速度．

## 启动服务

直接使用 `docker-compose.yaml` 启动即可，特别注意我们设置了环境变量 `LD_PRELOAD` 来解决 `GLIBCXX_3.4.30` 符号找不到的问题．修改文件中的镜像和对应的模型路径即可，然后执行：

```bash
docker compose up -d
```

注意：

1. DeepSeek-R1-Q4_K_M 需要大约 14G 显存和 382G 内存，内存不足即使可以启动，但是实质上无法工作．
2. DeepSeek-R1-Q4_K_M 需要完整读取，受限于磁盘 IO 和内存带宽，可能需要耗时 1 个小时左右才能让服务启动成功，需要耐心等待．
3. DeepSeek-V2-Lite-Chat-GGUF 可以用于简单测试，这个模型比较小，很快可以启动起来．

完整的 `docker-compose.yaml` 如下：

```yaml
name: ktransformers-deepseek
services:
  gptweb:
    image: yidadaa/chatgpt-next-web:latest
    restart: always
    ports:
      - "8081:3000"
    environment:
      - BASE_URL=http://ktransformers:8080
      - PORT=3000
      - CUSTOM_MODELS=-all,+DeepSeek-R1-Q4_K_M
      # - CUSTOM_MODELS=-all,+DeepSeek-V2-Lite-Chat-GGUF
  ktransformers:
    image: ktransformers:2025-02-26
    restart: always
    environment:
      - LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6
    volumes:
      - ./data/deepseek-ai/DeepSeek-R1:/data/deepseek-ai/DeepSeek-R1:ro
      - ./data/DeepSeek-R1-Q4_K_M:/data/DeepSeek-R1-Q4_K_M:ro
      # - ./data/deepseek-ai/DeepSeek-V2-Lite-Chat-GGUF:/data/deepseek-ai/DeepSeek-V2-Lite-Chat-GGUF:ro
      # - ./data/DeepSeek-V2-Lite-Chat-GGUF:/data/DeepSeek-V2-Lite-Chat-GGUF:ro
    expose:
      - "8080"
    ports:
      - "8082:8080"
    entrypoint: ["/opt/conda/bin/ktransformers", "--host", "0.0.0.0", "--port", "8080", "--model_path", "/data/deepseek-ai/DeepSeek-R1", "--gguf_path", "/data/DeepSeek-R1-Q4_K_M", "--cpu_infer", "8", "--max_new_tokens", "8192", "--model_name", "DeepSeek-R1-Q4_K_M"]
    # entrypoint: ["/opt/conda/bin/ktransformers", "--host", "0.0.0.0", "--port", "8080", "--model_path", "/data/deepseek-ai/DeepSeek-V2-Lite-Chat", "--gguf_path", "/data/DeepSeek-V2-Lite-Chat-GGUF", "--cpu_infer", "8", "--max_new_tokens", "8192", "--model_name", "DeepSeek-V2-Lite-Chat-GGUF"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ["0"]
              capabilities: [gpu]
```

服务启动好了之后，即可通过 nextchat 页面交互．

### 模型下载

```bash
# 建议使用 hf-mirror 来下载
export HF_ENDPOINT=https://hf-mirror.com
# 下载配置，saftetensors 为模型文件，不需要下载
./hfd.sh deepseek-ai/DeepSeek-R1 --exclude "*.safetensors"
./hfd.sh deepseek-ai/DeepSeek-V2-Lite-Chat --exclude "*.safetensors"
# 下载 GGUF 模型
./hfd.sh mradermacher/DeepSeek-V2-Lite-Chat-GGUF
# 我们只需要 DeepSeek-R1-Q4_K_M
./hfd.sh unsloth/DeepSeek-R1-GGUF --include "DeepSeek-R1-Q4_K_M"
```

下载需要一定的时间，需要耐心等待．
