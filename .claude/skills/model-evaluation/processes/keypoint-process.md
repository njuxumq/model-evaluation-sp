---
name: keypoint-process
description: Use when case-level evaluation requires generating keypoints for evaluation items
---

# 评测点生成流程

**触发条件**：定制用例级评测且标准化评测集无 keypoint 字段。

**目标**：根据问题、参考答案、上下文字段自动生成 1-3 个评测点，覆盖原 evalset-standard.jsonl 文件。

---

## 前置判断：多模型横评场景

若 `case_id` 存在重复值（多模型横评），按唯一 `case_id` 去重后生成评测点，保存时复制到同 `case_id` 的所有记录。

**示例**：20 条评测集（10 问题 × 2 模型）→ 生成 10 个评测点

---

## 步骤1：场景判断

根据标准化评测集字段组合识别场景类型。

**字段说明**

| 字段 | 字母代表 | 说明 | 用途 |
|------|------|------|------|
| `question` | Q | 用户问题 | 评测的核心输入 |
| `answer` | - |模型输出的回答 | 待评测内容，**不是参考答案** |
| `reference` | R | 参考答案/理想答案 | 用于提炼评测要点 |
| `context` | C | 上下文信息 | 用于理解问题背景 |

**重要区分**：
- `answer` 是待评测的模型输出，不属于场景判断的依据
- `reference` 才是参考答案，用于场景判断和评测点生成

| 场景 | 包含字段 | 评估点来源 | 典型用途 |
|------|----------|------------|----------|
| Q | 仅 `question` | 基于问题推断 | 通用数据集（质量一般） |
| QR | `question` + `reference` | 从参考答案提炼关键信息点 | 通用数据集（推荐） |
| QC | `question` + `context` | 从上下文推导应提取的要点 | 私域数据集 |
| QRC | `question` + `reference` + `context` | 最佳效果 | 私域数据集（推荐） |

**判断逻辑**（忽略 `answer` 字段）：
```
if reference AND context: scene = "QRC"
elif reference: scene = "QR"
elif context: scene = "QC"
else: scene = "Q"
```

---

## 步骤2：生成评测点

调用 `{skill-dir}/scripts/utils/keypoint_prompts.py` 生成提示词：

- **系统提示词**：使用 `SYSTEM_PROMPT`（包含四步思考法和7条质量标准）
- **用户提示词**：使用 `build_user_prompt(question, answer, context)` 构建

### 执行方式：

1. **准备提示词**：
   - 读取 `keypoint_prompts.py` 中的 `SYSTEM_PROMPT` 作为系统提示词
   - 使用 `build_user_prompt(question, reference, context)` 构建用户提示词

2. **调用内置模型**：
   - 使用 Agent 工具（subagent_type=general-purpose）批量生成评测点
   - 每条数据独立生成，确保评测点质量

3. **执行示例**：
```python
# 构建提示词
from keypoint_prompts import SYSTEM_PROMPT, build_user_prompt

user_prompt = build_user_prompt(question, reference, context)

# 通过 Agent 工具调用内置模型生成
# Agent prompt 示例：
# f"{SYSTEM_PROMPT}\n\n{user_prompt}"
```

4. **批量处理建议**：
   - 数据量 ≤ 50 条：主 Agent 直接生成
   - 数据量 > 50 条：使用 Agent 工具分批执行
   - 主 Agent context 满载时：必须使用 SubAgent 分批生成

**⚠️ 禁止行为**：
- ❌ 不调用模型直接"编造"评测点
- ❌ 跳过 SYSTEM_PROMPT 中的思考步骤
- ❌ 忽略质量标准要求

### 输出格式：
```json
["是否提及xxx", "是否包含xxx"]
```

### 失败处理
- 评测点生成失败时重试一次
- 仍失败则跳过该条数据，继续处理下一条
- 记录跳过的数据条目，最后汇总报告

---

## 步骤3：预览与确认

展示**前3条**数据的评测点预览：

| 序号 | 问题 | 生成的评测点 |
|------|------|--------------|
| 1 | 什么是机器学习？ | ["是否定义了机器学习", "是否提及与人工智能的关系"] |
| 2 | 如何评估模型性能？ | ["是否列举评估指标", "是否说明各指标适用场景"] |
| 3 | 深度学习有哪些应用？ | ["是否列举应用领域", "是否给出具体案例"] |

预览展示后，询问用户：

```
是否确认保存以上评测点？
- Y：保存并覆盖原文件
- n：取消生成，不修改文件
- 调整：描述调整需求，重新生成
- 查看更多：展示更多数据的评测点
```

**⚠️ 必须遵守**：
- 严格展示前3条数据，不得跳过
- 必须等待用户确认，不可自动继续

---

## 步骤4：保存结果

将 keypoint 字段添加到原 `{work-dir}/.eval/{session-id}/evalset/evalset-standard.jsonl`文件：

**字段格式**：字符串数组转为 JSON 字符串存储。
- 示例：`["是否...", "是否..."]` 存储为 `"[\"是否...\", \"是否...\"]"`

**多模型横评场景**：
- 同一 `case_id` 的多条记录共享同一评测点
- 保存时需将评测点复制到同 `case_id` 的所有记录

**注意**：覆盖原文件，不创建新文件。

---

## 返回点

流程结束后返回 **任务4步骤6**（上传评测集）。

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |