# 视觉伴侣指南 (Visual Companion Guide)

基于浏览器的视觉头脑风暴伴侣，用于展示原型图、图表和选项。

## 何时使用

针对每个问题决定是否使用，而非针对每个会话。测试标准：**用户通过查看是否比阅读能更好地理解这一点？**

在内容本身具有视觉属性时 **使用浏览器**：

- **UI 原型图** —— 线框图、布局、导航结构、组件设计。
- **架构图** —— 系统组件、数据流、关系图。
- **并排视觉对比** —— 比较两种布局、两种配色方案、两个设计方向。
- **设计润色** —— 当问题涉及观感、间距、视觉层级时。
- **空间关系** —— 状态机、流程图、渲染为图表的实体关系。

在内容为文本或表格时 **使用终端**：

- **需求和范围问题** —— “X 是什么意思？”、“哪些功能在范围内？”
- **概念性的 A/B/C 选择** —— 在用文字描述的方法之间进行挑选。
- **权衡列表** —— 优缺点对比、对比表。
- **技术决策** —— API 设计、数据建模、架构方法选择。
- **澄清问题** —— 任何答案为文字而非视觉偏好的问题。

关于 UI 主题的问题并不自动等同于视觉问题。“你想要哪种向导？”是概念性的 —— 使用终端。“这些向导布局中哪一个感觉是对的？”是视觉的 —— 使用浏览器。

## 工作原理

服务器监控一个目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到它并可以点击选择选项。选择结果会被记录到 `state_dir/events` 中，你可以在下个回合读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器将按原样提供（仅注入助手脚本）。否则，服务器会自动将你的内容包装在框架模板中 —— 添加页眉、CSS 主题、选择指示器和所有交互基础设施。**默认情况下请编写内容片段。** 仅在需要完全控制页面时才编写完整文档。

## 开始会话

```bash
# 启动带有持久化功能的服务器（原型图保存在项目中）
scripts/start-server.sh --project-dir /path/to/project

# 返回：{"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。告诉用户打开该 URL。

**查找连接信息：** 服务器会将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器且未捕获 stdout，请读取该文件以获取 URL 和端口。使用 `--project-dir` 时，请检查 `<project>/.superpowers/brainstorm/` 寻找会话目录。

**注：** 请传递项目根目录作为 `--project-dir`，以便原型图持久化在 `.superpowers/brainstorm/` 中，并在服务器重启后依然存在。如果不传，文件将进入 `/tmp` 并在随后被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore` 中（如果尚未添加）。

**按平台启动服务器：**

**Claude Code (macOS / Linux):**
```bash
# 默认模式即可 —— 脚本本身会让服务器在后台运行
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows 会自动检测并使用前台模式，这会阻塞工具调用。
# 在 Bash 工具调用中使用 run_in_background: true，以便服务器在对话回合之间保持运行。
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash 工具调用此脚本时，请设置 `run_in_background: true`。然后在下个回合读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex:**
```bash
# Codex 会清理后台进程。脚本会自动检测 CODEX_CI 并切换到前台模式。
# 正常运行即可 —— 无需额外标志。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在 shell 工具调用中设置 is_background: true，
# 以便进程在回合之间保持运行。
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在对话回合之间持续在后台运行。如果你的环境会清理分离的进程，请使用 `--foreground` 并通过你平台的后台执行机制启动命令。

如果浏览器无法访问该 URL（在远程/容器化设置中很常见），请绑定一个非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器是否存活**，然后将 **HTML 写入** `screen_dir` 中的新文件：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），则服务器已关闭 —— 在继续之前使用 `start-server.sh` 重启它。服务器在闲置 30 分钟后会自动退出。
   - 使用具有语义的名称：`platform.html`, `visual-style.html`, `layout.html`。
   - **切勿重复使用文件名** —— 每个屏幕都对应一个新文件。
   - 使用 Write 工具 —— **切勿使用 cat/heredoc** (会向终端倾倒杂讯)。
   - 服务器会自动提供最新的文件。

2. **告诉用户预期的内容并结束你的回合：**
   - 提醒他们 URL（每一步都要提醒，不只是第一步）。
   - 简要总结屏幕上的内容（例如，“展示首页的 3 种布局选项”）。
   - 要求他们在终端响应：“请查看并告诉我你的想法。如果愿意，可以点击选择一个选项。”

