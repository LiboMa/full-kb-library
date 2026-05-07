---
来源: Hermes Agent v0.11.0 官方文档 + 各工具官方 GitHub/Docs（见正文链接）
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, claude-code, codex-cli, opencode, agent-cli, 选型, 横向对比]
Ground Truth: ⚠️ 部分对比数据基于官方 README 公开信息，时效性以截至 2026-05 为准，需自行复核
主题: 四大 Agent CLI 横向对比与选型指南
系列: 附录 C（横向对比）
前置: part-1-入门篇.md
后续: 无
对齐版本: Hermes Agent v0.11.0
文档快照: 官方 llms-full.txt @ 2026-05-06（47166 行 / sha256 ae5fbc5e）
复查建议: 每 3 个月或 Hermes 发布 minor 版本后
维护者: 沙漠绿洲（Ma Ronnie）
---

# 附录 C：Hermes Agent vs Claude Code / Codex CLI / opencode

读完 Part 1 你大概率会问：市面上 agent CLI 不止一个，到底该用哪个？
本附录把四个主流选手放一张桌子上拆对比。立场是「选型指南」，不吹 Hermes，
也不捧或踩别家。凡是没在官方文档里查到实锤的，一律标「需自行验证」。

对比对象：

- Hermes Agent — Nous Research 出品，Python，本目录主角（v0.11.0）
- Claude Code — Anthropic 官方 CLI，Node，`@anthropic-ai/claude-code`
- Codex CLI — OpenAI 官方 CLI，Rust/TS，`github.com/openai/codex`
- opencode — SST 团队开源 TUI agent，Go/TS，`github.com/sst/opencode`

官方文档链接：
- Hermes: https://nousresearch.github.io/hermes-agent/
- Claude Code: https://docs.claude.com/en/docs/claude-code
- Codex CLI: https://github.com/openai/codex
- opencode: https://github.com/sst/opencode 、 https://opencode.ai

---

## 1. 一张大表看完

下表内容截至 2026-05-06，字段含「?」或「需验证」的，请以各自官方最新文档为准。

