# Superpowers 理解笔记

这份文档整理了对 `superpowers` 项目的几组核心理解，重点是：它是什么、怎么工作、和传统研发流程以及 harness 的关系是什么。

## 1. Superpowers 是什么

`Superpowers` 不是单个 skill，也不是某个 coding 工具本体的一部分。

更准确地说，它是：

- 一套独立的 skills 系统
- 少量用于接入不同工具的插件/脚本
- 一组用于约束模型工作方式的流程规则

它通常被接入到 Claude Code、Codex、OpenCode 这类 AI coding 工具里使用。

## 2. Superpowers 的核心目标

它的核心目标不是单纯“让模型能写代码”，而是：

**让模型更稳定、更准确地完成真实工作任务。**

它重点解决的问题包括：

- 不理解上下文就直接动手
- 没有澄清需求就开始实现
- 跳过设计、计划、验证、review
- 在不确定时假装确定
- 用“先看一眼再说”之类理由绕过流程

所以它本质上是在做一件事：

**给模型建立工作制度，而不只是给模型下一个一次性 prompt。**

## 3. using-superpowers 是什么

`using-superpowers` 不是具体干活的 skill，而是整个系统的元规则入口。

它负责规定：

- 每次任务开始前，都先检查有没有适用的 skill
- 哪怕只有 1% 可能适用，也必须调用 skill
- 先用流程类 skill，再用实现类 skill
- 用户指令永远高于 skill 指令
- 不能用“这个太简单”“我先看一眼”之类理由跳过 skill

所以它更像：

- 总调度规则
- 元工作流设计
- Superpowers 的总入口约束

## 4. 它是不是把判断交给模型

是，但不是“完全放权”。

更准确地说：

**Superpowers 先用强约束把模型框住，再把受约束的判断交给模型。**

这意味着：

- 它不是传统硬编码状态机
- 不是把每一步都写死成固定流转
- 也不是完全让模型自由发挥

它采用的方式是：

1. 先把 `using-superpowers` 这种强规则注入上下文
2. 再由模型根据当前任务判断适用哪个 skill
3. 再由 skill 继续推动下一步

所以它的设计哲学更接近：

**少写死流程，多让模型按规则自主调度。**

## 5. Superpowers 为什么会自动走流程

在 OpenCode 里，插件会在会话开始时做两件事：

1. 注册 `skills/` 目录，让模型能发现所有 Superpowers skills
2. 把 `using-superpowers` 的内容注入到会话开头

这样模型一开场就先拿到一套规则：

- 每个任务先检查 skill
- 根据任务类型选择 skill
- skill 之间按 next step 和触发条件继续衔接

所以你使用 `sp` 时看到的“自动走流程”，本质上不是一个写死的流程引擎，而是：

**启动时先注入总规则，然后让模型按规则动态调用 skill。**

## 6. Superpowers 的基础流程

根据 `README.md` 的 `The Basic Workflow`，主流程大致分成 7 步：

1. `brainstorming`
2. `using-git-worktrees`
3. `writing-plans`
4. `subagent-driven-development` 或 `executing-plans`
5. `test-driven-development`
6. `requesting-code-review`
7. `finishing-a-development-branch`

可以粗略理解为：

`头脑风暴 -> 隔离工作区 -> 写计划 -> 实现 -> 测试驱动 -> 代码审查 -> 收尾`

不过这不是每次都必须机械走完的固定流水线。真正执行时，会根据任务类型动态选路。

例如：

- 新功能更可能从 `brainstorming` 开始
- Debug 更可能先走 `investigate`
- 收到 code review 更可能先走 `receiving-code-review`

## 7. brainstorming 的产出物更像什么

`brainstorming` 的产出物，不是传统意义上的纯 PRD，也还不是完整详细技术方案。

它更像：

**介于 PRD 和详细技术方案之间的设计 spec / 方案设计稿 / 概要设计。**

因为它既包含：

- purpose
- constraints
- success criteria

也包含：

- architecture
- components
- data flow
- error handling
- testing
- approaches with trade-offs

所以它明显比 PRD 更技术一些，但又还没细到 API/DB/迁移/回滚/上线方案那种详细设计层级。

