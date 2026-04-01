# 评测集处理流程重构实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构评测集处理流程，支持"只有问题"和"问题+答案"两种评测集类型。

**Architecture:** 采用"步骤+分支"结构，通用步骤（解析、映射）始终执行，条件分支（模型选择/字段校验）根据 answer 字段状态执行。

**Tech Stack:** Python 3.x, argparse, requests

---

## 文件结构

| 文件 | 修改类型 | 职责 |
|------|----------|------|
| `scripts/clients/api_client.py` | 新增方法 | `get_models()` 调用模型列表接口 |
| `scripts/eval_set.py` | 修改 + 新增 | `analysis` 输出调整 + `list-models`/`expand` 子命令 |
| `processes/evalset-parse-process.md` | 重写 | 三步骤 + 两分支结构 |
| `eval-build.md` | 修改 | 任务4步骤3标准化命令选择 |
| `references/中间产物说明.md` | 修改 | 新增产物说明 |
| `references/评测服务接口说明.md` | 修改 | answer 可选 + 模型列表接口 |

---

## Task 1: 新增 API 客户端 `get_models()` 方法

**Files:**
- Modify: `.claude/skills/model-evaluation/scripts/clients/api_client.py`

- [ ] **Step 1: 添加 `get_models()` 方法**

在 `ApiClient` 类中新增方法：

```python
def get_models(self) -> list:
    """获取可用推理模型列表

    Returns:
        模型名称列表，如 ["deepseek-r1", "gpt-4", "claude-3"]
    """
    response = self.get("/open/api/v1/models")
    return response.get("models", [])
```

插入位置：在 `_handle_response` 方法之后。

- [ ] **Step 2: 验证语法正确**

运行：`python -m py_compile .claude/skills/model-evaluation/scripts/clients/api_client.py`
预期：无输出（语法正确）

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/model-evaluation/scripts/clients/api_client.py
git commit -m "feat(api): add get_models() method for fetching available models

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 2: 修改 `analysis` 子命令输出

**Files:**
- Modify: `.claude/skills/model-evaluation/scripts/eval_set.py`

- [ ] **Step 1: 新增 `analyze_answer_field()` 函数**

在 `cmd_analysis` 函数之前插入：

```python
def analyze_answer_field(data: list, fields: dict) -> dict:
    """分析 answer 字段状态

    Args:
        data: 评测集数据列表
        fields: 字段信息字典

    Returns:
        {"exists": bool, "status": "all_empty"|"partial"|"all_filled"}
    """
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

- [ ] **Step 2: 修改 `cmd_analysis()` 函数**

修改 `cmd_analysis` 函数，在 `save_json` 之前添加 answer 分析：

```python
def cmd_analysis(args):
    """解析评测集文件结构，输出结构文件"""
    load_result = load_data(args.input)
    if not load_result.get("success"):
        raise ValueError(f"数据加载失败: {load_result.get('message')}")
    data = load_result.get("data", {}).get("items", [])

    fields = extract_fields(data)

    # 新增：分析 answer 字段状态
    answer_info = analyze_answer_field(data, fields)

    # 结构文件：唯一产物
    structure = {
        "file": args.input,
        "format": Path(args.input).suffix.lower()[1:],
        "total_rows": len(data),
        "fields": fields,
        "answer": answer_info  # 新增字段
    }
    save_json(args.output, structure)

    return {
        "success": True,
        "total_rows": len(data),
        "fields": list(fields.keys()),
        "structure_file": args.output
    }
```

- [ ] **Step 3: 验证语法正确**

运行：`python -m py_compile .claude/skills/model-evaluation/scripts/eval_set.py`
预期：无输出（语法正确）

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/model-evaluation/scripts/eval_set.py
git commit -m "feat(eval_set): add answer field analysis to analysis command

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 3: 新增 `list-models` 子命令

**Files:**
- Modify: `.claude/skills/model-evaluation/scripts/eval_set.py`

- [ ] **Step 1: 新增 `cmd_list_models()` 函数**

在 `cmd_submit` 函数之后插入：

```python
def cmd_list_models(args):
    """获取可用推理模型列表"""
    config_result = load_config_kv(args.config)
    if not config_result.get("success"):
        raise ValueError(f"配置文件加载失败: {config_result.get('message')}")
    config = config_result.get("data", {})

    # 使用 TokenManager 和 ApiClient
    token_manager = TokenManager(args.auth)
    client = ApiClient(token_manager, config.get('base_url', 'http://127.0.0.1:8080'))

    models = client.get_models()

    save_json(args.output, {"models": models})
    return {"models": models, "output": args.output}
```

- [ ] **Step 2: 注册子命令**

在 `main()` 函数的 subparsers 定义区域，在 `submit` 子命令之后添加：

```python
    # list-models
    p = subparsers.add_parser('list-models', help='获取可用推理模型列表')
    p.add_argument('--auth', required=True, help='鉴权信息文件')
    p.add_argument('--config', required=True, help='服务配置文件')
    p.add_argument('--output', required=True, help='输出文件路径')
    p.set_defaults(func=cmd_list_models)
