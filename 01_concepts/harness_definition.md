
# Harness 的精确定义与组件清单
**Harness = Agent - Model**

Harness = 模型之外的一切代码、配置和执行逻辑。

## 完整组件清单

### 来自 LangChain
https://www.langchain.com/blog/the-anatomy-of-an-agent-harness

| 组件             | 说明                                 |
| -------------- | ---------------------------------- |
| System Prompts | AGENTS.md、CLAUDE.md                |
| Tools & MCP    | 扩展智能体能力的工具和协议（详见 [[mcp_architecture_and_protocol|MCP 架构定位与底层通信]]） |
| Skills         | 渐进式加载的知识包                          |
| 沙箱基础设施         | 文件系统、浏览器、隔离执行环境                    |
| 编排逻辑           | 子智能体生成、handoff (移交)、模型路由           |
| Hooks/中间件      | compaction (context 压缩)、续接、lint 检查 |
**Handoff的定义与区别**
在编排架构中，Handoff意味着当前subagent意味之自己已完成阶段性任务， 或者发现后续任务超出了自己的专长范围，然后主动将整个任务的‘交棒’ 给另一个更合适的subagent.

 
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
- [[throughput_and_backpressure|吞吐量演进、Ralph 循环与背压机制]]
- [[mcp_architecture_and_protocol|MCP 架构定位、底层通信与工程落地深度解析]]
- [[anthropic_dynamic_workflows|Anthropic Dynamic Workflows 深度解析]]


