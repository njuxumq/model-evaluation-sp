# eval-build.md 拆分实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 eval-build.md 拆分为两个独立阶段文档，提升逻辑独立性并减少单文档 Token 消耗。

**Architecture:** 保持现有目录结构不变，仅拆分文档内容。新建 eval-set.md 承载评测集处理逻辑，缩减 eval-build.md 至评测标准配置逻辑。

**Tech Stack:** Markdown 文档编辑，无代码变更。

---

## 文件结构

| 文件 | 操作 | 责任 |
|------|------|------|
| `.claude/skills/model-evaluation/SKILL.md` | 修改 | 主入口：更新流程表和概览 |
| `.claude/skills/model-evaluation/eval-build.md` | 修改 | 阶段2：缩减为任务1-3 |
| `.claude/skills/model-evaluation/eval-set.md` | 新建 | 阶段3：包含任务4内容 |
| `.claude/skills/model-evaluation/eval-execute.md` | 修改 | 阶段4：更新前置验证描述 |

---

## Task 1: 更新 SKILL.md 流程表

**Files:**
- Modify: `.claude/skills/model-evaluation/SKILL.md:26-33`

- [ ] **Step 1: 修改流程表**

将原三阶段流程表替换为四阶段：

```markdown
## 处理阶段

| 阶段 | 文档 | 核心任务 |
|------|------|----------|
| 初始化 | [eval-init.md](./eval-init.md) | 环境检测、鉴权验证、会话目录 |
| 构建配置 | [eval-build.md](./eval-build.md) | 场景确认、评测标准确认、评委配置 |
| 评测集处理 | [eval-set.md](./eval-set.md) | 评测集解析、标准化、上传 |
| 执行评测 | [eval-execute.md](./eval-execute.md) | 任务提交、状态监控、结果展示 |

**执行流程**：阶段1 → 阶段2 → 阶段3 → 阶段4 → 评测结束
```

- [ ] **Step 2: 验证修改**

运行: `grep -n "评测集处理" .claude/skills/model-evaluation/SKILL.md`
预期: 输出包含新阶段行

- [ ] **Step 3: 提交**

```bash
git add .claude/skills/model-evaluation/SKILL.md
git commit -m "refactor(skill): 更新流程表为四阶段结构

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 2: 更新 SKILL.md 流程概览

**Files:**
- Modify: `.claude/skills/model-evaluation/SKILL.md:44-54`

- [ ] **Step 1: 修改流程概览**

将原三阶段概览替换为四阶段：

```markdown
> **评测流程概览**
>
> **1. 初始化**
> 环境及依赖检测 → 验证鉴权Token → 确认会话目录
>
> **2. 构建配置**
> 确认评测场景 → 确认评测标准 → 配置评委
>
> **3. 评测集处理**
> 解析评测集 → 标准化转换 → 评测点生成（按需） → 上传评测集
>
> **4. 执行评测**
> 提交评测任务 → 监控执行状态 → 展示评测结果
```

- [ ] **Step 2: 验证修改**

运行: `grep -n "评测集处理" .claude/skills/model-evaluation/SKILL.md`
预期: 输出包含概览中的新阶段

- [ ] **Step 3: 提交**

```bash
git add .claude/skills/model-evaluation/SKILL.md
git commit -m "refactor(skill): 更新流程概览为四阶段

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 3: 缩减 eval-build.md - 删除任务4

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-build.md`

- [ ] **Step 1: 删除任务4内容**

删除从 `### 任务4：确认评测集` 开始到文件末尾（Red Flags、常见错误、变量速查除外）的所有内容。

保留以下章节：
- YAML frontmatter
- 目标
- 何时使用
- 阶段完成标志（需更新）
- 前置验证
- 流程速查（删除流程3、流程4引用）
- 任务1：确认评测场景
- 任务2：确认评测标准
- 任务3：确认评委配置
- Red Flags（保留，删除评测集相关项）
- 常见错误（保留，删除评测集相关项）
- 变量速查

- [ ] **Step 2: 更新阶段完成标志**

将原阶段完成标志替换为：

```markdown
## 阶段完成标志

| 验证条件 | 不满足时执行 |
|----------|--------------|
| `eval-dimension.json` 存在 | 任务1、任务2 |
| `eval-judge.json` 存在 | 任务3 |

全部通过后，进入评测集处理阶段。
```

- [ ] **Step 3: 更新流程速查**

删除流程3、流程4引用，仅保留流程5、流程6：

```markdown
## 流程速查

| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程5 | 定制用例级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程5定制用例级评测配置) | 任务2步骤3 |
| 流程6 | 通用维度级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程6通用维度级评测配置) | 任务2步骤3 |
```

- [ ] **Step 4: 更新 Red Flags**

删除评测集相关的 Red Flags 项，保留：

