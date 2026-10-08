# 请求与缓存生命周期

理解三个时间窗口：**context prefill → 基于缓存回答问题 → 回退该问题追加的 KV**。

## 1. 输入准备

`preprocess()` 对 context 应用 chat template，或在无模板时使用 BOS；分别编码成 `context_ids` 和 `questions_ids`。若 context 超过 `max_context_length`，只截断 context。普通路径里问题不参与 context prefill；评测 CLI 的 `query_aware` 选项会先把问题拼到 context 再压缩。[[kvpress/pipeline.py#L139-L171|输入处理]] / [[evaluation/evaluate.py#L353-L356|query-aware 分支]]

## 2. prefill 与每层缓存更新

没有传 cache 时 Pipeline 新建 `DynamicCache`。若 press 不是纯解码期 `DecodingPress`，就在 `with press(self.model)` 中调用 `self.model.model(input_ids=context_ids, past_key_values=cache)`。这次 forward 为每层产生 KV；层的 forward hook 之后检查 `cache_position` 是否仍为初始 prefill，再调用 press 的 `compress()`。[[kvpress/pipeline.py#L210-L225|调用]] / [[kvpress/presses/base_press.py#L101-L150|hook]]

K/V 不是 hook 的返回值：普通缓存通过覆盖当前 `cache.layers[module.layer_idx].keys/values` 生效。对 `QuantizedCache` 则先解量化，压缩后重新量化，并更新层的 `cumulative_length`。hook 返回原输出；副作用发生在缓存对象上。[[kvpress/utils.py#L131-L160|缓存读取]] / [[kvpress/presses/base_press.py#L148-L162|写回]]

## 3. 生成与上下文复用

问题的 forward 基于预填充后的缓存，`position_ids` 从原 context 长度继续递增；之后逐 token 贪心生成。生成完一个答案，Pipeline 把该问题追加的各层 KV 切回提问前长度，然后复用 context 缓存回答下一个问题。量化缓存当前实现也直接切其 `_quantized_keys/values` 内部结构，这不等于任意 backend 均已验证兼容。[[kvpress/pipeline.py#L233-L265|回退]] / [[kvpress/pipeline.py#L289-L319|生成]]

## 不能套用同一个生命周期的策略

- `DecodingPress` 在解码时积累每层 hidden states，达到间隔或阈值再压缩，退出 context 时 reset；Pipeline 限制它只处理单问题。[[kvpress/presses/decoding_press.py#L113-L192|解码钩子]] / [[kvpress/pipeline.py#L202-L205|限制]]
- `PrefillDecodingPress` 和 `DMSPress(decoding=True)` 有各自的生成期路径。[[kvpress/pipeline.py#L229-L233|条件]]
- `KVzipPress` 覆盖 context manager：初次 prefill 后进行额外的 context reconstruction、打分及后处理，而非单次标准 hook 裁剪。[[kvpress/presses/kvzip_press.py#L81-L145|退出时动作]] / [[kvpress/presses/kvzip_press.py#L177-L203|重放]]
