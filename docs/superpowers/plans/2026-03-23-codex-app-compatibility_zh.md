# Codex 应用兼容性实施计划

> **面向智能体工作者：** 必需子技能：使用超能力：子智能体驱动开发（推荐）或超能力：执行计划来按任务实施此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 使 `using-git-worktrees`、`finishing-a-development-branch` 及相关技能在 Codex 应用的沙盒化工作树环境中正常工作，同时不破坏现有行为。

**架构：** 在两个技能开始时进行只读环境检测（`git-dir` 与 `git-common-dir`）。如果已处于链接的工作树中，则跳过创建。如果处于分离的 HEAD 状态，则发出移交负载而非显示 4 选项菜单。沙盒回退机制捕获工作树创建期间的权限错误。

**技术栈：** Git、Markdown（技能文件是指令文档，非可执行代码）

**规范：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 操作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | 工作树创建 + 隔离 | 添加步骤 0 检测 + 沙盒回退 |
| `skills/finishing-a-development-branch/SKILL.md` | 分支完成工作流 | 添加步骤 1.5 检测 + 清理防护 |
| `skills/subagent-driven-development/SKILL.md` | 使用子智能体执行计划 | 更新集成描述 |
| `skills/executing-plans/SKILL.md` | 内联执行计划 | 更新集成描述 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 添加检测 + 完成文档 |

---

### 任务 1：向 `using-git-worktrees` 添加步骤 0

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:14-15`（在概述之后、目录选择过程之前插入）

- [ ] **步骤 1：读取当前技能文件**

完整阅读 `skills/using-git-worktrees/SKILL.md`。确定确切的插入点：在“开始时宣布”行（第 14 行）之后，在“## 目录选择过程”（第 16 行）之前。

- [ ] **步骤 2：插入步骤 0 部分**

在概述部分和“## 目录选择过程”之间插入以下内容：

```markdown
## 步骤 0：检查是否已处于隔离工作空间

在创建工作树之前，检查是否已存在一个：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**如果 `GIT_DIR` 与 `GIT_COMMON` 不同：** 您已处于链接的工作树中（由 Codex 应用、Claude Code 的智能体工具、先前技能运行或用户创建）。请勿创建另一个工作树。而是：

1. 运行项目设置（如下方“运行项目设置”中自动检测包管理器）
2. 验证干净基线（如下方“验证干净基线”中运行测试）
3. 报告分支状态：
   - 在分支上：“已处于隔离工作空间 `<路径>`，分支 `<名称>`。测试通过。准备实施。”
   - 分离的 HEAD：“已处于隔离工作空间 `<路径>`（分离的 HEAD，外部管理）。测试通过。注意：完成时需要创建分支。准备实施。”

报告后，停止。不要继续到目录选择或创建步骤。

**如果 `GIT_DIR` 等于 `GIT_COMMON`：** 继续下面的完整工作树创建流程。

**沙盒回退：** 如果您继续到创建步骤但 `git worktree add -b` 因权限错误（例如“操作不允许”）失败，则将其视为后期检测到的受限环境。回退到上述行为——在当前目录中运行设置和基线测试，相应报告，然后停止。
```

- [ ] **步骤 3：验证插入**

再次读取文件。确认：
- 步骤 0 出现在概述和目录选择过程之间
- 文件其余部分（目录选择、安全验证、创建步骤等）未更改
- 没有重复部分或损坏的 markdown

- [ ] **步骤 4：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): 添加步骤 0 环境检测 (PRI-823)

当已处于链接的工作树时跳过工作树创建。包含
`git worktree add` 权限错误的沙盒回退。"
```

---

### 任务 2：更新 `using-git-worktrees` 集成部分

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:211-215`（集成 > 被调用者）

- [ ] **步骤 1：更新三个“被调用者”条目**

将第 212-214 行从：

```markdown
- **brainstorming**（阶段 4）- 当设计批准且后续实施时必需
- **subagent-driven-development** - 在执行任何任务前必需
- **executing-plans** - 在执行任何任务前必需
```

更改为：

```markdown
- **brainstorming** - 必需：确保隔离工作空间（创建一个或验证现有）
- **subagent-driven-development** - 必需：确保隔离工作空间（创建一个或验证现有）
- **executing-plans** - 必需：确保隔离工作空间（创建一个或验证现有）
```

- [ ] **步骤 2：验证集成部分**

阅读集成部分。确认所有三个条目已更新，“配对使用”未更改。

- [ ] **步骤 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): 更新集成描述 (PRI-823)

澄清该技能确保工作空间存在，而非总是创建一个。"
```

---

### 任务 3：向 `finishing-a-development-branch` 添加步骤 1.5

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md:38`（在步骤 1 之后、步骤 2 之前插入）

- [ ] **步骤 1：读取当前技能文件**

完整阅读 `skills/finishing-a-development-branch/SKILL.md`。确定插入点：在“**如果测试通过：** 继续到步骤 2。”（第 38 行）之后，在“### 步骤 2：确定基础分支”（第 40 行）之前。

- [ ] **步骤 2：插入步骤 1.5 部分**

在步骤 1 和步骤 2 之间插入以下内容：

```markdown
### 步骤 1.5：检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**路径 A — `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 为空（外部管理的工作树，分离的 HEAD）：**

首先，确保所有工作都已暂存并提交（`git add` + `git commit`）。

然后向用户呈现此信息（请勿呈现 4 选项菜单）：

```
实施完成。所有测试通过。
当前 HEAD：<完整提交 SHA>

此工作空间由外部管理（分离的 HEAD）。
我无法从此处创建分支、推送或打开 PR。

⚠ 这些提交位于分离的 HEAD 上。如果您不创建分支，
当此工作空间被清理时，它们可能会丢失。

如果您的宿主应用程序提供以下控制：
- “创建分支” — 命名分支，然后提交/推送/PR
- “移交到本地” — 将更改移动到您的本地检出

建议分支名称：<工单 ID/简短描述>
建议提交消息：<工作摘要>
```

分支名称：如果可用则使用工单 ID（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个单词转换为 slug，否则省略。避免在分支名称中包含敏感内容。

跳转到步骤 5（清理是无操作 — 见下方防护）。

**路径 B — `GIT_DIR` 与 `GIT_COMMON` 不同 且 `BRANCH` 存在（外部管理的工作树，命名分支）：**

继续到步骤 2 并正常呈现 4 选项菜单。

**路径 C — `GIT_DIR` 等于 `GIT_COMMON`（正常环境）：**

继续到步骤 2 并正常呈现 4 选项菜单。
```

- [ ] **步骤 3：验证插入**

再次读取文件。确认：
- 步骤 1.5 出现在步骤 1 和步骤 2 之间
- 步骤 2-5 未更改
- 路径 A 移交包含提交 SHA 和数据丢失警告
- 路径 B 和 C 正常继续到步骤 2

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): 添加步骤 1.5 环境检测 (PRI-823)

检测具有分离 HEAD 的外部管理工作树并发出移交
负载而非 4 选项菜单。包含提交 SHA 和数据丢失警告。"
```

---

### 任务 4：向 `finishing-a-development-branch` 添加步骤 5 清理防护

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（步骤 5：清理工作树 — 通过章节标题查找，任务 3 后行号已偏移）

- [ ] **步骤 1：读取当前步骤 5 部分**

在 `skills/finishing-a-development-branch/SKILL.md` 中找到“### 步骤 5：清理工作树”部分（任务 3 插入后行号已偏移）。当前步骤 5 为：

```markdown
### 步骤 5：清理工作树

**对于选项 1、2、4：**

检查是否在工作树中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <工作树路径>
```

**对于选项 3：** 保留工作树。
```

- [ ] **步骤 2：在现有逻辑前添加清理防护**

将步骤 5 部分替换为：

```markdown
### 步骤 5：清理工作树

**首先，检查工作树是否由外部管理：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

如果 `GIT_DIR` 与 `GIT_COMMON` 不同：跳过工作树移除 — 宿主环境拥有此工作空间。

**否则，对于选项 1 和 4：**

检查是否在工作树中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <工作树路径>
```

**对于选项 3：** 保留工作树。
```

注意：原始文本说“对于选项 1、2、4”，但快速参考表和常见错误部分说“仅选项 1 和 4”。此编辑使步骤 5 与这些部分保持一致。

- [ ] **步骤 3：验证替换**

阅读步骤 5。确认：
- 清理防护（重新检测）首先出现
- 为非外部管理工作树保留了现有移除逻辑
- “选项 1 和 4”（非“1、2、4”）与快速参考和常见错误匹配

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): 添加步骤 5 清理防护 (PRI-823)

在清理时重新检测外部管理工作树并跳过移除。
同时修复了预先存在的不一致：清理现在正确地说
仅选项 1 和 4，与快速参考和常见错误匹配。"
```

---

### 任务 5：更新 `subagent-driven-development` 和 `executing-plans` 中的集成行

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/executing-plans/SKILL.md:68`

- [ ] **步骤 1：更新 `subagent-driven-development`**

将第 268 行从：
```
- **superpowers:using-git-worktrees** - 必需：在开始前设置隔离工作空间
```
更改为：
```
- **superpowers:using-git-worktrees** - 必需：确保隔离工作空间（创建一个或验证现有）
```

- [ ] **步骤 2：更新 `executing-plans`**

将第 68 行从：
```
- **superpowers:using-git-worktrees** - 必需：在开始前设置隔离工作空间
```
更改为：
```
- **superpowers:using-git-worktrees** - 必需：确保隔离工作空间（创建一个或验证现有）
```

- [ ] **步骤 3：验证两个文件**

阅读 `skills/subagent-driven-development/SKILL.md` 的第 268 行和 `skills/executing-plans/SKILL.md` 的第 68 行。确认两者都说“确保隔离工作空间（创建一个或验证现有）”。

- [ ] **步骤 4：提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): 更新工作树集成描述 (PRI-823)

澄清 using-git-worktrees 确保工作空间存在而非
总是创建一个。"
```

---

### 任务 6：向 `codex-tools.md` 添加环境检测文档

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md:25`（在末尾追加）

