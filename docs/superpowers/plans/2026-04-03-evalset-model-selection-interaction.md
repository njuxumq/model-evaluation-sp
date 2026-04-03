# 模型选择流程交互改进实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将模型选择流程从多轮文本输入交互改为 AskUserQuestion 分页交互，提升用户交互友好度。

**Architecture:** 按厂商分组，每页3个模型 + 1个"更多操作"选项，用户选择模型后继续展示当前页，点击"更多操作"可切换厂商或确认保存。

**Tech Stack:** Markdown 文档修改，遵循 Skill 文档规范。

---

## Task 1: 读取现有文档结构

**Files:**
- Read: `processes/evalset-model-selection.md`

- [ ] **Step 1: 读取现有文档**

读取 `d:\Projects\Claude\model-evaluation-sp\.claude\skills\model-evaluation\processes\evalset-model-selection.md`，定位步骤2的行号范围。

预期输出：确认步骤2位于第55-82行（原文）。

---

## Task 2: 重写步骤2部分

**Files:**
- Modify: `processes/evalset-model-selection.md:55-82`

- [ ] **Step 1: 准备替换内容**

将步骤2原文替换为新设计内容：

```markdown
## 步骤2：用户分类选择模型

⚠️ **此步骤需要用户交互确认**：用户需浏览模型列表、选择模型、确认后才能保存。

### 步骤2.1：按厂商分组

读取 `available-models.json`，按厂商分组（DeepSeek、Qwen、讯飞星火、测试模型等）。

识别规则：按模型 `name` 或 `model` 字段关键词识别厂商。

| 厂商 | 识别关键词 |
|------|------------|
| DeepSeek | `deepseek` |
| Qwen | `qwen`, `op3` |
| 讯飞星火 | `spark`, `星火` |
| 测试模型 | `test` 或其他未匹配 |

### 步骤2.2：分页展示与累积选择

使用 AskUserQuestion 工具分页展示，每页最多3个模型选项 + 1个"更多操作"选项。

**选项构成**：

| 位置 | 类型 | 格式 |
|------|------|------|
| 选项1-3 | 模型 | `{name} ({model})`，已选模型标注 `[已选]` |
| 选项4 | 操作 | `更多操作` |

**"更多操作"展开选项**：

| 选项 | 说明 |
|------|------|
| `下一页` | 查看当前厂商更多模型 |
| `跳过本厂商` | 进入下一厂商第1页 |
| `确认保存` | 结束选择并保存 |

**交互规则**：

| 规则 | 说明 |
|------|------|
| 分组依据 | 按厂商分类展示（DeepSeek、Qwen、讯飞星火等） |
| 分页规则 | 每页3个模型 + 1个"更多操作"选项 |
| 累积规则 | 用户选择模型后累积到已选列表，已选模型标注 `[已选]` |
| 页内循环 | 选择模型后继续展示当前页（标注已选状态） |
| 厂商切换 | 点击"跳过本厂商"进入下一厂商第1页 |
| 结束条件 | 点击"确认保存"结束流程，进入步骤3保存 |

**AskUserQuestion 标题格式**：
- 未选时：`选择 {厂商名} 系列模型`
- 已选时：`选择 {厂商名} 系列模型（已选{n}个）`

### 步骤2.3：保存已选列表

用户点击"确认保存"后，调用脚本保存：

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_model.py select-models \
  --input {work-dir}/.eval/{session-id}/available-models.json \
  --selection "{已选模型model字段值，逗号分隔}" \
  --output {work-dir}/.eval/{session-id}/selected-models.json
```

**示例**：已选 DeepSeek-V3.1 和 Qwen3-235B，`--selection` 参数为 `xdeepseekv31,xop3qwen235b`。
```

- [ ] **Step 2: 执行 Edit 替换**

使用 Edit 工具，将原文（第55-82行）替换为上述新内容。

---

## Task 3: 更新红线部分

**Files:**
- Modify: `processes/evalset-model-selection.md:29-33`

- [ ] **Step 1: 在红线表格添加新条目**

在现有红线表格后添加3条新红线：

```markdown
| 单页展示超过3个模型 | AskUserQuestion 选项限制 | 严格执行每页3模型 + 1操作选项 |
| 用户选择模型后直接进入下一页 | 未实现页内循环 | 选择模型后继续展示当前页 |
| 未标注已选状态 | 用户无法看到选择结果 | 已选模型必须标注 `[已选]` |
```

- [ ] **Step 2: 执行 Edit 添加红线**

定位红线表格末尾，添加上述3条新红线。

---

## Task 4: 验证文档修改

**Files:**
- Read: `processes/evalset-model-selection.md`

- [ ] **Step 1: 读取修改后文档**

读取完整文档，验证：
1. 步骤2结构正确（含2.1/2.2/2.3三个子步骤）
2. 交互规则表格完整（6条规则）
3. 红线表格新增3条

预期输出：文档结构符合设计文档要求。

---

## Task 5: 提交修改

**Files:**
- Commit: `processes/evalset-model-selection.md`

- [ ] **Step 1: Git add 并提交**

```bash
git add .claude/skills/model-evaluation/processes/evalset-model-selection.md
git commit -m "refactor(process): 改进模型选择流程交互设计

- 将多轮文本输入改为 AskUserQuestion 分页交互
- 每页最多3个模型选项 + 1个更多操作选项
- 添加厂商分组规则和识别关键词
- 补充红线：单页展示限制、页内循环、已选状态标注

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## 自检清单

| 检查项 | 状态 |
|--------|------|
| Spec 覆盖 | ✅ 步骤2重写 + 红线更新 |
| Placeholder scan | ✅ 无 TBD/TODO |
| Type consistency | ✅ 文档修改，无类型问题 |
| 任务粒度 | ✅ 每步骤2-5分钟 |