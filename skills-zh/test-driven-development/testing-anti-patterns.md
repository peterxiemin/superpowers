# 测试反模式 (Testing Anti-Patterns)

**在以下情况下加载此参考资料：** 编写或更改测试、添加 mock，或者想要在生产代码中添加仅供测试使用的方法。

## 概述

测试必须验证真实的行为，而不是 mock 的行为。Mock 是实现隔离的手段，而不是被测试的对象。

**核心原则：** 测试代码做了什么，而不是 mock 做了什么。

**遵循严格的 TDD 可以防止这些反模式。**

## 铁律

```
1. 严禁测试 mock 的行为
2. 严禁在生产类中添加仅供测试使用的方法
3. 严禁在不理解依赖关系的情况下进行 mock
```

## 反模式 1：测试 mock 的行为

**违规示例：**
```typescript
// ❌ 错误：测试 mock 是否存在
test('渲染侧边栏', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么是错的：**
- 你在验证 mock 是否工作，而不是组件是否工作。
- 当 mock 存在时测试通过，不存在时测试失败。
- 它没有告诉你关于真实行为的任何信息。

**伙伴的纠正：** “我们是在测试一个 mock 的行为吗？”

**修复方案：**
```typescript
// ✅ 正确：测试真实组件或不使用 mock
test('渲染侧边栏', () => {
  render(<Page />);  // 不要 mock 侧边栏
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者，如果为了隔离必须 mock 侧边栏：
// 不要对 mock 进行断言 —— 测试 Page 在侧边栏存在时的行为
```

### 门禁机制 (Gate Function)

```
在对任何 mock 元素进行断言之前：
  询问：“我是在测试真实的组件行为，还是仅仅在测试 mock 是否存在？”

  如果是测试 mock 是否存在：
    停止 —— 删除该断言或取消对组件的 mock

  改为测试真实行为
```

## 反模式 2：生产代码中的仅供测试方法

**违规示例：**
```typescript
// ❌ 错误：destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理工作
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么是错的：**
- 生产类被仅供测试的代码污染了。
- 如果在生产环境中被意外调用会很危险。
- 违反了 YAGNI 和职责分离原则。
- 混淆了对象生命周期与实体生命周期。

**修复方案：**
```typescript
// ✅ 正确：使用测试工具处理测试清理
// Session 没有 destroy() 方法 —— 它在生产中是无状态的

// 在 test-utils/ 中
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### 门禁机制

```
在向生产类添加任何方法之前：
  询问：“这个方法是否仅由测试使用？”

  如果是：
    停止 —— 不要添加它
    将其放入测试工具 (test utilities) 中

  询问：“此类是否拥有该资源的生命周期？”

  如果不是：
    停止 —— 此方法不属于该类
```

## 反模式 3：盲目 mock (不理解原理)

**违规示例：**
```typescript
// ❌ 错误：Mock 破坏了测试逻辑
test('检测重复的服务器', () => {
  // Mock 阻止了测试所依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出异常 —— 但现在不会了！
});
```

**为什么是错的：**
- 被 mock 的方法具有测试所依赖的副作用（写入配置）。
- 为了“安全”而进行的过度 mock 破坏了实际行为。
- 测试由于错误的原因通过，或者莫名其妙地失败。

**修复方案：**
```typescript
// ✅ 正确：在正确的层级进行 mock
test('检测重复的服务器', () => {
  // 仅 mock 耗时的部分，保留测试所需的行为
  vi.mock('MCPServerManager'); // 仅 mock 缓慢的服务器启动过程

  await addServer(config);  // 配置已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 门禁机制

```
在 mock 任何方法之前：
  停止 —— 先不要 mock

  1. 询问：“真实的方法有哪些副作用？”
  2. 询问：“此测试是否依赖于其中任何副作用？”
  3. 询问：“我是否完全理解此测试的需求？”

  如果依赖于副作用：
    在较低层级进行 mock（真实的耗时/外部操作）
    或者使用保留了必要行为的测试替身 (test doubles)
    不要 mock 测试所依赖的高层方法

  如果不确定测试依赖什么：
    首先使用真实实现运行测试
    观察实际需要发生什么
    然后在正确的层级添加最小化的 mock

  红旗信号：
    - “为了安全起见，我 mock 一下这个”
    - “这可能很慢，最好 mock 掉”
    - 在不理解依赖链的情况下进行 mock
```

## 反模式 4：不完整的 mock

**违规示例：**
```typescript
// ❌ 错误：部分 mock —— 仅包含你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的元数据 (metadata)
};

// 稍后：当代码访问 response.metadata.requestId 时崩溃
```

**为什么是错的：**
- **部分 mock 掩盖了结构性假设** —— 你只 mock 了你知道的字段。
- **下游代码可能依赖于你未包含的字段** —— 导致静默失败。
- **测试通过但集成失败** —— Mock 不完整，而真实 API 是完整的。
- **虚假的信心** —— 测试没有证明关于真实行为的任何事情。

**铁律：** Mock 现实中存在的完整数据结构，而不仅仅是当前测试用到的字段。

**修复方案：**
```typescript
// ✅ 正确：镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 门禁机制

```
在创建 mock 响应之前：
  检查：“真实的 API 响应包含哪些字段？”

  行动：
    1. 从文档/示例中检查实际的 API 响应
    2. 包含系统在下游可能消费的所有字段
    3. 验证 mock 是否完全符合真实响应的 schema

  关键点：
    如果你在创建 mock，你必须理解整个结构
    当代码依赖于遗漏的字段时，部分 mock 会导致静默失败

  如果不确定：包含所有文档中提到的字段
```

## 反模式 5：将集成测试视为事后工作

**违规示例：**
```
✅ 实施完成
❌ 未编写测试
“准备好测试了”
```

**为什么是错的：**
- 测试是实施过程的一部分，而非可选的后续工作。
- TDD 本可以捕获到这一点。
- 没写测试就不能声称已完成。

**修复方案：**
```
TDD 循环：
1. 编写失败的测试
2. 实施代码使之通过
3. 重构
4. 然后声称已完成
```

## 当 mock 变得过于复杂时

**预警信号：**
- Mock 的设置比测试逻辑还要长。
- 为了让测试通过而 mock 了一切。
- Mock 缺失了真实组件拥有的方法。
- 当 mock 更改时测试崩溃。

**伙伴的问题：** “这里真的需要使用 mock 吗？”

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单。

## TDD 可以防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么。
2. **观察它失败** → 确认测试的是真实行为而非 mock。
3. **最小化实施** → 避免在生产代码中混入仅供测试的方法。
4. **真实依赖** → 你在 mock 之前就能看到测试实际需要什么。

**如果你在测试 mock 的行为，你就违反了 TDD** —— 你在没有看到测试在真实代码上失败之前就添加了 mock。

## 快速参考

| 反模式 | 修复方案 |
|--------------|-----|
| 对 mock 元素进行断言 | 测试真实组件或取消 mock |
| 生产代码中的仅供测试方法 | 移至测试工具类 |
| 盲目 mock (不理解原理) | 先理解依赖关系，进行最小化 mock |
| 不完整的 mock | 镜像完整真实的 API |
| 将测试视为事后工作 | TDD —— 先写测试 |
| 过度复杂的 mock | 考虑集成测试 |

## 红旗信号

- 断言检查 `*-mock` 测试 ID。
- 仅在测试文件中被调用的方法。
- Mock 设置占测试内容的 >50%。
- 移除 mock 后测试失败。
- 无法解释为什么需要 mock。
- “仅仅为了安全起见”而进行的 mock。

## 底线

**Mock 是用于隔离的工具，而不是被测试的对象。**

如果 TDD 揭示出你在测试 mock 的行为，那你就走错路了。

修复：测试真实行为，或者质疑你为什么要进行 mock。
