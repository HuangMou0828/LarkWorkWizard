---
name: feishu-project
description: >
  飞书项目排期管理：查询周排期估分、修复超载、批量智能排期、
  排期预览甘特图、批量流转今日截止的工作节点到下一节点。
---

# 飞书项目排期助手

## 快速开始

首次使用前，复制配置模板并填入你的信息：

```bash
cp ref/config.example.yaml ref/config.yaml
```

配置项说明见 `ref/config.example.yaml`。下文中 `MY_*` 变量均指 `ref/config.yaml` 中的对应值。

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

**执行前读取详细步骤**：`ref/fix.md`

### batch-schedule — 批量智能排期

**触发词**：帮我排期、批量排期、安排排期、规划这周/下周的排期

**执行前读取详细算法**：`ref/batch-schedule.md`

### preview — 本周+下周排期预览

**触发词**：预览排期、看看排期、本周下周情况、甘特图、排期全景

**执行前读取详细算法**：`ref/preview.md`

### transition-today — 批量流转今日截止节点

**触发词**：流转今日截止的节点、今天到期的流转一下、批量流转今天的、今天开发完了帮我流转

> 通过 `transition_node` 将今日排期截止的节点流转到下一节点。不修改排期日期、不修改估分。

**执行前读取详细步骤**：`ref/transition-today.md`

---

## 写操作须知

执行 `update_node` 或 `transition_node` 前，**必须先读取**：`ref/write-rules.md`
