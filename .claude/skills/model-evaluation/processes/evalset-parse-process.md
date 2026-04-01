---
name: evalset-parse-process
description: Use when eval-build reaches evalset parsing step, or user directly requests parsing an evaluation dataset
---

# 评测集解析流程

你正在运行评测集解析流水线。**按顺序执行每个步骤，不得跳过。**

## 禁止规则

- 在 check-status 完成前，不得凭直觉选择分支
- 字段映射确认缺失时，不得保存配置
- 用户未回复时，不得自动继续

**违反字面意思 = 违反规则精神。**

---

## 目标

解析评测集 → 判断类型 → 执行对应处理 → 返回标准化阶段

核心原则：**步骤通用，分支条件执行**。

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

获取方式：

```bash
# 复制用户文件
cp {用户路径} {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext}

# 或远程下载
curl -o {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} {下载链接}
```

> `{ext}` 从文件路径提取（如 `data.xlsx` → `ext=xlsx`）

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

**输出字段**：`file`, `format`, `total_rows`, `fields`, `answer`

---

## 步骤3：生成初始映射

**判断**：`evalset-fields-mapping.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤4 |
| 不存在 | 生成映射 |

读取结构文件 → 匹配字段 → 生成映射配置。

### 字段匹配规则

| 标准字段 | 关键词 | 匹配规则 |
|----------|--------|----------|
| question | question, prompt, input, query, 问题, 提问 | 精确优先，包含次之 |
| answer | answer, response, output, reply, 回答, 回复 | 精确优先，包含次之 |
| model | model, model_name, model_id, llm, llm_name, 模型, 模型名称 | 精确优先，包含次之 |
| case_id | case_id, caseid, 用例id | **仅精确匹配** |
| reference | reference, ref, gold, 参考答案, 标准答案 | 精确优先，包含次之 |
| keypoint | keypoint, keypoints, 关键点, 评测点 | 精确优先，包含次之 |

> 其他字段（system, context, category）见 references/字段映射详表.md

**必填字段**：question、answer、model、case_id

**映射格式**：

```json
{
  "question": {"source_field": "问题", "default": null},
  "answer": {"source_field": "回答", "default": null},
  "model": {"source_field": "模型名称", "default": null},
  "case_id": {"source_field": "id", "default": null}
}
```

---

## 步骤4：判断后续走向 ⚠️

**必须执行 check-status**，不得凭直觉判断。

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py check-status \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-status.json
```

**判断逻辑**：读取 `evalset-status.json` 中 `answer.status`

| status | 后续动作 |
|--------|----------|
| `all_empty` | answer 字段不存在或全空 → 分支A |
| `all_filled` | answer 字段全部非空 → 分支B |
| `partial` | answer 字段部分为空 → 询问用户 |

**partial 处理**：

询问用户：
> 评测集中部分 answer 为空，请选择处理方式：
> 1. 视为"只有问题"（空 answer 的记录将云端推理）
> 2. 视为"问题+答案"（跳过空 answer 的记录）

---

## 分支A：模型选择（answer 为空）

### 步骤A1：获取可用模型列表

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py list-models \
  --auth {work-dir}/.eval/auth.json \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --output {work-dir}/.eval/{session-id}/evalset/available-models.json
```

**失败处理**：Token 失效 → 引导重新授权

### 步骤A2：用户选择模型

展示模型列表，等待用户选择（单模型输入序号，多模型用逗号分隔如 `1,2`）。

### 步骤A3：保存选择

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

完成后 → 返回标准化阶段

---

## 分支B：字段校验与确认（answer 非空）

> **注意**：如果评测集由 AI 助手生成（evalset-create-process），跳过步骤B3映射确认。

### 步骤B1：检查 model 字段状态

| 状态 | 后续 |
|------|------|
| 不存在或全空 | → 步骤B4（询问评测模式） |
| 已填充 | → 步骤B2 |

### 步骤B2：检查 case_id 字段状态

| 状态 | 处理 |
|------|------|
| 已填充 | 标准化时使用原值 |
| 不存在或全空 | 标准化时按 question 分组自动生成 |

> 多模型横评时，同一问题的不同模型回答共享相同 case_id

### 步骤B3：确认映射配置 ⚠️

**映射目的说明**：将评测集的原始字段映射为标准字段，标准化后便于后续统一处理（标准化转换、评测执行、评分判定等环节均基于标准字段工作）。

展示映射表（含标准字段含义），询问："以上映射是否正确？(Y/n)"

**此步骤必须等待用户确认，不可跳过。**

| 用户选择 | 后续 |
|------|------|
| Y | → 步骤B4/保存 |
| n | 调整映射，重新确认 |

### 步骤B4：处理 model 字段为空

**条件**：步骤B1判断 model 字段为空时执行。

询问评测模式：
> 请确认评测模式：
> 1. 单模型评测 - 评测单个模型的表现
> 2. 多模型横评 - 多个模型横向对比（需在评测集补充 model 字段）

| 选择 | 处理 |
|------|------|
| 1 | 设置 `model.default` 为用户指定的模型名称 |
| 2 | 提示用户补充 model 字段，重新执行步骤1 |

### 步骤B5：保存

保存至 `evalset-fields-mapping.json`。

完成后 → 返回标准化阶段

---

## Red Flags - STOP

以下想法意味着返回上一步重新执行：

- "映射看起来正确，无需确认"
- "用户说直接继续"
- 未执行 check-status 就选择分支
- "这个情况特殊，可以跳过"
- 用户未回复时自动选择 Y

---

## 常见错误

| 错误 | 解决方案 |
|------|----------|
| 文件格式不支持 | 转换为 JSONL/JSON/CSV/Excel |
| 缺少 question 字段 | 确保文件包含问题字段 |
| 字段映射不明确 | 展示示例数据请求确认 |
| 结构解析报错 | 检查编码，转为 UTF-8 |
| 模型列表获取失败 | 引导重新授权或检查网络 |
| 部分 answer 为空 | 询问用户选择处理方式 |
| model 字段为空但选择多模型横评 | 提示用户补充 model 字段 |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |
| `{ext}` | 文件扩展名 |
| `{python-env}` | Python环境变量前缀（Windows GBK 为 `PYTHONUTF8=1 `） |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |