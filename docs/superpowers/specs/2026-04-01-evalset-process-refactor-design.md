# 评测集处理流程重构设计

## 问题描述

**现状问题**：

| 问题 | 影响 |
|------|------|
| 评测集必须包含 `answer` 字段 | 无法处理"只有问题"的场景 |
| 无模型列表接口 | 无法让用户选择推理模型 |
| `normalize` 子命令要求 `answer` 非空 | 无法标准化"只有问题"的评测集 |
| 流程文档结构不清晰 | 难以理解不同场景的处理路径 |

**需求**：

1. 支持两种评测集类型：
   - 只有问题（需云端推理生成答案）
   - 问题+答案（现有流程）

2. "只有问题"场景：
   - 调用接口获取可用推理模型列表
   - 用户选择模型（单模型或多模型横评）
   - 标准化时展开为 N×M 条记录

3. "问题+答案"场景：
   - 校验 model、case_id 字段状态
   - 根据状态调整映射配置

---

## 设计方案

### 核心设计：步骤+分支结构

将 `evalset-parse-process.md` 重构为"通用步骤 + 条件分支"结构：

```
步骤1-4：解析与映射（通用步骤，始终执行）
    │
    ▼
步骤4：判断后续走向
    │
    ├─► 分支A：模型选择（answer 为空时执行）
    │       ├── 步骤A1：获取模型列表
    │       ├── 步骤A2：用户选择模型
    │       └── 步骤A3：保存选择结果
    │       │
    │       ▼
    │   selected-models.json
    │
    └─► 分支B：字段校验（answer 非空时执行）
            ├── 步骤B1：检查 model 字段状态
            ├── 步骤B2：检查 case_id 字段状态
            └── 步骤B3：调整映射配置
            │
            ▼
        更新 mapping 配置
            │
            └────────────────┘
                    │
                    ▼
              标准化转换（normalize/expand）
```

### 场景矩阵

| 场景 | answer | model 字段 | case_id 字段 | 标准化处理 |
|------|--------|------------|--------------|------------|
| S1: 只有问题（无model） | 空 | 不存在 | 不存在 | expand: N×M 条记录 |
| S2: 只有问题（有model） | 空 | 已填充 | 不存在/已填充 | expand: N×M 条记录（忽略原model） |
| S3: 问题+答案（单模型） | 非空 | 空/不存在 | 空/不存在 | normalize: 填充默认值 |
| S4: 问题+答案（多模型） | 非空 | 已填充 | 空 | normalize: 按question分组生成case_id |
| S5: 问题+答案（完整） | 非空 | 已填充 | 已填充 | normalize: 直接映射转换 |

---

## 文件修改清单

| 文件 | 修改类型 | 说明 |
|------|----------|------|
| `scripts/eval_set.py` | 修改 + 新增 | 调整 `analysis` 输出；新增 `list-models`、`expand` 子命令 |
| `scripts/clients/api_client.py` | 新增方法 | 新增 `get_models()` 方法 |
| `processes/evalset-parse-process.md` | 重写 | 三阶段结构：解析映射、模型选择、字段校验 |
| `eval-build.md` | 修改 | 任务4调整：标准化命令选择逻辑 |
| `references/中间产物说明.md` | 修改 | 新增 `selected-models.json`、`available-models.json`；更新 `evalset-structure.json` |
| `references/评测服务接口说明.md` | 修改 | `answer` 改为可选；新增模型列表接口 |

---

## 详细设计

### 1. `scripts/eval_set.py` 修改

#### 1.1 `analysis` 子命令调整

**新增输出字段**：`answer`

```json
{
  "file": "dataset.jsonl",
  "format": "jsonl",
  "total_rows": 10,
  "fields": {...},
  "answer": {
    "exists": true,
    "status": "all_empty"
  }
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `answer.exists` | `bool` | answer 字段是否存在 |
| `answer.status` | `string` | `all_empty`: 全空 / `partial`: 部分空 / `all_filled`: 全非空 |

**判断逻辑**：

```python
def analyze_answer_field(data: list, fields: dict) -> dict:
    """分析 answer 字段状态"""
    if 'answer' not in fields:
        return {"exists": False, "status": "all_empty"}

    # 检查所有 answer 值
    empty_count = 0
    for item in data:
        answer_value = item.get('answer')
        if is_empty_value(answer_value):
            empty_count += 1

    total = len(data)
    if empty_count == total:
        status = "all_empty"
    elif empty_count == 0:
        status = "all_filled"
    else:
        status = "partial"

    return {"exists": True, "status": status}
```

#### 1.2 新增 `list-models` 子命令

**命令格式**：

```bash
eval_set.py list-models \
  --auth auth.json \
  --config eval-server.cfg \
  --output available-models.json
```

**输出格式**：

```json
{
  "models": ["deepseek-r1", "gpt-4", "claude-3"]
}
```

#### 1.3 新增 `expand` 子命令

**职责**：处理 answer 为空的场景，展开评测集（N×M 条记录）

**命令格式**：

```bash
eval_set.py expand \
  --input evalset-prepared.jsonl \
  --mapping evalset-fields-mapping.json \
  --models selected-models.json \
  --output evalset-standard.jsonl
```

**展开逻辑**：

```python
def expand_data(data: list, mapping: dict, models: list) -> list:
    """展开评测集：N问题 × M模型 = N×M条记录"""
    result = []
    case_counter = 0
    question_to_case = {}

    for item in data:
        question = extract_field(item, mapping, 'question')
        if not question:
            continue

        # 生成 case_id（同一问题共享）
        if question not in question_to_case:
            case_counter += 1
            question_to_case[question] = f'case-{case_counter:04d}'
        case_id = question_to_case[question]

        # 为每个模型生成一条记录
        for model in models:
            record = {
                "question": question,
                "answer": "",  # 空字符串
                "model": model,
                "case_id": case_id
            }
            # 添加可选字段
            for field in OPTIONAL_FIELDS:
                value = extract_field(item, mapping, field)
                if value:
                    record[field] = value
            result.append(record)

    return result
```

**处理原 model 字段**：
- **忽略原 model 字段值**
- 使用用户选择的模型列表展开
- 原 model 字段值不保留（推理阶段由服务端填充）

---

### 2. `scripts/clients/api_client.py` 新增方法

```python
def get_models(self) -> list:
    """获取可用推理模型列表

    Returns:
        模型名称列表，如 ["deepseek-r1", "gpt-4", "claude-3"]
    """
    response = self.get("/open/api/v1/models")
    return response.get("models", [])
```

---

### 3. `processes/evalset-parse-process.md` 重写

**文档结构**：

```markdown
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

> **保留现有步骤1内容**：检查文件存在、复制/下载到工作目录。

---

## 步骤2：解析字段结构

> **保留现有步骤2内容**：调用 `analysis` 子命令解析结构。
> **新增**：输出文件 `evalset-structure.json` 新增 `answer` 字段。

---

## 步骤3：生成初始映射

> **保留现有步骤3内容**：根据字段匹配关键词表生成映射配置。

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

**选择格式**：
- 单模型：用户输入模型名称或序号
- 多模型：用户输入多个模型名称或序号（逗号分隔）

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

> **保留现有常见错误表格**，新增以下条目：

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 模型列表获取失败 | Token 失效或网络问题 | 引导重新授权或检查网络 |
| 部分 answer 为空 | 数据不一致 | 询问用户选择处理方式 |
| model 字段为空但选择多模型横评 | 缺少必要数据 | 提示用户补充 model 字段 |

---

## 变量速查

> **保留现有变量速查表格**。
```

---

### 4. `eval-build.md` 任务4修改

**任务4步骤3调整**：标准化转换命令选择

```markdown
### 步骤3：标准化转换

**判断**：检查阶段输出文件。

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
```

---

### 5. `references/中间产物说明.md` 修改

**新增文件说明**：

#### selected-models.json

**路径**：`{work-dir}/.eval/{session-id}/evalset/selected-models.json`

| 字段 | 类型 | 说明 |
|------|------|------|
| `models` | `array[string]` | 用户选择的模型名称列表 |
| `mode` | `string` | `single`: 单模型评测 / `multi`: 多模型横评 |

**示例**：

```json
{
  "models": ["deepseek-r1", "gpt-4"],
  "mode": "multi"
}
```

#### available-models.json

**路径**：`{work-dir}/.eval/{session-id}/evalset/available-models.json`

| 字段 | 类型 | 说明 |
|------|------|------|
| `models` | `array[string]` | 可用推理模型名称列表 |

#### evalset-structure.json 更新

新增 `answer` 字段：

```json
{
  "file": "dataset.jsonl",
  "format": "jsonl",
  "total_rows": 10,
  "fields": {...},
  "answer": {
    "exists": true,
    "status": "all_filled"
  }
}
```

---

### 6. `references/评测服务接口说明.md` 修改

#### 修改 `answer` 字段说明

在 2.1 上传评测集 - EvalsetItem 定义中：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `answer` | `string` | **否** | 模型回答（为空时由推理服务生成） |

#### 新增模型列表接口

**接口URL**：`GET /open/api/v1/models`

**响应参数**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "models": ["deepseek-r1", "gpt-4", "claude-3"]
  }
}
```

---

## 影响范围分析

### 功能一致性

| 功能 | 修改前 | 修改后 | 兼容性 |
|------|--------|--------|--------|
| 解析评测集 | `analysis` 输出无 `answer` 字段 | 新增 `answer` 字段 | 向后兼容（新增字段） |
| 标准化转换 | 仅 `normalize` 子命令 | 新增 `expand` 子命令 | 向后兼容（新增命令） |
| 上传评测集 | `answer` 必填 | `answer` 可选 | 服务端需同步支持 |

### 接口一致性

| 接口 | 变更 | 影响 |
|------|------|------|
| `analysis` 输出 | 新增字段 | 调用方需适配新字段 |
| `normalize` 参数 | 保持不变 | 无影响 |
| 新增 `list-models` | 新接口 | 新增调用点 |
| 新增 `expand` | 新接口 | 新增调用点 |

### 文档一致性

| 文档 | 需同步更新 |
|------|------------|
| `eval-build.md` | 任务4步骤3标准化命令选择 |
| `中间产物说明.md` | 新增产物说明 |
| `评测服务接口说明.md` | `answer` 可选 + 新接口 |

---

## 验证要点

| 验证项 | 验证方法 |
|--------|----------|
| `analysis` 输出正确 | 测试不同 answer 状态的评测集 |
| `list-models` 正常工作 | 测试接口调用和 Token 失效处理 |
| `expand` 展开逻辑正确 | 验证 N×M 条记录、case_id 共享 |
| 三阶段流程清晰 | 验证流程文档可理解性 |
| 向后兼容 | 验证现有"问题+答案"场景正常工作 |