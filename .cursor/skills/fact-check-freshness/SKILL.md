---
name: fact-check-freshness
description: Verify specific technical claims or a set of claims against primary specifications, official documentation, versioned source, or runtime artifacts. Use for protocol, implementation, security, and freshness checks; use audit-knowledge-base for whole-repository and cross-document audits.
---

# Technical Fact-Checking & Freshness Audit (技术准确性与时效保鲜审计)

本技能核验单项或一组技术断言。先读取 [事实核验规则](../../rules/fact-checking-freshness.mdc)，采用其中的证据要求、核验状态与报告字段；涉及机制混同时再读取 [概念边界](../../rules/concept-consistency.mdc)。本技能只定义执行流程，不复制技术判据。

全库或跨文档审计由 [综合审计技能](../audit-knowledge-base/SKILL.md) 组织；单项核验无需执行其全库扫描。

---

## 核验流程

1. **定位断言与范围**：记录待核验原文及位置，拆分其中可独立判断的结论；确认版本、产品、运行形态、配置和观察时间。范围不明时保留不确定项。
2. **取得直接证据**：协议断言打开对应版本的规范章节；实现断言查看固定版本源码的定义、配置入口及调用路径；运行行为查看与断言相关的脱敏工件；结构、语法或数学问题记录可复现的检验过程。只读相关资料，记录取不到的证据。
3. **对齐证据与结论**：逐项说明证据支持或否定原文的哪一部分。默认行为要核对启用条件和分支；运行工件只支持其观察范围。避免仅因来源出现相关术语就认定结论成立。
4. **判定核验状态**：使用事实核验规则第四节的分类。证据相互冲突时说明冲突范围与未决条件，不凭模型熟悉程度裁决；学习留白单独记录。
5. **输出发现与建议**：使用共享报告字段，对每个独立断言给出状态、证据及支持关系和建议。仅输出所请求的核验范围；综合审计可直接复用这些记录。

---

## 执行边界

审计请求默认只读，输出发现及修改建议。已有明确修改授权时按其范围执行；用户要求先审批时，先展示具体修改范围与拟替换内容，批准后再写入文件。

未决学习标记单独呈现，不作为必须填补的缺陷，不擅自代笔。核验限制按共享规则报告。
