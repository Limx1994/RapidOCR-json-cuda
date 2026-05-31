# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RapidOCR-json is a Windows offline OCR program built on RapidOcrOnnx. It reads image paths or base64-encoded images from stdin, runs a three-stage neural network pipeline, and outputs JSON-formatted OCR results. Targets Win7 x64+. Supports CUDA GPU acceleration (default enabled).

## Build

Dependencies: `cpp/lib.7z` contains `onnxruntime-static` and `opencv-static` (CPU). For CUDA, download `onnxruntime-gpu` separately.

### CUDA build (VS2022, recommended)

```bat
# Prerequisites: CUDA Toolkit 11.x + cuDNN 8.x, onnxruntime-gpu in cpp/onnxruntime-gpu/
mkdir build && cd build
cmake -G "Visual Studio 17 2022" -A x64 -DOCR_OUTPUT="BIN" -DOCR_ONNX="CUDA" -DOCR_BUILD_CRT="True" ../cpp
cmake --build . --config Release
```

### CPU build (VS2019/2022)

```bat
cd cpp
# 1. Extract lib.7z: onnxruntime-static/ and opencv-static/
# 2. Run generate-vs-project.bat
# 3. Open build-win-vs2019-x64/*.sln, build Release
# 4. Copy models/ to output directory
```

`generate-vs-project.bat` defaults to CUDA (`ONNX_TYPE="CUDA"`). Change `set flag=2` to `set flag=1` on line 42 for CPU.

CMake variables:
- `OCR_ONNX`: `"CUDA"` or `"CPU"` — controls onnxruntime backend
- `OCR_OUTPUT`: `"BIN"` (exe), `"JNI"` (shared lib), `"CLIB"` (C shared lib)
- `OCR_BUILD_CRT`: `"True"` for static CRT (`/MT`), required for CUDA build

MSVC builds require `/utf-8` flag (already handled in CMakeLists.txt for MSVC).

## Running

Single image: `RapidOCR-json.exe --image_path="test.png"`

Interactive loop mode (no `--image_path`): program reads JSON lines from stdin, each with `image_path` or `image_base64` key.

GPU flag: `--GPU N` where N is GPU index (-1 = CPU only). Default: `--GPU=0`.

Python wrapper: `api/python/RapidOCR_api.py` — wraps the exe as a subprocess, communicates via stdin/stdout JSON pipes.

## Testing

No formal test suite. Manual testing via the exe or `api/python/demo1.py`.

## Architecture

Three-stage ONNX pipeline orchestrated by `OcrLite` class:

```
Image → DbNet (text detection) → AngleNet (orientation 0°/180°) → CrnnNet (text recognition) → JSON
```

CUDA: DbNet and CrnnNet use GPU when `--GPU >= 0`. AngleNet is always forced to CPU (`setGpuIndex(-1)` in OcrLite.cpp).

Key source files in `cpp/`:
- `src/main.cpp` — CLI parsing (getopt), single-shot vs interactive loop entry point
- `src/OcrLite.cpp` + `include/OcrLite.h` — pipeline orchestrator owning DbNet, AngleNet, CrnnNet
- `src/DbNet.cpp` — text region detection (DBNet), `setGpuIndex()` gated by `#ifdef __CUDA__`
- `src/AngleNet.cpp` — text orientation classification (fixed 192×48 input)
- `src/CrnnNet.cpp` — text recognition (CRNN, fixed height 48, loads character dictionary)
- `src/tools.cpp` + `include/tools.h` — global config (namespace `tool`), UTF-8 image loading, base64 decode, JSON I/O
- `include/OcrStruct.h` — data types: TextBox, Angle, TextLine, TextBlock, OcrResult
- `include/tools_flags.h` — error codes (100=success, 101=no text, 200+=errors)
- `include/main.h` — CLI long_options array and help text

Python API: `api/python/RapidOCR_api.py` — `OcrAPI` class wrapping the exe subprocess. Methods: `run(imagePath)`, `runBase64(base64str)`, `runBytes(imageBytes)`.

Multi-language configs defined in `models/configs.txt`. Detection/classification models are shared; recognition model and dictionary are per-language.
