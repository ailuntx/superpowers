# Copilot CLI 工具映射

技能使用 Claude Code 工具名称。当在技能中遇到这些工具时，请使用您平台上的等效工具：

| 技能引用 | Copilot CLI 等效工具 |
|-----------------|----------------------|
| `Read`（文件读取） | `view` |
| `Write`（文件创建） | `create` |
| `Edit`（文件编辑） | `edit` |
| `Bash`（运行命令） | `bash` |
| `Grep`（搜索文件内容） | `grep` |
| `Glob`（按名称搜索文件） | `glob` |
| `Skill` 工具（调用技能） | `skill` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（分派子代理） | `task`（参见[代理类型](#代理类型)） |
| 多个 `Task` 调用（并行） | 多个 `task` 调用 |
| 任务状态/输出 | `read_agent`, `list_agents` |
| `TodoWrite`（任务跟踪） | `sql` 配合内置的 `todos` 表 |
| `WebSearch` | 无等效工具 — 使用 `web_fetch` 配合搜索引擎 URL |
| `EnterPlanMode` / `ExitPlanMode` | 无等效工具 — 保持在主会话中 |

## 代理类型

Copilot CLI 的 `task` 工具接受一个 `agent_type` 参数：

| Claude Code 代理 | Copilot CLI 等效工具 |
|-------------------|----------------------|
| `general-purpose` | `"general-purpose"` |
| `Explore` | `"explore"` |
| 命名插件代理（例如 `superpowers:code-reviewer`） | 从已安装的插件中自动发现 |

## 异步 Shell 会话

Copilot CLI 支持持久的异步 Shell 会话，这在 Claude Code 中没有直接等效功能：

| 工具 | 用途 |
|------|---------|
| `bash` 配合 `async: true` | 在后台启动一个长时间运行的命令 |
| `write_bash` | 向正在运行的异步会话发送输入 |
| `read_bash` | 从异步会话读取输出 |
| `stop_bash` | 终止异步会话 |
| `list_bash` | 列出所有活跃的 Shell 会话 |

## 其他 Copilot CLI 工具

| 工具 | 用途 |
|------|---------|
| `store_memory` | 持久化存储关于代码库的信息，供未来会话使用 |
| `report_intent` | 更新 UI 状态行以显示当前意图 |
| `sql` | 查询会话的 SQLite 数据库（待办事项、元数据） |
| `fetch_copilot_cli_documentation` | 查找 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（问题、拉取请求、代码搜索） |
