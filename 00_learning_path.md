---
tags: [MOC, roadmap]
created: 2026-06-29
---

# 00_Learning_Path — 企业级 AI Agent 全栈工程落地学习路线图 (V2.0)

> 本路线图的目标：**从零基础到具备 Harness Engineering / Loop Engineering 岗位的应聘能力**。
>
> 预计总时长：**8 周**（每天 30-60 分钟微任务 + 周日 2-4 小时硬核实战）

---

## 🧭 核心竞争力宣言

在企业级场景中，Agent 的核心竞争力绝非“Prompt 调优”，而是**工程化落地能力**。本路线图严格遵循 AI 协作方式的四次升级路径：

1. **Prompt Engineering** — 学会“怎么说”
2. **Context Engineering** — 学会“给什么信息”
3. **Harness Engineering（约束工程）** — 解决单次运行的安全性、合规性与可靠性
4. **Loop Engineering（循环工程）** — 将 Agent 变成可重复调度、能自我校验、支持断点续传与多智能体并行协同的工业级组件

---

## ⚙️ 弹性执行机制

- **周一至周五（每日微任务，30-60 分钟）**：侧重理论输入、源码阅读和核心概念理解。若某天因故未能完成，**允许直接顺延**。
- **周六（弹性缓冲日）**：不安排新任务，专门用于补齐周一至周五遗留的微任务。
- **周日（硬核实战日，2-4 小时）**：**必须雷打不动地完成**。这一天你将通过**从零手写（From Scratch）** 的方式，不依赖任何第三方 Agent 框架，亲手实现本周的核心工程组件，建立最扎实的底层直觉。

---

## 📅 八周攻坚计划

### 第一周：Prompt Engineering 深度筑基

**核心目标**：掌握结构化提示词设计的全部核心技巧，能够针对不同任务类型（分类、生成、推理、代码）设计对应的提示词策略。

#### 📅 每日任务（Mon-Fri）