```

- [ ] **Step 3: 验证语法正确**

运行：`python -m py_compile .claude/skills/model-evaluation/scripts/eval_set.py`
预期：无输出（语法正确）

- [ ] **Step 4: 验证 --help 输出**

运行：`python .claude/skills/model-evaluation/scripts/eval_set.py --help`
预期：显示包含 `list-models` 的子命令列表

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/model-evaluation/scripts/eval_set.py
git commit -m "feat(eval_set): add list-models subcommand

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 4: 新增 `expand` 子命令

**Files:**
- Modify: `.claude/skills/model-evaluation/scripts/eval_set.py`

- [ ] **Step 1: 新增辅助函数 `extract_field_value()`**

在 `analyze_answer_field` 函数之后插入：

```python
def extract_field_value(item: dict, mapping: dict, field_name: str) -> str:
    """根据映射配置提取字段值

    Args:
        item: 原始数据项
        mapping: 字段映射配置
        field_name: 标准字段名

    Returns:
        字段值（字符串）
    """
    config = mapping.get(field_name, {})
    if isinstance(config, str):
        # 兼容旧格式
        src_field = config
        default_val = None
    else:
        src_field = config.get('source_field')
        default_val = config.get('default')

    if src_field and src_field in item:
        value = item.get(src_field)
        if value is not None and not (isinstance(value, float) and math.isnan(value)):
            return str(value).strip()

    if default_val:
        return str(default_val)

    return ""
```

- [ ] **Step 2: 新增 `expand_data()` 函数**

在 `extract_field_value` 函数之后插入：

```python
def expand_data(data: list, mapping: dict, models: list) -> list:
    """展开评测集：N问题 × M模型 = N×M条记录

    Args:
        data: 原始评测集数据
        mapping: 字段映射配置
        models: 用户选择的模型列表

    Returns:
        展开后的标准化评测集
    """
    result = []
    case_counter = 0
    question_to_case = {}

    for item in data:
        question = extract_field_value(item, mapping, 'question')
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
                "answer": "",  # 空字符串，由推理服务填充
                "model": model,
                "case_id": case_id
            }
            # 添加可选字段
            for field in OPTIONAL_FIELDS:
                value = extract_field_value(item, mapping, field)
                if value:
                    record[field] = value
            result.append(record)

    return result
```

- [ ] **Step 3: 新增 `cmd_expand()` 函数**

在 `cmd_list_models` 函数之后插入：

```python
def cmd_expand(args):
    """展开评测集（answer 为空场景）"""
    # 加载原始数据
    load_result = load_data(args.input)
    if not load_result.get("success"):
        raise ValueError(f"数据加载失败: {load_result.get('message')}")
    data = load_result.get("data", {}).get("items", [])
    if not data:
        raise ValueError("评测集为空或无法解析")

    # 加载映射配置
    mapping_result = load_json(args.mapping)
    if not mapping_result.get("success"):
        raise ValueError(f"映射文件加载失败: {mapping_result.get('message')}")
    mapping = mapping_result.get("data", {})

    # 加载用户选择的模型列表
    models_result = load_json(args.models)
    if not models_result.get("success"):
        raise ValueError(f"模型列表文件加载失败: {models_result.get('message')}")
    models_data = models_result.get("data", {})
    models = models_data.get("models", [])
    if not models:
        raise ValueError("模型列表为空")

    # 展开数据
    expanded = expand_data(data, mapping, models)
    if not expanded:
        raise ValueError("展开后的评测集为空")

    # 输出 JSONL
    Path(args.output).parent.mkdir(parents=True, exist_ok=True)
    Path(args.output).write_text(
        '\n'.join(json.dumps(item, ensure_ascii=False) for item in expanded),
        encoding='utf-8'
    )

    return {
        "success": True,
        "input_rows": len(data),
        "output_rows": len(expanded),
        "models": models,
        "output_file": args.output
    }
```

- [ ] **Step 4: 注册子命令**

在 `main()` 函数的 subparsers 定义区域，在 `list-models` 子命令之后添加：

```python
    # expand
    p = subparsers.add_parser('expand', help='展开评测集（answer为空场景）')
    p.add_argument('--input', required=True, help='原始评测集文件路径')
    p.add_argument('--mapping', required=True, help='字段映射文件路径')
    p.add_argument('--models', required=True, help='用户选择的模型列表文件')
    p.add_argument('--output', required=True, help='输出文件路径')
    p.set_defaults(func=cmd_expand)
