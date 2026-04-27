---
title: Wiki Log
type: meta
created: 2026-04-15
updated: 2026-04-23
tags:
  - llm-wiki
  - log
---

# Wiki Log

## [2026-04-15] setup | 初始化 LLM Wiki 骨架

- 创建 schema 文件：[[AGENTS]]
- 创建索引：[[wiki/index]]
- 创建日志：[[wiki/log]]
- 创建首页：[[wiki/00-总览/知识库总览]]
- 创建 7 个领域地图页面
- 创建来源总表：[[wiki/30-来源/来源总表]]
- 创建来源摄取计划：[[wiki/30-来源/来源摄取计划]]
- 创建主题 / 实体 / 问答沉淀容器说明
- 创建 4 个模板页面
- 识别到当前 `raw/` 中共有 **81** 篇 Markdown 来源

## [2026-04-16] ingest | LLM Wiki 模式来源（Karpathy）

**来源**：`2026-04-15.md`（Karpathy Gist：LLM Wiki 原文）

**动作**：
- 摄取来源：建立来源摘要页
- 产出自述：创建 LLM Wiki 主题页
- 补充 schema：更新 `AGENTS.md`（新增 Obsidian 工具链、CLI 工具、为什么有效等章节）
- 更新索引：`wiki/index.md`（补充主题页和来源摘要入口）
- 更新来源总表：`wiki/30-来源/来源总表.md`（标记 2026-04-15 为已摄取）

**本次产出**：
- `wiki/10-主题/LLM Wiki模式.md` — 主题页
- `wiki/30-来源/来源摘要-2026-04-15-LLM Wiki.md` — 来源摘要

**新增 schema 章节**：
- 第 10 节：Obsidian 工具链（Web Clipper、图片本地化、Graph View、Dataview、Marp）
- 第 11 节：CLI 工具（qmd、临时搜索脚本）
- 第 12 节：为什么 Wiki 维护成本接近零（Memex 背景）

## [2026-04-23] ingest | 七猫春招 Go 学习路线

**来源**：`raw/mine/Go语言基础/七猫春招go学习路线.md`

**动作**：
- 产出来源摘要页，整理为更易读的学习路线
- 提炼前端转后端的准备清单
- 保留原始 `raw/` 文件不改写

**本次产出**：
- `wiki/30-来源/来源摘要-七猫春招go学习路线.md`

## [2026-04-23] refactor | 七猫春招 Go 学习路线优化整理

**动作**：
- 新增更贴近原文结构的优化版来源页
- 保留原文学习顺序与练习节点
- 将前端转后端的学习顺序建议单独提炼出来

**本次产出**：
- `wiki/30-来源/来源整理-七猫春招go学习路线.md`
