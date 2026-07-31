# LLM Agent & Harness Engineering 知识与实战指南

> **核心工程理念**: `Harness = Agent - Model`  
> 大模型（Model）是无状态的大脑，而 **Harness** 是包裹在模型之外的执行壳、安全护栏、上下文管理与验证循环系统。本仓库用于沉淀 AI Agent 全栈工程落地过程中的概念辨析、文章精读与开源项目研读。

---

## 🗺️ 知识库地图 (Map of Content / MOC)

### 📌 00. 路线图与学习规划 (`00_roadmap/`)
- [企业级 AI Agent 全栈工程落地路线图](./00_roadmap/learning_path.md) — 长期演进的技能路线与实践任务规划。

### 🧠 01. 核心概念与工程范式 (`01_concepts/`)
- [Harness 的精确定义与组件清单](./01_concepts/harness_definition.md) — 来自 LangChain、HumanLayer 与 Martin Fowler 的 Harness 架构视角。
- [Harness 生命周期与 Model/Project 关系](./01_concepts/harness_lifecycle.md) — 探讨通用运行时与项目特化配置的解耦。
- [Coding Agent 架构组件分析](existing_harness_solution.md) — 剖析 Coding Agent 的核心组成部分。
- [吞吐量改变合并理念 (Throughput Changes Merge Philosophy)](./01_concepts/throughput_changes_merge.md) — 工业级 Agent 状态合并与吞吐演进。

### 📚 02. 文章、论文与书籍研读 (`02_articles_notes/`)
- [Anthropic Harness 设计与 Dynamic Workflows 深度解析](anthropic_harness_dynamic-workflow.md) — 剖析 Planner/Generator/Evaluator 模式与 6 种编排 Pattern。
- [《AI Agent Book》读书与学习笔记](./02_articles_notes/ai_agent_book.md) — 开源 Agent 教程精读。
- **论文资源**：
  - `Agentic-Design-Patterns.pdf` & `Agentic-Design-Patterns-CN-v20260115.pdf` — Agentic 设计模式论文（中英文版）。

### 🛠️ 03. 开源项目研读与实践 (`03_projects_study/`)
- [OpenCode 开源 Agent 研读笔记](./03_projects_study/opencode.md) — 终端 Coding Agent 工具架构。
- [Pi Agent Harness 研读笔记](./03_projects_study/pi_agent_harness.md) — 轻量级 Agent 运行时探索。
- [Browser Harness 研读笔记](./03_projects_study/browser_harness.md) — 基于 CDP 的浏览器自动化 Agent 框架。

---

## 📂 目录结构预览

```text
llm-agent-learning/
├── README.md                 # 知识库主页与 MOC 导航
├── .gitignore                # Git 忽略配置
├── 00_roadmap/               # 路线图与学习规划
├── 01_concepts/              # 核心概念与工程范式笔记
├── 02_articles_notes/        # 经典文章、论文与书籍精读
├── 03_projects_study/        # 开源 Agent 项目研究与实践心得
└── assets/                   # 资源图片与架构图库
```

---

## 💡 使用指南
- **Obsidian 联动**：本仓库支持 Obsidian 双向链接（如 `[[harness_definition]]`），同时完美兼容 GitHub 相对路径导航。
- **持续更新**：随着学习深入，新的文章笔记可放入 `02_articles_notes/`，新的开源项目实践放入 `03_projects_study/`，保持结构清晰平铺。
