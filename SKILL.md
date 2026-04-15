---
name: feishu-project
description: >
  飞书项目排期管理：查询周排期估分、修复超载、批量智能排期、
  排期预览甘特图、今日截止节点流转。
---

# 飞书项目排期助手

## 快速开始

首次使用前，复制配置模板并填入你的信息：

```bash
cp ref/config.example.yaml ref/config.yaml
# 编辑 ref/config.yaml 填入你的 user_key、project_key 等
```

配置项说明见 `ref/config.example.yaml`。

下文中 `MY_*` 变量均指 `ref/config.yaml` 中的对应值。

---

## 执行约束 — 必须通过飞书项目 MCP 操作

**MCP Server**：`feishu-project`（请确认你的 MCP server 标识与此一致，不一致请在 `ref/config.yaml` 中修改 `mcp_server` 字段）

**所有操作只能通过飞书项目 MCP 工具完成。禁止直接调用 HTTP API、bash 脚本或任何非 MCP 方式。**

| 操作类型 | 必用工具 |
|---------|---------|
| 查询排期 | `list_schedule` |
| 查询工作项列表 | `search_by_mql` |
| 查询单个工作项 | `get_workitem_brief` |
| 更新节点排期/估分 | `update_node` |
| 流转节点 | `transition_node` |
| 查询字段/节点配置 | `list_workitem_field_config` |
| 查询空间信息 | `search_project_info` |
| 查询用户 user_key | `search_user_info` |

MCP 调用失败时优先检查连接状态，不要尝试 HTTP 替代。

---

## 功能一：schedule — 查询周排期估分

**触发词**：本周估分、下周估分、排期情况、这周多少分

1. 确定查询范围（本周 = 当前自然周 Mon~Sun，下周同理）
2. 调用 `list_schedule`（`project_key=MY_PROJECT_KEY`，`user_keys=[MY_USER_KEY]`，`work_item_type_keys=["_all"]`）
3. 按估分计算规则汇总
4. 输出明细表格 + 总分

---

## 功能二：fix — 修复超载排期

**触发词**：修复排期、排期超载、帮我调整估分

1. 查询目标周排期（同 schedule）
2. 总分 ≤ 6 → 告知无需修复，结束
3. 总分 > 6 → 生成调整方案（小任务 ≤1 保持不动，优先缩减最大任务，目标总分 5.5）
4. **展示调整方案，等待用户确认后再执行**：

```
以下估分将被调整，请确认：

| 需求       | 当前估分 | 调整后 | 变化  |
|------------|---------|--------|-------|
| 大需求A    | 3.0     | 2.0    | -1.0  |
| 小需求B    | 0.5     | 0.5    | 不变  |

调整后总分：5.5
确认执行？
```

5. 用户确认后，调用 `update_node`（见「更新节点规范」）
6. 重新查询验证

---

## 功能三：batch-schedule — 批量智能排期

**触发词**：帮我排期、批量排期、安排排期、规划这周/下周的排期

输入：时间范围 + 任务列表（直接指定需求 或 从迭代自动筛选）

**执行前读取详细算法**：`ref/batch-schedule.md`

---

## 功能四：preview — 本周+下周排期预览

**触发词**：预览排期、看看排期、本周下周情况、甘特图、排期全景

输出：按迭代分组的文本甘特图 + 周总估分 + 临期预警

**执行前读取详细算法**：`ref/preview.md`

---

## 功能五：clear-today — 流转今日截止节点

**触发词**：清空今日排期、今日截止流转、流转今天到期的、今天的排期完了

### Step 1 — 查询今日截止节点

```sql
SELECT `name`, `work_item_id`
FROM `MY_PROJECT_NAME`.`story`
WHERE array_contains(in_progress_nodes_name(), 'MY_NODE_NAME')
  AND RELATIVE_DATETIME_EQ(get_node_attribute('MY_NODE_NAME', '__排期_结束时间'), 'today')
```

若结果为空，告知今日无截止节点，结束。

### Step 2 — 获取确认所需信息

并行执行：
1. `get_workitem_brief`（fields=["planning_sprint"]）批量查询所有命中工作项的所属迭代
2. `get_node_detail(work_item_id=<任意一项>, node_id_list=[])`（不传 node_id_list 即取全部节点），在返回的有序节点列表中找到 `MY_NODE_STATE_KEY` 后的下一个节点名

### Step 3 — 展示确认表，等待用户确认

```
以下节点即将流转，请确认：

| 需求名            | 所属迭代       | 当前节点 | 下一节点 |
|-------------------|---------------|---------|---------|
| 系列调研-报名明细 | 版本一         | FE开发  | FE联调  |
| 在线报名          | 版本一         | FE开发  | FE联调  |

确认流转以上 N 项？(全部确认 / 指定排除项)
```

等用户明确回复后再执行，不得自动跳过确认步骤。

### Step 4 — 执行流转

对每个确认的工作项调用 `transition_node`：

```
transition_node(
  work_item_id = <工作项ID>,
  node_id      = MY_NODE_STATE_KEY,
  action       = "confirm",
  project_key  = MY_PROJECT_KEY
)
```

### Step 5 — 报告结果

输出成功/失败汇总。若某项流转失败（如节点有必填项未完成），展示报错原因，其余项继续执行。

---

## 更新节点规范（每次写操作必须遵守）

**必须同时传 `node_schedule` + `schedules`，缺一不可：**

```
update_node(
  work_item_id  = <工作项ID>,
  node_id       = MY_NODE_STATE_KEY,
  project_key   = MY_PROJECT_KEY,
  node_schedule = {
    "estimate_start_date": <ms>,
    "estimate_end_date":   <ms>,
    "points": <估分>
  },
  schedules = [{
    "estimate_start_date": <ms>,
    "estimate_end_date":   <ms>,
    "owners": [MY_USER_KEY],
    "points": <估分>
  }]
)
```

| 错误做法 | 后果 |
|---------|------|
| 只传 `node_schedule.points`，不传日期 | 排期被清空，任务变"未排期" |
| 只传 `node_schedule`，不传 `schedules` | `list_schedule` 看不到，排期视图不更新 |
| 时间戳用错年份 | 任务被排到过去，视图里不可见 |

时间戳计算方法：`ref/timestamps.md`

---

## 错误恢复策略（所有写操作通用）

> 若批量操作中某项 MCP 调用失败，**记录错误原因，继续处理其余任务**。
> 最终报告中分 ✅ 成功 / ❌ 失败 / ⚠️ 跳过 三类汇总。

---

## 常见问题与纠错

**`list_schedule` 看不到刚更新的任务**
→ 只传了 `node_schedule`，补传 `schedules`（含 `owners: [MY_USER_KEY]`）

**任务排期变空/未排期**
→ `node_schedule` 缺日期，必须同时传 start + end + points

**任务不在当前视图（排到了过去）**
→ 时间戳年份错误；用 `ref/timestamps.md` 中的公式重算

**MQL 报 `attribute key or value error`**
→ 字段 key 错误，先调 `list_workitem_field_config` 确认。常见：迭代字段 key = `planning_sprint`

**MQL 返回空结果**
→ 不是报错；检查字段值大小写/全半角是否与飞书一致，工作项类型是否正确

**不知道 project_key** → `search_project_info` 传空间名
**node_id 报错** → `list_workitem_field_config` 查 state_key（格式 `state_XX`）
**不知道 user_key** → `search_user_info` 传姓名或邮箱
