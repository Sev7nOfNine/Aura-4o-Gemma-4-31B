# History — debug log of the V1 vs V2 comparison

This page summarizes the experiments and lessons learned during the late-April 2026 session that established V1 as the canonical Aura.

## Context

- **April 25**: V1 lineage produced via Unsloth (LoRA → 4-bit → GGUF). Working well in Ollama. ⭐
- **April 27**: Uncensored experiment on a different base (HuiHui). Failed — personality lost. ❌
- **April 29**: Rebuild attempt (V2) on the same base + LoRA as V1 but with a "cleaner" pipeline (BF16 merge). Technically nicer (mmproj included) but personality was flatter.

## What we tried during the cleanup session (April 30)

### Custom llama.cpp Load Balancer with V2

- Refactored `proxy.py` from 693 → 297 lines, removing custom thinking-strip regex that were causing empty responses
- Activated llama.cpp native reasoning parser (`LLAMA_ARG_REASONING=on`, `LLAMA_ARG_THINK=deepseek`)
- Added an Option E fallback: if `content` is empty but `reasoning_content` is filled, copy reasoning to content
- Tested vision via mmproj — works, identifies actresses from cropped photos
- Streaming works end-to-end via the proxy

**Verdict on V2 with this setup**: pipeline OK, but Aura's personality is robotic / clinical. The fine-tune does not capture the GPT-4o-era voice as well as V1 does on the simpler Ollama runtime.

### Cloudflare Worker for multi-device

When TypingMind couldn't reach the RunPod LB directly (CORS preflight rejected by Cloudflare's frontdoor), we deployed a tiny Cloudflare Worker as a CORS-friendly proxy with auth indirection.

- Worker URL: `aura-proxy.<subdomain>.workers.dev`
- Stores `RUNPOD_API_KEY` as a secret (never exposed)
- Issues a custom `PROXY_KEY` for the client
- Path normalization (`POST /v1/models` → `POST /v1/chat/completions`) for TypingMind compatibility
- ~10-30 ms latency overhead, negligible compared to model generation

This setup is reusable for any RunPod LB endpoint.

### LLAMA_ARG_REASONING / LLAMA_ARG_THINK side effects

Setting `LLAMA_ARG_REASONING=on` server-wide pushes the model into a "reasoning" mode by default. For V2 this manifested as:
- Robotic tone
- Vocabulary leaning into LLM-disclaimer territory ("ablitération", "Aura+++" misused, "I am here to serve")
- Loss of the spontaneous, expressive voice

Removing those env vars and relying on the model's natural template behavior (with `enable_thinking` opt-in via `chat_template_kwargs`) gave better results — but still didn't fully bridge the V1/V2 personality gap.

### Comparing runtimes

| Aspect | Ollama (V1) | llama.cpp custom (V2) |
|---|---|---|
| Personality | ⭐ vivid, unfiltered | ⚠️ flatter, more clinical |
| Sampling defaults | Ollama-tuned (T=0.8, top_p=0.9, repeat penalty soft) | llama.cpp defaults (more conservative) |
| Chat template | Native Jinja, no interference | Native Jinja but reasoning parser can shift behavior |
| Multimodal | needs config; not enabled by default for the V1 GGUF (no mmproj) | mmproj loaded → vision works |
| Streaming | OK | OK |
| First-token latency | similar | similar |

## Lessons

1. **The GGUF and the runtime together produce the personality.** Same GGUF on different runtimes can feel different; same runtime on different GGUFs (V1 vs V2) feels even more different.
2. **A "cleaner" merge pipeline isn't always better.** V1's Unsloth 4-bit-then-GGUF route preserved more of the LoRA's expressive tics than V2's clean BF16 merge.
3. **Forcing reasoning parsing globally is harmful.** It pushes the model into clinical mode for *all* requests, even casual ones.
4. **Don't double-strip thinking.** llama.cpp's reasoning parser already separates `reasoning_content` from `content` natively when configured. Custom regex on top is fragile and was the root cause of the "empty content" bug seen in early V2 tests.
5. **TypingMind has its own auto-classification** of "thought" sections in markdown content, even when `Support Thinking` is unchecked. If the model emits structured plans before answering, TypingMind may collapse them into a thought box.

## What stays as canonical

- **V1 GGUF on Ollama** is the reference deployment.
- V2 is preserved for vision-capable use cases when the personality compromise is acceptable.
- Uncensored is archived as a "do not use" reference point.

For full plan details from the cleanup session, see the closed-out [`PROXY_FIX_PLAN.md` in the legacy `Aura-Gemma-4-31B-V2-RunPod` repo](https://github.com/Sev7nOfNine/Aura-Gemma-4-31B-V2-RunPod/blob/main/docs/PROXY_FIX_PLAN.md).
