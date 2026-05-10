---
date: 2026-05-10
tags: [tools, knowledge-management]
type: concept
status: active
---
# LLM Wiki

基于 Karpathy LLM Wiki 模式的持久化知识库系统。核心理念是将大语言模型的输出持久化到 Obsidian 知识库中，通过 ingestion（摄入）、compilation（编译）、query（查询）循环实现知识的复合增长。

## Details

工作流程：
1. **Ingest**：从 URL 或文件获取原始资料，保存到 `raw/articles/`（不可变）
2. **Compile**：读取原始资料，提取实体和概念，创建/更新 wiki 页面，建立交叉引用
3. **Query**：基于 wiki 内容回答问题，带有 `[[wikilink]]` 引用
4. **Lint**：检查死链、孤立页面、矛盾信息、索引漂移

每个 wiki 目录包含 `CLAUDE.md`（schema 定义）、`raw/`（源文件）、`wiki/`（LLM 管理的页面）、`log.md`（操作日志）。

## See Also
- [[obsidian]]
- [[claude-code]]
- [[qmd]]
- [[llm-wiki-plugin]]

## Counter-Arguments and Gaps
- 未讨论与其他知识管理工具（Notion、Roam Research）的对比
- 未涉及知识库规模增长后的性能和维护问题
