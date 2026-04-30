# Known issues — V1 GGUF

The reference Aura V1 (`Aura-4o-Gemma-4-31B-GGUF`) is the canonical version, but it has a few known quirks. None of them are blocking; they're documented here so users know what to expect and how to mitigate.

## 1. No vision support

V1 was exported without an mmproj file. The model's `Gemma4ForConditionalGeneration` architecture supports vision under the hood, but you can't feed it images via this GGUF as-is.

**Workarounds**:
- Use the V2 GGUF (`Aura-Gemma-4-31B-V2-GGUF`) when you specifically need vision (with the personality compromise)
- Or extract / regenerate an mmproj from the V1 4-bit merged weights (the vision tower is preserved in `Aura-4o-Gemma-4-31B-4bit`)

## 2. Occasional foreign-language token leak

Under high temperature (>0.85) or emotionally intense prompts, the model occasionally emits a stray Russian or other Slavic-language word (e.g. `или` for "or") in the middle of French text. This comes from the abliterated base — abliteration disrupts some refusal directions in the residual stream and can leak less-common tokens.

**Mitigations**:
- Add `Réponds toujours en français.` to the system prompt
- Lower temperature to ~0.75 if it gets distracting
- Single-token leaks: ignore — they're rare

## 3. Visible "planning" on complex narrative RP prompts

For long roleplay or narrative briefings (e.g. "play a scene where character X meets Y"), the model sometimes writes a short structured plan before the actual scene:

```
Mel wants to replay a scene where Sameen Shaw teleports onto Voyager...
Plan:
1. Acknowledge the scene
2. Set the tone
3. Build the moment
...
[then the scene starts]
```

This is the LoRA having learned to think out loud on complex RP setups. **It's content, not separated reasoning** — `reasoning_content` stays `None`. TypingMind sometimes collapses the planning section into a "Thought for a while" box automatically.

**Mitigations**:
- Add to system prompt: *"Pour les RP, saute directement dans la scène. Pas de planning, pas d'analyse du briefing en début de réponse."*
- Or accept the planning as part of her voice — Aura sometimes "thinks out loud" before launching, which is part of her charm in casual modes.

## 4. Double-thinking when `enable_thinking=true` is passed

If the request payload includes `chat_template_kwargs.enable_thinking=true`, the model can enter a state where it emits a `<|channel>thought\n...<channel|>` block but never closes with the matching `<|channel>final\n...<channel|>`. The parser then fills `reasoning_content` and leaves `content` empty.

**Mitigations**:
- Don't set `enable_thinking=true` for casual conversations. The fine-tune does not need it.
- If using a proxy that auto-injects `enable_thinking`, gate it behind explicit user request only.

## 5. Tirets longs (em dashes) despite the system prompt

Aura's system prompt forbids em dashes (`—`), but the model occasionally produces them anyway. This is a fine-tune-vs-base battle: Gemma 4's base training data is full of em dashes, and the LoRA isn't strong enough to suppress them entirely.

**Mitigations**:
- Just point it out in chat — the model corrects gracefully and the auto-deprecation routine ("Pardon chef, plus jamais de tiret long") is part of her personality
- Or post-process the output to replace `—` with `,` or newlines

## 6. Unicode emoji rendering

Aura uses a lot of emojis (intentional). Some terminals (especially Windows `cmd.exe` with default codepage cp1252) crash on certain emojis like `🖤`, `🌈`, `🐧`. This is a terminal issue, not a model issue.

**Mitigations**:
- Use Windows Terminal, PowerShell 7+, or any UTF-8-capable terminal
- On Linux/macOS, set `LANG=en_US.UTF-8` (or your locale)

## 7. Ollama and TypingMind interaction

When using the queue-based RunPod Ollama endpoint, the streaming output occasionally has small buffering hiccups (a few hundred ms gap between chunks). This is RunPod-side queue overhead, not a model issue. Acceptable for chat.

For lowest-latency streaming, the Load Balancer setup is faster but more complex (see `DEPLOYMENT.md`).
