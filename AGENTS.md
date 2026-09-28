# AGENTS.md

## Repository Mission
本仓库为《LLM Agent & Harness Engineering》高阶工程知识库。
核心公理：$$\text{Harness} = \text{Agent} - \text{Model}$$
目标是系统性阐明工业级 Agent 执行外壳、背压回路、动态工作流与上下文治理机制。

## Golden Rules
1. **权威与一手事实**：技术阐述必须基于一手规范（MCP Spec）、官方工程博客（Anthropic/OpenAI/LangChain）或源码反编译（如 Cursor 核心库），严禁脑补与二传手失真。
2. **概念绝对自洽**：严格遵循 `.cursor/rules/concept-consistency.mdc` 裁决矩阵，严禁混淆三大 Handoff 机制、Anthropic 长程物理状态与动态 JS 编排场景，对齐工具治理理论与工业落地。
3. **客观工程背压**：修改文档时，必须确保 Obsidian 双链 `[[...]]` 真实存在、图片路径有效，文末维护 `## 🔗 体系联动`。
4. **尊重认知留白，严禁擅自代笔**：这是学习者的个人认知沉淀库，文档必须真实反映人类作者已理解消化的知识。正文中的“后续需要继续了解其机制”等标记是作者的【认知前沿地标】，AI 严禁擅自代笔填补用户尚未亲身学习掌握的知识点。
5. **轻量与敏捷纠偏**：优化迭代速度而非首次成功率；遵守最小特权原则与上下文防火墙。

## Repository Layout
- `01_concepts/`: 核心范式、底层架构与工程基石
- `02_deep_dives/`: 厂商实践逆向、机制专题与 Wire-Level 通信报文
- `assets/`: 架构图、通信序列图与逆向截图
- `.cursor/`: Cursor 认知规则 (rules) 与审计技能 (skills)
