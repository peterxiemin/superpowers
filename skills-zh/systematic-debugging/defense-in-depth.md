# 深度防御验证 (Defense-in-Depth Validation)

## 概述

当你修复了一个由无效数据引起的 bug 时，在某一处添加验证似乎就足够了。但这一道检查可能会被不同的代码路径、重构或 mock 所绕过。

**核心原则：** 在数据经过的每一层都进行验证。使该 bug 在结构上变得不可能发生。

## 为什么需要多层防御

单层验证：“我们修复了 bug”。
多层防御：“我们使 bug 变得不可能发生”。

不同的层级捕获不同的情况：
- 入口验证捕获大多数 bug。
- 业务逻辑捕获边缘情况。
- 环境守卫防止特定上下文下的危险操作。
- 调试日志在其他层级失效时提供帮助。

## 四个层级

### 第一层：入口点验证 (Entry Point Validation)
**目的：** 在 API 边界拒绝明显无效的输入。

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory 不能为空');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory 不存在: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory 不是一个目录: ${workingDirectory}`);
  }
  // ... 继续执行
}
```

### 第二层：业务逻辑验证 (Business Logic Validation)
**目的：** 确保数据对于此操作是有意义的。

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('初始化工作空间需要 projectDir');
  }
  // ... 继续执行
}
```

### 第三层：环境守卫 (Environment Guards)
**目的：** 防止在特定上下文中执行危险操作。

```typescript
async function gitInit(directory: string) {
  // 在测试中，拒绝在临时目录之外执行 git init
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `拒绝在测试期间在临时目录之外执行 git init: ${directory}`
      );
    }
  }
  // ... 继续执行
}
```

### 第四层：调试监测 (Debug Instrumentation)
**目的：** 为事后分析捕获上下文。

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('准备执行 git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 继续执行
}
```

## 应用此模式

当你发现一个 bug 时：

1. **追踪数据流** —— 错误值源自何处？在哪里被使用？
2. **绘制所有检查点** —— 列出数据经过的每一个点。
3. **在每一层添加验证** —— 入口层、业务层、环境层、调试层。
4. **测试每一层** —— 尝试绕过第一层，验证第二层是否能捕获它。

## 会话中的示例

Bug：空的 `projectDir` 导致在源代码中执行了 `git init`。

**数据流：**
1. 测试设置 → 空字符串。
2. `Project.create(name, '')`。
3. `WorkspaceManager.createWorkspace('')`。
4. `git init` 在 `process.cwd()` 中运行。

**增加了四层防御：**
- 第 1 层：`Project.create()` 验证非空/存在/可写。
- 第 2 层：`WorkspaceManager` 验证 projectDir 非空。
- 第 3 层：`WorktreeManager` 拒绝在测试期间在 tmpdir 之外执行 git init。
- 第 4 层：在 git init 之前记录堆栈跟踪日志。

**结果：** 所有 1847 个测试通过，bug 无法复现。

## 核心见解

所有四层防御都是必要的。在测试期间，每一层都捕获了其他层漏掉的 bug：
- 不同的代码路径绕过了入口验证。
- Mock 绕过了业务逻辑检查。
- 不同平台上的边缘情况需要环境守卫。
- 调试日志识别了结构性的误用。

**不要止步于一个验证点。** 在每一层都添加检查。
