# 进度展示规范拆分实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将进度展示规范拆分为核心层（SKILL.md）和详细层（references/），优化 Token 效率

**Architecture:** 删除 SKILL.md 中的阶段/任务/步骤/流程级进度模板，保留全流程概览和状态符号定义；新建 references/进度展示详细规范.md 存放详细模板

**Tech Stack:** Markdown 文档编辑

---

## 文件结构

| 文件 | 操作 | 责任 |
|------|------|------|
| `.claude/skills/model-evaluation/SKILL.md` | 修改 | 保留核心层，删除详细模板，添加引用 |
| `.claude/skills/model-evaluation/references/进度展示详细规范.md` | 新建 | 存放详细进度模板 |

---

### Task 1: 创建详细层文档

**Files:**
- Create: `.claude/skills/model-evaluation/references/进度展示详细规范.md`

- [ ] **Step 1: 创建进度展示详细规范.md**

写入以下内容：

```markdown
---
name: progress-display-detail
description: Use when entering a stage, task, step, or flow and need to display progress path
---

# 进度展示详细规范

本文档定义阶段/任务/步骤/流程级的进度展示模板。

---

## 进入阶段时

```
📍 当前位置：{阶段名称}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
初始化 ✓ → 构建配置 ✓ → {当前阶段} ● → 执行评测 ○
                             │
                             ├─ 任务1: {名称} ○
                             └─ 任务2: {名称} ○
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 进入任务时

```
📍 {阶段} > 任务{n}: {任务名称}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
初始化 ✓ → 构建配置 ✓ → {当前阶段} ● → 执行评测 ○
                             │
                             ├─ 任务{n}: {名称} ●
                             │      │
                             │      步骤1 {名称} ● → 步骤2 ○ → 步骤3 ○
                             │
                             └─ 任务{n+1}: {名称} ○
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 进入步骤时

```
📍 {阶段} > 任务{n} > 步骤{m}: {步骤名称}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
初始化 ✓ → 构建配置 ✓ → {当前阶段} ● → 执行评测 ○
                             │
                             ├─ 任务{n}: {名称} ●
                             │      │
                             │      步骤1 ✓ → 步骤2 ✓ → 步骤{m} {名称} ● → 步骤{m+1} ○
                             │
                             └─ 任务{n+1}: {名称} ○
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 当前操作: {步骤描述}
```

---

## 进入流程时

```
📍 {阶段} > 任务{n} > 步骤{m} > 流程{k}: {流程名称}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
初始化 ✓ → 构建配置 ✓ → {当前阶段} ● → 执行评测 ○
                             │
                             ├─ 任务{n}: {名称} ●
                             │      │
                             │      步骤1 ✓ → 步骤{m} {名称} ● → 步骤{m+1} ○
                             │
                             ├─ 流程{k}: {流程名称} ●
                             │      │
                             │      步骤1 {名称} ○ → 步骤2 ○ → 步骤3 ○
                             │
                             └─ 任务{n+1}: {名称} ○
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 当前操作: {流程描述}
```

---

## 分支判断格式

**展示判断条件**：

```
{判断条件}
  ├─ {选项A} → {后果A}
  ├─ {选项B} → {后果B}
  └─ {选项C} → {后果C}
```

**展示判断结果**：

```
📋 判断结果：{结果} → {后续动作}
```
```

---

### Task 2: 修改 SKILL.md 保留核心层

**Files:**
- Modify: `.claude/skills/model-evaluation/SKILL.md:49-182`（进度展示规范章节）

- [ ] **Step 1: 删除详细进度模板**

删除 SKILL.md 中第 88-182 行（从"#### 进入阶段时"到"#### 分支判断格式"结束），保留：
- 核心原则表格（第 49-55 行）
- 展示层级表格（第 57-64 行）
- 全流程概览模板（第 66-85 行）
- 状态符号表格（第 157-162 行）

- [ ] **Step 2: 在状态符号表格后添加引用**

在状态符号表格后添加：

```markdown
> 阶段/任务级进度模板见 [进度展示详细规范](./references/进度展示详细规范.md)
```

---

### Task 3: 提交变更

- [ ] **Step 1: 提交变更**

```bash
git add .claude/skills/model-evaluation/SKILL.md .claude/skills/model-evaluation/references/进度展示详细规范.md
git commit -m "$(cat <<'EOF'
refactor(skill): 拆分进度展示规范为核心层和详细层

- SKILL.md 保留全流程概览和状态符号定义（减少约75%行数）
- 新建 references/进度展示详细规范.md 存放详细模板
- 优化 Token 效率，详细规范按需加载

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## 自检结果

- ✅ Spec coverage: 设计文档所有要求已覆盖
- ✅ Placeholder scan: 无 TBD/TODO
- ✅ Type consistency: 文档引用路径一致