# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 语言要求

**重要**：后续所有对话输出、文档编写、代码注释均须使用中文。

---

## 项目概述

这是一个 Claude Code 模型评测 Skill 的重构与开发项目。

**目标**：将原始评测 Skill 从 `skills/model-evaluation-origin/` 重构后输出至 `.claude/skills/model-evaluation/`。

**功能**：协助用户完成 AI 模型评测任务的全流程管理，包括评测集处理、维度配置、任务提交和结果查看。

---

## 项目结构说明

### 原始 Skill（待重构）

```
skills/model-evaluation-origin/
├── SKILL.md              # 主入口文档
├── eval-init.md          # 阶段1：初始化
├── eval-build.md         # 阶段2：构建配置
├── eval-execute.md       # 阶段3：执行评测
├── processes/            # 独立子流程文档
├── references/           # 参考文档
├── assets/               # 只读资源文件
└── scripts/              # Python 可执行脚本
```

### 重构参考文档

```
docs/
├── Claude-Code-Skills-实战经验.md      # Anthropic 内部 Skills 实战经验
├── Claude-Skills-完全构建指南.md       # 完整的 Skill 构建指南
└── Agent-Skill-五种设计模式.md         # 五种 Skill 设计模式

assets/media/                            # docs/ 文档引用的图片资源
```

**重要说明**：
- `docs/` 目录下的文档是重构的**参考指导**，包含 Skill 构建的最佳实践和设计模式
- 原始 Skill 在 `skills/model-evaluation-origin/` 中，**并未按照** `docs/` 中的规范编写
- 重构目标是将原始 Skill 按照 `docs/` 中的最佳实践进行改进

---

## 原始 Skill 架构

### 三阶段评测流程

原始 Skill 采用严格顺序执行的三阶段工作流程：

| 阶段 | 入口文档 | 核心任务 |
|------|----------|----------|
| **初始化** | `eval-init.md` | 环境检测、鉴权验证、会话目录确认 |
| **构建** | `eval-build.md` | 场景确认、评测标准、评委配置、评测集处理 |
| **执行** | `eval-execute.md` | 任务提交、状态轮询、结果展示 |

**执行规则**：
- **默认**：从初始化阶段开始，按顺序执行各阶段
- **用户明确指明**：可直接跳转到指定阶段/任务/步骤
- **阶段内自动流转**：当前阶段全部任务完成后，自动进入下一阶段

### 目录结构详解

```
skills/model-evaluation-origin/
├── SKILL.md                      # 主入口，定义触发条件和流程概览
├── eval-init.md                  # 阶段1：环境检测、鉴权、会话目录
├── eval-build.md                 # 阶段2：场景、维度、评委、评测集
├── eval-execute.md               # 阶段3：提交、轮询、结果展示
├── processes/                    # 独立子流程（由主阶段按需调用）
│   ├── dimension-process.md      # 流程5/6：维度配置
│   ├── keypoint-process.md       # 流程4：评测点生成
│   ├── evalset-create-process.md # 流程1：评测集生成
│   ├── evalset-parse-process.md  # 流程3：评测集解析
│   └── evalset-supplement-process.md # 流程2：答案补充
├── references/                   # 参考文档（按需加载）
│   ├── 评测服务接口说明.md       # API 规范
│   ├── 脚本定义.md               # 脚本使用说明
│   ├── 中间产物说明.md           # 中间文件格式
│   ├── 内置模板说明.md           # 内置模板列表
│   └── 评测维度说明.md           # 维度配置指南
├── assets/                       # 只读资源
│   ├── eval-judge.json           # 默认评委配置
│   ├── experts/                  # 专家模板（场景化评测方案）
│   └ensions/                     # 维度模板（单一评测维度定义）
└── scripts/                      # Python 可执行脚本
    ├── eval_auth.py              # 鉴权管理
    ├── eval_set.py               # 评测集管理
    ├── eval_task.py              # 任务管理
    └── eval_dimension.py         # 维度配置工具
```

---

## Python 脚本

| 脚本 | 子命令 | 功能 |
|------|--------|------|
| `eval_auth.py` | detect, login, token, check | OAuth 鉴权（支持回调/OOB模式） |
| `eval_set.py` | analysis, normalize, submit | 评测集解析、标准化、上传 |
| `eval_task.py` | submit, status, summary | 任务提交、状态轮询、结果摘要 |
| `eval_dimension.py` | check, update | 校验配置、填充 judge_id |

**执行格式**：`{python-env}{python-cmd} {脚本路径} {子命令} {参数}`
- `{python-env}`：Windows GBK 终端为 `PYTHONUTF8=1 `，其他环境为空
- `{python-cmd}`：根据环境检测结果为 `python` 或 `python3`

---

## 评测类型

| 类型 | 输出形式 | 适用场景 |
|------|----------|----------|
| `llm-score` | 1-5 分 | 语义理解、创意质量、用户体验 |
| `llm-judge` | 通过/不通过 | 合规性、安全性检查 |
| `builtin` | 数值指标 | BLEU、ROUGE、JSONFORMAT 等客观指标 |

---

## 关键约束（Red Flags）

原始 Skill 中定义的禁止行为：

**禁止行为**：
- ❌ 跳过缓存检测直接重新执行环境检测
- ❌ Token 失效时跳过重新授权
- ❌ 执行复合 bash 命令（必须逐行执行）
- ❌ 跳过用户对维度权重、字段映射的确认
- ❌ 在初始化阶段询问构建阶段问题（除非用户明确指明跳转）

**必须行为**：
- ✅ 首次执行任务前展示阶段概览
- ✅ 使用 TodoWrite 跟踪阶段内任务进度
- ✅ 文本信息优先于工具调用输出
- ✅ OAuth 手动模式下等待用户输入授权码

---

## 会话状态文件

关键中间文件位于 `{work-dir}/.eval/`：
- `auth.json`：OAuth Token 缓存
- `env.cfg`：环境配置缓存
- `{session-id}/eval-dimension.json`：评测维度配置
- `{session-id}/eval-judge.json`：评委模型配置
- `{session-id}/evalset/evalset-standard.jsonl`：标准化评测集
- `{session-id}/evaltask/evaltask-result.json`：评测结果

---

## 重构目标

将原始 Skill 按照 `docs/` 中的最佳实践进行重构：

1. **遵循五种设计模式**：原始 Skill 主要采用"流水线（Pipeline）"模式，重构时需验证是否符合设计模式规范
2. **应用渐进式披露**：将详细信息拆分到独立文件，按需加载
3. **完善踩坑点章节**：记录 Claude 常见失败模式
4. **优化 description 字段**：编写"何时触发"条件
5. **保持三阶段工作流**：维护核心流程结构不变