---
date: 2026-05-11
tags: [software-engineering, ddd]
type: concept
status: active
---
# Ubiquitous Language (通用语言)
来自 Eric Evans《领域驱动设计》(DDD)。开发者之间的对话、代码里的命名、与领域专家的交流，都应来自同一个领域模型。

## Details
在 AI 编程中，通用语言解决"AI 太啰嗦、鸡同鸭讲"的问题。当人和 AI 不在同一话语体系时，AI 试图通过说大量的话来弥补理解偏差。

实践方法：扫描代码库提取所有术语，生成 Markdown 表格。每次与 AI 规划时带入这个文件。

观察到的效果：通用语言不仅改善规划质量，还让 AI 的实现更贴近设想——因为 AI 在用和你完全一致的词汇思考。

## See Also
- [[software-fundamentals-in-ai-era]]
- [[matt-pocock]]
- [[design-concept]]

## Counter-Arguments and Gaps
- 对大型遗留代码库，提取通用语言本身的成本可能很高
