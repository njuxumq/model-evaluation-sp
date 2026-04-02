---
name: inference-template-design
description: 推理配置模板设计文档，用于评测集无答案场景的模型推理步骤配置
type: project
---

# 推理配置模板设计

## 背景

评测集解析流程已支持"只有问题"场景（answer 字段为空）。此场景下，评测服务需要先执行模型推理生成答案，再进行评测。

**当前状态**：
- `evalset-parse.md` 流程已支持检测 answer 字段状态
- `evalset-model-selection.md` 流程支持用户选择推理模型
- `selected-models.json` 保存推理模型信息

**缺失部分**：
- 提交评测任务时未配置推理步骤（`inference` 模板）
- 顶层 `models` 字段未包含推理模型信息

---

## 目标

在提交评测任务时，自动为"只有问题"场景添加推理配置：
1. 顶层 `models` 字段合并推理模型和评委模型
2. `templates` 数组新增 `inference` 模板，位于 `evaluation` 之前

---

## 设计方案

### 方案选择

采用**显式参数传递**方案（方案 B）：
- 脚本新增 `--inference_models` 参数，接收推理模型列表文件路径
- Skill 文档在提交任务前检测场景，显式传递参数

**Why**: 配置灵活性高，职责分离清晰，便于调试和扩展。

---

## 改动范围

| 文件 | 改动内容 |
|------|----------|
| `scripts/eval_task.py` | 新增 `--inference_models` 参数，构建推理配置 |
| `eval-execute.md` | 任务1新增推理配置检测逻辑 |
| `references/脚本定义.md` | 补充 `--inference_models` 参数说明 |
| `references/评测服务接口说明.md` | 补充 `inference` 类型模板定义 |
| `references/中间产物说明.md` | 补充推理配置数据流说明 |

---

## 详细设计

### 1. 脚本改动

#### 新增参数

```bash
python eval_task.py submit \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --auth {work-dir}/.eval/auth.json \
  --eval_set {work-dir}/.eval/{session-id}/evalset/evalset-meta.json \
  --eval_dimension {work-dir}/.eval/{session-id}/eval-dimension.json \
  --eval_judge {work-dir}/.eval/{session-id}/eval-judge.json \
  --inference_models {work-dir}/.eval/{session-id}/selected-models.json \
  --output {work-dir}/.eval/{session-id}/evaltask/evaltask-meta.json
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--inference_models` | 否 | 推理模型列表文件路径，不传则跳过推理配置 |

#### 构建逻辑

```python
def build_task_request(..., inference_models_path=None):
    # 基础配置（现有逻辑）
    task_request = {
        "apiVersion": "v1",
        "models": [],
        "agents": [],
        "spec": {"templates": []}
    }

    # 获取 evalset_id
    evalset_id = evalset_meta["dataset"]

    # 处理推理模型（新增）
    if inference_models_path:
        selected_models = load_json(inference_models_path)

        # 1. 构建推理模型列表（顶层 models）
        inference_models = [
            {
                "id": model["id"],
                "name": model["name"],
                "model": model["model"]
            }
            for model in selected_models["models"]
        ]

        # 2. 构建 inference 模板
        inference_template = {
            "name": "模型推理",
            "type": "inference",
            "parameters": {
                "evalset": evalset_id,  # 填充实际 evalset_id
                "models": [{"id": m["id"]} for m in selected_models["models"]]
            }
        }

        # 3. 合并到顶层 models
        task_request["models"] = inference_models + judge_models

        # 4. 插入 templates（inference 在 evaluation 之前）
        task_request["spec"]["templates"].insert(0, inference_template)
    else:
        # 无推理配置，仅使用评委模型
        task_request["models"] = judge_models

    # 添加 evaluation 模板（现有逻辑）
    task_request["spec"]["templates"].append(evaluation_template)

    return task_request
```

---

### 2. Skill 文档改动

#### eval-execute.md 任务1新增逻辑

在"提交命令"执行前，新增判断步骤：

