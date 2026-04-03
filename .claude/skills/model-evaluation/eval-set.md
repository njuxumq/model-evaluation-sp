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
| `evalset-meta.json` 存在 | 任务1、任务2、任务3、任务4 |

全部通过后，进入执行阶段。

---

## 流程速查

| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程1 | 问题集生成 | [evalset-create.md](./processes/evalset-create.md) | 任务1步骤2（无评测集分支） |
| 流程3 | 评测集解析 | [evalset-parse.md](./processes/evalset-parse.md) | 任务2步骤1 |
| 流程7 | 推理模型选择 | [evalset-model-selection.md](./processes/evalset-model-selection.md) | 任务2步骤2（all_empty分支） |
| 流程8 | 字段校验 | [evalset-field-validation.md](./processes/evalset-field-validation.md) | 任务2步骤2（all_filled分支） |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务3步骤4（定制用例级） |

---

## 任务列表

### 任务1：获取评测集

**目标**：判断评测集来源 → 获取评测集文件 → 进入解析阶段。

**输出**：评测集文件路径或调用 evalset-create 流程。

---

#### 步骤1：判断评测集来源

**判断**：检查历史对话中是否有文件路径或下载链接。

| 识别结果 | 判断依据 | 后续动作 |
|----------|----------|----------|
| 已有评测集 | 文件路径或文件描述 | → 任务2 |
| 无评测集 | 用户明确表示没有评测集或需要生成 | → 步骤2 |
| 无法判断 | 无相关信息 | → 询问用户 |

**禁止**：在任务2执行 analysis 前，不得 Read 文件内容或分析字段。

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

---

#### 步骤2：处理无评测集分支

当用户选择生成问题集时，执行流程1（evalset-create），完成后进入任务2。

> **注意**：若用户选择生成问题集，执行 evalset-create 流程，完成后继续任务2。

---

### 任务2：解析与预处理

**目标**：解析评测集 → 状态判断 → 执行对应预处理流程。

**输出**：`evalset-status.json` + 流程7/8输出文件。

---

#### 步骤1：执行解析流程

执行流程3（评测集解析），产出 `evalset-status.json`。

---

#### 步骤2：状态判断与分支处理

读取 `evalset-status.json` 中 `answer.status`，按表格执行：

| 状态 | 含义 | 后续处理 |
|------|------|----------|
| `all_empty` | answer 全空 | → 执行流程7 → 任务3 |
| `all_filled` | answer 全非空 | → 执行流程8 → 任务3 |
| `partial` | answer 部分填充 | → 询问用户 → 任务3 |

**判断结果输出**：

```
📋 判断结果：答案{状态} → {后续处理}
```

**partial 分支处理**（按需执行）

询问用户："answer 字段部分填充，请确认评测集类型："

- 只有问题 → 进入模型选择流程
- 问题+答案 → 进入字段校验流程

| 选择 | 后续流程 |
|------|----------|
| 只有问题 | → 流程7 → 任务3 |
| 问题+答案 | → 流程8 → 任务3 |

---

### 任务3：标准化与评测点生成

**目标**：标准化转换 → 按需生成评测点。

**输出**：`evalset-standard.jsonl` + `keypoints.json`（按需）。

---

#### 步骤1：判断标准化场景

检查解析流程输出文件：

| 输出文件 | 评测集类型 | 标准化命令 |
|----------|------------|------------|
| `selected-models.json` 存在 | 只有问题 | `expand` 子命令 |
| 仅 `evalset-fields-mapping.json` 存在 | 问题+答案 | `normalize` 子命令 |

---

#### 步骤2：执行标准化命令

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

#### 步骤3：判断评测方式

读取 `eval-dimension.json` 检查评测方式：

| 评测方式 | keypoint 字段状态 | 动作 |
|----------|-------------------|------|
| 通用维度级 | - | 跳过 → 任务4 |
| 定制用例级 | 存在且全部非空 | 跳过 → 任务4 |
| 定制用例级 | 不存在或部分为空 | → 步骤4 |

**如何判断评测方式**：
检查 `eval-dimension.json` 中 `evals` 数组首个元素的 `prompt.keypoints_prompt` 字段：
- 存在 → 定制用例级
- 不存在 → 通用维度级

**多模型横评场景**：若评测集包含多个模型对同一问题的回答（通过 `case_id` 关联），评测点按唯一 `case_id` 生成，避免重复。详见 [keypoint-process.md](./processes/keypoint-process.md#前置判断多模型横评场景)。

---

#### 步骤4：执行评测点生成流程

执行流程4（keypoint-process），完成后进入任务4。

---

### 任务4：上传评测集

**目标**：上传标准化评测集 → 产出元数据文件。

**输出**：`{work-dir}/.eval/{session-id}/evalset/evalset-meta.json`

---

#### 步骤1：执行上传命令

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py submit \
  --auth {work-dir}/.eval/auth.json \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --evalset {work-dir}/.eval/{session-id}/evalset/evalset-standard.jsonl \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-meta.json
```

**失败处理**：参考 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码)。

---

## Red Flags

| 违规行为 | 简洁理由 |
|----------|----------|
| 跳过脚本执行直接分析内容 | 字段结构需脚本解析，直觉判断不可靠 |
| 跳过前置验证 | 评测方式和评委配置影响评测集处理逻辑 |
| 跳过字段映射确认 | 字段映射必须经用户确认后才能标准化 |
| 跳过格式验证 | 评测集格式检查可防止上传失败 |
| 帮助用户生成评测集 | 评测集需用户真实数据，AI生成无法代表实际场景 |

> 通用违规行为见 [SKILL.md Red Flags](./SKILL.md#red-flags---停止并检查)

---

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 评测集解析失败 | 文件格式损坏或不支持 | 检查格式，提供支持格式列表 |
| 字段映射不明确 | 原始字段名与标准字段差异大 | 展示示例数据并请求确认 |
| 上传失败 | API 错误 | 参考 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码) |

---

## 变量速查

变量定义见 [SKILL.md 变量速查](./SKILL.md#变量速查)