# Superpowers 技能流程与关系图

## 1. 核心工作流程（主流程）

```mermaid
flowchart TD
    Start([用户请求]) --> CheckSkill{检查技能}
    CheckSkill -->|1% 可能性| UsingSP[using-superpowers<br/>核心引导技能]

    UsingSP --> DetectType{任务类型?}

    DetectType -->|创意/新功能| Brainstorming[brainstorming<br/>头脑风暴设计]
    DetectType -->|Bug修复| Debugging[systematic-debugging<br/>系统化调试]
    DetectType -->|代码审查| CodeReview[requesting-code-review<br/>请求代码审查]
    DetectType -->|其他| Direct[直接执行]

    Brainstorming -->|输出 design.md| WritingPlans[writing-plans<br/>编写实施计划]
    WritingPlans -->|输出 plan.md| CheckScope{计划复杂度?}

    CheckScope -->|复杂/多任务<br/>每个任务独立<br/>需严格审查| SDD[subagent-driven-development<br/>子代理驱动开发<br/><br/>特点: 派遣子代理<br/>两阶段审查]
    CheckScope -->|简单/当前会话<br/>人机协作<br/>快速迭代| Executing[executing-plans<br/>执行计划<br/><br/>特点: 主代理执行<br/>无内置审查]

    SDD --> Worktree[using-git-worktrees<br/>Git工作树隔离]
    Executing --> Worktree

    Worktree --> Implementation{实施过程}

    Implementation -->|写代码| TDD[test-driven-development<br/>测试驱动开发]
    Implementation -->|遇到问题| Debugging
    Implementation -->|并行任务| Parallel[dispatching-parallel-agents<br/>并行代理分发]

    TDD --> Verify[verification-before-completion<br/>完成前验证]
    Debugging --> Verify
    Parallel --> Verify

    Verify -->|通过| Finish[finishing-a-development-branch<br/>完成开发分支]
    Verify -->|不通过| Implementation

    Finish --> Review{需要审查?}
    Review -->|是| CodeReview
    Review -->|否| End([结束])

    CodeReview -->|使用| AgentCodeReview[code-reviewer<br/>代码审查员Agent]
    AgentCodeReview --> End

    style Start fill:#e1f5fe
    style End fill:#c8e6c9
    style UsingSP fill:#fff3e0
    style Brainstorming fill:#fff3e0
    style SDD fill:#fff3e0
```

## 2. SDD vs Executing-Plans 核心区别

```mermaid
flowchart LR
    subgraph Comparison [两种执行方式对比]
        direction TB

        subgraph SDD [subagent-driven-development<br/>子代理驱动开发]
            SDD_Mode[执行模式: 当前会话] --> SDD_Agent[每个任务派遣新鲜子代理]
            SDD_Agent --> SDD_Review[两阶段审查循环]
            SDD_Review --> SDD_Parallel[任务可并行执行]

            SDD_Review_Detail[Spec Reviewer<br/>验证符合需求<br/><br/>Code Quality Reviewer<br/>验证代码质量]
            SDD_Review -.-> SDD_Review_Detail
        end

        subgraph EP [executing-plans<br/>执行计划]
            EP_Mode[执行模式: 当前会话] --> EP_Main[主代理自己执行]
            EP_Main --> EP_NoReview[无内置审查循环]
            EP_NoReview --> EP_Sequential[任务顺序执行]
        end
    end

    subgraph Decision [选择依据]
        Q1{任务复杂度?}
        Q1 -->|高复杂度<br/>多独立任务<br/>需严格质量控制| SDD
        Q1 -->|低复杂度<br/>快速迭代<br/>人机协作| EP
    end

    style SDD fill:#e3f2fd
    style EP fill:#fff3e0
    style SDD_Agent fill:#bbdefb
    style EP_Main fill:#ffe0b2
```

## 3. 子代理驱动开发（SDD）内部详细流程

