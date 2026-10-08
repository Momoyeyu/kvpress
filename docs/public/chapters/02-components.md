# 模块与边界

## 核心库及其作用域

| 目录 / 文件 | 职责 | 从哪看 |
|---|---|---|
| `kvpress/pipeline.py` | 接收文本与 press，分词、prefill、生成和问题缓存回退 | [[kvpress/pipeline.py#L105-L171|preprocess]]、[[kvpress/pipeline.py#L202-L265|_forward]] |
| `kvpress/presses/base_press.py` | `with press(model)` 注册/移除每层 attention hook；读取、压缩并改写缓存 | [[kvpress/presses/base_press.py#L139-L207|BasePress]] |
| `kvpress/presses/scorer_press.py` | score → top-k → gather 的通用剪枝实现 | [[kvpress/presses/scorer_press.py#L77-L103|ScorerPress]] |
| `kvpress/presses/*` | 独立算法和组合、分层、解码期 wrapper | [[README.md#L99-L155|Press 列表]] |
| `kvpress/attention_patch.py` | 替换 Transformers attention 函数注册表中的函数，以支持 head-wise 掩码 | [[kvpress/attention_patch.py#L69-L118|补丁]] |
| `kvpress/utils.py` | 解量化层缓存、保留数、模型 query/key 投影与 RoPE 辅助 | [[kvpress/utils.py#L131-L160|缓存工具]] |
| `evaluation/` | Fire CLI、压缩方法/数据集/评分器注册表、预测与指标落盘 | [[evaluation/evaluate.py#L503-L577|评测 runner]] |
| `tests/` | press 单元测试、Pipeline/attention patch 测试和 RULER 集成测试 | [[tests/test_pipeline.py#L1-L45|Pipeline 测试]] |

## 两种影响范围不同的 hook

`BasePress.__call__` 在 `with` 范围内对各层 `self_attn` 添加 **PyTorch forward hook**，退出时 `finally` 移除。注册之前会调用 `post_init_from_model`，Gemma3 的 sliding-window attention 层被跳过。[[kvpress/presses/base_press.py#L188-L207|作用域源码]]

另一边，**导入 `kvpress`** 时就会调用 `patch_attention_functions()`，替换 Transformers 全局 `ALL_ATTENTION_FUNCTIONS` 表中的函数。这不是 context manager，不能把它当作一次调用后自动撤销的临时 hook。只有使用 `masked_key_indices` 的算法才依赖对应的掩码行为。[[kvpress/__init__.py#L54-L55|导入侧]] / [[kvpress/attention_patch.py#L98-L118|全局写入]]

## 依赖的是外部模型内部结构

`BasePress` 把 `SUPPORTED_MODELS` 列为已测试模型，但遇到其他模型是警告而非通用兼容承诺。默认实现假设有 `model.model.layers`、每层的 `self_attn` 和 `layer_idx`，并读取 forward kwargs 中的 `hidden_states`、`past_key_values`、`cache_position`。这也是升级 Transformers 或迁移到不同模型架构时优先检查的边界。[[kvpress/presses/base_press.py#L27-L34|模型列表]] / [[kvpress/presses/base_press.py#L139-L146|参数约定]] / [[kvpress/presses/base_press.py#L194-L203|层遍历]]
