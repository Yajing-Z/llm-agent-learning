# Browser Harness 研读笔记

> **项目仓库**: [browser-use/browser-harness](https://github.com/browser-use/browser-harness)  
> **核心定位**: 基于 CDP（Chrome DevTools Protocol）的超轻量浏览器自动化 Agent 运行时与自愈型 Harness。

---

## 🎯 核心架构亮点

### 1. 直连真实浏览器（零中间层抽象）
传统的浏览器自动化工具（如早期 Selenium 或重型包装库）往往封装了较厚的抽象层。Browser Harness 通过极轻量的 **CDP（Chrome DevTools Protocol）** 管道，将 LLM / Coding Agent 直接连接至真实本地 Chrome 实例，极大提升了控制自由度、调试透明度与执行效率。

### 2. 自愈与自我演进能力（Self-healing & Skill Learning）
- 在任务执行过程中，如果缺少某个特定操作函数或 UI 选择器，Agent 会直接就地编写或修正 Helper 代码（如 `agent_helpers.py`）以及特定站点的技能包（`domain-skills`）。
- **经验复用机制**：下一次运行同类任务时直接复用沉淀下来的技能包，实现运行时“越用越聪明”的自愈演进。

### 3. 无缝对接终端 Coding Agent
专为 Claude Code、Codex CLI 等终端 Coding Agent 量身设计：
- 只需粘贴一段预设的 Setup Prompt 即可在容器或宿主机中快速完成部署与通信连接。
- 与终端 Agent 的文件与执行上下文无缝融合。

---

## 🔗 体系联动
- [[harness_definition|Harness 核心定义与沙箱/浏览器执行环境]]
- [[opencode|OpenCode 终端 Agent 架构研读]]
