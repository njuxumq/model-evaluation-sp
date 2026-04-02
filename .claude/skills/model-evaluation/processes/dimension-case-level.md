---
name: dimension-case-level
description: Use when user selects case-level evaluation and expert template matching failed
---

# 定制用例级评测配置

**触发条件**：eval-build.md 任务2步骤2确定评测方式为定制用例级。

**目标**：基于内置用例级评测维度模板，输出维度配置。

> ⚠️ 注意：定制用例级评测只有一个评测维度。

---

## 步骤1：查阅内置模板

参考 [内置模板说明.md](../references/内置模板说明.md) 第2.2节"定制用例级评测模板"。

筛选合适的模板文件并读取。

---

## 步骤2：评估模板适配情况

| 情况 | 说明 | 动作 |
|------|------|------|
| 完全适配 | 内置模板满足评测场景 | 直接使用内置模板 |
| 部分适配 | 需调整部分字段内容 | 基于模板调整字段 |

**可调整字段**：
- `name`：维度名称
- `description`：维度描述
- `prompt.role`：评委角色设定
- `prompt.definition`：有效性定义
- `prompt.instruct`：通过/不通过判定标准

**保持不变的字段**：
- `type`：固定为 `llm-judge`
- `prompt.body`：固定依赖评测要点（keypoint）、上下文（context）、历史（history）

---

## 步骤3：保存维度配置

参考 [维度配置转化规则.md](../references/维度配置转化规则.md) 进行结构转化。

**保存路径**：`{work-dir}/.eval/{session-id}/eval-dimension.json`

---

**返回**：任务2步骤4