```
**判断**：`selected-models.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 存在 | 添加 `--inference_models` 参数 |
| 不存在 | 使用原命令（无推理配置） |
```

**提交命令（两种场景）**：

```bash
# 有推理配置（评测集无答案场景）
{python-env}{python-cmd} "{skill-dir}/scripts/eval_task.py" submit \
  --config "{skill-dir}/scripts/cfg/eval-server.cfg" \
  --auth "{work-dir}/.eval/auth.json" \
  --eval_set "{work-dir}/.eval/{session-id}/evalset/evalset-meta.json" \
  --eval_dimension "{work-dir}/.eval/{session-id}/eval-dimension.json" \
  --eval_judge "{work-dir}/.eval/{session-id}/eval-judge.json" \
  --inference_models "{work-dir}/.eval/{session-id}/selected-models.json" \
  --output "{work-dir}/.eval/{session-id}/evaltask/evaltask-meta.json"

# 无推理配置（评测集有答案场景）
{python-env}{python-cmd} "{skill-dir}/scripts/eval_task.py" submit \
  --config "{skill-dir}/scripts/cfg/eval-server.cfg" \
  --auth "{work-dir}/.eval/auth.json" \
  --eval_set "{work-dir}/.eval/{session-id}/evalset/evalset-meta.json" \
  --eval_dimension "{work-dir}/.eval/{session-id}/eval-dimension.json" \
  --eval_judge "{work-dir}/.eval/{session-id}/eval-judge.json" \
  --output "{work-dir}/.eval/{session-id}/evaltask/evaltask-meta.json"
```

---

### 3. API 文档补充

#### TaskTemplate.type 枚举补充

| 类型 | 必填 | 说明 |
|------|------|------|
| `evalset` | 否 | 评测集准备步骤 |
| `inference` | 否 | **模型推理步骤（新增）** |
| `evaluation` | 是 | 模型评测步骤（必需） |
| `report-upload` | 否 | 报告上传步骤 |
| `database` | 否 | 数据入库步骤 |

#### inference 类型定义

**parameters 字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `evalset` | `string` | 是 | 评测集标识（填充实际 evalset_id） |
| `models` | `array[InferenceModel]` | 是 | 推理模型列表 |

**InferenceModel 定义**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 模型服务标识 |

---

### 4. 数据流说明

```
构建阶段产物：
  selected-models.json → 推理模型信息
  evalset-meta.json → evalset_id (dataset 字段)
  eval-judge.json → 评委模型信息

执行阶段：
  任务1：检测 selected-models.json 存在
    │
    ▼
  传递 --inference_models 参数
    │
    ▼
  eval_task.py submit 构建：
    - 顶层 models = 推理模型 + 评委模型
    - templates = [inference, evaluation]
    │
    ▼
  提交至评测服务
```

---

## 请求示例

**完整请求体（有推理配置）**：

```json
{
  "apiVersion": "v1",
  "models": [
    {
      "id": "2402509314320990",
      "name": "DeepSeek Chat",
      "model": "deepseek-chat"
    },
    {
      "id": "2402509314320991",
      "name": "Spark Lite",
      "model": "spark-lite"
    },
    {
      "id": "ID_AJ001",
      "name": "DeepSeek-R1",
      "type": "api-openai",
      "model": "xdeepseekr1",
      "concurrency": 50,
      "params": {
        "max_tokens": 16384,
        "temperature": 0
      }
    }
  ],
  "spec": {
    "templates": [
      {
        "name": "模型推理",
        "type": "inference",
        "parameters": {
          "evalset": "eval-bw7adghvb",
          "models": [
            {"id": "2402509314320990"},
            {"id": "2402509314320991"}
          ]
        }
      },
      {
        "name": "模型评测",
        "type": "evaluation",
        "parameters": {
          "evalset": "eval-bw7adghvb",
          "eval": [...]
        }
      }
    ]
  }
}
```

---

## How to apply

1. **脚本实现**：修改 `eval_task.py` submit 子命令，新增参数和构建逻辑
2. **Skill 文档更新**：在 `eval-execute.md` 任务1添加检测逻辑
3. **参考文档补充**：更新 API 文档和脚本定义文档
4. **测试验证**：使用"只有问题"场景评测集验证推理配置正确生成