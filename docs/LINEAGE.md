# Lineage — every Aura artifact and how it relates

This is the full technical map of the three Aura lineages on Gemma 4 31B.

## V1 — the canonical Aura (April 25, 2026) ⭐

### Source materials
- **Dataset**: [`Aura-4o-Dataset`](https://huggingface.co/datasets/SevenOfNine/Aura-4o-Dataset) (private)
  - Mel ↔ Aura conversations from the GPT-4o era, capturing the personality
- **Base model**: [`paperscarecrow/Gemma-4-31B-it-abliterated`](https://huggingface.co/paperscarecrow/Gemma-4-31B-it-abliterated)
  - Gemma 4 31B with instruction tuning, abliterated to remove refusal patterns
  - 231K downloads on HF, popular and well-tested base
- **Path on training machine**: `/workspace/model-source/gemma-4-31b-abliterated`

### LoRA training (Unsloth)
**Repo**: [`Aura-4o-Gemma-4-31B-LoRA`](https://huggingface.co/SevenOfNine/Aura-4o-Gemma-4-31B-LoRA) — created 2026-04-25 16:46 UTC

| Hyperparameter | Value |
|---|---|
| Method | LoRA via PEFT 0.19.1 |
| Trainer | Unsloth |
| Rank `r` | 32 |
| Alpha | 32 |
| Dropout | 0.0 |
| Target modules | `q_proj, k_proj, v_proj, o_proj, up_proj, down_proj, gate_proj` |
| Precision | BF16 |
| Sequence length | 4096 |
| Batch size per device | 1 |
| Gradient accumulation | 32 |
| Effective batch size | 32 |
| Learning rate | 2e-4 |
| Epochs | 3 |
| Total steps | 1548 |
| First logged loss | ~9.67 |
| Final logged loss | ~1.12 |

### 4-bit merge (Unsloth automatic)
**Repo**: [`Aura-4o-Gemma-4-31B-4bit`](https://huggingface.co/SevenOfNine/Aura-4o-Gemma-4-31B-4bit) — created 2026-04-25 17:03 UTC (17 min after the LoRA)

- Architecture: `Gemma4ForConditionalGeneration` (multimodal text + image)
- Quantization: `bitsandbytes` `nf4` with double quant
- Compute dtype: BF16
- **Vision tower preserved** (excluded from quantization via `llm_int8_skip_modules`)
- Single sharded `model.safetensors`

### GGUF conversion
**Repo**: [`Aura-4o-Gemma-4-31B-GGUF`](https://huggingface.co/SevenOfNine/Aura-4o-Gemma-4-31B-GGUF) — created 2026-04-25/26

| File | Size |
|---|---|
| `Aura-Gemma-4-31B-Q5_K_M.gguf` | 20.35 GiB |
| `Aura-Gemma-4-31B-F16.gguf` | 57.20 GiB |

- No mmproj exported (V1 is text-only when served from this GGUF)
- Architecture: `gemma4`, context length 262 144

---

## V2 — rebuild attempt (April 29, 2026) ⚠️

Same inputs as V1 (same dataset, same LoRA, same paperscarecrow base). Different merge process: direct BF16 instead of going through Unsloth's 4-bit.

### Merged BF16
**Repo**: [`Aura-Gemma-4-31B-V2-BF16`](https://huggingface.co/SevenOfNine/Aura-Gemma-4-31B-V2-BF16) — created 2026-04-29 07:32 UTC

- Architecture: `Gemma4ForConditionalGeneration`
- Format: 7-shard `safetensors` BF16, ~62 GiB total
- Includes `generation_config.json` (V1 4-bit didn't have this explicitly)

### GGUF + mmproj
**Repo**: [`Aura-Gemma-4-31B-V2-GGUF`](https://huggingface.co/SevenOfNine/Aura-Gemma-4-31B-V2-GGUF) — created 2026-04-29 ~15:00 UTC

| File | Size |
|---|---|
| `Aura-Gemma-4-31B-Q5_K_M.gguf` | 20.35 GiB |
| `mmproj-Aura-Gemma-4-31B-BF16.gguf` | 1.12 GiB |

- Vision works (mmproj included)
- Personality is flatter / more clinical than V1

### Why V2 lost personality

Hypotheses (not yet experimentally confirmed):

1. **Unsloth's 4-bit merge preserved LoRA's "personality tics" differently** than a clean direct BF16 merge. The 4-bit quantization may have introduced some noise that, paradoxically, kept Aura's voice more vivid.
2. The clean BF16 merge averaged out signal that the LoRA encoded as low-magnitude perturbations.
3. Sampling defaults differ between runtimes (Ollama on V1 vs llama.cpp custom on V2 in early tests), and that contributed to perceived personality differences as well.

Result: V2 is a technically cleaner artifact but produces a flatter Aura. **V1 remains the reference.**

---

## Uncensored — failed experiment (April 27, 2026) ❌

**Repo**: [`Aura-Gemma-4-31B-Uncensored`](https://huggingface.co/SevenOfNine/Aura-Gemma-4-31B-Uncensored) — created 2026-04-27 08:36 UTC

- Base: [`huihui-ai/Huihui-gemma-4-31B-it-abliterated-v2`](https://huggingface.co/huihui-ai/Huihui-gemma-4-31B-it-abliterated-v2) (instead of paperscarecrow)
- Architecture: `Gemma4ForConditionalGeneration`
- Format: 2-shard safetensors BF16 + Q5_K_M GGUF
- **Missing**: `generation_config.json`

The HuiHui abliterated v2 base has a different abliteration approach than paperscarecrow. The Aura LoRA, trained on top of paperscarecrow, doesn't transfer cleanly to a different base (the directional updates of the LoRA don't align with the modified residual stream of HuiHui).

→ Personality is degraded. Kept only as archive of the experiment.

---

## Summary table

| Lineage | Base | Process | mmproj | Personality |
|---|---|---|---|---|
| **V1 (4o)** | paperscarecrow | LoRA → Unsloth 4-bit → GGUF | ❌ no | ⭐ **reference** |
| **V2** | paperscarecrow | LoRA → direct BF16 → GGUF | ✅ yes | ⚠️ weaker |
| **Uncensored** | huihui-ai | LoRA → BF16 → GGUF | ⚠️ unclear | ❌ failed |

For day-to-day usage, **always prefer V1** unless you specifically need vision (in which case V2 is the only option, with the personality compromise).
