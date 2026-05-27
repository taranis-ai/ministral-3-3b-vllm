# taranis-llm-images

Container images for serving baked-in models with `vllm`.

- Model baked into the image at build time
- Offline runtime startup
- Generic CPU and GPU images
- Model selection via build args mirrored into env vars

## Files

- `Containerfile.gpu`: generic GPU image, with configurable base image, baked model, and serve args
- `Containerfile.cpu`: generic CPU image built from source, with configurable baked model, vLLM ref, and serve args

## Defaults

- CPU and GPU default model: `cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit`
- CPU and GPU default model dir: `/models/Ministral-3-3B-Instruct-2512-AWQ-4bit`
- Default CPU serve args: `--tokenizer_mode mistral --config_format mistral --load_format mistral --enforce-eager`
- Default GPU serve args: `--tokenizer_mode mistral --config_format mistral --load_format mistral`
- CPU vLLM ref: `v0.21.0`
- GPU default base image: `vllm/vllm-openai:latest`

The images still bake the selected model at build time for offline startup. That means changing `MODEL_ID` at runtime does not fetch a new model; it only affects the env inside an image that already contains that model.

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

Build Gemma 4 E2B on CPU:

```bash
docker build \
  -f Containerfile.cpu \
  --build-arg MODEL_ID=google/gemma-4-E2B-it \
  --build-arg MODEL_DIR=/models/gemma-4-E2B-it \
  --build-arg SERVE_ARGS="--enforce-eager" \
  -t taranis-llm-images:cpu-gemma-4-e2b .
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

## Test

Default Mistral image:

```bash
curl -sS http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "/models/Ministral-3-3B-Instruct-2512-AWQ-4bit",
    "messages": [
      {
        "role": "user",
        "content": "Write a one-sentence summary of what you are."
      }
    ]
  }'
```

Gemma image:

```bash
curl -sS http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "/models/gemma-4-E2B-it",
    "messages": [
      {
        "role": "user",
        "content": "Write a one-sentence summary of what you are."
      }
    ]
  }'
```