## 8. PRD、brainstorming 产物、技术方案的区别

最简单的区分方法是：

- `PRD`：讲 **为什么做、做什么**
- `brainstorming spec`：讲 **做什么 + 方案怎么设计**
- `技术方案`：讲 **具体怎么实现和落地**

可以简化成这样：

### PRD

更偏业务和需求：

- 背景
- 目标
- 用户是谁
- 功能需求
- 业务规则
- 范围边界
- 成功标准

### Brainstorming 产物

更偏实现前设计：

- 需求澄清结果
- 设计方向
- 方案对比和 trade-off
- 架构/组件/数据流/测试等高层设计

### 技术方案

更偏工程落地：

- 接口设计
- 数据模型
- 详细模块拆分
- 兼容和迁移策略
- 发布/灰度/回滚
- 风险控制

## 9. YAGNI 是什么

`YAGNI` 全称是：

`You Aren't Gonna Need It`

意思是：

- 不要为“以后可能会用”提前做设计
- 只实现当前明确需要的东西
- 不要往方案里塞未来幻想出来的扩展点和复杂度

在 `brainstorming` 中，它强调的是：

**在设计阶段就把不必要的功能和过度设计去掉。**

## 10. 视觉辅助是什么

`visual companion` 是 `brainstorming` 的一个辅助能力。

它的用途是：

- 在浏览器中展示 mockup、布局、图示、方案对比
- 让用户通过“看图”和“点选”来参与头脑风暴

它不是默认模式，而是：

**当某个问题用视觉展示比纯文字更容易理解时，再启用。**

相关的 `scripts/` 目录，就是专门为这个视觉辅助服务的技术实现层。

## 11. Harness 和 Superpowers 的关系

`Harness` 的范围更大，`Superpowers` 的范围更窄。

可以这样理解：

### Harness 更偏基础设施层

负责：

- 工具接入
- 文件/命令/浏览器能力
- 上下文注入
- 会话管理
- 权限边界
- 模型和环境的桥接

### Superpowers 更偏流程编排层

负责：

- 什么时候该用哪个 skill
- 先 brainstorm 还是先 debug
- 什么时候必须写计划
- 什么时候必须 review
- 什么时候必须停下来让用户确认
- 怎么防止模型跳步骤和自我合理化

所以可以总结成：

**Harness 给能力，Superpowers 给流程。**

或者：

**Harness 决定模型“能做什么”，Superpowers 决定模型“应该怎么做”。**

## 12. 它们的最终目标是不是一样

是，最终目标是一致的。

两者都服务于同一个目标：

**让 AI 更精准地完成工作任务。**

区别在于侧重点不同：

- `Harness` 解决的是：AI **能不能做**
- `Superpowers` 解决的是：AI **会不会按正确方式做**

也可以说：

- `Harness` 提供可执行性
- `Superpowers` 提供可控性

## 13. AI 时代工程重心的变化

传统软件开发更多是在思考：

- 程序怎么实现
- 逻辑怎么执行
- 状态怎么流转

而在 AI coding 时代，一个越来越重要的问题变成了：

- 怎么给模型建立正确约束
- 怎么防止它跑偏
- 怎么让它在复杂任务中稳定工作

所以现在很多工程工作，本质上是在做：

**如何约束模型决策，而不只是如何实现代码逻辑。**

换句话说：

**以前软件工程是在约束机器执行代码；现在很大一部分工作，是在约束模型生成正确决策。**

## 14. 关于“一次性完成率不高”的理解

一个重要观察是：

很多时候，模型一次性完成得不好，不完全是模型本身能力不足，而是因为：

- 指令 framing 不清楚
- 流程约束不完整
- 上下文组织不好
- 缺少中间确认点
- 缺少验证闭环
- 没有限制典型跑偏方式

所以更准确的说法是：

**模型能力决定上限，约束设计决定它能不能稳定接近这个上限。**

## 15. 一句话总结

`Superpowers` 可以理解为：

**一套接入到 AI coding 工具中的 skill/workflow 系统。它不靠硬编码状态机来强行驱动流程，而是通过强约束元规则和 skill 链式衔接，让模型在上下文中动态决定下一步，并更稳定地完成真实工作任务。**
