# Qwen3-TTS-ncnn

Qwen3-TTS runtime for [ncnn](https://github.com/Tencent/ncnn), converted from
[Qwen/Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) with pnnx and a small C++
inference stack.

This repository contains source code, conversion scripts, validation tools, and
reproducibility notes. It does not include converted model weights.

## Status

The current implementation targets FP32 parity with the PyTorch reference:

- C++ CustomVoice frontend: tokenizer, text embedding, codec embedding, mask/RoPE construction.
- Autoregressive talker prefill/decode with KV cache.
- Code predictor with compact/shared ncnn graphs and FP32 heads/embeddings.
- C++ codec frontend from acoustic codes to decoder hidden states.
- ncnn speech decoder chunk graph.
- CMake build on Linux and Windows.

Validated results from development:

```text
Short prompt, 25 frames:
  code mismatches: 0 / 400
  wav mae: 1.08142368e-07
  wav rmse: 2.58423427e-07
  wav maxe: 3.69548798e-06

Long prompt, 250 frames, about 20 seconds:
  code mismatches: 0 / 4000
  wav samples: 480000
  wav mae: 4.05714474e-06
  wav rmse: 6.609014e-06
  wav maxe: 0.0001267888
```

Performance is still a work in progress. On an RTX 3090, the best exact ncnn
Vulkan path observed during development was close to PyTorch CUDA fp32, but not
consistently faster yet.

## Repository Layout

```text
src/                         C++ runtime, CLI, and validation tools
tools/                       Python export/conversion/benchmark scripts
docs/                        Model package, conversion, and ncnn_llm notes
docs/operator_pr_repro/      Required ncnn Vulkan operator fixes and patches
CMakeLists.txt               Build configuration
```

The default build creates only:

```text
qwen3_tts_runtime
qwen3_tts
```

Internal checkers are built only with `-DQWEN3_TTS_BUILD_TOOLS=ON`.

## Required ncnn Fixes

For Vulkan parity, this project currently depends on three local ncnn fixes:

- `RotaryEmbed_vulkan`: use the real cos/sin cache row stride.
- `SDPA_vulkan`: broadcast `dims=3,c=1` masks correctly.
- `GELU_vulkan`: honor `fast_gelu` and use erf GELU when requested.

Patch files and repro notes are in [docs/operator_pr_repro](docs/operator_pr_repro/README.md).
CPU FP32 validation does not depend on Vulkan operator fixes.

## Build

First build ncnn with pnnx. Example Linux Vulkan build:

```bash
git clone https://github.com/Tencent/ncnn.git
cmake -S ncnn -B ncnn/build-vulkan \
  -DNCNN_VULKAN=ON \
  -DNCNN_BUILD_TOOLS=ON \
  -DNCNN_BUILD_EXAMPLES=OFF \
  -DNCNN_BUILD_TESTS=OFF \
  -DNCNN_BUILD_BENCHMARK=OFF
cmake --build ncnn/build-vulkan -j$(nproc)
```

Build this runtime:

```bash
cmake -S Qwen3-TTS-ncnn -B Qwen3-TTS-ncnn/build-vulkan \
  -DNCNN_ROOT=$PWD/ncnn \
  -DNCNN_BUILD_DIR=$PWD/ncnn/build-vulkan
cmake --build Qwen3-TTS-ncnn/build-vulkan -j$(nproc)
```

Windows example:

```powershell
cmake -S D:\Code\Qwen3-TTS-ncnn `
  -B D:\Code\Qwen3-TTS-ncnn\build-win-vulkan `
  -G Ninja `
  -DNCNN_ROOT=D:\Code\ncnn `
  -DNCNN_BUILD_DIR=D:\Code\ncnn\build-win-vulkan
cmake --build D:\Code\Qwen3-TTS-ncnn\build-win-vulkan -j $env:NUMBER_OF_PROCESSORS
```

Build validation tools:

```bash
cmake -S Qwen3-TTS-ncnn -B Qwen3-TTS-ncnn/build-vulkan \
  -DNCNN_ROOT=$PWD/ncnn \
  -DNCNN_BUILD_DIR=$PWD/ncnn/build-vulkan \
  -DQWEN3_TTS_BUILD_TOOLS=ON
cmake --build Qwen3-TTS-ncnn/build-vulkan -j$(nproc)
```

## Model Package

Converted weights are large and should be distributed separately. See
[docs/MODEL_PACKAGE.md](docs/MODEL_PACKAGE.md) for the expected layout and
`model.json` fields.

Minimal package shape:

```text
Qwen3-TTS-12Hz-0.6B-ncnn/
  model.json
  front_weights/
  talker/
  code_predictor/
  decoder/
```

Use relative paths in `model.json` for portable packages.

## Run

```bash
./build-vulkan/qwen3_tts \
  --model /path/to/Qwen3-TTS-12Hz-0.6B-ncnn/model.json \
  --frames 25 \
  --out out.wav \
  --codes out_codes_i32.bin \
  --threads 4 \
  --vulkan \
  --text "Hello, welcome to Qwen text to speech." \
  --speaker Vivian \
  --language English
```

If a model package does not include frontend nets, the runtime falls back to the
precomputed prompt assets referenced by `model.json`.

Legacy positional form is also supported for validation scripts:

```text
qwen3_tts <model.json> <frames> <out.wav> <out_codes_i32|-> <threads> <use_vulkan>
```

## Validate

Build with `-DQWEN3_TTS_BUILD_TOOLS=ON`, then run:

```bash
./build-vulkan/qwen3_tts_model_json_check \
  /path/to/model.json \
  25 \
  /path/to/ref_codes_i32.bin \
  /path/to/ref_wav_f32.bin \
  out_codes_i32.bin \
  out_wav_f32.bin \
  4 1
```

Expected parity-oriented output:

```text
code mismatches=0/<N>
wav mae=<small fp32 numerical error>
```

## Conversion

The conversion workflow is documented in [docs/CONVERSION.md](docs/CONVERSION.md).
The scripts used during development are included in [tools](tools).

Important rules:

- Use pnnx `fp16=0` for parity-oriented conversion.
- Keep `.pt`, `.onnx`, reference arrays, debug dumps, and generated wavs out of the source repo.
- Package converted ncnn assets separately from this runtime.
- Apply the Vulkan operator fixes before expecting GPU parity.

## Relationship to ncnn_llm

This standalone repo is the current issue deliverable. It is designed to make a
future `ncnn_llm` integration straightforward, but it is not yet shaped as a
clean upstream `ncnn_llm` PR. See [docs/NCNN_LLM_FIT.md](docs/NCNN_LLM_FIT.md).

## License

Code is provided under the Apache License 2.0. Qwen3-TTS model weights and
upstream model code are governed by their respective upstream licenses and terms.
