---
name: evalset-parse-process
description: Use when user has an evaluation dataset file and needs to parse, analyze type, and prepare for standardization
---

# 评测集解析流程

## 目标
解析评测集 → 判断类型 → 执行对应处理 → 返回标准化阶段

核心原则：**步骤通用，分支条件执行**。

---

## 何时使用
- 用户已有评测集文件
- 需要解析字段结构并判断评测集类型

---

## 流程概览

| 步骤 | 名称 | 执行条件 |
|------|------|----------|
| 步骤1-3 | 解析与映射 | 始终执行 |
| 步骤4 | 判断后续走向 | 始终执行 |
| 分支A | 模型选择 | answer 为空时执行 |
| 分支B | 字段校验 | answer 非空时执行 |

---

## 步骤1：获取评测集文件

检查文件存在、复制/下载到工作目录。

---

## 步骤2：解析字段结构

调用 `analysis` 子命令解析结构，输出新增 `answer` 字段。

---

## 步骤3：生成初始映射

根据字段匹配关键词表生成映射配置。

---

## 步骤4：判断后续走向

**判断**：分析 `evalset-structure.json` 的 `answer` 字段。

| answer.status | 后续动作 |
|---------------|----------|
| `all_empty` | answer 字段不存在或全空 → 执行分支A |
| `all_filled` | answer 字段全部非空 → 执行分支B |
| `partial` | answer 字段部分为空 → 询问用户处理方式，建议执行分支A |

**partial 状态处理**：

询问用户：
> 评测集中部分 answer 为空，请选择处理方式：
> 1. 视为"只有问题"（空 answer 的记录将云端推理）
> 2. 视为"问题+答案"（跳过空 answer 的记录）

---

## 分支A：模型选择

**触发条件**：步骤4判断 answer 为空

### 步骤A1：获取可用模型列表

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py list-models \
  --auth {work-dir}/.eval/auth.json \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --output {work-dir}/.eval/{session-id}/evalset/available-models.json
```

**失败处理**：Token 失效则引导重新授权。

### 步骤A2：用户选择模型

展示模型列表，让用户选择一个或多个。

**示例交互**：

```
可用推理模型：
1. deepseek-r1
2. gpt-4
3. claude-3

请选择模型（单模型输入序号，多模型用逗号分隔，如 1,2）：
```

### 步骤A3：保存选择结果

生成 `selected-models.json`：

```json
{
  "models": ["deepseek-r1", "gpt-4"],
  "mode": "multi"
}
```

| 字段 | 说明 |
|------|------|
| `models` | 用户选择的模型名称列表 |
| `mode` | `single`: 单模型评测 / `multi`: 多模型横评 |

---

## 分支B：字段校验

**触发条件**：步骤4判断 answer 非空

### 步骤B1：检查 model 字段状态

分析原始评测集的 `model` 字段：

| 状态 | 判断依据 | 后续动作 |
|------|----------|----------|
| 不存在或全空 | `model` 字段不存在或所有值为空 | → 步骤B3 |
| 已填充 | `model` 字段存在且有非空值 | → 步骤B2 |

### 步骤B2：检查 case_id 字段状态

| case_id 状态 | 处理方式 |
|--------------|----------|
| 已填充 | 无需调整，标准化时使用原值 |
| 不存在或全空 | 标准化时按 question 分组自动生成 |

### 步骤B3：调整映射配置

**model 字段为空时**：

询问用户评测模式：

```
请确认评测模式：
1. 单模型评测 - 评测单个模型的表现
2. 多模型横评 - 多个模型横向对比（需在评测集补充 model 字段）
```

- 选择1：设置 `model.default` 为用户指定的模型名称
- 选择2：提示用户在评测集补充 model 字段，重新执行步骤1

---

## 返回点

完成后返回 **eval-build.md 任务4步骤3**（标准化转换）。

---

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 文件格式不支持 | 文件不是 JSONL/JSON/CSV/Excel 格式 | 转换为支持的格式后重试 |
| 缺少 question 字段 | 文件不包含问题字段 | 确保文件包含 `question` 字段 |
| 字段映射不明确 | 原始字段名与标准字段差异大 | 展示示例数据并请求确认 |
| 模型列表获取失败 | Token 失效或网络问题 | 引导重新授权或检查网络 |
| 部分 answer 为空 | 数据不一致 | 询问用户选择处理方式 |
| model 字段为空但选择多模型横评 | 缺少必要数据 | 提示用户补充 model 字段 |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |
| `{python-env}` | Python环境变量前缀 |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |