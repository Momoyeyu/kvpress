# 评测系统

评测是单独的 CLI 应用：读配置、选算法和数据集、调用核心 Pipeline、由数据集专属评分器计算指标。它不负责训练 press，也不能代替真实模型上的准确率实验。

## 从输入到结果

1. **配置：** `CliEntryPoint` 合并 `EvaluationConfig` 默认值、YAML、CLI 覆盖（CLI 优先），验证数据集、评分器及 press 名称。[[evaluation/evaluate.py#L81-L109|校验]] / [[evaluation/evaluate.py#L543-L577|配置优先级]]
2. **实例与模型：** `PRESS_REGISTRY` 保存预制的 press 实例；runner 修改选中实例的参数，创建 `kv-press-text-generation` Pipeline。注册表里的实例是可变对象，不是每次动态构造的新类。[[evaluation/evaluate_registry.py#L82-L142|注册表]] / [[evaluation/evaluate.py#L256-L316|配置 press]] / [[evaluation/evaluate.py#L361-L405|创建模型]]
3. **数据：** `load_dataset(..., split="test")` 转成 pandas；可按比例抽样、插入 needle 或启用 query-aware。[[evaluation/evaluate.py#L319-L359|数据处理]]
4. **推理：** 普通 press 按 context 分组复用一次 prefill；`DecodingPress` 按 context/question 逐行生成。[[evaluation/evaluate.py#L407-L463|推理分支]]
5. **落盘：** 评分器从 DataFrame 计算指标，run 目录写 `predictions.csv`、`metrics.json` 与 `config.yaml`。[[evaluation/evaluate.py#L465-L529|结果]]

## 如何启动一次评测

```bash
cd evaluation
python evaluate.py --dataset ruler --press_name knorm --compression_ratio 0.5
```

此命令会用模型与外部数据集，可能需要网络、额外依赖和计算资源；**本架构站点不会执行它**。正式运行前先阅读 [[evaluation/README.md#L7-L28|评测文档]] 和 [[evaluation/evaluate_registry.py#L53-L79|数据集/评分器注册]]。

## 什么不算验证通过

静态图反映代码调用关系，而不是准确率证据；单元测试证明接口与局部逻辑，不自动证明某种方法在特定数据集上优于基线。量化 backend 的兼容性也要通过对应实际环境验证，不能只根据 `QuantizedCache` 分支推断。[[kvpress/pipeline.py#L252-L265|量化回退路径]]
