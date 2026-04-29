# 可视化头脑风暴重构：浏览器展示与终端命令

**日期：** 2026-02-19
**状态：** 已批准
**范围：** `lib/brainstorm-server/`, `skills/brainstorming/visual-companion.md`, `tests/brainstorm-server/`

## 问题

在可视化头脑风暴过程中，Claude 将 `wait-for-feedback.sh` 作为后台任务运行，并阻塞在 `TaskOutput(block=true, timeout=600s)` 上。这会完全占用 TUI——在可视化头脑风暴运行时，用户无法向 Claude 输入。浏览器成为唯一的输入通道。

Claude Code 的执行模型是基于回合的。Claude 无法在单个回合内同时监听两个通道。阻塞式的 `TaskOutput` 模式是错误的原语——它模拟了平台不支持的基于事件的行为。

## 设计

### 核心模型

**浏览器 = 交互式展示。** 显示模拟界面，让用户点击选择选项。选择结果记录在服务器端。

**终端 = 对话通道。** 始终保持非阻塞状态，始终可用。用户在此与 Claude 交流。

### 循环流程

1. Claude 将 HTML 文件写入会话目录
2. 服务器通过 chokidar 检测到文件变化，通过 WebSocket 推送重载到浏览器（保持不变）
3. Claude 结束当前回合——告诉用户查看浏览器并在终端中回应
4. 用户查看浏览器，可选地点击选择选项，然后在终端中输入反馈
5. 在下一回合中，Claude 读取 `$SCREEN_DIR/.events` 获取浏览器交互流（点击、选择），与终端文本合并
6. 迭代或推进

无需后台任务。无需 `TaskOutput` 阻塞。无需轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。其目的是桥接"服务器将事件记录到 stdout"和"Claude 需要接收这些事件"。`.events` 文件取代了这一功能——服务器直接将用户交互事件写入文件，Claude 使用平台提供的任何文件读取机制来读取它们。

### 关键新增：`.events` 文件（每屏事件流）

服务器将所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这为 Claude 提供了当前屏幕的完整交互流——不仅仅是最终选择，还包括用户的探索路径（点击了 A，然后 B，最终选择了 C）。

用户探索选项后的示例内容：

```jsonl
{"type":"click","choice":"a","text":"选项 A - 预设优先向导","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项 C - 手动配置","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项 B - 混合方法","timestamp":1706000115}
```

- 在单个屏幕内是追加写入的。每个用户事件都作为新行追加。
- 当 chokidar 检测到新的 HTML 文件（推送了新屏幕）时，文件会被清除（删除），防止陈旧事件延续到下一屏。
- 如果 Claude 读取时文件不存在，则表示没有发生浏览器交互——Claude 仅使用终端文本。
- 文件仅包含用户事件（`click` 等）——不包含服务器生命周期事件（`server-started`、`screen-added`）。这使其保持小巧且专注。
- Claude 可以读取完整流来理解用户的探索模式，或者仅查看最后的 `choice` 事件以获取最终选择。

## 按文件变更

### `index.js`（服务器）

**A. 将用户事件写入 `.events` 文件。**

在 WebSocket `message` 处理程序中，将事件记录到 stdout 后：将事件作为 JSON 行追加到 `$SCREEN_DIR/.events`，通过 `fs.appendFileSync`。仅写入用户交互事件（带有 `source: 'user-event'` 的事件），不写入服务器生命周期事件。

**B. 在新屏幕上清除 `.events`。**

在 chokidar `add` 处理程序（检测到新的 `.html` 文件）中，如果 `$SCREEN_DIR/.events` 存在则删除它。这是明确的"新屏幕"信号——比在每次重载都会触发的 GET `/` 上清除更好。

**C. 替换 `wrapInFrame` 内容注入。**

当前的正则表达式锚定在 `<div class="feedback-footer">` 上，该元素将被移除。替换为注释占位符：移除 `#claude-content` 中现有的默认内容（`<h2>可视化头脑风暴</h2>` 和副标题段落），替换为单个 `<!-- CONTENT -->` 标记。内容注入变为 `frameTemplate.replace('<!-- CONTENT -->', content)`。更简单且不会因模板格式更改而中断。

### `frame-template.html`（UI 框架）

**移除：**
- `feedback-footer` div（文本区域、发送按钮、标签、`.feedback-row`）
- 关联的 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中的文本区域和按钮样式）

