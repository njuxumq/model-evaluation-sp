# evalset-parse.md 重构实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构 evalset-parse.md 文档，使步骤清晰、流程速查完整、产物定义明确。

**Architecture:** 保持现有文档结构，优化步骤描述格式，新增流程速查表和产物定义，统一子流程编号为流程3.1/3.2/3.3。

**Tech Stack:** Markdown 文档编辑。

---

## 文件结构

| 文件 | 操作 | 职责 |
|------|------|------|
| `processes/evalset-parse.md` | 重写 | 主文档重构 |
| `processes/evalset-field-mapping.md` | 修改 | 调用方/返回点更新 |
| `processes/evalset-model-selection.md` | 修改 | 调用方更新 |
| `processes/evalset-field-validation.md` | 修改 | 调用方更新 |
| `eval-set.md` | 修改 | 流程速查表增加子流程 |

---

### Task 1: 重写 evalset-parse.md

**Files:**
- Rewrite: `.claude/skills/model-evaluation/processes/evalset-parse.md`

- [ ] **Step 1: 备份原文档**

```bash
cp .claude/skills/model-evaluation/processes/evalset-parse.md .claude/skills/model-evaluation/processes/evalset-parse.md.bak
```

- [ ] **Step 2: 写入重构后文档**

完整替换文件内容：

```markdown
---
name: evalset-parse
description: Use when eval-build reaches evalset parsing step, or user directly requests parsing an evaluation dataset
---

# 评测集解析流程

## 目标

解析评测集 → 检测状态 → 调用子流程 → 返回标准化阶段。

核心原则：**步骤顺序执行，检测结果决定子流程**。

---

## 流程速查

| 编号 | 名称 | 调用时机 |
|------|------|----------|
| 流程3.1 | 字段映射 | 步骤3 |
| 流程3.2 | 模型选择 | 步骤4检测结果为空 |
| 流程3.3 | 字段校验 | 步骤4检测结果为非空 |

---

## 步骤1：获取评测集文件

**目的**：确保评测集文件就位。

| 状态 | 动作 |
|------|------|
| `evalset-prepared.{ext}` 已存在 | → 步骤2 |
| 不存在 | 获取用户文件 |

获取方式：复制用户文件或远程下载至 `{work-dir}/.eval/{session-id}/evalset/`。

---

## 步骤2：解析字段结构

**目的**：识别评测集的字段组成，为字段映射提供依据。

| 状态 | 动作 |
|------|------|
| `evalset-structure.json` 已存在 | → 步骤3 |
| 不存在 | 执行下方命令 |

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py analysis \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-structure.json
```

输出字段：`file`, `format`, `total_rows`, `fields`。

---

## 步骤3：生成字段映射

**目的**：建立原始字段与标准字段的对应关系。

| 状态 | 动作 |
|------|------|
| `evalset-fields-mapping.json` 已存在 | → 步骤4 |
| 不存在 | 执行流程3.1 |

执行 **流程3.1** 完成字段映射生成与确认。

---

## 步骤4：检测评测集状态

**目的**：判断 answer 字段状态，决定后续处理流程。

**必须执行脚本检测**，不得凭直觉判断。

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py check-status \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-status.json
```

读取 `evalset-status.json` 中 `answer.status`：

| status | 后续流程 |
|--------|----------|
| `all_empty` | → 流程3.2 |
| `all_filled` | → 流程3.3 |
| `partial` | 询问用户 |

**partial 处理**：询问用户视为"只有问题"或"问题+答案"。

---

## 产物

| 文件 | 用途 |
|------|------|
| `evalset-structure.json` | 字段结构分析 |
| `evalset-fields-mapping.json` | 字段映射配置 |
| `evalset-status.json` | 状态检测结果 |
| `selected-models.json` | 模型列表（流程3.2产出） |

---

## Red Flags

- 未执行 check-status 就选择流程
- 用户未回复时自动继续

---

## 变量速查

见 [SKILL.md 变量速查](../SKILL.md#变量速查)。
```

---

### Task 2: 修改 evalset-field-mapping.md

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-field-mapping.md`

- [ ] **Step 1: 修改调用方描述**

将第8行 `**调用方**：evalset-parse.md 步骤3` 改为：

```markdown
**调用方**：流程3步骤3
```

- [ ] **Step 2: 修改返回点描述**

将第87行 `**返回**：evalset-parse.md 步骤4` 改为：

```markdown
**返回**：流程3步骤4
```

---

### Task 3: 修改 evalset-model-selection.md

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-model-selection.md`

- [ ] **Step 1: 修改调用方描述**

将第8行 `**调用方**：evalset-parse.md 步骤4（分支A）` 改为：

