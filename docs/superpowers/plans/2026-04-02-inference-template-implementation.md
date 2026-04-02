# 推理配置模板实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为评测集无答案场景添加推理配置，使任务提交时自动构建 inference 模板并合并推理模型到顶层 models 字段。

**Architecture:** 采用显式参数传递方案，脚本新增 `--inference_models` 参数接收推理模型列表，Skill 文档在任务提交前检测场景并传递参数。

**Tech Stack:** Python 3.x, argparse, JSON 文件处理

---

## 文件结构

| 文件 | 操作 | 说明 |
|------|------|------|
| `scripts/eval_task.py` | Modify | 新增 `--inference_models` 参数和推理配置构建逻辑 |
| `eval-execute.md` | Modify | 任务1新增推理配置检测与参数传递步骤 |
| `references/脚本定义.md` | Modify | 补充 `--inference_models` 参数说明 |
| `references/评测服务接口说明.md` | Modify | 补充 `inference` 类型模板定义 |
| `references/中间产物说明.md` | Modify | 补充推理配置数据流说明 |

---

## Task 1: 修改脚本新增推理配置参数

**Files:**
- Modify: `.claude/skills/model-evaluation/scripts/eval_task.py`

### Step 1: 新增 CLI 参数定义

在 `main()` 函数的 submit 子命令解析器中添加新参数：

```python
# 在第 284-291 行之间，submit 子命令参数定义区域
p.add_argument('--inference_models', default=None, help='推理模型列表文件(selected-models.json，可选)')
```

**修改位置**：第 290 行（`p.add_argument('--output'...)` 之后）

---

### Step 2: 新增推理模型加载函数

在文件顶部（约第 24-27 行，`cmd_submit` 函数定义之前）添加辅助函数：

```python
def load_inference_models(inference_models_path: str) -> list:
    """加载推理模型列表，返回顶层 models 格式"""
    if not inference_models_path:
        return []

    result = load_json(inference_models_path)
    if not result.get("success"):
        raise ValueError(f"推理模型文件加载失败: {result.get('message')}")

    selected_models = result.get("data", {})
    models = selected_models.get("models", [])

    # 转换为顶层 models 格式
    return [
        {
            "id": m.get("id"),
            "name": m.get("name"),
            "model": m.get("model")
        }
        for m in models
    ]


def build_inference_template(evalset_id: str, inference_models_path: str) -> dict:
    """构建 inference 模板"""
    if not inference_models_path:
        return None

    result = load_json(inference_models_path)
    if not result.get("success"):
        raise ValueError(f"推理模型文件加载失败: {result.get('message')}")

    selected_models = result.get("data", {})
    models = selected_models.get("models", [])

    return {
        "name": "模型推理",
        "type": "inference",
        "parameters": {
            "evalset": evalset_id,
            "models": [{"id": m.get("id")} for m in models]
        }
    }
```

---

### Step 3: 修改 cmd_submit 函数构建逻辑

修改 `cmd_submit` 函数（第 29-89 行），在构建 payload 时处理推理配置：

**修改 payload 构建部分（第 73-84 行）**：

```python
    # 加载推理模型（新增）
    inference_models_list = []
    inference_template = None
    if args.inference_models:
        inference_models_list = load_inference_models(args.inference_models)
        inference_template = build_inference_template(evalset_id, args.inference_models)

    # 构建评委模型列表
    judge_models = [judges] if judges else []

    # 合并模型列表：推理模型 + 评委模型
    all_models = inference_models_list + judge_models

    # 构建 templates 数组
    templates = []
    if inference_template:
        templates.append(inference_template)
    templates.append({
        "name": "模型评测",
        "type": "evaluation",
        "parameters": {"evalset": evalset_id, "eval": dimensions.get("evals")}
    })

    payload = {
        "apiVersion": "v1",
        "models": all_models,
        "agents": [],
        "spec": {
            "templates": templates
        }
    }
```

---

### Step 4: 验证脚本语法正确

```bash
python -m py_compile .claude/skills/model-evaluation/scripts/eval_task.py
```

Expected: 无输出（语法正确）

---

### Step 5: Commit

```bash
git add .claude/skills/model-evaluation/scripts/eval_task.py
git commit -m "feat(eval_task): 新增 --inference_models 参数支持推理配置构建"

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

---

## Task 2: 更新 Skill 文档添加推理配置检测步骤

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-execute.md`

### Step 1: 在任务1添加推理配置检测步骤

在任务1"提交评测任务"的"执行流程"描述后（约第 47-59 行之间），新增判断步骤：

**在第 59 行（提交命令之前）插入**：

```
**判断**：`{work-dir}/.eval/{session-id}/selected-models.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 存在 | 评测集无答案场景，添加 `--inference_models` 参数 |
| 不存在 | 评测集有答案场景，使用原命令 |

---

**提交命令**：

**有推理配置**（评测集无答案场景）：
```bash
{python-env}{python-cmd} "{skill-dir}/scripts/eval_task.py" submit \
  --config "{skill-dir}/scripts/cfg/eval-server.cfg" \
  --auth "{work-dir}/.eval/auth.json" \
  --eval_set "{work-dir}/.eval/{session-id}/evalset/evalset-meta.json" \
  --eval_dimension "{work-dir}/.eval/{session-id}/eval-dimension.json" \
  --eval_judge "{work-dir}/.eval/{session-id}/eval-judge.json" \
  --inference_models "{work-dir}/.eval/{session-id}/selected-models.json" \
  --output "{work-dir}/.eval/{session-id}/evaltask/evaltask-meta.json"
```

