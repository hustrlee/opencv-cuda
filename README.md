# OpenCV + CUDA Wheel 构建指南

本项目旨在解决如何快速在 Conda 虚拟环境中构建带 CUDA 支持的 OpenCV，并生成可在同构主机上分发的 wheel 包。

> ⚠️ **注意**：  
本项目并不是一个“编译 OpenCV 的完整通用指南”。它是在 Ubuntu 24.04 / WSL2 Ubuntu 24.04 上构建的。如果你需要不同的 CUDA / 系统版本组合，请在此基础上自行调整。

## 1. 背景与动机 (Motivation)

在处理高清视频流、大规模图像预处理、或需要和深度学习推理流水线结合时，CPU 很容易成为瓶颈。启用 **OpenCV 的 CUDA 模块** 可以充分利用 GPU 的并行计算能力，大幅提升性能，同时减少 CPU 占用。  

编译带 CUDA 支持的 OpenCV 主要难点包括：

- **依赖复杂**：需要同时匹配 CUDA Toolkit、cuDNN、NVIDIA 驱动、FFmpeg/GStreamer 等库，任一版本不符都可能导致失败。
- **系统兼容性**：生成的 `cv2.so` 会绑定构建时的 `glibc`、`libstdc++`、FFmpeg SONAME 等，如果目标机环境不一致，就会出现 `undefined symbol` 或缺少库文件错误。
- **编译参数复杂**：完整编译 OpenCV（含 contrib + CUDA + 多媒体支持）需要精确设置 CMake 参数（CUDA 架构、模块开关、RPATH 等）。

为了解决这些问题，本仓库提供了一种 **在 Conda 虚拟环境中编译 OpenCV + CUDA 并产出 Python wheel** 的方法。

**这样做的好处：**

- 固定构建环境，避免“在 A 机能编译，在 B 机失败”的情况。
- 构建步骤完全可复现。
- 避免在宿主机安装大量依赖，保持系统干净。
- wheel 可在满足「同构主机」条件的多台机器上直接安装，无需重复编译。
- wheel 支持在 Conda 虚拟环境中安装。

## 2. 创建编译环境

`environment.yaml`：

```yaml
name: cvbuild
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.9
  - numpy

  # 低版本 glibc，保持系统兼容性
  - sysroot_linux-64=2.28

  # CUDA 工具链
  - cuda-toolkit=12.9.1
  - cudnn=9.12

  # 编译工具链
  - gcc=12
  - gxx=12
  - cmake
  - ninja
  - make
  - pkg-config
  - nasm
  - wheel
  - auditwheel
  - patchelf

  # OpenCV 依赖
  - zlib
  - libjpeg-turbo
  - libpng
  - libtiff
  - libwebp
  - libavif
  - openblas
  - lapack
  - eigen
  - tbb
  - ffmpeg=7
```

创建编译环境：

```bash
conda env create -f environment.yaml
```

## 3. 准备源码

### 3.1 下载 opencv-python 源码

```bash
git clone --recursive https://github.com/opencv/opencv-python
```

### 3.2 Checkout 4.12.0 版本

```bash
git checkout 88
git submodule update --init --recursive
```

> ⚠️ **注意**：  
请自行检查 tags 选取正确的版本。截止到 2025-9-1，`opencv-python` 最新 tag: `88`，对应 `4.12.0` 版本。

### 3.3 下载 Video Codec SDK Header Files

