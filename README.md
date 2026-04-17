# LarkWorkWizard — 飞书项目排期助手

AI Agent Skill，通过飞书项目 MCP 管理你的节点排期与估分。

## 功能

直接用自然语言对 Agent 说，skill 会自动匹配并执行：

| 你可以说 | 功能 |
|---------|------|
| "本周估分多少" / "下周排期情况" | 查询周排期估分明细 |
| "排期超载了" / "帮我修复排期" | 检测并修复超载排期 |
| "帮我排下周的排期" / "批量排期" | 批量智能排期（交错排列） |
| "看看排期" / "甘特图" | 文本甘特图预览本周+下周全景 |
| "今天到期的流转一下" / "今天开发完了帮我流转" | 批量流转今日截止节点到下一节点 |

## 前置条件

- 已接入 **飞书项目 MCP Server**（skill 的所有操作通过 MCP 工具完成）
- Agent 环境支持 skill 加载

## 安装

### OpenClaw

```bash
git clone https://github.com/HuangMou0828/LarkWorkWizard.git ~/.openclaw/skills/lark-work-wizard
```

### Claude Code

```bash
git clone https://github.com/HuangMou0828/LarkWorkWizard.git ~/.claude/skills/lark-work-wizard
```

### Cursor

```bash
git clone https://github.com/HuangMou0828/LarkWorkWizard.git ~/.cursor/skills/lark-work-wizard
```

> 不同 agent 的 skill 目录可能不同，以上为常见约定路径。如果你的环境使用其他路径，克隆到对应目录即可。配置步骤相同。

## 配置

```bash
cd ~/.openclaw/skills/lark-work-wizard
cp ref/config.example.yaml ref/config.yaml
```

编辑 `ref/config.yaml`，填入你的信息：

```yaml
user_name: "你的姓名"
user_key: "你的飞书 user_key"
project_key: "你的空间 project_key"
project_name: "你的空间名称"
node_name: "你负责的节点（如 FE开发、BE开发、QA测试）"
node_state_key: "对应的 state_key（如 state_59）"
```

> **如何获取这些值？** 见 `ref/config.example.yaml` 中的注释说明，每个字段都标注了对应的 MCP 查询方式。

## 使用方式

不需要特殊命令或前缀，直接用自然语言和 Agent 对话即可。Skill 通过语义匹配自动触发，例如：

```
你：本周估分多少？
你：帮我把这个迭代的需求排到下周
你：看看排期
你：排期超载了，帮我调一下
你：今天到期的流转一下
```

## 文件结构

```
├── SKILL.md                    # Agent 入口 — 功能路由（agent 首次读这个）
├── README.md                   # 人类文档（你在读这个）
└── ref/                        # 按需加载 — agent 触发具体功能时才读取
    ├── config.example.yaml     # 配置模板
    ├── fix.md                  # 修复超载排期步骤
    ├── batch-schedule.md       # 批量排期算法
    ├── preview.md              # 甘特图渲染规则
    ├── transition-today.md     # 批量流转节点步骤
    ├── write-rules.md          # 写操作规范 + 常见问题
    └── timestamps.md           # 时间戳计算公式
```

## 适配你的角色

`config.yaml` 中的 `node_name` 决定了你操作的节点，常见角色：

| 角色 | node_name 示例 |
|------|---------------|
| 前端开发 | `FE开发` |
| 后端开发 | `BE开发` / `RD开发` |
| 测试 | `QA测试` |
| 产品 | `产品设计` |
| 设计 | `UI设计` |

`node_state_key` 需通过 `list_workitem_field_config` 查询获取，每个空间不同。

## 验证（不修改飞书数据）

首次使用或修改 skill 后，建议先验证再用于实际操作。

### 第一步：验证只读功能

以下功能只查询、不写入，可以放心运行：

```
你：本周估分多少？          ← 验证 schedule（查询排期）
你：看看排期                ← 验证 preview（甘特图渲染）
```

检查返回结果是否与飞书项目中看到的一致。

### 第二步：dry-run 验证写入功能

对任何写入操作加上 **"dry-run"**，skill 会完整执行查询和计算，但**跳过实际写入**，只展示将要执行的操作：

```
你：dry-run 帮我排下周的排期       ← 验证 batch-schedule 算法
你：dry-run 修复排期               ← 验证 fix 调整方案
你：dry-run 流转今日截止的节点      ← 验证 transition-today 查询
```

dry-run 模式下你会看到完整的确认表格和操作计划，但不会调用 `update_node` 或 `transition_node`。确认输出符合预期后，去掉 "dry-run" 重新执行即可。

### 验证清单

| 检查项 | 怎么验证 |
|--------|---------|
| config.yaml 配置正确 | `schedule` 能查到你的排期数据 |
| MQL 查询正常 | `preview` 能按迭代分组显示 |
| 估分计算准确 | 对比 `schedule` 结果与飞书项目页面 |
| 时间戳无误 | `dry-run batch-schedule` 的日期与预期一致 |
| 节点流转目标正确 | `dry-run transition-today` 显示的下一节点正确 |

## 贡献

欢迎 PR，以下方向特别欢迎：

- 新功能：超期检测（overdue）、待办汇总（todo）
- 节假日数据年度更新（`ref/holidays.yaml`）
- 更多角色的配置模板与实际验证
- Bug 修复与 MCP API 兼容性更新

## License

[MIT](LICENSE)
