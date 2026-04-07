---
name: Harness Design 镅读报告
description: Anthropic 博客「Harness design for long-running application development」阅读报告
---

# Harness Design for Long-running Application Development

> **阅读报告**
> 博客来源：https://www.anthropic.com/engineering/harness-design-long-running-apps

## 文章元信息

| 项目 | 内容 |
|------|------|
| **标题** | Harness design for long-running application development |
| **作者** | Prithvi Rajasekaran (Labs team) |
| **发布日期** | 2026-03-24 |
| **核心主题** | Harness 设计在 AI Agent 长时间自主编程任务中的应用 |

---

## 核心问题与动机

文章探讨两个相互关联的问题：

1. **前端设计质量**：让 Claude 产出高质量的前端设计
2. **长时间自主编程**：让 Agent 无需人工干预构建完整应用

**背景**：之前的工作（frontend design skill 和 long-running coding agent harness）通过 prompt engineering 和 harness 设计提升了性能，但都遇到了瓶颈。

---

## 关键洞察：失败模式分析

### 问题一：上下文窗口填充导致失去连贯性

| 现象 | 说明 |
|------|------|
| **上下文焦虑** | 模型在接近上下文限制时过早收尾工作 |
| **解决方案** | **上下文重置**（Context Reset）—— 清空上下文窗口，启动新 Agent，通过结构化 artifacts 传递状态 |
| **与 Compaction 的区别** | Compaction 是原地压缩，不提供"干净起点"，上下文焦虑仍可能存在 |

> Claude Sonnet 4.5 的上下文焦虑足够强，仅靠 compaction 无法支持长任务，必须使用上下文重置。

### 问题二：自我评估偏差

| 现象 | 说明 |
|------|------|
| **过度乐观** | Agent 评估自己工作时，倾向于自信赞扬——即使质量明显平庸 |
| **主观任务尤甚** | 设计类任务无二元验证标准（如软件测试），判断"布局是否精致"时 Agent 严重偏正向 |
| **解决思路** | **生成器与评估器分离**——让一个 Agent 做工作，另一个 Agent 判断质量 |

---

## 核心方法：GAN 启发的多 Agent 架构

### 灵感来源

从 **Generative Adversarial Networks (GANs)** 中获得启发，设计多 Agent 结构：

- **Generator Agent**：生成内容（前端设计/代码）
- **Evaluator Agent**：评估输出质量，提供反馈

### 前端设计实验

#### 评估标准设计

将主观判断转化为可评分的维度：

```markdown
1. **设计质量 (Design Quality)**：设计是否是一个连贯整体而非部件集合？
2. **原创性 (Originality)**：是否有自定义决策的证据？避免模板布局、库默认、AI slop 模式
3. **工艺 (Craft)**：技术执行：字体层级、间距一致性、色彩和谐、对比度
4. **功能性 (Functionality)**：独立于美学的可用性
```

**关键设计**：
- 权重偏向 **设计质量与原创性**（Claude 默认在工艺与功能性上表现良好）
- 明确惩罚 "AI slop" 模式（如紫色渐变覆盖白色卡片）
- 使用 few-shot 示例校准评估器，减少评分漂移

#### 迭代流程

```
Generator → 创建 HTML/CSS/JS 前端
    ↓
Evaluator (Playwright MCP) → 交互式浏览页面 → 截图 → 评分 → 详细批判
    ↓
Generator → 基于反馈改进（或策略性转向新美学方向）
    ↓
（5-15 次迭代，每次约 4 小时 wall-clock 时间）
```

#### 关键发现

| 发现 | 说明 |
|------|------|
| **评分随迭代改进后趋于稳定** | 存在提升空间但会 plateau |
| **非线性改进** | 中间迭代有时优于最终版本 |
| **实现复杂度增加** | Generator 为响应评估器反馈，尝试更雄心勃勃的方案 |
| **Prompting 直接塑造输出特征** | 如 "the best designs are museum quality" 引导视觉收敛方向 |

**典型案例**：荷兰艺术博物馆网站
- 第 9 次迭代：干净深色主题落地页（符合预期）
- 第 10 次迭代：**激进转向** → 3D 空间体验（CSS perspective 渲染棋盘地面、自由位置悬挂艺术品、门户导航替代滚动/点击）

---

## 全栈开发架构

### 三 Agent 系统

| Agent | 角色 | 核心职责 |
|-------|------|----------|
| **Planner** | 规划 | 1-4 句提示 → 扩展为完整产品规格（高层技术设计，不指定细节实现） |
| **Generator** | 构建 | 逐 sprint 实现（React + Vite + FastAPI + SQLite/PostgreSQL） |
| **Evaluator** | QA | Playwright MCP 点击测试 → 评分 → Bug 报告 |

### Sprint Contract 机制

```
Generator 与 Evaluator 协商"完成定义"：
    ↓
    - Generator 提议：将构建什么 + 如何验证成功
    - Evaluator 审核：确保构建正确的东西
    ↓
    双方迭代直至达成一致
    ↓
    Generator 按合同构建 → QA 验证
```

**通信方式**：文件传递（一个 Agent 写文件，另一个读取并响应）

### 技术栈

```
Frontend: React + Vite
Backend: FastAPI
Database: SQLite → PostgreSQL
Version Control: git
Testing: Playwright MCP (浏览器交互测试)
```

---

## 实验对比：Solo vs Harness

### 测试任务

> **Prompt**: "Build a retro video game maker"

### Solo Run 问题

| 问题 | 说明 |
|------|------|
| **布局浪费空间** | 固定高度面板导致视口大部分空白 |
| **工作流僵化** | UI 未引导用户正确序列（先创建 sprites/entities → 再 populate level） |
| **核心功能损坏** | 实体出现在屏幕但不响应输入 → 实体定义与游戏运行时之间连接断裂 |

### Harness Run 优势

| 维度 | Solo | Harness |
|------|------|---------|
| **规格规模** | 直接开始构建 | Planner 扩展为 16-feature spec（10 sprints） |
| **额外功能** | 基础编辑器 + 播放模式 | sprite 动画系统、行为模板、音效音乐、AI 辅助 sprite 生成、游戏导出分享 |
| **视觉一致性** | 无明确设计语言 | 使用 frontend design skill 创建视觉设计语言 |
| **Play Mode** | **损坏** | **可实际操作**（物理有粗糙边缘但核心工作） |

**Evaluator 有效性示例**：

| Sprint | Criteria 数量 | Issue 示例 |
|--------|---------------|------------|
| Sprint 3 | 27 criteria | 评估器逐一验证 → 发现偏差并提交具体 bug |

---

## Harness 简化演进

### 问题

原始 Harness：
- **Bulky**: 结构复杂
- **Slow**: 运行时间长
- **Expensive**: Token 成本高（Solo: $6 vs Harness: $120+）

### 简化原则

> **"Find the simplest solution possible, and only increase complexity when needed."**

**核心洞察**：Harness 的每个组件都编码了一个假设——模型无法独自完成什么。这些假设值得压力测试：
1. 可能不正确
2. 随模型改进快速过时

### Opus 4.6 的改进

根据发布博客：

> "[Opus 4.6] plans more carefully, sustains agentic tasks for longer, can operate more reliably in larger codebases, and has better code review and debugging skills to catch its own mistakes."

改进的能力：
- 更谨慎的规划
- 更长的 agentic 任务维持
- 更可靠的大型代码库操作
- 更好的代码审查和调试技能
- 显著改进的长上下文检索

### 简化步骤

#### 1. 移除 Sprint 结构

**原假设**：Sprint 帮助分解工作为可管理块
**新假设**：Opus 4.6 可原生处理无分解任务
**结果**：Generator 可连续运行 2+ 小时无需 Sprint 分解

#### 2. 单次评估替代 Per-Sprint 评估

**Evaluator 价值边界**：

| 模型 | 任务边界 | Evaluator 价值 |
|------|----------|----------------|
| Opus 4.5 | 任务接近 Generator 能力边界 → Evaluator 捕获有意义问题 | **高** |
| Opus 4.6 | 任务能力边界扩展 → 原需评估的任务现可独自完成 | **降低** |

**结论**：Evaluator 不是固定 yes/no 决策——当任务超出当前模型可靠独立完成范围时，成本值得。

#### 3. 改进 AI 功能构建

添加 prompting 使 Generator 构建正确 Agent（能通过工具驱动应用自身功能）。

---

## 简化后测试：DAW 构建

### Prompt

> "Build a Digital Audio Workstation (DAW)"

### 运行数据

| 维度 | 数值 |
|------|------|
| **时长** | ~4 小时 |
| **Token 成本** | $124 |
| **Generator 运行** | 连续 2+ 小时（无 Sprint 分解） |

### QA Agent 捕获的问题

**Round 1 Feedback**:
- 功能性 gaps

**Round 2 Feedback**:
- 最后 mile issues

**Generator 特性**：
- 无监督时仍可能遗漏细节或 stub features
- QA 在捕获这些最后 mile 问题方面仍有价值

### 结果评估

| 维度 | 评价 |
|------|------|
| **专业性** | 远非专业音乐制作程序 |
| **Agent 作曲技能** | 需大量改进 |
| **Claude 听觉限制** | QA 反馈循环在音乐品味方面效果有限 |
| **核心功能** | ✅ 编曲视图、混音器、传输控制均在浏览器工作 |
| **Agent 驱动** | ✅ 可通过 prompt 完整创作歌曲片段（设 tempo/key → 铺旋律 → 构鼓点 → 调混音 → 加混响） |

> "You might say it's not pitch-perfect yet—but it's getting there."

---

## 关键经验教训

### 1. 实验与调优

> "It is always good practice to experiment with the model you're building against, read its traces on realistic problems, and tune its performance to achieve your desired outcomes."

### 2. 任务分解

复杂任务存在 headroom：
- 分解任务
- 为每个方面应用专业化 Agent

### 3. 模型升级后的 Harness 重检

新模型发布后：
- **剥离不再承重的组件**
- **添加新组件** 以实现之前不可能的更大能力

### 4. Harness 组合空间的移动而非收缩

> "The space of interesting harness combinations doesn't shrink as models improve. Instead, it moves, and the interesting work for AI engineers is to keep finding the next novel combination."

---

## 附录：Planner 生成的规格示例

**RetroForge - 2D Retro Game Maker**

### Overview

Web-based creative studio for designing and building 2D retro-style video games.

### 核心模块

1. **Tile-based Level Editor**：设计游戏世界
2. **Pixel-art Sprite Editor**：创建视觉资产
3. **Visual Entity Behavior System**：定义游戏逻辑
4. **Playable Test Mode**：实时游戏测试

### Features（部分）

**1. Project Dashboard & Management**

- User Stories: 创建项目、查看现有项目、打开编辑器、删除/复制项目
- Project Data Model: 元数据 + Canvas 设置 + Tile 配置 + 色彩选择 + 所有关联资源

---

## 总结要点

| 类别 | 关键结论 |
|------|----------|
| **架构设计** | Generator-Evaluator 分离是提升主观任务质量的关键杠杆 |
| **上下文管理** | Context Reset > Compaction（对强上下文焦虑模型） |
| **评估设计** | 将主观判断转化为具体可评分维度 + few-shot 校准 |
| **模型演进** | Harness 需随模型能力增长而简化——移除不再 load-bearing 的组件 |
| **成本效益** | Evaluator 成本在任务超出模型独立能力边界时值得 |
| **未来展望** | Harness 组合空间随模型改进而移动，而非收缩 |

---

## Sources

- [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)