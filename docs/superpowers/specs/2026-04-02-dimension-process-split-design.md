# dimension-process.md 拆分设计

**日期**：2026-04-02
**状态**：待实现

---

## 背景

当前 `dimension-process.md` 文件存在以下问题：
- 204 行，包含两个独立流程（流程5 + 流程6）
- 通用说明内容（JSON 转化规则）过长
- 与项目中其他 process 文件的"单流程单文件"模式不一致

---

## 目标

1. **减少 Token 消耗**：每次调用只加载需要的流程
2. **提高可读性**：每个文件单一职责
3. **符合现有模式**：与其他 process 文件保持一致

---

## 设计方案

### 文件拆分结构

```
processes/
  dimension-case-level.md     # 流程5：定制用例级评测配置（~45行）
  dimension-general.md        # 流程6：通用维度级评测配置（~40行）

references/
  dimension-transform.md      # 维度配置结构转化规则（~50行）【新增】
```

### 删除文件

- `processes/dimension-process.md`（拆分后删除）

### 修改文件

- `eval-build.md`：更新流程速查表的链接

---

## 文件内容设计

### dimension-case-level.md

**frontmatter**：
```yaml
name: dimension-case-level
description: Use when user selects case-level evaluation and expert template matching failed
```

**内容结构**：
- 触发条件说明
- 步骤1：查阅内置模板
- 步骤2：评估模板适配情况（可调整/固定字段说明）
- 步骤3：保存维度配置（链接到 dimension-transform.md）
- 返回点

**预估行数**：45 行

### dimension-general.md

**frontmatter**：
```yaml
name: dimension-general
description: Use when user selects dimension-level evaluation and expert template matching failed
```

**内容结构**：
- 触发条件说明
- 步骤1：选择评测维度
- 步骤2：评估维度覆盖情况（三种情况处理）
- 步骤3：保存维度配置（权重设置 + 链接到 dimension-transform.md）
- 返回点

**预估行数**：40 行

### dimension-transform.md

**frontmatter**：
```yaml
name: dimension-transform
description: Reference for converting dimension templates to eval-dimension.json format
```

**内容结构**：
- 转化步骤（3步）
- 结构对比表
- 单维度转化示例
- 多维度合并示例
- 保存前检查清单

**预估行数**：50 行

---

## eval-build.md 修改

修改第 37-42 行的流程速查表：

**修改前**：
```markdown
| 流程5 | 定制用例级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程5定制用例级评测配置) | 任务2步骤3 |
| 流程6 | 通用维度级评测配置 | [dimension-process.md](./processes/dimension-process.md#流程6通用维度级评测配置) | 任务2步骤3 |
```

**修改后**：
```markdown
| 流程5 | 定制用例级评测配置 | [dimension-case-level.md](./processes/dimension-case-level.md) | 任务2步骤3 |
| 流程6 | 通用维度级评测配置 | [dimension-general.md](./processes/dimension-general.md) | 任务2步骤3 |
```

---

## 效果对比

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 单次加载行数 | 204 行 | 40-45 行 |
| 文件职责 | 混合 | 单一 |
| 通用说明位置 | process 内 | references |
| Token 节省 | - | ~75% |

---

## 实现步骤

1. 创建 `references/dimension-transform.md`
2. 创建 `processes/dimension-case-level.md`
3. 创建 `processes/dimension-general.md`
4. 修改 `eval-build.md` 流程速查表
5. 删除 `processes/dimension-process.md`