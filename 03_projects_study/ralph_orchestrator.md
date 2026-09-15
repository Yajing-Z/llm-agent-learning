# Ralph Orchestrator 研读笔记

> **项目仓库**: [mikeyobrien/ralph-orchestrator](https://github.com/mikeyobrien/ralph-orchestrator)  
> **核心定位**: 基于 "Hat" (帽子) 系统的循环编排框架，通过事件驱动使 AI Agent 保持在执行与校验循环中直至任务完成。  
> **关联概念**: 参见 [[throughput_and_backpressure|Ralph Loop 机制与背压]]

---

## 🔁 执行流程示例

```text
加载 ralph.yml 配置 
     │
     ▼
读取下一个子任务 (planning.md)
     │
     ▼
触发 Hooks (-H builtin:code-assist)  ──► [注入代码上下文/最佳实践]
     │
     ▼
调用 AI Backend 执行写代码/改代码
     │
     ▼
触发 Hooks (-H builtin:code-assist)  ──► [运行 Lint / 测试 / 代码格式化]
     │
     ├── 失败：将错误带回循环，让 AI 重试
     └── 成功：自动 Commit 并标记任务完成，推进到下一项
```

---

## 🎭 Subagent 角色拆解

Ralph Orchestrator 在执行任务时会拆解问题，从而创造出不同职责的 Subagent：

```text
Planner  ──►  Builder  ──►  Critic  ──►  Finalizer
(规划)        (写代码)      (审查测试)   (归档提交)
```

### 这些 Subagent 是怎么创建的？
**答：通过代码中的“Hat (帽子) 系统”和“事件驱动架构”创建。**

在 Ralph 的源码架构（主要是 [ralph-core](https://github.com/mikeyobrien/ralph-orchestrator/tree/main/crates/ralph-core) 核心模块）里，这些 Subagent 的本质是 **“带着不同角色 Prompt 和工具指令的智能体形态”**。

代码层面
 **Ralph 内部配置或源码预设（概念示例）**
hats:
  planner:
    name: "Planner"
    instructions: "你现在的身份是架构规划师。只允许读取需求并拆解为任务列表，严禁改动业务代码..."
    triggers: ["loop.start"]
    publishes: ["plan.completed"]

  builder:
    name: "Code Builder"
    instructions: "你现在的身份是程序员。请读取 .ralph/tasks/ 下的任务，并只改动实现该任务所需的代码..."
    triggers: ["plan.completed", "review.failed"]
    publishes: ["code.updated"]

  reviewer: # 即 Critic 角色
    name: "Code Reviewer"
    instructions: "你现在的身份是严苛的测试与 Code Review 专员。请运行测试命令并进行代码安全性审查..."
    triggers: ["code.updated"]
    publishes: ["review.passed", "review.failed"]

**运行时层面**：如何“激活”并创建 Subagent？
- **事件触发（Events）**：上一个阶段完成后会向系统发出一个 Event（例如 `code.updated`）。
    
- **状态匹配与组装**：Ralph 监听到该事件，查表得知下一个该轮到 `Critic` “戴上帽子”上线了。
    
- **注入上下文与 Prompt**：Ralph 从本地磁盘读取当前的最新代码、记忆文件（`memory.md`）以及 `Critic` 的系统提示词。
    
- **唤起底层 Agent Backend**：把组装好的全新 Prompt 喂给底层引擎（如 `claude` CLI），此时这个被唤起的 LLM 进程，就**变成了拥有 Critic 行为逻辑的 Subagent**。

