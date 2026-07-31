
https://github.com/browser-use/browser-harness

CDP（Chrome DevTools Protocol）
- **直连真实浏览器（零中间层）：** 传统的自动化工具往往封装了厚厚的抽象层，而它通过极轻量的 CDP（Chrome DevTools Protocol）管道将 LLM / Coding Agent 直接连接到你的真实 Chrome 浏览器，极大提升了控制自由度和执行效率。
    
- **自愈与自我演进能力（Self-healing & Learning）：** 在任务执行过程中，如果缺少某个特定操作函数或选择器，Agent 会直接编写或修正 helper 代码（如 `agent_helpers.py`）以及特定网站的技能包（`domain-skills`）。下一次运行时，它就能复用这些经验，实现“越用越聪明”。
    
- **无缝对接开发者工作流：** 专为 Claude Code、Codex 等终端 Coding Agent 设计，只需粘贴一段 Setup Prompt 即可完成安装部署与连接，极大地降低了上手门槛。

