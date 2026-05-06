# Hermes Agent 框架教程

> 聚焦 **Nous Research Hermes Agent**（开源 agent CLI/Gateway 框架，v0.11.0）
> ⚠️ 本目录讲的是**应用层框架**，不是 Hermes 3 语言模型。模型层请看 `../hermes-agents/`。

作者：沙漠绿洲（Ma Ronnie 知识库）
最后更新：2026-05-06

---

## 📚 三段式教程

| # | 篇章 | 文件 | 状态 | 读者画像 |
|---|------|------|------|---------|
| Part 1 | 入门篇：十分钟让它跑起来 | [part-1-入门篇.md](./part-1-入门篇.md) | ✅ 已完成 | 零基础，会装会用 |
| Part 2 | 进阶篇：把它用成生产力 | part-2-进阶篇.md | ⏳ 待续 | 日常用户，要沉淀工作流 |
| Part 3 | 深入篇：源码级机制 | part-3-源码深入.md | ⏳ 待续 | 开发者，要二次开发 |

原合订版全文：[01-三段式入门到深入教程.md](./01-三段式入门到深入教程.md)（包含 Part 1/2/3 全部内容，可作为速查）

---

## 🗺️ 知识地图

### Part 1【入门】—— 已完成 ✅
```
安装 → 配置 → 三种用法 → 斜杠命令 → 三个实战例子 → 常见错误 → 厂商选型 → 速查卡
```
覆盖：`hermes setup / doctor / model / tools / config`，交互模式、一次性问答、后台任务，
LLM 厂商对照表（Anthropic/OpenAI/OpenRouter/DeepSeek/Gemini/Groq/Bedrock/Ollama），
8 个高频报错自查，本地目录速查，一页纸速查卡。

### Part 2【进阶】—— 待续 ⏳
```
Tools / Toolsets
Skills（技能系统）：结构、frontmatter、linked_files、category
Memory（长期记忆）：user / memory 双库、FTS5 召回
Session 持久化
Cron（定时任务）+ context_from 链式依赖
Gateway（Telegram / Discord / 飞书 / 微信 / Slack / Matrix）
Profiles（多实例隔离）
子 agent 并行（delegate_task / role=orchestrator）
MCP（Model Context Protocol）外部工具接入
```

### Part 3【深入】—— 待续 ⏳
```
总体架构：CLI → AIAgent → ToolRegistry → Provider Adapter → Tool Environment
Agent 主循环（run_agent.py）
Provider 适配层：openai_adapter / anthropic_adapter / bedrock_adapter / …
工具注册与分发（tools/registry.py + model_tools.py）
终端后端（tools/environments/：local / ssh / docker / e2b）
上下文压缩（context_compressor.py）
记忆系统（memory_manager.py + plugins/memory/）
Session 存储（hermes_state.py + SQLite FTS5）
Gateway 架构
插件系统
斜杠命令中央注册
ACP 适配（IDE 集成：Claude Code / Zed）
调试与二次开发入口
```

---

## 🔀 与 `docs/技术学习/hermes-agents/` 的关系

两者名字像，内容完全不同，千万别混：

| 对比项 | 本目录 hermes-agent-framework | 姊妹目录 hermes-agents |
|--------|------------------------------|----------------------|
| 定位 | 应用层 · Agent 运行时框架 | 模型层 · 基础语言模型 |
| 主角 | Hermes Agent CLI (Python) | Hermes 3 / Hermes 4 LLM |
| 关键词 | CLI、Gateway、Skills、Tools、Cron、MCP | ChatML、Function Calling、GOAP、Fine-tune |
| 类比 | Claude Code / Codex CLI | Llama 3 / Qwen 2.5 |
| 出品方 | Nous Research | Nous Research |

可以把它们理解为：**Hermes 3 模型是发动机，Hermes Agent 框架是整辆车**。

---

## ✍️ 写作计划

- [x] Part 1 入门篇（本次完成）
- [ ] Part 2 进阶篇（下一次更新）
- [ ] Part 3 源码深入（下下次更新）
- [ ] 附录：与 Claude Code / Codex CLI / opencode 横向对比
- [ ] 附录：常见 Skill 模板库（量化复盘、GitHub PR 审查、邮件分类…）

如需调整顺序或优先级，直接提。