```mermaid
flowchart TD
    subgraph SDD_Phase [subagent-driven-development 阶段]
        ReadPlan[读取计划<br/>提取所有任务] --> TodoWrite[创建 TodoWrite<br/>任务追踪]
        TodoWrite --> Loop{还有任务?}

        Loop -->|是| DispatchImpl[派遣 Implementer<br/>实现者子代理<br/><br/>特点: 上下文隔离<br/>专注单一任务]
        DispatchImpl --> ImplWork[实现任务<br/>写代码+测试+提交]
        ImplWork --> SelfReview[自审检查]

        SelfReview --> DispatchSpec[派遣 Spec Reviewer<br/>合规审查员<br/><br/>验证: 不多不少<br/> exactly符合需求]
        DispatchSpec --> SpecCheck{符合Spec?}
        SpecCheck -->|否| FixSpec[Implementer修复<br/>Spec差距] --> DispatchSpec
        SpecCheck -->|是| DispatchQuality[派遣 Code Quality Reviewer<br/>质量审查员<br/><br/>验证: 代码质量<br/>设计合理性]

        DispatchQuality --> QualityCheck{质量通过?}
        QualityCheck -->|否| FixQuality[Implementer修复<br/>质量问题] --> DispatchQuality
        QualityCheck -->|是| MarkComplete[标记任务完成] --> Loop

        Loop -->|否| FinalReview[最终代码审查]
    end

    style DispatchImpl fill:#e3f2fd
    style DispatchSpec fill:#fce4ec
    style DispatchQuality fill:#fce4ec
    style FixSpec fill:#fff3e0
    style FixQuality fill:#fff3e0
```

## 3. 技能依赖关系

```mermaid
flowchart TB
    subgraph Entry [入口]
        USP[using-superpowers<br/>统一入口]
    end

    subgraph Design [设计阶段]
        BS[brainstorming<br/>头脑风暴]
        WP[writing-plans<br/>编写计划]
    end

    subgraph Exec [执行阶段]
        direction TB
        SDD[subagent-driven-development<br/>子代理驱动]
        EP[executing-plans<br/>直接执行]
    end

    subgraph Practice [开发实践]
        WGW[using-git-worktrees<br/>工作树隔离]
        TDD[test-driven-development<br/>测试驱动开发]
        DPA[dispatching-parallel-agents<br/>并行分发]
    end

    subgraph Quality [质量门禁]
        VBC[verification-before-completion<br/>完成前验证]
        FDB[finishing-a-development-branch<br/>分支收尾]
    end

    subgraph Review [审查机制]
        RCR[requesting-code-review<br/>请求审查]
        CRA[code-reviewer Agent<br/>审查代理]
    end

    %% 主流程
    USP --> BS --> WP
    WP -->|复杂任务| SDD
    WP -->|简单任务| EP

    %% 执行层到开发实践
    SDD & EP --> WGW
    SDD & EP --> TDD
    SDD -.->|可选| DPA

    %% 开发实践到质量门禁
    WGW & TDD --> VBC --> FDB

    %% 审查机制
    FDB -.->|可选| RCR --> CRA

    %% 调试贯穿全程（侧边）
    SD[systematic-debugging<br/>系统调试]
    SD -.-> BS & WP & SDD & EP
```

## 4. 用户请求路由决策树

```mermaid
flowchart TD
    Request[用户请求] --> Q1{涉及代码编写<br/>或修改?}

    Q1 -->|是| Q2{是修复现有问题?}
    Q1 -->|否| Productivity[其他 productivity 技能]

    Q2 -->|是| Debugging[systematic-debugging]
    Q2 -->|否| Q3{有明确设计/计划?}

    Q3 -->|有| Q4{任务是否独立<br/>可在当前会话完成?}
    Q3 -->|没有| Brainstorming[brainstorming<br/>头脑风暴]

    Q4 -->|是<br/>任务独立可并行<br/>需严格质量控制| SDD[subagent-driven-development<br/><br/>派遣子代理<br/>Spec审查→质量审查]
    Q4 -->|否<br/>任务简单/紧密耦合<br/>人机协作| Executing[executing-plans<br/><br/>主代理执行<br/>无内置审查]

    Brainstorming -->|输出设计| WritingPlans[writing-plans]
    WritingPlans -->|输出计划| Q4

    SDD --> Worktree[using-git-worktrees]
    Executing --> Worktree

    Worktree --> Q5{实施中遇到问题?}
    Q5 -->|Bug/错误| Debugging
    Q5 -->|正常开发| TDD[test-driven-development]

    TDD --> Q6{验证通过?}
    Debugging --> Q6
    Q6 -->|否| Q5
    Q6 -->|是| Verify[verification-before-completion]

    Verify --> Finish[finishing-a-development-branch]
    Finish --> Q7{需要代码审查?}
    Q7 -->|是| Review[requesting-code-review]
    Q7 -->|否| Done[完成]
    Review --> Done

    style Request fill:#e1f5fe
    style Done fill:#c8e6c9
    style Brainstorming fill:#fff3e0
    style Debugging fill:#ffccbc
    style Review fill:#e1bee7
```

