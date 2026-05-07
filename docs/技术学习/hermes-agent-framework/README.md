# Hermes Agent 框架教程

> 聚焦 **Nous Research Hermes Agent**（开源 agent CLI/Gateway 框架）
> ⚠️ 本目录讲的是**应用层框架**，不是 Hermes 3 语言模型。

作者：沙漠绿洲（Ma Ronnie 知识库）
最后更新：2026-05-06

---

## 📚 三部曲 + 附录

| # | 篇章 | 文件 | 读者画像 |
|---|------|------|---------|
| 📍 | 从这里开始（按画像选路径） | [00-从这里开始.md](./00-从这里开始.md) | 所有人先读这个 |
| Part 1 | 入门篇：十分钟让它跑起来 | [part-1-入门篇.md](./part-1-入门篇.md) | 零基础 |
| Part 2 | 进阶篇：把它用成生产力 | [part-2-进阶篇.md](./part-2-进阶篇.md) | 日常用户 |
| Part 3A | 架构篇：总体架构与主循环 | [part-3a-架构与主循环.md](./part-3a-架构与主循环.md) | 想懂原理 |
| Part 3B | 架构篇：环境存储与扩展 | [part-3b-环境存储与扩展.md](./part-3b-环境存储与扩展.md) | 想二次开发 |
| 附录 | 最佳应用场景 12 例 + 速查表 | [appendix-最佳应用场景与速查表.md](./appendix-最佳应用场景与速查表.md) | 选型参考 |
| 归档 | 合订版（已被新版取代） | [01-三段式入门到深入教程.md](./01-三段式入门到深入教程.md) | 历史记录 |

完整学习路径：小白 → 会用 → 进阶使用 → 懂架构 → 懂原理 → 选对场景。

---

## 🗺️ 知识地图

### Part 1【入门】
```
安装 → 配置 → 三种用法 → 斜杠命令 → 三个实战 → 常见错误 → 厂商选型 → 速查卡
```
`hermes setup / doctor / model / tools / config`，交互模式、一次性问答、后台任务，
Anthropic/OpenAI/OpenRouter/DeepSeek/Gemini/Groq/Bedrock/Ollama 对照表。

### Part 2【进阶】
```
Tools / Toolsets（启用/禁用/权限）
Skills（frontmatter / linked_files / category）
Memory（user + memory 双库 / FTS5 召回）
Sessions + 上下文压缩（/new /compress）
Cron（调度 + context_from 链式 + script 注入）
Gateway（Telegram / Discord / Slack / 飞书 / 微信 / Matrix …）
delegate_task 子 agent 并行（leaf vs orchestrator）
MCP（stdio / HTTP）
Voice Mode / Personality
```

### Part 3A【架构上半】
```
总体架构图（CLI → AIAgent → ToolRegistry → Provider → Environment）
Agent 主循环（run_conversation 9 步）
Prompt 10 层装配
上下文压缩 + Prompt Cache
Provider 适配层（20+ 厂商如何统一）
Tools Runtime（注册、分发、审批）
```

### Part 3B【架构下半】
```
五种 Environment 后端（local / docker / ssh / singularity / modal）
Session 存储（SQLite + FTS5 + trajectory 格式）
Gateway 架构（message adapter / 事件路由 / thread 管理）
二次开发入口：加 tool / 加 provider / 加平台 adapter / 扩展 CLI
调试技巧（HERMES_DEBUG / trajectory 复盘）
贡献流程
```

### 附录 A / B
```
A. 12 个最佳应用场景（每个含方案 + 配置片段）：
   1) 个人知识管家  2) A 股 / 美股周报  3) GitHub PR 审查
   4) 邮件分类      5) 多 agent 研究    6) Telegram 群管家
   7) CI/CD 诊断    8) Ollama 本地隐私  9) 团队共享 Gateway
   10) MCP 跨工具桥接 11) 语音对话助手   12) 长周期后台任务
   + 场景选型对照表
B. 速查表：CLI / 斜杠命令 / 关键配置字段 / 关键环境变量
```

---

## 🔀 与 `hermes-agents/` 的关系

两个目录名字像，内容不同：

| 对比项 | 本目录 hermes-agent-framework | 姊妹目录 hermes-agents |
|--------|------------------------------|----------------------|
| 定位 | 应用层 · Agent 运行时框架 | 模型层 · 基础语言模型（原计划，已废弃） |
| 主角 | Hermes Agent CLI (Python) | Hermes 3 / Hermes 4 LLM |
| 关键词 | CLI、Gateway、Skills、Tools、Cron、MCP | ChatML、Function Calling |

**Hermes 3 模型是发动机，Hermes Agent 框架是整辆车。** 本目录只讲车。

---

## ✍️ 完成状态

- [x] Part 1 入门篇
- [x] Part 2 进阶篇
- [x] Part 3A 架构与主循环
- [x] Part 3B 环境存储与扩展
- [x] 附录 A 最佳应用场景
- [x] 附录 B 速查表
