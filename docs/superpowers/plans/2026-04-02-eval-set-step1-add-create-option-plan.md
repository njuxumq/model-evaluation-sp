# eval-set 步骤1新增"生成问题集"选项实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 eval-set.md 步骤1新增"无评测集"分支，支持用户生成问题集。

**Architecture:** 修改两个 markdown 文档（eval-set.md 和 evalset-create.md），新增识别分支、询问选项、流程引用和前置验证逻辑。

**Tech Stack:** Markdown 文档编辑

---

## 文件结构

| 文件 | 操作 | 责任 |
|------|------|------|
| `.claude/skills/model-evaluation/eval-set.md` | Modify | 步骤1表格、询问用户部分、流程速查表 |
| `.claude/skills/model-evaluation/processes/evalset-create.md` | Modify | 步骤1前置验证逻辑 |

---

### Task 1: 修改 eval-set.md 步骤1识别结果表格

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-set.md` 步骤1表格部分

- [ ] **Step 1: 定位当前表格内容**

读取文件，找到步骤1中"识别结果"表格的位置（约第56-60行）。

- [ ] **Step 2: 编辑表格，新增"无评测集"行**

在"已有评测集"行和"无法判断"行之间插入新行：

```markdown
| 识别结果 | 判断依据 | 后续动作 |
|----------|----------|----------|
| 已有评测集 | 文件路径或文件描述 | → 步骤2 |
| 无评测集 | 用户明确表示没有评测集或需要生成 | → 执行 evalset-create 流程 → 步骤2 |
| 无法判断 | 无相关信息 | → 询问用户 |
```

- [ ] **Step 3: 验证修改正确**

读取文件确认表格格式正确，行间对齐。

---

### Task 2: 修改 eval-set.md 步骤1"询问用户"部分

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-set.md` 步骤1询问用户部分

- [ ] **Step 1: 定位当前"询问用户"内容**

读取文件，找到步骤1中"询问用户（无法判断时）"部分的位置（约第62-84行）。

- [ ] **Step 2: 替换询问内容为双选项格式**

将原有单一格式说明替换为提供评测集/生成问题集双选项：

```markdown
**询问用户**（无法判断时）：

请问您是否有现成的评测集？

**选项1：提供评测集**

评测集是评测任务的必需数据源，请提供包含问题和答案的评测集文件。

**支持的格式**：CSV、JSON、JSONL、XLSX

**评测集内容示例**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `question` | `string` | 是 | 评测问题 |
| `answer` | `string` | 是 | 模型回答 |
| `model` | `string` | 是 | 模型标识（默认 `default`） |
| `case_id` | `string` | 否 | 问题标识，用于关联同一问题的多模型回答 |
| `reference` | `string` | 否 | 参考答案 |

**JSONL 格式示例**：

```jsonl
{"question": "什么是大语言模型？", "answer": "大语言模型（LLM）是一种基于深度学习的自然语言处理模型...", "model": "gpt-4"}
{"question": "如何提高代码质量？", "answer": "提高代码质量可以从以下几个方面入手：1. 遵循编码规范...", "model": "gpt-4"}
```

**选项2：生成问题集**

如果您没有评测集，我可以根据已确认的评测场景和维度，帮您生成问题集。

> **前置条件**：需要评测场景和维度已确认（构建配置阶段已完成）。若未完成，系统将引导您先完成构建配置。

请选择：提供评测集 / 生成问题集

> **注意**：若用户选择生成问题集，执行 evalset-create 流程，完成后继续步骤2。
```

- [ ] **Step 3: 验证修改正确**

读取文件确认询问部分格式正确，选项清晰。

---

### Task 3: 修改 eval-set.md 流程速查表

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-set.md` 流程速查部分

- [ ] **Step 1: 定位当前流程速查表**

读取文件，找到"流程速查"表格位置（约第35-39行）。

- [ ] **Step 2: 新增流程1引用行**

在表格中新增"流程1"行（位于流程3之前）：

```markdown
| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程1 | 问题集生成 | [evalset-create.md](./processes/evalset-create.md) | 任务1步骤1（无评测集分支） |
| 流程3 | 评测集解析 | [evalset-parse.md](./processes/evalset-parse.md) | 任务1步骤2 |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务1步骤4 |
```

- [ ] **Step 3: 验证修改正确**

读取文件确认流程速查表完整，编号顺序正确。

---

### Task 4: 修改 evalset-create.md 步骤1

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-create.md` 步骤1

- [ ] **Step 1: 定位当前步骤1内容**

读取文件，找到步骤1"确认生成依据"部分（约第34-49行）。

- [ ] **Step 2: 修改步骤名称**

将步骤名称从"确认生成依据"改为"前置验证与确认生成依据"：

```markdown
## 步骤1：前置验证与确认生成依据
```

- [ ] **Step 3: 替换执行逻辑表格**

将原有执行逻辑表格替换为前置验证优先的版本：

```markdown
**目的**：验证前置条件满足 → 确认生成依据 → 进入配置收集。

**执行逻辑**：

| 状态 | 判断 | 动作 |
|------|------|------|
| 前置已完成 | `eval-dimension.json` 存在 | → 确认场景和维度内容 → 进入步骤2 |
| 前置未完成 | `eval-dimension.json` 不存在 | → 提示用户先完成构建配置阶段（eval-build.md），流程终止 |

**提示用户（前置未完成时）**：

> 生成问题集需要先确认评测场景和评测维度。请先完成构建配置阶段（eval-build.md）中的场景确认和维度配置任务。

**输出变量**（前置已完成时）：
- `{scene}`：从 `eval-dimension.json` 中提取的评测场景描述
- `{dimensions}`：从 `eval-dimension.json` 中提取的评测维度列表
```

- [ ] **Step 4: 验证修改正确**

读取文件确认步骤1逻辑清晰，前置验证优先执行。

---

### Task 5: 提交修改

**Files:**
- All modified files

- [ ] **Step 1: 检查所有修改**

运行 `git status` 确认修改的文件列表：
- `.claude/skills/model-evaluation/eval-set.md`
- `.claude/skills/model-evaluation/processes/evalset-create.md`

- [ ] **Step 2: 提交修改**

```bash
git add .claude/skills/model-evaluation/eval-set.md .claude/skills/model-evaluation/processes/evalset-create.md
git commit -m "$(cat <<'EOF'
feat(eval-set): 步骤1新增"生成问题集"选项

- eval-set.md 步骤1识别结果表格新增"无评测集"分支
- eval-set.md 步骤1询问用户部分新增生成问题集选项
- eval-set.md 流程速查表新增流程1引用
- evalset-create.md 步骤1新增前置验证逻辑

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 3: 验证提交成功**

运行 `git log -1` 确认提交内容。

---

## 自检清单

| 检查项 | 状态 |
|--------|------|
| Spec 覆盖完整 | ✅ 所有4项修改均已覆盖 |
| 无占位符 | ✅ 所有步骤包含具体内容 |
| 类型一致 | ✅ 文档修改无类型问题 |