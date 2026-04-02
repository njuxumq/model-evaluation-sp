# evalset-parse-process 拆分设计文档

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this design task-by-task.

**目标**：将 `evalset-parse-process.md` 拆分为主流程 + 3个子流程，符合流水线设计模式。

**设计模式**：流水线（Pipeline）+ 子流程按需加载

**方案选择**：选项 A（主流水线 + 子流程），理由：实际场景"评测集没有答案"更常见，分支A按需加载收益更高。

---

## 文件结构变化

### 当前结构

```
processes/
├── evalset-parse-process.md      # 276行，包含全部步骤和分支
```

### 重构后结构

```
processes/
├── evalset-parse-process.md      # 主流程（步骤1-4 + 门控 + 子流程调用入口）
├── evalset-field-mapping.md      # 子流程1：字段映射审查（步骤3详细）
├── evalset-model-selection.md    # 子流程2：模型选择（分支A）
├── evalset-field-validation.md   # 子流程3：字段校验（分支B）
```

---

## 各文件内容规划

### 1. evalset-parse-process.md（主流程）

**保留内容**：
- YAML frontmatter（name, description）
- 目标说明
- 流程概览表格
- 禁止规则（Red Flags）
- 步骤1：获取评测集文件
- 步骤2：解析字段结构
- 步骤3：生成初始映射（调用子流程入口）
- 步骤4：判断后续走向（门控 + 分支调用入口）
- 变量速查表

**移除内容**：
- 步骤3的详细映射规则 → evalset-field-mapping.md
- 步骤3.2确认映射详细 → evalset-field-mapping.md
- 分支A全部内容 → evalset-model-selection.md
- 分支B全部内容 → evalset-field-validation.md
- 常见错误表 → 各子流程中相关部分

**预估行数**：~120行

---

### 2. evalset-field-mapping.md（子流程1）

**来源**：从主流程步骤3提取

**内容**：
- YAML frontmatter
- 目标：生成字段映射配置，等待用户确认
- 字段匹配规则表（含标准字段、关键词、匹配规则）
- 必填字段说明
- 映射格式示例
- 确认映射流程（展示 → 询问 → 处理用户选择）
- 返回：主流程步骤4

**预估行数**：~50行

---

### 3. evalset-model-selection.md（子流程2）

**来源**：从主流程分支A提取

**内容**：
- YAML frontmatter
- 目标：获取可用模型列表，用户选择并保存
- 触发条件：answer 字段不存在或全空
- 步骤A1：获取可用模型列表（命令 + 失败处理）
- 步骤A2：用户选择并保存（展示格式 + 命令 + selection 参数格式）
- 返回：标准化阶段

**预估行数**：~60行

---

### 4. evalset-field-validation.md（子流程3）

**来源**：从主流程分支B提取

**内容**：
- YAML frontmatter
- 目标：校验字段状态，处理缺失字段，保存配置
- 触发条件：answer 字段全部非空
- 步骤B1：检查 model 字段状态
- 步骤B2：检查 case_id 字段状态
- 步骤B3：处理 model 字段为空（询问评测模式）
- 步骤B4：保存
- 返回：标准化阶段

**预估行数**：~40行

---

## 子流程 Frontmatter 规范

每个子流程的 YAML frontmatter 需符合规范：

### evalset-field-mapping.md

```yaml
---
name: evalset-field-mapping
description: Use when evalset-parse-process reaches step 3 and needs to generate field mapping configuration
---
```

### evalset-model-selection.md

```yaml
---
name: evalset-model-selection
description: Use when evalset-parse-process determines answer field is empty and needs model selection
---
```

### evalset-field-validation.md

```yaml
---
name: evalset-field-validation
description: Use when evalset-parse-process determines answer field is filled and needs field validation
---
```

---

## 调用与返回机制

### 主流程调用子流程

```markdown
## 步骤3：生成初始映射

**判断**：`evalset-fields-mapping.json` 是否存在？

| 状态 | 动作 |
|------|------|
| 已存在 | → 步骤4 |
| 不存在 | 执行 [evalset-field-mapping.md](./evalset-field-mapping.md) |

完成后 → 步骤4
```

### 子流程返回主流程

每个子流程末尾统一格式：
```markdown
---

**返回**：evalset-parse-process.md 步骤4 / 标准化阶段
```

---

## 功能一致性保证

| 检查项 | 保证方式 |
|--------|----------|
| 流程顺序不变 | 主流程保留步骤编号，子流程按调用顺序执行 |
| 禁止规则不丢失 | 主流程保留核心禁止规则，子流程补充特定规则 |
| 变量定义完整 | 主流程保留变量速查表，子流程引用主流程变量 |
| 命令不变 | 所有脚本命令保持原样，仅调整位置 |
| 用户交互不变 | 确认步骤、询问内容保持原样 |

---

## 实施步骤

1. 创建 `evalset-field-mapping.md`（从步骤3提取）
2. 创建 `evalset-model-selection.md`（从分支A提取）
3. 创建 `evalset-field-validation.md`（从分支B提取）
4. 编辑 `evalset-parse-process.md`（精简主流程，添加子流程调用入口）
5. 验证拆分结果（完整性、一致性检查）

---

## 验证清单

### 完整性检查

- [x] 所有原有步骤都有对应位置（主流程或子流程）
- [x] 所有脚本命令都已保留
- [x] 所有用户交互点都已保留
- [x] 所有禁止规则都已保留
- [x] 所有变量定义都已保留

### 一致性检查

- [x] 流程顺序与原文档一致
- [x] 门控逻辑与原文档一致
- [x] 分支触发条件与原文档一致
- [x] 返回点与原文档一致

### 逻辑检查

- [x] 子流程调用入口清晰
- [x] 子流程返回点明确
- [x] 无循环调用或死锁
- [x] 无遗漏的中间文件检查

---

## 实施结果

**行数统计**：

| 文件 | 行数 |
|------|------|
| evalset-parse-process.md（主流程） | 157行 |
| evalset-field-mapping.md（子流程1） | 86行 |
| evalset-model-selection.md（子流程2） | 73行 |
| evalset-field-validation.md（子流程3） | 80行 |
| **总计** | **396行** |

**原文档**：276行

**行数增加原因**：
1. 每个子流程文件都有完整的 YAML frontmatter
2. 每个子流程文件都有变量速查表
3. 每个子流程文件都有调用方/返回点说明

**拆分完成，功能一致性已验证。**

---

## 变量速查（供子流程引用）

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |
| `{ext}` | 文件扩展名 |
| `{python-env}` | Python环境变量前缀（Windows GBK 为 `PYTHONUTF8=1 `） |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |