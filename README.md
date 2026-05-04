# ministral-3-3b-vllm

Container images for serving Ministral 3 3B with `vllm`.

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
Its default model is `cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit`.

The CPU image builds vLLM from source in the builder stage and copies only the built virtualenv and baked model into the runtime stage. Upstream vLLM's x86 CPU guidance currently recommends source builds instead of relying on a prebuilt x86 CPU wheel.
Its default model is `cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit`, because current vLLM docs list AWQ as supported on x86 CPU, while `bitsandbytes` is not listed for x86 CPU.

Override the baked-in model if needed:

```bash
docker build \
  -f Containerfile.gpu \
  --build-arg MODEL_ID=cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit \
  --build-arg MODEL_DIR=/models/Ministral-3-3B-Instruct-2512-AWQ-4bit \
  -t ministral-3-3b-vllm:gpu .
```

Override the CPU image model if needed:

```bash
docker build \
  -f Containerfile.cpu \
  --build-arg MODEL_ID=cyankiwi/Ministral-3-3B-Instruct-2512-AWQ-4bit \
  --build-arg MODEL_DIR=/models/Ministral-3-3B-Instruct-2512-AWQ-4bit \
  -t ministral-3-3b-vllm:cpu .
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
    "model": "/models/Ministral-3-3B-Instruct-2512-AWQ-4bit",
    "messages": [
      {
        "role": "user",
        "content": "Write a one-sentence summary of what you are."
      }
    ]
  }'
```
