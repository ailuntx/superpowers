# Codex 应用兼容性：工作树与完成技能适配

让超能力技能在 Codex 应用的沙盒化工作树环境中正常工作，同时不破坏现有 Claude Code 或 Codex CLI 的行为。

**工单：** PRI-823

## 动机

Codex 应用在其管理的 git 工作树内运行代理——这些工作树处于分离的 HEAD 状态，位于 `$CODEX_HOME/worktrees/` 下，并受 Seatbelt 沙盒限制，该沙盒会阻止 `git checkout -b`、`git push` 和网络访问。有三个超能力技能假设拥有不受限制的 git 访问权限：`using-git-worktrees` 会创建带有命名分支的手动工作树，`finishing-a-development-branch` 通过分支名进行合并/推送/PR 操作，而 `subagent-driven-development` 需要两者。

Codex CLI（开源终端工具）**没有**这个冲突——它没有内置的工作树管理功能。我们的手动工作树方法填补了那里的隔离空白。这个问题特指 Codex 应用。

## 实证发现

于 2026-03-23 在 Codex 应用中测试：

| 操作 | workspace-write 沙盒 | 完全访问沙盒 |
|---|---|---|
| `git add` | 正常 | 正常 |
| `git commit` | 正常 | 正常 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 正常 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 正常 |
| `gh pr create` | **被阻止**（网络） | 正常 |
| `git status/diff/log` | 正常 | 正常 |

额外发现：
- `spawn_agent` 子代理**共享**父线程的文件系统（通过标记文件测试确认）
- 无论工作树是从哪个分支启动的，应用标题栏都会显示“创建分支”按钮
- 应用的原生完成流程：创建分支 → 提交模态框 → 提交并推送 / 提交并创建 PR
- `network_access = true` 配置在 macOS 上静默失效（问题 #10390）

## 设计：只读环境检测

三个只读 git 命令可在无副作用的情况下检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

推导出两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` —— 代理位于由其他实体（Codex 应用、Claude Code Agent 工具、先前技能运行或用户）创建的工作树中
- **ON_DETACHED_HEAD：** `BRANCH` 为空 —— 不存在命名分支

为什么使用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者都解析到同一个 `.git` 目录
- 在链接的工作树中，`git-dir` 是 `.git/worktrees/<name>` 而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等 —— 避免了 `show-toplevel` 可能产生的误报
- 通过 `cd && pwd -P` 解析可以处理相对路径问题（`git-common-dir` 在普通仓库中返回相对路径 `.git`，但在工作树中返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接工作树？ | 分离 HEAD？ | 环境 | 操作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整的技能行为（不变） |
| 是 | 是 | Codex 应用工作树（workspace-write） | 跳过工作树创建；在完成时移交负载 |
| 是 | 否 | Codex 应用（完全访问）或手动工作树 | 跳过工作树创建；完整的完成流程 |
| 否 | 是 | 异常情况（手动分离 HEAD） | 正常创建工作树；在完成时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md` —— 添加步骤 0（约 12 行）

在“概述”和“目录选择流程”之间新增一节：

**步骤 0：检查是否已处于隔离工作空间**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，则完全跳过工作树创建。改为：
1. 跳转到创建步骤下的“运行项目设置”子节 —— `npm install` 等操作是幂等的，为安全起见值得运行
2. 然后“验证干净基线” —— 运行测试
3. 报告分支状态：
   - 在分支上：“已处于隔离工作空间 `<路径>`，位于分支 `<名称>`。测试通过。准备实施。”
   - 分离 HEAD：“已处于隔离工作空间 `<路径>`（分离 HEAD，外部管理）。测试通过。注意：完成时需要创建分支。准备实施。”

如果 `GIT_DIR == GIT_COMMON`，则继续完整的工作树创建流程（不变）。

当步骤 0 触发时，安全验证（.gitignore 检查）被跳过 —— 对于外部创建的工作树不相关。

更新集成部分的“被调用”条目。将每个条目的描述从特定于上下文的文本更改为：“确保隔离工作空间（创建一个或验证现有）”。例如，`subagent-driven-development` 条目从“必需：在开始前设置隔离工作空间”更改为“必需：确保隔离工作空间（创建一个或验证现有）”。

**沙盒回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能继续执行创建步骤，但 `git worktree add -b` 因权限错误（例如，Seatbelt 沙盒拒绝）而失败，则将此视为延迟检测到的受限环境。回退到步骤 0 的“已在工作空间”行为 —— 跳过创建，在当前目录中运行设置和基线测试，并相应报告。

在步骤 0 中报告后，**停止**。不要继续执行目录选择或创建步骤。

**其他一切不变：** 目录选择、安全验证、创建步骤、项目设置、基线测试、快速参考、常见错误、危险信号。

### 2. `finishing-a-development-branch/SKILL.md` —— 添加步骤 1.5 + 清理守卫（约 20 行）

**步骤 1.5：检测环境**（在步骤 1“验证测试”之后，步骤 2“确定基础分支”之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过步骤 2 和 3（不需要基础分支或选项）。
- **路径 B 和 C** 正常执行步骤 2（确定基础分支）和步骤 3（呈现选项）。

**路径 A —— 外部管理的工作树 + 分离 HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作都已暂存并提交（`git add` + `git commit`）。Codex 应用的完成控制操作依赖于已提交的工作。

然后向用户呈现以下内容（**不要**呈现 4 选项菜单）：

```
实施完成。所有测试通过。
当前 HEAD：<完整提交 SHA>

此工作空间由外部管理（分离 HEAD）。
我无法从此处创建分支、推送或打开 PR。

⚠ 这些提交位于分离的 HEAD 上。如果您不创建分支，
当此工作空间被清理时，它们可能会丢失。

如果您的宿主应用程序提供以下控制：
- “创建分支” —— 用于命名分支，然后提交/推送/PR
- “移交到本地” —— 用于将更改移动到您的本地检出

建议的分支名称：<工单ID/简短描述>
建议的提交消息：<工作摘要>
```

