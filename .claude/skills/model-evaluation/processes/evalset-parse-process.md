---
name: evalset-parse-process
description: Use when user has an existing evaluation dataset file and needs to parse field mapping
---

# 评测集解析流程

## 目标

获取评测集文件 → 解析字段结构 → 生成映射配置 → 用户确认，完成后返回标准化转换流程。

---

## 何时使用

- 用户已有完整评测集文件（JSONL/JSON/CSV/XLSX）
- 需要解析字段映射关系
- 接收生成流程或补充流程输出的评测集

---

## 步骤1：获取评测集文件

**判断**：`evalset-prepared.{ext}` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤2 |
| 不存在 | 执行如下命令，获取用户文件 |

```bash
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

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py analysis \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-structure.json
```

输出包含：`file`、`format`、`total_rows`、`fields`（字段名+类型）。

---

## 步骤3：生成字段映射

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

## 步骤4：确认映射配置

> 注意，如果先前评测集全部由AI助手生成，则跳过此步骤。因为评测集都是按标准格式生成，无需用户确认映射配置。

**判断**：用户是否已确认映射？

| 状态 | 动作 |
|------|------|
| 已确认 | → 保存并返回 |
| 未确认 | 执行确认流程 |

### 4.1 字段映射确认

向用户说明映射目的：将评测集的原始字段映射为标准字段，标准化后便于后续统一处理（标准化转换、评测执行、评分判定等环节均基于标准字段工作）。

展示映射表（含标准字段含义），等待用户确认。

> **此步骤必须等待用户确认，不可跳过。**

| 用户选择 | 后续动作 |
|----------|----------|
| Y | → 4.2 |
| n | 调整映射，重新确认 |

### 4.2 模型字段确认

| model.source_field | 处理 |
|---------------------|------|
| 有值 | 直接使用源字段值，无需询问 |
| null | 判断评测场景（见下表） |

**model 字段为空时的处理**：

| 评测场景 | 处理方式 | 后续处理 |
|----------|----------|----------|
| 多模型横评 | 结合标准字段列表，提示用户在评测集补充模型字段 | 待用户补充后，重新解析结构 |
| 单模型评测 | 询问用户模型名称 | 将模型名称写入映射文件的 `default` 字段 |
| 单模型评测 | 使用 `model_a` 作为默认值 | 将 `model_a` 写入映射文件的 `default` 字段 |

### 4.3 case_id 字段确认

| case_id.source_field | 处理 |
|-----------------------|------|
| 有值 | 直接使用源数据值 |
| null | 标准化时根据 question 分组自动生成 |

> 多模型横评时，同一问题的不同模型回答共享相同 case_id。

### 4.4 保存

保存至 `{work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json`，返回 **eval-build.md 任务4步骤4**（标准化转换）。

---

## 常见错误

| 错误 | 解决方案 |
|------|----------|
| 字段匹配失败 | 手动指定映射 |
| 文件格式不支持 | 转换为JSONL/JSON/CSV/XLSX |
| 结构解析报错 | 检查编码，转为UTF-8 |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名（`session-{8位字母数字}`） |
| `{skill-dir}` | 技能安装目录 |
| `{ext}` | 文件扩展名 |
| `{python-env}` | Python环境变量前缀 |
| `{python-cmd}` | Python命令 |