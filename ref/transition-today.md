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

---

# 功能六：clear-all — 批量流转今日截止节点（含额外节点）

> 在 `clear` 基础上，额外处理 `EXTRA_CLEAR_NODES`（config 中 `extra_clear_nodes` 字段，默认：联调、自测）。
> 额外节点区别在于：**有排期看排期，无排期默认今日到期**。

## Step A — 查询主节点（同 clear）

与 `clear` Step 1 相同，查询 `MY_NODE_NAME` 今日排期截止的工作项，记为集合 **S_main**。

## Step B — 查询额外节点

对 `EXTRA_CLEAR_NODES` 中每个节点名（如 `联调`、`自测`），分别执行两条 MQL 查询，**并行执行**：

**查询 B1 — 有排期且今日截止：**

```sql
SELECT `name`, `work_item_id`
FROM `MY_PROJECT_NAME`.`story`
WHERE array_contains(in_progress_nodes_name(), '<EXTRA_NODE_NAME>')
  AND RELATIVE_DATETIME_EQ(get_node_attribute('<EXTRA_NODE_NAME>', '__排期_结束时间'), 'today')
```

**查询 B2 — 无排期（排期结束时间为空）：**

```sql
SELECT `name`, `work_item_id`
FROM `MY_PROJECT_NAME`.`story`
WHERE array_contains(in_progress_nodes_name(), '<EXTRA_NODE_NAME>')
  AND get_node_attribute('<EXTRA_NODE_NAME>', '__排期_结束时间') IS NULL
```

将 B1 ∪ B2 的结果合并，对每个额外节点得到命中集合。去重后记为集合 **S_extra**。

## Step C — 动态解析节点 state_key

对 S_main ∪ S_extra 中取任意一个工作项，调用：

```
get_node_detail(work_item_id=<任意工作项ID>, node_id_list=[])
```

从返回的节点列表中，**仅按节点名匹配**找到：
- `MY_NODE_NAME`（主节点）的 `state_key` → 用于流转主节点
- 每个 `EXTRA_NODE_NAME`（额外节点）的 `state_key` → 用于流转对应节点

**禁止**用有序列表的位置推断"下一节点"——节点列表顺序 ≠ 工作流跳转路径，实际目标节点由飞书工作流引擎决定。

若某个额外节点名在列表中找不到，跳过该节点并在输出中备注"节点未找到"。

## Step D — 获取迭代信息

并行调用 `get_workitem_brief`（fields=["planning_sprint"]）批量查询 S_main ∪ S_extra 中所有工作项的所属迭代。

## Step E — 展示确认表，等待用户确认

将主节点和额外节点的命中工作项**合并**展示，按节点分组：

- **主节点**（MY_NODE_NAME）：显示"下一节点"列（从 Step 2 原有逻辑取）
- **额外节点**（联调、自测等）：**不显示"下一节点"**，改为备注"工作流决定"；无排期的工作项额外加备注"无排期"

```
以下节点即将流转，请确认：

【FE开发】
| 需求名    | 所属迭代 | 当前节点 | 下一节点 |
|-----------|---------|---------|---------|
| 在线报名  | 版本一   | FE开发  | 联调    |

【联调】
| 需求名                 | 所属迭代 | 当前节点 | 备注            |
|------------------------|---------|---------|----------------|
| 系列调研-报名明细       | 版本一   | 联调    | 工作流决定      |
| 工时统计-分析师详情页   | 版本二   | 联调    | 工作流决定·无排期 |

【自测】
（若无命中，展示"今日无截止节点"）

共 N 项，确认流转以上工作项？(全部确认 / 指定排除项)
```

等用户明确回复后再执行，不得自动跳过确认步骤。

## Step F — 执行流转

对每个确认的工作项，用对应节点的 `state_key` 调用 `transition_node`：

```
transition_node(
  work_item_id = <工作项ID>,
  node_id      = <该节点的 state_key>,
  action       = "confirm",
  project_key  = MY_PROJECT_KEY
)
```

## Step G — 报告结果

输出成功/失败汇总，按节点分组展示。失败项展示报错原因，其余项继续执行。
