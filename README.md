# taranis-llm-images

Container images for serving baked-in models with `vllm` on GPU and `llama.cpp` on CPU.

Runtime arguments passed after the image name are forwarded to the image's server process.

- Model baked into the image at build time
- Offline runtime startup
- Generic CPU and GPU images
- Model selection via build args mirrored into env vars

## Files

- `Containerfile.gpu`: generic GPU image, with configurable base image, baked model, and serve args
- `Containerfile.cpu`: generic CPU image based on `ghcr.io/ggml-org/llama.cpp:server`, with a baked GGUF model

## Defaults

- CPU default model repo: `unsloth/Qwen3.5-2B-GGUF`
- CPU default GGUF file: `Qwen3.5-2B-Q4_K_M.gguf`
- CPU default model dir: `/models/unsloth-Qwen3.5-2B-GGUF`
- Default GPU serve args: `--tokenizer_mode mistral --config_format mistral --load_format mistral`
- GPU default model: `cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit`
- GPU default model dir: `/models/Ministral-3-3B-Instruct-2512-AWQ-4bit`
- GPU default base image: `vllm/vllm-openai:latest`

The images still bake the selected model at build time for offline startup. That means changing `MODEL_ID` at runtime does not fetch a new model; it only affects the env inside an image that already contains that model.

For the CPU image, `MODEL_ID` is the Hugging Face repo and `MODEL_FILE` is the exact GGUF file to bake into the image. Only that file is downloaded during `docker build`, and `llama.cpp` serves it directly at runtime with no network access.

## Build

```bash
docker build -f Containerfile.gpu -t taranis-llm-images:gpu .
docker build -f Containerfile.cpu -t taranis-llm-images:cpu .
```

Build Gemma 4 E2B on GPU:

```bash
docker build \
  -f Containerfile.gpu \
  --build-arg VLLM_BASE_IMAGE=vllm/vllm-openai:gemma4 \
  --build-arg MODEL_ID=google/gemma-4-E2B-it \
  --build-arg MODEL_DIR=/models/gemma-4-E2B-it \
  --build-arg SERVE_ARGS="" \
  -t taranis-llm-images:gpu-gemma-4-e2b .
```

Build the default CPU image explicitly:

```bash
docker build \
  -f Containerfile.cpu \
  --build-arg MODEL_ID=unsloth/Qwen3.5-2B-GGUF \
  --build-arg MODEL_FILE=Qwen3.5-2B-Q4_K_M.gguf \
  --build-arg MODEL_DIR=/models/unsloth-Qwen3.5-2B-GGUF \
  -t taranis-llm-images:cpu-qwen3.5-2b .
```

## Run

Default GPU image:

```bash
docker run --rm -p 8000:8000 --gpus all taranis-llm-images:gpu
```

Default CPU image:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  --cap-add SYS_NICE \
  --shm-size=4g \
  -p 8000:8000 \
  taranis-llm-images:cpu
```

Pass extra `llama-server` arguments at runtime:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  --cap-add SYS_NICE \
  --shm-size=4g \
  -p 8000:8000 \
  taranis-llm-images:cpu \
  -c 4096
```

The CPU image always uses the GGUF baked at build time. Clients select that model in API requests with the baked alias `unsloth/Qwen3.5-2B-GGUF`.

## Test

Default CPU image:

```bash
curl -sS http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "unsloth/Qwen3.5-2B-GGUF",
    "messages": [
      {
        "role": "user",
        "content": "Write a one-sentence summary of what you are."
      }
    ]
  }'
```

List models on CPU:

```bash
curl -sS http://localhost:8000/v1/models
```
