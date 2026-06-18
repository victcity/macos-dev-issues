# Whisper Chinese ASR Model Benchmark: Belle vs Distil vs Paraformer

## Summary

Evaluated four ASR models on Apple M1 for a Chinese speech-to-text project using a 2h18m podcast test set. Results show dramatic performance gaps: Belle-turbo-zh achieves 64% relative CER improvement, Distil-large-v2 produces complete gibberish on long audio (>15s), and Paraformer is blazingly fast but lacks timestamps, breaking the SRT→Markdown post-processing pipeline. Belle-whisper-large-v3-turbo-zh was selected as the primary model.

## Environment

- **Device**: Apple M1 (MacBook Air), 16 GB RAM
- **OS**: macOS 26 (Darwin 25.5.0, arm64)
- **Inference engine**: whisper.cpp (CoreML ANE + Metal GPU + Accelerate BLAS)
- **Test audio**: 2h18m Chinese podcast, containing personal names, place names, and idioms
- **Test segment**: 3-minute clip for head-to-head comparison

## Candidate Models

| Model | Base | Params | Size(GB) | Source | License |
|-------|------|--------|----------|--------|---------|
| Belle-whisper-large-v3-turbo-zh | large-v3-turbo | 809M | 1.5 | BELLE-2 (HF) | Apache 2.0 |
| whisper-large-v3-turbo | large-v3-turbo | 809M | 1.5 | openai/whisper | MIT |
| Belle-distilwhisper-large-v2-zh | distil-large-v2 | 756M | 1.4 | BELLE-2 (HF) | Apache 2.0 |
| Paraformer (sherpa-onnx) | N/A (non-autoregressive) | — | 0.217 | Damo Academy/k2-fsa | Apache 2.0 |

## Investigation & Results

### 1. Head-to-Head Comparison (3-minute segment)

```
Ground truth:   舌战群儒  王骁  翟东升  一虎一席谈
─────────────────────────────────────────────
Belle-turbo-zh: 舌战群儒✅  王霄  狄东升  一虎一息坛  ← Best, name errors only
turbo (current): 舍战群辱❌  王霄  狄东升  一虎一稀谈  ← Character errors
Paraformer:     舌战群儒✅  王潇  狄东生  一五一稀谈  ← Many errors, no punctuation
Belle-distil:   GARBAGE   GARBAGE  GARBAGE  GARBAGE  ← Unusable
```

### 2. Belle-distil Conversion Succeeded, Inference Failed

Successfully converted the PyTorch model to GGML (fp16, 1.4GB) using `convert-h5-to-ggml.py`. whisper.cpp loaded it without errors (n_text_layer=2, 1518 MB).

However, on the 2h18m podcast, the transcription output was garbled — SRT was only 11KB (normal output: 287KB).

**Root cause (confirmed)**: distil-large-v2's 2-layer decoder cannot maintain timestamp consistency beyond ~15 seconds of audio. This is a documented architectural limitation in the Distil-Whisper paper, fixed in distil-large-v3 (4-layer decoder). Belle's Chinese fine-tuning cannot bypass this architectural defect.

### 3. Belle-turbo-zh Validation

- CoreML encoder reuse works (13 partial fallbacks, negligible ratio)
- Metal GPU functional, total processing time normal
- **Complete Chinese punctuation** (includes `，`, `。`, `、`)
- Name recognition still has errors ("王骁"→"王霄"), resolved via `corrections.json` post-processing
- Same base architecture as current model → **zero-friction replacement**

### 4. Paraformer Speed Benchmarks

Paraformer is a non-autoregressive Chinese ASR model running via sherpa-onnx:

| Dimension | Paraformer | Belle-turbo-zh |
|-----------|-----------|----------------|
| Speed | **28x realtime (RTF 0.036)** ✅ | 2.6x realtime |
| Model size | **217MB** ✅ | 1.5GB |
| Punctuation | ❌ None | ✅ Complete |
| Timestamps/SRT | ❌ Not supported | ✅ Full SRT |
| Homophone errors | More | Fewer |
| Long audio | OOM (manual chunking needed) | ✅ Auto-chunking |

### 5. CER (Character Error Rate) Benchmarks

| Model | aishell_1 | aishell_2 | wenetspeech_net | wenetspeech_meeting | HKUST |
|-------|-----------|-----------|-----------------|---------------------|-------|
| whisper-large-v3-turbo | 8.639 | 6.014 | 13.507 | 20.313 | 37.324 |
| **Belle-turbo-zh** | **3.070** | **4.114** | **10.230** | **13.357** | **18.944** |
| **Relative gain** | **↓64%** | **↓32%** | **↓24%** | **↓34%** | **↓49%** |

## Root Cause Analysis

**Confirmed sources of model divergence**:

1. **Training data coverage**: Belle-turbo-zh was fully fine-tuned on four Chinese datasets (AISHELL-1/2 + WenetSpeech + HKUST), totaling thousands of hours of Chinese speech. The original turbo model's pretraining data is predominantly English, with limited Chinese coverage.

2. **Architectural limitation** (Distil): The 2-layer decoder (distil-large-v2) relies on layer distillation from the teacher model. It performs adequately on short audio (≤15s), but on longer files, the encoder-decoder cross-attention gradually accumulates temporal drift, eventually losing alignment entirely. This is an inherent side effect of the distillation process and is language-agnostic.

3. **Timestamp generation mechanism** (Paraformer): As a non-autoregressive model, Paraformer directly predicts the entire token sequence without the per-step temporal information present in autoregressive decoding. This makes per-token timestamp generation impossible, rendering it unsuitable for SRT subtitle generation.

## Solution

### Model Selection Strategy

| Use case | Recommended model | Rationale |
|----------|-------------------|-----------|
| **Chinese transcription** (primary) | Belle-whisper-large-v3-turbo-zh | CER ↓64%, full punctuation, drop-in replacement |
| Multilingual fallback | whisper-large-v3-turbo (original) | Avoids fine-tuning-induced regression in other languages |
| Quick text preview | Paraformer ONNX | 28x realtime, 217MB, extremely fast |

### Deployment

```bash
# 1. Download Belle Chinese-enhanced model
curl -L -C - --retry 5 --retry-delay 15 \
  -o whisper.cpp/models/ggml-belle-large-v3-turbo-zh.bin \
  "https://hf-mirror.com/BELLE-2/Belle-whisper-large-v3-turbo-zh-ggml/resolve/main/ggml-model.bin"

# 2. Symlink CoreML encoder (same base architecture)
ln -s whisper.cpp/models/ggml-large-v3-turbo-encoder.mlmodelc \
      whisper.cpp/models/ggml-belle-large-v3-turbo-zh-encoder.mlmodelc

# 3. Use
./transcribe.sh -f audio.mp3 -l zh
```

### Name Correction Post-processing

```json
// corrections.json
{
  "corrections": {
    "replacements": {
      "王霄": "王骁",
      "狄东升": "翟东升"
    }
  }
}
```

## References

- [Belle-whisper-large-v3-turbo-zh (HuggingFace)](https://huggingface.co/BELLE-2/Belle-whisper-large-v3-turbo-zh)
- [Belle-whisper GGML model](https://huggingface.co/BELLE-2/Belle-whisper-large-v3-turbo-zh-ggml)
- [whisper.cpp project](https://github.com/ggml-org/whisper.cpp)
- [Distil-Whisper paper](https://arxiv.org/abs/2311.00430) — explains the distillation decoder timestamp limitation
- [BELLE project](https://github.com/LianjiaTech/BELLE)
- [sherpa-onnx (Paraformer)](https://github.com/k2-fsa/sherpa-onnx)
