---
name: requesting-code-review
description: 在完成任务、实施重要功能或合并前使用，以验证工作是否满足要求。
---

# 请求代码审查 (Requesting Code Review)

派发 superpowers:code-reviewer 子代理，在问题级联扩散之前将其捕获。审查者获得为评估精心构造的上下文 —— 绝不是你当前会话的历史记录。这让审查者专注于工作成果本身，而非你的思考过程，同时也为你保留了用于继续工作的上下文。

**核心原则：** 早审查，勤审查。

## 何时请求审查

**必须：**
- 在子代理驱动开发中每个任务完成后。
- 完成重大功能后。
- 合并到 main 之前。

**可选但有价值：**
- 陷入困境时（换个视角）。
- 重构之前（建立基线）。
- 修复复杂 bug 之后。

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派发 code-reviewer 子代理：**

使用 Task 工具，类型为 superpowers:code-reviewer，填充 `code-reviewer.md` 中的模板。

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` —— 你刚刚构建了什么。
- `{PLAN_OR_REQUIREMENTS}` —— 它应该做什么。
- `{BASE_SHA}` —— 起始提交。
- `{HEAD_SHA}` —— 结束提交。
- `{DESCRIPTION}` —— 简要摘要。

**3. 根据反馈行动：**
- 立即修复严重 (Critical) 问题。
- 在继续之前修复重要 (Important) 问题。
- 记录次要 (Minor) 问题留待以后处理。
- 如果审查者判断有误，提出有理有据的推辞。

## 示例

```
[刚刚完成任务 2：添加验证函数]

你：在继续之前，我先请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派发 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: 对话索引的验证与修复函数
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，支持 4 种问题类型

[子代理返回结果]：
  优点：架构清晰，有真实测试
  问题：
    重要：缺少进度指示器
    次要：报告间隔使用了魔法数字 (100)
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流的集成

**子代理驱动开发：**
- 每个任务后都进行审查。
- 在问题累积之前将其捕获。
- 在进入下一个任务前进行修复。

**执行计划：**
- 每个批次（3 个任务）后进行审查。
- 获取反馈、应用、继续。

**临时 (Ad-Hoc) 开发：**
- 合并前审查。
- 陷入困境时审查。

## 红旗信号

**绝不：**
- 因为“很简单”而跳过审查。
- 忽略严重 (Critical) 问题。
- 在未修复重要 (Important) 问题的情况下继续。
- 与有效的技术反馈争辩。

**如果审查者错了：**
- 通过技术推理进行推辞。
- 出示能证明其有效的代码/测试。
- 请求澄清。

请参阅模板：`requesting-code-review/code-reviewer.md`
