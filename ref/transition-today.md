# 批量流转今日截止节点 — 详细步骤

> 功能五 (transition-today) 的完整执行步骤。
> 写操作规范见 `ref/write-rules.md`。

## 明确语义

通过 `transition_node` MCP 调用，将今日排期截止的工作节点流转到下一个节点（如 FE开发 → FE联调）。**不修改排期日期、不修改估分。**

## Step 1 — 查询今日截止节点

```sql
SELECT `name`, `work_item_id`
FROM `MY_PROJECT_NAME`.`story`
WHERE array_contains(in_progress_nodes_name(), 'MY_NODE_NAME')
  AND RELATIVE_DATETIME_EQ(get_node_attribute('MY_NODE_NAME', '__排期_结束时间'), 'today')
```

若结果为空，告知今日无截止节点，结束。

## Step 2 — 获取确认所需信息

并行执行：
1. `get_workitem_brief`（fields=["planning_sprint"]）批量查询所有命中工作项的所属迭代
2. `get_node_detail(work_item_id=<任意一项>, node_id_list=[])`（不传 node_id_list 即取全部节点），在返回的有序节点列表中找到 `MY_NODE_STATE_KEY` 后的下一个节点名

## Step 3 — 展示确认表，等待用户确认

```
以下节点即将流转，请确认：

| 需求名            | 所属迭代       | 当前节点 | 下一节点 |
|-------------------|---------------|---------|---------|
| 系列调研-报名明细 | 版本一         | FE开发  | FE联调  |
| 在线报名          | 版本一         | FE开发  | FE联调  |

确认流转以上 N 项？(全部确认 / 指定排除项)
```

等用户明确回复后再执行，不得自动跳过确认步骤。

## Step 4 — 执行流转

对每个确认的工作项调用 `transition_node`：

```
transition_node(
  work_item_id = <工作项ID>,
  node_id      = MY_NODE_STATE_KEY,
  action       = "confirm",
  project_key  = MY_PROJECT_KEY
)
```

## Step 5 — 报告结果

输出成功/失败汇总。若某项流转失败（如节点有必填项未完成），展示报错原因，其余项继续执行。
