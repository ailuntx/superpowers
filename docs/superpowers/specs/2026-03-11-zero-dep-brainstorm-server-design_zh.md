# 零依赖头脑风暴服务器

将头脑风暴伴侣服务器的打包 node_modules（express、ws、chokidar — 714个跟踪文件）替换为仅使用 Node.js 内置模块的单个零依赖 `server.js` 文件。

## 动机

将 node_modules 打包到 git 仓库会带来供应链风险：冻结的依赖无法获得安全补丁，714个第三方代码文件未经审计就被提交，对打包代码的修改看起来像普通提交。虽然实际风险较低（仅限本地开发服务器），但消除这种风险是直接了当的。

## 架构

单个 `server.js` 文件（约250-300行），使用 `http`、`crypto`、`fs` 和 `path` 模块。该文件承担两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被引用时**（`require('./server.js')`）：导出 WebSocket 协议函数供单元测试使用

### WebSocket 协议

仅实现 RFC 6455 文本帧：

**握手：** 使用 SHA-1 + RFC 6455 魔术 GUID 从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 切换协议。

**帧解码（客户端到服务器）：** 处理三种掩码长度编码：
- 小型：有效载荷 < 126 字节
- 中型：126-65535 字节（16位扩展）
- 大型：> 65535 字节（64位扩展）

使用4字节掩码密钥对有效载荷进行 XOR 解掩码。返回 `{ opcode, payload, bytesConsumed }` 或 `null`（对于不完整的缓冲区）。拒绝未掩码的帧。

**帧编码（服务器到客户端）：** 未掩码的帧，使用相同的三种长度编码。

**处理的操作码：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。无法识别的操作码会收到状态为 1003（不支持的数据）的关闭帧。

**故意跳过的功能：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对于本地客户端之间的小型 JSON 文本消息是不必要的。扩展和子协议在握手时协商 — 通过不声明它们，它们永远不会被激活。

**缓冲区累积：** 每个连接维护一个缓冲区。在 `data` 事件时，追加数据并循环调用 `decodeFrame`，直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`** — 按修改时间从屏幕目录提供最新的 `.html` 文件。检测完整文档与片段，将片段包装在框架模板中，注入 helper.js。返回 `text/html`。当不存在 `.html` 文件时，提供硬编码的等待页面（"等待 Claude 推送屏幕..."）并注入 helper.js。
2. **`GET /files/*`** — 从屏幕目录提供静态文件，使用硬编码的扩展名映射（html、css、js、png、jpg、gif、svg、json）查找 MIME 类型。如果未找到则返回 404。
3. **其他所有请求** — 404。

WebSocket 升级通过 HTTP 服务器的 `'upgrade'` 事件处理，与请求处理器分开。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT` — 绑定的端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定的接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 启动 JSON 中 URL 的主机名（默认：当主机为 `127.0.0.1` 时为 `localhost`，否则与主机相同）
- `BRAINSTORM_DIR` — 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 如果不存在则创建 `SCREEN_DIR`（`mkdirSync` 递归）
2. 从 `__dirname` 加载框架模板和 helper.js
3. 在配置的主机/端口上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 成功监听后，将 `server-started` JSON 记录到 stdout：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便在 stdout 被隐藏时（后台执行）代理可以找到连接详情

### 应用级 WebSocket 消息

当从客户端收到 TEXT 帧时：

1. 解析为 JSON。如果解析失败，记录到 stderr 并继续。
2. 记录到 stdout 为 `{ source: 'user-event', ...event }`。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每个事件一行）。

### 文件监视

`fs.watch(SCREEN_DIR)` 替换 chokidar。HTML 文件事件：

- 新文件时（`rename` 事件对应存在的文件）：如果存在则删除 `.events` 文件（`unlinkSync`），将 `screen-added` 记录到 stdout 为 JSON
- 文件更改时（`change` 事件）：将 `screen-updated` 记录到 stdout 为 JSON（不清除 `.events`）
- 两个事件：向所有连接的 WebSocket 客户端发送 `{ type: 'reload' }`

按文件名进行约100ms超时的防抖，以防止重复事件（在 macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端的格式错误 JSON：记录到 stderr，继续
- 未处理的操作码：以状态 1003 关闭
- 客户端断开连接：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 无优雅关闭逻辑 — shell 脚本通过 SIGTERM 处理进程生命周期

## 变更内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714个 `node_modules` 文件 | `server.js`（单个文件） |
| express、ws、chokidar 依赖 | 无依赖 |
| 无静态文件服务 | `/files/*` 从屏幕目录提供服务 |

## 保持不变的内容

- `helper.js` — 无变化
- `frame-template.html` — 无变化
- `start-server.sh` — 单行更新：`index.js` 改为 `server.js`
- `stop-server.sh` — 无变化
- `visual-companion.md` — 无变化
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台的 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上对单个扁平目录可靠
- Shell 脚本需要 bash（Windows 上需要 Git Bash，这是 Claude Code 所要求的）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过引用 `server.js` 导出的函数，直接测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整的服务器行为 — HTTP 服务、WebSocket 通信、文件监视、头脑风暴工作流。使用 `ws` npm 包作为仅测试的客户端依赖（不提供给最终用户）。
