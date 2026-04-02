---
name: evalset-model-selection
description: Use when evalset-parse determines answer field is empty and needs model selection
---

# 模型选择流程

**调用方**：evalset-parse.md 步骤4（分支A）

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

## 变量速查

| 变量 | 说明 |
|------|------|
| `{work-dir}` | 当前工作目录 |
| `{session-id}` | 会话目录名，格式 `session-{8位字母数字}` |
| `{skill-dir}` | 技能安装目录 |
| `{python-env}` | Python环境变量前缀（Windows GBK 为 `PYTHONUTF8=1 `） |
| `{python-cmd}` | Python命令（`python` 或 `python3`） |

---

**返回**：标准化阶段