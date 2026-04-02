# Skill 重构方案大纲

## 一、现有问题分析

### 1.1 流程僵硬的具体表现

| 问题 | 具体表现 | 根因 |
|------|----------|------|
| 阶段强制顺序 | 必须从初始化→构建→评测集→执行，无法灵活跳转 | SKILL.md 定义了严格阶段顺序 |
| 步骤不可跳过 | 每个任务内的步骤标注"不得跳过"，即使用户已有信息也无法跳过 | 流水线模式过度使用 |
| 判断逻辑固化 | 每个步骤的判断分支固定写在文档中，无法适应新场景 | 缺乏动态判断机制 |
| Red Flags 过多 | 大量禁止规则限制了 Agent 的灵活适应能力 | 过度约束导致僵化 |

### 1.2 交互不友好的具体表现

| 问题 | 具体表现 | 根因 |
|------|----------|------|
| 信息量过大 | 每次进入阶段就展示完整流程概览和任务列表 | 递进式披露不足 |
| 打断用户意图 | 用户说"开始新评测"却被引导从初始化开始，不能直接进入核心任务 | 流程感知机制过于机械 |
| 确认点过多 | 每个步骤都要用户确认，即使内容显而易见 | 反转模式过度使用 |
| 选项不够清晰 | 询问用户时选项描述不够具体，用户难以理解 | 交互设计缺乏用户视角 |

### 1.3 职责隔离不足的具体表现

| 问题 | 具体表现 | 根因 |
|------|----------|------|
| SKILL.md 职责过重 | 包含流程概览、交互规范、Red Flags、目录结构、变量速查 | 主文件承载过多内容 |
| 阶段文档职责混合 | eval-build.md 同时包含流程定义、判断逻辑、错误处理 | 流程与规范混合 |
| 子流程职责不清 | processes/ 文件有的定义流程，有的定义规范 | 缺乏明确的角色划分 |
| references/ 混杂 | 同时包含接口说明、模板说明、规范、转化规则 | 知识与规范未分离 |

### 1.4 当前设计模式使用评估

| 设计模式 | 当前使用情况 | 评估 |
|----------|--------------|------|
| 流水线 | 四阶段主流程、子流程都采用流水线结构 | **过度使用**：导致流程僵硬 |
| 反转 | 每个步骤都需要用户确认 | **过度使用**：导致交互繁琐 |
| 工具封装 | references/ 存放接口说明、脚本定义 | **使用不足**：知识未有效封装 |
| 审查器 | Red Flags 作为审查规则 | **使用不当**：审查规则散落各处 |
| 生成器 | evalset-create.md 使用生成模式 | **使用恰当**：生成问题集流程合理 |

---

## 二、重构目标

**核心目标**：将"僵硬流水线"重构为"灵活编排"，在保证流程完整性的前提下：
- 提升交互友好性（递进式披露、适度确认）
- 强化职责隔离（知识层、执行层、编排层分离）

---

## 三、核心策略：四层架构

采用**"工具封装为底 + 生成器为中 + 流水线为顶 + 反转适度"**的组合策略：

```
┌─────────────────────────────────────────────────────────────┐
│                    重构后架构层次                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 顶层：编排层（流水线模式）                              │  │
│  │ ├─ orchestrator.md - 流程编排器                        │  │
│  │ ├─ checkpoints.md - 检查点定义                         │  │
│  │ └─ transitions.md - 状态转换规则                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 中层：执行层（生成器 + 反转适度）                       │  │
│  │ ├─ phases/ - 四阶段执行单元                            │  │
│  │ ├─ generators/ - 生成器组件                            │  │
│  │ └─ collectors/ - 信息收集器                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 底层：知识层（工具封装模式）                            │  │
│  │ ├─ knowledge/ - 知识封装                                │  │
│  │ ├─ schemas/ - 数据格式定义                             │  │
│  │ └─ validators/ - 审查器组件                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 支撑：资源层（保持不变）                                │  │
│  │ ├─ assets/ - 静态资源                                  │  │
│  │ ├─ scripts/ - 可执行脚本                               │  │
│  │ └─ cfg/ - 配置文件                                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、各层职责定义

| 层次 | 职责 | 设计模式 | 核心文件 |
|------|------|----------|----------|
| **编排层** | 流程调度、状态管理、跳转控制、意图识别 | 流水线 | orchestrator.md, checkpoints.md, transitions.md |
| **执行层** | 阶段执行、信息收集、内容生成 | 反转+生成器 | phases/, collectors/, generators/ |
| **知识层** | API规范、脚本定义、模板说明、错误处理、数据格式 | 工具封装 | knowledge/, schemas/ |
| **审查层** | 禁止规则、配置校验、格式校验 | 审查器 | validators/ |

---

## 五、设计模式应用规则

| 设计模式 | 使用位置 | 使用原则 | 避免问题 |
|----------|----------|----------|----------|
| **流水线** | 编排层 | 定义检查点和门控条件 | 不在执行步骤中强制顺序 |
| **反转** | collectors/ | 只收集无法自动推断的关键信息 | 不过度确认，不每步确认 |
| **生成器** | generators/ | 模板驱动生成结构化内容 | 保持模板引用清晰 |
| **工具封装** | knowledge/ | 按需加载知识模块 | 不在执行层嵌入知识 |
| **审查器** | validators/ | 集中存放审查规则清单 | 不散落各处 |

---

## 六、文件拆分方案

### SKILL.md 精简（187行 → ~80行）

```yaml
---
name: model-evaluation
description: 当用户提到'模型评测'/'评测模型'/'评估模型'/'测试模型效果'，或'查看评测进度'/'查看评测结果'，或说'继续评测'/'恢复评测'/'重构评测流程'时使用。
---

# 模型评测技能

## 核心能力

协助用户完成 AI 模型评测任务的全流程管理。

## 流程编排

加载 [orchestrator.md](./orchestrator.md) 获取完整流程编排规则。

## 知识封装

按需加载以下知识模块：
- API 规范：[knowledge/api-knowledge.md](./knowledge/api-knowledge.md)
- 脚本定义：[knowledge/script-knowledge.md](./knowledge/script-knowledge.md)
- 模板说明：[knowledge/template-knowledge.md](./knowledge/template-knowledge.md)
- 错误处理：[knowledge/error-knowledge.md](./knowledge/error-knowledge.md)

## 审查规则

加载 [validators/red-flags.md](./validators/red-flags.md) 获取禁止规则清单。

## 特殊流程

| 流程 | 文档 | 触发条件 |
|------|------|----------|
| 重构保证 | [refactor-guarantee.md](./refactor-guarantee.md) | 用户请求重构 Skill |
```

### 新增编排层文件

| 文件 | 职责 | 内容结构 |
|------|------|----------|
| orchestrator.md | 流程编排、阶段调度 | 四阶段入口、状态管理、意图识别 |
| checkpoints.md | 检查点定义 | 各阶段前置条件、完成标志 |
| transitions.md | 状态转换规则 | 正常流转、跳转规则、异常处理 |

### 重构阶段执行单元

| 原文件 | 重构后 | 职责变化 |
|--------|--------|----------|
| eval-init.md | phases/init-phase.md | 精简为执行单元，环境检测逻辑移至 knowledge/ |
| eval-build.md | phases/build-phase.md | 精简为执行单元，场景收集移至 collectors/ |
| eval-set.md | phases/evalset-phase.md | 精简为执行单元，生成逻辑移至 generators/ |
| eval-execute.md | phases/execute-phase.md | 精简为执行单元 |

### 新增信息收集器（反转模式）

| 文件 | 职责 | 收集时机 |
|------|------|----------|
| collectors/scene-collector.md | 收集评测场景信息 | 构建阶段开始，无法自动推断时 |
| collectors/dimension-collector.md | 收集评测维度偏好 | 场景确认后，需用户指定时 |
| collectors/evalset-collector.md | 收集评测集来源信息 | 评测集阶段开始，来源不明时 |

**反转模式设计原则**：
- 每个收集器只收集 2-3 个关键信息
- 信息可自动推断时跳过收集
- 使用 AskUserQuestion 批量收集多问题
- 收集完成后立即确认，不等待后续

### 新增生成器组件

| 文件 | 职责 | 模板来源 |
|------|------|----------|
| generators/dimension-generator.md | 生成评测维度配置 | assets/dimensions/ |
| generators/evalset-generator.md | 生成问题集 | 根据场景和维度 |
| generators/keypoint-generator.md | 生成评测点 | 根据问题+参考答案 |

### 新增知识封装文件

| 文件 | 职责 | 加载时机 |
|------|------|----------|
| knowledge/api-knowledge.md | 评测服务 API 规范 | 调用 API 时 |
| knowledge/script-knowledge.md | 脚本定义和用法 | 执行脚本时 |
| knowledge/template-knowledge.md | 内置模板说明 | 匹配模板时 |
| knowledge/dimension-knowledge.md | 评测维度说明 | 配置维度时 |
| knowledge/error-knowledge.md | 错误码和错误处理 | 错误发生时 |

### 新增审查器文件

| 文件 | 职责 | 审查时机 |
|------|------|----------|
| validators/red-flags.md | 禁止规则清单 | 每次行动前 |
| validators/config-validator.md | 配置完整性检查 | 提交任务前 |
| validators/evalset-validator.md | 评测集格式检查 | 上传评测集前 |
| validators/task-validator.md | 任务状态检查 | 展示结果前 |

### 新增数据格式定义

| 文件 | 职责 | 用途 |
|------|------|------|
| schemas/evalset-schema.md | 评测集格式定义 | 校验评测集格式 |
| schemas/dimension-schema.md | 维度配置格式定义 | 校验维度配置 |
| schemas/task-schema.md | 任务结果格式定义 | 校验任务结果 |

---

## 七、目录结构对比

```
重构前:                           重构后:
model-evaluation/                 model-evaluation/
├── SKILL.md (187行)              ├── SKILL.md (80行)          ← 精简
├── eval-init.md                  ├── orchestrator.md          ← 新增
├── eval-build.md                 ├── checkpoints.md           ← 新增
├── eval-set.md                   ├── transitions.md           ← 新增
├── eval-execute.md               ├── refactor-guarantee.md    ← 新增
├── processes/                    ├── phases/                  ← 重构
│   ├── evalset-parse.md          │   ├── init-phase.md
│   ├── evalset-create.md         │   ├── build-phase.md
│   ├── dimension-general.md      │   ├── evalset-phase.md
│   ├── dimension-case-level.md   │   └── execute-phase.md
│   ├── keypoint-process.md       ├── collectors/              ← 新增
│   ├── evalset-field-mapping.md  │   ├── scene-collector.md
│   ├── evalset-field-validation  │   ├── dimension-collector.md
│   ├── evalset-model-selection   │   └── evalset-collector.md
│   ├── evalset-supplement.md     ├── generators/              ← 新增
│   └── python-env-process.md     │   ├── dimension-generator.md
├── references/                   │   ├── evalset-generator.md
│   ├── 评测服务接口说明.md        │   └── keypoint-generator.md
│   ├── 认证服务接口说明.md        ├── knowledge/              ← 新增
│   ├── 脚本定义.md                │   ├── api-knowledge.md
│   ├── 中间产物说明.md            │   ├── script-knowledge.md
│   ├── 内置模板说明.md            │   ├── template-knowledge.md
│   ├── 评测维度说明.md            │   ├── dimension-knowledge.md
│   ├── 维度配置转化规则.md        │   └── error-knowledge.md
├── assets/                       ├── validators/              ← 新增
├── scripts/                      │   ├── red-flags.md
                                  │   ├── config-validator.md
                                  │   ├── evalset-validator.md
                                  │   └── task-validator.md
                                  ├── schemas/                ← 新增
                                  │   ├── evalset-schema.md
                                  │   ├── dimension-schema.md
                                  │   └── task-schema.md
                                  ├── assets/                 ← 保持
                                  ├── scripts/                ← 保持
                                  └── processes/              ← 简化
                                      ├── evalset-parse.md
                                      ├── keypoint-process.md
                                      └── python-env-process.md
