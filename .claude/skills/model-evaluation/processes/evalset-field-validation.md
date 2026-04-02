---
name: evalset-field-validation
description: Use when evalset-parse determines answer field is filled and needs field validation
---

# 字段校验流程

**调用方**：evalset-parse.md 步骤4（分支B）

**触发条件**：answer 字段全部非空

**返回点**：标准化阶段

---

## 目标

校验字段状态，处理缺失字段，保存配置。

---

## 步骤B1：检查 model 字段状态

| 状态 | 后续 |
|------|------|
| 不存在或全空 | → 步骤B3（询问评测模式） |
| 已填充 | → 步骤B2 |

---

## 步骤B2：检查 case_id 字段状态

| 状态 | 处理 |
|------|------|
| 已填充 | 标准化时使用原值 |
| 不存在或全空 | 标准化时按 question 分组自动生成 |

> 多模型横评时，同一问题的不同模型回答共享相同 case_id

---

## 步骤B3：处理 model 字段为空

**条件**：步骤B1判断 model 字段为空时执行。

询问评测模式：
> 请确认评测模式：
> 1. 单模型评测 - 评测单个模型的表现
> 2. 多模型横评 - 多个模型横向对比（需在评测集补充 model 字段）

| 选择 | 处理 |
|------|------|
| 1 | 设置 `model.default` 为用户指定的模型名称 |
| 2 | 提示用户补充 model 字段，重新执行步骤1 |

---

## 步骤B4：保存

保存至 `evalset-fields-mapping.json`。

---

## 常见错误

| 错误 | 解决方案 |
|------|----------|
| model 字段为空但选择多模型横评 | 提示用户补充 model 字段 |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |

---

**返回**：标准化阶段