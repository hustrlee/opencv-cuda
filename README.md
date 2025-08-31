# OpenCV + CUDA Wheel 构建指南

本项目旨在解决如何快速在 Docker 中构建带 CUDA 支持的 OpenCV，并生成可在同构主机上分发的 wheel 包。

> ⚠️ **注意**：  
本项目并不是一个“编译 OpenCV 的完整通用指南”。它是在 Ubuntu 24.04 / Ubuntu 22.04 / WSL2 Ubuntu 上构建的。如果你需要不同的 CUDA / 系统版本组合，请在此基础上自行调整。

## 1. 背景与动机 (Motivation)

很多人使用 OpenCV 时只依赖 CPU 版本，但在处理高清视频流、大规模图像预处理、或需要和深度学习推理流水线结合时，CPU 很容易成为瓶颈。启用 **OpenCV 的 CUDA 模块** 可以充分利用 GPU 的并行计算能力，大幅提升性能，同时减少 CPU 占用。  

然而，编译带 CUDA 支持的 OpenCV 并不简单，主要难点包括：

- **依赖复杂**：需要同时匹配 CUDA Toolkit、cuDNN、NVIDIA 驱动、FFmpeg/GStreamer 等库，任一版本不符都可能导致失败。
- **系统兼容性**：生成的 `cv2.so` 会绑定构建时的 `glibc`、`libstdc++`、FFmpeg SONAME 等，如果目标机环境不一致，就会出现 `undefined symbol` 或缺少库文件错误。
- **编译耗时与参数复杂**：完整编译 OpenCV（含 contrib + CUDA + 多媒体支持）可能耗时很久，还需要精确设置 CMake 参数（CUDA 架构、模块开关、RPATH 等）。
- **Python ABI 问题**：若不使用 `abi3`，每个 Python 版本都要单独编译 wheel，维护负担很重。

为了解决这些问题，本仓库提供了一种 **在 Docker 中编译 OpenCV + CUDA 并产出 Python wheel** 的方法。

**这样做的好处：**

- 固定构建环境，避免“在 A 机能编译，在 B 机失败”的情况。
- 构建步骤完全可复现，任何人只需 `docker build` 就能得到相同结果。
- 避免在宿主机安装大量依赖，保持系统干净。
- wheel 可在满足「同构主机」条件的多台机器上直接安装，无需重复编译。
- wheel 支持在 Conda 虚拟环境中安装。

## 2. 宿主机环境要求（Prerequisites）

- Ubuntu 22.04 / 24.04 / Windows WSL2 Ubuntu
- 安装 Nvidia Driver
- [Driver 版本满足 CUDA Toolkit 的要求](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

    |    CUDA Toolkit    | Toolkit Driver Version |
    |:------------------:|:----------------------:|
    |    CUDA 13.0 GA    |      >=580.65.06       |
    | CUDA 12.9 Update 1 |      >= 575.57.08      |

- [安装 Nvidia Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

## ngc docker 镜像

[Nvidia 提供了 CUDA 预装镜像](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/cuda)，免除了安装 CUDA Toolkit 和 cuDNN 的麻烦。使用 `cuda:xx.x.x-cudnn-devel-ubuntu2x.04` 镜像。例如：`cuda:12.9.1-cudnn-devel-ubuntu24.04`。

```bash
docker pull nvcr.io/nvidia/cuda:12.9.1-cudnn-devel-ubuntu24.04
```

### 运行容器

```bash
docker run --name cvcuda -it --gpus all \
    -e NVIDIA_DRIVER_CAPABILITIES=compute,video,utility \
    nvcr.io/nvidia/cuda:12.9.1-cudnn-devel-ubuntu24.04
```

`-e NVIDIA_DRIVER_CAPABILITIES=compute,video,utility` 这个参数至关重要，默认情况下容器并不会启用视频编解码相关功能（NVENC/NVDEC），需要明确开启 `video`。

检查 `/usr/lib/x86_64-linux-gnu/libnvcuvid.so.1` 和 `usr/lib/x86_64-linux-gnu/libnvidia-encode.so.1` 是否存在判断硬件编解码功能是否成功开启。

## 准备源码

### 下载 opencv-python 源码

```bash
git clone --recursive https://github.com/opencv/opencv-python
```

### Checkout 4.12.0 版本

```bash
git checkout 88
```

> ⚠️ **注意**：  
请自行检查 tags 选取正确的版本。截止到 2025-9-1，最新的稳定版本是 `4.12.0.88` 对应 tag: `88`。

### 将源码拷贝到容器中

为了避免污染源码（可能会多次编译），选择将源码拷贝到容器。

```bash
docker cp opencv-python/ cvcuda:/opencv-python/
```

### 下载 Video Codec SDK Header Files

启用硬件编解码功能需要 Video Codec SDK。其中，库文件是封源的，随 Driver 一起提供。[运行容器](#运行容器) 时启用 `video` 功能，就是把编解码库映射到容器内部。

编译时只需要头文件，可以在 [Nvidia Video Codec SDK - Get Started](https://developer.nvidia.com/nvidia-video-codec-sdk/download) 下载。

> ⚠️ **注意**：下载 Video Codec SDK 需要登录 Nvidia 账户。

### 安装 Video Codec SDK Header Files

将 SDK 拷贝到容器中，并在 `/usr/local/cuda/include` 中建立符号链接。

```bash
ln -s /video_codec_interface/Interface/cuviddec.h /usr/local/cuda/include/cuviddec.h
ln -s /video_codec_interface/Interface/nvcuvid.h /usr/local/cuda/include/nvcuvid.h
ln -s /video_codec_interface/Interface/nvEncodeAPI.h /usr/local/cuda/include/nvEncodeAPI.h
```

> 这里假设将 sdk 拷贝到容器内的 `/video_codec_interface` 目录中。

## 准备编译环境

### 安装系统工具及编译工具链

```bash
apt update && DEBIAN_FRONTEND=noninteractive apt install -y \
    curl wget vim build-essential cmake ninja-build git \ 
    pkg-config ca-certificates software-properties-common
```

### 安装 Conda

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ~/Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

### 创建编译虚拟环境

```bash
conda create -n opencv-cuda python=3.9 numpy
conda install -c conda-forge \
      pkg-config ffmpeg=7 \
      libjpeg-turbo libpng libtiff libwebp openexr libavif \
      openblas lapack eigen \
      tbb
```