## 5. 技能分类矩阵

```mermaid
graph LR
    subgraph Category [技能分类]
        direction TB

        subgraph Process [流程控制]
            USP[using-superpowers]
            BS[brainstorming]
            WP[writing-plans]
        end

        subgraph Execution [执行实施]
            SDD[subagent-driven-development]
            EP[executing-plans]
            DPA[dispatching-parallel-agents]
        end

        subgraph DevPractice [开发实践]
            TDD[test-driven-development]
            SD[systematic-debugging]
            WGW[using-git-worktrees]
        end

        subgraph QualityGate [质量门禁]
            VBC[verification-before-completion]
            RCR[requesting-code-review]
            REQR[receiving-code-review]
            FDB[finishing-a-development-branch]
        end

        subgraph SkillDev [技能开发]
            WS[writing-skills]
        end
    end

    USP --> BS --> WP --> SDD --> TDD --> VBC --> FDB
    WP --> EP
    SDD --> DPA
    SD -.-> SDD
    SD -.-> EP
    RCR -.-> FDB
    REQR -.-> FDB
```

## 6. 关键原则说明

```mermaid
flowchart TD
    subgraph Principles [Superpowers 核心原则]
        P1[强制使用技能<br/>1%规则] --> P2[先设计后编码<br/>Hard Gate]
        P2 --> P3[测试验证<br/>verification-before-completion]
        P3 --> P4[质量门禁<br/>多阶段审查]

        P5[上下文隔离<br/>子代理模式] --> P6[工作树隔离<br/>using-git-worktrees]
        P6 --> P7[并行执行<br/>dispatching-parallel-agents]

        P8[用户决策优先<br/>AskUserQuestion] --> P9[增量确认<br/>避免误解]
    end

    style P1 fill:#ffcc80
    style P2 fill:#ffcc80
    style P3 fill:#c8e6c9
    style P5 fill:#b3e5fc
    style P8 fill:#e1bee7
```

---

## 7. SDD vs Executing-Plans 关键区别对照表

| 维度 | subagent-driven-development | executing-plans |
|------|----------------------------|-----------------|
| **执行主体** | 派遣新鲜子代理（每个任务一个） | 主代理自己执行 |
| **上下文隔离** | ✅ 完全隔离（子代理只看任务描述） | ❌ 继承主会话上下文 |
| **审查机制** | ✅ 两阶段强制审查（Spec→Quality） | ❌ 无内置审查 |
| **任务关系** | 任务独立，可并行 | 任务顺序，依赖主代理协调 |
| **人机协作** | 低（子代理自动执行） | 高（主代理可询问用户） |
| **适用场景** | 复杂实现、严格质量要求 | 简单任务、快速迭代、探索性 |
| **成本** | 较高（多子代理+多轮审查） | 较低（单会话） |
| **速度** | 较慢（审查循环） | 较快（无审查门槛） |

### 选择决策

```
任务是否独立？
    ├─ 是 → 是否需要严格质量控制？
    │         ├─ 是 → subagent-driven-development
    │         └─ 否 → executing-plans
    └─ 否 → executing-plans（顺序执行更合适）
```

---

## 图例说明

| 颜色 | 含义 |
|------|------|
| 🟧 橙色 | 核心/入口技能 |
| 🟦 蓝色 | 主要流程技能 |
| 🟩 绿色 | 质量/验证技能 |
| 🟥 红色 | 调试/修复技能 |
| 🟪 紫色 | 审查/代码质量 |
| ⬜ 白色 | 支持/通用技能 |
