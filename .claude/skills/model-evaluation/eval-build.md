---
name: eval-build
description: Use when initialization completed and need to configure evaluation dimensions, judges, or process evaluation datasets
---

# 评测构建阶段

## 目标

完成评测场景确认、维度配置、评委配置、评测集处理后，进入执行阶段。

核心原则：**按序执行，用户确认优先，配置校验后置**。

**前置验证**：`auth.json` 存在、`{session-id}/` 目录存在。验证失败则返回初始化阶段。

---

## 何时使用

- 初始化阶段已完成（环境、鉴权、会话目录就绪）
- 需要配置评测场景、维度、评委或评测集时

---

## 阶段完成标志

| 验证条件 | 不满足时执行 |
|----------|--------------|
| `eval-dimension.json` 存在 | 任务1、任务2 |
| `eval-judge.json` 存在 | 任务3 |
| `evalset-meta.json` 存在 | 任务4 |

全部通过后，进入执行阶段。

---

## 流程速查

| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程3 | 评测集解析 | [evalset-parse-process.md](./processes/evalset-parse-process.md) | 任务4步骤2 |
| 流程4 | 评测点生成 | [keypoint-process.md](./processes/keypoint-process.md) | 任务4步骤4 |
| 流程5 | 定制用例级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程5定制用例级评测配置) | 任务2步骤3 |
| 流程6 | 通用维度级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程6通用维度级评测配置) | 任务2步骤3 |

---

## 任务列表

### 任务1：确认评测场景

**目标**：收集用户需求 → 确认评测场景。

---

#### 步骤1：收集用户需求

**判断1**：分析历史对话，识别用户已提供的信息。

| 信息类型 | 识别依据 | 后续处理 |
|----------|----------|----------|
| 评测场景 | 场景名称或描述 | 必需。若未提供则询问 |
| 评测方式 | "维度级"/"用例级"/"评测要点"等关键词 | 若用户提供则保存，否则跳过 |
| 评测维度 | 具体维度名称列表 | 若用户提供则保存，否则跳过 |

| 场景识别结果 | 动作 |
|--------------|------|
| 场景已明确 | 输出识别结果（含可选信息），进入步骤2 |
| 场景不明确 | → 判断2 |

**判断2**：展示内置模板列表，等待用户选择。

参考 [内置模板说明.md](./references/内置模板说明.md) 第1节"专家模板"。

**注意** 以**表格形式**展示（含序号、模板名称、适用说明、评测方式），末尾添加"其他场景"选项。让用户输入序号。

当选择"其他场景"后，需进一步记录场景描述，再进入步骤2。

---

#### 步骤2：确认场景信息

输出识别结果，等待用户确认。

| 条件 | 动作 |
|------|------|
| 用户已明确给出具体场景名称 | 无需确认，进入任务2 |
| 用户提供场景描述或多个场景名称 | 输出识别结果，等待用户确认 |

确认后进入任务2。

---

### 任务2：确认评测标准

**目标**：匹配专家模板 → 自定义配置（按需） → 确认配置摘要。

**输出**：`{work-dir}/.eval/{session-id}/eval-dimension.json`

---

#### 步骤1：匹配专家模板

参考 [内置模板说明.md](./references/内置模板说明.md) 第1节"专家模板"，根据任务1确认的评测场景匹配模板。

| 匹配结果 | 动作 |
|----------|------|
| 匹配成功 | 复制模板 → 进入步骤4 |
| 匹配失败 | → 步骤2 |

**复制命令**：
```bash
cp {skill-dir}/assets/experts/{template-name}.json {work-dir}/.eval/{session-id}/eval-dimension.json
```

---

#### 步骤2：确定评测方式

**判断**：分析任务1收集的评测方式信息。

| 评测方式 | 判断依据 |
|----------|----------|
| 定制用例级 | 用户明确提及"用例级"或"评测要点" |
| 通用维度级 | 用户明确提及"维度级" |
| 未明确 | 无相关信息 | → 询问用户 |

**询问用户**：

请选择评测方式：
1. 通用维度级评测 - 宏观全面覆盖，适用标准评测场景
2. 定制用例级评测 - 细节偏好对齐，适用定制化需求

确认后进入步骤3。

---

#### 步骤3：自定义评测配置

根据评测方式执行对应流程：

| 评测方式 | 执行流程 |
|----------|----------|
| 定制用例级 | 执行流程5 → 步骤4 |
| 通用维度级 | 执行流程6 → 步骤4 |

---

#### 步骤4：确认配置摘要

展示维度配置摘要，等待用户确认或调整。

| 调整类型 | 处理方式 |
|----------|----------|
| 权重调整 | 更新权重，确保总和为1.0 |
| 维度增删 | 从内置库选择或删除 |
| 高级调整 | 提供配置路径 `{work-dir}/.eval/{session-id}/eval-dimension.json`，用户自行编辑 |

**⚠️ 不可跳过**：必须等待用户确认或调整维度配置。

---

### 任务3：确认评委配置

#### 步骤1：检查 `eval-judge.json` 是否存在。

- **文件有效** → 进入任务4
- **文件无效或不存在** → 进入步骤2

#### 步骤2：复制默认配置：
```bash
cp {skill-dir}/assets/eval-judge.json {work-dir}/.eval/{session-id}/eval-judge.json
```

---

### 任务4：确认评测集

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
  --models {work-dir}/.eval/{session-id}/evalset/selected-models.json \
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

**判断**：是否需要评测点生成？

| 评测方式 | keypoint 字段状态 | 动作 |
|----------|-------------------|------|
| 通用维度级 | - | 跳过 → 步骤5 |
| 定制用例级 | 存在且全部非空 | 跳过 → 步骤5 |
| 定制用例级 | 不存在或部分为空 | 执行流程4 → 步骤5 |

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

## Red Flags

| 违规行为 | 简洁理由 |
|----------|----------|
| 跳过场景确认 | 场景是专家模板匹配的必需依据 |
| 跳过维度确认 | 维度权重影响评测结果，必须经用户确认 |
| 跳过映射确认 | 字段映射必须经用户确认后才能标准化 |
| 权重设置不正确 | 所有维度权重总和必须为1.0 |
| 跳过格式验证 | 评测集格式检查可防止上传失败 |
| 帮助用户生成评测集 | 评测集需用户真实数据，AI生成无法代表实际场景 |

> 通用违规行为见 [SKILL.md Red Flags](./SKILL.md#red-flags---停止并检查)

---

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 场景识别失败 | 描述不清晰或超出预设场景 | 明确询问用户或要求补充说明 |
| 专家模板匹配失败 | 场景不在预设模板库 | 进入自定义流程，参考内置维度 |
| 维度配置错误 | 模板格式不符合规范 | 检查 evals 数组结构和 judge_id 字段 |
| 评测集解析失败 | 文件格式损坏或不支持 | 检查格式，提供支持格式列表 |
| 字段映射不明确 | 原始字段名与标准字段差异大 | 展示示例数据并请求确认 |

> API 错误码见 [评测服务接口说明.md](./references/评测服务接口说明.md#错误码)

---

## 变量速查

变量定义见 [SKILL.md 变量速查](./SKILL.md#变量速查)