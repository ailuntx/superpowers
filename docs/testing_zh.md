# 测试超能力技能

本文档描述了如何测试超能力技能，特别是针对复杂技能（如`subagent-driven-development`）的集成测试。

## 概述

测试涉及子代理、工作流和复杂交互的技能，需要在无头模式下运行实际的Claude Code会话，并通过会话记录验证其行为。

## 测试结构

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # 共享测试工具
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # Token分析工具
│   └── run-skill-tests.sh                 # 测试运行器（如果存在）
```

## 运行测试

### 集成测试

集成测试使用实际技能执行真实的Claude Code会话：

```bash
# 运行subagent-driven-development集成测试
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注意：** 集成测试可能需要10-30分钟，因为它们会执行包含多个子代理的实际实施计划。

### 要求

- 必须从**superpowers插件目录**运行（不能从临时目录运行）
- 必须安装Claude Code并可通过`claude`命令使用
- 必须启用本地开发市场：在`~/.claude/settings.json`中设置`"superpowers@superpowers-dev": true`

## 集成测试：subagent-driven-development

### 测试内容

集成测试验证`subagent-driven-development`技能是否正确：

1. **计划加载**：在开始时读取计划一次
2. **完整任务文本**：向子代理提供完整的任务描述（不让它们读取文件）
3. **自我审查**：确保子代理在报告前执行自我审查
4. **审查顺序**：在代码质量审查之前运行规范合规性审查
5. **审查循环**：发现问题时使用审查循环
6. **独立验证**：规范审查员独立阅读代码，不信任实施者报告

### 工作原理

1. **设置**：创建一个包含最小实施计划的临时Node.js项目
2. **执行**：在无头模式下使用技能运行Claude Code
3. **验证**：解析会话记录（`.jsonl`文件）以验证：
   - 技能工具被调用
   - 子代理被分派（任务工具）
   - 使用了TodoWrite进行跟踪
   - 创建了实施文件
   - 测试通过
   - Git提交显示正确的工作流
4. **Token分析**：显示按子代理划分的Token使用情况

### 测试输出

```
========================================
 集成测试：subagent-driven-development
========================================

测试项目：/tmp/tmp.xyz123

=== 验证测试 ===

测试1：技能工具调用...
  [通过] subagent-driven-development技能被调用

测试2：子代理分派...
  [通过] 7个子代理被分派

测试3：任务跟踪...
  [通过] TodoWrite使用了5次

测试6：实施验证...
  [通过] src/math.js已创建
  [通过] add函数存在
  [通过] multiply函数存在
  [通过] test/math.test.js已创建
  [通过] 测试通过

测试7：Git提交历史...
  [通过] 创建了多个提交（共3个）

测试8：未添加额外功能...
  [通过] 未添加额外功能

=========================================
 Token使用分析
=========================================

使用情况细分：
----------------------------------------------------------------------------------------------------
代理           描述                         消息数     输入    输出     缓存     成本
----------------------------------------------------------------------------------------------------
main           主会话（协调器）             34         27      3,996  1,213,703 $   4.09
3380c209       实施任务1：创建Add函数       1          2        787     24,989 $   0.09
34b00fde       实施任务2：创建Multiply函数  1          4        644     25,114 $   0.09
3801a732       审查实施是否匹配...          1          5        703     25,742 $   0.09
4c142934       进行最终代码审查...          1          6        854     25,319 $   0.09
5f017a42       代码审查员。审查任务2...     1          6        504     22,949 $   0.08
a6b7fbe4       代码审查员。审查任务1...     1          6        515     22,534 $   0.08
f15837c0       审查实施是否匹配...          1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

总计：
  总消息数：         41
  输入Token：        62
  输出Token：        8,419
  缓存创建Token：    132,742
  缓存读取Token：    1,382,835

  总输入（含缓存）： 1,515,639
  总Token：          1,524,058

  估计成本：$4.67
  （按每百万Token输入/输出$3/$15计算）

========================================
 测试摘要
========================================

状态：通过
```

## Token分析工具

### 使用方法

分析任何Claude Code会话的Token使用情况：

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<项目目录>/<会话ID>.jsonl
```

### 查找会话文件

会话记录存储在`~/.claude/projects/`中，工作目录路径被编码：

```bash
# 例如：/Users/jesse/Documents/GitHub/superpowers/superpowers
SESSION_DIR="$HOME/.claude/projects/-Users-jesse-Documents-GitHub-superpowers-superpowers"

# 查找最近的会话
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### 显示内容

- **主会话使用情况**：协调器（您或主Claude实例）的Token使用情况
- **按子代理细分**：每个任务调用包含：
  - 代理ID
  - 描述（从提示中提取）
  - 消息计数
  - 输入/输出Token
  - 缓存使用情况
  - 估计成本
- **总计**：总体Token使用情况和成本估计

### 理解输出

- **高缓存读取**：良好 - 表示提示缓存正在工作
- **主会话高输入Token**：预期 - 协调器具有完整上下文
- **每个子代理成本相似**：预期 - 每个任务复杂度相似
- **每个任务成本**：典型范围是每个子代理$0.05-$0.15，取决于任务复杂度

## 故障排除

### 技能未加载

**问题**：运行无头测试时找不到技能

**解决方案**：
1. 确保从superpowers目录运行：`cd /path/to/superpowers && tests/...`
2. 检查`~/.claude/settings.json`中`enabledPlugins`是否包含`"superpowers@superpowers-dev": true`
3. 验证技能是否存在于`skills/`目录中

### 权限错误

**问题**：Claude被阻止写入文件或访问目录

**解决方案**：
1. 使用`--permission-mode bypassPermissions`标志
2. 使用`--add-dir /path/to/temp/dir`授予对测试目录的访问权限
3. 检查测试目录的文件权限

### 测试超时

**问题**：测试时间过长并超时

**解决方案**：
1. 增加超时时间：`timeout 1800 claude ...`（30分钟）
2. 检查技能逻辑中是否存在无限循环
3. 审查子代理任务复杂度

### 找不到会话文件

**问题**：测试运行后找不到会话记录

**解决方案**：
1. 检查`~/.claude/projects/`中的正确项目目录
2. 使用`find ~/.claude/projects -name "*.jsonl" -mmin -60`查找最近的会话
3. 验证测试是否实际运行（检查测试输出中的错误）

## 编写新的集成测试

### 模板

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# 创建测试项目
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# 设置测试文件...
cd "$TEST_PROJECT"

# 使用技能运行Claude
PROMPT="您的测试提示在此"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# 查找并分析会话
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# 通过解析会话记录验证行为
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[通过] 技能被调用"
fi

# 显示Token分析
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### 最佳实践

1. **始终清理**：使用trap清理临时目录
2. **解析记录**：不要grep面向用户的输出 - 解析`.jsonl`会话文件
3. **授予权限**：使用`--permission-mode bypassPermissions`和`--add-dir`
4. **从插件目录运行**：只有从superpowers目录运行时才会加载技能
5. **显示Token使用情况**：始终包含Token分析以了解成本
6. **测试实际行为**：验证实际创建的文件、通过的测试、进行的提交

## 会话记录格式

会话记录是JSONL（JSON行）文件，其中每行是一个表示消息或工具结果的JSON对象。

### 关键字段

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### 工具结果

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "您正在实施任务1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId`字段链接到子代理会话，`usage`字段包含该特定子代理调用的Token使用情况。