```

---

## 八、交互改进方案

| 改进点 | 改进方案 | 效果 |
|--------|----------|------|
| 递进式披露 | SKILL.md 精简，首次只展示概览，按需加载 knowledge/ | 减少初始信息量 |
| 流程灵活跳转 | transitions.md 定义意图识别规则，用户明确意图可直接跳转 | 不打断用户意图 |
| 适度确认 | collectors/ 只收集无法推断的信息，可推断则跳过 | 减少确认次数 |
| 批量收集 | 使用 AskUserQuestion 多问题并行收集 | 提升效率 |
| 清晰选项 | 使用 preview 展示选项内容 | 提升可理解性 |

---

## 九、职责隔离改进方案

| 改进点 | 改进方案 | 效果 |
|--------|----------|------|
| SKILL.md 精简 | 只保留能力声明和知识引用 | 主文件职责单一 |
| 阶段执行单元化 | phases/ 只定义执行步骤，不包含知识 | 执行层职责清晰 |
| 知识独立封装 | knowledge/ 独立存放，按需加载 | 知识层职责明确 |
| 审查规则集中 | validators/ 集中存放审查规则 | 审查层职责集中 |

---

## 十、预期效果

### 流程灵活性提升

| 指标 | 重构前 | 重构后 | 改进 |
|------|--------|--------|------|
| 用户意图跳转支持 | 不支持 | 支持 | 用户可直接进入指定阶段 |
| 步骤跳过支持 | 不支持 | 支持（可推断时） | 减少不必要的步骤 |
| 判断逻辑动态性 | 固定 | 可扩展 | 适应新场景 |

### 交互友好性提升

| 指标 | 重构前 | 重构后 | 改进 |
|------|--------|--------|------|
| SKILL.md 行数 | 187行 | ~80行 | 减少初始信息量 |
| 确认次数 | 每步确认 | 关键信息确认 | 减少确认次数 |
| 选项清晰度 | 文字描述 | preview 展示 | 提升可理解性 |

### 职责隔离提升

| 指标 | 重构前 | 重构后 | 改进 |
|------|--------|--------|------|
| 主文件职责 | 流程+规范+知识 | 能力声明+引用 | 职责单一 |
| 阶段文档职责 | 流程+知识混合 | 执行单元 | 职责清晰 |
| 知识存放 | references/ 混杂 | knowledge/ 独立 | 知识封装 |

### 可维护性提升

| 指标 | 重构前 | 重构后 | 改进 |
|------|--------|--------|------|
| 文件职责边界 | 模糊 | 清晰 | 易于定位修改点 |
| 知识更新影响 | 散落多处 | 集中 knowledge/ | 更新范围可控 |
| 新流程扩展 | 需修改多处 | 新增 phases/ | 扩展点明确 |

---

## 参考资源

- [Agent Skill 五种设计模式](../../references/Agent-Skill-五种设计模式.md)
- [Writing Skills 规范](../../references/Writing-Skills规范.md)
- [Claude Skills 完全构建指南](../../references/Claude-Skills-完全构建指南.md)
- [现有 Skill 结构](../../../.claude/skills/model-evaluation/SKILL.md)