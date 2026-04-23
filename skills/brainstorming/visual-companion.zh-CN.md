# 视觉辅助指南

这是一个基于浏览器的可视化头脑风暴辅助工具，用来展示 mockup、图示和候选方案。

## 何时使用

要按“每个问题”来判断，而不是按“整次会话”来判断。判断标准是：**用户通过看它，会不会比读文字更容易理解？**

**当内容本身是视觉性的时，使用浏览器：**

- **UI mockup**：线框图、布局、导航结构、组件设计
- **架构图**：系统组件、数据流、关系图
- **并排视觉对比**：比较两种布局、两套配色、两种设计方向
- **设计打磨**：当问题聚焦于观感、间距、视觉层级时
- **空间关系**：状态机、流程图、实体关系图这类更适合画出来的内容

**当内容是文本或表格时，使用终端：**

- **需求和范围问题**：例如“X 是什么意思？”、“哪些功能在范围内？”
- **概念性的 A/B/C 选择**：用文字描述不同方案，然后做选择
- **取舍列表**：优缺点、对比表
- **技术决策**：API 设计、数据建模、架构方案选择
- **澄清问题**：任何答案本质上是文字，而不是视觉偏好的问题

一个“关于 UI 的问题”并不自动等于“视觉问题”。例如“你想要什么样的向导流程？”是概念问题，应使用终端；“这几种向导布局哪一种感觉更对？”才是视觉问题，应使用浏览器。

## 它是怎么工作的

服务端会监视一个目录里的 HTML 文件，并把最新的那个页面提供给浏览器。你把 HTML 内容写到 `screen_dir`，用户在浏览器里看到内容并可以点击选择。选择结果会记录到 `state_dir/events`，你在下一轮读取它。

**内容片段 vs 完整 HTML 文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务端就按原样提供（只会额外注入辅助脚本）。否则，服务端会自动把你的内容包进统一的页面框架中，自动加上页头、CSS 主题、选择指示器和交互基础设施。**默认应写“内容片段”。** 只有在你需要完全掌控整个页面时，才写完整文档。

## 启动一个会话

```bash
# 启动服务，并把会话内容持久化到项目目录中
scripts/start-server.sh --project-dir /path/to/project

# 返回示例：{"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

保存返回结果里的 `screen_dir` 和 `state_dir`。然后告诉用户打开返回的 URL。

**如何找到连接信息：** 服务端会把启动时的 JSON 写到 `$STATE_DIR/server-info`。如果你是后台启动服务但没拿到 stdout，可以读这个文件来获取 URL 和端口。如果使用了 `--project-dir`，会话目录会在 `<project>/.superpowers/brainstorm/` 下。

**注意：** 传入项目根目录作为 `--project-dir`，这样 mockup 会保存在 `.superpowers/brainstorm/` 中，服务端重启后也还在。不传的话，文件会写到 `/tmp`，后续会被清理掉。还要提醒用户把 `.superpowers/` 加进 `.gitignore`，如果项目里还没有的话。

**不同平台如何启动服务：**

**Claude Code（macOS / Linux）：**
```bash
# 默认模式即可，脚本会自己把服务放到后台
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code（Windows）：**
```bash
# Windows 会自动检测并改用前台模式，这会阻塞工具调用。
# 调用 Bash 工具时要设置 run_in_background: true，
# 这样服务才能跨会话存活。
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash 工具调用时，设置 `run_in_background: true`。然后在下一轮读取 `$STATE_DIR/server-info` 来拿到 URL 和端口。

**Codex：**
```bash
# Codex 会清理后台进程。脚本会自动检测 CODEX_CI，
# 然后切换到前台模式。正常运行即可，不需要额外参数。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI：**
```bash
# 使用 --foreground，并在 shell 工具调用上设置 is_background: true，
# 这样进程才能跨轮次存活。
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务端必须能在多轮对话之间持续运行。如果你的环境会清理脱离终端的后台进程，就用 `--foreground` 并通过你所在平台支持的后台运行机制来启动它。

如果浏览器无法访问这个 URL（远程环境、容器环境里很常见），要绑定到非 loopback 地址：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 来控制返回的 URL JSON 里显示什么主机名。

## 使用循环

1. **先确认服务还活着**，然后**往 `screen_dir` 写一个新的 HTML 文件**：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），说明服务已关闭，需要先重新运行 `start-server.sh` 再继续。服务端在空闲 30 分钟后会自动退出。
   - 文件名要有语义，例如：`platform.html`、`visual-style.html`、`layout.html`
   - **不要复用文件名**，每个 screen 都要是新的文件
   - 使用 Write 工具，**不要用 cat/heredoc**（会把大量噪音输出到终端）
   - 服务端会自动把最新的文件提供给浏览器

2. **告诉用户他们会看到什么，并结束当前轮次：**
   - 每一步都提醒他们当前的 URL，不只是第一次
   - 用简短文字概括屏幕内容（例如“现在展示首页的 3 种布局方案”）
   - 请他们在终端中回复：`Take a look and let me know what you think. Click to select an option if you'd like.`

