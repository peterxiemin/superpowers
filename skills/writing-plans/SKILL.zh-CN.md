---
name: writing-plans
description: "当你已经有 spec 或一项多步骤任务的需求，并且在动代码之前需要先产出实现计划时使用。"
---

# 编写实现计划

## 概述

编写完整的 implementation plan 时，要假设执行工程师对我们的代码库几乎没有上下文，而且工程判断也未必可靠。把他们需要知道的一切写清楚：每个任务要碰哪些文件、可能需要查看哪些代码、测试和文档、以及应该如何验证。把整个计划拆成一口一口可执行的小任务。DRY。YAGNI。TDD。频繁提交。

假设他们是熟练开发者，但对我们的工具链和问题领域几乎一无所知。也假设他们不太擅长测试设计。

**开始时要声明：** `I'm using the writing-plans skill to create the implementation plan.`

**上下文：** 这个 skill 应该运行在独立 worktree 中（由 `brainstorming` skill 创建）。

**计划保存位置：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （如果用户对计划存放位置有偏好，以用户要求为准）

## 范围检查

如果 spec 覆盖了多个相互独立的子系统，那么它本应在 brainstorming 阶段就被拆成多个子项目 spec。如果还没有拆，此时应建议拆成多份独立计划，每个子系统一份。每一份计划都应该能独立产出可运行、可测试的软件。

## 文件结构

在定义任务之前，先梳理要创建或修改哪些文件，以及每个文件负责什么。这一步会把拆分决策固定下来。

- 设计边界清晰、接口明确的单元。每个文件都应只有一个清晰职责。
- 你对能一次放进上下文的代码推理效果最好，文件越聚焦，修改越可靠。优先选择小而专注的文件，而不是职责过多的大文件。
- 会一起变化的文件应该放在一起。按职责拆分，而不是按技术层拆分。
- 在已有代码库里，要遵循现有模式。如果代码库本来就使用大文件，不要擅自重构；但如果你正在修改的文件已经失控，把拆分写进计划是合理的。

这个结构会影响后续任务拆分。每个任务都应该形成一组自洽、可独立理解的改动。

## 小任务粒度

**每一步都只能是一个动作（2 到 5 分钟）：**
- “写出失败的测试” 是一步
- “运行测试并确认它失败” 是一步
- “写最小实现让测试通过” 是一步
- “运行测试并确认通过” 是一步
- “提交” 是一步

## 计划文档头部

**每一份计划都必须以这个头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 不要写占位符

每一步都必须包含工程师真正需要的内容。下面这些都属于 **计划失败**，绝对不要写：
- `TBD`、`TODO`、`implement later`、`fill in details`
- `Add appropriate error handling` / `add validation` / `handle edge cases`
- `Write tests for the above`（但没有实际测试代码）
- `Similar to Task N`（要把代码重复写出来，因为执行者可能是跳着读的）
- 只说“做什么”却不展示“怎么做”的步骤（凡是代码步骤，都必须有代码块）
- 引用了某个类型、函数或方法，但它没有在任何任务里被定义

## 记住
- 永远写精确文件路径
- 每一步都给出完整代码，如果这一步涉及改代码，就把代码写出来
- 命令必须精确，并写出预期输出
- 遵循 DRY、YAGNI、TDD，频繁提交

## 自审

写完整份计划后，要带着新鲜视角重新看 spec，并对照检查这份计划。这是你自己执行的检查清单，不是派发 subagent。

**1. Spec 覆盖度：** 快速浏览 spec 的每个章节和要求。你是否能指出对应实现它的任务？把所有缺口列出来。

**2. 占位符扫描：** 在计划里搜索红旗模式，也就是上面“不要写占位符”那一节中的内容。发现了就修掉。

**3. 类型一致性：** 你在后面任务里使用的类型、方法签名和属性名，是否和前面任务里定义的一致？例如 Task 3 里叫 `clearLayers()`，Task 7 里却变成 `clearFullLayers()`，这就是 bug。

如果发现问题，就直接内联修复。不需要重新审一轮，修完继续。如果发现 spec 里某项要求没有对应任务，就补上任务。

## 执行交接

计划保存后，要给出执行方式选择：

**`Plan complete and saved to docs/superpowers/plans/<filename>.md. Two execution options:`**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**`Which approach?`**

**如果用户选择 Subagent-Driven：**
- **REQUIRED SUB-SKILL:** 使用 `superpowers:subagent-driven-development`
- 每个任务一个全新的 subagent + 两阶段审查

**如果用户选择 Inline Execution：**
- **REQUIRED SUB-SKILL:** 使用 `superpowers:executing-plans`
- 在当前会话里分批执行，并设置检查点供审查

Base directory for this skill: file:///Users/xiemin04/github/superpowers/skills/writing-plans
Relative paths in this skill (e.g., scripts/, reference/) are relative to this base directory.
Note: file list is sampled.

<skill_files>
<file>/Users/xiemin04/github/superpowers/skills/writing-plans/plan-document-reviewer-prompt.md</file>
</skill_files>
