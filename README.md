# Gonka mlnode on clore.ai - quick cheat sheet

This README is a practical checklist for running Gonka mlnode in a single-container clore.ai order (no docker-compose).

## 1) Current images (what to use)

- Blackwell (RTX 50xx / sm120, e.g. 5060/5070/5080/5090)
  - Image: `vasyagun/gonka:mlnode-3.0.11-clore-sm120-post12`
  - Digest: `sha256:436bcb6361f71683fae31fd7adbc1c908d5f16e9d9ed14eafb54e88fd8efe437`
  - Use on Blackwell GPUs only.

- Ada / Ampere (RTX 40xx / 30xx, e.g. 4090/3090)
  - Image: `vasyagun/gonka:mlnode-3.0.11-clore-ada-post12`
  - Digest: `sha256:6566f4fd2a0c1ebc00d774424faefe2535fa53d41270aac442ecf14c55ef15d8`
  - Use on Ada/Ampere GPUs.

## 2) Supported HTTP endpoints (with examples)

Base URL for clore external access:
- `http://<HOST>:<PUBLIC_PORT>` where `<PUBLIC_PORT>` is the forwarded port for container port 5000.

Basic status:
- `GET /api/v1/state`
  - `curl -sS http://<HOST>:<PORT>/api/v1/state`

Inference startup:
- `POST /api/v1/inference/up`
- `POST /api/v1/inference/up/async`
- `GET /api/v1/inference/up/status`

Example (async start):
```
curl -sS -X POST http://<HOST>:<PORT>/api/v1/inference/up/async \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct",
    "dtype": "float16",
    "max_model_len": 8192,
    "gpu_memory_utilization": 0.90,
    "tensor_parallel_size": 1,
    "pipeline_parallel_size": 1
  }'

curl -sS http://<HOST>:<PORT>/api/v1/inference/up/status
```

Logs and diagnostics (no SSH required):
- `GET /api/v1/logs/vllm?lines=200`
- `GET /api/v1/logs/uvicorn?lines=200`
- `GET /api/v1/logs/preload?lines=200`
- `GET /api/v1/diag`

Examples:
```
curl -sS http://<HOST>:<PORT>/api/v1/logs/preload?lines=200
curl -sS http://<HOST>:<PORT>/api/v1/logs/vllm?lines=200
curl -sS http://<HOST>:<PORT>/api/v1/diag
```

Health endpoint:
- `GET /health` (proxied to vLLM backend)
  - Returns 502 until vLLM is running. This is expected.

## 3) Supported environment variables (with examples)

SSH access:
- `ROOT_PASSWORD` (recommended for clore)
- `SSH_PUBLIC_KEY`

Hugging Face cache:
- `HF_HOME=/root/.cache`
- `HUGGINGFACE_HUB_CACHE=/root/.cache/hub`
- `TRANSFORMERS_CACHE=/root/.cache/hub`

Model preload:
- `MODEL_KEY` (easy selector)
- `MODEL_REPO` (explicit HF repo)
- `MODEL_REVISION` (optional HF revision)
- `PRELOAD_MODEL=1` (force preload on startup)

Supported MODEL_KEY values:
- `qwen2.5-7b` / `qwen2.5-7b-instruct`
- `qwen3-235b-fp8` / `qwen3-235b-a22b-fp8`
- `qwen3-32b-fp8`
- `qwq-32b`
- `redhat-qwen2.5-7b-w8a16` / `w8a16`

Inference tuning:
- `VLLM_ATTENTION_BACKEND=FLASHINFER` (optional)
- `INFERENCE_MAX_INSTANCES=2` (max vLLM instances)
- `VLLM_COMPAT_SERVER=0` (disable internal compatibility server on port 5000)
- `CUDA_VISIBLE_DEVICES=0,1` (optional GPU pinning)
- `NVIDIA_VISIBLE_DEVICES=all` (optional)
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` (optional)

Example (clore UI env):
```
ROOT_PASSWORD=your-password
HF_HOME=/root/.cache
HUGGINGFACE_HUB_CACHE=/root/.cache/hub
TRANSFORMERS_CACHE=/root/.cache/hub
MODEL_KEY=qwen2.5-7b-instruct
PRELOAD_MODEL=1
VLLM_ATTENTION_BACKEND=FLASHINFER
INFERENCE_MAX_INSTANCES=2
VLLM_COMPAT_SERVER=0
```

## 4) Minimal clore order setup (text only)

- Image: use one of the `post12` images above.
- Port forwarding:
  - `22/TCP` (SSH)
  - `5000/TCP` (public API)
- Environment:
  - `ROOT_PASSWORD` (required if SSH autoinstall is OFF)
  - `HF_HOME`, `HUGGINGFACE_HUB_CACHE`, `TRANSFORMERS_CACHE`
  - `MODEL_KEY` or `MODEL_REPO` (optional, for preload)
- DO NOT enable "Use SSH Autoinstall entrypoint" (it overrides our entrypoint).
- Startup script: not required for these images.

## 5) How to check the node

External checks:
```
curl -sS http://<HOST>:<PORT>/api/v1/state
curl -sS http://<HOST>:<PORT>/api/v1/diag
curl -sS http://<HOST>:<PORT>/api/v1/logs/preload?lines=200
curl -sS http://<HOST>:<PORT>/api/v1/logs/vllm?lines=200
```

Start vLLM and check status:
```
curl -sS -X POST http://<HOST>:<PORT>/api/v1/inference/up/async \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-7B-Instruct","dtype":"float16","max_model_len":8192,"gpu_memory_utilization":0.90,"tensor_parallel_size":1,"pipeline_parallel_size":1}'

curl -sS http://<HOST>:<PORT>/api/v1/inference/up/status
```

Notes:
- `state=STOPPED` is normal until vLLM starts.
- `/health` returns 502 until vLLM is running.

## 6) How to register the node (network node)

Run on the network node (Admin API):
```
curl -sS -X POST http://127.0.0.1:9200/admin/v1/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "id": "clore-4090-1",
    "host": "<PUBLIC_HOST>",
    "inference_port": <PUBLIC_PORT>,
    "poc_port": <PUBLIC_PORT>,
    "max_concurrent": 50,
    "models": {
      "Qwen/Qwen2.5-7B-Instruct": {
        "args": [
          "--tensor-parallel-size","1",
          "--pipeline-parallel-size","1",
          "--dtype","float16",
          "--gpu-memory-utilization","0.90",
          "--max-model-len","8192"
        ]
      }
    }
  }' | jq
```

Use the public host and the forwarded port for container 5000.

## 7) Common errors and fixes

- SSH timeout/reset:
  - Container may be restarting or preload is heavy. Use `/api/v1/diag` and `/api/v1/logs/preload` externally.

- `502 Bad Gateway` on `/health`:
  - vLLM is not running yet. Start it via `/api/v1/inference/up/async`.

- `/api/v1/inference/up/status` = `not_started`:
  - vLLM not started. Run the async start call.

- Preload errors in logs:
  - `Unknown MODEL_KEY` or `MODEL_REPO must be set` - fix env vars.

- `VLLM_ATTENTION_BACKEND is set to FLASHINFER, but flashinfer not installed`:
  - Remove `VLLM_ATTENTION_BACKEND` or set `XFORMERS`.

- `error while attempting to bind on address ('0.0.0.0', 5000)`:
  - Port 5000 already used by nginx. Set `VLLM_COMPAT_SERVER=0` (default in newer images).

- Port confusion:
  - Use the public port mapped to container 5000 for all HTTP API calls.
  - SSH port is separate.

If you want more diagnostics, check `/api/v1/diag` first.