**无推理配置**（评测集有答案场景）：
```bash
{python-env}{python-cmd} "{skill-dir}/scripts/eval_task.py" submit \
  --config "{skill-dir}/scripts/cfg/eval-server.cfg" \
  --auth "{work-dir}/.eval/auth.json" \
  --eval_set "{work-dir}/.eval/{session-id}/evalset/evalset-meta.json" \
  --eval_dimension "{work-dir}/.eval/{session-id}/eval-dimension.json" \
  --eval_judge "{work-dir}/.eval/{session-id}/eval-judge.json" \
  --output "{work-dir}/.eval/{session-id}/evaltask/evaltask-meta.json"
```
```

---

### Step 2: 删除原有的提交命令（避免重复）

删除原有的提交命令块（第 51-59 行），因为已在 Step 1 中按场景拆分展示。

---

### Step 3: Commit

```bash
git add .claude/skills/model-evaluation/eval-execute.md
git commit -m "docs(eval-execute): 新增推理配置检测步骤和场景化提交命令"

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

---

## Task 3: 更新脚本定义文档

**Files:**
- Modify: `.claude/skills/model-evaluation/references/脚本定义.md`

### Step 1: 补充 submit 子命令参数说明

在第 319-328 行（submit 子命令说明区域）添加新参数：

**在第 327 行后添加**：

```
| `--inference_models` | `string` | 否 | 推理模型列表文件路径（selected-models.json）。不传则跳过推理配置，适用于评测集已包含答案的场景。 |
```

---

### Step 2: 补充参数说明表格

在"参数说明"表格（约第 354-367 行）添加：

```
| `--inference_models` | 推理模型列表文件。传入时自动构建 inference 模板并合并推理模型到顶层 models 字段。 |
```

---

### Step 3: Commit

```bash
git add .claude/skills/model-evaluation/references/脚本定义.md
git commit -m "docs(脚本定义): 补充 --inference_models 参数说明"

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

---

## Task 4: 更新评测服务接口说明文档

**Files:**
- Modify: `.claude/skills/model-evaluation/references/评测服务接口说明.md`

### Step 1: 补充 TaskTemplate.type 枚举

在第 286-294 行（TaskTemplate.type 枚举表格）添加 `inference` 类型：

**修改表格**：

```
**TaskTemplate.type 枚举**：

| 类型 | 必填 | 说明 |
|------|------|------|
| `evalset` | 否 | 评测集准备步骤 |
| `inference` | 否 | **模型推理步骤（评测集无答案时使用）** |
| `evaluation` | 是 | 模型评测步骤（必需） |
| `report-upload` | 否 | 报告上传步骤 |
| `database` | 否 | 数据入库步骤 |
```

---

### Step 2: 添加 inference 类型 parameters 定义

在 TaskTemplate.type 枚举表格后（约第 294 行后）添加：

```
**inference 类型 parameters 定义**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `evalset` | `string` | 是 | 评测集标识（填充实际 evalset_id） |
| `models` | `array[InferenceModel]` | 是 | 推理模型列表 |

**InferenceModel 定义**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 模型服务标识 |

**配置示例**：

```json
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
}
```
```

---

### Step 3: Commit

```bash
git add .claude/skills/model-evaluation/references/评测服务接口说明.md
git commit -m "docs(评测服务接口): 补充 inference 类型模板定义"

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

---

## Task 5: 更新中间产物说明文档

**Files:**
- Modify: `.claude/skills/model-evaluation/references/中间产物说明.md`

### Step 1: 补充 selected-models.json 用途说明

在 selected-models.json 说明区域（约第 248-288 行）补充用途：

**在字段说明表格后添加**：

```
**用途扩展**：

| 场景 | 用途 |
|------|------|
| 构建阶段 | `eval_set.py expand` 子命令展开评测集 |
| 执行阶段 | `eval_task.py submit` 子命令构建推理配置（新增） |
```

---

### Step 2: 补充数据流说明

在文件末尾（约第 469 行后）添加推理配置数据流：

```
---

## 4. 推理配置数据流（新增）

**场景**：评测集无答案（answer 字段为空）

**数据流**：

```
构建阶段产物：
  selected-models.json → 推理模型信息
  evalset-meta.json → evalset_id (dataset 字段)
  eval-judge.json → 评委模型信息

执行阶段（任务提交）：
  检测 selected-models.json 存在
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

**请求体示例**：

```json
{
  "apiVersion": "v1",
  "models": [
    {"id": "2402509314320990", "name": "DeepSeek Chat", "model": "deepseek-chat"},
    {"id": "ID_AJ001", "name": "DeepSeek-R1", "type": "api-openai", "model": "xdeepseekr1"}
  ],
  "spec": {
    "templates": [
      {"name": "模型推理", "type": "inference", "parameters": {"evalset": "eval-xxx", "models": [{"id": "2402509314320990"}]}},
      {"name": "模型评测", "type": "evaluation", "parameters": {...}}
    ]
  }
}
```
```

---

### Step 3: Commit

```bash
git add .claude/skills/model-evaluation/references/中间产物说明.md
git commit -m "docs(中间产物): 补充推理配置数据流说明"

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

---

## Self-Review

**Spec Coverage:**
- ✅ 脚本新增参数 → Task 1
- ✅ Skill 文档检测逻辑 → Task 2
- ✅ 脚本定义补充 → Task 3
- ✅ API 文档补充 → Task 4
- ✅ 中间产物补充 → Task 5

**Placeholder Scan:**
- ✅ 无 TBD/TODO
- ✅ 所有代码步骤包含完整实现
- ✅ 所有命令包含完整参数

**Type Consistency:**
- ✅ `selected-models.json` 格式与中间产物说明一致
- ✅ `inference_template` 结构与 API 文档定义一致
- ✅ `models` 字段合并逻辑与设计一致