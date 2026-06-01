# Project Overview

`sherpa-onnx` is a comprehensive, multi-language, and cross-platform speech AI library. It leverages ONNX Runtime for fast, local inference.

Key capabilities include:
- **Speech Recognition (ASR)** (Streaming & Offline)
- **Speech Synthesis (TTS)** (Text-to-Speech)
- **Source Separation** & **Speech Enhancement**
- **Speaker Identification** & **Diarization**
- **Voice Activity Detection (VAD)**
- **Spoken Language Identification**
- **Keyword Spotting** & **Audio Tagging**
- **Punctuation Addition**

## Technologies and Architecture

- **Core Engine:** Written in C++ (`sherpa-onnx/csrc/`).
- **Inference Backend:** ONNX Runtime (`onnxruntime`).
- **Build System:** CMake (main `CMakeLists.txt` at the root).
- **Supported Platforms:** Windows, macOS, Linux, Android, iOS, HarmonyOS, WebAssembly (Wasm).
- **Supported Architectures:** x64, x86, arm64, arm32, riscv64.
- **Language Bindings:** C, C++, Python, JavaScript, Java, C#, Kotlin, Swift, Go, Dart, Rust, Pascal.
  - Examples and specific integrations are located in directories like `python-api-examples/`, `java-api-examples/`, `flutter/`, `wasm/`, etc.

## Building and Running

The project relies heavily on CMake for building the core library and its language bindings.

### Standard C++ Build (Example)
```bash
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j4
```
*(Check the official documentation or CI workflows in `.github/workflows/` for platform-specific CMake flags and dependencies).*

### Python
The Python package is built using `setup.py`, which internally calls CMake.
```bash
pip install -e .
```

### Models
Models are generally downloaded, converted to ONNX format (e.g., via scripts in the `scripts/` directory or workflows), and then loaded at runtime. 

## Development Conventions

- **Code Style:** The C++ code style is enforced via `cpplint` (see `CPPLINT.cfg` and `scripts/check_style_cpplint.sh`).
- **CI/CD:** The project has an extensive GitHub Actions setup (`.github/workflows/`) that automatically builds binaries, tests multi-platform compatibility, and packages libraries for Android (APK, AAR), iOS (XCFramework), Python (wheels), Node.js, and more.
- **API Design:** The core functionality is exposed via a C API, which is then wrapped by various language-specific bindings. This ensures consistent behavior and easy integration across the wide range of supported languages.