3. **在你的下个回合** —— 用户在终端响应后：
   - 如果 `$STATE_DIR/events` 存在则读取它 —— 它包含 JSON 行形式的用户浏览器交互（点击、选择）。
   - 将其与用户的终端文本合并，获取完整信息。
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据。

4. **迭代或推进** —— 如果反馈改变了当前屏幕，写入一个新文件（例如 `layout-v2.html`）。只有当前步骤通过验证后才移动到下一个问题。

5. **返回终端时卸载内容** —— 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推入一个等待屏幕以清除陈旧内容：

   ```html
   <!-- 文件名: waiting.html (或 waiting-2.html, 等) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">正在终端继续...</p>
   </div>
   ```

   这可以防止用户在对话已经推进时还盯着已经解决的选择项。当下一个视觉问题出现时，照常推入新的内容文件。

6. 重复直至完成。

## 编写内容片段 (Content Fragments)

只需编写页面内部的内容。服务器会自动将其包装在框架模板中（页眉、主题 CSS、选择指示器和所有交互基础设施）。

**极简示例：**

```html
<h2>哪种布局效果更好？</h2>
<p class="subtitle">请考虑可读性和视觉层级</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单栏布局</h3>
      <p>整洁、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双栏布局</h3>
      <p>侧边栏导航配主内容区</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、CSS 或 `<script>` 标签。服务器会提供所有这些。

## 可用的 CSS 类

框架模板为你的内容提供了以下 CSS 类：

### 选项 (A/B/C 选择)

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

**多选：** 在容器上添加 `data-multiselect` 以允许用户选择多个选项。每次点击都会切换选中状态。指示条会显示计数。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记 —— 用户可以多选/取消选择 -->
</div>
```

### 卡片 (视觉设计)

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

### 原型图容器 (Mockup container)

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表盘布局</div>
  <div class="mockup-body"><!-- 你的原型图 HTML --></div>
</div>
```

### 分屏视图 (并排对比)

```html
<div class="split">
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优缺点 (Pros/Cons)

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>不足</li></ul></div>
</div>
```

### 模拟元素 (线框图构建块)

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系我们</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航栏</div>
  <div class="mock-content">主内容区</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入框">
<div class="placeholder">占位区域</div>
```

### 排版与章节

- `h2` —— 页面标题
- `h3` —— 章节标题
- `.subtitle` —— 标题下方的副标题文字
- `.section` —— 带有下边距的内容块
- `.label` —— 小型大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推入新屏幕时，该文件会自动清空。

```jsonl
{"type":"click","choice":"a","text":"选项 A - 简单布局","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项 C - 复杂网格","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项 B - 混合布局","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径 —— 他们可能会在最终确定前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击模式可以揭示犹豫或值得询问的偏好。

如果 `$STATE_DIR/events` 不存在，则用户未与浏览器交互 —— 仅使用其终端文本。

## 设计贴士

- **根据问题调整保真度** —— 针对布局使用线框图，针对观感问题使用精致的视觉。
- **在每个页面上解释问题** —— 使用“哪种布局感觉更专业？”而非仅仅是“选一个”。
- **在推进前先迭代** —— 如果反馈改变了当前屏幕，编写一个新版本。
- **每屏最多 2-4 个选项。**
- **在关键处使用真实内容** —— 比如摄影作品集，请使用真实的图片 (Unsplash)。占位内容会掩盖设计问题。
- **保持原型图简单** —— 专注于布局和结构，而非像素级的完美设计。

## 文件命名

- 使用语义化名称：`platform.html`, `visual-style.html`, `layout.html`。
- 切勿重复使用文件名 —— 每个屏幕必须是一个新文件。
- 对于迭代版本：追加版本后缀，如 `layout-v2.html`, `layout-v3.html`。
- 服务器按修改时间提供最新的文件。

## 清理工作

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，原型图文件将保留在 `.superpowers/brainstorm/` 中供以后参考。只有 `/tmp` 会话会在停止时被删除。

## 参考资料

- 框架模板 (CSS 参考)：`scripts/frame-template.html`
- 助手脚本 (客户端)：`scripts/helper.js`
