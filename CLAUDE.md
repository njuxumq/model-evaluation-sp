# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 语言要求

**重要**：后续所有对话输出、文档编写、代码注释均须使用中文。

---

## 项目概述

这是一个 Claude Code 模型评测 Skill 的**持续重构与开发项目**。

**当前状态**：核心重构已完成，四阶段流程结构已建立，持续迭代优化中。

**功能**：协助用户完成 AI 模型评测任务的全流程管理，包括评测集处理、维度配置、任务提交和结果查看。

---

## 项目结构

```
model-evaluation-sp/
├── .claude/skills/model-evaluation/    # 已部署的 Skill（主要工作目录）
├── docs/references/                    # Skill 构建参考文档
├── assets/media/                       # 文档引用的图片资源
├── CLAUDE.md                           # 本文件
└── REQUEST.md                          # 需求文档（迭代开发规范）
```

**重要说明**：
- 原始 Skill 目录 `skills/model-evaluation-origin/` 已移除，重构输出在 `.claude/skills/model-evaluation/`
- `REQUEST.md` 包含迭代开发规范，定义了文档书写规范、Bulletproofing、TDD 验证流程等
- `docs/references/` 包含 Skill 构建的最佳实践参考

---

## 已部署 Skill 结构

```
.claude/skills/model-evaluation/
├── SKILL.md                    # 主入口：流程概览与交互规范（内嵌迭代开发规范）
├── eval-init.md                # 阶段1：初始化
├── eval-build.md               # 阶段2：构建配置
├── eval-set.md                 # 阶段3：评测集处理
├── eval-execute.md             # 阶段4：执行评测
├── processes/                  # 独立子流程（10个流程文件）
│   ├── dimension-case-level.md     # 流程5：定制用例级评测配置
│   ├── dimension-general.md        # 流程6：通用维度级评测配置
│   ├── evalset-create.md           # 流程1：问题集生成
│   ├── evalset-parse.md            # 流程3：评测集解析
│   ├── evalset-supplement.md       # 流程2：答案补充
│   ├── evalset-field-mapping.md    # 字段映射流程
│   ├── evalset-field-validation.md # 字段验证流程
│   ├── evalset-model-selection.md  # 模型选择流程
│   ├── keypoint-process.md         # 流程4：评测点生成
│   └── python-env-process.md       # Python 环境检测
├── references/                 # 参考文档（7个文档，按需加载）
│   ├── 评测服务接口说明.md
│   ├── 认证服务接口说明.md
│   ├── 中间产物说明.md
│   ├── 脚本定义.md
│   ├── 内置模板说明.md
│   ├── 评测维度说明.md
│   └── 维度配置转化规则.md
├── assets/                     # 只读资源
│   ├── eval-judge.json         # 默认评委配置
│   ├── experts/                # 专家模板（14个场景）
│   ├── dimensions/             # 维度模板（46个维度）
│   └── templates/              # 自定义维度模板
└── scripts/                    # Python 可执行脚本（只读）
    ├── eval_auth.py            # 鉴权管理
    ├── eval_set.py             # 评测集管理
    ├── eval_task.py            # 任务管理
    ├── eval_dimension.py       # 维度配置工具
    ├── eval_model.py           # 模型评测工具
    ├── clients/                # API 客户端
    ├── files/                  # 文件处理工具
    └── utils/                  # 通用工具
```

---

## 四阶段评测流程

| 阶段 | 入口文档 | 核心任务 | 完成标志 |
|------|----------|----------|----------|
| **初始化** | `eval-init.md` | 环境检测、鉴权验证、会话目录确认 | `env.cfg`、`auth.json`、`session-id/` 存在 |
| **构建配置** | `eval-build.md` | 场景确认、评测标准确认、评委配置 | `eval-dimension.json`、`eval-judge.json` 存在 |
| **评测集处理** | `eval-set.md` | 评测集解析、标准化、评测点生成、上传 | `evalset-meta.json` 存在 |
| **执行评测** | `eval-execute.md` | 任务提交、状态监控、结果展示 | 评测结果已展示 |

**执行规则**：
- **默认**：从初始化阶段开始，按顺序执行各阶段
- **用户明确指明**：可直接跳转到指定阶段/任务/步骤
- **阶段内自动流转**：当前阶段全部任务完成后，自动进入下一阶段

---

## 流程速查

| 编号 | 流程名称 | 文档位置 | 调用时机 |
|------|----------|----------|----------|
| 流程1 | 问题集生成 | `processes/evalset-create.md` | 评测集处理阶段任务1步骤1（无评测集分支） |
| 流程2 | 答案补充 | `processes/evalset-supplement.md` | 问题集生成后补充答案 |
| 流程3 | 评测集解析 | `processes/evalset-parse.md` | 评测集处理阶段任务1步骤2 |
| 流程4 | 评测点生成 | `processes/keypoint-process.md` | 评测集处理阶段任务1步骤4（定制用例级） |
| 流程5 | 定制用例级评测配置 | `processes/dimension-case-level.md` | 构建阶段任务2步骤3 |
| 流程6 | 通用维度级评测配置 | `processes/dimension-general.md` | 构建阶段任务2步骤3 |

---

## Python 脚本

| 脚本 | 子命令 | 功能 |
|------|--------|------|
| `eval_auth.py` | detect, login, token, check | OAuth 鉴权（支持回调/OOB模式） |
| `eval_set.py` | analysis, normalize, expand, submit | 评测集解析、标准化、扩展、上传 |
| `eval_task.py` | submit, status, summary | 任务提交、状态轮询、结果摘要 |
| `eval_dimension.py` | check, update | 校验配置、填充 judge_id |
| `eval_model.py` | list, get | 模型列表获取、模型信息查询 |

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

## 评测方式

| 方式 | 说明 | 配置流程 |
|------|------|----------|
| **定制用例级** | 针对每个评测样本生成评测要点，细节偏好对齐 | 流程5 |
| **通用维度级** | 统一的评测标准，宏观全面覆盖 | 流程6 |

---

## 关键约束（Red Flags）

**禁止行为**：
- ❌ 跳过缓存检测直接重新执行环境检测
- ❌ Token 失效时跳过重新授权
- ❌ 执行复合 bash 命令（必须逐行执行）
- ❌ 跳过用户对维度权重、字段映射的确认
- ❌ 在当前阶段未完成时询问后续阶段问题（除非用户明确指明跳转）
- ❌ 修改 `assets/` 或 `scripts/` 目录下的只读文件
- ❌ 跳过前置验证直接执行后续任务
- ❌ 未等待任务完成就展示评测结果

**必须行为**：
- ✅ 首次执行任务前展示流程概览
- ✅ 进入阶段时使用 TodoWrite 初始化任务列表
- ✅ 文本信息优先于工具调用输出
- ✅ OAuth 手动模式下等待用户输入授权码
- ✅ 任务成功后必须展示在线报告链接

---

## 会话状态文件

关键中间文件位于 `{work-dir}/.eval/`：
- `env.cfg`：环境配置缓存
- `auth.json`：OAuth Token 缓存
- `{session-id}/eval-dimension.json`：评测维度配置
- `{session-id}/eval-judge.json`：评委模型配置
- `{session-id}/evalset/evalset-standard.jsonl`：标准化评测集
- `{session-id}/evalset/evalset-meta.json`：评测集元数据（上传后）
- `{session-id}/selected-models.json`：选定模型列表（评测集无答案场景）
- `{session-id}/evaltask/evaltask-meta.json`：任务元数据
- `{session-id}/evaltask/evaltask-result.json`：评测结果

---

## 参考文档

`docs/references/` 目录包含 Skill 构建的最佳实践：

| 文档 | 内容 |
|------|------|
| `Claude-Skills-完全构建指南.md` | 完整的 Skill 构建指南 |
| `Claude-Code-Skills-实战经验.md` | Anthropic 内部 Skills 实战经验 |
| `Agent-Skill-五种设计模式.md` | 五种 Skill 设计模式 |
| `Writing-Skills规范.md` | Writing Skills 规范 |

---

## 开发规范

修改 Skill 文档或脚本时，遵循 `REQUEST.md`（或 `SKILL.md` 内嵌）的迭代开发规范：

**核心原则**：
1. **Description = 触发条件**，绝不总结工作流程
2. **Token 效率优先**，按需加载，引用不重复
3. **验证驱动**，No document without failing test first

**文档结构要求**：
- YAML frontmatter：name + description
- 目标：目标清单 + 核心原则
- 阶段完成标志：验证顺序，状态驱动
- 任务列表：过程导向格式（动作A → 动作B → 动作C）

**文档字数目标**：
| 文档类型 | 目标 | 原因 |
|----------|------|------|
| SKILL.md | < 5000 字 | 高频加载 |
| eval-*.md | < 2000 字 | 阶段触发时加载 |
| processes/*.md | < 1500 字 | 按需加载 |
| references/*.md | 按需精简 | 按需加载但仍需精简 |

---

## 持续重构方向

当前重构重点：

| 方向 | 说明 | 状态 |
|------|------|------|
| 流程拆分 | 将复杂流程拆分为独立 processes 文件 | ✅ 已完成 |
| 四阶段结构 | 从三阶段扩展为四阶段（新增评测集处理） | ✅ 已完成 |
| 多模型横评 | 支持多模型对同一问题的评测点去重生成 | ✅ 已完成 |
| Token 精简 | 减少文档冗余，提高加载效率 | 🔄 进行中 |
| Bulletproofing | 完善 Red Flags 和理性化借口表格 | 🔄 进行中 |
| TDD 验证 | 为关键流程添加测试验证 | 🔄 进行中 |

---

## 修改检查清单

| 检查项 | 说明 |
|--------|------|
| 功能验证 | 修改后功能符合预期 |
| 依赖检查 | 是否影响引用 |
| 格式检查 | 符合书写规范 |
| 测试验证 | 相关用例通过 |
| 字数控制 | 不超过目标字数 |
| YAML frontmatter | 符合 CSO 规范 |
| 确认约束 | 标注"不可跳过"步骤 |

---

## 变量速查

| 变量 | 说明 |
|------|------|
| `{python-cmd}` | Python命令，根据初始化阶段检测结果为 `python` 或 `python3` |
| `{python-env}` | Python环境变量前缀，Windows非UTF-8终端为 `PYTHONUTF8=1 `，其他情况为空 |
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |
| `{state_token}` | OAuth授权状态令牌（UUID格式） |
| `{auth_code}` | 用户授权后获得的授权码 |
| `{task_id}` | 评测任务ID |
| `{platform_url}` | 在线评测报告链接 |
| `{ext}` | 文件扩展名 |