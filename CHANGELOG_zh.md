# 更新日志

## [5.0.5] - 2026-03-17

### 修复

- **Brainstorm 服务器 ESM 修复**：将 `server.js` 重命名为 `server.cjs`，使头脑风暴服务器能在 Node.js 22+ 上正确启动。此前根目录 `package.json` 中的 `"type": "module"` 导致 `require()` 失败。（[PR #784](https://github.com/obra/superpowers/pull/784) 由 @sarbojitrana 提交，修复了 [#774](https://github.com/obra/superpowers/issues/774)、[#780](https://github.com/obra/superpowers/issues/780)、[#783](https://github.com/obra/superpowers/issues/783)）
- **Brainstorm 在 Windows 上的所有者 PID 处理**：在 Windows/MSYS2 环境下跳过 `BRAINSTORM_OWNER_PID` 生命周期监控，因为 Node.js 无法访问其 PID 命名空间。这防止了服务器在 60 秒后自行终止。30 分钟的空闲超时仍作为安全机制保留。（[#770](https://github.com/obra/superpowers/issues/770)，文档来自 [PR #768](https://github.com/obra/superpowers/pull/768) 由 @lucasyhzhu-debug 提交）
- **stop-server.sh 可靠性提升**：在报告成功前验证服务器进程确实已终止。最多等待 2 秒以进行优雅关闭，然后升级为 `SIGKILL`，如果进程仍然存活则报告失败。（[#723](https://github.com/obra/superpowers/issues/723)）

### 变更

- **执行交接**：恢复用户在计划编写后选择子代理驱动开发或执行计划的功能。子代理驱动模式仍为推荐选项，但不再是强制要求。（撤销 `5e51c3e` 提交）