| 维度 | Hermes Agent | Claude Code | Codex CLI | opencode |
|---|---|---|---|---|
| 出品方 | Nous Research（开源社区+商业） | Anthropic（商业闭源壳） | OpenAI（官方开源） | SST（纯社区开源） |
| 定位 | 多平台 agent 运行时 + CLI | 终端编码副驾 | 终端编码副驾（OpenAI 原生） | 轻量 TUI 编码 agent |
| 主语言 | Python ≥3.10 | Node.js | Rust + TS | Go + TS |
| 安装（一行） | `uvx hermes-agent` 或 `pipx install hermes-agent` | `npm i -g @anthropic-ai/claude-code` | `npm i -g @openai/codex` 或下载二进制 | `npm i -g opencode-ai` 或 `brew install sst/tap/opencode` |
| 启动命令 | `hermes chat` / `hermes run` | `claude` | `codex` | `opencode` |
| 默认模型 | 用户自选（见下） | Claude Sonnet / Opus 系列 | GPT-5 / o 系列（OpenAI 专属） | 用户自选 |
| Provider 范围 | OpenAI / Anthropic / Gemini / Bedrock / Azure / Ollama / OpenRouter / DeepSeek / Qwen / MiniMax / 豆包… 20+ | 仅 Anthropic（含 Bedrock/Vertex 转发） | 仅 OpenAI（含 Azure OpenAI） | 多 provider，类似 Hermes（OpenAI/Anthropic/本地等） |
| 上下文窗 | 由所选模型决定；带自动压缩 + 消息裁剪 | 原生 200K（Claude）+ 自动压缩 | 由所选模型决定；支持 `/compact` | 由所选模型决定；支持压缩 |
| Session 持久化 | ✅ SQLite，`hermes sessions` 管理 | ✅ `--resume` / `--continue` | ✅ `codex resume` | ✅ 内置持久化 |
| Built-in 工具 | shell/file/http/search/python/browser/delegate 等 30+ | bash/file/search/web/todo 等 | shell/file/apply_patch 等 | bash/file/edit/web 等 |
| MCP 支持 | ✅（client + server 两端） | ✅（client） | ✅（client，2025 中加入） | ✅（client） |
| 自定义工具 | Python 插件 + YAML 声明 + Tool Gateway HTTP | Hook / slash command / MCP | MCP / 配置脚本 | Plugin（TS）+ MCP |
| Skills 系统 | ✅ 原生 Skills（含 godmode/google-workspace 等） | ✅ Agents/Subagents（2025 下半年推出） | ❌（无对应抽象，需自建 prompt） | ⚠️ 通过 agent 配置模拟 |
| 长期记忆 | ✅ Memory 模块（向量+结构化） | ⚠️ `CLAUDE.md` 项目记忆 + memory tool（beta） | ⚠️ `AGENTS.md` + 会话恢复 | ⚠️ 项目级 `AGENTS.md` 约定 |
| 多平台 Gateway | ✅ Telegram/Discord/Slack/飞书/钉钉/企微/Teams/微信/QQ/BlueBubbles/Open WebUI… 20+ | ❌（纯终端，需自接） | ❌（纯终端） | ❌（纯 TUI） |
| 定时/后台/webhook | ✅ 原生 cron + webhook（如 GitHub PR review） | ❌ 需自己 wrap 为 systemd/cron | ❌ 同上 | ❌ 同上 |
| 子 agent 并行 | ✅ `delegate_task` 工具 + 多 agent 编排 | ✅ Task / Subagent（并行） | ⚠️ 通过 prompt 分解，无原生 delegate | ⚠️ 通过 agent 切换 |
| 沙箱/安全 | 可选 Docker / SSH 远程 / 只读模式 | macOS/Linux seatbelt 沙箱 + 权限提示 | Seatbelt(macOS)/Landlock(Linux) 原生沙箱 | 进程级权限提示（沙箱较弱，需自行验证） |
| IDE 集成 | 通过 ACP / MCP / Web Dashboard；深度 IDE 插件较弱 | VS Code / JetBrains 官方插件 | VS Code 扩展（官方） | Zed ACP 原生，VS Code 支持中 |
| 开源协议 | Apache-2.0 | 闭源（二进制 + EULA） | Apache-2.0 | MIT |
| 商业支持 | 社区为主，Nous 有 API 业务 | Anthropic 商用订阅 + Enterprise | OpenAI ChatGPT Plus/Pro 账户可用 | 社区 + SST 企业服务 |
| 典型场景 | agent 工作流、多平台 bot、自动化 | 在仓库里写/改/改代码 | OpenAI 生态内写代码 | 轻量本地 TUI 写代码 |

字段说明：✅=官方明确支持；⚠️=部分支持或变通方案；❌=没有原生能力；?=截至 2026-05 未核实。

---

## 2. 维度细看（只挑差异大的）

### 2.1 Provider 范围
这是最直观的差别。Claude Code 和 Codex CLI 是各自厂商的"官方皮肤"，你只能
用自家模型（Claude Code 可走 Bedrock/Vertex，但后端仍是 Claude；Codex 可走
Azure OpenAI，但后端仍是 OpenAI 模型）。Hermes 和 opencode 则是 provider-agnostic，
一份配置里可以按任务路由到不同模型，这对"日常用便宜模型、关键任务上旗舰"的
场景很关键。见 Hermes 的 provider 文档和 guides/aws-bedrock、azure-foundry、
google-gemini、local-ollama-setup。

### 2.2 Skills / Memory
Hermes 的 Skills 是命名空间化的能力包（见 `user-guide/skills/`），带 prompt、
工具、上下文注入一整套；Memory 模块把历史结构化落库。Claude Code 走的是
"Subagent + CLAUDE.md + memory tool"组合，形态上接近但没有 Skills 的打包概念。
Codex CLI 和 opencode 目前主要靠 `AGENTS.md` 这类项目级 markdown 约定，
自动化记忆较弱（截至 2026-05，需自行验证最新动向）。

### 2.3 多平台 Gateway
这是 Hermes 独一档的维度。它原生把同一个 agent 接到 Telegram/Discord/Slack/
飞书/钉钉/企微/Teams/微信/QQ/BlueBubbles(iMessage)/Open WebUI/Yuanbao 等
20+ 平台（见 `user-guide/messaging/*`）。其他三家都是纯终端，要做 bot 得
自己套一层。换句话说：如果你要把 agent 做成"群里一个账号"，这一维直接决定选型。

### 2.4 定时 / Webhook
Hermes 内置 cron 和 webhook（参考 `guides/webhook-github-pr-review.md`、
`guides/cron-*`）。另外三个都需要你在外层写 systemd timer / GitHub Action /
自建 HTTP server 来触发。不是做不了，是工作量差一个数量级。

### 2.5 沙箱 / 安全
Codex CLI 这点做得相当扎实：macOS 用 Apple Seatbelt、Linux 用 Landlock，
默认不联网、只读文件系统，要写才提升权限。Claude Code 也有类似的 seatbelt
+ 交互式权限确认。Hermes 的策略偏"可选"：支持 Docker 隔离、SSH 跳板、
只读模式，但默认信任本地环境，安全基线弱于 Codex。opencode 的沙箱能力
截至 2026-05 较弱，建议在 VM/容器里跑。

### 2.6 IDE 深度集成
这一维 Hermes 明显落后。Claude Code 有官方 VS Code / JetBrains 插件，
Codex CLI 有 VS Code 扩展，opencode 因为 SST 团队的关系在 Zed ACP
集成上很顺。Hermes 目前走 Web Dashboard + MCP + ACP 通用路径，没有
一等公民的 IDE 插件（截至 v0.11.0）。如果你大部分工作都在 IDE 里，
这是个切实的体验差。

---

## 3. 什么时候选哪个

### 3.1 选 Claude Code
你主要工作是"在一个代码仓库里写代码、改 bug、做 refactor"，你付得起或
公司买了 Claude 订阅，你喜欢 Claude 的代码风格和 200K 原生窗。Claude Code
在"懂仓库、会 plan、会长上下文编辑"这件事上目前是第一梯队，IDE 插件也齐。
劣势是锁在 Anthropic 一家、贵、脱离仓库做 agent 工作流（发消息、跑定时、
接多平台）几乎没有原生方案。纯编码党 + 有预算 + 不要多平台，闭眼选它。

### 3.2 选 Codex CLI
你深度绑定 OpenAI 生态，有 ChatGPT Plus/Pro 或 API 账号，偏好 GPT-5/o 系列
的推理风格；或者你对"默认沙箱 + 默认不联网"这种保守安全模型有硬需求
（审计、企业合规）。Codex 在代码改动上走 `apply_patch` 这类结构化 diff，
可复现性不错，而且开源（Apache-2.0）。劣势是 provider 只有 OpenAI、
没有 Gateway、没有定时、Skills/Memory 生态薄。OpenAI 原教旨主义 + 重视
安全基线的场景首选。

### 3.3 选 opencode
你想要一个轻量、开源、TUI 友好、可以随便换 provider 的本地 agent。
你可能用 Zed，希望通过 ACP 拿到 IDE 内的 agent 体验；或者你在一台开发机
本地跑 Ollama，想要个 UI 比裸 CLI 好但又不想要 web dashboard 那么重的东西。
opencode 的社区活跃、MIT、上手极快。劣势是没有 Gateway、没有 cron/webhook、
Skills 抽象弱、沙箱较浅。"个人开发者 + 本地 + 爱折腾 TUI"的首选。

### 3.4 选 Hermes Agent
你要做的不只是"在一个仓库里写代码"，而是一个持续运行的 agent：接到
飞书/Telegram/Discord 群里答客服、跑定时任务、收 GitHub webhook 自动
review PR、把 Skills 组合起来做 Google Workspace / 音乐 / 浏览器自动化、
在多 provider 之间切换省钱。这些是 Hermes 的原生形态。劣势是 IDE 集成
比不过 Claude Code、沙箱默认弱于 Codex、商业支持远不如前两家官方产品、
Python 栈对纯前端 / Go 工程师不那么友好。"Agent 工作流 + 多平台 + 自动化"
场景就是它的主场。

