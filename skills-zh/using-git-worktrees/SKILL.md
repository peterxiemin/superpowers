---
name: using-git-worktrees
description: 在开始需要与当前工作区隔离的功能开发或执行实施计划之前使用 —— 创建具有智能目录选择和安全验证的隔离 Git 工作树。
---

# 使用 Git 工作树 (Using Git Worktrees)

## 概述

Git 工作树 (worktrees) 创建共享同一仓库的隔离工作空间，允许在不切换分支的情况下同时处理多个分支。

**核心原则：** 系统化的目录选择 + 安全验证 = 可靠的隔离。

**开始时宣布：** “我正在使用 using-git-worktrees 技能来设置隔离的工作空间。”

## 目录选择流程

遵循以下优先级顺序：

### 1. 检查现有目录

```bash
# 按优先级顺序检查
ls -d .worktrees 2>/dev/null     # 首选（隐藏目录）
ls -d worktrees 2>/dev/null      # 备选
```

**如果找到：** 使用该目录。如果两者都存在，`.worktrees` 胜出。

### 2. 检查 CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**如果指定了偏好：** 直接使用，无需询问。

### 3. 询问用户

如果不存在任何目录且 CLAUDE.md 中没有偏好设置：

```
未找到工作树目录。我应该在哪里创建工作树？

1. .worktrees/ (项目本地，隐藏)
2. ~/.config/superpowers/worktrees/<project-name>/ (全局位置)

你更倾向于哪一个？
```

## 安全验证

### 针对项目本地目录 (.worktrees 或 worktrees)

**在创建工作树之前，必须验证目录是否已被忽略 (ignored)：**

```bash
# 检查目录是否被忽略（尊重本地、全局和系统的 gitignore 设置）
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：**

根据 Jesse 的规则“立即修复损坏的东西”：
1. 向 `.gitignore` 添加相应的行。
2. 提交更改。
3. 继续创建工作树。

**为何至关重要：** 防止意外将工作树内容提交到仓库。

### 针对全局目录 (~/.config/superpowers/worktrees)

无需验证 `.gitignore` —— 它完全位于项目之外。

## 创建步骤

### 1. 检测项目名称

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. 创建工作树

```bash
# 确定完整路径
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# 创建带有新分支的工作树
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. 运行项目初始化

自动检测并运行相应的初始化命令：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. 验证干净的基准线 (Baseline)

运行测试以确保工作树起始状态是干净的：

```bash
# 示例 —— 使用适合项目的命令
npm test
cargo test
pytest
go test ./...
```

**如果测试失败：** 报告失败情况，询问是继续还是先进行调查。

**如果测试通过：** 报告已就绪。

### 5. 报告位置

```
工作树已在 <full-path> 就绪
测试通过（<N> 个测试通过，0 个失败）
准备好实施 <feature-name> 功能
```

## 快速参考

| 情况 | 行动 |
|-----------|--------|
| `.worktrees/` 已存在 | 使用它（验证是否被忽略） |
| `worktrees/` 已存在 | 使用它（验证是否被忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 都不存在 | 检查 CLAUDE.md → 询问用户 |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 基准线测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 跳过忽略验证

- **问题：** 工作树内容被追踪，污染 git 状态。
- **修复：** 在创建项目本地工作树之前，始终使用 `git check-ignore`。

### 臆断目录位置

- **问题：** 造成不一致，违反项目惯例。
- **修复：** 遵循优先级：现有目录 > CLAUDE.md > 询问。

### 在测试失败的情况下继续

- **问题：** 无法区分新产生的 bug 和预先存在的旧问题。
- **修复：** 报告失败，获得明确许可后再继续。

### 硬编码初始化命令

- **问题：** 在使用不同工具的项目中失效。
- **修复：** 从项目文件（package.json 等）中自动检测。

## 示例工作流

```
你：我正在使用 using-git-worktrees 技能来设置隔离的工作空间。

[检查 .worktrees/ - 存在]
[验证忽略情况 - git check-ignore 确认 .worktrees/ 已被忽略]
[创建工作树：git worktree add .worktrees/auth -b feature/auth]
[运行 npm install]
[运行 npm test - 47 个通过]

工作树已在 /Users/jesse/myproject/.worktrees/auth 就绪
测试通过（47 个测试通过，0 个失败）
准备好实施身份验证 (auth) 功能
```

## 红旗信号

**绝不：**
- 在未验证项目本地工作树目录是否被忽略的情况下创建它。
- 跳过基准线测试验证。
- 未经询问就在测试失败的情况下继续。
- 在位置含糊不清时臆断目录位置。
- 跳过 CLAUDE.md 检查。

**始终：**
- 遵循目录优先级：现有 > CLAUDE.md > 询问。
- 验证项目本地目录是否被忽略。
- 自动检测并运行项目初始化。
- 验证干净的测试基准线。

## 集成

**调用方：**
- **brainstorming** (第四阶段) —— 在设计获得批准且随后要进行实施时是必需的。
- **subagent-driven-development** —— 在执行任何任务之前是必需的。
- **executing-plans** —— 在执行任何任务之前是必需的。
- 任何需要隔离工作空间的技能。

**配对技能：**
- **finishing-a-development-branch** —— 必需用于工作完成后进行清理。
