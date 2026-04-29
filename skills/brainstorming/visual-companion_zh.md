# 可视化协作指南

基于浏览器的可视化头脑风暴伴侣，用于展示原型图、图表和选项。

## 何时使用

按问题决定，而非按会话决定。判断标准是：**用户通过观看是否比阅读更能理解此内容？**

**使用浏览器**当内容本身具有视觉属性时：

- **UI原型图** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉对比** — 比较两种布局、两种配色方案、两种设计方向
- **设计润色** — 当问题涉及外观感受、间距、视觉层次时
- **空间关系** — 状态机、流程图、以图表形式呈现的实体关系

**使用终端**当内容是文本或表格时：

- **需求和范围问题** — “X是什么意思？”、“哪些功能在范围内？”
- **概念性A/B/C选择** — 在文字描述的方法之间进行选择
- **权衡列表** — 优缺点、对比表格
- **技术决策** — API设计、数据建模、架构方法选择
- **澄清性问题** — 任何答案以文字形式呈现而非视觉偏好的问题

关于UI主题的问题并不自动成为视觉问题。“你想要哪种向导？”是概念性问题——使用终端。“这些向导布局中哪个感觉合适？”是视觉问题——使用浏览器。

## 工作原理

服务器监视目录中的HTML文件，并将最新的文件提供给浏览器。你将HTML内容写入`screen_dir`，用户在浏览器中查看并可以点击选择选项。选择记录到`state_dir/events`中，你可以在下一轮读取。

**内容片段与完整文档：** 如果你的HTML文件以`<!DOCTYPE`或`<html`开头，服务器会原样提供（仅注入辅助脚本）。否则，服务器会自动将你的内容包装在框架模板中——添加页眉、CSS主题、选择指示器和所有交互基础设施。**默认情况下编写内容片段。** 仅在需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 启动带持久化的服务器（原型图保存到项目）
scripts/start-server.sh --project-dir /path/to/project

# 返回：{"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

保存响应中的`screen_dir`和`state_dir`。告诉用户打开URL。

**查找连接信息：** 服务器将其启动JSON写入`$STATE_DIR/server-info`。如果你在后台启动服务器且未捕获标准输出，请读取该文件以获取URL和端口。使用`--project-dir`时，检查`<project>/.superpowers/brainstorm/`中的会话目录。

**注意：** 将项目根目录作为`--project-dir`传递，以便原型图持久保存在`.superpowers/brainstorm/`中并在服务器重启后保留。没有此参数，文件将进入`/tmp`并被清理。提醒用户如果尚未添加，则将`.superpowers/`添加到`.gitignore`中。

**按平台启动服务器：**

**Claude Code (macOS / Linux):**
```bash
# 默认模式有效——脚本自行将服务器置于后台
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows自动检测并使用前台模式，这会阻塞工具调用。
# 在Bash工具调用上使用run_in_background: true，以便服务器在
# 跨对话轮次中保持运行。
scripts/start-server.sh --project-dir /path/to/project
```
通过Bash工具调用此命令时，设置`run_in_background: true`。然后在下一轮读取`$STATE_DIR/server-info`以获取URL和端口。