启用硬件编解码功能需要 Video Codec SDK。其中，库文件随 Driver 一起提供，SDK 中只提供 stub 库。编译时只需要头文件，可以在 [Nvidia Video Codec SDK - Get Started](https://developer.nvidia.com/nvidia-video-codec-sdk/download) 下载。

### 3.4 安装 Video Codec SDK Header Files

解压 SDK，并将头文件链接到编译器的 INCLUDE PATH 中。

```bash
ln -s $PWD/video-codec-sdk/Interface/cuviddec.h $CONDA_PREFIX/targets/x86_64-linux/include/cuviddec.h
ln -s $PWD/video-codec-sdk/Interface/nvcuvid.h $CONDA_PREFIX/targets/x86_64-linux/include/nvcuvid.h
ln -s $PWD/video-codec-sdk/Interface/nvEncodeAPI.h $CONDA_PREFIX/targets/x86_64-linux/include/nvEncodeAPI.h
```

## 4. 编译

### 4.1 设置编译参数

```bash
export https_proxy=http://proxy.home:1081
export PATH="$CONDA_PREFIX/nvvm/bin:$PATH"
unset NVCC_PREPEND_FLAGS
export ENABLE_CONTRIB=1
export ENABLE_HEADLESS=1
export CMAKE_ARGS="
-DCMAKE_BUILD_TYPE=RELEASE
-DWITH_CUDA=ON
-DWITH_CUDNN=ON
-DOPENCV_DNN_CUDA=ON
-DCUDA_ARCH_BIN=75;86;89
-DCUDA_ARCH_PTX=89
-DWITH_FFMPEG=ON
-DWITH_PROTOBUF=ON
-DBUILD_PROTOBUF=ON
-DPROTOBUF_UPDATE_FILES=OFF
-DWITH_OPENCL=OFF
-DWITH_VA=OFF
-DWITH_VA_INTEL=OFF
-DWITH_V4L=OFF
-DCMAKE_PREFIX_PATH=$CONDA_PREFIX
-DCMAKE_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu
-DCMAKE_SYSROOT=$CONDA_PREFIX/x86_64-conda-linux-gnu/sysroot
-DCMAKE_INSTALL_RPATH=$CONDA_PREFIX/lib:/usr/lib/x86_64-linux-gnu:/usr/lib/wsl/lib
"
```

### 4.2 编译

`python setup.py bdist_wheel --py-limited-api=cp37` 是旧式的命令，官方建议手动编译使用新的命令：

```bash
pip wheel . -w dist -v
```

### 4.3 将常用的库打包到 Wheel

下面的这些库通常不应打包到 Wheel：

- CUDA Driver 的库
  - `libcuda.so.1`
  - `libnvcuvid.so.1`
  - `libnvidia-encode.so.1`
- ffmpeg 的库
  - `libavcodec.so.61`
  - `libavformat.so.61`
  - `libutil.so.59`
  - `libswscale.so.8`
  - `libswresample.so.5`
- Intel 的显卡加速库 `libva.so.2`
- CUDA Toolkit 和 cuDNN 的库

```bash
auditwheel repair dist/opencv_contrib_python_headless-4.12.0.88-cp39-cp39-linux_x86_64.whl \
--exclude libnppitc.so.12 \
--exclude libnppif.so.12 \
--exclude libnppicc.so.12 \
--exclude libnppidei.so.12 \
--exclude libnppial.so.12 \
--exclude libnppist.so.12 \
--exclude libnppim.so.12 \
--exclude libnppig.so.12 \
--exclude libnppc.so.12 \
--exclude libcufft.so.11 \
--exclude libcublas.so.12 \
--exclude libcublasLt.so.12 \
--exclude libcudnn.so.9 \
--exclude libcuda.so.1 \
--exclude libnvcuvid.so.1 \
--exclude libnvidia-encode.so.1 \
--exclude libavcodec.so.61 \
--exclude libavformat.so.61 \
--exclude libavutil.so.59 \
--exclude libswscale.so.8 \
--exclude libswresample.so.5 \
--exclude libva.so.2 \
--wheel-dir ./wheelhouse
```

## 5. 运行

```bash
conda create -n cv -c conda-forge \
python=3.9 \
cuda-runtime=12.9 \
cudnn=9 \
ffmpeg=7
conda activate cv
pip install opencv_contrib_python_headless-4.12.0.88-cp39-cp39-manylinux_2_38_x86_64.whl
```

## 6. 补充说明

### 6.1 关于 FFMPEG

OpenCV 4.12.0 兼容 ffmpeg 4/6/7。这里使用了 ffmpeg 7，因为最常用的视频工具链 PyAV 15 需要 ffmpeg 7。

ffmpeg 7 依赖 libprotobuf 6，与 OpenCV 不兼容，因此增加了编译参数让 OpenCV 使用自带的 protobuf：
```
-DWITH_PROTOBUF=ON
-DBUILD_PROTOBUF=ON
-DPROTOBUF_UPDATE_FILES=OFF
```

ffmpeg 7 使用了 GLIBC_2.35 及 GLIBCXX_3.4.30，因此即使安装了 `sysroot=2.28`，也无法将 Wheel 真正降到 `2_28`，最终落在 `2_35`。

### 6.2 关于 `cv.cudacodec`

编译时会自动侦测 Video Codec SDK 是否已安装。除了将头文件链接到正确的路径外，还需要让编译器侦测到“库”文件。

- Ubuntu 主机上，库文件位于 `/usr/lib/x86_64-linux-gnu/` 目录，增加编译参数：`-DCMAKE_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu`。
- WSL Ubuntu 上，库文件位于 `/usr/lib/wsl/lib/` 目录，增加编译参数：`-DCMAKE_LIBRARY_PATH=/usr/lib/wsl/lib`。

### 6.3 关于代理

在编译期间，需要从 `https://raw.githubusercontent.com` 下载一些模型文件，因此需要设置代理。

### 6.4 关于 cp37-abi3

由于 Numpy 不支持 `abi3`（稳定 ABI），因此官方建议对不同的 Python 版本编译不同的 wheel，而不是生成一个 abi3 wheel。

可以修改 `environment.yaml` 中 Python 的版本，根据需要的编译不同的版本。
