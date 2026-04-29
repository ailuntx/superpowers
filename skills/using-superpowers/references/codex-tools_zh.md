# Codex 工具映射

技能使用 Claude Code 工具名称。当你在技能中遇到这些时，请使用你平台的等效工具：

| 技能引用 | Codex 等效工具 |
|-----------------|------------------|
| `Task` 工具（调度子代理） | `spawn_agent`（参见[命名代理调度](#命名代理调度)） |
| 多个 `Task` 调用（并行） | 多个 `spawn_agent` 调用 |
| 任务返回结果 | `wait` |
| 任务自动完成 | `close_agent` 以释放槽位 |
| `TodoWrite`（任务跟踪） | `update_plan` |
| `Skill` 工具（调用技能） | 技能原生加载 — 只需遵循说明 |
| `Read`、`Write`、`Edit`（文件） | 使用你的原生文件工具 |
| `Bash`（运行命令） | 使用你的原生 shell 工具 |

## 子代理调度需要多代理支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这将启用 `spawn_agent`、`wait` 和 `close_agent`，用于像 `dispatching-parallel-agents` 和 `subagent-driven-development` 这样的技能。

## 命名代理调度

Claude Code 技能引用命名代理类型，如 `superpowers:code-reviewer`。
Codex 没有命名代理注册表 — `spawn_agent` 从内置角色（`default`、`explorer`、`worker`）创建通用代理。

当技能指示调度命名代理类型时：

1. 找到代理的提示文件（例如，`agents/code-reviewer.md` 或技能的本地提示模板，如 `code-quality-reviewer-prompt.md`）
2. 读取提示内容
3. 填充任何模板占位符（`{BASE_SHA}`、`{WHAT_WAS_IMPLEMENTED}` 等）
4. 使用填充的内容作为 `message` 生成一个 `worker` 代理

| 技能指令 | Codex 等效操作 |
|-------------------|------------------|
| `Task tool (superpowers:code-reviewer)` | `spawn_agent(agent_type="worker", message=...)`，其中 `message` 包含 `code-reviewer.md` 的内容 |
| `Task tool (general-purpose)` 带有内联提示 | `spawn_agent(message=...)`，其中 `message` 包含相同的提示 |

### 消息框架

`message` 参数是用户级别的输入，而不是系统提示。请为其构建结构以实现最大程度的指令遵循：

```
你的任务是执行以下操作。请严格按照以下说明执行。

<agent-instructions>
[从代理的 .md 文件填充的提示内容]
</agent-instructions>

现在执行此任务。仅输出遵循上述指令中指定格式的结构化响应。
```

- 使用任务委派框架（"你的任务是..."）而非角色框架（"你是..."）
- 将指令包裹在 XML 标签中 — 模型将带标签的块视为权威
- 以明确的执行指令结尾，以防止对指令进行总结

### 何时可以移除此变通方案

此方法是为了弥补 Codex 插件系统尚未支持 `plugin.json` 中的 `agents` 字段。当 `RawPluginManifest` 获得 `agents` 字段时，插件可以符号链接到 `agents/`（镜像现有的 `skills/` 符号链接），并且技能可以直接调度命名代理类型。

## 环境检测

创建 worktree 或完成分支的技能应在继续之前使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已处于链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → 分离的 HEAD（无法从沙箱中分支/推送/PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1，了解每个技能如何使用这些信号。

## Codex 应用完成

当沙箱阻止分支/推送操作（在外部管理的 worktree 中处于分离的 HEAD 状态）时，代理会提交所有工作并通知用户使用应用的原生控件：

- **"创建分支"** — 命名分支，然后通过应用 UI 提交/推送/PR
- **"移交到本地"** — 将工作转移到用户的本地检出

代理仍然可以运行测试、暂存文件，并输出建议的分支名称、提交消息和 PR 描述供用户复制。
