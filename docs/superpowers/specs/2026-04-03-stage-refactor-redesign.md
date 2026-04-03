# stage-refactor 技能重构设计

## 概述

将 `stage-refactor` 技能从混合模式重构为纯流水线模式，符合五种设计模式规范，提升 Token 效率。

---

## 问题分析

### 当前问题

| 类别 | 问题 | 影响 |
|------|------|------|
| Description | "阶段文档重构专家"总结功能，非触发条件 | 违反 Writing Skills 规范 |
| metadata | 无 | 缺少设计模式标识 |
| 指令长度 | 247行，检查清单内嵌 | Token 消耗过高 |
| 门控条件 | "询问用户"表述模糊 | 流水线门控不明确 |
| Pitfalls | 无 | 缺少踩坑点指导 |

### 根因

设计时未严格遵循五种设计模式规范，流水线模式特征不明显。

---

## 设计方案

### 目标

- 将 SKILL.md 从 247 行精简至 ~80 行
- 检查清单移至外部文件按需加载
- 明确流水线门控条件
- 补全 metadata 和 Pitfalls

### 目录结构

```
.claude/skills/stage-refactor/
├── SKILL.md                    # 主技能文件（~80行）
└── references/
    ├── writing-guide.md        # 阶段文档书写规范（已有）
    └── checklist.md            # 五维度检查清单（新增）
```

---

## 详细设计

### 1. YAML Frontmatter

```yaml
---
name: stage-refactor
description: Use when user needs to adjust, optimize, or refactor eval-init.md, eval-build.md, eval-set.md, or eval-execute.md stage documents
metadata:
  pattern: pipeline
  domain: stage-refactor
  steps: "4"
  checklists: references/checklist.md
---
```

**改进点**：
- Description 仅包含触发条件
- 新增 metadata 标注设计模式属性

### 2. SKILL.md 正文结构

| 章节 | 内容 | 行数 |
|------|------|------|
| 标题描述 | 简短陈述式描述 | ~2行 |
| 核心原则 | 表格形式 | ~6行 |
| 流水线执行 | 四步骤带门控条件 | ~50行 |
| 常见踩坑点 | 3个 Pitfall | ~15行 |
| 协同说明 | 与 process-refactor 协同 | ~8行 |

**门控条件设计**：

| 步骤 | 门控条件 |
|------|----------|
| 第一步 | 输出差异摘要后，不得进入第二步直到用户确认 |
| 第二步 | 用户选择调整方式后，不得进入第三步直到确认完毕 |
| 第三步 | 全部调整完成，不得进入第四步直到用户确认 |
| 第四步 | 询问"是否保存"，不得结束直到用户确认 |

### 3. Pitfalls 章节

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| 跳过用户确认 | 直接进入下一步骤 | 门控条件被忽视 | 标注"不得继续直到用户确认" |
| 未加载规范 | 凭直觉判断 | 跳过第一步规范读取 | 第一步必须读取 writing-guide.md |
| 逻辑调整未经确认 | 直接调整任务顺序 | 误认为无需确认 | 逻辑调整必须单独确认 |

### 4. checklist.md 结构

五维度检查清单，独立存储：

| 维度 | 检查项数 |
|------|----------|
| 结构与格式 | 10项 |
| 交互友好性 | 4项 |
| 内容精简性 | 4项 |
| 内容逻辑合理性 | 5项 |
| 阶段特有元素 | 4项 |

---

## 实施清单

### 文件变更

| 文件 | 操作 | 内容 |
|------|------|------|
| `SKILL.md` | 重写 | 精简至 ~80 行，添加门控条件和 Pitfalls |
| `references/checklist.md` | 新增 | 五维度检查清单 |

### 验证标准

- [ ] SKILL.md 行数 ≤ 100
- [ ] Description 仅包含触发条件
- [ ] metadata 包含 pattern/domain/steps/checklists
- [ ] 每个流水线步骤有明确门控条件
- [ ] Pitfalls 章节包含至少 3 个踩坑点
- [ ] checklist.md 包含五维度检查项

---

## 预期效果

| 指标 | 改进前 | 改进后 | 提升 |
|------|--------|--------|------|
| SKILL.md 行数 | 247 | ~80 | 67% |
| Token 消耗（估算） | ~7500 | ~2000 | 73% |
| 设计模式符合度 | 部分 | 完全 | - |