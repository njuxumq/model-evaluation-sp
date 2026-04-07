# Harness Design Skill 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建一个通用型 Harness 工程 Skill，为任何任务提供质量保障机制设计指导。

**Architecture:** 基于 5 维度诊断 → 决策表定位 → 动态设计机制的流程，配合 4 个机制方法论文档和诊断示例。

**Tech Stack:** Markdown, YAML frontmatter

---

## 文件结构

```
.claude/skills/harness-design/
├── SKILL.md                              # 主入口文档
└── references/
    ├── mechanisms/
    │   ├── planner.md                    # Planner 设计方法论
    │   ├── evaluator.md                  # Evaluator 设计方法论
    │   ├── sprint-contract.md            # Sprint Contract 设计方法论
    │   └── context-reset.md              # Context Reset 设计方法论
    └── examples/
        └── diagnosis-examples.md         # 诊断过程示例
```

---

## Task 1: 创建 Skill 目录结构

**Files:**
- Create: `.claude/skills/harness-design/`
- Create: `.claude/skills/harness-design/references/mechanisms/`
- Create: `.claude/skills/harness-design/references/examples/`

- [ ] **Step 1: 创建目录结构**

```bash
mkdir -p .claude/skills/harness-design/references/mechanisms
mkdir -p .claude/skills/harness-design/references/examples
```

- [ ] **Step 2: 验证目录创建成功**

```bash
ls -la .claude/skills/harness-design/
ls -la .claude/skills/harness-design/references/
```

Expected: 显示 harness-design 目录及 references 子目录

---

## Task 2: 创建 SKILL.md

**Files:**
- Create: `.claude/skills/harness-design/SKILL.md`

- [ ] **Step 1: 创建 SKILL.md 文件**

```markdown
---
name: harness-design
description: Design quality assurance mechanisms for any task. Use when user invokes '/harness-design', asks for "harness engineering", "Planner", "Evaluator", "Sprint Contract", "Context Reset", or mentions "task reliability", "quality control", "make task controllable" for code development, skill development, refactoring, content generation, or presentation tasks.
---

# Harness Design

为任何任务设计质量保障机制，确保可控、可靠。

---

## 核心流程

```
触发 → 5维度诊断 → 决策表定位 → 动态设计机制 → 输出选择
```

---

## 5维度诊断

快速判断当前任务特征，定位所需机制。

| 维度 | 判断依据 | 等级划分 |
|------|----------|----------|
| **复杂度** | 改动文件数、模块数、依赖关系 | 简单 / 中等 / 复杂 |
| **主观性** | 是否有二元验证标准 vs 需人为判断 | 客观 / 半主观 / 高主观 |
| **验收清晰度** | 用户是否给出可验证的交付标准 | 清晰 / 模糊 / 缺失 |
| **任务时长** | 预计操作步骤数、文件处理量 | 短 / 中 / 长 |
| **迭代预期** | 用户是否要求多方案或暗示需打磨 | 单次 / 需打磨 / 持续 |

**诊断输出**：填写上述表格，得出任务特征画像。

---

## 决策表

根据诊断结果，定位所需机制。

| 机制 | 触发条件 | 详细方法论 |
|------|----------|------------|
| **Planner** | 复杂度 ≥ 中等 | [planner.md](./references/mechanisms/planner.md) |
| **Evaluator** | 主观性 ≥ 半主观 或 任务重要 | [evaluator.md](./references/mechanisms/evaluator.md) |
| **Sprint Contract** | 验收清晰度 ≤ 模糊 | [sprint-contract.md](./references/mechanisms/sprint-contract.md) |
| **Context Reset** | 任务时长 ≥ 长 | [context-reset.md](./references/mechanisms/context-reset.md) |
| **G-E 循环** | 迭代预期 ≥ 需打磨 | 见 evaluator.md |

**重要任务强制规则**：生产环境、安全、资金相关 → 必须启用 Evaluator

---

## 动态设计机制

定位所需机制后，**根据当前具体任务**设计各机制参数：

| 机制 | 设计内容 | 参考 |
|------|----------|------|
| **Planner** | 分解策略、子任务边界、验收标准 | planner.md 方法论 |
| **Evaluator** | 评估维度、评分标准、校准方式 | evaluator.md 方法论 |
| **Contract** | 协商重点、验收项、边界定义 | sprint-contract.md 方法论 |
| **Context Reset** | 触发时机、Artifact 内容 | context-reset.md 方法论 |

**关键**：不预置场景模板，而是在执行时**动态分析**当前任务来设计。

---

## 输出选择

完成设计后，询问用户：

| 形式 | 产出 |
|------|------|
| **仅参考建议** | 设计建议文档 |
| **立即执行** | Claude 进入 Harness 循环执行 |
| **详细咨询** | 深入讨论某个机制 |

---

## 纠正模式

若任务进行中触发，先诊断问题：

| 问题表现 | 原因 | 处理 |
|----------|------|------|
| 输出质量虚高 | 自我评估偏差 → 添加/加强 Evaluator |
| 反复修正同一问题 | 理解偏差 → 添加 Contract 重新协商 |
| 提前结束/收尾倾向 | 上下文焦虑 → 检查是否需要 Context Reset |
| 任务执行混乱 | 缺少分解 → 添加 Planner 重新规划 |

---

## 诊断示例

见 [diagnosis-examples.md](./references/examples/diagnosis-examples.md)
```

- [ ] **Step 2: 验证文件创建成功**

```bash
cat .claude/skills/harness-design/SKILL.md | head -20
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 3: 创建 planner.md

**Files:**
- Create: `.claude/skills/harness-design/references/mechanisms/planner.md`

- [ ] **Step 1: 创建 planner.md 文件**

```markdown
---
name: planner-methodology
description: Use when task complexity requires decomposition, to learn how to design Planner for current task
---

# Planner 设计方法论

## 核心理念

复杂任务先分解再执行，避免一次性处理导致失控。

---

## 如何判断是否需要 Planner

| 判断依据 | 需要 Planner | 不需要 Planner |
|----------|--------------|----------------|
| 改动文件数 | > 3 文件 | ≤ 3 文件 |
| 模块数 | 涉及多模块 | 单模块内 |
| 依赖关系 | 有跨模块依赖 | 无或简单依赖 |
| 任务边界 | 需要分阶段完成 | 一次性完成 |

---

## Planner 设计要点

### 1. 分解粒度

| 原则 | 说明 |
|------|------|
| 每个子任务可独立验收 | 有产出 + 可验证标准 |
| 每个子任务 1-3 小时可完成 | 不过大也不过小 |
| 依赖关系显式标注 | 明确执行顺序 |

### 2. 分解边界

如何划分子任务边界？

| 方法 | 说明 |
|------|------|
| **功能边界** | 每个子任务完成一个独立功能 |
| **模块边界** | 每个子任务涉及一个模块 |
| **时间边界** | 每个子任务可在一个时间段内完成 |
| **验收边界** | 每个子任务有明确的验收标准 |

**选择依据**：根据任务类型选择合适的边界划分方式

### 3. 验收标准定义

每个子任务必须有验收标准。

| 好的验收标准 | 不好的验收标准 |
|--------------|----------------|
| 可验证（能测试/能检查） | 模糊（"做得好一点"） |
| 具体（明确行为或产出） | 抽象（"功能完成"） |
| 可达成（在子任务范围内） | 过大（超出子任务范围） |

---

## Planner 输出结构

根据当前任务设计分解结果：

```markdown
## 任务分解

**总目标**：{当前任务的目标}

**子任务列表**：

| 序号 | 子任务 | 涉及范围 | 依赖 | 产出 | 验收标准 |
|------|--------|----------|------|------|----------|
| ... | ... | ... | ... | ... | ... |

**关键决策点**：{需要先确认的设计决策}
**风险点**：{可能导致偏离的问题}
```

---

## 设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| 高层不指定细节 | 只定义目标和验收 | 过度指定实现方案 |
| 依赖显式标注 | 明确执行顺序 | 隐式假设顺序 |
| 每个子任务独立验收 | 有产出可验证 | 验收依赖后续任务 |
| 边界清晰 | 包含/不包含明确 | 边界模糊导致范围蔓延 |

---

## Common Pitfalls

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| **分解粒度过细** | 子任务过多（> 15 个），执行效率低 | 过度拆分简单任务 | 合并相关子任务，保持 1-3 小时粒度 |
| **分解粒度过粗** | 子任务无法独立验收或耗时过长 | 拆分不足 | 按功能/模块边界进一步拆分 |
| **验收标准模糊** | "功能完成"等不可验证表述 | 未转化为具体行为 | 必须有可测试/可检查的验收项 |
| **依赖关系遗漏** | 执行时发现前置条件未满足 | 未显式标注依赖 | 检查所有子任务的前置条件 |
| **过度指定实现** | Planner 直接指定代码实现细节 | 越权决策 | Planner 只定义目标和验收，实现交给 Generator |

---

## 与其他机制配合

| 配合机制 | Planner 输出如何使用 |
|----------|----------------------|
| **Sprint Contract** | 子任务 → Contract 的 Sprint 目标 |
| **Evaluator** | 子任务的验收标准 → Evaluator 检查清单来源 |
| **Context Reset** | 分解结果 → Artifact 的任务列表 |
```

- [ ] **Step 2: 验证文件创建成功**

```bash
head -30 .claude/skills/harness-design/references/mechanisms/planner.md
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 4: 创建 evaluator.md

**Files:**
- Create: `.claude/skills/harness-design/references/mechanisms/evaluator.md`

- [ ] **Step 1: 创建 evaluator.md 文件**

```markdown
---
name: evaluator-methodology
description: Use when task subjectivity or importance requires independent quality assessment, to learn how to design Evaluator for current task
---

# Evaluator 设计方法论

## 核心理念

生成者与审查者分离，防止自我评估偏差导致质量虚高。

---

## 为什么需要独立 Evaluator

| 问题 | 说明 |
|------|------|
| **自我评估偏差** | Agent 评估自己工作倾向于正面评价 |
| **主观任务尤甚** | 无二元标准时偏差更严重 |
| **Claude 特性** | 天然倾向正面评价，缺乏批判性 |

---

## 如何判断是否需要 Evaluator

| 条件 | 强度 |
|------|------|
| 主观性 = 高主观 | **必须** |
| 任务涉及生产/安全/资金 | **必须** |
| 用户明确表示重要 | **必须** |
| 用户反复修正同一问题 | **必须**（暗示质量不稳定） |
| 主观性 = 半主观 | 推荐 |
| 任务时长 > 1 小时 | 推荐 |

---

## Evaluator 设计三要素

根据当前任务，设计以下三要素：

### 1. 评估维度

将模糊判断转化为可评分维度。

**转化方法**：
- 从"好不好"的模糊问题 → 拆解为具体维度
- 每个维度独立评分（1-5 或通过/不通过）

**转化示例**：

| 模糊问题 | 转化为维度 |
|----------|------------|
| "代码质量好吗？" | → 可读性、可维护性、测试覆盖、性能、安全性 |
| "文档清晰吗？" | → 结构完整性、逻辑连贯、语言精准 |
| "设计好看吗？" | → 整体感、原创性、技术执行、可用性 |

---

### 2. 评分标准

为每个维度定义评分等级。

**设计要点**：
- 每个等级有具体表现描述
- 至少一个维度有硬阈值（低于某等级强制失败）
- 权重偏向 Claude 默认表现弱的维度

**评分模板**：

| 等级 | 标准 | 具体表现 |
|------|------|----------|
| 5 | 优秀 | {描述} |
| 4 | 良好 | {描述} |
| 3 | 合格 | {描述} |
| 2 | 需改进 | {描述} |
| 1 | 不合格 | {描述} |

---

### 3. 校准机制

防止评分漂移。

| 方法 | 说明 |
|------|------|
| **Few-shot 示例** | 提供带评分的案例供参照 |
| **检查清单** | 具体检查项替代模糊判断 |
| **硬阈值** | 低于某等级强制失败 |

---

## G-E 循环设计

### 循环流程

```
Generator 执行 → Evaluator 评估 → 判断 → Generator 改进 → 循环
                      │
                      ├─ 通过 → 继续下一任务
                      ├─ 不通过 → 返回 Generator
                      └─ 边缘 → 用户确认
```

### 循环参数

| 参数 | 设计依据 |
|------|----------|
| 最大轮次 | 根据迭代预期设定（默认 5） |
| 退出条件 | 所有维度 ≥ 硬阈值 |
| 策略转向 | 连续无进展时可转向新方案 |

---

## Evaluator 输出结构

```markdown
## 评估报告

**维度评分**：

| 维度 | 分数 | 等级 | 具体问题 |
|------|------|------|----------|

**硬阈值检查**：{通过/不通过}

**总评**：{通过/不通过}

**问题清单**（按优先级）：

| 序号 | 问题 | 改进建议 |
|------|------|----------|

**下一步**：{继续/返回 Generator/用户确认}
```

---

## 设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| 维度可评分 | 每个维度有具体标准 | "整体效果"不可评分 |
| 有硬阈值 | 至少一个维度强制失败条件 | 全软评分易妥协 |
| 反馈可执行 | 每个问题配改进建议 | "整体不错"无改进方向 |
| 有校准 | few-shot 或检查清单 | 无参照易评分漂移 |

---

## Common Pitfalls

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| **评分过宽容** | 明显问题仍获高分 | Claude 倾向正面评价 | 添加 skeptical 角度的 prompt，强调批判性 |
| **维度过于抽象** | "整体效果"不可评分 | 未转化为具体特征 | 每个维度必须有可观察的具体指标 |
| **缺少硬阈值** | 所有维度都软评分，易妥协 | 未设置强制失败条件 | 至少一个维度有硬阈值，低于则强制失败 |
| **反馈不可执行** | "整体不错"无改进方向 | 未给出具体建议 | 每个问题必须配具体可执行的改进建议 |
| **评分漂移** | 相似产出评分不一致 | 缺少校准参照 | 添加 few-shot 示例或检查清单 |
| **过早批准** | 未发现明显 bug 就通过 | 测试 superficially | 强调 probing edge cases，添加深度检查要求 |

---

## Evaluator 调优

若 Evaluator 过早批准或评分不准：

| 偏差表现 | 调优方法 |
|----------|----------|
| 说服自己通过 | 添加反借口 prompt |
| 测试 superficially | 添加 probing edge cases 要求 |
| 评分漂移 | 添加 few-shot 示例 |
| 过于正面 | 调整 prompt 强调 skeptical 角度 |
```

- [ ] **Step 2: 验证文件创建成功**

```bash
head -30 .claude/skills/harness-design/references/mechanisms/evaluator.md
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 5: 创建 sprint-contract.md

**Files:**
- Create: `.claude/skills/harness-design/references/mechanisms/sprint-contract.md`

- [ ] **Step 1: 创建 sprint-contract.md 文件**

```markdown
---
name: sprint-contract-methodology
description: Use when acceptance criteria is unclear, to learn how to negotiate "definition of done" before execution
---

# Sprint Contract 设计方法论

## 核心理念

执行前先协商"完成定义"，防止理解偏差导致无效迭代。

---

## 为什么需要 Contract

| 问题 | 说明 |
|------|------|
| **理解偏差** | Generator 和用户对"完成"理解不一致 |
| **模糊需求** | "好看"、"好用"无明确标准 |
| **无效迭代** | 按自己理解完成后用户不满意 |

---

## 如何判断是否需要 Contract

| 条件 | 说明 |
|------|------|
| 验收清晰度 = 模糊 | 用户需求如"好看"、"好用"等 |
| 验收清晰度 = 缺失 | 用户只说"帮我做个 X"无具体要求 |
| 任务重要 | 涉及生产/安全/资金等 |
| 用户反复修正 | 暗示此前理解有偏差 |

---

## Contract 设计要点

### 1. 验收标准设计

**核心原则**：可验证

| 好的验收标准 | 不好的验收标准 |
|--------------|----------------|
| 可测试/可检查 | 模糊描述 |
| 具体行为或产出 | 抽象概念 |
| 有验证方式 | 无法验证 |

**转化方法**：

| 模糊表述 | 转化为验收标准 |
|----------|----------------|
| "功能完成" | → 用户可点击 X 并收到 Y 结果 |
| "用户体验好" | → 用户在 3 步内可完成核心操作 |
| "代码质量高" | → 无 lint error + 核心函数有注释 |

---

### 2. 边界定义

明确包含和不包含范围。

| 作用 | 说明 |
|------|------|
| 防止范围蔓延 | 明确不在本次范围内 |
| 避免理解歧义 | 双方对边界有一致认知 |

---

### 3. 协商流程

```
Generator 提议 → Evaluator/用户审核 → 判断 → 达成一致
      │                │              │
      ├─ 构建内容      ├─ 是否正确    ├─ 无异议 → 达成
      ├─ 验收标准      ├─ 是否完整    └─ 有异议 → Generator 调整
      └─ 边界定义      └─ 是否可行
```

---

## Contract 输出结构

根据当前任务设计 Contract：

```markdown
## Sprint Contract

**目标**：{本次要完成的目标}

**构建内容**：
| 序号 | 内容 | 方案摘要 |
|------|------|----------|

**验收标准**（必须全部满足）：
| 序号 | 验收项 | 验证方式 |
|------|--------|----------|

**不包含范围**：
- {明确排除的内容}

**风险点**：{可能导致偏离的问题}
```

---

## 设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| 验收可验证 | 能测试/能检查 | "功能完善"不可验证 |
| 边界明确 | 包含/不包含清晰 | 边界模糊导致范围蔓延 |
| 保持精简 | 核心验收项 ≤ 5 | Contract 过于复杂 |
| 协商后再执行 | 未达成不可执行 | Generator 直接执行 |

---

## Common Pitfalls

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| **验收标准模糊** | "功能完成"等不可验证表述 | 未转化为具体行为 | 必须有可测试/可检查的验收项和验证方式 |
| **边界不明确** | 执行中范围蔓延 | 未明确不包含什么 | 必须列出"不包含范围" |
| **Contract 过于复杂** | 验收项过多（> 7 个） | 过度细化 | 精简为核心验收项 ≤ 5 |
| **跳过协商直接执行** | Generator 未等用户确认就开始 | 忽略协商流程 | 强制：未达成一致不可执行 |
| **Contract 不更新** | 需求变更后仍按旧 Contract 执行 | 未及时重新协商 | 发现需求变化立即重新协商 |

---

## 与其他机制配合

| 配合机制 | Contract 的作用 |
|----------|------------------|
| **Planner** | Planner 子任务 → Contract 目标来源 |
| **Evaluator** | Contract 验收标准 → Evaluator 检查清单 |
| **Generator** | Contract → Generator 执行的约束边界 |
```

- [ ] **Step 2: 验证文件创建成功**

```bash
head -30 .claude/skills/harness-design/references/mechanisms/sprint-contract.md
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 6: 创建 context-reset.md

**Files:**
- Create: `.claude/skills/harness-design/references/mechanisms/context-reset.md`

- [ ] **Step 1: 创建 context-reset.md 文件**

```markdown
---
name: context-reset-methodology
description: Use when task is expected to run long, to learn how to design context reset and state transfer
---

# Context Reset 设计方法论

## 核心理念

清空上下文窗口，通过 Artifact 传递状态，防止上下文焦虑导致过早收尾。

---

## 为什么需要 Context Reset

| 问题 | 说明 |
|------|------|
| **上下文填充** | 长任务中 context window 填满，失去连贯性 |
| **上下文焦虑** | 接近上下文限制时过早收尾（"差不多了"） |
| **Compaction 局限** | 原地压缩不提供"干净起点" |

---

## Compaction vs Context Reset

| 方式 | 操作 | 效果 |
|------|------|------|
| **Compaction** | 原地压缩历史为摘要 | 保持同一 agent，焦虑可能残留 |
| **Context Reset** | 清空 context，启动新 agent | 干净起点，需 Artifact 传递状态 |

**选择依据**：超长任务（几小时）仍需 Context Reset

---

## 如何判断是否需要 Context Reset

| 条件 | 说明 |
|------|------|
| 任务时长 = 长 | 预计 > 50 步操作或几小时 |
| 上下文填充度 > 70% | Claude 出现收尾倾向 |
| Generator 提前结束 | "差不多了，剩下的你自己处理" |
| 跨 session 执行 | 用户中断后恢复 |

---

## Context Reset 设计要点

### 1. 触发时机

| 时机判断 | 说明 |
|----------|------|
| 填充度检测 | 观察是否接近上下文限制 |
| 收尾倾向 | Claude 开始说"差不多完成了" |
| 预估时长 | 任务开始前预估是否需要 Reset |

---

### 2. Artifact 内容设计

Artifact 必须让新 Agent 无历史 context 也能理解：

| 必需内容 | 说明 |
|----------|------|
| **任务背景** | 为什么做（目标） |
| **当前状态** | 做到哪里（进度） |
| **下一步指引** | 怎么做（具体操作） |
| **关键决策** | 已做决策（避免重复决策） |
| **注意事项** | 执行过程中的注意点 |

---

### 3. 验证衔接点

新 Agent 启动后必须验证：

| 验证内容 | 说明 |
|----------|------|
| 已完成产出存在 | 文件确实存在 |
| 内容符合预期 | 产出质量可用 |
| 无断裂遗漏 | 可正常继续 |

---

## Artifact 输出结构

根据当前任务设计 Artifact：

```markdown
# Harness State Artifact

**生成时间**：{timestamp}
**任务目标**：{当前任务的目标}

## 执行进度

**当前阶段**：{阶段名}

**已完成**：
| 序号 | 任务 | 产出 | 状态 |

**待完成**：
| 序号 | 任务 | 依赖 | 预计产出 |

## 关键决策

| 决策点 | 决策内容 | 原因 |

## 下一步

**正在处理**：{当前任务摘要}
**下一步操作**：{具体指引}

## 注意事项

- {执行过程中的特殊注意点}
```

---

## 设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| Artifact 自描述 | 新 Agent 无历史 context 也能理解 | 信息不足无法接续 |
| 验证衔接点 | 新 Agent 验证已完成产出 | 假设产出存在 |
| Harness 连续性 | Reset 后继续相同机制 | Reset 后跳过 Evaluator |
| 决策已记录 | 关键决策记录在 Artifact | 新 Agent 可能重做决策 |

---

## Common Pitfalls

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| **Artifact 信息不足** | 新 Agent 无法理解上下文 | 缺少背景/状态/指引 | Artifact 必须包含：任务背景、当前状态、下一步指引 |
| **未验证衔接** | 假设已完成产出存在 | 跳过验证步骤 | 新 Agent 启动后必须验证文件存在和内容 |
| **Harness 中断** | Reset 后跳过 Evaluator/Contract | 未在 Artifact 中要求 | 明确记录需要继续的 Harness 机制 |
| **决策未记录** | 新 Agent 重复做决策 | 关键决策未写入 Artifact | 所有设计决策必须记录 |
| **Reset 时机错误** | 过早或过晚 Reset | 未监控上下文状态 | 填充度 > 70% 或出现收尾倾向时触发 |
| **Artifact 过大** | Token 消耗过多 | 记录过多细节 | 保持精简，只记录关键信息 |

---

## 与其他机制配合

| 配合机制 | Artifact 来源 |
|----------|---------------|
| **Planner** | Planner 分解结果 → 任务列表 |
| **Contract** | Contract → 验收标准 |
| **Evaluator** | Artifact 明确要求继续 Evaluator |
```

- [ ] **Step 2: 验证文件创建成功**

```bash
head -30 .claude/skills/harness-design/references/mechanisms/context-reset.md
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 7: 创建 diagnosis-examples.md

**Files:**
- Create: `.claude/skills/harness-design/references/examples/diagnosis-examples.md`

- [ ] **Step 1: 创建 diagnosis-examples.md 文件**

```markdown
---
name: diagnosis-examples
description: Use when learning how to apply 5-dimension diagnosis to different tasks
---

# 诊断示例

展示如何对不同任务进行 5 维度诊断。注意：这些是**诊断过程示例**，不是场景模板。

---

## 示例1：修改单个函数

**用户需求**：修复 login 函数的 bug

**诊断过程**：

| 维度 | 分析 | 结论 |
|------|------|------|
| **复杂度** | 改动 1 个文件内 1 个函数 | 简单 |
| **主观性** | bug 修复有明确结果（修复/未修复） | 客观 |
| **验收清晰度** | 用户给出具体 bug 描述 | 清晰 |
| **任务时长** | 几分钟可完成 | 短 |
| **迭代预期** | 一次修复即可 | 单次 |

**诊断结果**：简单 + 客观 + 清晰 + 短 + 单次

**所需机制**：无 Harness，直接执行 + 自动测试验证

---

## 示例2：重构模块代码

**用户需求**：重构用户认证模块，提升代码可维护性

**诊断过程**：

| 维度 | 分析 | 结论 |
|------|------|------|
| **复杂度** | 改动多个文件，拆分服务 | 中等 |
| **主观性** | 有测试标准但代码质量需判断 | 半主观 |
| **验收清晰度** | "提升可维护性"需具体化 | 模糊 |
| **任务时长** | 预计几十分钟 | 中 |
| **迭代预期** | 可能需要多轮调整 | 需打磨 |

**诊断结果**：中等 + 半主观 + 模糊 + 中 + 需打磨

**所需机制**：Planner + Evaluator + Contract + G-E 循环

---

## 示例3：制作 PPT

**用户需求**：帮我做一个产品发布演示 PPT

**诊断过程**：

| 维度 | 分析 | 结论 |
|------|------|------|
| **复杂度** | 单一产物但内容较多 | 中等 |
| **主观性** | 视觉效果依赖品味判断 | 高主观 |
| **验收清晰度** | 用户未给出具体要求 | 缺失 |
| **任务时长** | 预计几十分钟 | 中 |
| **迭代预期** | 用户暗示"做得好看"需打磨 | 需打磨 |

**诊断结果**：中等 + 高主观 + 缺失 + 中 + 需打磨

**所需机制**：Planner（生成结构） + Evaluator（视觉检查） + Contract（协商标准） + G-E 循环

---

## 示例4：构建 Web 应用

**用户需求**：构建一个任务管理应用

**诊断过程**：

| 维度 | 分析 | 结论 |
|------|------|------|
| **复杂度** | 多文件、前后端、完整系统 | 复杂 |
| **主观性** | 有测试但 UI/UX 需判断 | 半主观 |
| **验收清晰度** | 用户只说"任务管理"无细节 | 缺失 |
| **任务时长** | 预计几小时 | 长 |
| **迭代预期** | 功能需反复测试调整 | 需打磨 |

**诊断结果**：复杂 + 半主观 + 缺失 + 长 + 需打磨

**所需机制**：Planner + Sprint + Contract + Evaluator + Context Reset + G-E 循环

---

## 示例5：写技术文档

**用户需求**：写一份 API 接口说明文档

**诊断过程**：

| 维度 | 分析 | 结论 |
|------|------|------|
| **复杂度** | 单一文档 | 简单 |
| **主观性** | 文档清晰度需判断 | 高主观 |
| **验收清晰度** | 用户未说明具体要求 | 模糊 |
| **任务时长** | 预计几十分钟 | 中 |
| **迭代预期** | 可能需要多轮调整 | 需打磨 |

