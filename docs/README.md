# kvpress 本地架构研究

基于 fork `66c5b0903b3be742bcba6b96d9796bd561e58f2d` 的源码阅读笔记；不属于上游官方文档，不用于提交 PR。参考 Soup 的本地学习站点：`docs/public/index.html` 是统一入口，`manifest.json` 管理章节，`chapters/` 放按需加载的 Markdown，Archify 图直接在顶部「架构图」标签中展示。不依赖 CDN。

从项目 worktree 根目录运行：

```bash
python3 -m http.server 8797 --bind 127.0.0.1 --directory docs/public
```

在浏览器打开 `http://127.0.0.1:8797/`。站点包含项目定位、模块边界、请求生命周期、Press 家族、评测链路和源码阅读路线；源码链接固定到上述提交。图的生成页面与章节共用同一个静态服务，不需单独启动。Archify 源文件是 `docs/diagrams/architecture.json`，生成文件是 `docs/public/architecture.html`。只读浏览文档无需安装项目的 PyTorch/Transformers 依赖。

更新图表时在 worktree 根目录运行：

```bash
node "$ARCHIFY_CLI" finalize architecture docs/diagrams/architecture.json docs/public/architecture.html --repo-root . --quality showcase --json
```

图表校验使用所固定提交的源码作为证据；更新源码时应重新核对来源行号并同步 `meta.repository.revision` 与站点上的版本标识。
