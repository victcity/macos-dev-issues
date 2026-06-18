# Talkie 1930 13B: Metal GPU Backend Produces Garbage Tokens (CPU-only Works)

## Summary

When running the Talkie 1930 13B model with llama.cpp (thomasgauthier/talkie-1930 branch) on Apple M1, the Metal GPU backend produces garbage output (repeated `\x1f` control characters). CPU-only inference works correctly and consistently.

## Environment

- **Device**: Apple M1 (MacBook Air), 16 GB RAM
- **OS**: macOS 26 (Darwin 25.5.0, arm64)
- **llama.cpp**: [thomasgauthier/llama.cpp](https://github.com/thomasgauthier/llama.cpp) `talkie-1930` branch
- **Commit**: d246b2d ("first pass support for talkie-1930 model architecture")
- **Build**: `cmake -B build -DGGML_METAL=ON -DGGML_METAL_EMBED_LIBRARY=ON -DCMAKE_BUILD_TYPE=Release`
- **Model**: `talkie-1930-13b-it-Q4_K_M.gguf` (Q4_K - Medium, 7.98 GiB, 443 tensors)

## Model Architecture

```
talkie-1930-13b-it: 40 layers, 40 heads, d_model=5120, d_head=128, d_ffn=13696
RoPE: base=1,000,000, dim=128
Tokenizer: gpt-4o BPE, vocab=65540
Special tokens: <|endoftext|>(65535/eos), <|end|>(65536/eot), <|user|>(65537), <|assistant|>(65538), <|system|>(65539)
```

**Differences from standard LLaMA**: RMS Norm applied to Q/K after RoPE, per-head Q gain, per-layer attention/MLP/embedding-skip gain scalars.

## Reproduction

### 1. Start GPU server

```bash
./build/bin/llama-server \
  -m talkie-1930-13b-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --ctx-size 512 --no-warmup
```

### 2. Short prompt — may work initially

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":16,"temperature":0.7}'
# Output: "I thank you." ✅
```

### 3. Long prompt — triggers garbage

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"tell me something happened in 1911"}],"max_tokens":32,"temperature":0.7}'
# Output: "\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f..." ❌
```

### 4. All subsequent requests produce garbage

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":8,"temperature":0.7}'
# Output: "\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f" ❌
```

Server logs in degraded state:
```
decode: failed to find a memory slot for batch of size 1
srv  try_clear_id: purging slot 1 with 19 tokens
srv  update_slots: failed to find free space in the KV cache, retrying with smaller batch size
```

## Diagnostic Matrix

| Configuration | Short prompt ("hi") | Long prompt ("…1911") | Degrades over time? |
|---|---|---|---|
| Metal, default | ✅ (first few) → ❌ | ❌ immediately | **Yes** |
| Metal, `--flash-attn 0` | ✅ (first few) → ❌ | ❌ immediately | **Yes** |
| CPU-only (`-ngl 0`) | ✅ | ✅ | **No** — 6 consecutive alternating tests all correct |

### CPU-only Works Correctly

With `-ngl 0`, the model produces consistent, high-quality 1930s-style text:

| Input | Output |
|---|---|
| "hi" | "Hail, fellow!" |
| "tell me something happened in 1911" | "In 1911 the first part of the National Gallery was…" |
| "hi" | "I am obliged to you." |
| "tell me something happened in 1911" | "In 1911 a great railway strike occurred in Great Britain." |
| "hi" | "I thank you for your kind present." |
| "tell me something happened in 1911" | "In 1911, the number of paupers in receipt of relief…" |

CPU generation speed: ~6.5 t/s on M1.

## Root Cause Analysis

The `\x1f` character (ASCII Unit Separator, token ID 31 in the BPE vocabulary) is not a normal text token. Consistent output of this same low-value token indicates the GPU-side logits or attention computation is producing degenerate results.

The degradation pattern (works briefly → fails after longer sequence → stays broken) points to **GPU-side KV cache state corruption**, consistent with the `"failed to find a memory slot"` log entries.

**Hypothesized causes** (unconfirmed):

1. **KV cache layout incompatibility**: Talkie's MHA (no GQA) with post-RoPE Q/K norms may conflict with Metal's KV cache tensor layout assumptions.
2. **Metal RMS Norm on Q/K bug**: The Q/K RMS Norm operates on per-head reshaped tensors — the Metal kernel for this path may have a numerical or shape-handling bug.
3. **Per-head gain broadcast**: `q_gain` reshaped to `[1, n_head, 1]` multiplied with Q `[seq, n_head, head_dim]` — Metal may mishandle this broadcast combination.

## Solution

Use CPU-only inference (`-ngl 0`):

```bash
./build/bin/llama-server \
  -m talkie-1930-13b-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --ctx-size 2048 \
  --no-warmup -ngl 0
```

Performance on M1 with 16 GB is acceptable: ~6.5 t/s generation, ~4.7 t/s prompt processing.

A convenience startup script is available: [scripts/start-talkie.sh](https://github.com/victcity/llm-local/blob/main/scripts/start-talkie.sh)

## References

- [thomasgauthier/llama.cpp talkie-1930 branch](https://github.com/thomasgauthier/llama.cpp/tree/talkie-1930)
- [talkie-lm/talkie project](https://github.com/talkie-lm/talkie)
- [solwyc/talkie-1930-13b-it-q5](https://github.com/solwyc/talkie-1930-13b-it-q5) — alternative talkie GGUF implementation
- [Talkie model on HuggingFace](https://huggingface.co/talkie-lm/talkie-1930-13b-it)