3. **下一轮**，等用户在终端回复后：
   - 如果 `$STATE_DIR/events` 存在，就读取它。里面是用户在浏览器中的交互记录（点击、选择），按 JSON Lines 格式保存
   - 把这些结构化事件和用户在终端中的文字反馈合并起来理解
   - 终端消息仍然是主反馈来源；`state_dir/events` 只是补充结构化交互数据

4. **迭代或前进**：如果反馈会改变当前 screen，就写一个新文件（例如 `layout-v2.html`）。只有当前步骤确认无误后，才进入下一个问题。

5. **回到终端讨论时，要卸载浏览器内容**：如果下一步不需要浏览器（例如澄清问题、取舍讨论），就推送一个“等待页”来清掉旧内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这样可以避免用户一直盯着已经讨论完的旧页面。等下一个视觉问题出现时，再像平常一样推送新内容。

6. 重复这个循环，直到结束。

## 如何写“内容片段”

只写页面中间那块内容即可。服务端会自动把它包进统一框架，补上页头、主题 CSS、选择指示器和交互基础设施。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就这些。不需要 `<html>`、CSS，也不需要 `<script>` 标签。服务端都会帮你补好。

## 可用的 CSS 类

页面框架会为你的内容提供这些 CSS 类：

### Options（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 给容器加上 `data-multiselect`，用户就可以选多个。每次点击会切换选中状态，指示条会显示已选数量。

```html
<div class="options" data-multiselect>
  <!-- 标记结构不变，用户可以选择/取消多个选项 -->
</div>
```

### Cards（展示视觉方案）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### Split View（左右对比）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### Pros/Cons

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元素（线框图积木）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版与区块

- `h2`：页面标题
- `h3`：分区标题
- `.subtitle`：标题下方的次级说明文字
- `.section`：底部有间距的内容块
- `.label`：小号、大写、标签风格的文字

## 浏览器事件格式

当用户在浏览器里点击选项时，交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新的 screen 时，这个文件会被自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流能告诉你用户探索过哪些选项，他们可能会在最终决定前点好几个。最后一个 `choice` 事件通常就是最终选择，但整个点击路径也可能透露出犹豫或偏好，值得继续追问。

如果 `$STATE_DIR/events` 不存在，说明用户没有在浏览器中进行交互，此时只看他们在终端里的文字反馈即可。

## 设计建议

- **保真度要和问题匹配**：布局问题用线框图，打磨问题再上更精细的视觉
- **每个页面都要把问题写清楚**：比如“哪个布局看起来更专业？”而不是只写“选一个”
- **先迭代，再前进**：如果反馈改变了当前页面，就先写新版本，不要急着进入下一题
- **每屏最多 2-4 个选项**
- **该用真实内容时就用真实内容**：例如做摄影作品集时，就应该放真实图片（比如 Unsplash）。全是占位内容会掩盖真实设计问题。
- **mockup 要保持简洁**：重点是布局和结构，不是像素级还原

## 文件命名

- 用有语义的名字：`platform.html`、`visual-style.html`、`layout.html`
- 不要重复使用文件名：每个 screen 都必须是新文件
- 如果是迭代版本，就追加版本后缀，例如 `layout-v2.html`、`layout-v3.html`
- 服务端按修改时间提供最新的文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话用了 `--project-dir`，mockup 文件会保留在 `.superpowers/brainstorm/` 中，方便之后复盘。只有 `/tmp` 下的临时会话会在停止时被删除。

## 参考

- 页面框架模板（CSS 参考）：`scripts/frame-template.html`
- 客户端辅助脚本：`scripts/helper.js`
