# 项目定位

kvpress 是基于 PyTorch 和 Hugging Face Transformers 的 **KV cache 压缩实验库**：用统一的 Press 接口比较不同方法，在长上下文推理中权衡缓存占用、准确率和预填充开销。它不是模型服务或训练框架。项目依赖、版本与 Python 要求见 [[pyproject.toml#L1-L24|包配置]]；设计目标见 [[README.md#L10-L13|README]]。

## 学习时先记住三件事

1. **Prefill 和 decode 不同。** 默认在 context 的初始 prefill 期间安装 attention hook，随后用修改后的缓存对问题生成答案；`DecodingPress` 则在生成阶段继续工作。[[kvpress/pipeline.py#L210-L249|Pipeline 分支]]
2. **压缩不一定意味着缩短张量。** 普通 `ScorerPress` 会 top-k gather K/V；`AdaKVPress` 设置待屏蔽位置，由 attention patch 在 decode 时使用伪 key 屏蔽，原 K/V 张量并不因此缩短。[[kvpress/presses/scorer_press.py#L87-L103|gather]] / [[kvpress/presses/adakv_press.py#L54-L81|mask]]
3. **公开 API 和评测注册表不是一回事。** `kvpress/__init__.py` 导出类并打全局 patch；`evaluation/evaluate_registry.py` 列出可通过评测 CLI 选择的预制实例、数据集和评分器。[[kvpress/__init__.py#L5-L55|库入口]] / [[evaluation/evaluate_registry.py#L53-L82|评测入口]]

## 主路径一眼看懂

点击顶部 **架构图**，在本站内查看可交互的 Archify 图；再依次读「请求与缓存生命周期」「Press 家族」。图展示的是库的核心调用路径，并非所有 press 同时运行的部署拓扑。

| 输入 | 处理 | 输出 |
|---|---|---|
| context、question、press | Pipeline 模板化与编码 → context prefill → Press 修改 KV cache → greedy decode | `{"answer": ...}` 或多问题的 `{"answers": ...}` |
| dataset、模型、press 名、配置 | evaluation CLI → Pipeline → dataset scorer | `predictions.csv`、`metrics.json`、`config.yaml` |

## 使用形态

```python
from transformers import pipeline
from kvpress import KnormPress

pipe = pipeline("kv-press-text-generation", model="Qwen/Qwen3-8B", device_map="auto")
result = pipe("长上下文", question="问题？", press=KnormPress(compression_ratio=0.5))
print(result["answer"])
```

这是阅读示例，来自 [[README.md#L37-L55|上游使用说明]] 和 [[kvpress/pipeline.py#L327-L331|Pipeline 注册]]；此学习站不下载模型，也不声称做了推理质量测试。