```markdown
## Red Flags

| 违规行为 | 简洁理由 |
|----------|----------|
| 跳过场景确认 | 场景是专家模板匹配的必需依据 |
| 跳过维度确认 | 维度权重影响评测结果，必须经用户确认 |
| 权重设置不正确 | 所有维度权重总和必须为1.0 |
| 用例级评测调整权重或维度 | 单维度结构固定 |

> 通用违规行为见 [SKILL.md Red Flags](./SKILL.md#red-flags---停止并检查)
```

- [ ] **Step 5: 更新常见错误**

删除评测集相关的常见错误项，保留：

```markdown
## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 场景识别失败 | 描述不清晰或超出预设场景 | 明确询问用户或要求补充说明 |
| 专家模板匹配失败 | 场景不在预设模板库 | 进入自定义流程，参考内置维度 |
| 维度配置错误 | 模板格式不符合规范 | 检查 evals 数组结构和 judge_id 字段 |
| 用例级评测误调整 | 尝试增删维度或调整权重 | 仅允许修改维度配置字段 |

> API 错误码见 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码)
```

- [ ] **Step 6: 更新目标章节**

将"进入执行阶段"改为"进入评测集处理阶段"：

```markdown
## 目标

完成评测场景确认、维度配置、评委配置后，进入评测集处理阶段。

核心原则：**按序执行，用户确认优先，配置校验后置**。

**前置验证**：`auth.json` 存在、`{session-id}/` 目录存在。验证失败则返回初始化阶段。
```

- [ ] **Step 7: 验证文件完整性**

运行: `wc -l .claude/skills/model-evaluation/eval-build.md`
预期: 约 150 行（原 300+ 行缩减）

- [ ] **Step 8: 提交**

```bash
git add .claude/skills/model-evaluation/eval-build.md
git commit -m "refactor(eval-build): 缩减为评测标准配置阶段

删除任务4（评测集处理），保留任务1-3（场景、标准、评委）
更新阶段完成标志和流程速查

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 4: 新建 eval-set.md

**Files:**
- Create: `.claude/skills/model-evaluation/eval-set.md`

- [ ] **Step 1: 创建文档框架**

创建包含 YAML frontmatter 的文档框架：

```markdown
---
name: eval-set
description: Use when initialization and evaluation standards are configured, need to process evaluation dataset.
---

# 评测集处理阶段

## 目标
完成评测集解析、标准化、上传后，进入执行阶段。

核心原则：**按序执行，用户确认优先，格式验证前置**。

**前置验证**：`auth.json` 存在、`eval-dimension.json` 存在、`eval-judge.json` 存在、`{session-id}/` 目录存在。验证失败则返回对应前置阶段。

---

## 何时使用
- 构建配置阶段已完成（场景、标准、评委配置就绪）
- 需要处理评测集时

---

## 阶段完成标志

| 验证条件 | 不满足时执行 |
|----------|--------------|
| `evalset-meta.json` 存在 | 任务1 |

全部通过后，进入执行阶段。

---

## 流程速查

| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程3 | 评测集解析 | [evalset-parse.md](./processes/evalset-parse.md) | 任务1步骤2 |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务1步骤4 |

---
```

- [ ] **Step 2: 添加任务1内容**

从原 eval-build.md 任务4复制内容，添加任务1：

```markdown
## 任务列表

### 任务1：确认评测集

**目标**：识别评测集来源 → 执行前置流程 → 解析转换 → 上传。

**输出**：`{work-dir}/.eval/{session-id}/evalset/evalset-meta.json`

---

#### 步骤1：识别评测集来源

**判断**：分析历史对话，查找评测集相关信息。

| 识别结果 | 判断依据 | 后续动作 |
|----------|----------|----------|
| 已有评测集 | 文件路径或文件描述 | → 步骤2 |
| 无法判断 | 无相关信息 | → 询问用户 |

**询问用户**（无法判断时）：

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

> **注意**：若用户未提供评测集，需等待用户提供后方可继续。

---

#### 步骤2：执行解析流程

执行流程3，完成后进入步骤3。

---

#### 步骤3：标准化转换

**判断**：检查解析流程输出文件。

| 输出文件 | 评测集类型 | 标准化命令 |
|----------|------------|------------|
| `selected-models.json` 存在 | 只有问题 | `expand` 子命令 |
| 仅 `evalset-fields-mapping.json` 存在 | 问题+答案 | `normalize` 子命令 |

**只有问题场景**：

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py expand \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --models {work-dir}/.eval/{session-id}/selected-models.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-standard.jsonl
```

**问题+答案场景**：

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py normalize \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-standard.jsonl
```

---

#### 步骤4：评测点生成（按需）

**判断**：读取 `eval-dimension.json` 检查评测方式。

| 评测方式 | keypoint 字段状态 | 动作 |
|----------|-------------------|------|
| 通用维度级 | - | 跳过 → 步骤5 |
| 定制用例级 | 存在且全部非空 | 跳过 → 步骤5 |
| 定制用例级 | 不存在或部分为空 | 执行流程4 → 步骤5 |

