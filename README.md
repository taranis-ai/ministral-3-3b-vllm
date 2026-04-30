# ministral-3-3b-vllm

Container images for serving `unsloth/Ministral-3-3B-Base-2512-bnb-4bit` with `vllm`.

- Model baked into the image at build time
- Offline runtime startup
- Multi-stage builds
- Separate CPU and GPU variants

## Files

- `Containerfile.gpu`: GPU-oriented multi-stage image based on `vllm/vllm-openai:latest`
- `Containerfile.cpu`: CPU-oriented multi-stage image based on `astral/uv:python3.12-trixie-slim`, building vLLM from source

## Build

```bash
docker build -f Containerfile.gpu -t ministral-3-3b-vllm:gpu .
docker build -f Containerfile.cpu -t ministral-3-3b-vllm:cpu .
```

The GPU image uses `vllm/vllm-openai` for both builder and runtime stages and only carries the baked model into the final image.

The CPU image builds vLLM from source in the builder stage and copies only the built virtualenv and baked model into the runtime stage. Upstream vLLM's x86 CPU guidance currently recommends source builds instead of relying on a prebuilt x86 CPU wheel.

Override the baked-in model if needed:

```bash
docker build \
  -f Containerfile.gpu \
  --build-arg MODEL_ID=unsloth/Ministral-3-3B-Base-2512-bnb-4bit \
  --build-arg MODEL_DIR=/models/Ministral-3-3B-Base-2512-bnb-4bit \
  -t ministral-3-3b-vllm:gpu .
```

## Run

GPU:

```bash
docker run --rm -p 8000:8000 --gpus all ministral-3-3b-vllm:gpu
```

CPU:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  --cap-add SYS_NICE \
  --shm-size=4g \
  -p 8000:8000 \
  ministral-3-3b-vllm:cpu
```

## Test

```bash
curl -sS http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "/models/Ministral-3-3B-Base-2512-bnb-4bit",
    "messages": [
      {
        "role": "user",
        "content": "Write a one-sentence summary of what you are."
      }
    ]
  }'
```
