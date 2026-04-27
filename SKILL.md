---
name: lark-work-wizard
description: >
  飞书项目排期管理：查询周排期估分、修复超载、批量智能排期、
  排期预览甘特图、批量流转今日截止的工作节点到下一节点。
---

# LarkWorkWizard — 飞书项目排期助手

## 快速开始

首次使用前，复制配置模板并填入你的信息：

```bash
cp ref/config.example.yaml ref/config.yaml
```

配置项说明见 `ref/config.example.yaml`。下文中 `MY_*` 变量均指 `ref/config.yaml` 中的对应值。

---

## Args 快捷路由

通过 `/lark-work-wizard <cmd>` 调用时，**args 优先**，直接执行对应功能，无需匹配触发词：

| 命令 | 说明 | 示例 |
|------|------|------|
| `week [this-week\|next-week\|自然语言]` | 查询周排期估分，默认本周 | `week` / `week next-week` / `week 下周` |
| `fix [this-week\|next-week\|自然语言]` | 修复指定周的超载排期，默认本周 | `fix` / `fix next-week` / `fix 下周` |
| `batch [this-week\|next-week\|自然语言]` | 批量智能排期指定周，默认本周 | `batch` / `batch next-week` / `batch 下周` |
| `preview` | 本周+下周排期预览 | `preview` |
| `clear` | 批量流转今日截止节点 | `clear` |
| `clear-all` | 批量流转今日截止节点（含联调、自测额外节点） | `clear-all` |

未传 args 时，回退到下方触发词路由。

---

## 执行约束

- **MCP Server**：`feishu-project`（以 `ref/config.yaml` 中 `mcp_server` 为准）
- **所有操作只能通过飞书项目 MCP 工具完成**，禁止 HTTP API 或 bash 替代
- MCP 调用失败时优先检查连接状态

---

## Dry-run 模式

用户指令包含 **"dry-run"** 时：查询正常执行，**跳过所有写入**（`update_node`、`transition_node`），输出末尾标注 `🔒 dry-run 模式，未实际执行`。

---

## 功能路由

### schedule — 查询周排期估分

**触发词**：本周估分、下周估分、排期情况、这周多少分

1. 确定查询范围（本周 = 当前自然周 Mon~Sun，下周同理）
2. 调用 `list_schedule`（`project_key=MY_PROJECT_KEY`，`user_keys=[MY_USER_KEY]`，`work_item_type_keys=["_all"]`）
3. 按估分计算规则汇总 → 输出明细表格 + 总分

### fix — 修复超载排期

**触发词**：修复排期、排期超载、帮我调整估分

**时间范围**：从用户指令中提取本周 / 下周；args 模式下从 `this-week` / `next-week` 参数读取；缺省时询问用户确认。

**执行前读取详细步骤**：`ref/fix.md`

### batch-schedule — 批量智能排期

**触发词**：帮我排期、批量排期、安排排期、规划这周/下周的排期

**时间范围**：从用户指令中提取本周 / 下周；args 模式下从 `this-week` / `next-week` 参数读取；缺省时询问用户确认。

**执行前读取详细算法**：`ref/batch-schedule.md`

### preview — 本周+下周排期预览

**触发词**：预览排期、看看排期、本周下周情况、甘特图、排期全景

**执行前读取详细算法**：`ref/preview.md`

### transition-today — 批量流转今日截止节点

**触发词**：流转今日截止的节点、今天到期的流转一下、批量流转今天的、今天开发完了帮我流转

> 通过 `transition_node` 将今日排期截止的节点流转到下一节点。不修改排期日期、不修改估分。

**执行前读取详细步骤**：`ref/transition-today.md`（功能五）

### transition-today-all — 批量流转今日截止节点（含额外节点）

**触发词**：clear-all、全部流转、联调自测也一起流转、今天所有节点都流转

> 在 `transition-today` 基础上，额外处理 `extra_clear_nodes` 配置的节点（联调、自测）。
> 额外节点：有排期看排期是否今日到期，无排期直接视为今日到期。

**执行前读取详细步骤**：`ref/transition-today.md`（功能六 clear-all）

---

## 写操作须知

执行 `update_node` 或 `transition_node` 前，**必须先读取**：`ref/write-rules.md`
