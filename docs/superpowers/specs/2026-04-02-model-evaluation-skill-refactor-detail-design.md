# model-evaluation Skill 重构详细设计方案

本文档是 [大纲设计方案](./2026-04-02-model-evaluation-skill-refactor-design.md) 的详细展开，定义每个文件的具体内容结构、字段格式和交互逻辑。

---

## 一、编排层详细设计

### 1.1 orchestrator.md 结构

**职责**：流程编排、阶段调度、状态管理、意图识别

**YAML Frontmatter**：
```yaml
---
name: orchestrator
description: 流程编排器。加载后负责评测流程的阶段调度和状态管理。
---
```

**内容结构**：

```markdown
# 流程编排器

## 意图识别

分析用户输入，识别用户意图：

| 用户意图 | 识别关键词 | 执行动作 |
|----------|------------|----------|
| 新建评测 | "新建"、"创建"、"开始评测" | 进入初始化阶段 |
| 继续评测 | "继续"、"恢复"、会话ID | 跳转至指定阶段 |
| 查看进度 | "进度"、"状态" | 跳转至执行阶段查询 |
| 查看结果 | "结果"、"报告" | 跳转至执行阶段展示 |
| 重构Skill | "重构"、"改进" | 进入重构保证流程 |

## 阶段调度

| 阶段 | 入口文件 | 前置条件 | 完成标志 |
|------|----------|----------|----------|
| 初始化 | [phases/init-phase.md](./phases/init-phase.md) | 无 | env.cfg + auth.json + session-id/ |
| 构建配置 | [phases/build-phase.md](./phases/build-phase.md) | 初始化完成 | eval-dimension.json + eval-judge.json |
| 评测集处理 | [phases/evalset-phase.md](./phases/evalset-phase.md) | 构建配置完成 | evalset-meta.json |
| 执行评测 | [phases/execute-phase.md](./phases/execute-phase.md) | 评测集处理完成 | evaltask-result.json |

## 状态管理

加载 [checkpoints.md](./checkpoints.md) 获取各阶段检查点定义。

## 流程跳转

加载 [transitions.md](./transitions.md) 获取跳转规则。
```

### 1.2 checkpoints.md 结构

**职责**：定义各阶段的前置条件和完成标志

**YAML Frontmatter**：
```yaml
---
name: checkpoints
description: 检查点定义。定义各阶段的验证条件和门控规则。
---
```

**内容结构**：

```markdown
# 检查点定义

## 初始化阶段

### 前置条件
无

### 完成标志
| 检查项 | 文件路径 | 验证方法 |
|--------|----------|----------|
| 环境配置 | {work-dir}/.eval/env.cfg | 文件存在且字段完整 |
| 鉴权Token | {work-dir}/.eval/auth.json | 文件存在且Token有效 |
| 会话目录 | {work-dir}/.eval/{session-id}/ | 目录存在 |

## 构建配置阶段

### 前置条件
- 初始化阶段完成标志全部满足

### 完成标志
| 检查项 | 文件路径 | 验证方法 |
|--------|----------|----------|
| 维度配置 | {session-id}/eval-dimension.json | 文件存在且格式正确 |
| 评委配置 | {session-id}/eval-judge.json | 文件存在且格式正确 |

## 评测集处理阶段

### 前置条件
- 构建配置阶段完成标志全部满足

### 完成标志
| 检查项 | 文件路径 | 验证方法 |
|--------|----------|----------|
| 评测集元信息 | {session-id}/evalset/evalset-meta.json | 文件存在且包含dataset字段 |

## 执行评测阶段

### 前置条件
- 评测集处理阶段完成标志全部满足

### 完成标志
| 检查项 | 文件路径 | 验证方法 |
|--------|----------|----------|
| 评测结果 | {session-id}/evaltask/evaltask-result.json | 文件存在 |
```

### 1.3 transitions.md 结构

**职责**：定义状态转换规则和跳转逻辑

**YAML Frontmatter**：
```yaml
---
name: transitions
description: 状态转换规则。定义阶段间流转和跳转逻辑。
---
```

**内容结构**：

```markdown
# 状态转换规则

## 正常流转

```
初始化 → 构建配置 → 评测集处理 → 执行评测 → 完成
```

**流转条件**：当前阶段完成标志全部满足后自动进入下一阶段

## 跳转规则

| 跳转类型 | 触发条件 | 目标位置 | 验证要求 |
|----------|----------|----------|----------|
| 前向跳转 | 用户明确指定阶段/任务 | 目标阶段 | 验证前置条件 |
| 后向跳转 | 用户请求修改配置 | 指定阶段 | 无需验证 |
| 恢复执行 | 用户说"继续"且存在历史会话 | 断点阶段 | 验证断点状态 |

## 跳转验证

前向跳转时需验证目标阶段的前置条件：

| 目标阶段 | 验证内容 |
|----------|----------|
| 构建配置 | env.cfg存在、auth.json有效、session-id目录存在 |
| 评测集处理 | eval-dimension.json存在、eval-judge.json存在 |
| 执行评测 | evalset-meta.json存在 |

## 异常处理

| 异常情况 | 处理方式 |
|----------|----------|
| 前置条件不满足 | 返回对应阶段重新执行 |
| 文件损坏或丢失 | 提示用户并引导重新生成 |
| Token失效 | 引导重新授权 |
```