| 日期     | 任务                                              | 产出                                                                                         |
| ------ | ----------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **周一** | 结构化提示词设计：Role + Task + Constraints + Format 四要素 | 写一篇笔记 `Prompt_Structure.md`，用你自己的话解释四要素                                                    |
| **周二** | Zero-shot / Few-shot 原理与适用场景                    | 对比实验：同一任务用 Zero-shot vs Few-shot，记录效果差异                                                    |
| **周三** | Chain of Thought (CoT) 思维链                      | 阅读 [Wei et al., 2022](https://arxiv.org/abs/2201.11903) 论文摘要，写一篇 `CoT.md`，附一个你自己设计的 CoT 示例 |
| **周四** | Prompt Chaining 与 Role Playing                  | 设计一个“面试模拟器”的 Prompt Chain（至少 3 步）                                                          |
| **周五** | 输出结构化控制（JSON / XML / Markdown）                  | 写一个强制输出 JSON 格式的 Prompt，并用 API 调试验证                                                        |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 1】手写一个 Prompt 测试框架**
- **任务说明**：不使用任何第三方库，纯用 Python 标准库，构建一个批量 Prompt 测试工具。
- **实现要求**：
  1. 读取一个 `test_cases.json` 文件，包含 10 个不同任务的测试用例
  2. 对每个测试用例，用 3 种不同的 Prompt 策略（Zero-shot、Few-shot、CoT）分别调用 LLM API
  3. 记录每次调用的：输入 Token 数、输出 Token 数、响应时间
  4. 输出一份对比报告，量化不同策略的效果差异


### 第二周：LLM 基础设施与 Context Engineering

**核心目标**：理解大模型服务底层的显存与计算瓶颈，掌握在高并发、长上下文场景下降低延迟和成本的工程手段。

#### 📅 每日任务（Mon-Fri）

| 日期     | 任务                                        | 产出                                                                                                                                |
| ------ | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **周一** | 自回归生成与 Self-Attention 的计算瓶颈               | 观看 [Andrej Karpathy: Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY) 中 Attention 机制的讲解                              |
| **周二** | **KV Cache** 原理                           | 阅读 [Hugging Face KV Cache Guide](https://huggingface.co/docs/transformers/main/en/llm_tutorial#kv-cache)，写一篇 `KV_Cache.md` 用流程图解释 |
| **周三** | **Prompt Caching** 提示词缓存                  | 阅读 [Anthropic Prompt Caching Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)，理解命中缓存的计费规则              |
| **周四** | 推理引擎 vLLM 的 PagedAttention                | 阅读 [vLLM 论文](https://arxiv.org/abs/2309.06180) 摘要，理解虚拟内存分页管理                                                                      |
| **周五** | Context Engineering vs Prompt Engineering | 写一篇 `Context_Engineering.md`，用对比表格说明两者的本质区别                                                                                       |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 2】手写流式 API 客户端 + Prompt Caching 性能评测工具**
- **任务说明**：不使用 `openai` 等第三方 SDK，纯用 `httpx` 库直接调用支持 Prompt Caching 的 API。
- **实现要求**：
  1. 手动解析 Server-Sent Events (SSE) 流，实现终端打字机效果
  2. 构造一个包含 15k Token 背景知识的 System Prompt
  3. 连续发送 5 轮对话，记录并对比：**首轮（未命中缓存）** vs **后续轮次（命中缓存）** 的 TTFT（首字延迟）和计费 Token 数
  4. 用数据量化缓存带来的性能与成本优势


### 第三周：Harness Engineering 基础——安全沙箱与工具约束

**核心目标**：构建单个 Agent 单次运行的物理与逻辑边界，确保 Agent 无法危害宿主机。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | Agent 工具执行的安全隐患分析 | 研究：Prompt 注入攻击如何让 Agent 执行 `rm -rf /`，写一篇安全威胁笔记 |
| **周二** | 安全沙箱技术对比 | 对比 Docker 容器隔离 vs WASM 运行时 vs E2B 沙箱，写对比表格 |
| **周三** | **Model Context Protocol (MCP)** 全面攻克 | 阅读 [MCP 官方文档](https://modelcontextprotocol.io/)，理解 JSON-RPC 2.0 规范 |
| **周四** | 工具调用的参数校验（Schema Validation） | 学习 Pydantic 如何强约束 LLM 输出的工具参数格式 |
| **周五** | 企业级 Harness 架构设计 | 画出 Harness 五大子系统（指令/状态/验证/范围/会话）的关系图 |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 3】构建安全的 Python 代码执行沙箱 + MCP 数据库安全网关**
- **任务说明**：为单个 Agent 的单次运行构建安全约束容器。
- **实现要求**：
  1. **安全代码沙箱**：使用 `docker-py` 库，动态启动轻量级 Docker 容器（限制 CPU 0.1 核，内存 50MB，无外网，超时 5 秒），运行 Agent 生成的 Python 代码并捕获输出
  2. **MCP 安全网关**：使用 `mcp` Python SDK，连接本地 SQLite 数据库，将 Schema 暴露为 MCP Resource，将“安全 SQL 查询”（限制返回 50 行，通过 AST 禁止写操作）暴露为 MCP Tool


### 第四周：Harness Engineering 高级——确定性回放与安全防线

**核心目标**：解决单次运行中的安全拦截与调试难题，确保任何失败都能 100% 确定性复现。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | 确定性回放（Deterministic Replay）原理 | 理解为什么 Temperature > 0 会导致 Bug 极难复现 |
| **周二** | 捕获与 Mock 所有 I/O 边界 | 研究：如何 Mock LLM 响应、时间戳、工具返回值来实现 100% 复现 |
| **周三** | **Input Guardrails（输入防线）** | 研究如何检测和拦截 Prompt 注入攻击和越狱提示词 |
| **周四** | **Output Guardrails（输出防线）** | 研究 PII 脱敏（身份证号、手机号、API Key 自动遮蔽） |
| **周五** | 阅读 [NVIDIA NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) 设计思想 | 写一篇 Guardrails 设计模式的总结笔记 |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 4】手写确定性回放调试 Harness + I/O 安全护栏**
- **任务说明**：为单个 Agent 的单次运行构建完整的调试与安全护栏。
- **实现要求**：
  1. **回放 Harness**：设计 `SessionRecorder`，将所有 LLM API 输入输出、工具调用参数、返回值、时间戳按顺序记录到 `trace_log.json`
  2. **确定性重放**：编写 `ReplayDebugger`，读取 `trace_log.json`，**在完全断网、不消耗 Token、不调用真实工具**的情况下，单步回放 Agent 运行过程
  3. **I/O 护栏**：输入端正则检测注入攻击；输出端实现手机号、邮箱、API Key 的自动脱敏


### 第五周：Loop Engineering 核心——状态机、断点续传与自我纠错

**核心目标**：在 Harness 之上，构建高可用、可中断、防死循环、能自我纠错的鲁棒状态机。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | 阅读经典论文 [ReAct](https://arxiv.org/abs/2210.03629) | 理解 Thought → Action → Observation 基本循环，画流程图 |
| **周二** | **State Checkpointing（状态持久化）** | 研究为什么每一步 Loop 结束时必须将 State 原子性写入数据库 |
| **周三** | **Human-in-the-Loop（人工介入）** | 研究“主动挂起（Interrupt）”和“信号唤醒（Resume）”的实现 |
| **周四** | **Self-Correction（自我纠错）** | 研究当沙箱报错时，Loop 如何将 Traceback 喂回模型让其自愈 |
| **周五** | 阅读 [LangGraph 架构文档](https://langchain-ai.github.io/langgraph/) | 重点研究 `StateGraph`、`SqliteSaver`、`interrupt` 的设计模式 |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 5】从零手写支持“断点续传 + 自我纠错 + 人工审批”的 Agent Loop 引擎**
- **任务说明**：不使用 LangGraph 或任何第三方框架，纯 Python 实现高可用 Agent 运行容器。
- **实现要求**：
  1. **状态序列化**：定义 `AgentState` 类，支持序列化为 JSON 并写入 SQLite
  2. **断点续传**：模拟程序在第 3 步被 `kill -9` 强行终止，重启后自动读取最新 Checkpoint 继续执行
  3. **自我纠错**：沙箱执行报错时，Loop 自动捕获 Traceback，拼回 Prompt 让 Agent 重试，最多 3 次
  4. **人工审批**：提供 `execute_refund` 工具，调用时 Loop 自动挂起为 `PENDING_APPROVAL`，等待终端输入 `y/n` 后继续


### 第六周：Loop Engineering 高级——多智能体协同与 Subagent

**核心目标**：攻克复杂业务下的“分而治之”策略，掌握多 Agent 的状态流转、上下文隔离与并行调度。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | 单 Agent 处理超长任务的瓶颈分析 | 分析上下文污染、注意力涣散问题，写一篇“为什么需要多 Agent” |
| **周二** | **Supervisor-Subagent 模式** | 研究主管 Agent 如何动态创建、分配、销毁子 Agent |
| **周三** | **Choreography（事件驱动编排）** | 阅读 [LangGraph Multi-Agent 教程](https://langchain-ai.github.io/langgraph/)，理解共享全局 State |
| **周四** | 阅读 [CrewAI 官方文档](https://docs.crewai.com/) | 理解基于角色扮演的多 Agent 协作设计 |
| **周五** | 多 Agent 通信协议与并行执行 | 研究如何避免 Agent 之间互相推卸责任或陷入对话死循环 |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 6】手写“软件开发与安全审计”多智能体协同网络**
- **任务说明**：不依赖第三方多 Agent 框架，纯 Python 实现三角色协同开发系统。
- **实现要求**：
  1. **Supervisor Agent**：唯一入口，接收需求（如“写快速排序算法”），动态唤起 Coder Agent
  2. **Coder Agent (Subagent 1)**：专属 Prompt + 文件写入工具，编写完代码后提交给全局 State
  3. **Security Auditor Agent (Subagent 2)**：专属 Prompt（代码安全审计），发现漏洞时将任务退回 Coder 附修改意见；审计通过则汇报“安全通过”
  4. **上下文隔离**：Coder 和 Auditor 运行在独立 LLM 会话中，中间思考不污染 Supervisor 对话历史


### 第七周：企业级记忆与可观测性

**核心目标**：赋予 Agent 长期进化能力，建立工业级的监控体系。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | **Letta (MemGPT)** 记忆架构 | 阅读 [MemGPT 论文](https://arxiv.org/abs/2310.08560)，理解 Core Memory vs Archival Memory |
| **周二** | **Mem0** 记忆架构 | 阅读 [Mem0 GitHub](https://github.com/mem0ai/mem0)，理解异步事实提取机制 |
| **周三** | **OpenTelemetry** 全链路追踪 | 研究 [Arize Phoenix](https://github.com/Arize-AI/phoenix) 核心架构 |
| **周四** | **LLM-as-a-Judge** 机制 | 阅读 [G-Eval 论文](https://arxiv.org/abs/2303.16634)，理解 Judge 的 Rubric 设计 |
| **周五** | RAGAS 评测指标 | 研究 Faithfulness、Answer Relevance 等量化评估指标 |

#### 📅 周六（弹性缓冲日）
- 补齐本周未完成的任务

#### 📅 周日（硬核实战日）
**【每周硬核实战 7】为 Agent 接入 Phoenix Tracing + 构建自动化 CI/CD 评测流水线**
- **任务说明**：为之前编写的 Agent 注入监控，并建立自动化测试集。
- **实现要求**：
  1. **全链路 Tracing**：使用 `phoenix.otel` SDK 埋点，每次 LLM 请求、工具执行都作为独立 Span
  2. **黄金数据集**：准备 10 个包含标准答案的测试用例
  3. **自动化评测**：编写 `eval_runner.py`，跑完测试用例，用更强模型作为 Judge 进行 1-5 分打分，输出通过率、平均延迟、平均成本的评测报告


### 第八周：Capstone 综合实战

**核心目标**：将前七周所有知识融会贯通，完成一个完整的端到端项目。

#### 📅 每日任务（Mon-Fri）

| 日期 | 任务 | 产出 |
|---|---|---|
| **周一** | 需求分析与架构设计 | 确定 Capstone 项目选题，画出完整架构图 |
| **周二** | Prompt + Context 层实现 | 设计所有提示词和上下文管理 |
| **周三** | Harness 层实现 | 搭建五大子系统 + 安全沙箱 |
| **周四** | Loop 层实现 | 实现断点续传、自我纠错、人工审批 |
| **周五** | 评测与文档 | 运行评测流水线，撰写 README 和架构文档 |

#### 📅 周六（弹性缓冲日）
- 完善项目代码和文档

#### 📅 周日（最终交付日）
**Capstone 项目最终交付**

**选题建议**（三选一）：

| 选题 | 描述 | 亮点 |
|---|---|---|
| **A. 代码重构 Agent** | 自动完成代码分析→重构→测试→提交 PR | 覆盖 Tools、Harness、Loop |
| **B. 知识库问答 Agent** | 基于 Obsidian 笔记自动检索→生成→自检 | 覆盖 RAG、Guardrails、评测 |
| **C. 自动周报 Agent** | 多源数据收集→分析→生成→自我检查→发送 | 覆盖多 Agent、Tracing、记忆 |

**交付物清单**：
- [ ] 完整项目代码（GitHub 仓库）
- [ ] README：背景、架构图、使用说明
- [ ] Harness 配置文件（AGENTS.md、init.sh）
- [ ] Loop 设计文档（含 Mermaid 流程图）
- [ ] 评测报告（通过率、延迟、成本数据）
- [ ] 学习笔记（Obsidian 仓库链接）


## 🎯 核心竞争力通关标志

完成这 8 周的学习和实战后，你将拥有以下**企业面试的降维打击武器**：

1. **“我不依赖 LangChain 或 LlamaIndex。我能用纯 Python 手写高可用的 Agent 运行时，支持断点续传、精确到工具级的容错、以及高危操作的人工审批拦截。”**

2. **“我能架构安全的 Agent 执行沙箱，防止任意代码注入攻击；并且我实现了一套确定性回放 Harness，能 100% 离线复现生产环境中的偶发性 Bug。”**

3. **“我懂底层的 KV Cache 和 Prompt Caching 机制，能为公司在大规模长上下文场景下优化 80% 的 Token 账单和延迟；我建立过基于 LLM-as-a-Judge 的 Agent CI/CD 评测流水线，用数据证明每一次 Prompt 迭代的优劣。”**


## 📊 进度追踪表

| 周次 | 主题 | 开始日期 | 完成日期 | 状态 |
|---|---|---|---|---|
| 第 1 周 | Prompt Engineering 深度筑基 | ___ | ___ | ⬜ |
| 第 2 周 | LLM 基础设施 + Context Engineering | ___ | ___ | ⬜ |
| 第 3 周 | Harness 基础——安全沙箱 + 工具约束 | ___ | ___ | ⬜ |
| 第 4 周 | Harness 高级——确定性回放 + 安全防线 | ___ | ___ | ⬜ |
| 第 5 周 | Loop 核心——状态机 + 断点续传 + 自我纠错 | ___ | ___ | ⬜ |
| 第 6 周 | Loop 高级——多智能体协同 + Subagent | ___ | ___ | ⬜ |
| 第 7 周 | 企业级记忆 + 可观测性 + CI/CD 评测 | ___ | ___ | ⬜ |
| 第 8 周 | Capstone 综合实战 | ___ | ___ | ⬜ |

---

> **最后提醒**：学习过程中遇到的每一个疑惑、每一次踩坑、每一个顿悟时刻，都值得记下来。**这些记录本身就是你求职时最好的故事素材**。
>
> 祝你学习顺利，早日成为合格的 Harness / Loop Engineer！🚀