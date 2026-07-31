
吞吐量改变合并理念（Throughput Changes Merge Philosophy）

- Ralph Wiggum 循环：Codex 本地审核 → 请求额外智能体审查 → 对反馈做出响应 → 循环直到所有审核通过

HumanLayer --- 优化迭代速度而非首次成功率
实战结论：
* ❌ 每次改动跑全量测试
* ✅ 优化迭代速度，快速发现和修复问题
* ✅ 便宜模型 Sonnet/Haiku 做子任务，贵模型 Opus 做编排

**前提条件：必须有足够的背压机制（测试、lint、结构检查）来保证基本质量，否则就不是"快速迭代"而是"快速腐烂"。**

### LangChain — Ralph Loop 机制

长时间自主执行需要：

1. 文件系统 + git 追踪持久化工作
2. **Ralph Loop** 拦截退出，在新上下文窗口中重注入原始提示词
3. 规划 + 自我验证分解目标为步骤
