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
| 分支B | 字段校验与确认 | answer 非空时执行 |

---

## 步骤1：获取评测集文件

**判断**：`evalset-prepared.{ext}` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤2 |
| 不存在 | 获取用户文件 |

获取文件命令：

```bash
# 复制用户文件
cp {用户路径} {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext}

# 或远程下载
curl -o {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} {下载链接}
```

> `{ext}` 从文件路径提取（如 `data.xlsx` → `ext=xlsx`）。

---

## 步骤2：解析字段结构

**判断**：`evalset-structure.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤3 |
| 不存在 | 执行解析 |

解析命令：

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py analysis \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-structure.json
```

**输出说明**：

| 字段 | 说明 |
|------|------|
| `file` | 文件路径 |
| `format` | 文件格式（json/jsonl/csv/xlsx） |
| `total_rows` | 总行数 |
| `fields` | 字段信息（字段名+类型） |
| `answer` | answer 字段状态（新增） |

---

## 步骤3：生成初始映射

**判断**：`evalset-fields-mapping.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤4 |
| 不存在 | 生成映射 |

读取结构文件 → 匹配字段 → 生成映射配置。

### 字段匹配关键词表

| 标准字段 | 含义 | 匹配关键词 | 匹配规则 |
|----------|------|------------|----------|
| question | 评测输入问题 | question, prompt, input, query, 问题, 提问 | 精确优先，包含次之 |
| answer | 模型实际回答 | answer, response, output, reply, 回答, 回复 | 精确优先，包含次之 |
| model | 生成回答的模型标识 | model, model_name, model_id, llm, llm_name, 模型, 模型名称 | 精确优先，包含次之 |
| case_id | 用例唯一标识，用于关联同一问题的多模型回答 | case_id, caseid, 用例id | **仅精确匹配**，未命中时用户确认 |
| system | 系统提示词 | system, system_prompt, 系统提示 | 精确优先，包含次之 |
| context | 附加上下文信息 | context, 上下文 | 精确优先，包含次之 |
| category | 用例分类标签 | category, type, 分类, 类别 | 精确优先，包含次之 |
| reference | 参考答案，用于评分对比 | reference, ref, gold, 参考答案, 标准答案 | 精确优先，包含次之 |
| keypoint | 评测关键点，用于细粒度评分 | keypoint, keypoints, 关键点, 评测点 | 精确优先，包含次之 |

> 包含匹配优先更长关键词（如 `model_name` 优先于 `model`）。

**必填字段**：question、answer、model、case_id。

### 映射格式

```json
{
  "question": {"source_field": "问题", "default": null},
  "answer": {"source_field": "回答", "default": null},
  "model": {"source_field": "模型名称", "default": null},
  "case_id": {"source_field": "id", "default": null}
}
```

**生成规则**：匹配到 → `source_field=源字段名`；未匹配到 → `source_field=null, default=待确认`。

> `case_id` 的 `default` 不使用，有 `source_field` 用源数据值，无则自动生成。

---

## 步骤4：判断后续走向

**判断**：调用 `check-status` 子命令，检查关键字段状态。

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py check-status \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-status.json
```

**输出文件**：`evalset-status.json`

```json
{
  "answer": {
    "exists": true,
    "source_field": "答案",
    "status": "all_filled"
  },
  "model": {
    "exists": true,
    "source_field": "模型名称",
    "status": "all_filled"
  },
  "case_id": {
    "exists": false,
    "source_field": null,
    "status": "all_empty"
  }
}
```

**判断逻辑**：读取 `evalset-status.json` 中 `answer.status` 字段。

| status | 后续动作 |
|--------|----------|
| `all_empty` | answer 字段不存在或全空 → 执行分支A |
| `all_filled` | answer 字段全部非空 → 执行分支B |
| `partial` | answer 字段部分为空 → 询问用户处理方式 |

**partial 状态处理**：

询问用户：
> 评测集中部分 answer 为空，请选择处理方式：
> 1. 视为"只有问题"（空 answer 的记录将云端推理）
> 2. 视为"问题+答案"（跳过空 answer 的记录）

> **说明**：判断在映射确认后执行，`check-status` 子命令结合映射配置分析原始数据，可正确识别各种字段命名形式（如"答案"、"模型输出"等）。

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

## 分支B：字段校验与确认

**触发条件**：步骤4判断 answer 非空

> **注意**：如果先前评测集全部由AI助手生成，则跳过映射确认步骤（步骤B3），因为评测集已按标准格式生成。

### 步骤B1：检查 model 字段状态

分析原始评测集的 `model` 字段：

| 状态 | 判断依据 | 后续动作 |
|------|----------|----------|
| 不存在或全空 | `model` 字段不存在或所有值为空 | → 步骤B4 |
| 已填充 | `model` 字段存在且有非空值 | → 步骤B2 |

### 步骤B2：检查 case_id 字段状态

| case_id 状态 | 处理方式 |
|--------------|----------|
| 已填充 | 无需调整，标准化时使用原值 |
| 不存在或全空 | 标准化时按 question 分组自动生成 |

> 多模型横评时，同一问题的不同模型回答共享相同 case_id。

### 步骤B3：确认映射配置

**判断**：用户是否已确认映射？

| 状态 | 动作 |
|------|------|
| 已确认 | → 保存并返回 |
| 未确认 | 执行确认流程 |

**映射目的说明**：将评测集的原始字段映射为标准字段，标准化后便于后续统一处理（标准化转换、评测执行、评分判定等环节均基于标准字段工作）。

展示映射表（含标准字段含义），等待用户确认。

> **此步骤必须等待用户确认，不可跳过。**

| 用户选择 | 后续动作 |
|----------|----------|
| Y | → 步骤B4 |
| n | 调整映射，重新确认 |

### 步骤B4：处理 model 字段为空情况

**条件**：步骤B1判断 model 字段为空时执行。

询问用户评测模式：

```
请确认评测模式：
1. 单模型评测 - 评测单个模型的表现
2. 多模型横评 - 多个模型横向对比（需在评测集补充 model 字段）
```

| 选择 | 处理方式 |
|------|----------|
| 1 | 设置 `model.default` 为用户指定的模型名称 |
| 2 | 提示用户在评测集补充 model 字段，重新执行步骤1 |

### 步骤B5：保存

保存至 `{work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json`。

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
| 结构解析报错 | 编码问题 | 检查编码，转为UTF-8 |
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
| `{ext}` | 文件扩展名 |
| `{python-env}` | Python环境变量前缀 |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |