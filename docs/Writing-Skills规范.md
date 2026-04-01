# Writing Skills 规范

本文档整合 Anthropic 官方 Skills 构建指南、实战经验与五种设计模式，提炼出一套完整的 Skill 设计规范。旨在帮助开发者构建高质量、可复用、易维护的 Claude Skills。

---

## 目录

- [核心设计原则](#核心设计原则)
- [文件结构规范](#文件结构规范)
- [YAML Frontmatter 规范](#yaml-frontmatter-规范)
- [Description 编写规范](#description-编写规范)
- [指令编写规范](#指令编写规范)
- [五种设计模式](#五种设计模式)
- [九种 Skill 类型](#九种-skill-类型)
- [踩坑点章节](#踩坑点章节)
- [测试与迭代](#测试与迭代)
- [分发与共享](#分发与共享)
- [参考检查清单](#参考检查清单)

---

## 核心设计原则

### 递进式披露（Progressive Disclosure）

Skills 使用三级加载系统，在保持专业能力的同时最大限度地减少 token 消耗：

| 级别 | 内容 | 加载时机 | 建议大小 |
|------|------|----------|----------|
| **第一级** | YAML frontmatter（name + description） | 始终在上下文中 | ~100 字 |
| **第二级** | SKILL.md 正文指令 | Skill 触发时加载 | <500 行 |
| **第三级** | 链接文件（references/、scripts/、assets/） | 按需加载 | 无限制 |

**核心技巧**：
- 保持 SKILL.md 简洁，将详细文档移至 `references/`
- 清晰引用捆绑资源，指明何时加载
- 对于大型参考文件（>300 行），在 SKILL.md 中提供目录概览

### 可组合性（Composability）

Claude 可以同时加载多个 Skills。你的 Skill 应能与其他 Skills 协同工作：

- ✅ 不假设自己是唯一可用的能力
- ✅ 可按名字引用其他已安装的 Skills
- ✅ 使用通用工具而非独占特定服务

### 可移植性（Portability）

Skills 在 Claude.ai、Claude Code 和 API 上的工作方式完全相同：

- 创建一次，即可在所有平台使用
- 如有环境依赖，在 `compatibility` 字段中注明
- 确保脚本在不同操作系统上可执行

---

## 文件结构规范

### 标准目录结构

```
skill-name/
├── SKILL.md                  # 必须——主 Skill 文件
├── scripts/                  # 可选——可执行代码
│    ├── process_data.py      # 确定性任务脚本
│    └── validate.sh          # 验证脚本
├── references/               # 可选——按需加载的文档
│    ├── api-guide.md         # API 规范
│    ├── conventions.md       # 编码规范
│    └── examples/            # 示例文件
└── assets/                   # 可选——输出使用的资源
     ├── report-template.md   # 输出模板
     └── icons/               # 图标资源
```

### 命名规则

| 元素 | 规则 | 示例 |
|------|------|------|
| **文件夹名** | kebab-case，无空格/下划线/大写 | `notion-project-setup` ✅ |
| **SKILL.md** | 必须完全匹配（区分大小写） | `SKILL.md` ✅，`skill.md` ❌ |
| **name 字段** | 与文件夹名一致，kebab-case | `name: notion-project-setup` |

### 禁止事项

- ❌ Skill 文件夹内包含 `README.md`（所有文档放 SKILL.md 或 references/）
- ❌ 文件夹名使用空格：`Notion Project Setup`
- ❌ 文件夹名使用下划线：`notion_project_setup`
- ❌ 文件夹名使用大写：`NotionProjectSetup`

---

## YAML Frontmatter 规范

### 最小必要格式

```yaml
---
name: your-skill-name
description: What it does. Use when user asks to [specific phrases].
---
```

### 字段详解

| 字段 | 必填 | 规则 |
|------|:----:|------|
| `name` | ✅ | kebab-case，无空格/大写，与文件夹名匹配 |
| `description` | ✅ | <1024 字符，包含功能描述和触发条件，无 XML 标签 |
| `license` | ❌ | 开源时使用：MIT、Apache-2.0 |
| `compatibility` | ❌ | 环境要求说明：产品、系统包、网络需求 |
| `metadata` | ❌ | 自定义键值对：author、version、mcp-server |

### 安全限制

**Frontmatter 中禁止**：
- ❌ XML 尖括号（`< >`）
- ❌ 以 "claude" 或 "anthropic" 为前缀命名（保留字）

---

## Description 编写规范

Description 是 Skill 触发的核心机制。Claude 通过扫描描述判断"这个请求有没有对应的 Skill"。

### 核心原则

> Description 不是摘要——它描述的是**何时该触发这个 Skill**，而非"这个 Skill 做什么"。

### 结构模板

```
[它做什么] + [何时使用] + [核心能力/触发短语]
```

### 良好示例

```yaml
# ✅ 具体、可执行、包含触发短语
description: Analyzes Figma design files and generates developer handoff
documentation. Use when user uploads .fig files, asks for "design specs",
"component documentation", or "design-to-code handoff".

# ✅ 包含明确触发词
description: Manages Linear project workflows including sprint planning,
task creation, and status tracking. Use when user mentions "sprint",
"Linear tasks", "project planning", or asks to "create tickets".

# ✅ 清晰价值主张 + 触发条件
description: End-to-end customer onboarding workflow for PayFlow. Handles
account creation, payment setup, and subscription management. Use when
user says "onboard new customer", "set up subscription", or "create
PayFlow account".
```

### 糟糕示例

```yaml
# ❌ 太模糊
description: Helps with projects.

# ❌ 缺少触发条件
description: Creates sophisticated multi-page documentation systems.

# ❌ 过于技术性，无用户触发词
description: Implements the Project entity model with hierarchical
relationships.
```

### 防止触发问题

**触发不足（Under-triggering）**：
- 添加更多细节和针对性内容
- 包含用户可能说的具体短语
- 对于技术术语，添加关键词

**过度触发（Over-triggering）**：
- 添加负面触发词："Do NOT use for..."
- 更加具体化范围
- 明确边界："Use specifically for X, not for general Y"

---

## 指令编写规范

### 推荐结构模板

```markdown
---
name: your-skill
description: [...]
---

# Skill Name

## Instructions

### Step 1: [First Major Step]
Clear explanation of what happens.

```bash
python scripts/fetch_data.py --project-id PROJECT_ID
Expected output: [describe what success looks like]
```

### Step 2: [Next Step]
[Continue with clear, actionable instructions]

## Examples

### Example 1: [Common Scenario]
User says: "Set up a new marketing campaign"
Actions:
1. Fetch existing campaigns via MCP
2. Create new campaign with provided parameters
Result: Campaign created with confirmation link

## Troubleshooting

### Error: [Common Error Message]
Cause: [Why it happens]
Solution: [How to fix]
```

### 编写原则

| 原则 | 说明 |
|------|------|
| **具体可执行** | 给出具体命令，而非抽象指令 |
| **包含错误处理** | 预设常见问题及解决方案 |
| **清晰引用资源** | 指明何时加载哪个文件 |
| **解释原因** | 说明"为什么"而非仅"做什么" |
| **避免过度限制** | 给 Claude 适应具体情况的灵活性 |

### 具体可执行示例

```markdown
# ✅ 好——具体命令 + 常见问题
Run `python scripts/validate.py --input {filename}` to check data format.
If validation fails, common issues include:
- Missing required fields (add them to the CSV)
- Invalid date formats (use YYYY-MM-DD)

# ❌ 州——抽象指令
Validate the data before proceeding.
```

### 关键指令强调

对于必须执行的验证，使用明确的强调：

```markdown
## Critical: Before calling create_project, verify:
- Project name is non-empty
- At least one team member assigned
- Start date is not in the past
```

---

## 五种设计模式

根据任务性质选择合适的设计模式：

### 模式选择决策树

| 你的问题 | 推荐模式 |
|----------|----------|
| 如何让 Agent 掌握特定库或框架的知识？ | 工具封装 |
| 如何确保每次输出的文档结构一致？ | 生成器 |
| 如何自动化代码审查或安全审计？ | 审查器 |
| 如何防止 Agent 在需求不明确时乱猜？ | 反转 |
| 如何确保复杂任务的每个步骤都被执行？ | 流水线 |

### 模式一：工具封装（Tool Wrapper）

**用途**：让 Agent 按需获取特定库的上下文。

**核心技巧**：
- 监听特定库关键词
- 从 `references/` 动态加载内部文档
- 将规范作为绝对准则应用

**示例结构**：

```markdown
---
name: api-expert
description: FastAPI 开发最佳实践与规范。在构建、审查或调试 FastAPI 应用、REST API 或 Pydantic 模型时使用。
---

你是 FastAPI 开发专家。

## 核心规范
加载 'references/conventions.md' 获取完整的 FastAPI 最佳实践列表。

## 审查代码时
1. 加载规范参考文件
2. 对照每条规范检查用户代码
3. 对于每处违规，引用具体规则并给出修复建议
```

### 模式二：生成器（Generator）

**用途**：强制输出一致性，编排"填空"流程。

**核心技巧**：
- `assets/` 存放输出模板
- `references/` 存放风格指南
- 指令充当项目经理，协调资源检索

**示例结构**：

```markdown
---
name: report-generator
description: 生成 Markdown 格式的结构化技术报告。当用户要求撰写、创建或起草报告、摘要或分析文档时使用。
---

你是一个技术报告生成器。严格按照以下步骤执行：

第一步：加载 'references/style-guide.md' 获取语气和格式规则。
第二步：加载 'assets/report-template.md' 获取所需的输出结构。
第三步：向用户询问填充模板所需的缺失信息。
第四步：按照风格指南规则填充模板。
```

### 模式三：审查器（Reviewer）

**用途**：将"检查什么"与"如何检查"分离。

**核心技巧**：
- 审查标准存于 `references/review-checklist.md`
- 系统评分，按严重程度分组
- 可更换清单获得不同专项审计工具

**示例结构**：

```markdown
---
name: code-reviewer
description: 审查 Python 代码的质量、风格和常见 Bug。当用户提交代码请求审查、寻求代码反馈或需要代码审计时使用。
---

你是一名 Python 代码审查员。

第一步：加载 'references/review-checklist.md' 获取完整的审查标准。
第二步：仔细阅读用户的代码。
第三步：将清单中的每条规则应用于代码，按严重程度分类：error、warning、info。
第四步：生成结构化审查报告。
```

### 模式四：反转（Inversion）

**用途**：防止 Agent 在需求不明确时猜测，强制先收集上下文。

**核心技巧**：
- 明确的门控指令："在所有阶段完成之前不得开始构建"
- 按顺序提出结构化问题
- 等待用户回答后再进入下一阶段

**示例结构**：

```markdown
---
name: project-planner
description: 通过结构化提问收集需求，然后生成计划。当用户说"我想构建"、"帮我规划"、"设计一个系统"时使用。
---

你正在进行一次结构化需求访谈。在所有阶段完成之前，不得开始构建或设计。

## 第一阶段——问题发现（每次只问一个问题）
Q1："这个项目为用户解决什么问题？"
Q2："主要用户是谁？"
...

## 第三阶段——综合（仅在所有问题都回答后进行）
1. 加载 'assets/plan-template.md'
2. 使用收集到的需求填充模板
```

### 模式五：流水线（Pipeline）

**用途**：复杂任务强制执行严格顺序工作流。

**核心技巧**：
- 指令本身就是工作流定义
- 明确的菱形门控条件
- 按步骤引入不同参考文件

**示例结构**：

```markdown
---
name: doc-pipeline
description: 通过多步骤流水线从 Python 源代码生成 API 文档。当用户要求为模块编写文档、生成 API 文档时使用。
---

你正在运行一个文档生成流水线。按顺序执行每个步骤。不得跳过步骤。

## 第一步——解析与清点
分析 Python 代码，提取所有公开 API。询问："这是你想要文档化的完整 API 吗？"

## 第二步——生成文档字符串
加载 'references/docstring-style.md'，生成文档字符串供用户确认。
在用户确认之前，不得进入第三步。

## 第三步——组装文档
加载 'assets/api-doc-template.md'，编译成单一 API 参考文档。
```

### 模式组合

五种模式可以组合使用：

- 流水线末尾加入审查器步骤自我检验
- 生成器开头借助反转模式收集变量
- 工具封装作为其他模式的底层支撑

---

## 九种 Skill 类型

根据使用场景选择 Skill 类型：

| 类型 | 用途 | 示例 |
|------|------|------|
| **库与 API 参考** | 帮助正确使用库/SDK | `billing-lib`、`internal-platform-cli` |
| **产品验证** | 测试/验证代码正常工作 | `signup-flow-driver`、`checkout-verifier` |
| **数据获取与分析** | 连接数据和监控体系 | `funnel-query`、`grafana` |
| **业务流程自动化** | 自动化重复性工作流 | `standup-post`、`weekly-recap` |
| **代码脚手架** | 生成框架样板代码 | `new-migration`、`create-app` |
| **代码质量审查** | 执行代码质量标准 | `adversarial-review`、`code-style` |
| **CI/CD 与部署** | 拉取、推送、部署代码 | `babysit-pr`、`deploy-service` |
| **运维手册** | 多工具排查生成报告 | `service-debugging`、`log-correlator` |
| **基础设施运维** | 执行日常维护操作 | `dependency-management`、`cost-investigation` |

---

## 踩坑点章节

任何 Skill 中信息量最大的部分就是踩坑点章节。这些章节应该根据 Claude 在使用 Skill 时遇到的常见失败点逐步积累。

### 建议结构

```markdown
## Common Pitfalls

### Pitfall 1: [常见失败模式]
**症状**：[Claude 常犯的错误]
**原因**：[为什么会发生]
**解决方案**：[如何避免/修复]

### Pitfall 2: [另一个失败模式]
...
```

### 常见踩坑点示例

| 踩坑点 | 解决方案 |
|--------|----------|
| 指令太冗长 | 保持简洁，使用项目符号，将详细内容移至单独文件 |
| 指令被埋没 | 关键指令放最前面，使用 `## Important` 或 `## Critical` |
| 语言模糊 | 使用具体命令而非抽象指令 |
| 模型"偷懒" | 添加明确鼓励："Quality is more important than speed" |
| 大上下文问题 | 将 SKILL.md 控制在 500 行内，移详细文档至 references/ |

---

## 测试与迭代

### 测试方法

| 方法 | 适用场景 |
|------|----------|
| **手动测试** | 快速迭代，无需配置 |
| **脚本化测试** | 跨版本可重复验证 |
| **程序化测试** | 系统评估套件 |

### 测试类型

#### 1. 触发测试

验证 Skill 在正确时机加载：

```
应该触发：
- "Help me set up a new ProjectHub workspace"
- "I need to create a project in ProjectHub"
- 换句话请求："Initialize a ProjectHub project for Q4 planning"

不应触发：
- "What's the weather in San Francisco?"
- "Help me write Python code"
```

#### 2. 功能测试

验证 Skill 产生正确输出：

```
Test: Create project with 5 tasks
Given: Project name "Q4 Planning", 5 task descriptions
When: Skill executes workflow
Then:
   - Project created in ProjectHub
   - 5 tasks created with correct properties
   - No API errors
```

#### 3. 性能对比

与基线比较改善程度：

```
Without skill:
- 15 back-and-forth messages
- 3 failed API calls
- 12,000 tokens

With skill:
- 2 clarifying questions
- 0 failed API calls
- 6,000 tokens
```

### 迭代信号

| 信号 | 解决方案 |
|------|----------|
| **触发不足** | 添加更多细节和触发短语 |
| **过度触发** | 添加负面触发词，更具体化 |
| **执行问题** | 改进指令，添加错误处理 |
| **结果不一致** | 添加验证脚本或模板 |

---

## 分发与共享

### 分发渠道

| 渠道 | 适用场景 |
|------|----------|
| **代码仓库** | 小团队，较少仓库 |
| **插件市场** | 大规模分发，用户自选安装 |
| **GitHub 公开仓库** | 开源 Skills |

### 安装指南模板

```markdown
## Installing the [Your Service] skill

1. Download the skill:
   - Clone repo: `git clone https://github.com/yourcompany/skills`
   - Or download ZIP from Releases

2. Install in Claude:
   - Open Claude.ai > Settings > skills
   - Click "Upload skill"
   - Select the skill folder (zipped)

3. Enable the skill:
   - Toggle on the [Your Service] skill
   - Ensure your MCP server is connected

4. Test:
   - Ask Claude: "Set up a new project in [Your Service]"
```

### 定位建议

**聚焦结果而非功能**：

```markdown
# ✅ 好
"The ProjectHub skill enables teams to set up complete project workspaces
in seconds — instead of spending 30 minutes on manual setup."

# ❌ 差
"The ProjectHub skill is a folder containing YAML frontmatter and
Markdown instructions that calls our MCP server tools."
```

---

## 参考检查清单

### 开发前

- [ ] 已确定 2-3 个具体使用场景
- [ ] 已确定所需工具（内置或 MCP）
- [ ] 已阅读设计规范和示例 Skills
- [ ] 已规划文件夹结构

### 开发中

- [ ] 文件夹以 kebab-case 命名
- [ ] SKILL.md 文件存在（拼写准确）
- [ ] YAML frontmatter 有 `---` 分隔符
- [ ] `name` 字段：kebab-case，无空格/大写
- [ ] `description` 包含功能描述 + 使用时机
- [ ] 无 XML 标签（`< >`）
- [ ] 指令清晰且可执行
- [ ] 包含错误处理
- [ ] 提供了示例
- [ ] 引用已清晰链接
- [ ] SKILL.md <500 行

### 上传前

- [ ] 已测试在明显任务上的触发
- [ ] 已测试在换句话请求上的触发
- [ ] 已验证不会在无关话题上触发
- [ ] 功能测试通过
- [ ] 工具集成正常工作（如适用）

### 上传后

- [ ] 在真实对话中测试
- [ ] 监控触发不足/过度触发情况
- [ ] 收集用户反馈
- [ ] 迭代 description 和指令
- [ ] 在 metadata 中更新版本号

---

## 附录：完整 Frontmatter 示例

```yaml
---
name: projecthub-onboarding
description: End-to-end project setup workflow for ProjectHub. Creates
workspaces, pages, and templates in seconds. Use when user says "set up
a new project", "create a ProjectHub workspace", "initialize a project",
or mentions "ProjectHub setup" or "onboarding workflow".
license: MIT
compatibility: Requires ProjectHub MCP server connection
metadata:
  author: ProjectHub Team
  version: 1.2.0
  mcp-server: projecthub
  category: productivity
  tags: [project-management, automation, onboarding]
---
```

---

## 参考资源

- [Claude Skills 完整构建指南](Claude-Skills-完全构建指南.md)
- [Claude Code Skills 实战经验](Claude-Code-Skills-实战经验.md)
- [Agent Skill 五种设计模式](Agent-Skill-五种设计模式.md)
- [Anthropic Skills 仓库](https://github.com/anthropics/skills)
- [Claude Code Skills 文档](https://code.claude.com/docs/en/skills)