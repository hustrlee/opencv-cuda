# OpenCV + CUDA Wheel 构建指南

本项目旨在解决如何快速在 Docker 中构建带 CUDA 支持的 OpenCV，并生成可在同构主机上分发的 wheel 包。

> ⚠️ **注意**：  
本项目并不是一个“编译 OpenCV 的完整通用指南”。它是在 Ubuntu 24.04 / Ubuntu 22.04 / WSL2 Ubuntu 上构建的。如果你需要不同的 CUDA / 系统版本组合，请在此基础上自行调整。

## 背景与动机 (Motivation)

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
