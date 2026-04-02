# evalset-parse-process 拆分实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal**：将 evalset-parse-process.md 拆分为主流程 + 3个子流程，符合流水线设计模式

**Architecture**：主流程保留步骤1-4框架和门控逻辑，步骤3详细内容、分支A、分支B分别拆分为独立子流程文件

**Tech Stack**：Markdown Skill 文档，符合 Agent Skills 规范

---

## File Structure

```
processes/
├── evalset-parse-process.md      # 主流程（精简）
├── evalset-field-mapping.md      # 子流程1：字段映射审查
├── evalset-model-selection.md    # 子流程2：模型选择（分支A）
├── evalset-field-validation.md   # 子流程3：字段校验（分支B）
```

---

## Task 1: 创建 evalset-field-mapping.md

**Files:**
- Create: `.claude/skills/model-evaluation/processes/evalset-field-mapping.md`

- [ ] **Step 1: 创建字段映射子流程文件**

```markdown
---
name: evalset-field-mapping
description: Use when evalset-parse-process reaches step 3 and needs to generate field mapping configuration
---

# 字段映射流程

**调用方**：evalset-parse-process.md 步骤3

**返回点**：evalset-parse-process.md 步骤4

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

> **注意**：如果评测集由 AI 助手生成（evalset-create-process），跳过此确认步骤。

---

## Red Flags

- 确认映射缺失时，不得保存配置
- 用户未回复时，不得自动继续
- "映射看起来正确，无需确认"

---

**返回**：evalset-parse-process.md 步骤4
```

---

## Task 2: 创建 evalset-model-selection.md

**Files:**
- Create: `.claude/skills/model-evaluation/processes/evalset-model-selection.md`

- [ ] **Step 1: 创建模型选择子流程文件**

```markdown
---
name: evalset-model-selection
description: Use when evalset-parse-process determines answer field is empty and needs model selection
---

# 模型选择流程

**调用方**：evalset-parse-process.md 步骤4（分支A）

**触发条件**：answer 字段不存在或全空

**返回点**：标准化阶段

---

## 目标

获取可用模型列表，用户选择并保存。

---

## 步骤A1：获取可用模型列表

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_model.py list-models \
  --auth {work-dir}/.eval/auth.json \
  --config {skill-dir}/scripts/cfg/eval-server.cfg \
  --output {work-dir}/.eval/{session-id}/available-models.json
```

**失败处理**：Token 失效 → 引导重新授权

---

## 步骤A2：用户选择并保存

1. 读取 `available-models.json`，展示模型列表（格式：`序号. name - description`）
2. 等待用户输入选择的模型序号或名称
3. 调用脚本保存选择结果

```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_model.py select-models \
  --input {work-dir}/.eval/{session-id}/available-models.json \
  --selection "{用户输入}" \
  --output {work-dir}/.eval/{session-id}/selected-models.json
```

**`--selection` 参数格式**：
- 序号选择：`1` 或 `1,2,3`
- 模型名称：`deepseek-chat` 或 `deepseek-chat,spark-lite`

---

## 常见错误

| 错误 | 解决方案 |
|------|----------|
| 模型列表获取失败 | 引导重新授权或检查网络 |

---

**返回**：标准化阶段
```

---

## Task 3: 创建 evalset-field-validation.md

**Files:**
- Create: `.claude/skills/model-evaluation/processes/evalset-field-validation.md`

- [ ] **Step 1: 创建字段校验子流程文件**

```markdown
---
name: evalset-field-validation
description: Use when evalset-parse-process determines answer field is filled and needs field validation
---

# 字段校验流程

**调用方**：evalset-parse-process.md 步骤4（分支B）

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

**返回**：标准化阶段
```

---

## Task 4: 编辑 evalset-parse-process.md

**Files:**
- Modify: `.claude/skills/model-evaluation/processes/evalset-parse-process.md`

- [ ] **Step 1: 精简主流程，替换全文内容**

将原文档替换为精简版，保留步骤1-4框架，添加子流程调用入口：

```markdown
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
```

---

## Task 5: 验证拆分结果

**Files:**
- Read: `.claude/skills/model-evaluation/processes/evalset-parse-process.md`
- Read: `.claude/skills/model-evaluation/processes/evalset-field-mapping.md`
- Read: `.claude/skills/model-evaluation/processes/evalset-model-selection.md`
- Read: `.claude/skills/model-evaluation/processes/evalset-field-validation.md`

- [ ] **Step 1: 检查文件完整性**

检查所有文件是否已创建：
- evalset-parse-process.md（主流程）
- evalset-field-mapping.md（子流程1）
- evalset-model-selection.md（子流程2）
- evalset-field-validation.md（子流程3）

- [ ] **Step 2: 检查功能一致性**

对比原文档，确保：
- 所有步骤都有对应位置
- 所有脚本命令都已保留
- 所有用户交互点都已保留
- 所有禁止规则都已保留
- 所有变量定义都已保留

- [ ] **Step 3: 检查调用链完整性**

确保调用链正确：
- 主流程步骤3 → evalset-field-mapping.md → 返回主流程步骤4
- 主流程步骤4（分支A）→ evalset-model-selection.md → 返回标准化阶段
- 主流程步骤4（分支B）→ evalset-field-validation.md → 返回标准化阶段

---

## 实施完成标志

所有文件创建/编辑完成，功能一致性验证通过后，拆分任务完成。