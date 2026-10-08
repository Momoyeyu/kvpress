# 源码阅读路线

此站固定分析 fork `main` 的 `66c5b0903b3be742bcba6b96d9796bd561e58f2d`。分析分支 `docs/local-architecture-study` 只保留在本地，不用于 PR。本站由静态 HTTP server 提供文件，不运行模型。

## 本地启动

在 `architecture-study` worktree 根目录运行：

```bash
python3 -m http.server 8797 --bind 127.0.0.1 --directory docs/public
```

浏览器打开 `http://127.0.0.1:8797/`；顶部的 **架构图** 标签直接在本站展示 Archify 交互图，不必再起另一个服务。Markdown 章节由浏览器从 `manifest.json` 按需加载，因此应使用 HTTP 而非 `file://`。该命令仅监听回环地址，不下载模型、不占用 GPU。

## 推荐阅读顺序

1. [[kvpress/pipeline.py#L202-L265|Pipeline._forward]]：缓存怎么创建、hook 生效范围、如何复用。
2. [[kvpress/presses/base_press.py#L101-L207|BasePress.forward_hook / __call__]]：谁读写缓存、怎么安装与卸载 hook。
3. [[kvpress/presses/scorer_press.py#L77-L103|ScorerPress.compress]]：最标准的 score / top-k / gather 路径。
4. [[kvpress/presses/adakv_press.py#L54-L81|AdaKV]] 和 [[kvpress/attention_patch.py#L69-L118|attention_patch]]：为什么逻辑淘汰不等于张量缩短。
5. [[kvpress/presses/decoding_press.py#L113-L192|DecodingPress]]、[[kvpress/presses/kvzip_press.py#L81-L203|KVzipPress]]：对比默认路径的两类时序例外。
6. [[evaluation/evaluate.py#L503-L577|评测 runner]]、[[tests/test_pipeline.py#L1-L45|Pipeline 测试]]：分清指标实验与接口验证。

## 扩展与限制

实现新的 score-based 方法时覆盖 `ScorerPress.score()`；其它机制可覆盖 `BasePress.compress()`，特殊时序还需要核实/覆写 context manager 或 hook。按项目约定导出类、加测试，只有需要参与评测 CLI 时才增加 `PRESS_REGISTRY` 项。[[AGENTS.md#L24-L36|项目贡献约定]]

`SUPPORTED_MODELS` 是已测试模型列表，不是对所有 Transformers 模型的兼容保证；默认 hook 假设层级 `self_attn`、`layer_idx` 和特定 kwargs。量化 backend 的实际内部结构也可能不同。[[kvpress/presses/base_press.py#L27-L34|模型列表]] / [[kvpress/presses/base_press.py#L139-L203|内部约定]]

若更新分析的代码版本，先核查上述调用链，再更新 `manifest.json`、Archify JSON 中的 revision 和本章源码引用，重跑 Archify `finalize`。不要把旧图说成新版源码事实。