**Codex:**
```bash
# Codex会回收后台进程。脚本自动检测CODEX_CI并
# 切换到前台模式。正常运行——无需额外标志。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# 使用--foreground并在你的shell工具调用上设置is_background: true
# 以便进程在跨轮次中保持运行
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在跨对话轮次中在后台保持运行。如果你的环境回收分离的进程，请使用`--foreground`并使用你平台的背景执行机制启动命令。

如果URL无法从浏览器访问（在远程/容器化设置中常见），请绑定非环回主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用`--url-host`控制返回的URL JSON中打印的主机名。

## 循环流程

1. **检查服务器是否存活**，然后**将HTML写入**`screen_dir`中的新文件：
   - 每次写入前，检查`$STATE_DIR/server-info`是否存在。如果不存在（或`$STATE_DIR/server-stopped`存在），服务器已关闭——在继续之前使用`start-server.sh`重新启动。服务器在30分钟不活动后自动退出。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **切勿重用文件名**——每个屏幕使用新文件
   - 使用Write工具——**切勿使用cat/heredoc**（会在终端中产生噪音）
   - 服务器自动提供最新文件

2. **告诉用户预期内容并结束你的轮次：**
   - 提醒他们URL（每一步，不仅仅是第一步）
   - 简要文字总结屏幕内容（例如，“展示主页的3种布局选项”）
   - 请他们在终端中回应：“查看后告诉我你的想法。如果需要，请点击选择选项。”

3. **在你的下一轮**——用户在终端回应后：
   - 如果存在，读取`$STATE_DIR/events`——这包含用户的浏览器交互（点击、选择）作为JSON行
   - 与用户的终端文本合并以获得完整情况
   - 终端消息是主要反馈；`state_dir/events`提供结构化交互数据

4. **迭代或推进**——如果反馈改变当前屏幕，写入新文件（例如`layout-v2.html`）。仅当当前步骤验证通过后才进入下一个问题。

5. **返回终端时卸载**——当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送等待屏幕以清除过时内容：

   ```html
   <!-- 文件名：waiting.html（或waiting-2.html等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">正在终端中继续...</p>
   </div>
   ```

   这可以防止用户在对话已继续时盯着已解决的选项。当下一个视觉问题出现时，照常推送新的内容文件。

6. 重复直到完成。

## 编写内容片段

仅编写页面内部的内容。服务器会自动将其包装在框架模板中（页眉、主题CSS、选择指示器和所有交互基础设施）。

**最小示例：**

```html
<h2>哪种布局更好？</h2>
<p class="subtitle">考虑可读性和视觉层次</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单列布局</h3>
      <p>简洁、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双列布局</h3>
      <p>带主内容的侧边栏导航</p>
    </div>
  </div>
</div>
```

就这样。不需要`<html>`、CSS或`<script>`标签。服务器提供所有这些。

## 可用的CSS类

框架模板为你的内容提供以下CSS类：

### 选项（A/B/C选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>标题</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

**多选：** 向容器添加`data-multiselect`以允许用户选择多个选项。每次点击切换项目。指示条显示计数。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记——用户可以选择/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- 原型图内容 --></div>
    <div class="card-body">
      <h3>名称</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

### 原型图容器

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表板布局</div>
  <div class="mockup-body"><!-- 你的原型图HTML --></div>
</div>
```

### 分割视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>缺点</li></ul></div>
</div>
```

### 模拟元素（线框图构建块）

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主内容区域</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入字段">
<div class="placeholder">占位区域</div>
```

### 排版和部分

- `h2` — 页面标题
- `h3` — 部分标题
- `.subtitle` — 标题下方的次要文本
- `.section` — 带底部边距的内容块
- `.label` — 小型大写标签文本

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互记录到`$STATE_DIR/events`（每行一个JSON对象）。当你推送新屏幕时，文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"选项A - 简单布局","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项C - 复杂网格","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项B - 混合布局","timestamp":1706000115}
```

完整的事件流显示用户的探索路径——他们可能在确定前点击多个选项。最后的`choice`事件通常是最终选择，但点击模式可能揭示值得询问的犹豫或偏好。

如果`$STATE_DIR/events`不存在，用户未与浏览器交互——仅使用他们的终端文本。

## 设计技巧

- **根据问题调整保真度**——布局用线框图，润色问题用精细设计
- **在每个页面上解释问题**——“哪种布局感觉更专业？”而不仅仅是“选一个”
- **在推进前迭代**——如果反馈改变当前屏幕，编写新版本
- **每屏最多2-4个选项**
- **在重要时使用真实内容**——对于摄影作品集，使用实际图像（Unsplash）。占位内容会掩盖设计问题。
- **保持原型图简单**——专注于布局和结构，而非像素级完美设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 切勿重用文件名——每个屏幕必须是新文件
- 对于迭代：添加版本后缀如`layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用`--project-dir`，原型图文件将持久保存在`.superpowers/brainstorm/`中以供后续参考。仅`/tmp`会话在停止时被删除。

## 参考

- 框架模板（CSS参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
