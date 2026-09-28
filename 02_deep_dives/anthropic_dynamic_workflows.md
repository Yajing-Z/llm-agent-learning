# Anthropic Harness 设计与 Dynamic Workflows 深度解析

> **核心参考资料**:
> - [Anthropic Engineering: Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)
> - [Claude Blog: A Harness for Every Task — Dynamic Workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
> - [CSDN 解读文章](https://agent.csdn.net/6a388c9f662f9a54cb82826b.html)

---

## 🧩 1. Harness 是什么？

$$ \text{Harness} = \text{Agent} - \text{Model} $$

在 AI Agent 领域，Harness 是包裹在基础大模型（Foundation Model）外的那一层 **[该干啥、怎么干、谁先谁后]** 的调度与控制逻辑。

> **当一个 Agent 单 Loop 干不完一件事的时候，就让 Agent 自己写一个一次性、为该任务量身定做的 Harness。**  
> 例如：Claude Code 默认的 plan-execute loop 是 Harness；Cursor 的 Apply 流程也是 Harness。

---

## 🎯 2. 为什么构建 Harness？

构建 Harness 的核心目标，是解决大模型在复杂/长时间长程任务（Long-running tasks）中的三大结构性痛点：

1. **Agentic Laziness（智能体偷懒）**：面对包含几十个步骤的复杂任务，模型在完成其中一部分后，倾向于宣布“我已经全部做完了”。
2. **Self-preferential Bias（自评偏差）**：让 Claude 自己 judge 自己的输出，它天然倾向于认为自己是对的。这是单 Context 的结构性缺陷。
3. **Goal Drift（目标漂移）**：多轮交互与 Compaction（上下文压缩）后，长上下文遗忘导致偏离原始需求，尤其是那些“不要做 XX”的负向约束最容易丢。

### Anthropic 的 Harness 架构基石
**三智能体架构**：`Planner (规划)` + `Generator (生成)` + `Evaluator (评估)`

---

## ⚡ 3. 什么是 Dynamic Workflow？

Dynamic Workflow 是让 Claude 自己写 Harness、自己调度几十上百个 Subagent 的那套机制。

![Claude Harness](../assets/claude_harness.png)

> **Anthropic 官方定义**：  
> *"A dynamic workflow is a JavaScript script that orchestrates subagents at scale. Claude writes the script for the task you describe, and a runtime executes it in the background while your session stays responsive."*

**机制核心**：
- Claude 分析任务后，当场生成一段 **JavaScript 编排脚本**。
- 脚本中调用 `agent()` 函数，决定开几个 Subagent、怎么开、分别用什么模型（Opus/Sonnet/Haiku）、以及 Subagent 之间如何交叉验证与自我纠错。
- 这个脚本是 **用过即弃（ephemeral）** 的。

![Dynamic Workflow Example](../assets/dynamic_workflow_example.png)

---

## 📐 4. Claude 写 Harness 的 6 种 Pattern

Claude 根据具体任务，将以下 6 种编排模式自由组合进生成的 JS 脚本中：

![Claude Harness Pattern](../assets/claude_harness_pattern.png)

### ① Classify-and-Act (分类与分流)
最基础的模式：“专人做专事”。接收用户任务后，分类器 Agent 判断任务类型（如 bug 修复、功能开发、代码审计等），然后将任务 Router 给最合适的专用 Subagent。

### ② Fan-Out-Synthesize (扇出与屏障汇总)
![Fan-out Synthesize](../assets/fan-out-synthesize.png)
大规模并行处理模式：
- **拆分 (Fan-out)**：将大任务拆分为多个独立子任务，为每个步骤启动一个干净上下文的 Subagent。
- **独立上下文**：互不污染，消除 Context Rot。
- **屏障综合 (Synthesize Barrier)**：等待所有并发 Subagent 完成，再合并为一个统一的结构化成果（如 500 个文件批量重构/迁移）。

### ③ Adversarial-Verify (对抗性验证)
![Adversarial Verify](../assets/adversarial-verify.png)
解决单一 Agent 的自评偏差（Self-preferential Bias）：
- 每一个 Subagent 的产出，都由一个**全新的 Critic Agent** 去严格挑刺攻击。
- Critic 看不到 Generator 的思考链过程，只看输出结果，依照预设的评分准则（Rubric）检查，过关后才允许流转到下一阶段。

---

## 🤖 5. 深入机制：模型选择与 Token 预算

### ① 不同 Subagent 如何分配模型？
在 Dynamic Workflow 脚本中直接指定：
- **智能分级**：分类器可以根据任务难度动态路由——简单子任务派发给 Sonnet 或 Haiku，高难度复杂推理派发给 Opus。
- 这种逻辑直接内嵌在生成的 JavaScript 调度逻辑中。

### ② Token 预算控制 (Token Budget)
用户或系统可在 Prompt 中显式声明预算约束，例如：
```text
use 10k tokens... <具体任务描述>
```

---

## 🖼️ 架构全景速查
![Claude Harness Overview](../assets/Screenshot%202026-07-16%20at%2007.05.03.png)
