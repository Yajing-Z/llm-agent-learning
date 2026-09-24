# Harness 生命周期与 Model/Project 关系辨析

> **核心命题**: 跨模型可移植的通用 Harness 是一个幻觉吗？Harness 究竟与 Model 绑定还是与 Project 绑定？

---

## 🔄 1. 模型与 Harness 的“循环依赖”困境

- **LangChain 观察**：模型在 Post-training（后训练）阶段往往会过拟合（Overfit）到特定的 Harness。
- **Anthropic 观察**：Harness 本身不可避免地编码了对特定模型能力与特性的强假设。

这构成了一个闭环的**循环依赖**：

```text
模型训练时适配 Harness ──► Harness 为模型量身设计 ──► 模型能力升级 ──► 原 Harness 过时
       ▲                                                                   │
       └─────────────────────── 重新设计 Harness ◄─────────────────────────┘
```

### 核心拷问
1. 是否每个模型供应商最终都会捆绑专属的 Harness？（如 Anthropic 的 Claude Code，OpenAI 的 Codex CLI 等）。
2. 跨模型完全通用的 Harness 是否只是一个工程理想？

---

## 🏛️ 2. Model、Harness、Project 三者关系

| 实体 | 角色定位 | 核心特征 |
| :--- | :--- | :--- |
| **Model** | “大脑” | 无状态、易出错、无外部 IO 设备感知 |
| **Harness** | “执行壳 / 操作系统” | 提供执行调度、记忆持久化、安全护栏与背压验证循环 |
| **Project** | “物理边界与事实来源” | 代码库、业务领域规则、既有架构与测试套件 |

这三者始终处于动态演进中：无论是 Model 升级、还是 Project 发生技术栈与规则演进，Harness 必须随之调整。

---

## 🧱 3. 解耦之道：通用运行时 vs. 项目特化配置

解决循环依赖的最佳工程实践是将 Harness 划分为两层解耦架构：

### 维度 A：通用运行时层 (Runtime Layer)
- **代表实现**：Claude Code、Codex CLI、OpenCode 等底层执行框架。
- **职责**：提供通用的子智能体编排（Orchestration）、MCP 协议通信、沙箱（Sandbox）隔离、Skill 动态加载、Handoff 状态机等。
- **详见组件清单**: [[harness_definition|Harness 的精确定义与组件清单]]

### 维度 B：项目特化配置层 (Project-Specific Config Layer)
- **与 Project 强绑定**：
  - 项目专属规则：`AGENTS.md` / `CLAUDE.md`
  - 领域知识与自愈脚本：Custom Skills
  - 背压护栏：Linter 规范、单元/集成测试套件、构建指令
  - 跨会话进度跟踪：`Plan.md`、`tasks/`、Git Commit 门控

---

## 🔗 关联阅读
- [[harness_definition|Harness 的精确定义与组件清单（含背压机制）]]
- [[coding_agent_architecture|Coding Agent 核心架构与主流厂商方案对比]]
