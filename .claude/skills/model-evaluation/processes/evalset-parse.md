---
name: evalset-parse
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

**输出字段**：`file`, `format`, `total_rows`, `fields`

---

## 步骤3：生成初始映射

**判断**：`evalset-fields-mapping.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤4 |
| 不存在 | 执行子流程 |

执行 [evalset-field-mapping.md](./evalset-field-mapping.md) 完成字段映射生成与确认。

完成后 → 步骤4

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
| `all_empty` | answer 字段不存在或全空 → 执行 [evalset-model-selection.md](./evalset-model-selection.md) |
| `all_filled` | answer 字段全部非空 → 执行 [evalset-field-validation.md](./evalset-field-validation.md) |
| `partial` | answer 字段部分为空 → 询问用户 |

**partial 处理**：

询问用户：
> 评测集中部分 answer 为空，请选择处理方式：
> 1. 视为"只有问题"（空 answer 的记录将云端推理）
> 2. 视为"问题+答案"（跳过空 answer 的记录）

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