---

## 4. 从 Claude Code / Codex CLI 迁移到 Hermes

如果你已经熟悉 Claude Code 或 Codex CLI，迁到 Hermes 最容易踩坑的几点：

1. 配置位置不同：Hermes 在 `~/.hermes/`（config.yaml + sessions.db + skills/），
   不是 `~/.claude/` 或 `~/.codex/`。初次迁移建议直接跑 `hermes setup` + `hermes doctor`。
2. 模型要显式选：Hermes 默认不假设你用哪家，首次启动必须配 provider + model。
   对比 Claude Code/Codex 开箱即用的体验，这步多 2 分钟。
3. `CLAUDE.md` / `AGENTS.md` 没有直接对应。Hermes 用 Skills + Memory 来承载
   这类"项目常识"，语义更丰富，但你得学一下 Skills 怎么写（见 `user-guide/skills/`）。
4. 权限模型不一样。Claude Code/Codex 默认收紧、按需提示；Hermes 默认更放，
   想要收紧得自己开启 Docker/只读模式。生产环境强烈建议加沙箱。
5. 子 agent：Claude Code 的 Subagent 和 Hermes 的 `delegate_task` 概念接近，
   但 API 不同。如果你重度用 Claude 的 Task 工具，迁移时需要重写编排层。
6. 终端 UX：Claude Code/Codex/opencode 都有比较精致的 TUI。Hermes 的终端
   交互偏朴素，精致界面在 Web Dashboard 那一侧（`hermes dashboard`）。

---

## 5. 诚实标注：Hermes 的强项与短板

明显领先：

- 多平台 Gateway（20+ 平台，其他三家都没有）
- 定时 + webhook 原生支持
- Provider 覆盖最广（含国内厂商：豆包/MiniMax/Qwen/Yuanbao）
- Skills + Memory 这套抽象比其他家的 markdown 约定成熟
- Apache-2.0 开源、可自部署

明显落后或不成熟（截至 v0.11.0 / 2026-05）：

- IDE 深度集成弱于 Claude Code 和 opencode(Zed)
- 默认沙箱基线弱于 Codex CLI
- 纯编码任务的"改代码质量"受限于你选的模型，本身没有 Claude Code 那种
  围绕 Claude 定制的 prompt 工程红利
- 商业支持（SLA、企业合同、合规认证）远不如 Anthropic/OpenAI 官方
- 生态（第三方插件市场、教程数量）比不过 Claude Code
- Python 依赖链对非 Python 工程师有一定门槛

以上几条如果命中你的硬需求，应该诚实选别家，不用勉强 Hermes。

---

## 6. 懒人建议（TL;DR）

- 只是在仓库里写代码、改 bug、做 refactor → **Claude Code**
- 深度 OpenAI 生态 / 要强沙箱基线 → **Codex CLI**
- 想要轻量开源 TUI、随便切 provider、本地跑 → **opencode**
- 要做 agent 工作流 / 多平台 bot / 定时自动化 / 跨 provider → **Hermes Agent**

四者不是零和，完全可以同机共存：用 Claude Code 写代码、用 Hermes 跑 bot
和定时任务、用 Codex 做需要强沙箱的自动脚本、用 opencode 在 Zed 里搭一个
轻量副驾。选对工具比忠于一个品牌更重要。

---

参考来源：

- Hermes Agent 官方文档：https://nousresearch.github.io/hermes-agent/
- Hermes llms-full.txt 快照 @ 2026-05-06（本目录 Ground Truth）
- Claude Code 文档：https://docs.claude.com/en/docs/claude-code
- Codex CLI GitHub：https://github.com/openai/codex
- opencode GitHub：https://github.com/sst/opencode
- opencode 官网：https://opencode.ai

本附录所有"官方不清楚"的条目，以各自上游为准，欢迎 PR 勘误。
