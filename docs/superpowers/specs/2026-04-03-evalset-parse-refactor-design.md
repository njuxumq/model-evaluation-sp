---
name: evalset-parse-refactor
description: evalset-parse.md 文档重构设计规格
---

# evalset-parse.md 重构设计规格

## 问题描述

`evalset-parse.md` 存在以下问题：

| 问题 | 现状 | 影响 |
|------|------|------|
| 步骤描述不清晰 | 无目的说明，用户不知为何要做 | 交互困惑 |
| 流程速查缺失 | 子流程未列出，引用混乱 | 无法快速定位 |
| "分支"概念混乱 | 分支A/B命名无语义 | 用户不理解 |
| 产物定义缺失 | 产物分散在各步骤 | 来源去向不明 |
| 变量速查冗余 | 与 SKILL.md 重复 | Token浪费 |

## 设计目标

- 内容精简
- 结构清晰
- 步骤明确
- 交互友好

## 改进方案

### 1. 步骤描述格式统一

每步骤采用固定格式：

```markdown
## 步骤X：名称

**目的**：一句话说明。

| 状态 | 动作 |
|------|------|
| 条件1 | → 下一步 |
| 条件2 | 执行操作 |

命令/说明（按需）。
```

### 2. 流程速查表

新增流程速查表，统一子流程编号：

| 编号 | 名称 | 调用时机 |
|------|------|----------|
| 流程3.1 | 字段映射 | 步骤3 |
| 流程3.2 | 模型选择 | 步骤4检测结果为空 |
| 流程3.3 | 字段校验 | 步骤4检测结果为非空 |

编号规则：流程3为父流程编号（eval-set.md定义），3.X为子流程编号。

### 3. 去掉"分支"概念

将"分支A/B"改为"流程3.2/3.3"，使用编号引用：

| 原命名 | 新命名 |
|--------|--------|
| 分支A：模型选择 | 流程3.2 |
| 分支B：字段校验 | 流程3.3 |

### 4. 产物定义精简

新增产物表格，2列精简：

| 文件 | 用途 |
|------|------|
| `evalset-structure.json` | 字段结构分析 |
| `evalset-fields-mapping.json` | 字段映射配置 |
| `evalset-status.json` | 状态检测结果 |
| `selected-models.json` | 模型列表（流程3.2产出） |

### 5. 变量速查改为引用

```markdown
## 变量速查

见 [SKILL.md 变量速查](../SKILL.md#变量速查)。
```

### 6. 补充核心原则

```markdown
核心原则：**步骤顺序执行，检测结果决定子流程**。
```

## 重构后文档结构

```markdown
---
name: evalset-parse
description: Use when eval-build reaches evalset parsing step, or user directly requests parsing an evaluation dataset
---

# 评测集解析流程

## 目标

解析评测集 → 检测状态 → 调用子流程 → 返回标准化阶段。

核心原则：**步骤顺序执行，检测结果决定子流程**。

---

## 流程速查

| 编号 | 名称 | 调用时机 |
|------|------|----------|
| 流程3.1 | 字段映射 | 步骤3 |
| 流程3.2 | 模型选择 | 步骤4检测结果为空 |
| 流程3.3 | 字段校验 | 步骤4检测结果为非空 |

---

## 步骤1：获取评测集文件

**目的**：确保评测集文件就位。

| 状态 | 动作 |
|------|------|
| `evalset-prepared.{ext}` 已存在 | → 步骤2 |
| 不存在 | 获取用户文件 |

获取方式：复制用户文件或远程下载至 `{work-dir}/.eval/{session-id}/evalset/`。

---

## 步骤2：解析字段结构

**目的**：识别评测集的字段组成，为字段映射提供依据。

| 状态 | 动作 |
|------|------|
| `evalset-structure.json` 已存在 | → 步骤3 |
| 不存在 | 执行下方命令 |

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py analysis \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-structure.json
```

输出字段：`file`, `format`, `total_rows`, `fields`。

---

## 步骤3：生成字段映射

**目的**：建立原始字段与标准字段的对应关系。

| 状态 | 动作 |
|------|------|
| `evalset-fields-mapping.json` 已存在 | → 步骤4 |
| 不存在 | 执行流程3.1 |

执行 **流程3.1** 完成字段映射生成与确认。

---

## 步骤4：检测评测集状态

**目的**：判断 answer 字段状态，决定后续处理流程。

**必须执行脚本检测**，不得凭直觉判断。

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_set.py check-status \
  --input {work-dir}/.eval/{session-id}/evalset/evalset-prepared.{ext} \
  --mapping {work-dir}/.eval/{session-id}/evalset/evalset-fields-mapping.json \
  --output {work-dir}/.eval/{session-id}/evalset/evalset-status.json
```

读取 `evalset-status.json` 中 `answer.status`：

| status | 后续流程 |
|--------|----------|
| `all_empty` | → 流程3.2 |
| `all_filled` | → 流程3.3 |
| `partial` | 询问用户 |

**partial 处理**：询问用户视为"只有问题"或"问题+答案"。

---

## 产物

| 文件 | 用途 |
|------|------|
| `evalset-structure.json` | 字段结构分析 |
| `evalset-fields-mapping.json` | 字段映射配置 |
| `evalset-status.json` | 状态检测结果 |
| `selected-models.json` | 模型列表（流程3.2产出） |

---

## Red Flags

- 未执行 check-status 就选择流程
- 用户未回复时自动继续

---

## 变量速查

见 [SKILL.md 变量速查](../SKILL.md#变量速查)。
```

## 字数对比

| 版本 | 字数 |
|------|------|
| 原文档 | ~160字 |
| 重构后 | ~450字 |

扩充原因：补充必要信息（目的说明、流程速查、产物定义）。

## 关联文档修改

| 文档 | 修改内容 |
|------|----------|
| `evalset-field-mapping.md` | 调用方改为"流程3.1"，返回点改为"流程3步骤4" |
| `evalset-model-selection.md` | 调用方改为"流程3.2" |
| `evalset-field-validation.md` | 调用方改为"流程3.3" |
| `eval-set.md` | 流程速查表增加流程3.1/3.2/3.3 |

## 验证清单

- [ ] 步骤目的清晰
- [ ] 流程速查完整
- [ ] 产物定义清晰
- [ ] 变量速查引用
- [ ] Red Flags 精简
- [ ] 关联文档同步修改