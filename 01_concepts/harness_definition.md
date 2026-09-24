# Harness 的精确定义与组件清单

**Harness = Agent - Model**

Harness = 模型之外的一切代码、配置和执行逻辑。

## 完整组件清单

### 来自 LangChain

[https://www.langchain.com/blog/the-anatomy-of-an-agent-harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)


| 组件                                                                                            | 说明                                               |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| System Prompts                                                                                | AGENTS.md、CLAUDE.md                              |
| Tools & MCP                                                                                   | 扩展智能体能力的工具和协议（详见 [[mcp_architecture_and_protocol|MCP 架构定位与底层通信]]） |
| Skills                                                                                        | 渐进式加载的知识包                                        |
| 沙箱基础设施                                                                                        | 文件系统、浏览器、隔离执行环境                                  |
| 编排逻辑                                                                                          | 子智能体生成、handoff (移交)、模型路由                         |
| Hooks/中间件                                                                                     | compaction (context 压缩)、续接、lint 检查               |
| **Handoff的定义与区别**                                                                             |                                                  |
| 在编排架构中，Handoff意味着当前subagent意味之自己已完成阶段性任务， 或者发现后续任务超出了自己的专长范围，然后主动将整个任务的‘交棒’ 给另一个更合适的subagent. |                                                  |


LangChain, OpenAI, Anthropic 是有不同的handoff机制.
Anthropic 的Handoff 是一种状态的物理交接。
	机制特点是，它是通过**持久化工件**来维持进度。举例，初始化智能体 Initializer设置好环境后，会生成Plan.md （进度文件）、Git 提交记录和测试门控（Test gates）作为‘跨会话状态’。当Subagent A 交接给 Subagent B，实际交接的是这些物理文件，让Subagent B 读取这些物理文件（物理状态）来获知任务的进度。

OpenAI 的 handoff 是**去中心化**交接。Subagent A 直接交给Subagent B，有内部通信协议Agents SDK进行沟通（后续需要继续了解其机制）

LangChain 的 handoff 是状态图（State Graph）中的节点流转，交接的载体是 类型化字典（Typed State）、嵌套状态图 （后续需要继续了解其机制）

### 来自 HumanLayer（六个配置杠杆）


| #   | 杠杆            | 要点                     |
| --- | ------------- | ---------------------- |
| 1   | AGENTS.md     | ≤60 行，禁止自动生成           |
| 2   | MCP Servers   | 信任边界 + 工具数量控制          |
| 3   | Skills        | 渐进式披露，按需加载             |
| 4   | Sub-Agents    | 上下文防火墙，隔离防 context rot |
| 5   | Hooks         | 生命周期脚本，成功静默/失败报错       |
| 6   | Back-Pressure | 测试/构建/类型检查 = 自我验证回路    |

#### 深度拆解：背压机制与吞吐量理念 (Throughput & Back-Pressure)

> **核心工程哲学**: **优化迭代速度而非首次成功率**（Optimize for iteration speed, not first-shot success rate）。

- **吞吐量改变开发范式**：在工业级 Agent 落地中，高吞吐量与快速交付的关键在于“快速失败与敏捷纠偏”，而非单次极慢的完美主义执行。
- **低效做法 vs 高效做法**：
  - ❌ 每次微小改动都盲目跑全量庞大测试，浪费巨大的 Context 与 Token。
  - ✅ 建立轻量级反馈回路（Linter、静态类型、轻量单元测试）快速拦截问题。
  - ✅ **模型分层**：便宜模型（Sonnet / Haiku）处理局部具体子任务，昂贵模型（Opus）专注于顶层编排与最终仲裁。
- ⚠️ **背压是底线防护**：必须有严谨的测试、Lint 与结构检查作为护栏。**没有背压的“快速迭代”本质上只是“快速腐烂”（Fast Rotting）**。

#### 长程执行与循环拦截机制 (Ralph Loop)

长时间自主执行（Long-running Autonomous Execution）的核心挑战在于：如何让 Agent 在单次上下文用尽或出现错误时不直接崩溃退出？

- **物理持久化**：使用文件系统 + Git 追踪持久化中间成果与任务状态。
- **退出拦截与重启 (Ralph Loop)**：拦截 Agent 的退出意图，在干净的新上下文窗口中重新注入初始提示词与当前任务状态文件。
- **规划与步步自我验证**：将宏观目标逐级分解为微任务，配合 Critic/Evaluator 实行步步验证。

```text
Codex 本地审核 ──► 请求额外智能体审查 ──► 对反馈做出响应 ──► 循环直到所有审核通过
```

### 来自 Martin Fowler（三层框架）

```
┌─────────────────────────────────────────┐
│        Context Engineering              │  知识库 + 动态上下文
├─────────────────────────────────────────┤
│     Architectural Constraints           │  LLM 审查 + linter + 结构测试
├─────────────────────────────────────────┤
│     Garbage Collection Agents           │  定期扫描 + 修复漂移
└─────────────────────────────────────────┘
```

---



## 🔗 体系联动

- [[harness_lifecycle|Harness 生命周期与通用运行时 vs 项目特化配置]]
- [[coding_agent_architecture|Coding Agent 架构演进与方案分析]]
- [[mcp_architecture_and_protocol|MCP 架构定位、底层通信与工程落地深度解析]]
- [[anthropic_dynamic_workflows|Anthropic Dynamic Workflows 深度解析]]