```markdown
**调用方**：流程3步骤4（检测结果为空）
```

- [ ] **Step 2: 修改返回点描述**

将第74行 `**返回**：标准化阶段` 改为：

```markdown
**返回**：标准化阶段（eval-set.md 任务1步骤3）
```

---

### Task 4: 修改 evalset-field-validation.md

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-field-validation.md`

- [ ] **Step 1: 修改调用方描述**

将第8行 `**调用方**：evalset-parse.md 步骤4（分支B）` 改为：

```markdown
**调用方**：流程3步骤4（检测结果为非空）
```

- [ ] **Step 2: 修改返回点描述**

将第81行 `**返回**：标准化阶段` 改为：

```markdown
**返回**：标准化阶段（eval-set.md 任务1步骤3）
```

---

### Task 5: 修改 eval-set.md

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-set.md`

- [ ] **Step 1: 扩展流程速查表**

将第35-39行的流程速查表从：

```markdown
| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程1 | 问题集生成 | [evalset-create.md](./processes/evalset-create.md) | 任务1步骤1（无评测集分支） |
| 流程3 | 评测集解析 | [evalset-parse.md](./processes/evalset-parse.md) | 任务1步骤2 |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务1步骤4 |
```

改为：

```markdown
| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程1 | 问题集生成 | [evalset-create.md](./processes/evalset-create.md) | 任务1步骤1（无评测集分支） |
| 流程2 | 答案补充 | [evalset-supplement.md](./processes/evalset-supplement.md) | 问题集生成后补充答案 |
| 流程3 | 评测集解析 | [evalset-parse.md](./processes/evalset-parse.md) | 任务1步骤2 |
| 流程3.1 | 字段映射 | [evalset-field-mapping.md](./processes/evalset-field-mapping.md) | 流程3步骤3 |
| 流程3.2 | 模型选择 | [evalset-model-selection.md](./processes/evalset-model-selection.md) | 流程3步骤4检测结果为空 |
| 流程3.3 | 字段校验 | [evalset-field-validation.md](./processes/evalset-field-validation.md) | 流程3步骤4检测结果为非空 |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务1步骤4 |
```

---

### Task 6: 验证与提交

- [ ] **Step 1: 验证文档结构完整性**

检查 evalset-parse.md 包含以下章节：
- YAML frontmatter
- 目标 + 核心原则
- 流程速查表
- 步骤1-4（含目的说明）
- 产物定义
- Red Flags
- 变量速查引用

- [ ] **Step 2: 验证关联文档一致性**

检查 evalset-field-mapping.md、evalset-model-selection.md、evalset-field-validation.md 的调用方/返回点已更新为流程编号。

- [ ] **Step 3: 验证流程速查表完整性**

检查 eval-set.md 流程速查表包含流程1、流程2、流程3、流程3.1、流程3.2、流程3.3、流程4。

- [ ] **Step 4: 删除备份文件**

```bash
rm .claude/skills/model-evaluation/processes/evalset-parse.md.bak
```

- [ ] **Step 5: 提交更改**

```bash
git add .claude/skills/model-evaluation/processes/evalset-parse.md
git add .claude/skills/model-evaluation/processes/evalset-field-mapping.md
git add .claude/skills/model-evaluation/processes/evalset-model-selection.md
git add .claude/skills/model-evaluation/processes/evalset-field-validation.md
git add .claude/skills/model-evaluation/eval-set.md
git commit -m "refactor(evalset-parse): 重构评测集解析流程文档

改进内容：
- 步骤描述增加目的说明
- 新增流程速查表（流程3.1/3.2/3.3）
- 去掉分支概念，改为流程编号引用
- 新增产物定义
- 变量速查改为引用

关联文档同步更新：
- evalset-field-mapping.md: 调用方/返回点更新
- evalset-model-selection.md: 调用方更新
- evalset-field-validation.md: 调用方更新
- eval-set.md: 流程速查表增加子流程

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Self-Review

**1. Spec coverage:**
- ✅ 步骤描述格式统一 → Task 1
- ✅ 流程速查表 → Task 1, Task 5
- ✅ 去掉分支概念 → Task 1, Task 2-4
- ✅ 产物定义 → Task 1
- ✅ 变量速查引用 → Task 1
- ✅ 关联文档修改 → Task 2-5

**2. Placeholder scan:**
- ✅ 无 TBD/TODO
- ✅ 无 "add validation" 等模糊描述
- ✅ 所有代码完整

**3. Type consistency:**
- ✅ 流程编号一致：流程3.1/3.2/3.3
- ✅ 引用格式一致：流程X步骤Y