**如何判断评测方式**：
检查 `eval-dimension.json` 中 `evals` 数组首个元素的 `prompt.keypoints_prompt` 字段：
- 存在 → 定制用例级
- 不存在 → 通用维度级

---

#### 步骤5：上传评测集

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py submit \
  --auth {work-dir}/.eval/auth.json \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --evalset {work-dir}/.eval/{session-id}/evalset/evalset-standard.jsonl \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-meta.json
```

**失败处理**：参考 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码)。

---
```

- [ ] **Step 3: 添加 Red Flags**

```markdown
## Red Flags

| 违规行为 | 简洁理由 |
|----------|----------|
| 跳过前置验证 | 评测方式和评委配置影响评测集处理逻辑 |
| 跳过字段映射确认 | 字段映射必须经用户确认后才能标准化 |
| 跳过格式验证 | 评测集格式检查可防止上传失败 |
| 帮助用户生成评测集 | 评测集需用户真实数据，AI生成无法代表实际场景 |

> 通用违规行为见 [SKILL.md Red Flags](./SKILL.md#red-flags---停止并检查)
```

- [ ] **Step 4: 添加常见错误**

```markdown
## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 评测集解析失败 | 文件格式损坏或不支持 | 检查格式，提供支持格式列表 |
| 字段映射不明确 | 原始字段名与标准字段差异大 | 展示示例数据并请求确认 |
| 上传失败 | API 错误 | 参考 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码) |

---
```

- [ ] **Step 5: 添加变量速查**

```markdown
## 变量速查

变量定义见 [SKILL.md 变量速查](./SKILL.md#变量速查)
```

- [ ] **Step 6: 验证文件创建**

运行: `ls -la .claude/skills/model-evaluation/eval-set.md`
预期: 文件存在，约 150 行

- [ ] **Step 7: 提交**

```bash
git add .claude/skills/model-evaluation/eval-set.md
git commit -m "feat(eval-set): 新建评测集处理阶段文档

从 eval-build.md 拆分评测集处理逻辑（原任务4）
包含解析、标准化、评测点生成、上传流程

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 5: 更新 eval-execute.md 前置验证

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-execute.md:14`

- [ ] **Step 1: 更新前置验证描述**

将原前置验证替换为更清晰的描述：

```markdown
**前置验证**：`auth.json` 存在、`eval-dimension.json` 存在、`eval-judge.json` 存在、`evalset-meta.json` 存在。缺失则返回对应前置阶段。
```

- [ ] **Step 2: 更新"何时使用"章节**

将"构建阶段已完成"改为更精确的描述：

```markdown
## 何时使用

- 评测集处理阶段已完成（维度、评测集、评委配置就绪）
- 需要提交、查询或展示评测任务时
```

- [ ] **Step 3: 验证修改**

运行: `grep -n "evalset-meta.json" .claude/skills/model-evaluation/eval-execute.md`
预期: 输出包含前置验证行

- [ ] **Step 4: 提交**

```bash
git add .claude/skills/model-evaluation/eval-execute.md
git commit -m "refactor(eval-execute): 更新前置验证描述

明确依赖 evalset-meta.json，更新阶段引用描述

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 6: 验证拆分完整性

**Files:**
- 无文件变更，仅验证

- [ ] **Step 1: 验证文件数量**

运行: `ls .claude/skills/model-evaluation/*.md | wc -l`
预期: 4 个文件（SKILL.md, eval-init.md, eval-build.md, eval-set.md, eval-execute.md 共 5 个）

- [ ] **Step 2: 验证 eval-build.md 行数**

运行: `wc -l .claude/skills/model-evaluation/eval-build.md`
预期: 约 150 行（缩减后）

- [ ] **Step 3: 验证 eval-set.md 行数**

运行: `wc -l .claude/skills/model-evaluation/eval-set.md`
预期: 约 150 行

- [ ] **Step 4: 验证引用完整性**

检查 eval-set.md 中的引用路径：
- `[evalset-parse.md](./processes/evalset-parse.md)` 应存在
- `[keypoint-process.md](./processes/keypoint-process.md)` 应存在

运行: `ls .claude/skills/model-evaluation/processes/evalset-parse.md .claude/skills/model-evaluation/processes/keypoint-process.md`
预期: 两个文件都存在

- [ ] **Step 5: 最终提交（如有遗漏修复）**

若有任何遗漏修复，在此提交。

---

## Self-Review

**1. Spec coverage:**

| Spec 章节 | Task 覆盖 |
|-----------|-----------|
| SKILL.md 流程表 | Task 1 |
| SKILL.md 流程概览 | Task 2 |
| eval-build.md 缩减 | Task 3 |
| eval-set.md 新建 | Task 4 |
| eval-execute.md 前置验证 | Task 5 |
| 验证清单 | Task 6 |

**2. Placeholder scan:**
- ✅ 无 TBD/TODO
- ✅ 所有代码块包含完整内容
- ✅ 所有步骤包含具体命令

**3. Type consistency:**
- ✅ 文件路径一致
- ✅ 引用路径使用相对路径 `./` 格式