---

## 二、执行层详细设计

### 2.1 phases/ 目录结构

| 文件 | 职责 | 调用的 collectors/generators |
|------|------|------------------------------|
| init-phase.md | 初始化执行单元 | 无 |
| build-phase.md | 构建配置执行单元 | scene-collector, dimension-collector, dimension-generator |
| evalset-phase.md | 评测集处理执行单元 | evalset-collector, evalset-generator, keypoint-generator |
| execute-phase.md | 执行评测执行单元 | 无 |

### 2.2 init-phase.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: init-phase
description: 初始化阶段执行单元。环境检测、鉴权验证、会话目录创建。
---
```

**内容结构**：

```markdown
# 初始化阶段

## 目标

完成环境检测、鉴权验证、会话目录创建。

## 前置验证

无前置条件。

## 执行步骤

### 步骤1：环境检测

加载 [knowledge/env-knowledge.md](../knowledge/env-knowledge.md) 获取环境检测知识。

执行检测：
```bash
# 检测Python命令
{python-env}{python-cmd} --version

# 检测必需依赖
{python-env}{python-cmd} -c "import requests; print('OK')"
```

**输出变量**：`{python-cmd}`, `{python-env}`

### 步骤2：鉴权验证

加载 [knowledge/auth-knowledge.md](../knowledge/auth-knowledge.md) 获取鉴权知识。

检查Token有效性：
```bash
{python-env}{python-cmd} {skill-dir}/scripts/eval_auth.py check --output {work-dir}/.eval/auth.json
```

| 状态 | 动作 |
|------|------|
| valid | 继续下一步 |
| invalid/not_found | 执行登录授权 |

### 步骤3：会话目录

判断用户意图：

| 意图 | 动作 |
|------|------|
| 新建 | 创建新session-id目录 |
| 继续 | 使用现有session-id目录 |

## 完成标志

- {work-dir}/.eval/env.cfg 存在
- {work-dir}/.eval/auth.json 存在且有效
- {work-dir}/.eval/{session-id}/ 目录存在

## 下一步

自动进入构建配置阶段（加载 build-phase.md）
```

### 2.3 build-phase.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: build-phase
description: 构建配置阶段执行单元。场景确认、维度配置、评委配置。
---
```

**内容结构**：

```markdown
# 构建配置阶段

## 目标

完成评测场景确认、维度配置、评委配置。

## 前置验证

加载 [checkpoints.md](../checkpoints.md) 验证初始化阶段完成标志。

## 执行步骤

### 步骤1：收集场景信息

执行 [collectors/scene-collector.md](../collectors/scene-collector.md) 收集评测场景。

**输出变量**：`{scene}`, `{eval-type}`

### 步骤2：生成维度配置

执行 [generators/dimension-generator.md](../generators/dimension-generator.md) 生成维度配置。

**输出文件**：`{session-id}/eval-dimension.json`

### 步骤3：配置评委

复制默认评委配置：
```bash
cp {skill-dir}/assets/eval-judge.json {work-dir}/.eval/{session-id}/eval-judge.json
```

## 完成标志

- eval-dimension.json 存在且格式正确
- eval-judge.json 存在且格式正确

## 下一步

自动进入评测集处理阶段（加载 evalset-phase.md）
```

### 2.4 collectors/ 目录设计

#### scene-collector.md

**YAML Frontmatter**：
```yaml
---
name: scene-collector
description: 场景信息收集器。收集评测场景和评测方式。
---
```

**内容结构**：

```markdown
# 场景信息收集器

## 触发条件

构建配置阶段开始时调用。

## 收集策略

### 自动推断（优先）

检查历史对话中是否有场景信息：

| 信息类型 | 识别关键词 | 推断结果 |
|----------|------------|----------|
| 场景名称 | "知识问答"、"内容创造"、... | 直接使用 |
| 场景描述 | "评测模型的知识问答能力"、... | 匹配专家模板 |

加载 [knowledge/template-knowledge.md](../knowledge/template-knowledge.md) 获取专家模板列表。

### 主动收集（推断失败时）

展示专家模板列表（表格格式：序号、名称、适用说明、评测方式），末尾添加"其他场景"选项。

**收集问题**：
1. 请选择评测场景？（单选）

**输出变量**：
- `{scene}`：评测场景名称
- `{eval-type}`：评测方式（dimension-level / case-level）

## 返回点

收集完成后返回构建配置阶段步骤2。
```

#### dimension-collector.md

**YAML Frontmatter**：
```yaml
---
name: dimension-collector
description: 维度偏好收集器。收集维度权重调整偏好。
---
```

**内容结构**：

```markdown
# 维度偏好收集器

## 触发条件

维度配置生成后，需用户确认时调用。

## 收集策略

### 展示配置摘要

展示维度列表及权重，询问确认。

### 收集偏好

**收集问题**：
1. 是否需要调整维度配置？（Y/n/调整）
   - Y：确认保存
   - n：重新生成
   - 调整：收集具体调整需求

### 调整范围

| 评测方式 | 允许调整 |
|----------|----------|
| dimension-level | 权重调整、维度增删 |
| case-level | 仅修改prompt字段内容 |

## 返回点

收集完成后返回构建配置阶段步骤3。
```

#### evalset-collector.md

**YAML Frontmatter**：
```yaml
---
name: evalset-collector
description: 评测集信息收集器。收集评测集来源信息。
---
```

**内容结构**：

```markdown
# 评测集信息收集器

## 触发条件

评测集处理阶段开始时调用。

## 收集策略

### 自动推断（优先）

检查历史对话中是否有文件路径或下载链接。

### 主动收集（推断失败时）

**收集问题**：
1. 是否有现成的评测集？（提供评测集 / 生成问题集）

**选项说明**：
- 提供评测集：用户上传或提供文件路径
- 生成问题集：AI根据场景和维度生成

## 输出变量

| 变量 | 说明 |
|------|------|
| `{evalset-source}` | 评测集来源类型：file / generate |
| `{evalset-path}` | 文件路径（来源为file时） |

## 返回点

收集完成后返回评测集处理阶段对应步骤。
```

### 2.5 generators/ 目录设计

#### dimension-generator.md

**YAML Frontmatter**：
```yaml
---
name: dimension-generator
description: 维度配置生成器。根据场景生成评测维度配置。
---
```

**内容结构**：

```markdown
# 维度配置生成器

## 触发条件

构建配置阶段步骤2调用。

## 生成策略

### 专家模板匹配

加载 [knowledge/template-knowledge.md](../knowledge/template-knowledge.md) 获取专家模板列表。

| 匹配结果 | 动作 |
|----------|------|
| 完全匹配 | 复制专家模板 |
| 部分匹配 | 使用专家模板 + 补充维度 |
| 无匹配 | 使用通用维度组合 |

### 生成流程

1. 根据场景匹配专家模板
2. 复制模板至会话目录
3. 填充judge_id（从eval-judge.json读取）
4. 展示配置摘要供确认

### 模板复制命令

```bash
cp {skill-dir}/assets/experts/{template-name}.json {work-dir}/.eval/{session-id}/eval-dimension.json
```

## 输出文件

`{session-id}/eval-dimension.json`

## 格式校验

加载 [schemas/dimension-schema.md](../schemas/dimension-schema.md) 校验输出格式。
```

#### evalset-generator.md

**YAML Frontmatter**：
```yaml
---
name: evalset-generator
description: 问题集生成器。根据场景和维度生成问题集。
---
```

**内容结构**：

```markdown
# 问题集生成器

## 触发条件

评测集来源为"生成问题集"时调用。

## 生成策略

### 配置收集

使用AskUserQuestion一次性收集：

| 配置项 | 问题 | 默认值 |
|--------|------|--------|
| 数量 | 生成多少条问题？ | 10 |
| 类型分布 | 问题类型偏好？ | 均衡分布 |
| 难度分布 | 难度偏好？ | 均衡分布 |

### 生成规则

- 问题覆盖评测维度关注点
- 问题风格与评测场景匹配
- 字段格式：case_id, question, category

### 预览确认

展示前3条问题预览，等待用户确认。

## 输出文件

`{session-id}/evalset/evalset-prepared.jsonl`

## 字段格式

```jsonl
{"case_id": "case-001", "question": "...", "category": "..."}
```
```

#### keypoint-generator.md

**YAML Frontmatter**：
```yaml
---
name: keypoint-generator
description: 评测点生成器。根据问题和参考答案生成评测点。
---
```

**内容结构**：

```markdown
# 评测点生成器

## 触发条件

定制用例级评测且评测集无keypoint字段时调用。

## 生成策略

### 场景判断

根据字段组合识别场景：

| 场景 | 字段组合 | 评测点来源 |
|------|----------|------------|
| QR | question + reference | 从参考答案提炼 |
| QC | question + context | 从上下文推导 |
| QRC | question + reference + context | 综合提炼 |
| Q | 仅question | 基于问题推断 |

### 生成方式

1. 读取keypoint_prompts.py中的SYSTEM_PROMPT
2. 构建用户提示词
3. 调用Agent工具批量生成
4. 预览前3条确认

### 多模型横评处理

按唯一case_id去重生成，保存时复制到同case_id的所有记录。

## 输出格式

```json
["是否提及xxx", "是否包含xxx"]
```

## 输出文件

覆盖 `{session-id}/evalset/evalset-standard.jsonl`
```

---

## 三、知识层详细设计

### 3.1 knowledge/ 目录结构

| 文件 | 职责 | 来源文件 |
|------|------|----------|
| api-knowledge.md | 评测服务API规范 | references/评测服务接口说明.md |
| auth-knowledge.md | 认证服务API规范 | references/认证服务接口说明.md |
| script-knowledge.md | 脚本定义和用法 | references/脚本定义.md |
| template-knowledge.md | 内置模板说明 | references/内置模板说明.md |
| dimension-knowledge.md | 评测维度说明 | references/评测维度说明.md |
| error-knowledge.md | 错误码和错误处理 | 从API文档提取 |
| env-knowledge.md | 环境检测知识 | 新增 |

### 3.2 api-knowledge.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: api-knowledge
description: 评测服务API知识。调用评测服务API时加载。
---
```

**内容结构**：

```markdown
# 评测服务API知识

## 快速参考

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 上传评测集 | POST | /open/api/v1/evalset | 上传评测数据 |
| 获取模型列表 | GET | /open/api/v1/models | 获取可用推理模型 |
| 提交评测任务 | POST | /open/api/v1/eval/tasks | 创建评测任务 |
| 查询任务详情 | GET | /open/api/v1/eval/tasks/:id | 获取任务状态 |

## 认证方式

| Header | 值 |
|--------|------|
| Authorization | Bearer {access_token} |
| Content-Type | application/json |

## 错误码

| 错误码 | 说明 | 处理方式 |
|--------|------|----------|
| 10002 | Token失效 | 重新授权 |
| 80001 | 评测集参数无效 | 检查参数格式 |
| 90001 | 任务定义无效 | 检查配置完整性 |

## 详细规范

完整API规范见原文件：[原文件路径](../references/评测服务接口说明.md)
```

### 3.3 auth-knowledge.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: auth-knowledge
description: 认证服务API知识。处理鉴权时加载。
---
```

**内容结构**：

```markdown
# 认证服务API知识

## 认证流程

```
1. 生成state_token
2. 调用登录初始化获取login_url
3. 用户浏览器完成登录
4. 用户获取授权码code
5. 用code+state_token换取access_token
```

## 登录模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| 回调模式 | 自动启动HTTP服务器接收授权码 | 本地桌面环境 |
| OOB模式 | 手动复制授权码 | 服务器终端 |

## 脚本命令

```bash
# 智能登录
python eval_auth.py login --config {cfg} --output {output}

# 检查Token
python eval_auth.py check --output {output}
```

## 错误处理

| 错误 | 处理方式 |
|------|----------|
| state mismatch | 重新生成state_token |
| unauthorized | 重新授权 |
```

### 3.4 script-knowledge.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: script-knowledge
description: 脚本定义知识。执行脚本时加载。
---
```

**内容结构**：

```markdown
# 脚本定义知识

## 脚本清单

| 脚本 | 功能 | 子命令 |
|------|------|--------|
| eval_auth.py | 鉴权管理 | detect, login, token, check |
| eval_set.py | 评测集管理 | analysis, normalize, submit, expand, check-status |
| eval_model.py | 模型管理 | list-models, select-models |
| eval_task.py | 任务管理 | submit, status, summary |
| eval_dimension.py | 维度工具 | check, update |

## 执行格式

```bash
{python-env}{python-cmd} {skill-dir}/scripts/{script} {subcommand} {args}
```

## 子命令速查

### eval_set.py

| 子命令 | 用途 | 必要参数 |
|--------|------|----------|
| analysis | 解析评测集结构 | --input, --output |
| normalize | 标准化评测集 | --input, --mapping, --output |
| expand | 展开评测集（无答案场景） | --input, --mapping, --models, --output |
| submit | 上传评测集 | --auth, --config, --evalset, --output |
| check-status | 检查字段状态 | --input, --mapping, --output |

### eval_task.py

| 子命令 | 用途 | 必要参数 |
|--------|------|----------|
| submit | 提交任务 | --config, --auth, --eval_set, --eval_dimension, --eval_judge, --output |
| status | 查询状态 | --config, --auth, --evaltask, --output |
| summary | 生成摘要 | --result, --platform_url |

## 详细定义

完整脚本定义见原文件：[原文件路径](../references/脚本定义.md)
```

### 3.5 template-knowledge.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: template-knowledge
description: 内置模板知识。选择专家模板或维度模板时加载。
---
```

**内容结构**：

