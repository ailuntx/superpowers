# Superpowers

Superpowers 是一个为你的编程代理设计的完整软件开发方法论，构建在一套可组合的技能和一些初始指令之上，确保你的代理使用它们。

## 工作原理

从你启动编程代理的那一刻起。当它看到你正在构建某样东西时，它*不会*直接跳进去写代码。相反，它会退一步，问清楚你真正想做什么。

一旦它从对话中梳理出一份规格说明，它会分块展示给你，每块足够短，让你能真正阅读和消化。

在你对设计签字认可之后，你的代理会制定一个实施计划，这个计划清晰到连一个充满热情但品味不佳、缺乏判断力、不了解项目背景、不愿测试的初级工程师都能照着做。它强调真正的红/绿 TDD、YAGNI（You Aren't Gonna Need It，不需要就不要）和 DRY。

接下来，当你说“开始”时，它会启动一个*子代理驱动开发*流程，让代理们处理每个工程任务，检查并审查它们的工作，然后继续推进。Claude 能够自主工作几个小时而不偏离你们共同制定的计划，这并不罕见。

还有很多其他内容，但这是系统的核心。由于这些技能会自动触发，你无需做任何特殊的事情。你的编程代理就这么拥有了超能力。


## 赞助

如果 Superpowers 帮助你做了一些能赚钱的事情，而且你愿意，我会非常感激，如果你能考虑[赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse


## 安装

**注意：** 安装方式因平台而异。

### Claude Code 官方市场

Superpowers 可通过 [Claude 官方插件市场](https://claude.com/plugins/superpowers) 获取

从 Anthropic 的官方市场安装该插件：

```bash
/plugin install superpowers@claude-plugins-official
```

### Claude Code（Superpowers 市场）

Superpowers 市场为 Claude Code 提供了 Superpowers 和其他一些相关插件。

在 Claude Code 中，首先注册市场：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从此市场安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### OpenAI Codex CLI

- 打开插件搜索界面

```bash
/plugins
```

搜索 Superpowers

```bash
superpowers
```

选择 `Install Plugin`

### OpenAI Codex App

- 在 Codex 应用中，点击侧边栏的 Plugins。
- 你应该能在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。


### Cursor（通过插件市场）

在 Cursor Agent 聊天中，从市场安装：

```text
/add-plugin superpowers
```

或在插件市场中搜索 "superpowers"。

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档：** [docs/README.opencode.md](docs/README.opencode.md)

### GitHub Copilot CLI

```bash
copilot plugin marketplace add obra/superpowers-marketplace
copilot plugin install superpowers@superpowers-marketplace
```

### Gemini CLI

```bash
gemini extensions install https://github.com/obra/superpowers
```

更新：

```bash
gemini extensions update superpowers
```

## 基本工作流程

1. **brainstorming** - 在写代码之前激活。通过提问提炼初步想法，探索替代方案，将设计分段呈现以供验证。保存设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离的工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在拥有已批准的设计时激活。将工作分解为一口大小的任务（每个 2-5 分钟）。每个任务都包含确切的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划时激活。为每个任务分派一个全新的子代理，并执行两阶段审查（先是规范符合性，然后是代码质量），或者以批次方式执行，并在中途设置人工检查点。

5. **test-driven-development** - 实现过程中激活。强制执行 RED-GREEN-REFACTOR 循环：先写失败的测试，看着它失败，写最少的代码让它通过，看着它通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review** - 任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题会阻塞进度。

7. **finishing-a-development-branch** - 所有任务完成时激活。验证测试，呈现选项（合并/PR/保留/丢弃），清理工作树。

**代理在执行任何任务之前都会检查相关的技能。** 强制性的工作流，而非建议。

## 包含内容

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 四阶段根因分析流程（包含根因追踪、纵深防御、基于条件的等待技巧）
- **verification-before-completion** - 确保问题确实已修复

**协作**
- **brainstorming** - 苏格拉底式的设计精炼
- **writing-plans** - 详细的实施计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发子代理工作流
- **requesting-code-review** - 审查前的自检清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行的开发分支
- **finishing-a-development-branch** - 合并/PR 决策流程
- **subagent-driven-development** - 带两阶段审查（规范符合性，然后是代码质量）的快速迭代

**元**
- **writing-skills** - 遵循最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## 理念

- **测试驱动开发** - 永远先写测试
- **系统化而非即兴** - 遵循流程而非靠猜测
- **降低复杂性** - 把简洁作为首要目标
- **用证据代替断言** - 在宣布成功之前先验证

阅读[原始的发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的一般贡献流程如下。请记住，我们通常不接受新技能的贡献，而且任何对技能的更新都必须适用于我们支持的所有编程代理。

1. Fork 代码仓库
2. 切换到 'dev' 分支
3. 为你所做的修改创建一个分支
4. 遵循 `writing-skills` 技能来创建和测试新的或修改后的技能
5. 提交 PR 时务必填写 pull request 模板。

完整指南见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新方式在某种程度上取决于所使用的编程代理，但通常是自动进行的。

## 许可证

MIT 许可证 - 详情见 LICENSE 文件

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的伙伴们共同构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz)，获取社区支持、提问并分享你用 Superpowers 构建的项目
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[注册](https://primeradiant.com/superpowers/) 以获取新版本通知
