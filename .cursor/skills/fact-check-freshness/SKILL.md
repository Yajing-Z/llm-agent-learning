---
name: fact-check-freshness
description: Audit technical accuracy, temporal freshness, and protocol alignment for LLM Agent & Harness Engineering concepts. Use when checking whether notes are outdated, verifying MCP protocol specifications, inspecting Cursor runtime architecture, or comparing framework implementations (OpenAI, Anthropic, LangChain).
---

# Technical Fact-Checking & Freshness Audit (技术准确性与时效保鲜审计)

本技能用于对知识库的技术事实、协议版本、线缆级报文及架构实现进行穿透式核验，防止文档陈旧化与概念失真。

---

## 🔍 关键核验清单 (Audit Checklists)

### 1. MCP (Model Context Protocol) 规范核验

- [ ] **基准版本**: 协议基准版本是否明确标为 `2024-11-05`？
- [ ] **传输通道**: 是否明确区分了本地 `stdio` 与远程 `streamable-http`（或 SSE）？
- [ ] **三大原语完整性**: 是否完整涵盖 `Resources`（只读数据）、`Prompts`（用户触发模版）与 `Tools`（具有外部副作用的动作）？
- [ ] **解耦事实**: 是否清晰强调 **LLM 不跑 MCP 协议**，而是由 Host 宿主（如 Cursor）将 MCP inputSchema 转译为 LLM tools Schema？
- [ ] **Wire-Level 报文**: `initialize`、`tools/list`、`tools/call` 的 JSON-RPC 2.0 报文字段是否符合官方 Spec？

---

### 2. Cursor 宿主运行时与工具治理事实核验

- [ ] **元工具网关**: 代理外部 MCP 的宿主内置工具是否准确写为 `GetDynamicTools` 与 `CallDynamicTool`？
- [ ] **提示词目录标签**: XML 索引标签是否准确为 `<dynamic_tool_catalog>`，行为协议标签是否准确为 `<dynamic_tools>`？
- [ ] **内置工具清单**: 主会话工作集（~15个）与底层全量储备（48个）的关系是否阐述清晰？是否严格遵循最小特权原则？
- [ ] **防爆舱工件**: 大报文转储路径是否准确对应 `~/.cursor/projects/.../agent-tools/*.txt` 机制？
- [ ] **安全存储**: 是否真实还原 Electron `safeStorage` 与 macOS Keychain 硬件保护的 OAuth Token 注入流程？

---

### 3. 主流前沿厂商方案时效性核验

- [ ] **Anthropic**:
  - Dynamic Workflows 编排机制（临时 JS 脚本生成、`agent()` 调度、6 大 Harness Pattern）。
  - 长程任务的物理工件流转（Initializer、`Plan.md`、Ralph Loop 退出拦截）。
- [ ] **OpenAI**:
  - 去中心化 Handoff 机制依托专用的 OpenAI Agents SDK。
  - 函数调用中的字符串反序列化与多工具并发（Parallel Tool Calling）。
- [ ] **LangChain / LangGraph**:
  - 基于有向状态图（StateGraph）的节点流转与类型化字典（Typed State）。

---

## 🛠️ 处置与修复流程

当审计发现某部分内容过时或存在事实偏差时：

1. **修正或版本标注**:
   - 若属于描述笔误或失真，依据官方最新规范直接修正；
   - 若属于技术架构升级（如 Cursor V0 -> V1 -> V2），保留演进过程并在旧方案处添加说明：
     ```markdown
     > ⚠️ **历史版本演进**: 该方案为早期实现，现行工业界标准已迭代为...
     ```
2. **凭据核对**:
   - 优先通过本地可执行命令（如查看本地安装配置、抓包报文、官方 Spec）作为佐证，确保修改具有确定性。
