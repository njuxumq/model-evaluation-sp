---
name: evalset-field-mapping
description: Use when evalset-parse reaches step 3 and needs to generate field mapping configuration
---

# 字段映射流程

**调用方**：evalset-parse.md 步骤3

**返回点**：evalset-parse.md 步骤4

---

## 目标

生成字段映射配置，等待用户确认。

---

## 步骤1：生成映射

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

## 步骤2：确认映射配置 ⚠️

**映射目的说明**：将评测集的原始字段映射为标准字段，标准化后便于后续统一处理（标准化转换、评测执行、评分判定等环节均基于标准字段工作）。

展示映射表（含标准字段含义），询问："以上映射是否正确？(Y/n)"

**此步骤必须等待用户确认，不可跳过。**

| 用户选择 | 后续 |
|------|------|
| Y | 保存映射 → 返回主流程 |
| n | 调整映射，重新确认 |

> **注意**：如果评测集由 AI 助手生成（evalset-create），跳过此确认步骤。

---

## Red Flags

- 确认映射缺失时，不得保存配置
- 用户未回复时，不得自动继续
- "映射看起来正确，无需确认"

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |

---

**返回**：evalset-parse.md 步骤4