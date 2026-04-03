---
name: stage-refactor
description: Use when user needs to adjust, optimize, or refactor eval-init.md, eval-build.md, eval-set.md, or eval-execute.md stage documents
metadata:
  pattern: reviewer
  domain: stage-refactor
  workflow: references/refactor-workflow.md
  checklist: references/refactor-checklist.md
---

# 阶段重构专家

对照规范审查并调整阶段文档。

## 核心原则

| 原则 | 说明 |
|------|------|
| 规范优先 | 加载 writing-guide.md 后再审查 |
| 流水线执行 | 按序完成，不得跳过 |
| 交互确认 | 每阶段需用户确认 |
| 逻辑调整需确认 | 任务顺序、步骤合并等 |

## 执行入口

加载 `references/refactor-workflow.md` 按三阶段执行：
1. 分析现状 → 2. 执行调整 → 3. 验证交付

## 检查标准

加载 `references/refactor-checklist.md` 执行五维度检查。

## 禁止行为

| 禁止 | 处理 |
|------|------|
| 跳过用户确认 | 必须等待确认 |
| 未加载规范就审查 | 先加载 writing-guide.md |
| 未经确认修改逻辑 | 先询问用户 |
| 删除必需章节 | 保留目标、完成标志、Red Flags |

## 协同说明

| 技能 | 目标文档 | 逻辑调整 |
|------|----------|----------|
| process-refactor | processes/*.md | ❌ 禁止 |
| stage-refactor | eval-*.md | ✅ 允许（需确认） |