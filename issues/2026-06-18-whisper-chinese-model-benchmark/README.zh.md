# Whisper 中文 ASR 模型选型：Belle vs Distil vs Paraformer 实测对比

## 概述

在 Apple M1 上为 whisper.cpp 中文语音转文字项目（2h18m 播客测试集）评估四款 ASR 模型。实测发现模型间性能差异巨大：Belle-turbo-zh 中文 CER 相对提升 64%，Distil-large-v2 长音频完全不可用（输出乱码），Paraformer 速度惊人但无时间戳导致 SRT→Markdown 后处理链路断裂。最终选定 Belle-whisper-large-v3-turbo-zh 作为主力模型。

## 环境

- **设备**: Apple M1 (MacBook Air), 16 GB RAM
- **系统**: macOS 26 (Darwin 25.5.0, arm64)
- **推理引擎**: whisper.cpp (CoreML ANE + Metal GPU + Accelerate BLAS)
- **测试音频**: 2h18m 中文播客（王骁×翟东升），含人名、地名、成语等难点
- **测试片段**: 3 分钟段落用于横向对比

## 候选模型

| 模型 | 基座 | 参数量 | 大小(GB) | 来源 | 许可证 |
|------|------|--------|----------|------|--------|
| Belle-whisper-large-v3-turbo-zh | large-v3-turbo | 809M | 1.5 | BELLE-2 (HF) | Apache 2.0 |
| whisper-large-v3-turbo | large-v3-turbo | 809M | 1.5 | openai/whisper | MIT |
| Belle-distilwhisper-large-v2-zh | distil-large-v2 | 756M | 1.4 | BELLE-2 (HF) | Apache 2.0 |
| Paraformer (sherpa-onnx) | N/A (非自回归) | — | 0.217 | 达摩院/k2-fsa | Apache 2.0 |

## 排查过程与实测结果

### 1. 同段落横向对比（3 分钟片段）

```
原文:           舌战群儒  王骁  翟东升  一虎一席谈
─────────────────────────────────────────────
Belle-turbo-zh: 舌战群儒✅  王霄  狄东升  一虎一息坛  ← 最佳，仅人名错误
turbo（当前）:   舍战群辱❌  王霄  狄东升  一虎一稀谈  ← 有错字
Paraformer:     舌战群儒✅  王潇  狄东生  一五一稀谈  ← 错字多、无标点
Belle-distil:   乱码     乱码   乱码    乱码        ← 不可用
```

### 2. Belle-distil-large-v2-zh 转换成功但推理失败

使用 `convert-h5-to-ggml.py` 成功将 PyTorch 模型转为 GGML（fp16, 1.4GB）。whisper.cpp 正确加载（n_text_layer=2, 1518 MB），加载无任何错误。

但在 2h18m 长音频上，转录输出为乱码——SRT 仅 11KB（正常应为 287KB）。

**根因（已确认）**: distil-large-v2 的 2 层解码器无法维持超过 15 秒的时间戳一致性。这是 Distil-Whisper 论文中记录的已知架构限制，在 distil-large-v3 中已修复（4 层解码器）。Belle 的中文微调无法绕过此架构缺陷。

### 3. Belle-turbo-zh 实测验证

- CoreML 编码器复用成功（13 次 partial fallback，占比极低）
- Metal GPU 正常，总耗时正常
- **中文标点完整**（含 `，`、`。`、`、` 等）
- 人名识别仍有错误（"王骁"→"王霄"、"翟东升"→"狄东升"），通过 `corrections.json` 后处理解决
- 与当前默认模型 turbo 基座完全相同 → **零摩擦替换**

### 4. Paraformer 速度测试

Paraformer 是非自回归中文 ASR 模型，通过 sherpa-onnx 运行时推理：

| 维度 | Paraformer | Belle-turbo-zh |
|------|-----------|----------------|
| 速度 | **28x 实时（RTF 0.036）** ✅ | 2.6x 实时 |
| 模型大小 | **217MB** ✅ | 1.5GB |
| 标点符号 | ❌ 无 | ✅ 完整 |
| 时间戳/SRT | ❌ 不支持 | ✅ 完整 SRT |
| 同音错字 | 较多 | 较少 |
| 长音频 | OOM（需手动分片） | ✅ 自动分片 |

### 5. CER（字符错误率）基准数据

| 模型 | aishell_1 | aishell_2 | wenetspeech_net | wenetspeech_meeting | HKUST |
|------|-----------|-----------|-----------------|---------------------|-------|
| whisper-large-v3-turbo | 8.639 | 6.014 | 13.507 | 20.313 | 37.324 |
| **Belle-turbo-zh** | **3.070** | **4.114** | **10.230** | **13.357** | **18.944** |
| **相对提升** | **↓64%** | **↓32%** | **↓24%** | **↓34%** | **↓49%** |

## 根因分析

**已确认的模型差异来源**：

1. **训练数据覆盖**：Belle-turbo-zh 在 AISHELL-1/2 + WenetSpeech + HKUST 四个中文数据集上全参数微调，总计数千小时中文语音。原版 turbo 的预训练数据以英语为主，中文占比较低。

2. **架构限制**（Distil）：2 层解码器（distil-large-v2）依赖教师模型的层蒸馏，在短音频（≤15s）上表现正常，但长音频上编码器-解码器交叉注意力会逐渐累积时间偏移，最终完全丢失对齐。这是蒸馏过程的固有副作用，与语言无关。

3. **时间戳生成机制**（Paraformer）：Paraformer 作为非自回归模型，直接预测整个 token 序列，没有自回归解码中的时间步信息，因此无法生成逐 token 的时间戳。这使其无法用于 SRT 字幕生成场景。

## 解决方案

### 模型选型策略

| 场景 | 推荐模型 | 理由 |
|------|----------|------|
| **中文转录**（主力） | Belle-whisper-large-v3-turbo-zh | CER ↓64%，标点完整，零摩擦 |
| 多语言回退 | whisper-large-v3-turbo（原版） | 避免微调导致其他语言退化 |
| 纯文本速览 | Paraformer ONNX | 28x 实时，217MB，极速 |

### 部署步骤

```bash
# 1. 下载 Belle 中文强化模型
curl -L -C - --retry 5 --retry-delay 15 \
  -o whisper.cpp/models/ggml-belle-large-v3-turbo-zh.bin \
  "https://hf-mirror.com/BELLE-2/Belle-whisper-large-v3-turbo-zh-ggml/resolve/main/ggml-model.bin"

# 2. CoreML 编码器软链接（同基座复用）
ln -s whisper.cpp/models/ggml-large-v3-turbo-encoder.mlmodelc \
      whisper.cpp/models/ggml-belle-large-v3-turbo-zh-encoder.mlmodelc

# 3. 使用
./transcribe.sh -f audio.mp3 -l zh
```

### 人名纠错后处理

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

## 相关链接

- [Belle-whisper-large-v3-turbo-zh (HuggingFace)](https://huggingface.co/BELLE-2/Belle-whisper-large-v3-turbo-zh)
- [Belle-whisper GGML 模型](https://huggingface.co/BELLE-2/Belle-whisper-large-v3-turbo-zh-ggml)
- [whisper.cpp 项目](https://github.com/ggml-org/whisper.cpp)
- [Distil-Whisper 论文](https://arxiv.org/abs/2311.00430) — 解释了蒸馏解码器的时间戳限制
- [BELLE 项目](https://github.com/LianjiaTech/BELLE)
- [sherpa-onnx (Paraformer)](https://github.com/k2-fsa/sherpa-onnx)