```markdown
# 内置模板知识

## 专家模板列表

### 通用维度级评测

| 模板名称 | 适用场景 | 模板路径 |
|----------|----------|----------|
| 内容创造 | 文章创作、文案生成 | assets/experts/内容创造.json |
| 知识库问答 | 基于知识库的问答系统 | assets/experts/知识库问答.json |
| 旅游出行 | 旅游攻略、行程规划 | assets/experts/旅游出行.json |
| ... | ... | ... |

### 定制用例级评测

| 模板名称 | 适用场景 | 模板路径 |
|----------|----------|----------|
| 营销数字人评测 | 虚拟人多轮对话 | assets/experts/营销数字人评测.json |
| 知识问答 | 通用知识问答 | assets/experts/知识问答.json |

## 维度模板分类

| 分类 | 说明 | 典型维度 |
|------|------|----------|
| 质量评估类 | 内容质量特征 | 相关性、逻辑连贯性、完整性 |
| 效果验证类 | 目标达成验证 | 有效性、准确性、正确性 |
| 约束遵循类 | 规则约束检查 | 指令遵循、格式合规 |
| 语义相似度类 | 文本相似计算 | BLEU、ROUGE、BERTScore |

## 模板使用

1. 匹配专家模板 → 复制至会话目录
2. 无匹配 → 选择维度模板组合
3. 转化规则见 [knowledge/transform-knowledge.md](./transform-knowledge.md)
```

### 3.6 schemas/ 目录设计

#### evalset-schema.md

**YAML Frontmatter**：
```yaml
---
name: evalset-schema
description: 评测集数据格式定义。校验评测集格式时加载。
---
```

**内容结构**：

```markdown
# 评测集数据格式

## 标准字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|:----:|------|
| question | string | ✅ | 评测问题 |
| answer | string | ✅ | 模型回答（可为空字符串） |
| model | string | ✅ | 模型标识 |
| case_id | string | ✅ | 用例标识 |

## 可选字段

| 字段 | 类型 | 说明 |
|------|------|------|
| system | string | 系统提示词 |
| context | string | 上下文信息 |
| category | string | 分类标签 |
| reference | string | 参考答案 |
| keypoint | string | 评测点（JSON字符串数组） |
| metainfo | object | 元信息 |

## 格式示例

```jsonl
{"case_id": "case-001", "question": "什么是大语言模型？", "answer": "...", "model": "gpt-4"}
```

## 校验规则

- 每行必须是有效JSON
- 必填字段不能缺失
- case_id在同一问题的多模型回答中必须相同
```

#### dimension-schema.md

**YAML Frontmatter**：
```yaml
---
name: dimension-schema
description: 维度配置格式定义。校验维度配置时加载。
---
```

**内容结构**：

```markdown
# 维度配置格式

## 根节点字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|:----:|------|
| name | string | ✅ | 评测场景名称 |
| description | string | ❌ | 评测场景描述 |
| evals | array | ✅ | 评测维度数组 |

## 维度对象字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|:----:|------|
| name | string | ✅ | 维度名称 |
| type | string | ✅ | llm-score / llm-judge / builtin |
| judge_id | string | ✅* | 评委ID（主观评测必填） |
| func | string | ✅* | 内置函数（客观评测必填） |
| weight | number | ✅ | 权重（0-1，总和为1） |
| prompt | object | ✅* | 提示词配置（主观评测必填） |

## 校验规则

- evals必须为数组（非config嵌套）
- 主观评测必须有judge_id和prompt
- 客观评测必须有func
- 所有维度weight总和必须为1.0
```

---

## 四、审查层详细设计

### 4.1 validators/ 目录结构

| 文件 | 职责 | 审查时机 |
|------|------|----------|
| red-flags.md | 禁止规则清单 | 每次行动前 |
| config-validator.md | 配置完整性检查 | 提交任务前 |
| evalset-validator.md | 评测集格式检查 | 上传评测集前 |

### 4.2 red-flags.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: red-flags
description: 禁止规则清单。每次行动前审查。
---
```

**内容结构**：

```markdown
# 禁止规则清单

## 流程类禁止

| 禁止行为 | 正确做法 |
|----------|----------|
| 跳过缓存检测直接重新执行环境检测 | 优先检查env.cfg缓存 |
| Token失效时跳过重新授权 | 必须重新授权 |
| 当前阶段未完成时询问后续阶段问题 | 等当前阶段完成 |
| 复合bash命令 | 逐条执行命令 |

## 交互类禁止

| 禁止行为 | 正确做法 |
|----------|----------|
| 跳过用户确认步骤 | 必须等待用户确认 |
| 替用户做维度权重决定 | 必须经用户确认 |
| 帮助用户生成评测集内容 | 评测集需用户真实数据 |

## 配置类禁止

| 禁止行为 | 正确做法 |
|----------|----------|
| 修改assets/或scripts/目录文件 | 只读使用，配置通过参数传递 |
| 权重总和不为1.0 | 确保权重总和为1.0 |
| 用例级评测调整权重或维度 | 仅允许修改prompt字段 |

## 常见借口

