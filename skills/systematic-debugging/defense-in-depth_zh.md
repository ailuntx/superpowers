# 深度防御验证

## 概述

当你修复一个由无效数据引发的错误时，在某个位置添加验证似乎就足够了。但这一层检查可能被不同的代码路径、重构或模拟测试绕过。

**核心原则：** 在数据流经的每一层都进行验证。让错误在结构上无法发生。

## 为何需要多层验证

单层验证："我们修复了错误"
多层验证："我们让错误无法发生"

不同层级捕获不同情况：
- 入口验证捕获大多数错误
- 业务逻辑捕获边缘情况
- 环境防护阻止特定上下文中的危险操作
- 调试日志在其他层级失效时提供帮助

## 四层防御体系

### 第一层：入口点验证
**目的：** 在API边界拒绝明显无效的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... 继续执行
}
```

### 第二层：业务逻辑验证
**目的：** 确保数据对当前操作有意义

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... 继续执行
}
```

### 第三层：环境防护
**目的：** 防止在特定上下文中执行危险操作

```typescript
async function gitInit(directory: string) {
  // 在测试环境中，拒绝在临时目录外执行git init
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... 继续执行
}
```

### 第四层：调试工具
**目的：** 捕获上下文信息用于问题分析

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 继续执行
}
```

## 应用模式

发现错误时：

1. **追踪数据流** - 错误值从哪里产生？在哪里使用？
2. **映射所有检查点** - 列出数据经过的每个节点
3. **在每一层添加验证** - 入口层、业务层、环境层、调试层
4. **测试每一层** - 尝试绕过第一层，验证第二层能否捕获

## 会话示例

错误：空的`projectDir`导致在源代码目录执行`git init`

**数据流：**
1. 测试设置 → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init`在`process.cwd()`中执行

**添加的四层防御：**
- 第一层：`Project.create()`验证非空/存在/可写
- 第二层：`WorkspaceManager`验证projectDir非空
- 第三层：`WorktreeManager`在测试中拒绝临时目录外的git init
- 第四层：git init前记录堆栈跟踪

**结果：** 所有1847个测试通过，错误无法复现

## 关键洞察

所有四层防御都是必要的。在测试过程中，每一层都捕获了其他层遗漏的错误：
- 不同代码路径绕过了入口验证
- 模拟测试绕过了业务逻辑检查
- 不同平台的边缘情况需要环境防护
- 调试日志识别了结构性误用

**不要止步于单一验证点。** 在每一层都添加检查。