```

- [ ] **Step 5: 验证语法正确**

运行：`python -m py_compile .claude/skills/model-evaluation/scripts/eval_set.py`
预期：无输出（语法正确）

- [ ] **Step 6: 验证 --help 输出**

运行：`python .claude/skills/model-evaluation/scripts/eval_set.py --help`
预期：显示包含 `expand` 的子命令列表

- [ ] **Step 7: Commit**

```bash
git add .claude/skills/model-evaluation/scripts/eval_set.py
git commit -m "feat(eval_set): add expand subcommand for question-only dataset

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 5: 重写 `evalset-parse-process.md`

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-parse-process.md`

- [ ] **Step 1: 备份原文件**

```bash
cp .claude/skills/model-evaluation/processes/evalset-parse-process.md .claude/skills/model-evaluation/processes/evalset-parse-process.md.bak
```

- [ ] **Step 2: 重写文档内容**

将文件内容替换为设计文档中定义的新结构（步骤+分支）。保留变量速查部分。

参见设计文档 `docs/superpowers/specs/2026-04-01-evalset-process-refactor-design.md` 第3节"processes/evalset-parse-process.md 重写"的完整内容。

- [ ] **Step 3: 验证文档格式**

检查 Markdown 格式正确，无语法错误。

- [ ] **Step 4: 删除备份文件**

```bash
rm .claude/skills/model-evaluation/processes/evalset-parse-process.md.bak
```

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/model-evaluation/processes/evalset-parse-process.md
git commit -m "refactor(evalset-parse): rewrite with step+branch structure

- Add step 1-4 for parse and mapping
- Add branch A for model selection (answer empty)
- Add branch B for field validation (answer filled)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 6: 修改 `eval-build.md` 任务4

**Files:**
- Modify: `.claude/skills/model-evaluation/eval-build.md`

- [ ] **Step 1: 修改任务4步骤3**

找到任务4步骤3"标准化转换"部分，替换为：

```markdown
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
```

- [ ] **Step 2: 验证文档格式**

检查 Markdown 格式正确，表格和代码块无语法错误。

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/model-evaluation/eval-build.md
git commit -m "refactor(eval-build): update task4 step3 for expand command

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 7: 更新 `中间产物说明.md`

**Files:**
- Modify: `.claude/skills/model-evaluation/references/中间产物说明.md`

- [ ] **Step 1: 新增 `selected-models.json` 说明**

在 `evalset-meta.json` 之后添加：

```markdown
### selected-models.json

**路径**：`{work-dir}/.eval/{session-id}/evalset/selected-models.json`

**说明**：用户选择的推理模型列表，用于"只有问题"场景的标准化展开。

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

### available-models.json

**路径**：`{work-dir}/.eval/{session-id}/evalset/available-models.json`

**说明**：从评测服务获取的可用推理模型列表。

| 字段 | 类型 | 说明 |
|------|------|------|
| `models` | `array[string]` | 可用推理模型名称列表 |
```

- [ ] **Step 2: 更新 `evalset-structure.json` 说明**

在现有 `evalset-structure.json` 说明中新增 `answer` 字段：

```markdown
**示例**：

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

**answer 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `answer.exists` | `bool` | answer 字段是否存在 |
| `answer.status` | `string` | `all_empty`: 全空 / `partial`: 部分空 / `all_filled`: 全非空 |
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/model-evaluation/references/中间产物说明.md
git commit -m "docs(intermediate): add selected-models and update structure

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 8: 更新 `评测服务接口说明.md`

**Files:**
- Modify: `.claude/skills/model-evaluation/references/评测服务接口说明.md`

- [ ] **Step 1: 修改 `answer` 字段说明**

在 2.1 上传评测集 - EvalsetItem 定义中，修改 `answer` 字段：

找到：
```markdown
| `answer` | `string` | 是 | 模型回答 |
```

改为：
```markdown
| `answer` | `string` | 否 | 模型回答（为空时由推理服务生成） |
```

- [ ] **Step 2: 新增模型列表接口**

在快速参考表格中添加新行：

```markdown
| 获取模型列表 | GET | `/open/api/v1/models` | 获取可用推理模型列表 |
```

在分页响应格式之后添加新章节：

```markdown
### 2.4 获取模型列表

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

| 字段 | 类型 | 说明 |
|------|------|------|
| `data.models` | `array[string]` | 可用推理模型名称列表 |
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/model-evaluation/references/评测服务接口说明.md
git commit -m "docs(api): make answer optional and add models endpoint

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 9: 最终验证与提交

- [ ] **Step 1: 运行 Python 语法检查**

```bash
python -m py_compile .claude/skills/model-evaluation/scripts/eval_set.py
python -m py_compile .claude/skills/model-evaluation/scripts/clients/api_client.py
```

预期：无输出（语法正确）

- [ ] **Step 2: 检查所有修改文件**

```bash
git status
```

预期：所有修改已提交

- [ ] **Step 3: 查看提交历史**

```bash
git log --oneline
```

预期：显示所有功能提交

- [ ] **Step 4: 创建总结提交（如有遗漏文件）**

```bash
git add -A
git commit -m "feat(evalset): complete evalset process refactor

Support two dataset types:
- Questions only (with model selection and expand)
- Questions + answers (existing normalize flow)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## 验证清单

| 验证项 | 验证方法 |
|--------|----------|
| `analysis` 输出正确 | 测试不同 answer 状态的评测集 |
| `list-models` 正常工作 | 测试接口调用和 Token 失效处理 |
| `expand` 展开逻辑正确 | 验证 N×M 条记录、case_id 共享 |
| 流程文档清晰 | 验证步骤+分支结构易于理解 |
| 向后兼容 | 验证现有"问题+答案"场景正常工作 |