| 借口 | 现实 |
|------|------|
| "用户应该知道选哪个" | 用户可能不清楚概念，需展示选项 |
| "这个确认可以跳过" | 所有确认步骤都是必需的 |
| "默认配置就行" | 维度权重影响评测结果，必须确认 |
```

### 4.3 config-validator.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: config-validator
description: 配置完整性检查器。提交任务前执行。
---
```

**内容结构**：

```markdown
# 配置完整性检查器

## 检查清单

### 评测维度配置

| 检查项 | 验证方法 | 失败处理 |
|--------|----------|----------|
| evals数组存在 | JSON解析 | 返回构建阶段 |
| judge_id已填充 | 字段非空检查 | 从eval-judge.json读取并填充 |
| 权重总和为1.0 | 数值计算 | 提示用户调整 |

### 评委配置

| 检查项 | 验证方法 | 失败处理 |
|--------|----------|----------|
| models数组存在 | JSON解析 | 使用默认配置 |
| id字段存在 | 字段检查 | 返回构建阶段 |

### 评测集

| 检查项 | 验证方法 | 失败处理 |
|--------|----------|----------|
| dataset字段存在 | JSON解析 | 返回评测集阶段 |
| 文件可访问 | 文件存在检查 | 重新上传 |

## 校验脚本

```bash
python eval_dimension.py -a check -d {dimension-file}
```
```

### 4.4 evalset-validator.md 详细设计

**YAML Frontmatter**：
```yaml
---
name: evalset-validator
description: 评测集格式检查器。上传评测集前执行。
---
```

**内容结构**：

```markdown
# 评测集格式检查器

## 检查清单

### 格式检查

| 检查项 | 验证方法 | 失败处理 |
|--------|----------|----------|
| JSONL格式有效 | 逐行JSON解析 | 提示格式错误行号 |
| 必填字段存在 | 字段检查 | 提示缺失字段 |

### 字段状态检查

| 检查项 | 验证方法 | 后续分支 |
|--------|----------|----------|
| answer字段状态 | check-status脚本 | all_empty → expand / all_filled → normalize |
| model字段状态 | check-status脚本 | 空时询问评测模式 |

## 校验脚本

```bash
python eval_set.py check-status --input {input} --mapping {mapping} --output {output}
```
```

---

## 五、重构保证方案详细设计

### 5.1 分析器详细设计

**输入**：用户重构请求

**输出**：scope.json

**分析维度**：

```markdown
## 文件清单分析

| 分析项 | 分析方法 | 输出字段 |
|--------|----------|----------|
| 文件列表 | Glob匹配 | files[].path |
| 文件类型 | 扩展名判断 | files[].type (script/process/reference/asset) |
| 文件角色 | 内容分析 | files[].role (核心流程/子流程/知识/配置) |
| 可修改性 | 目录判断 | files[].modifiable (assets/scripts不可修改) |
| 风险等级 | 影响范围评估 | files[].risk (low/medium/high) |

## 依赖关系分析

| 分析项 | 分析方法 | 输出字段 |
|--------|----------|----------|
| 引用关系 | Grep搜索\[.*\]\( | dependencies[].from, to |
| 引用类型 | 内容判断 | dependencies[].type (process_call/knowledge_load) |
| 引用强度 | 是否必须 | dependencies[].strength (strong/weak) |

## 影响范围分析

| 分析项 | 分析方法 | 输出字段 |
|--------|----------|----------|
| 直接影响 | 当前文件引用者 | impact_scope.direct |
| 间接影响 | 引用链追溯 | impact_scope.indirect |

## 风险评估

| 风险等级 | 判断条件 |
|----------|----------|
| low | 独立文件，无依赖 |
| medium | 被1-2个文件依赖 |
| high | 被3+个文件依赖，或为核心流程文件 |
| critical | 修改会影响所有流程 |
```

**输出格式（scope.json）**：

```json
{
  "analysis_id": "analyze-{timestamp}",
  "refactor_request": {
    "description": "重构描述"
  },
  "files": [
    {
      "path": "processes/evalset-parse.md",
      "type": "process",
      "role": "核心解析流程",
      "modifiable": true,
      "risk": "high"
    }
  ],
  "dependencies": [
    {
      "from": "eval-set.md",
      "to": "evalset-parse.md",
      "type": "process_call",
      "strength": "strong"
    }
  ],
  "impact_scope": {
    "direct": ["eval-set.md"],
    "indirect": ["eval-execute.md"]
  },
  "risk_level": "medium",
  "completeness_check": {
    "all_files_identified": true,
    "all_deps_traced": true,
    "all_risks_assessed": true
  }
}
```

### 5.2 设计器详细设计

**输入**：scope.json

**输出**：design.json

**设计流程**：