- [ ] **步骤 1：读取当前文件**

完整阅读 `skills/using-superpowers/references/codex-tools.md`。确认它在 multi_agent 部分之后结束于第 25-26 行。

- [ ] **步骤 2：追加两个新部分**

在文件末尾添加：

```markdown

## 环境检测

创建工作树或完成分支的技能应在继续前使用
只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已处于链接的工作树中（跳过创建）
- `BRANCH` 为空 → 分离的 HEAD（无法从沙盒分支/推送/PR）

有关每个技能如何使用这些信号的详细信息，请参阅
`using-git-worktrees` 步骤 0 和 `finishing-a-development-branch`
步骤 1.5。

## Codex 应用完成

当沙盒阻止分支/推送操作时（外部管理的工作树中
分离的 HEAD），智能体提交所有工作并通知
用户使用应用的原生控制：

- **“创建分支”** — 命名分支，然后通过应用 UI 提交/推送/PR
- **“移交到本地”** — 将工作转移到用户的本地检出

智能体仍可以运行测试、暂存文件，并输出建议的分支
名称、提交消息和 PR 描述供用户复制。
```

- [ ] **步骤 3：验证添加内容**

完整阅读文件。确认：
- 两个新部分出现在现有内容之后
- Bash 代码块正确渲染（未转义）
- 存在对步骤 0 和步骤 1.5 的交叉引用

- [ ] **步骤 4：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): 添加环境检测和应用完成文档 (PRI-823)

记录 git-dir 与 git-common-dir 检测模式以及 Codex
应用的原生完成流程，供需要适应的技能使用。"
```

---

### 任务 7：自动化测试 — 环境检测

**文件：**
- 创建：`tests/codex-app-compat/test-environment-detection.sh`

- [ ] **步骤 1：创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **步骤 2：编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 测试来自 PRI-823 的环境检测逻辑
# 测试 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1.5 使用的
# git-dir 与 git-common-dir 比较

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  通过: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  失败: $1"; FAIL=$((FAIL + 1)); }

# 辅助函数：运行检测并返回 "linked" 或 "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== 测试 1：正常仓库检测 ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "正常仓库检测为 normal"
else
  log_fail "正常仓库检测为 '$result'（预期 'normal'）"
fi

echo "=== 测试 2：链接工作树检测 ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "链接工作树检测为 linked"
else
  log_fail "链接工作树检测为 '$result'（预期 'linked'）"
fi

echo "=== 测试 3：分离 HEAD 检测 ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "分离 HEAD：分支为空"
else
  log_fail "分离 HEAD：分支为 '$branch'（预期为空）"
fi

echo "=== 测试 4：链接工作树 + 分离 HEAD（Codex 应用模拟）==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex 应用模拟：linked + 分离 HEAD"
else
  log_fail "Codex 应用模拟：result='$result', branch='$branch'"
fi

echo "=== 测试 5：清理防护 — 链接工作树应不移除 ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "清理防护：链接工作树正确检测（将跳过移除）"
else
  log_fail "清理防护：预期 'linked'，得到 '$result'"
fi

echo "=== 测试 6：清理防护 — 主仓库应移除 ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "清理防护：主仓库正确检测（将继续移除）"
else
  log_fail "清理防护：预期 'normal'，得到 '$result'"
fi

# 在临时目录移除前清理工作树
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== 结果：$PASS 通过，$FAIL 失败 ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **步骤 3：使其可执行并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 通过，0 失败。

- [ ] **步骤 4：提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: 为 Codex 应用兼容性添加环境检测测试 (PRI-823)

测试正常仓库、链接工作树、分离 HEAD 和清理防护场景中的
git-dir 与 git-common-dir 比较。"
```

---

### 任务 8：最终验证

**文件：**
- 读取：所有 5 个修改的技能文件

- [ ] **步骤 1：运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 通过，0 失败。

- [ ] **步骤 2：读取每个修改的文件并验证更改**

端到端阅读每个文件：
- `skills/using-git-worktrees/SKILL.md` — 步骤 0 存在，其余未更改
- `skills/finishing-a-development-branch/SKILL.md` — 步骤 1.5 存在，清理防护存在，其余未更改
- `skills/subagent-driven-development/SKILL.md` — 第 268 行已更新
- `skills/executing-plans/SKILL.md` — 第 68 行已更新
- `skills/using-superpowers/references/codex-tools.md` — 末尾有两个新部分

- [ ] **步骤 3：验证无意外更改**

```bash
git diff --stat HEAD~7
```

应恰好显示 6 个文件更改（5 个技能文件 + 1 个测试文件）。未修改其他文件。

- [ ] **步骤 4：运行现有测试套件**

如果存在测试运行器：
```bash
# 运行技能触发测试
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "此环境中技能触发测试不可用"

# 运行 SDD 集成测试
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "此环境中 SDD 集成测试不可用"
```

注意：这些测试需要带有 `--dangerously-skip-permissions` 的 Claude Code。如果不可用，请记录应手动运行回归测试。
