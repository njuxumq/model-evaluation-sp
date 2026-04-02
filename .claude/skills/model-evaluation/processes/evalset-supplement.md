---
name: evalset-supplement
description: Use when user has a question-only evaluation dataset and needs AI to generate answers
---

# 评测集补充答案流程

本文档定义补充答案流程，为仅有问题的评测集生成答案。

---

## 触发条件

用户仅有问题评测集，需AI补充答案。

---

## 目标

读取仅有问题的文件，AI 为每条问题生成答案，输出为 JSONL 格式。

---

## 步骤1：获取问题集

支持格式：JSONL、JSON、CSV、Excel（xlsx）

```
请提供仅有问题的评测集文件：
```

复制文件至：`{work-dir}/.eval/{session-id}/evalset/evalset-questions.{ext}`

**注意**：Excel 格式仅作为输入，输出统一转为 JSONL。

---

## 步骤2：验证文件格式

执行脚本验证文件结构：

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py analysis \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-questions.{ext} \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-structure.json
```

验证结果：
- 文件格式有效（JSONL/JSON/CSV/Excel）
- 包含 `question` 字段
- 缺少或为空 `answer` 字段

- **验证通过** → 进入步骤3
- **验证失败** → 提示错误，返回步骤1

---

## 步骤3：AI 生成答案

遍历每条问题，生成理想参考答案。

**主 Agent 职责**（格式处理）：
- JSONL 格式：直接读取问题列表
- 其他格式：读取文件 → 转换为问题列表 → 准备接收答案
- 答案生成后：添加 `answer` 字段 → 输出 JSONL

**SubAgent 职责**（仅答案生成，大批量时启用）：
- 输入：问题文本列表
- 输出：答案文本列表
- 不涉及：文件读取、格式转换、文件写入

**执行方式**：
- 数据量 ≤50 条：主 Agent 直接生成
- 数据量 >50 条：可选使用 SubAgent 分批生成答案
- 主 Agent context 满载时：必须使用 SubAgent 分批生成

**生成要求**：
- 答案完整、准确、符合问题要求
- 作为评测基准
- 保留原始字段，仅补充 `answer` 字段

保存至：`{work-dir}/.eval/{session-id}/evalset/evalset-prepared.jsonl`

---

## 步骤4：预览与确认

展示前3条数据，等待用户确认：

```
是否确认保存？(Y/n/重新生成)
```

**⚠️ 不可跳过**：此步骤必须等待用户确认。

- **Y** → 返回解析流程
- **n** → 取消补充，返回步骤1
- **重新生成** → 返回步骤3

---

## 返回点

流程结束后返回 **eval-build.md 任务4步骤3**（执行解析流程）。

---

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 文件格式不支持 | 文件不是 JSONL/JSON/CSV/Excel 格式 | 转换为支持的格式后重试 |
| 缺少 question 字段 | 文件不包含问题字段 | 确保文件包含 `question` 字段 |
| 答案风格不一致 | 多个 SubAgent 生成风格差异 | 主 Agent 后处理统一风格，或单 Agent 生成 |
| 用户确认被跳过 | ClaudeCode 直接保存 | 预览步骤标注 `⚠️ 不可跳过`，必须等待确认 |

---

## 产物命名规范

| 产物 | 说明 |
|------|------|
| `evalset-prepared.jsonl` | JSONL 格式评测集（含答案） |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{ext}` | 文件扩展名（jsonl/json/csv/xlsx） |
| `{skill-dir}` | 技能安装目录 |
| `{python-env}` | Python环境变量前缀 |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |