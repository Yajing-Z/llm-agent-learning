
LangChain 说：模型在 post-training 阶段 overfit 到特定 harness。 Anthropic 说：harness 编码了对特定模型能力的假设。

这构成一个**循环依赖**：

```
模型训练时适配 harness → harness 为模型量身设计 → 模型升级 → harness 过时
                ↑                                                    │
                └────────────── 重新设计 harness ←──────────────────┘
```


问题：
* 是否每个模型的提供商最终都会捆绑自己的harness? (事实上Claude code, Codex 已经在做了)
* 跨模型可移植的harness 是否是一个幻觉？

### harness, model, project 三者之间的关系？
（PS: 目前我认为harness中的背压机制（ lint, 测试, 类型检查等） 在不同的project 之间肯定是不同的，那harness 是否其实也是和 project 绑定的？）


Model: ‘大脑’，但它本身是无状态、容易出错且没有外设驱动的。
Harness: 包装在 Model 外的 ‘执行壳’ 或 ‘操作系统’，它提供执行、记忆、安全护栏和验证循环。
Project：Agent 工作的物理边界与事实来源
这三者是不断变化的。无论是 model 更新了，还是 project 有更新，harness 可能相对的也要更新。

**可以把 Harness 理解成两个层次：通用运行时 vs. 项目特化配置**
* Harness 的通用运行时 （Runtime 层），也就是Claude code，Codex在做的，它们提供通用的 编排、MCP、sandbox、Skill、handoff等 （可在 02_harness/concepts/Harness-definition 查看）
* 特化的Harness配置，这一层与Project 强绑定，包括项目专属的 **System Prompts、Custom Skills、Linter 规则、测试套件以及进度文件（Plan.md）**


