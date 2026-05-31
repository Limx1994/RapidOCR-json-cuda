# RapidOCR-json 构建指南

本文档帮助如何在 Windows x64 上编译 RapidOCR-json 。

本文参考了 RapidAI官方的[编译说明](https://github.com/RapidAI/RapidOcrOnnx/blob/main/BUILD.md) 。

## 1. 前期准备

### 1.1 需要安装的工具：

- [Visual Studio 2019](https://learn.microsoft.com/zh-cn/visualstudio/releases/2019/release-notes) 或 [Visual Studio 2022](https://learn.microsoft.com/zh-cn/visualstudio/releases/2022/release-notes)（Community，需安装 "C++ 桌面开发" 工作负载）
- [CMake](https://cmake.org/download/) (Windows x64 Installer)

### 1.2 CUDA 构建额外需要（可选）：

如需 GPU 加速版本，还需安装：

- [CUDA Toolkit 11.x](https://developer.nvidia.com/cuda-toolkit)
- [cuDNN 8.x](https://developer.nvidia.com/cudnn)（需注册 NVIDIA 开发者账号）
- [onnxruntime-gpu](https://github.com/microsoft/onnxruntime/releases) 对应版本，解压到 `cpp/onnxruntime-gpu/windows-x64/`

## 2. 构建项目

1. 解压 `lib.7z` ，将其中两个文件夹 `onnxruntime-static` 和 `opencv-static` 解压到 `cpp/` 中。（注意，是直接放在 `cpp/opencv-static` ，而不是 `cpp/lib/opencv-static` ！）
2. 动动手指，点击运行 `generate-vs-project.bat` ，静等文件生成。
3. 打开 `build-win-vs2019-x64` ，用vs2019打开 `RapidOcrOnnx` 。
4. vs2019上方控制栏，将 `Debug` 改为 `Release` 。
5. 解决方案资源管理器 → ALL_BUILD → 常规：
  - 输出目录 → 改为 `$(ProjectDir)/Release` 。
  - 目标文件名 → 改为 `RapidOCR-json` 或你喜欢的名字。
6. 解决方案资源管理器 → ALL_BUILD → 调试：
  - 工作目录 → 改为 `$(ProjectDir)/Release` 。
7. 解决方案资源管理器 → ALL_BUILD → 高级：
  - 字符集 → 改为 `使用Unicode字符集` 。应用，关闭此页面。
8. 解决方案资源管理器 → RapidOcrOnnx → 常规：
  - 目标文件名 → 改为 `RapidOCR-json` 或你喜欢的名字。
9. 解决方案资源管理器 → RapidOcrOnnx → 高级：
  - 字符集 → 改为 `使用Unicode字符集` 。应用，关闭此页面。
10. F5尝试编译。如果有 `成功*个，失败0个……` 那就成功了。如果一个黑窗口一闪而过，那就正常。
11. 项目根目录的 `models.7z` ，解压，将 `models` 文件夹整个放到 `Release` 目录下。
12. 随便拿一张测试图片，命名为 `test.png` 之类，放到 `Release` 目录下。
13. 命令行启动程序并调试： `RapidOCR-json.exe --models=models --det=ch_PP-OCRv3_det_infer.onnx --cls=ch_ppocr_mobile_v2.0_cls_infer.onnx --rec=ch_PP-OCRv3_rec_infer.onnx --keys=ppocr_keys_v1.txt --image=test.png`

## 3. CUDA GPU 加速构建（VS2022，推荐）

如需使用 NVIDIA GPU 加速 OCR 推理，可按以下步骤构建 CUDA 版本。

### 3.1 前置条件

确保已安装以下组件：

- Visual Studio 2022（C++ 桌面开发工作负载）
- CUDA Toolkit 11.x
- cuDNN 8.x
- onnxruntime-gpu 解压到 `cpp/onnxruntime-gpu/windows-x64/`

### 3.2 使用 CMake 命令行构建

```bat
cd cpp
mkdir build-cuda && cd build-cuda
cmake -G "Visual Studio 17 2022" -A x64 -DOCR_OUTPUT="BIN" -DOCR_ONNX="CUDA" -DOCR_BUILD_CRT="True" ..
cmake --build . --config Release
```

### 3.3 使用 `generate-vs-project.bat`

该脚本默认已启用 CUDA（`ONNX_TYPE="CUDA"`），生成 VS2019 项目。如需切换为 CPU 构建，将第 42 行的 `set flag=2` 改为 `set flag=1`。

### 3.4 部署

构建完成后，需将以下文件复制到输出目录：

- `models/` 目录（含模型文件和字典）
- onnxruntime DLL（`onnxruntime.dll`、`onnxruntime_providers_cuda.dll` 等）
- CUDA/cuDNN 运行时 DLL（`cudnn64_8.dll`、`cublas64_11.dll`、`cudart64_110.dll` 等）
- `zlibwapi.dll`

### 3.5 运行

```bat
RapidOCR-json.exe --GPU=0 --image_path="test.png"
```

使用 `--GPU=-1` 可切换回 CPU 模式。
