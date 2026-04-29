# Superpowers 版本说明

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入** — Copilot CLI v1.0.11 在 sessionStart 钩子输出中添加了对 `additionalContext` 的支持。session-start 钩子现在会检测 `COPILOT_CLI` 环境变量并输出 SDK 标准的 `{ "additionalContext": "..." }` 格式，为 Copilot CLI 用户在会话开始时提供完整的 superpowers 引导。(原始修复由 @culinablaz 在 PR #910 中提交)
- **工具映射** — 添加了 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具等价表
- **技能和 README 更新** — 在 `using-superpowers` 技能的平台说明和 README 安装部分添加了 Copilot CLI

### OpenCode 修复

- **技能路径一致性** — 引导文本不再宣传与实际运行时路径不匹配的误导性 `configDir/skills/superpowers/` 路径。代理应使用原生的 `skill` 工具，而不是通过路径导航到文件。测试现在使用源自单一事实来源的一致路径。(#847, #916)
- **引导作为用户消息** — 将引导注入从 `experimental.chat.system.transform` 移动到 `experimental.chat.messages.transform`，将其添加到第一条用户消息之前，而不是添加系统消息。避免了因每轮重复系统消息导致的令牌膨胀 (#750)，并修复了与 Qwen 等因多个系统消息而中断的模型的兼容性 (#894)。

## v5.0.6 (2026-03-24)

### 内联自检取代子代理审查循环

子代理审查循环（派遣新代理审查计划/规格）使执行时间翻倍（约 25 分钟开销），但并未显著提高计划质量。对 5 个版本各进行 5 次试验的回归测试显示，无论是否运行审查循环，质量得分都相同。

- **brainstorming** — 将规格审查循环（子代理派遣 + 3 次迭代上限）替换为内联规格自检清单：占位符扫描、内部一致性、范围检查、模糊性检查
- **writing-plans** — 将计划审查循环（子代理派遣 + 3 次迭代上限）替换为内联自检清单：规格覆盖、占位符扫描、类型一致性
- **writing-plans** — 添加了明确的“无占位符”部分，定义了计划失败情况（待定、模糊描述、未定义引用、“类似于任务 N”）
- 自检每次运行在约 30 秒内捕获 3-5 个真实错误，而不是约 25 分钟，缺陷率与子代理方法相当

### Brainstorm 服务器

- **会话目录重构** — brainstorm 服务器会话目录现在包含两个同级子目录：`content/`（提供给浏览器的 HTML 文件）和 `state/`（事件、服务器信息、pid、日志）。之前，服务器状态和用户交互数据与提供的内容存储在一起，使得它们可以通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在服务器启动的 JSON 中。（由 吉田仁 报告）

### 错误修复

- **所有者-PID 生命周期修复** — brainstorm 服务器的所有者-PID 监控存在两个导致 60 秒内错误关闭的错误：(1) 跨用户 PID（Tailscale SSH 等）的 EPERM 被当作“进程已死”处理，以及 (2) 在 WSL 上，祖进程 PID 解析为一个短暂存在的子进程，该进程在第一次生命周期检查之前就退出了。通过将 EPERM 视为“存活”并在启动时验证所有者 PID 来修复 — 如果它已经死亡，则禁用监控，服务器依赖 30 分钟空闲超时。这也从 `start-server.sh` 中移除了 Windows/MSYS2 特定的例外，因为服务器现在可以通用地处理它。(#879)
- **writing-skills** — 更正了关于 SKILL.md 前置元数据“仅支持两个字段”的错误说法；现在改为“两个必填字段”，并链接到 agentskills.io 规范以获取所有支持的字段（由 @arittr 在 PR #882 中提交）

### Codex 应用兼容性

- **codex-tools** — 添加了命名代理派遣映射，记录了如何将 Claude Code 的命名代理类型转换为 Codex 的 `spawn_agent` 与工作角色（由 @arittr 在 PR #647 中提交）
- **codex-tools** — 添加了环境检测和 Codex 应用完成部分，用于工作树感知技能（由 @arittr 提交）
- **设计规格** — 添加了 Codex 应用兼容性设计规格 (PRI-823)，涵盖只读环境检测、工作树安全技能行为和沙箱回退模式（由 @arittr 提交）

## v5.0.5 (2026-03-17)

### 错误修复

- **Brainstorm 服务器 ESM 修复** — 将 `server.js` 重命名为 `server.cjs`，以便在 Node.js 22+ 上正确启动 brainstorming 服务器，因为根目录的 `package.json` 中的 `"type": "module"` 导致 `require()` 失败。（由 @sarbojitrana 在 PR #784 中提交，修复 #774, #780, #783）
- **Brainstorm 所有者-PID 在 Windows 上** — 在 Windows/MSYS2 上跳过 PID 生命周期监控，因为 Node.js 无法看到 PID 命名空间，防止服务器在 60 秒后自行终止。(#770，文档来自 @lucasyhzlu-debug 的 PR #768)
- **stop-server.sh 可靠性** — 在报告成功之前验证服务器进程确实已终止。SIGTERM + 2 秒等待 + SIGKILL 回退。(#723)

### 变更

- **执行交接** — 在计划编写后恢复用户在子代理驱动和内联执行之间的选择。推荐子代理驱动，但不再是强制性的。

## v5.0.4 (2026-03-16)

### 审查循环优化

通过消除不必要的审查轮次和收紧审查者关注点，显著减少令牌使用并加速规格和计划审查。

- **单次完整计划审查** — 计划审查者现在一次性审查完整计划，而不是分块审查。移除了所有与块相关的概念（`## Chunk N:` 标题、1000 行块限制、每块派遣）。
- **提高了阻塞问题的门槛** — 规格和计划审查者提示现在都包含“校准”部分：仅标记会在实施过程中导致真实问题的问题。次要措辞、风格偏好和格式挑剔不应阻止批准。
- **减少了最大审查迭代次数** — 规格和计划审查循环都从 5 次减少到 3 次。如果审查者校准正确，3 轮就足够了。
- **精简了审查者清单** — 规格审查者从 7 个类别精简到 5 个；计划审查者从 7 个精简到 4 个。移除了以格式为重点的检查（任务语法、块大小），转而关注实质内容（可构建性、规格对齐）。

### OpenCode

- **单行插件安装** — OpenCode 插件现在通过 `config` 钩子自动注册技能目录。无需符号链接或 `skills.paths` 配置。安装只需在 `opencode.json` 中添加一行。(PR #753)
- **添加了 `package.json`**，以便 OpenCode 可以从 git 将 superpowers 作为 npm 包安装。

### 错误修复

- **验证服务器确实已停止** — `stop-server.sh` 现在在报告成功之前确认进程已死亡。SIGTERM + 2 秒等待 + SIGKILL 回退。如果进程存活，则报告失败。(PR #751)
- **通用代理语言** — brainstorm 伴侣等待页面现在说“代理”而不是“Claude”。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor 钩子** — 添加了 `hooks/hooks-cursor.json`，包含 Cursor 的驼峰格式（`sessionStart`, `version: 1`），并更新了 `.cursor-plugin/plugin.json` 以引用它。修复了 `session-start` 中的平台检测，优先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。（基于 PR #709）

### 错误修复

- **停止在 `--resume` 时触发 SessionStart 钩子** — 启动钩子在恢复的会话上重新注入上下文，而这些会话的对话历史中已经包含上下文。钩子现在仅在 `startup`、`clear` 和 `compact` 时触发。
- **Bash 5.3+ 钩子挂起** — 在 `hooks/session-start` 中用 `printf` 替换了 heredoc（`cat <<EOF`）。修复了 macOS 上 Homebrew bash 5.3+ 因 heredoc 中大变量扩展的 bash 回归导致的无限挂起。(#572, #571)
- **POSIX 安全的钩子脚本** — 在 `hooks/session-start` 中用 `$0` 替换了 `${BASH_SOURCE[0]:-$0}`。修复了 Ubuntu/Debian 上 `/bin/sh` 是 dash 时的“Bad substitution”错误。(#553)
- **可移植的 shebang** — 在所有 shell 脚本中用 `#!/usr/bin/env bash` 替换了 `#!/bin/bash`。修复了在 NixOS、FreeBSD 和带有 Homebrew bash 的 macOS 上执行的问题，这些系统中 `/bin/bash` 已过时或缺失。(#700)
- **Brainstorm 服务器在 Windows 上** — 自动检测 Windows/Git Bash（`OSTYPE=msys*`, `MSYSTEM`）并切换到前台模式，修复了因 `nohup`/`disown` 进程回收导致的静默服务器故障。(#737)
- **Codex 文档修复** — 在 Codex 文档中用 `multi_agent` 替换了已弃用的 `collab` 标志。(PR #749)

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm 服务器

**移除了所有打包的 node_modules — server.js 现在完全自包含**

- 用零依赖的 Node.js 服务器替换了 Express/Chokidar/WebSocket 依赖，使用内置的 `http`、`fs` 和 `crypto` 模块
- 移除了约 1,200 行打包的 `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 帧、ping/pong、正确的关闭握手）
- 原生 `fs.watch()` 文件监视替换了 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监视和集成测试

### Brainstorm 服务器可靠性

- **空闲 30 分钟后自动退出** — 当没有客户端连接时，服务器关闭，防止孤儿进程
- **所有者进程跟踪** — 服务器监控父进程 PID，并在所属会话死亡时退出
- **活跃度检查** — 技能在重用现有实例之前验证服务器是否响应
- **编码修复** — 在提供的 HTML 页面上正确设置 `<meta charset="utf-8">`

### 子代理上下文隔离

- 所有委派技能（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在都包含上下文隔离原则
- 子代理仅接收他们需要的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规性

**Brainstorm-server 移至技能目录**

- 将 `lib/brainstorm-server/` 移动到 `skills/brainstorming/scripts/`，符合 [agentskills.io](https://agentskills.io) 规范
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用替换为相对的 `scripts/` 路径
- 技能现在完全跨平台可移植 — 无需平台特定的环境变量来定位脚本
- `lib/` 目录已移除（是最后剩余的内容）

### 新功能

**Gemini CLI 扩展**

- 通过仓库根目录的 `gemini-extension.json` 和 `GEMINI.md` 提供原生 Gemini CLI 扩展支持
- `GEMINI.md` 在会话开始时 @imports `using-superpowers` 技能和工具映射表
- Gemini CLI 工具映射参考（`skills/using-superpowers/references/gemini-tools.md`）— 将 Claude Code 工具名称（Read、Write、Edit、Bash 等）转换为 Gemini CLI 等效项（read_file、write_file、replace 等）
- 记录 Gemini CLI 限制：无子代理支持，技能回退到 `executing-plans`
- 扩展根目录位于仓库根目录以实现跨平台兼容性（避免 Windows 符号链接问题）
- 安装说明已添加到 README

### 改进

**多平台 brainstorm 服务器启动**

- visual-companion.md 中的每平台启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（`--foreground` 与 `is_background`）以及其他环境的回退
- 服务器现在将启动 JSON 写入 `$SCREEN_DIR/.server-info`，以便代理即使在后台执行隐藏 stdout 时也能找到 URL 和端口

**Brainstorm 服务器依赖项打包**

- 将 `node_modules` 打包到仓库中，以便 brainstorm 服务器在全新插件安装后立即工作，无需在运行时安装 `npm`
- 从打包的依赖项中移除了 `fsevents`（仅限 macOS 的原生二进制文件；chokidar 在没有它的情况下优雅回退）
- 如果 `node_modules` 缺失，通过 `npm install` 进行回退自动安装

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（之前错误地映射到 `update_plan`）；已根据 OpenCode 源代码验证

### 错误修复

**Windows/Linux：单引号破坏 SessionStart 钩子** (#577, #529, #644, PR #585)

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围的单引号在 Windows 上失败（cmd.exe 不将单引号识别为路径分隔符），在 Linux 上也失败（单引号阻止变量扩展）
- 修复：用转义的双引号替换单引号 — 在 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux 上均可工作，无论路径中是否有空格
- 已在 Windows 11（NT 10.0.26200.0）上通过 Claude Code 2.1.72 和 Git for Windows 验证

**Brainstorming 规格审查循环被跳过** (#677)

- 规格审查循环（派遣 spec-document-reviewer 子代理，迭代直到批准）存在于散文“设计之后”部分，但在清单和流程图中缺失
- 由于代理遵循图表和清单比散文更可靠，规格审查步骤被完全跳过
- 在清单中添加了步骤 7（规格审查循环）以及相应的节点到点图
- 使用 `claude --plugin-dir` 和 `claude-session-driver` 测试：工作器现在正确派遣审查者

**Cursor 安装命令** (PR #676)

- 修复了 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（通过 Cursor 2.5 发布公告确认）

**Brainstorming 中的用户审查门** (#565)

- 在规格完成和 writing-plans 交接之间添加了明确的用户审查步骤
- 用户必须在实施计划开始之前批准规格
- 清单、流程和散文已更新，包含新门

**Session-start 钩子每个平台仅发出一次上下文**

- 钩子现在检测它是在 Claude Code 还是其他平台上运行
- 为 Claude Code 发出 `hookSpecificOutput`，为其他平台发出 `additional_context` — 防止双重上下文注入

**令牌分析脚本中的 linting 修复**

- `tests/claude-code/analyze-token-usage.py` 中的 `except:` → `except Exception:`

### 维护

**移除了死代码**

- 删除了 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）— 自 2026 年 2 月起未使用
- 从 `tests/opencode/test-plugin-loading.sh` 中移除了 skills-core 存在性检查

### 社区

- @karuturi — Claude Code 官方市场安装说明 (PR #610)
- @mvanhorn — session-start 钩子双重发出修复，OpenCode 工具映射修复
- @daniel-graham — 裸 except 的 linting 修复
- PR #585 作者 — Windows/Linux 钩子引号修复

---

## v5.0.0 (2026-03-09)

### 破坏性变更

**规格和计划目录重构**

- 规格（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- 计划（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 用户对规格/计划位置的偏好会覆盖这些默认值
- 所有内部技能引用、测试文件和示例路径已更新以匹配
- 迁移：如果需要，将现有文件从 `docs/plans/` 移动到新位置

**在支持的马具上强制使用子代理驱动开发**

Writing-plans 不再提供子代理驱动和 executing-plans 之间的选择。在支持子代理的马具（Claude Code、Codex）上，子代理驱动开发是必需的。Executing-plans 保留给不支持子代理能力的马具，现在会告诉用户 Superpowers 在支持子代理的平台上效果更好。

**Executing-plans 不再分批执行**

移除了“执行 3 个任务然后停止审查”的模式。计划现在连续执行，仅在遇到阻塞时停止。

**斜杠命令已弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示弃用通知，指向用户使用相应的技能。命令将在下一个主要版本中移除。

### 新功能

**可视化 brainstorming 伴侣**

用于 brainstorming 会话的可选基于浏览器的伴侣。当主题受益于可视化时，brainstorming 技能会提供在浏览器窗口中与终端对话一起显示线框图、图表、比较和其他内容。

- `lib/brainstorm-server/` — 带有浏览器辅助库、会话管理脚本和深色/浅色主题框架模板（“Superpowers Brainstorming”带 GitHub 链接）的 WebSocket 服务器
- `skills/brainstorming/visual-companion.md` — 服务器工作流、屏幕创作和反馈收集的渐进式披露指南
- Brainstorming 技能在其流程中添加了一个可视化伴侣决策点：在探索项目上下文后，技能评估即将到来的问题是否涉及视觉内容，并在其自己的消息中提供伴侣
- 每个问题的决策：即使在接受后，每个问题也会评估浏览器还是终端更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审查系统**

使用子代理派遣的规格和计划文档的自动化审查循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md` — 审查者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` — 审查者检查规格对齐、任务分解、文件结构和文件大小
- Brainstorming 在编写设计文档后派遣规格审查者
- Writing-plans 包括每个部分后的基于块的计划审查循环
- 审查循环重复直到批准或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计规格和实施计划

**跨技能管道的架构指导**

为 brainstorming、writing-plans 和 subagent-driven-development 添加了隔离设计和文件大小意识指导：

- **Brainstorming** — 新部分：“为隔离和清晰度设计”（清晰边界、定义良好的接口、独立可测试单元）和“在现有代码库中工作”（遵循现有模式，仅进行有针对性的改进）
- **Writing-plans** — 新“文件结构”部分：在定义任务之前映射文件和职责。新“范围检查”后盾：捕获应在 brainstorming 期间分解的多子系统规格
- **SDD 实施者** — 新“代码组织”部分（遵循计划的文件结构，报告对增长文件的担忧）和“当你力不从心时”升级指导
- **SDD 代码质量审查者** — 现在检查架构、单元分解、计划一致性和文件增长
- **规格/计划审查者** — 架构和文件大小已添加到审查标准中
- **范围评估** — Brainstorming 现在评估项目是否过大而无法用单个规格处理。多子系统请求会被早期标记并分解为子项目，每个子项目都有自己的规格 → 计划 → 实施周期

**子代理驱动开发改进**

- **模型选择** — 按任务类型选择模型能力的指导：机械实施用廉价模型，集成用标准模型，架构和审查用强大模型
- **实施者状态协议** — 子代理现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。控制器适当处理每个状态：用更多上下文重新派遣、升级模型能力、分解任务或升级给人类

### 改进

**指令优先级层次结构**

为 using-superpowers 添加了明确的优先级排序：

1. 用户的明确指令（CLAUDE.md、AGENTS.md、直接请求）— 最高优先级
2. Superpowers 技能 — 覆盖默认系统行为
3. 默认系统提示 — 最低优先级

如果 CLAUDE.md 或 AGENTS.md 说“不要使用 TDD”而技能说“始终使用 TDD”，则用户的指令优先。

**SUBAGENT-STOP 门**

在 using-superpowers 中添加了 `<SUBAGENT-STOP>` 块。为特定任务派遣的子代理现在跳过技能，而不是激活 1% 规则并调用完整的技能工作流。

**多平台改进**

- Codex 工具映射移动到渐进式披露参考文件（`references/codex-tools.md`）
- 添加了平台适应指针，以便非 Claude-Code 平台可以找到工具等效项
- 计划标题现在针对“代理工作器”而不是特定的“Claude”
- 协作功能要求在 `docs/README.codex.md` 中记录

**Writing-plans 模板更新**

- 计划步骤现在使用复选框语法（`- [ ] **Step N:**`）进行进度跟踪
- 计划标题引用 subagent-driven-development 和 executing-plans，并具有平台感知的路由

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支持**

Superpowers 现在与 Cursor 的插件系统兼容。包括 `.cursor-plugin/plugin.json` 清单和 README 中的 Cursor 特定安装说明。SessionStart 钩子输出现在除了现有的 `hookSpecificOutput.additionalContext` 外，还包括一个 `additional_context` 字段，以兼容 Cursor 钩子。

### 修复

**Windows：恢复了多语言包装器以实现可靠的钩子执行** (#518, #504, #491, #487, #466, #440)

Claude Code 在 Windows 上的 `.sh` 自动检测会在钩子命令前添加 `bash`，破坏了执行。修复：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名），以便自动检测不会干扰
- 恢复了 `run-hook.cmd` 多语言包装器，具有多位置 bash 发现（标准 Git for Windows 路径，然后是 PATH 回退）
- 如果未找到 bash，则静默退出而不是报错
- 在 Unix 上，包装器通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（适用于 dash/sh，而不仅仅是 bash）

这修复了 Windows 上因路径空格、缺少 WSL、MSYS 上 `set -euo pipefail` 脆弱性以及反斜杠错误处理导致的 SessionStart 失败。

## v4.3.0 (2026-02-12)

此修复应显著提高 superpowers 技能合规性，并应减少 Claude 无意中进入其原生计划模式的机会。

### 变更

**Brainstorming 技能现在强制执行其工作流程，而不是描述它**

模型跳过设计阶段，直接跳转到实施技能如 frontend-design，或将整个 brainstorming 过程折叠成单个文本块。该技能现在使用硬门、强制清单和 graphviz 流程来强制执行合规性：

- `<HARD-GATE>`：在呈现设计并获得用户批准之前，不得使用实施技能、代码或脚手架
- 必须作为任务创建并按顺序完成的明确清单（6 项）
- 以 `writing-plans` 为唯一有效终止状态的 Graphviz 流程
- 针对“这太简单了，不需要设计”的反模式点名 — 模型用来跳过此过程的合理化
- 基于部分复杂性而非项目复杂性的设计部分大小调整

**Using-superpowers 工作流图拦截 EnterPlanMode**

在技能流程图中添加了 `EnterPlanMode` 拦截。当模型即将进入 Claude 的原生计划模式时，它会检查是否进行了 brainstorming，并改为通过 brainstorming 技能路由。计划模式永远不会进入。

### 修复

**SessionStart 钩子现在同步运行**

将 hooks.json 中的 `async: true` 更改为 `async: false`。当异步时，钩子可能在模型的第一个回合之前未能完成，这意味着 using-superpowers 指令在第一条消息的上下文中不存在。

## v4.2.0 (2026-02-05)

### 破坏性变更

**Codex：用原生技能发现替换了引导 CLI**

移除了 `superpowers-codex` 引导 CLI、Windows `.cmd` 包装器和相关的引导内容文件。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，因此旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只需克隆 + 符号链接（在 INSTALL.md 中记录）。无需 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows：修复了 Claude Code 2.1.x 钩子执行** (#331)

Claude Code 2.1.x 改变了钩子在 Windows 上的执行方式：它现在自动检测命令中的 `.sh` 文件并前置 `bash`。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 尝试将 `.cmd` 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制执行 shell 脚本的 LF 行结尾（修复 Windows 检出时的 CRLF 问题）。

**Windows：SessionStart 钩子异步运行以防止终端冻结** (#404, #413, #414, #419)

同步的 SessionStart 钩子阻止了 TUI 在 Windows 上进入原始模式，冻结了所有键盘输入。异步运行钩子可以防止冻结，同时仍注入 superpowers 上下文。

**Windows：修复了 O(n^2) `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环由于子字符串复制开销在 bash 中是 O(n^2)。在 Windows Git Bash 上，这需要 60 多秒。替换为 bash 参数替换（`${s//old/new}`），每个模式作为单个 C 级传递运行 — 在 macOS 上快 7 倍，在 Windows 上显著更快。

**Codex：修复了 Windows/PowerShell 调用** (#285, #243)

- Windows 不尊重 shebang，因此直接调用无扩展名的 `superpowers-codex` 脚本会触发“打开方式”对话框。所有调用现在都前缀了 `node`。
- 修复了 Windows 上的 `~/` 路径扩展 — PowerShell 在作为参数传递给 `node` 时不会扩展 `~`。更改为 `$HOME`，它在 bash 和 PowerShell 中都能正确扩展。

**Codex：修复了安装程序中的路径解析**

使用 `fileURLToPath()` 而不是手动 URL 路径名解析，以在所有平台上正确处理带有空格和特殊字符的路径。

**Codex：修复了 writing-skills 中过时的技能路径**

将 `~/.codex/skills/` 引用（已弃用）更新为 `~/.agents/skills/` 以进行原生发现。

### 改进

**实施前现在需要工作树隔离**

将 `using-git-worktrees` 添加为 `subagent-driven-development` 和 `executing-plans` 的必需技能。实施工作流现在明确要求在开始工作之前设置隔离的工作树，防止意外直接在 main 分支上工作。

**Main 分支保护软化，需要明确同意**

技能现在允许在获得用户明确同意的情况下在 main 分支上工作，而不是完全禁止。在确保用户了解影响的同时更加灵活。

**简化了安装验证**

从验证步骤中移除了 `/help` 命令检查和特定的斜杠命令列表。技能主要通过描述你想要做什么来调用，而不是通过运行特定命令。

**Codex：在引导中澄清了子代理工具映射**

改进了关于 Codex 工具如何映射到 Claude Code 等效项以用于子代理工作流的文档。

### 测试

- 为 subagent-driven-development 添加了工作树要求测试
- 添加了 main 分支红旗警告测试
- 修复了技能识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### 修复

**OpenCode：根据官方文档标准化为 `plugins/` 目录** (#343)

OpenCode 的官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档之前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，但我们已经标准化为官方约定以避免混淆。

变更：
- 在仓库结构中重命名 `.opencode/plugin/` 为 `.opencode/plugins/`
- 更新了所有安装文档（INSTALL.md、README.opencode.md）在所有平台上
- 更新了测试脚本以匹配

**OpenCode：修复了符号链接说明** (#339, #342)

- 在 `ln -s` 之前添加了明确的 `rm`（修复重新安装时的“文件已存在”错误）
- 添加了 INSTALL.md 中缺失的技能符号链接步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 破坏性变更

**OpenCode：切换到原生技能系统**

Superpowers for OpenCode 现在使用 OpenCode 的原生 `skill` 工具，而不是自定义的 `use_skill`/`find_skills` 工具。这是一个更简洁的集成，适用于 OpenCode 的内置技能发现。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（参见更新的安装文档）。

### 修复

**OpenCode：修复了会话开始时的代理重置** (#226)

之前使用 `session.prompt({ noReply: true })` 的引导注入方法导致 OpenCode 在第一条消息时将所选代理重置为“build”。现在使用 `experimental.chat.system.transform` 钩子，它直接修改系统提示而没有副作用。

**OpenCode：修复了 Windows 安装** (#232)

- 移除了对 `skills-core.js` 的依赖（消除了文件被复制而不是符号链接时损坏的相对导入）
- 为 cmd.exe、PowerShell 和 Git Bash 添加了全面的 Windows 安装文档
- 记录了每个平台的正确符号链接与连接点用法

**Claude Code：修复了 Claude Code 2.1.x 的 Windows 钩子执行**

Claude Code 2.1.x 改变了钩子在 Windows 上的执行方式：它现在自动检测命令中的 `.sh` 文件并前置 `bash `。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 尝试将 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制执行 shell 脚本的 LF 行结尾（修复 Windows 检出时的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### 改进

**加强了 using-superpowers 技能以处理明确的技能请求**

解决了一个故障模式，即使用户明确按名称请求技能（例如，“subagent-driven-development, please”），Claude 也会跳过调用技能。Claude 会认为“我知道那是什么意思”并直接开始工作，而不是加载技能。

变更：
- 将“规则”更新为“调用相关或请求的技能”而不是“检查技能” — 强调主动调用而非被动检查
- 添加了“在任何响应或行动之前” — 原始措辞仅提及“响应”，但 Claude 有时会在不先响应的情况下采取行动
- 添加了保证，调用错误的技能是可以的 — 减少犹豫
- 添加了新红旗：“我知道那是什么意思” → 知道概念 ≠ 使用技能

**添加了明确的技能请求测试**

`tests/explicit-skill-requests/` 中的新测试套件，验证当用户按名称请求技能时 Claude 是否正确调用技能。包括单回合和多回合测试场景。

## v4.0.2 (2025-12-23)

### 修复

**斜杠命令现在仅限用户使用**

在所有三个斜杠命令（`/brainstorm`、`/execute-plan`、`/write-plan`）中添加了 `disable-model-invocation: true`。Claude 不能再通过技能工具调用这些命令 — 它们仅限于手动用户调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍然可供 Claude 自主调用。此更改防止了当 Claude 调用一个只是重定向到技能的命令时的混淆。

## v4.0.1 (2025-12-23)

### 修复

**澄清了如何在 Claude Code 中访问技能**

修复了一个令人困惑的模式，即 Claude 通过技能工具调用技能，然后尝试单独读取技能文件。`using-superpowers` 技能现在明确说明技能工具直接加载技能内容 — 无需读取文件。

- 在 `using-superpowers` 中添加了“如何访问技能”部分
- 将“读取技能”更改为“调用技能”在指令中
- 更新了斜杠命令以使用完全限定的技能名称（例如，`superpowers:brainstorming`）

**在 receiving-code-review 中添加了 GitHub 线程回复指导**（感谢 @ralphbean）

添加了关于在原始线程中回复内联审查评论而不是作为顶级 PR 评论的说明。

**在 writing-skills 中添加了自动化优于文档的指导**（感谢 @EthanJStark）

添加了指导，即机械约束应自动化，而不是记录 — 为判断调用保留技能。

## v4.0.0 (2025-12-17)

### 新功能

**子代理驱动开发中的两阶段代码审查**

子代理工作流现在在每个任务后使用两个独立的审查阶段：

1. **规格合规审查** — 持怀疑态度的审查者验证实施是否完全符合规格。捕获缺失的要求和过度构建。不信任实施者的报告 — 读取实际代码。

2. **代码质量审查** — 仅在规格合规通过后运行。审查代码整洁度、测试覆盖率和可维护性。

这捕获了常见的故障模式，即代码编写良好但与请求的内容不匹配。审查是循环，而不是一次性：如果审查者发现问题，实施者修复它们，然后审查者再次检查。

其他子代理工作流改进：
- 控制器向工作器提供完整的任务文本（而不是文件引用）
- 工作器可以在工作之前和工作期间询问澄清问题
- 在报告完成之前进行自检清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` — 包括自检清单，鼓励提问
- `spec-reviewer-prompt.md` — 根据要求进行持怀疑态度的验证
- `code-quality-reviewer-prompt.md` — 标准代码审查

**调试技术与工具整合**

`systematic-debugging` 现在

**描述陷阱**（记录于 `writing-skills`）：发现当技能描述包含工作流摘要时，会覆盖流程图内容。Claude 会遵循简短的描述，而不是阅读详细的流程图。修复方法：描述必须仅为触发条件（"在 X 时使用"），不包含过程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，过程技能（头脑风暴、调试）现在明确排在实现技能之前。"构建 X" 会先触发头脑风暴，然后是领域技能。

**头脑风暴触发条件强化**

描述改为祈使句："在进行任何创造性工作之前——创建功能、构建组件、添加功能或修改行为时——你**必须**使用此技能。"

### 重大变更

**技能合并** - 六个独立技能已合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑到 `systematic-debugging/`
- `testing-skills-with-subagents` → 捆绑到 `writing-skills/`
- `testing-anti-patterns` → 捆绑到 `test-driven-development/`
- `sharing-skills` 已移除（已过时）

### 其他改进

- **render-graphs.js** - 从技能中提取 DOT 图表并渲染为 SVG 的工具
- **using-superpowers 中的合理化表格** - 可扫描的格式，包含新条目："我需要更多上下文"、"让我先探索一下"、"这感觉很有成效"
- **docs/testing.md** - 使用 Claude Code 集成测试测试技能的指南

---

## v3.6.2 (2025-12-03)

### 修复

- **Linux 兼容性**：修复了多语言钩子包装器 (`run-hook.cmd`) 以使用符合 POSIX 标准的语法
  - 在第 16 行将 bash 特有的 `${BASH_SOURCE[0]:-$0}` 替换为标准 `$0`
  - 解决了在 Ubuntu/Debian 系统（其中 `/bin/sh` 是 dash）上的 "Bad substitution" 错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更

- **OpenCode 引导程序重构**：从 `chat.message` 钩子切换到 `session.created` 事件进行引导程序注入
  - 引导程序现在通过 `session.prompt()` 在会话创建时注入，使用 `noReply: true`
  - 明确告知模型 using-superpowers 已加载，以防止冗余技能加载
  - 将引导程序内容生成整合到共享的 `getBootstrapContent()` 辅助函数中
  - 更简洁的单实现方法（移除了回退模式）

---

## v3.5.0 (2025-11-23)

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 用于跨上下文压缩保持技能持久性的消息插入模式
  - 通过 chat.message 钩子自动上下文注入
  - 在 session.compacted 事件上自动重新注入
  - 三层技能优先级：项目 > 个人 > superpowers
  - 项目本地技能支持 (`.opencode/skills/`)
  - 共享核心模块 (`lib/skills-core.js`) 用于与 Codex 的代码复用
  - 具有适当隔离的自动化测试套件 (`tests/opencode/`)
  - 平台特定文档 (`docs/README.opencode.md`, `docs/README.codex.md`)

### 变更

- **重构的 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除了 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一事实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进的文档**：重写 README 以清晰解释问题/解决方案
  - 移除了重复部分和冲突信息
  - 添加了完整的工作流描述（头脑风暴 → 计划 → 执行 → 完成）
  - 简化了平台安装说明
  - 强调技能检查协议而非自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进

- 优化了 superpowers 引导程序，以消除冗余的技能执行。`using-superpowers` 技能内容现在直接在会话上下文中提供，并附有明确的指导，仅将 Skill 工具用于其他技能。这减少了开销，并防止了代理尽管在会话开始时已获得内容但仍手动执行 `using-superpowers` 的混乱循环。

## v3.4.0 (2025-10-30)

### 改进

- 简化了 `brainstorming` 技能，回归到最初的对话式愿景。移除了带有正式检查清单的繁重 6 阶段流程，转而采用自然对话：一次问一个问题，然后以 200-300 字的段落呈现设计并进行验证。保留了文档和实现交接功能。

## v3.3.1 (2025-10-28)

### 改进

- 更新了 `brainstorming` 技能，要求在提问前进行自主侦察，鼓励基于推荐的决策，并防止代理将优先级排序任务交还给人类。
- 根据 Strunk 的《风格的要素》原则（省略不必要的词语，将否定形式转换为肯定形式，改进平行结构），对 `brainstorming` 技能应用了写作清晰度改进。

### 错误修复

- 澄清了 `writing-skills` 的指导，使其指向正确的代理特定个人技能目录（Claude Code 为 `~/.claude/skills`，Codex 为 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新功能

**实验性 Codex 支持**
- 添加了统一的 `superpowers-codex` 脚本，包含引导程序/使用技能/查找技能命令
- 跨平台 Node.js 实现（适用于 Windows、macOS、Linux）
- 命名空间技能：superpowers 技能为 `superpowers:skill-name`，个人技能为 `skill-name`
- 当名称匹配时，个人技能会覆盖 superpowers 技能
- 干净的技能显示：显示名称/描述，不显示原始 frontmatter
- 有用的上下文：显示每个技能的支持文件目录
- Codex 的工具映射：TodoWrite→update_plan，subagents→手动回退等。
- 与最小化 AGENTS.md 的引导程序集成，用于自动启动
- 特定于 Codex 的完整安装指南和引导程序说明

**与 Claude Code 集成的主要区别：**
- 单一统一脚本，而非单独的工具
- Codex 特定等效工具替换系统
- 简化的子代理处理（手动工作而非委托）
- 更新的术语："Superpowers 技能" 而非 "核心技能"

### 新增文件
- `.codex/INSTALL.md` - Codex 用户的安装指南
- `.codex/superpowers-bootstrap.md` - 带有 Codex 适配的引导程序说明
- `.codex/superpowers-codex` - 包含所有功能的统一 Node.js 可执行文件

**注意：** Codex 支持是实验性的。该集成提供了核心 superpowers 功能，但可能需要根据用户反馈进行改进。

## v3.2.3 (2025-10-23)

### 改进

**更新了 using-superpowers 技能以使用 Skill 工具而非 Read 工具**
- 将技能调用说明从 Read 工具更改为 Skill 工具
- 更新描述："使用 Read 工具" → "使用 Skill 工具"
- 更新步骤 3："使用 Read 工具" → "使用 Skill 工具来读取和运行"
- 更新合理化列表："读取当前版本" → "运行当前版本"

Skill 工具是 Claude Code 中调用技能的正确机制。此更新更正了引导程序说明，以指导代理使用正确的工具。

### 变更文件
- 已更新：`skills/using-superpowers/SKILL.md` - 将工具引用从 Read 更改为 Skill

## v3.2.2 (2025-10-21)

### 改进

**强化了 using-superpowers 技能以对抗代理的合理化行为**
- 添加了 EXTREMELY-IMPORTANT 块，包含关于强制技能检查的绝对性语言
  - "即使只有 1% 的可能性适用某个技能，你**必须**读取它"
  - "你没有选择。你不能通过合理化来逃避。"
- 添加了 MANDATORY FIRST RESPONSE PROTOCOL 检查清单
  - 代理在做出任何响应前必须完成的 5 步流程
  - 明确的 "不遵守此协议即失败" 后果
- 添加了 Common Rationalizations 部分，包含 8 种特定的规避模式
  - "这只是个简单问题" → 错误
  - "我可以快速检查文件" → 错误
  - "让我先收集信息" → 错误
  - 加上另外 5 种在代理行为中观察到的常见模式

这些更改解决了观察到的代理行为，即尽管有明确的指示，他们仍会围绕技能使用进行合理化。强有力的语言和先发制人的反驳旨在使不遵守行为更难发生。

### 变更文件
- 已更新：`skills/using-superpowers/SKILL.md` - 添加了三层强制措施以防止技能跳过合理化

## v3.2.1 (2025-10-20)

### 新功能

**代码审查代理现已包含在插件中**
- 将 `superpowers:code-reviewer` 代理添加到插件的 `agents/` 目录
- 该代理根据计划和编码标准提供系统的代码审查
- 之前要求用户拥有个人代理配置
- 所有技能引用已更新为使用命名空间的 `superpowers:code-reviewer`
- 修复 #55

### 变更文件
- 新增：`agents/code-reviewer.md` - 包含审查清单和输出格式的代理定义
- 已更新：`skills/requesting-code-review/SKILL.md` - 对 `superpowers:code-reviewer` 的引用
- 已更新：`skills/subagent-driven-development/SKILL.md` - 对 `superpowers:code-reviewer` 的引用

## v3.2.0 (2025-10-18)

### 新功能

**头脑风暴工作流中的设计文档**
- 在 brainstorming 技能中添加了第 4 阶段：设计文档
- 设计文档现在在实现之前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了原始 brainstorming 命令在技能转换过程中丢失的功能
- 文档在工作树设置和实施计划之前编写
- 已通过子代理测试，以验证在时间压力下的合规性

### 重大变更

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（以前只是 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用技能的方式保持一致
- 更新文件：brainstorming, executing-plans, subagent-driven-development, systematic-debugging, testing-skills-with-subagents, writing-plans, writing-skills

### 改进

**设计与实施计划命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实施计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，命名清晰可辨

## v3.1.1 (2025-10-17)

### 错误修复

- **修复了 README 中的命令语法** (#44) - 更新了所有命令引用以使用正确的命名空间语法（`/superpowers:brainstorm` 而不是 `/brainstorm`）。插件提供的命令由 Claude Code 自动命名空间化，以避免插件之间的冲突。

## v3.1.0 (2025-10-17)

### 重大变更

**技能名称标准化为小写**
- 所有技能 frontmatter 的 `name:` 字段现在使用小写 kebab-case，与目录名匹配
- 示例：`brainstorming`, `test-driven-development`, `using-git-worktrees`
- 所有技能公告和交叉引用已更新为小写格式
- 这确保了目录名、frontmatter 和文档之间命名的一致性

### 新功能

**增强的 brainstorming 技能**
- 添加了快速参考表，显示阶段、活动和工具使用情况
- 添加了可复制的流程检查清单以跟踪进度
- 添加了决策流程图，用于决定何时重新访问早期阶段
- 添加了全面的 AskUserQuestion 工具指南，包含具体示例
- 添加了 "问题模式" 部分，解释何时使用结构化与开放式问题
- 将关键原则重构为可扫描的表格

**Anthropic 最佳实践集成**
- 添加了 `skills/writing-skills/anthropic-best-practices.md` - 官方的 Anthropic 技能编写指南
- 在 writing-skills SKILL.md 中引用，以提供全面的指导
- 提供了渐进式披露、工作流和评估的模式

### 改进

**技能交叉引用清晰度**
- 所有技能引用现在使用明确的标记：
  - `**REQUIRED BACKGROUND:**` - 你必须理解的先决条件
  - `**REQUIRED SUB-SKILL:**` - 必须在工作流中使用的技能
  - `**Complementary skills:**` - 可选但有用的相关技能
- 移除了旧的路径格式（`skills/collaboration/X` → 仅 `X`）
- 使用分类关系（必需 vs 补充）更新了集成部分
- 使用最佳实践更新了交叉引用文档

**与 Anthropic 最佳实践保持一致**
- 修复了描述语法和语态（完全第三人称）
- 添加了用于扫描的快速参考表
- 添加了 Claude 可以复制和跟踪的流程检查清单
- 适当使用流程图处理非显而易见的决策点
- 改进了可扫描的表格格式
- 所有技能都远低于 500 行的推荐限制

### 错误修复

- **重新添加了缺失的命令重定向** - 恢复了在 v3.0 迁移中意外删除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复了 `defense-in-depth` 名称不匹配问题（原为 `Defense-in-Depth-Validation`）
- 修复了 `receiving-code-review` 名称不匹配问题（原为 `Code-Review-Reception`）
- 修复了 `commands/brainstorm.md` 中对正确技能名称的引用
- 移除了对不存在的相关技能的引用

### 文档

**writing-skills 改进**
- 使用明确的标记更新了交叉引用指导
- 添加了对 Anthropic 官方最佳实践的引用
- 改进了显示正确技能引用格式的示例

## v3.0.1 (2025-10-16)

### 变更

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2 (2025-10-12)

### 错误修复

- **修复了当本地技能仓库领先于上游时的错误警告** - 当本地仓库的提交领先于上游时，初始化脚本错误地警告 "有新的技能可从上游获取"。现在逻辑正确区分了三种 git 状态：本地落后（应更新）、本地领先（无警告）和分叉（应警告）。

## v2.0.1 (2025-10-12)

### 错误修复

- **修复了插件上下文中的 session-start 钩子执行问题** (#8, PR #9) - 该钩子静默失败，显示 "Plugin hook error"，阻止了技能上下文的加载。通过以下方式修复：
  - 在 Claude Code 的执行上下文中 BASH_SOURCE 未绑定时，使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 添加 `|| true` 以在过滤状态标志时优雅地处理空的 grep 结果

---

# Superpowers v2.0.0 发布说明

## 概述

Superpowers v2.0 通过重大的架构转变，使技能更易于访问、维护和社区驱动。

头条变化是**技能仓库分离**：所有技能、脚本和文档都已从插件中移出，转移到一个专门的仓库 ([obra/superpowers-skills](https://github.com/obra/superpowers-skills))。这将 superpowers 从一个单体插件转变为一个轻量级的垫片，用于管理技能仓库的本地克隆。技能在会话开始时自动更新。用户通过标准的 git 工作流进行分叉和贡献改进。技能库的版本独立于插件。

除了基础设施，此版本还增加了九个专注于问题解决、研究和架构的新技能。我们以命令式的语气和更清晰的结构重写了核心的 **using-skills** 文档，使 Claude 更容易理解何时以及如何使用技能。**find-skills** 现在输出可以直接粘贴到 Read 工具中的路径，消除了技能发现工作流中的摩擦。

用户体验无缝操作：插件自动处理克隆、分叉和更新。贡献者发现新的架构使得改进和共享技能变得简单。此版本为技能作为社区资源快速演进奠定了基础。

## 重大变更

### 技能仓库分离

**最大的变化：** 技能不再存在于插件中。它们已移至单独的仓库，地址为 [obra/superpowers-skills](https://github.com/obra/superpowers-skills)。

**这对你意味着什么：**

- **首次安装：** 插件自动将技能克隆到 `~/.config/superpowers/skills/`
- **分叉：** 在设置过程中，你将有机会选择分叉技能仓库（如果安装了 `gh`）
- **更新：** 技能在会话开始时自动更新（尽可能快进）
- **贡献：** 在分支上工作，本地提交，向上游提交 PR
- **不再有影子系统：** 旧的两层系统（个人/核心）被单仓库分支工作流取代

**迁移：**

如果你有现有安装：
1. 你旧的 `~/.config/superpowers/.git` 将备份到 `~/.config/superpowers/.git.bak`
2. 旧技能将备份到 `~/.config/superpowers/skills.bak`
3. 将在 `~/.config/superpowers/skills/` 创建 obra/superpowers-skills 的新鲜克隆

### 移除的功能

- **个人 superpowers 覆盖系统** - 被 git 分支工作流取代
- **setup-personal-superpowers 钩子** - 被 initialize-skills.sh 取代

## 新功能

### 技能仓库基础设施

**自动克隆与设置** (`lib/initialize-skills.sh`)
- 首次运行时克隆 obra/superpowers-skills
- 如果安装了 GitHub CLI，则提供创建分叉的选项
- 正确设置上游/源远程仓库
- 处理从旧安装的迁移

**自动更新**
- 每次会话开始时从跟踪的远程仓库获取
- 尽可能自动快进合并
- 当需要手动同步时（分支分叉）通知
- 使用 pulling-updates-from-skills-repository 技能进行手动同步

### 新技能

**问题解决技能** (`skills/problem-solving/`)
- **collision-zone-thinking** - 强制不相关的概念碰撞以获得涌现的见解
- **inversion-exercise** - 翻转假设以揭示隐藏的约束
- **meta-pattern-recognition** - 跨领域发现通用原则
- **scale-game** - 在极端情况下测试以暴露基本真理
- **simplification-cascades** - 找到能消除多个组件的见解
- **when-stuck** - 分派到正确的问题解决技术

**研究技能** (`skills/research/`)
- **tracing-knowledge-lineages** - 理解思想如何随时间演变

**架构技能** (`skills/architecture/`)
- **preserving-productive-tensions** - 保持多种有效方法，而不是强制过早解决

### 技能改进

**using-skills (原 getting-started)**
- 从 getting-started 重命名为 using-skills
- 使用命令式语气完全重写 (v4.0.0)
- 前置关键规则
- 为所有工作流添加了 "为什么" 解释
- 引用中始终包含 /SKILL.md 后缀
- 更清晰地区分严格规则和灵活模式

**writing-skills**
- 交叉引用指导从 using-skills 移出
- 添加了令牌效率部分（字数目标）
- 改进了 CSO（Claude 搜索优化）指导

**sharing-skills**
- 更新为新的分支和 PR 工作流 (v2.0.0)
- 移除了个人/核心分割的引用

**pulling-updates-from-skills-repository** (新)
- 与上游同步的完整工作流
- 取代旧的 "updating-skills" 技能

### 工具改进

**find-skills**
- 现在输出带有 /SKILL.md 后缀的完整路径
- 使路径可直接与 Read 工具一起使用
- 更新了帮助文本

**skill-run**
- 从 scripts/ 移动到 skills/using-skills/
- 改进了文档

### 插件基础设施

**会话开始钩子**
- 现在从技能仓库位置加载
- 在会话开始时显示完整的技能列表
- 打印技能位置信息
- 显示更新状态（成功更新 / 落后于上游）
- 将 "技能落后" 警告移至输出末尾

**环境变量**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有路径中一致使用

## 错误修复

- 修复了分叉时重复添加上游远程仓库的问题
- 修复了 find-skills 输出中双重的 "skills/" 前缀
- 从 session-start 中移除了过时的 setup-personal-superpowers 调用
- 修复了整个钩子和命令中的路径引用

## 文档

### README
- 更新以适应新的技能仓库架构
- 显眼的 superpowers-skills 仓库链接
- 更新了自动更新描述
- 修复了技能名称和引用
- 更新了元技能列表

### 测试文档
- 添加了全面的测试检查清单 (`docs/TESTING-CHECKLIST.md`)
- 为测试创建了本地市场配置
- 记录了手动测试场景

## 技术细节

### 文件变更

**新增：**
- `lib/initialize-skills.sh` - 技能仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**移除：**
- `skills/` 目录 (82 个文件) - 现在在 obra/superpowers-skills 中
- `scripts/` 目录 - 现在在 obra/superpowers-skills/skills/using-skills/ 中
- `hooks/setup-personal-superpowers.sh` - 已过时

**修改：**
- `hooks/session-start.sh` - 使用来自 ~/.config/superpowers/skills 的技能
- `commands/brainstorm.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 为新架构完全重写

### 提交历史

此版本包括：
- 20+ 次提交用于技能仓库分离
- PR #1：受 Amplifier 启发的问题解决和研究技能
- PR #2：个人 superpowers 覆盖系统（后来被取代）
- 多次技能细化和文档改进

## 升级说明

### 全新安装

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件会自动处理一切。

### 从 v1.x 升级

1. **备份你的个人技能**（如果有的话）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **在下一个会话开始时：**
   - 旧安装将自动备份
   - 将克隆新的技能仓库
   - 如果你有 GitHub CLI，你将有机会选择分叉

4. **迁移个人技能**（如果你有的话）：
   - 在你的本地技能仓库中创建一个分支
   - 从备份中复制你的个人技能
   - 提交并推送到你的分叉
   - 考虑通过 PR 贡献回社区

## 下一步计划

### 对于用户

- 探索新的问题解决技能
- 尝试基于分支的技能改进工作流
- 将技能贡献回社区

### 对于贡献者

- 技能仓库现在位于 https://github.com/obra/superpowers-skills
- 分叉 → 分支 → PR 工作流
- 参见 skills/meta/writing-skills/SKILL.md 了解文档的 TDD 方法

## 已知问题

目前没有。

## 致谢

- 问题解决技能受 Amplifier 模式启发
- 社区贡献和反馈
- 对技能有效性的广泛测试和迭代

---

**完整更新日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能仓库：** https://github.com/obra/superpowers-skills
**问题：** https://github.com/obra/superpowers/issues