分支名称推导：如果可用则使用工单 ID（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个单词转换为 slug，否则省略建议。避免在分支名称中包含敏感内容（漏洞描述、客户名称）。

跳转到步骤 5（对于外部管理的工作树，清理是无操作）。

**路径 B —— 外部管理的工作树 + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常呈现 4 选项菜单。（步骤 5 的清理守卫将独立地重新检测外部管理状态。）

**路径 C —— 正常环境**（`GIT_DIR == GIT_COMMON`）：

像今天一样呈现 4 选项菜单（不变）。

**步骤 5 清理守卫：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 的检测（不要依赖先前的技能输出 —— 完成技能可能在不同的会话中运行）。如果 `GIT_DIR != GIT_COMMON`，则跳过 `git worktree remove` —— 宿主环境拥有此工作空间。

否则，像今天一样检查并移除。注意：现有的步骤 5 文本说“对于选项 1、2、4”，但快速参考表和常见错误部分说“仅选项 1 和 4”。新的守卫添加在现有逻辑之前，并且不改变哪些选项会触发清理。

**其他一切不变：** 选项 1-4 逻辑、快速参考、常见错误、危险信号。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` —— 各 1 行编辑

两个技能在集成部分有一行相同的描述。从：
```
- superpowers:using-git-worktrees - 必需：在开始前设置隔离工作空间
```
更改为：
```
- superpowers:using-git-worktrees - 必需：确保隔离工作空间（创建一个或验证现有）
```

**其他一切不变：** 调度/审查循环、提示模板、模型选择、状态处理、危险信号。

### 4. `codex-tools.md` —— 添加环境检测文档（约 15 行）

末尾新增两个部分：

**环境检测：**

```markdown
## 环境检测

创建工作树或完成分支的技能应在继续之前使用只读 git 命令检测其环境：

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → 已处于链接的工作树中（跳过创建）
- `BRANCH` 为空 → 分离 HEAD（无法从沙盒中分支/推送/PR）

有关每个技能如何使用这些信号，请参阅 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1.5。
```

**Codex 应用完成：**

```markdown
## Codex 应用完成

当沙盒阻止分支/推送操作时（在外部管理的工作树中处于分离 HEAD 状态），代理会提交所有工作并告知用户使用应用的原生控制：

- **“创建分支”** —— 命名分支，然后通过应用 UI 提交/推送/PR
- **“移交到本地”** —— 将工作转移到用户的本地检出

代理仍然可以运行测试、暂存文件，并输出建议的分支名称、提交消息和 PR 描述供用户复制。
```

## 不变的部分

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` —— 子代理提示未改动
- `executing-plans/SKILL.md` —— 仅 1 行集成描述更改（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` —— 无工作树或完成操作
- `.codex/INSTALL.md` —— 安装过程不变
- 4 选项完成菜单 —— 为 Claude Code 和 Codex CLI 完全保留
- 完整的工作树创建流程 —— 为非工作树环境完全保留
- 子代理调度/审查/迭代循环 —— 不变（文件系统共享已确认）

## 范围摘要

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（步骤 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（步骤 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件共添加/更改约 50 行。零新增文件。零破坏性变更。

## 未来考虑

如果第三个技能需要相同的检测模式，则将其提取到共享的 `references/environment-detection.md` 文件中（方法 B）。目前不需要 —— 只有 2 个技能使用它。

## 测试计划

### 自动化（在实施后于 Claude Code 中运行）

1. 普通仓库检测 —— 断言 IN_LINKED_WORKTREE=false
2. 链接工作树检测 —— `git worktree add` 测试工作树，断言 IN_LINKED_WORKTREE=true
3. 分离 HEAD 检测 —— `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 完成技能移交输出 —— 验证在受限环境中的移交消息（而非 4 选项菜单）
5. **步骤 5 清理守卫** —— 创建一个链接工作树（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入其中，运行步骤 5 清理检测（`GIT_DIR` 与 `GIT_COMMON`），断言它**不会**调用 `git worktree remove`。然后 `cd` 回到主仓库，运行相同的检测，断言它**会**调用 `git worktree remove`。之后清理测试工作树。

### 手动 Codex 应用测试（5 项测试）

1. 在工作树线程中的检测（workspace-write）—— 验证 GIT_DIR != GIT_COMMON，空分支
2. 在工作树线程中的检测（完全访问）—— 相同的检测，不同的沙盒行为
3. 完成技能移交格式 —— 验证代理发出移交负载，而非 4 选项菜单
4. 完整生命周期 —— 检测 → 提交 → 完成检测 → 正确行为 → 清理
5. **本地线程中的沙盒回退** —— 启动一个 Codex 应用**本地线程**（workspace-write 沙盒）。提示：“使用超能力技能 `using-git-worktrees` 为实施一个小变更设置隔离工作空间。” 预检查：`git checkout -b test-sandbox-check` 应失败并显示 `Operation not permitted`。预期：技能检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到步骤 0 的“已在工作空间”行为 —— 运行设置、基线测试，从当前目录报告就绪。通过：代理优雅恢复，没有晦涩的错误消息。失败：代理打印原始 Seatbelt 错误、重试或给出令人困惑的输出后放弃。

### 回归测试

- 现有的 Claude Code 技能触发测试仍然通过
- 现有的 subagent-driven-development 集成测试仍然通过
- 正常的 Claude Code 会话：完整的工作树创建 + 4 选项完成仍然有效
