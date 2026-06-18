# Talkie 1930 13B: Metal GPU 后端输出垃圾 token（CPU-only 正常）

## 概述

使用 llama.cpp (thomasgauthier/talkie-1930 分支) 在 Apple M1 上运行 Talkie 1930 13B 模型时，Metal GPU 后端产生垃圾输出（重复的 `\x1f` 控制字符），而 CPU-only 推理完全正常且稳定。

## 环境

- **设备**: Apple M1 (MacBook Air), 16 GB RAM
- **系统**: macOS 26 (Darwin 25.5.0, arm64)
- **llama.cpp**: [thomasgauthier/llama.cpp](https://github.com/thomasgauthier/llama.cpp) `talkie-1930` 分支
- **Commit**: d246b2d ("first pass support for talkie-1930 model architecture")
- **编译参数**: `cmake -B build -DGGML_METAL=ON -DGGML_METAL_EMBED_LIBRARY=ON -DCMAKE_BUILD_TYPE=Release`
- **模型**: `talkie-1930-13b-it-Q4_K_M.gguf` (Q4_K - Medium, 7.98 GiB, 443 个 tensor)

## 模型架构

```
talkie-1930-13b-it: 40 层, 40 注意力头, d_model=5120, d_head=128, d_ffn=13696
RoPE: base=1,000,000, dim=128
分词器: gpt-4o BPE, vocab=65540
特殊 token: <|endoftext|>(65535/eos), <|end|>(65536/eot), <|user|>(65537), <|assistant|>(65538), <|system|>(65539)
```

**与标准 LLaMA 的差异**: Q/K 在 RoPE 之后做了 RMS Norm、per-head Q gain、每层的 attention/MLP/embedding-skip 三组 gain 标量参数。

## 复现步骤

### 1. 启动 GPU 服务

```bash
./build/bin/llama-server \
  -m talkie-1930-13b-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --ctx-size 512 --no-warmup
```

### 2. 短 prompt — 前几次可能正常

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":16,"temperature":0.7}'
# 输出: "I thank you." ✅
```

### 3. 长 prompt — 触发垃圾输出

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"tell me something happened in 1911"}],"max_tokens":32,"temperature":0.7}'
# 输出: "\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f..." ❌
```

### 4. 后续所有请求都变垃圾

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":8,"temperature":0.7}'
# 输出: "\x1f\x1f\x1f\x1f\x1f\x1f\x1f\x1f" ❌
```

服务端日志在退化后出现：
```
decode: failed to find a memory slot for batch of size 1
srv  try_clear_id: purging slot 1 with 19 tokens
srv  update_slots: failed to find free space in the KV cache, retrying with smaller batch size
```

## 排查过程

| 配置 | 短 prompt ("hi") | 长 prompt ("1911年发生了什么") | 随时间退化？ |
|---|---|---|---|
| Metal, 默认 | ✅（前几次）→ ❌ | ❌ 立即出现 | **是** |
| Metal, `--flash-attn 0` | ✅（前几次）→ ❌ | ❌ 立即出现 | **是** |
| CPU-only (`-ngl 0`) | ✅ | ✅ | **否** — 6 次交替测试全部正常 |

### CPU-only 完全正常

`-ngl 0` 模式下模型生成稳定，风格符合 1930 年代设定：

| 输入 | 输出 |
|---|---|
| "hi" | "Hail, fellow!" |
| "tell me something happened in 1911" | "In 1911 the first part of the National Gallery was…" |
| "hi" | "I am obliged to you." |
| "tell me something happened in 1911" | "In 1911 a great railway strike occurred in Great Britain." |
| "hi" | "I thank you for your kind present." |
| "tell me something happened in 1911" | "In 1911, the number of paupers in receipt of relief…" |

CPU 生成速度: M1 上 ~6.5 t/s。

## 根因分析

`\x1f` 字符（ASCII Unit Separator，BPE 词表中的 token ID 31）不是正常的文本 token。模型持续输出相同的低值 token 说明 GPU 上的 logits 或 attention 计算产生了退化结果。

退化模式（短暂正常 → 长序列后失败 → 保持损坏）指向 GPU 端 **KV cache 状态损坏**，与日志中的 `"failed to find a memory slot"` 一致。

**推测原因**（未确认）：

1. **KV cache 布局不兼容**: Talkie 的 MHA（无 GQA）+ post-RoPE Q/K Norm 可能与 Metal 的 KV cache tensor 布局假设冲突。
2. **Metal RMS Norm on Q/K bug**: Q/K 的 RMS Norm 在 per-head 的张量上操作，Metal kernel 对此路径可能存在数值或形状处理错误。
3. **Per-head gain 广播问题**: `q_gain` 被 reshape 为 `[1, n_head, 1]` 后与 Q `[seq, n_head, head_dim]` 相乘，Metal 可能无法正确处理此广播组合。

## 解决方案

使用 CPU-only 推理 (`-ngl 0`)：

```bash
./build/bin/llama-server \
  -m talkie-1930-13b-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --ctx-size 2048 \
  --no-warmup -ngl 0
```

M1 + 16GB 下性能可接受：生成 ~6.5 t/s，prompt 处理 ~4.7 t/s。

也可使用启动脚本：[scripts/start-talkie.sh](https://github.com/victcity/llm-local/blob/main/scripts/start-talkie.sh)

## 相关链接

- [thomasgauthier/llama.cpp talkie-1930 分支](https://github.com/thomasgauthier/llama.cpp/tree/talkie-1930)
- [talkie-lm/talkie 项目](https://github.com/talkie-lm/talkie)
- [solwyc/talkie-1930-13b-it-q5](https://github.com/solwyc/talkie-1930-13b-it-q5) — 另一个 talkie GGUF 的实现
- [Talkie 模型 HuggingFace](https://huggingface.co/talkie-lm/talkie-1930-13b-it)