**诊断结果**：简单 + 高主观 + 模糊 + 中 + 需打磨

**所需机制**：Evaluator（内容质量检查） + Contract（协商文档结构） + G-E 循环

---

## 关键洞察

从以上示例可以看出：

| 维度组合 | 决定机制组合 | 原因 |
|----------|--------------|------|
| 简单 + 客观 + 清晰 | 无 Harness | 任务简单直接可完成 |
| 高主观 | **必须 Evaluator** | 自我评估偏差严重 |
| 模糊/缺失 | **必须 Contract** | 需协商"完成定义" |
| 复杂 | **必须 Planner** | 需分解才能可控 |
| 长 | **必须 Context Reset** | 防止上下文焦虑 |
| 需打磨 | **必须 G-E 循环** | 需多轮迭代改进 |

**核心原则**：根据当前任务的具体特征动态判断，不预设场景模板。
```

- [ ] **Step 2: 验证文件创建成功**

```bash
head -30 .claude/skills/harness-design/references/examples/diagnosis-examples.md
```

Expected: 显示 YAML frontmatter 和标题

---

## Task 8: 验证 Skill 完整性

**Files:**
- Verify: `.claude/skills/harness-design/`

- [ ] **Step 1: 检查目录结构完整性**

```bash
find .claude/skills/harness-design -type f | sort
```

Expected:
```
.claude/skills/harness-design/SKILL.md
.claude/skills/harness-design/references/examples/diagnosis-examples.md
.claude/skills/harness-design/references/mechanisms/context-reset.md
.claude/skills/harness-design/references/mechanisms/evaluator.md
.claude/skills/harness-design/references/mechanisms/planner.md
.claude/skills/harness-design/references/mechanisms/sprint-contract.md
```

- [ ] **Step 2: 验证 SKILL.md YAML frontmatter 格式**

```bash
head -5 .claude/skills/harness-design/SKILL.md
```

Expected:
```
---
name: harness-design
description: Design quality assurance mechanisms for any task...
---
```

- [ ] **Step 3: 验证所有引用路径正确**

检查 SKILL.md 中的链接：
- `[planner.md](./references/mechanisms/planner.md)` → 文件存在
- `[evaluator.md](./references/mechanisms/evaluator.md)` → 文件存在
- `[sprint-contract.md](./references/mechanisms/sprint-contract.md)` → 文件存在
- `[context-reset.md](./references/mechanisms/context-reset.md)` → 文件存在
- `[diagnosis-examples.md](./references/examples/diagnosis-examples.md)` → 文件存在

```bash
ls -la .claude/skills/harness-design/references/mechanisms/
ls -la .claude/skills/harness-design/references/examples/
```

Expected: 所有引用的文件都存在

---

## Task 9: 提交到 git

**Files:**
- Commit: `.claude/skills/harness-design/`

- [ ] **Step 1: 检查 git 状态**

```bash
git status
```

Expected: 显示 .claude/skills/harness-design/ 下所有文件为 untracked

- [ ] **Step 2: 添加文件到暂存区**

```bash
git add .claude/skills/harness-design/
```

- [ ] **Step 3: 提交**

```bash
git commit -m "$(cat <<'EOF'
feat: 添加 harness-design Skill

通用型 Harness 工程 Skill，为任何任务提供质量保障机制设计指导。

核心特性：
- 5维度诊断：复杂度、主观性、验收清晰度、任务时长、迭代预期
- 决策表定位：根据诊断结果动态选择所需机制
- 4个机制方法论：Planner、Evaluator、Sprint Contract、Context Reset
- 诊断示例：展示如何动态分析不同任务

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 4: 验证提交成功**

```bash
git log --oneline -1
```

Expected: 显示最新提交信息

---

## Self-Review

| 检查项 | 状态 |
|--------|------|
| 规范覆盖完整 | ✓ 所有设计规范内容已实现 |
| 无占位符 | ✓ 所有文件内容完整 |
| 路径一致 | ✓ 所有引用路径正确 |
| YAML 格式正确 | ✓ 所有 frontmatter 格式正确 |