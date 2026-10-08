# Press 家族

按 **实际缓存行为** 分类，比只背论文名称更容易判断兼容性。准确率、运行时间与显存取决于模型和任务；以下仅描述此源码快照中的机制，不是实验排行榜。

| 类型 | 代表 | 行为与代价 |
|---|---|---|
| 分数剪枝 | `ScorerPress`、`KnormPress`、`SnapKVPress` | `score()` 输出 `[batch, kv heads, seq]`，按保留预算 top-k，再 gather K/V；序列长度真的缩短。[[kvpress/presses/scorer_press.py#L77-L103|源码]] |
| 跨 head 掩码 | `AdaKVPress(ScorerPress)` | 全局跨 head 分配淘汰预算，写 `masked_key_indices`，decode 时经 attention patch 用伪 key 消除其权重；原 K/V 张量并未因此物理缩短。[[kvpress/presses/adakv_press.py#L54-L81|源码]] / [[kvpress/attention_patch.py#L69-L93|patch]] |
| 合并再剪枝 | `MergingPress(ScorerPress)` | 先选择幸存项，按 key 余弦相似度把淘汰项的 value 混入幸存项，再 gather；只接收 `ScorerPress`。[[kvpress/presses/merging_press.py#L43-L98|源码]] |
| 定期解码压缩 | `DecodingPress` | 缓存最近的 hidden states，按步数或阈值触发 scorer 剪枝，目标是指定缓存 token 数。[[kvpress/presses/decoding_press.py#L113-L179|源码]] |
| 上下文重建 | `KVzipPress` | 初始 prefill 后分块重放 context 做重要性评分，增加额外前向；警告提示约 2–3 倍 prefill 计算开销。[[kvpress/presses/kvzip_press.py#L57-L65|开销]] / [[kvpress/presses/kvzip_press.py#L177-L203|实现]] |
| 组合 wrapper | `ComposedPress` | 顺序调用每个 press 的 hook，拒绝 AdaKV/KVzip；裁剪后长度与原 attention weights 可能不匹配。[[kvpress/presses/composed_press.py#L11-L62|实现及警告]] |

## 为什么 head-wise 路径要改 attention

`AdaKVPress` 记录被淘汰的 `(batch, head, position)`，而 attention patch 在 decode 时为这些位置生成一个注意力权重近似为零的伪 key。源码也明确写了这个方案不降低 peak memory、略微增加运行时间；因此不要只看算法名称就称其已节省缓存字节。[[kvpress/presses/adakv_press.py#L72-L81|索引来源]] / [[kvpress/attention_patch.py#L51-L93|伪 key 逻辑]]

## 量化不等于独立 press

Pipeline 接收可选的 Transformers `Cache`，默认 `DynamicCache`。`BasePress` 在遇到 `QuantizedCache` 时会解量化当前层、压缩、再量化写回；它是缓存存储形式的一个分支，不是在 CLI 中自动叠加的压缩算法。[[kvpress/pipeline.py#L173-L224|缓存参数]] / [[kvpress/presses/base_press.py#L148-L160|量化写回]]
