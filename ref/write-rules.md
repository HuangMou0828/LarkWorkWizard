# 写操作规范

> 每次调用 `update_node` 或 `transition_node` 前必须遵守本文件的规则。

## update_node 参数规范

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

## 错误恢复策略

若批量操作中某项 MCP 调用失败，**记录错误原因，继续处理其余任务**。
最终报告中分 ✅ 成功 / ❌ 失败 / ⚠️ 跳过 三类汇总。

## 常见问题

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
