---
name: finishing-a-development-branch
description: 在实现已完成、所有测试通过、需要决定如何整合工作时使用——通过呈现合并、PR 或清理的结构化选项，指导开发工作的收尾
---

# 结束一个开发分支

## 概述

通过呈现清晰的选项并处理所选工作流，来指导开发工作的收尾。

**核心原则：** 验证测试 → 呈现选项 → 执行选择 → 清理。

**开始时声明：** "我正在使用 finishing-a-development-branch 技能来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在呈现选项之前，先验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**若测试失败：**
```
测试失败（<N> 个失败）。完成前必须修复：

[显示失败信息]

在测试通过之前无法进行合并/PR。
```

停下。不要进入步骤 2。

**若测试通过：** 继续步骤 2。

### 步骤 2：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："这个分支是从 main 切出来的——对吗？"

### 步骤 3：呈现选项

严格呈现以下 4 个选项：

```
实现已完成。你想怎么做？

1. 在本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支现状（我稍后处理）
4. 丢弃这份工作

选哪个？
```

**不要附加解释** —— 保持选项简洁。

### 步骤 4：执行选择

#### 选项 1：本地合并

```bash
# 切换到基础分支
git checkout <base-branch>

# 拉取最新
git pull

# 合并特性分支
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 若测试通过
git branch -d <feature-branch>
```

然后：清理 worktree（步骤 5）

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 条关于变更内容的要点>

## Test Plan
- [ ] <验证步骤>
EOF
)"
```

然后：清理 worktree（步骤 5）

#### 选项 3：保持现状

报告："保留分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>
- Worktree：<path>

输入 'discard' 以确认。
```

等待精确的确认输入。

若已确认：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后：清理 worktree（步骤 5）

### 步骤 5：清理 Worktree

**对选项 1、2、4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

若是：
```bash
git worktree remove <worktree-path>
```

**对选项 3：** 保留 worktree。

## 速查表

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | ✓ | - | - | ✓ |
| 2. 创建 PR | - | ✓ | ✓ | - |
| 3. 保持现状 | - | - | ✓ | - |
| 4. 丢弃 | - | - | - | ✓（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并了坏掉的代码，创建出失败的 PR
- **修复：** 在提供选项前务必验证测试

**开放式问题**
- **问题：** "接下来我该做什么？" → 含糊不清
- **修复：** 严格呈现 4 个结构化选项

**自动清理 worktree**
- **问题：** 在可能还需要的时候删除 worktree（选项 2、3）
- **修复：** 仅在选项 1 和 4 时清理

**丢弃时未确认**
- **问题：** 意外删除了工作成果
- **修复：** 要求输入 "discard" 进行确认

## 红旗信号

**绝不：**
- 在测试失败的情况下继续推进
- 不在结果上验证测试就合并
- 未经确认就删除工作
- 未经明确请求就强制推送

**始终：**
- 在提供选项前验证测试
- 严格呈现 4 个选项
- 对选项 4 要求输入 "discard" 确认
- 仅对选项 1 和 4 清理 worktree

## 整合

**被调用于：**
- **subagent-driven-development**（步骤 7）——所有任务完成之后
- **executing-plans**（步骤 5）——所有批次完成之后

**配套使用：**
- **using-git-worktrees** —— 清理由该技能创建的 worktree
