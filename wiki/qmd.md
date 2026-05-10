---
date: 2026-05-10
tags: [tools, search]
type: concept
status: active
---
# qmd

混合搜索引擎，结合 BM25（关键词匹配）和向量搜索，用于在 LLM Wiki 中进行语义查询。可选组件——小规模 wiki 可回退到直接读取 `index.md`。

## Details

qmd 通过 collection 概念组织文档：
- `qmd collection add <path> --name <name>`：添加 wiki 目录为可搜索集合
- `qmd embed --collection <name>`：生成向量嵌入
- `qmd query "<question>" --collection <name>`：执行混合搜索

通过 `qmd.yml` 配置文件定义集合的路径和文件匹配模式。安装时需要 Node.js 18+，且需通过 `env -u BUN_INSTALL` 调用以强制使用 Node.js 运行时（Bun 的 SQLite 构建不支持扩展加载）。

## See Also
- [[llm-wiki]]
- [[llm-wiki-plugin]]

## Counter-Arguments and Gaps
- 未涉及 qmd 的具体向量模型和索引参数
- 未讨论与 Elasticsearch、Meilisearch 等方案的对比