```markdown
## 单元划分规则

| 划分依据 | 划分规则 |
|----------|----------|
| 文件边界 | 每个文件为一个单元 |
| 功能边界 | 独立功能模块为一个单元 |
| 风险边界 | 高风险操作独立单元 |

## 变更类型定义

| 变更类型 | 说明 | 示例 |
|----------|------|------|
| create | 新建文件 | 创建新的collector |
| modify | 修改内容 | 调整流程步骤 |
| rewrite | 重写文件 | 拆分文件结构 |
| delete | 删除文件 | 移除废弃流程 |
| merge | 合并文件 | 合并相似流程 |

## 执行顺序规划

1. 无依赖单元优先
2. 低风险单元优先
3. 依赖单元按依赖顺序执行
```

**输出格式（design.json）**：

```json
{
  "design_id": "design-{timestamp}",
  "analysis_ref": "analyze-{timestamp}",
  "units": [
    {
      "unit_id": "unit-001",
      "file": "processes/evalset-parse.md",
      "change_type": "rewrite",
      "change_scope": "document",
      "change_detail": {
        "target": "全文",
        "action": "拆分为主流程+分支"
      },
      "verification": {
        "method": "流程完整性检查",
        "standard": "所有原步骤有对应位置",
        "rollback": "git checkout"
      },
      "execution_order": 1,
      "dependencies": [],
      "risk": "high"
    }
  ],
  "execution_plan": {
    "total_units": 3,
    "execution_order": ["unit-001", "unit-002", "unit-003"],
    "parallel_units": [],
    "sequential_units": ["unit-001", "unit-002", "unit-003"]
  },
  "rollback_strategy": {
    "method": "git checkout",
    "checkpoint_after_each_unit": true
  }
}
```

### 5.3 审核器详细设计

**输入**：design.json

**输出**：review.json

**审核维度**：

```markdown
## 完整性审核

| 检查项 | 检查方法 | 通过条件 |
|--------|----------|----------|
| 文件覆盖 | files vs units对比 | 100%覆盖 |
| 变更描述 | 每个unit有change_detail | 非空 |

## 依赖一致性审核

| 检查项 | 检查方法 | 通过条件 |
|--------|----------|----------|
| 执行顺序 | execution_order vs dependencies | 无循环依赖 |
| 并行安全 | parallel_units互不依赖 | 无交叉依赖 |

## 风险可控性审核

| 检查项 | 检查方法 | 通过条件 |
|--------|----------|----------|
| 回滚方案 | risk=high有rollback | 100%覆盖 |
| 检查点 | checkpoint_after_each_unit | true |

## 验证可行性审核

| 检查项 | 检查方法 | 通过条件 |
|--------|----------|----------|
| 验证方法 | verification.method可执行 | 明确可操作 |
| 验证标准 | verification.standard明确 | 可判断结果 |
```

**输出格式（review.json）**：

```json
{
  "review_id": "review-{timestamp}",
  "design_ref": "design-{timestamp}",
  "result": "passed",
  "checks": [
    {
      "dimension": "设计完整性",
      "passed": true,
      "details": "所有变更单元已设计"
    },
    {
      "dimension": "依赖一致性",
      "passed": true,
      "details": "执行顺序符合依赖关系"
    },
    {
      "dimension": "风险可控性",
      "passed": true,
      "details": "高风险单元有回滚方案"
    }
  ],
  "issues": [],
  "suggestions": []
}
```

### 5.4 执行器详细设计

**输入**：design.json + review.json

**输出**：changes.json

**执行流程**：

```markdown
## 准备阶段

1. 创建执行目录
2. 备份当前状态：git stash
3. 加载design.json

## 执行阶段

按execution_order顺序执行：

### 每单元执行

1. 执行变更操作
2. 检查点验证
   - 文件存在性
   - 语法正确性（脚本类）
   - 格式正确性（文档类）
3. 记录变更

### 失败处理

| 失败类型 | 处理方式 |
|----------|----------|
| 语法错误 | 回滚 → 返回设计器 |
| 依赖缺失 | 回滚 → 返回分析器 |
| 验证失败 | 回滚当前单元 → 重试 |

## 记录阶段

生成changes.json
```

**输出格式（changes.json）**：

```json
{
  "execution_id": "exec-{timestamp}",
  "design_ref": "design-{timestamp}",
  "executed_units": [
    {
      "unit_id": "unit-001",
      "status": "success",
      "checkpoint_passed": true,
      "changes": [
        {
          "file": "processes/evalset-parse.md",
          "type": "rewrite",
          "lines_added": 50,
          "lines_removed": 30
        }
      ]
    }
  ],
  "git_state": {
    "branch": "refactor-{session-id}",
    "commits": ["commit-001"],
    "stash_ref": "stash@{0}"
  },
  "summary": {
    "total_units": 3,
    "success_units": 3,
    "failed_units": 0
  }
}
```

### 5.5 校验器详细设计

**输入**：changes.json

**输出**：validation.json

**校验维度**：

