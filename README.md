# LarkWorkWizard — 飞书项目排期助手

AI Agent Skill，通过飞书项目 MCP 管理你的节点排期与估分。

## 功能

| 命令 | 说明 |
|------|------|
| **schedule** | 查询本周/下周排期估分明细 |
| **fix** | 检测并修复超载排期（自动缩减估分至目标值） |
| **batch-schedule** | 批量智能排期（交错排列，自动分配估分与时间段） |
| **preview** | 文本甘特图预览本周+下周排期全景 |
| **clear-today** | 一键流转今日截止的节点 |

## 前置条件

- 已接入 **飞书项目 MCP Server**（skill 的所有操作通过 MCP 工具完成）
- Agent 环境支持 skill 加载（如 [OpenClaw](https://github.com/nicepkg/openclaw) 等）

## 安装

将本仓库克隆到你的 agent skills 目录：

```bash
git clone https://github.com/HuangMou0828/LarkWorkWizard.git ~/.openclaw/skills/feishu-project
```

## 配置

```bash
cd ~/.openclaw/skills/feishu-project
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

## 使用示例

对 agent 说：

- "本周估分多少" → 查询当前周排期
- "帮我排下周的排期" → 批量智能排期
- "看看排期" → 甘特图预览
- "排期超载了，帮我修复" → 自动缩减至目标值
- "今天的排期完了" → 流转今日截止节点

## 文件结构

```
├── SKILL.md                    # Agent 入口（agent 读这个）
├── README.md                   # 人类文档（你在读这个）
└── ref/
    ├── config.example.yaml     # 配置模板
    ├── batch-schedule.md       # 批量排期算法细节
    ├── preview.md              # 甘特图渲染规则
    └── timestamps.md           # 时间戳计算公式与校验表
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

## 贡献

欢迎 PR，以下方向特别欢迎：

- 新功能：超期检测（overdue）、待办汇总（todo）
- 节假日数据年度更新（`ref/holidays.yaml`）
- 更多角色的配置模板与实际验证
- Bug 修复与 MCP API 兼容性更新

## License

[MIT](LICENSE)
