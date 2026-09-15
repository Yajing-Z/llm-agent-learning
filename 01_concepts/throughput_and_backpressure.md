# 吞吐量演进、Ralph 循环与背压机制 (Throughput, Ralph Loop & Back-Pressure)

> **核心工程哲学**: **优化迭代速度而非首次成功率**（Optimize for iteration speed, not first-shot success rate）。

---

## ⚡ 1. 吞吐量改变合并理念 (Throughput Changes Merge Philosophy)

在工业级 AI Coding Agent 落地中，高吞吐量与快速交付的关键在于“快速失败与敏捷纠偏”，而非单次极慢的完美主义执行。

### 来自 HumanLayer 的实战结论
- ❌ **低效做法**：每次微小改动都盲目跑全量测试，浪费巨大的 Context 与 Token。
- ✅ **高效做法**：优化迭代速度，通过轻量级机制快速发现和修复问题。
- ✅ **模型分层**：便宜模型（Sonnet / Haiku）处理局部具体子任务，昂贵模型（Opus）专注于顶层编排与仲裁。

> ⚠️ **关键前提**：**必须建立足够的背压机制（Back-Pressure）**！  
> 必须有严谨的单元测试、Lint 检查、类型系统与结构检查来充当底线防护。**没有背压的“快速迭代”本质上只是“快速腐烂”**。

---

## 🔁 2. Ralph Loop 自主循环机制

长时间自主执行（Long-running Autonomous Execution）的核心挑战是：如何让 Agent 在单次上下文用尽或出现错误时不直接崩溃退出？

### LangChain / Ralph 实现的核心三要素
1. **物理持久化**：使用文件系统 + Git 追踪持久化中间成果与状态。
2. **退出拦截与重启 (Ralph Loop)**：拦截 Agent 的退出意图，在干净的新上下文窗口中重新注入初始提示词与当前任务状态文件。
3. **规划与自我验证**：将宏观目标逐级分解为微任务，配合 Critic/Evaluator 实行步步验证。

```text
Codex 本地审核 ──► 请求额外智能体审查 ──► 对反馈做出响应 ──► 循环直到所有审核通过
```

---

## 🔗 关联阅读
- [[ralph_orchestrator|Ralph Orchestrator 项目深度研读]]
- [[harness_definition|Harness 核心定义与组件清单（含 HumanLayer 6 个杠杆）]]
- [[bun_rust_porting_case|Bun 53万行代码 Rust 重写实战复盘]]
