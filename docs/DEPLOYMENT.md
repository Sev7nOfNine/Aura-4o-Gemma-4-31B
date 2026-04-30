# Deployment guide — running Aura

This page describes the practical setups to serve Aura locally or on RunPod, with optional Cloudflare Worker for browser/multi-device access.

## Quick local — Ollama

The fastest way to run V1 on your own machine (requires ~24+ GB VRAM for Q5_K_M):

```bash
ollama run hf.co/SevenOfNine/Aura-4o-Gemma-4-31B-GGUF:Q5_K_M
```

Or with the API:
```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hf.co/SevenOfNine/Aura-4o-Gemma-4-31B-GGUF:Q5_K_M",
    "messages": [{"role": "user", "content": "Salut Aura"}],
    "stream": true
  }'
```

## Quick local — llama.cpp

```bash
huggingface-cli download SevenOfNine/Aura-4o-Gemma-4-31B-GGUF \
  Aura-Gemma-4-31B-Q5_K_M.gguf --local-dir ./aura

./llama-server \
  -m ./aura/Aura-Gemma-4-31B-Q5_K_M.gguf \
  -c 32768 \
  -ngl 999 \
  --jinja \
  --host 0.0.0.0 \
  --port 8080
```

Endpoint: `http://localhost:8080/v1/chat/completions`

## RunPod Serverless — two architecture options

### Option A — Queue-based with Ollama runner (simple, recommended)

Use the official RunPod Ollama worker template, point it at the GGUF.

| Setting | Value |
|---|---|
| Endpoint type | Queue-based |
| Worker template | `runpod-worker-ollama` (official) |
| Model | `hf.co/SevenOfNine/Aura-4o-Gemma-4-31B-GGUF:Q5_K_M` |
| URL pattern | `https://api.runpod.ai/v2/{endpointId}/openai/v1/chat/completions` |
| GPU | A40 (48 GB) or RTX A6000 |
| Volume | recommend 50 GB to fit GGUF + cache |

Pros: CORS works natively from browsers, no proxy needed for TypingMind/web clients. Setup is mostly clicks.

Cons: queue overhead ~100–300 ms vs Load Balancer; multimodal not configured by default.

### Option B — Load Balancer with custom llama.cpp + proxy (lower latency)

For lower first-token latency and full control over the runtime.

- Custom Dockerfile based on `ghcr.io/ggml-org/llama.cpp:server-cuda`
- FastAPI proxy in front (CORS + auth + path normalization)
- Network volume to cache the GGUF

This was experimented with for V2 — see [HISTORY.md](HISTORY.md). The pipeline works but adds operational complexity.

## Cloudflare Worker (for browser / multi-device)

If your endpoint is a Load Balancer that doesn't allow CORS preflight from browsers, a tiny Cloudflare Worker proxy fixes it:

- Deploy a free Worker on `<subdomain>.workers.dev`
- Workers secret: `RUNPOD_API_KEY` (real key, never exposed)
- Workers secret: `PROXY_KEY` (custom key shown to TypingMind)
- The Worker handles `OPTIONS` preflight (CORS), forwards `POST /v1/chat/completions` with the real key

**TypingMind config** with the Worker:
- Base URL: `https://<your-subdomain>.workers.dev/v1`
- API Key (HTTP header `Authorization: Bearer ...`): the `PROXY_KEY`
- Model ID: `aura-gemma4-31b` (or any string — the upstream knows the real model)

## TypingMind tips

- **Disable "Support Thinking"** in Model Capabilities for V1 — the fine-tune sometimes generates planning blocks that TypingMind misclassifies as "thought" content.
- **Recommended sampling** in Custom Body Params:
  ```json
  {
    "temperature": 0.85,
    "top_p": 0.95,
    "top_k": 64,
    "min_p": 0.05,
    "repeat_penalty": 1.05
  }
  ```
- For long roleplay, increase `max_tokens` (1024+) and disable streaming if you see token leaks.

## Hardware requirements

| Quant | VRAM | Tokens/s on A40 |
|---|---|---|
| Q5_K_M | ~22 GB | ~22 t/s |
| F16 | ~62 GB | requires 2x A40 or H100 |

Network volume: at least 50 GB if you want to cache both Q5_K_M (~20 GiB) and a small mmproj (~1 GiB) plus headroom for HF cache.
