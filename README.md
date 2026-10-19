# LLM Agent & Harness Engineering 知识库

围绕 Agent 执行外壳、工具调用、上下文治理与动态工作流的个人学习笔记。

$$\text{Harness} = \text{Agent} - \text{Model}$$

## 核心概念

- [Harness 定义与组件](./01_concepts/harness_definition.md)：组成、背压与工程边界。
- [Harness 生命周期](./01_concepts/harness_lifecycle.md)：通用运行时与项目配置的关系。
- [Coding Agent 架构](./01_concepts/coding_agent_architecture.md)：架构演进与厂商方案讨论。
- [MCP 架构与协议](./01_concepts/mcp_architecture_and_protocol.md)：协议定位与通信流程。
- [工具爆炸与上下文治理](./01_concepts/tool_explosion_and_governance.md)：以 Cursor 实践为例讨论工具发现、裁剪与结果管理。

## 专题精读

- [Codex 全景架构](./02_deep_dives/openai_codex_architecture.md)：源码结构与执行流程。
- [Codex CLI](./02_deep_dives/openai_codex_cli.md)：入口启动与运行形态。
- [Codex 沙箱](./02_deep_dives/openai_codex_sandbox.md)：权限配置、平台隔离与网络生命周期。
- [Codex 沙箱提权与审批](./02_deep_dives/openai_codex_sandbox_escalation.md)：通俗理解权限申请、Guardian 自动审批、两次尝试与保留禁令的文件系统扩权。
- [Codex 进程隔离与权限边界](./02_deep_dives/openai_codex_sandbox_processes.md)：通过测试任务理解进程视野、系统调用过滤、身份权限与子进程清场。
- [Anthropic Dynamic Workflows](./02_deep_dives/anthropic_dynamic_workflows.md)：动态工作流与 Harness 设计。
- [《AI Agent Book》笔记](./02_deep_dives/ai_agent_book.md)：工具调用与通信报文。

## 建议阅读顺序

1. 建立框架：Harness 定义 → 生命周期 → Coding Agent 架构。
2. 理解交互：MCP 架构与协议 → 工具爆炸与上下文治理 → 《AI Agent Book》笔记。
3. 深入实现：Codex 全景架构 → CLI → 沙箱总览，再按需阅读沙箱提权与审批、进程隔离与权限边界。

## 🔗 体系联动

- [仓库约定](./AGENTS.md)：Codex 协作、认知留白与笔记格式。
- [知识库审计参考](./KNOWLEDGE_AUDIT.md)：按需使用的概念边界、证据要求与核验判据。