**添加：**
- `<!-- CONTENT -->` 占位符，放在 `#claude-content` 内，替换默认文本
- 在页脚位置添加选择指示栏，有两种状态：
  - 默认："点击上方选项，然后返回终端"
  - 选择后："已选择选项 B——返回终端继续"
- 指示栏的 CSS（微妙，与现有标题栏视觉权重相似）

**保持不变：**
- 带有"头脑风暴伴侣"标题和连接状态的标题栏
- `.main` 包装器和 `#claude-content` 容器
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、占位符、模拟元素）
- 深色/浅色主题变量和媒体查询

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数和"已发送给 Claude"页面接管
- `window.send()` 函数（与已移除的发送按钮绑定）
- 表单提交处理程序——没有反馈文本区域后无用途，增加日志噪音
- 输入变更处理程序——同样原因
- `pageshow` 事件监听器（原用于修复文本区域持久化——现在没有文本区域了）

**保留：**
- WebSocket 连接、重连逻辑、事件队列
- 重载处理程序（服务器推送时执行 `window.location.reload()`）
- `window.toggleSelect()` 用于选择高亮
- `window.selectedChoice` 跟踪
- `window.brainstorm.send()` 和 `window.brainstorm.choice()`——这些与已移除的 `window.send()` 不同。它们调用 `sendEvent`，通过 WebSocket 将事件记录到服务器。对于自定义完整文档页面很有用。

**收窄：**
- 点击处理程序：仅捕获 `[data-choice]` 点击，而非所有按钮/链接。当浏览器是反馈通道时需要广泛捕获；现在仅用于选择跟踪。

**添加：**
- 在 `data-choice` 点击时，更新选择指示栏文本以显示选择了哪个选项。

**从 `window.brainstorm` API 中移除：**
- `brainstorm.sendToClaude`——不再存在

### `visual-companion.md`（技能说明）

**重写"循环流程"部分**为上述非阻塞流程。移除所有对以下内容的引用：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- 超时/重试逻辑（600 秒超时，30 分钟上限）
- 描述 `send-to-claude` JSON 的"用户反馈格式"部分

**替换为：**
- 新循环（写入 HTML → 结束回合 → 用户在终端回应 → 读取 `.events` → 迭代）
- `.events` 文件格式文档
- 指导说明：终端消息是主要反馈；`.events` 提供完整的浏览器交互流作为额外上下文

**保留：**
- 服务器启动/关闭说明
- 内容片段与完整文档指导
- CSS 类参考和可用组件
- 设计技巧（根据问题调整保真度，每屏 2-4 个选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言片段响应中存在 `feedback-footer` 的测试——更新为断言选择指示栏或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试——更新以反映收窄后的 API
- 断言 `sendToClaude` CSS 变量使用的测试——移除（函数不再存在）

## 平台兼容性

服务器代码（`index.js`、`helper.js`、`frame-template.html`）完全与平台无关——纯 Node.js 和浏览器 JavaScript。没有 Claude Code 特定引用。已通过后台终端交互在 Codex 上验证可用。

技能说明（`visual-companion.md`）是平台适配层。每个平台的 Claude 使用自己的工具来启动服务器、读取 `.events` 等。非阻塞模型自然适用于所有平台，因为它不依赖于任何平台特定的阻塞原语。

## 这实现了什么

- **TUI 在可视化头脑风暴期间始终保持响应**
- **混合输入**——在浏览器中点击 + 在终端中输入，自然合并
- **优雅降级**——浏览器宕机或用户未打开？终端仍然工作
- **更简单的架构**——无后台任务，无轮询脚本，无超时管理
- **跨平台**——相同的服务器代码适用于 Claude Code、Codex 和任何未来平台

## 这放弃了什么

- **纯浏览器反馈工作流**——用户必须返回终端才能继续。选择指示栏会引导他们，但与旧的点击发送并等待流程相比，多了一个额外步骤。
- **浏览器内联文本反馈**——文本区域已移除。所有文本反馈都通过终端进行。这是有意为之——终端是比框架中的小文本区域更好的文本输入通道。
- **浏览器发送后的即时响应**——旧系统在用户点击发送时 Claude 会立即响应。现在用户切换到终端时有一个间隙。实际上这只需几秒钟，而且用户可以在终端消息中添加上下文。