```markdown
## 功能完整性校验

| 校验项 | 校验方法 | 通过条件 |
|--------|----------|----------|
| 新流程可执行 | 模拟流程执行 | 无阻塞错误 |
| 新文档可加载 | 文件读取测试 | 格式正确 |

## 向后兼容性校验

| 校验项 | 校验方法 | 通过条件 |
|--------|----------|----------|
| 原有场景正常 | 回归测试 | 测试通过 |
| 引用无断裂 | Grep检查 | 无404引用 |

## 文档一致性校验

| 校验项 | 校验方法 | 通过条件 |
|--------|----------|----------|
| 文档与代码一致 | 内容比对 | 描述匹配 |
| 引用路径正确 | 文件存在检查 | 路径有效 |
```

**输出格式（validation.json）**：

```json
{
  "validation_id": "valid-{timestamp}",
  "execution_ref": "exec-{timestamp}",
  "result": "passed",
  "checks": [
    {
      "check_id": 1,
      "name": "流程可执行",
      "method": "模拟执行",
      "passed": true,
      "details": "无阻塞错误"
    }
  ],
  "regression_tests": [
    {
      "scenario": "标准评测流程",
      "passed": true,
      "details": "流程正常完成"
    }
  ],
  "issues": []
}
```

---

## 六、实施步骤详细设计

### 6.1 实施顺序

| 阶段 | 任务 | 输入 | 输出 | 验证方法 |
|------|------|------|------|----------|
| 1 | 创建重构保证流程 | 大纲设计 | refactor-guarantee.md | 文件存在 |
| 2 | 创建编排层 | 现有SKILL.md | orchestrator.md, checkpoints.md, transitions.md | 引用正确 |
| 3 | 创建知识层 | 现有references/ | knowledge/*.md, schemas/*.md | 内容完整 |
| 4 | 创建审查层 | 现有Red Flags | validators/*.md | 规则完整 |
| 5 | 创建收集器 | 现有阶段文档 | collectors/*.md | 流程正确 |
| 6 | 创建生成器 | 现有processes/ | generators/*.md | 流程正确 |
| 7 | 重构阶段单元 | 现有eval-*.md | phases/*.md | 流程完整 |
| 8 | 精简SKILL.md | 新结构 | SKILL.md (精简版) | 引用正确 |
| 9 | 简化processes/ | 重构结果 | processes/ (简化版) | 流程正确 |
| 10 | 最终校验 | 全部产物 | validation.json | 校验通过 |

### 6.2 检查点详细定义

| 检查点 | 验证内容 | 验证命令 | 通过条件 |
|--------|----------|----------|----------|
| CP1 | 编排层文件完整 | `ls orchestrator.md checkpoints.md transitions.md` | 3个文件存在 |
| CP2 | 知识层文件完整 | `ls knowledge/*.md | wc -l` | >= 5个文件 |
| CP3 | 审查层文件完整 | `ls validators/*.md | wc -l` | >= 3个文件 |
| CP4 | 执行层文件完整 | `ls phases/*.md collectors/*.md generators/*.md` | 全部存在 |
| CP5 | SKILL.md精简有效 | `wc -l SKILL.md` | <= 100行 |
| CP6 | 流程可执行 | 模拟评测流程 | 无阻塞错误 |
| CP7 | 向后兼容 | 回归测试 | 原有场景正常 |

---

## 七、自问自答验证

### Q1：详细设计是否足够完整？

**答**：详细设计覆盖了：
- 编排层：3个核心文件的具体结构
- 执行层：4个阶段文件 + 3个收集器 + 3个生成器
- 知识层：6个知识文件 + 2个schema文件
- 审查层：3个校验器文件
- 重构保证：5个组件的详细输入输出格式

### Q2：设计是否存在矛盾？

**答**：检查后发现：
- 知识层从references/拆分，但保留原文件引用，无矛盾
- 执行层调用收集器/生成器，调用关系清晰
- 重构保证流程独立于主流程，无冲突

### Q3：是否有遗漏的设计点？

**答**：补充以下内容：
- processes/目录简化方案：保留evalset-parse.md、keypoint-process.md，其他移至对应层
- 变量速查统一管理：在SKILL.md保留核心变量，各层文件引用
- 错误处理统一：在error-knowledge.md集中定义

### Q4：重构保证流程是否可执行？

**答**：可执行性验证：
- 分析器：基于现有文件结构和依赖关系
- 设计器：有明确的输出格式和验证方法
- 执行器：基于git操作，可回滚
- 校验器：基于文件检查和流程验证

---

## 八、参考资源

- [大纲设计方案](./2026-04-02-model-evaluation-skill-refactor-design.md)
- [Agent Skill 五种设计模式](../../references/Agent-Skill-五种设计模式.md)
- [Writing Skills 规范](../../references/Writing-Skills规范.md)
- [现有 Skill 结构](../../../.claude/skills/model-evaluation/SKILL.md)