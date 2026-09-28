---
name: audit-knowledge-base
description: Audit knowledge base notes for conceptual consistency, contradictory claims across documents, broken Obsidian bidirectional links, missing system linkages, and tracking learning backlogs. Use when the user asks to audit the repository, verify notes, inspect concept consistency, or check for knowledge gaps.
---

# Knowledge Base Audit (知识库自洽性与完整性审计技能)

本技能用于对《LLM Agent & Harness Engineering》知识库进行全维度的概念自洽性、双向链接完整度及认知前沿审计。
**完全基于 Agent 原生工具（Glob、Grep、Read），不依赖任何外置脚本。**

---

## 🚀 快速执行工作流 (Workflow)

当用户发起知识库审计请求时，Agent 按以下四阶段执行：

```text
[阶段 1: 原生工具扫描双向链接与图谱联动 (Glob / Grep)]
        │
        ▼
[阶段 2: 核心概念自洽性审查 (对比 concept-consistency 规则)]
        │
        ▼
[阶段 3: 认知前沿与学习待办梳理 (仅列出，绝不代笔)]
        │
        ▼
[阶段 4: 输出结构化审计报告]
```

---

## 步骤 1：原生工具检查客观工程质量

使用内置工具进行扫描：

1. **扫描双向链接完整性**：
   - 使用 `Glob` 收集工作区内所有 Markdown 文件名清单（如 `harness_definition.md`）；
   - 使用 `Grep` 提取所有笔记中的 `\[\[([a-zA-Z0-9_\-]+)` 引用目标；
   - 比对是否存在引用了不存在的文件（死链）。
2. **扫描体系联动章节**：
   - 检查 `01_concepts/` 与 `02_deep_dives/` 下的各文件，确认文末均包含 `## 🔗 体系联动`。
3. **扫描图片资产有效性**：
   - 检查文档中的 `![alt](../assets/xxx.png)` 是否在 `assets/` 真实存在。

---

## 步骤 2：核心三大概念冲突审查 (Consistency Audit)

比对全库文档是否符合 `.cursor/rules/concept-consistency.mdc` 裁决基准：

### 1. Handoff（智能体移交）三流派审查
- [ ] **Anthropic**: 是否明确强调以“物理工件（Plan.md、Git commit、测试门控）”为跨会话载体？
- [ ] **OpenAI**: 是否明确指出其为“去中心化控制流移交（Agents SDK）”？
- [ ] **LangChain**: 是否明确指出其为“StateGraph 类型化字典（Typed State）”流转？
- [ ] **冲突排查**: 任何文档不得将 Anthropic 物理工件流转与 OpenAI SDK 移交混淆。

### 2. Anthropic 双 Harness 场景审查
- [ ] **长程任务 (SWE-bench)**: 是否阐述了 Initializer 启动、物理工件追踪与 Ralph Loop 拦截退出？
- [ ] **单会话编排 (Dynamic Workflows)**: 是否阐明了 Claude 现场动态生成用过即弃的 JavaScript 编排脚本与 6 种 Harness Pattern？
- [ ] **冲突排查**: 确认未将单会话临时编排脚本误解为“替代了物理工件”，二者是局部并发与全局持久的正交关系。

### 3. 工具爆炸治理与 Cursor 工业实践映射
- [ ] `02_deep_dives/ai_agent_book.md` 中的理论三大方案（Tool RAG、Sub-Agents、两阶段元发现）是否与 `01_concepts/tool_explosion_and_governance.md` 中的 Cursor 工业落地四道防线形成相互映射？
- [ ] 确认未将 Cursor 的实现误述为向量检索（Tool RAG）。

---

## 步骤 3：认知前沿与学习待办梳理 (Learning Backlog Tracking)

使用 `Grep` 检索作者标注的认知前沿与探索标记：
- `后续需要继续了解其机制`
- `待学习` / `待探索`
- `TODO` / `WIP`

⚠️ **核心边界准则**：
1. **严禁 AI 擅自代笔填平**：这些标记是作者本人的【认知前沿地标】，反映作者当前的真实学习边界。知识库必须忠实反映作者真正理解吸收的知识，严禁 AI 擅自扩写并假装已掌握。
2. **作为待办看板呈现**：在审计报告中汇总为“📌 认知前沿与学习待办”，供作者在未来主动学习时查阅或发起深度讨论。

---

## 步骤 4：生成标准审计报告模板

```markdown
# 知识库综合审计报告 (Audit Report)

## 📊 总体工程健康度: [100/100]

## 1. 原生双链与图谱检查 (客观工程层)
- 扫描文档: X 篇
- 双链死链: 0 处
- 本地图片资源: 100% 有效
- 体系联动覆盖率: 100%

## 2. 概念自洽性审查结果 (逻辑层)
- Handoff 机制对齐: ✅ 一致
- Anthropic 双场景解耦: ✅ 一致
- 工具治理理论与工业落地映射: ✅ 一致

## 3. 📌 认知前沿与学习待办 (Learning Backlog - 仅呈现，非缺陷)
- 01_concepts/harness_definition.md:
  • OpenAI 的 handoff 去中心化交接机制待深入了解
  • LangChain 的 StateGraph 状态图流转机制待深入了解
```
