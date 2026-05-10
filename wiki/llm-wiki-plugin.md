---
date: 2026-05-10
tags: [tools, knowledge-management]
type: source-summary
source-url: https://github.com/ekadetov/llm-wiki
---
# LLM Wiki — Claude Code Plugin for Obsidian

ekadetov 开发的 Claude Code 插件，在 Obsidian 知识库中构建持久化的、可复合增长的知识库。采用 Karpathy LLM Wiki 模式，支持源文件摄入、知识编译、查询合成和完整性检查。

## Key Points

- 安装：`claude plugin install /path/to/llm-wiki`
- 前置条件：Node.js 18+、Git、Obsidian vault 含 `03-Resources/` 目录
- 核心操作：init（创建wiki结构）、ingest（摄入源文件）、compile（编译为wiki页面）、query（查询合成）、lint（完整性检查）
- 目录结构：raw/（不可变源文件）、wiki/（LLM 管理页面）、outputs/（报告）、CLAUDE.md（schema）
- 可选集成 qmd 提供混合搜索（BM25 + 向量），小 wiki 可回退到 index.md

## Entities Mentioned
- [[llm-wiki]]
- [[obsidian]]
- [[claude-code]]
- [[qmd]]
