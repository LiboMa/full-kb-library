---
来源: 基于 Hermes Agent v0.11.0 源码调研 + 官方 developer-guide
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, architecture, agent-loop, prompt, provider, tools, nous-research]
Ground Truth: ✅ 基于 /tmp/hermes-docs-split/developer-guide__*.md 六个文件
主题: Hermes Agent 源码深入（上）—— 架构 / 主循环 / Prompt / 压缩 / Provider / 工具
系列: 三段式教程 Part 3 / 3（上半部分 A–F）
前置: part-2-进阶篇.md
后续: part-3b-环境存储与扩展.md、appendix-最佳应用场景与速查表.md
---

# Hermes Agent 源码深入（上）：从架构图到主循环

> 这一段是写给「想改 Hermes / 想给它加 provider / 想给它加工具 / 想把它嵌进自己产品」的人看的。
> 不再教怎么用，直接拆源码、画图、列模块路径。

## 0. 为什么要看架构

Part 2 里我们把所有命令、配置、skill、plugin、gateway、cron 都摸了一遍。那一段是「用户视角」：知道有哪些开关、哪些文件、哪些命令。

但一旦你要做下面这几件事，用户视角就不够了：

- 想接一个 Hermes 还不支持的 LLM 厂商；
- 想往 agent loop 里塞一个拦截钩子（比如全局限速、审计日志）；
- 想把「终端执行」改成「沙箱执行」，或者换个 session 存储；
- 想知道为什么 context 被压成了那个样子、为什么 prompt cache 有时候命中有时候不命中；
- 想在消息网关层做二次分发（把一个 agent 同时暴露给 Telegram + Slack + 企微）。

这时候你需要读代码。而 Hermes 仓库光 `run_agent.py` 就 1.37 万行，`cli.py` 1.15 万行，`gateway/run.py` 1.22 万行 —— 直接打开会迷路。

所以 Part 3 的写法是：**先给你一张总图 + 每个子系统的入口文件，再带你把主要控制流走一遍**。上半部分（本文，A–F）讲 agent 内核；下半部分（part-3b）讲环境后端、session 存储、gateway、二次开发。

---

## A. 总体架构：五层一眼看完

Hermes 不是单体，但也不是微服务。它是「一个 Agent 内核 + 多个入口 + 多个后端」的结构。

```
┌─────────────────────────────────────────────────────────────────────┐
│  入口层 Entry Points                                                 │
│                                                                      │
│   CLI (cli.py)     Gateway (gateway/run.py)    ACP (acp_adapter/)   │
│   Batch Runner     API Server                  Python Library        │
└──────────┬──────────────┬───────────────────────┬───────────────────┘
           │              │                       │
           ▼              ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Agent 内核  AIAgent (run_agent.py, ~13,700 行)                     │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │ Prompt       │  │ Provider     │  │ Tool         │              │
│   │ Builder      │  │ Resolution   │  │ Dispatch     │              │
│   │ (prompt_     │  │ (runtime_    │  │ (model_      │              │
│   │  builder.py) │  │  provider.py)│  │  tools.py)   │              │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│          │                 │                  │                      │
│   ┌──────┴───────┐  ┌──────┴───────┐  ┌───────┴──────┐              │
│   │ Compression  │  │ 3 种 API 模式│  │ Tool Registry│              │
│   │ & Caching    │  │ chat_compl.  │  │ (registry.py)│              │
│   │              │  │ codex_resp.  │  │ 61 tools     │              │
│   │              │  │ anthropic    │  │ 52 toolsets  │              │
│   └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────┬────────────────────┬──────────────────┬───────────────────┘
          │                    │                  │
          ▼                    ▼                  ▼
┌───────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
│ Session 存储      │  │ Provider 厂商    │  │ Tool 后端             │
│ SQLite + FTS5     │  │ OpenAI/Claude/   │  │ Terminal (7 种)       │
│ hermes_state.py   │  │ Gemini/OpenRouter│  │ Browser (5 种)        │
│ gateway/session.py│  │ /Nous/Z.AI/...   │  │ Web, MCP, File, ...   │
└───────────────────┘  └──────────────────┘  └──────────────────────┘
```

### 模块职责对照表

| 层 | 代表模块 | 做什么 |
|---|---|---|
| 入口层 | cli.py / gateway/run.py / acp_adapter/ / batch_runner.py | 把「平台事件」翻译成「一次 AIAgent 调用」 |
| Agent 内核 | run_agent.py (AIAgent) | 组 prompt、选 provider、跑循环、调工具、存 session |
| Prompt 系统 | agent/prompt_builder.py, agent/prompt_caching.py | 从 SOUL.md / MEMORY.md / skills / 项目 context 拼 system prompt |
| 压缩引擎 | agent/context_engine.py (ABC), agent/context_compressor.py | 当上下文超阈值时做结构化摘要 |
| Provider 解析 | hermes_cli/runtime_provider.py, hermes_cli/auth.py, providers/ | `(provider, model) → (api_mode, api_key, base_url)` |
| 工具系统 | tools/registry.py, model_tools.py, toolsets.py | 自注册、schema 收集、dispatch、审批 |
| Session 存储 | hermes_state.py, gateway/session.py | SQLite + FTS5 全文搜索，带 lineage |
| 工具后端 | tools/environments/*, tools/browser_tool.py, tools/mcp_tool.py | 7 种 terminal 后端、5 种浏览器、MCP 动态 |

**一句话总结**：所有入口最终都是 `AIAgent.run_conversation(user_message=...)` 这一个函数。看懂它，等于看懂 Hermes 80%。

---

## B. Agent 主循环：run_conversation 一步一步拆

核心类是 `run_agent.py` 里的 `AIAgent`，它的两个对外入口：

```python
# 简版：返回最终字符串
response = agent.chat("Fix the bug in main.py")

# 完整版：返回 messages / metadata / usage
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,           # 不给就自动构造
    conversation_history=None,     # 不给就从 session 加载
    task_id="task_abc123"
)
```

`chat()` 只是 `run_conversation()` 的薄封装，从返回 dict 里取 `final_response`。

### B.1 一个 turn 的 9 步流水线

```
run_conversation()
  1. 没给 task_id 就生成一个
  2. user message 追加到 conversation history
  3. 构造 or 复用缓存的 system prompt（prompt_builder.py）
  4. preflight 压缩检查（>50% context 就先压再跑）
  5. 按 api_mode 把 history 转成 API messages
     - chat_completions:  OpenAI 原格式
     - codex_responses:   转成 Responses API input items
     - anthropic_messages: 用 anthropic_adapter.py 转
  6. 注入 ephemeral 层（budget 警告、context pressure）
  7. Anthropic 场景打 cache_control 标记
  8. _interruptible_api_call() 发请求（可被打断）
  9. 解析响应：
     - 有 tool_calls → 执行工具 → 把结果 append 回 → 跳回第 5 步
     - 纯文本 → 存 session、flush memory、return
```

### B.2 消息格式（内部统一用 OpenAI 格式）

```python
{"role": "system", "content": "..."}
{"role": "user", "content": "..."}
{"role": "assistant", "content": "...", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}
```

有 reasoning 的模型，其 reasoning 存在 `assistant_msg["reasoning"]`，通过 `reasoning_callback` 展示。

**角色交替硬规则**（provider 会校验）：
- system 之后：user → assistant → user → assistant → ...
- 工具调用期：assistant(tool_calls) → tool → tool → ... → assistant
- 不允许连续两个 assistant，也不允许连续两个 user
- 只有 `tool` 角色可以连续（并行工具返回）

### B.3 三种 API 模式 & 如何选出来

| api_mode | 用于 | 客户端 |
|---|---|---|
| `chat_completions` | OpenAI 兼容端点（OpenRouter、custom、多数 provider） | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` + Responses 格式 |
| `anthropic_messages` | 原生 Anthropic Messages API | `anthropic.Anthropic` + 适配器 |

模式解析优先级：
1. 构造器显式传 `api_mode`（最高）
2. provider 专属探测（provider == "anthropic" → anthropic_messages）
3. base_url 启发式（`api.anthropic.com` → anthropic_messages）
4. 默认：`chat_completions`

### B.4 可打断的 API 调用

`_interruptible_api_call()` 把 HTTP 请求放子线程，主线程等三个事件之一：

```
┌─────────────────────────────────────────────────┐
│  主线程                     API 线程              │
│                                                 │
│   wait on:                   HTTP POST          │
│    - response ready    ─▶    to provider        │
│    - interrupt event                            │
│    - timeout                                    │
└─────────────────────────────────────────────────┘
```

用户发新消息、`/stop`、或收到信号 → 主线程设置 interrupt event → API 线程被「遗弃」（响应丢弃）→ 历史里不会塞半截内容。

### B.5 max_depth / IterationBudget / Fallback

- 默认 `agent.max_turns = 90`；
- 每个 agent 有自己的 budget；subagent 独立，上限是 `delegation.max_iterations`（默认 50）；
- 达到 100% 就停，返回一份「已做的事情」总结；
- 主 model 挂了（429 / 5xx / 401 / 403）按 `fallback_providers` 列表顺序试；
- 401/403 会先试一次 credential refresh 再 fallback；
- 辅助任务（vision、compression、web 抓取、session 搜索）各有自己的 fallback 链，在 `auxiliary.*` 配置。

### B.6 Agent 级工具：不过 registry 的四个

下面这四个 tool 在 `run_agent.py` 里被拦截，不经过 `model_tools.handle_function_call()`：

| Tool | 拦的原因 |
|---|---|
| `todo` | 读写 agent 本地 task 状态 |
| `memory` | 写持久 memory 文件，带字符上限 |
| `session_search` | 走 agent 自己的 session DB |
| `delegate_task` | 产生独立 context 的 subagent |

它们直接改 agent state 然后合成一个 tool result 塞回 history。

### B.7 Checkpoint = 每 turn 结束的 session persist

每 turn 结束后：
1. messages 写进 SQLite（hermes_state.py）；
2. memory 变更 flush 到 `MEMORY.md` / `USER.md`；
3. session 可以被 `/resume` 或 `hermes chat --resume` 拿起来继续。

这就是 Hermes 的 checkpoint 模型 —— 没有显式 checkpoint 命令，每 turn 本身就是一次存档。

---

## C. Prompt 组装：cached 前缀 + ephemeral 覆盖

这是 Hermes 里被设计最仔细的一块，也是**为什么 prompt cache 能省 75% token** 的根本原因。

`agent/prompt_builder.py` 把 prompt 明确切成两部分：

- **cached 系统前缀**（跨 turn 稳定，不变就能命中 cache）；
- **ephemeral API-call-time 覆盖**（临时加，不进 cache）。

### C.1 cached 系统前缀的十层装配顺序

```
 1. Agent Identity      ← ~/.hermes/SOUL.md（没有就用 DEFAULT_AGENT_IDENTITY）
 2. Tool-aware guidance ← memory 用法提示、session_search 提示等
 3. Honcho 静态块        ← 接了 Honcho 的话
 4. Optional system msg ← config/API 里的覆盖
 5. MEMORY 快照          ← frozen，session 开始时冻结
 6. USER profile 快照    ← frozen
 7. Skills index         ← <available_skills> YAML 清单
 8. Context files        ← .hermes.md / AGENTS.md / CLAUDE.md / .cursorrules（优先级一冲突只留一个）
 9. 时间戳 + session id
10. Platform hint        ← "You are a CLI AI Agent" / "You are a Telegram bot" 之类
```

### C.2 装配结果长啥样（片段）

```
# Layer 1: Agent Identity (from ~/.hermes/SOUL.md)
You are Hermes, an AI assistant created by Nous Research. ...

# Layer 5: Frozen MEMORY snapshot
## Persistent Memory
- User prefers Python 3.12, uses pyproject.toml
- Working on project "atlas" in ~/code/atlas

# Layer 7: Skills index
<available_skills>
  software-development:
    - code-review: Structured code review workflow
  research:
    - arxiv: Search and summarize arXiv papers
</available_skills>

# Layer 8: Context files
## AGENTS.md
This is the atlas project. Use pytest for testing. ...

# Layer 9: Timestamp + session
Current time: 2026-03-30T14:30:00-07:00
Session: abc123

# Layer 10: Platform hint
You are a CLI AI Agent. Try not to use markdown ...
```

### C.3 Context files 的优先级（只装第一个）

`build_context_files_prompt()` 里是「first match wins」：

| 优先级 | 文件 | 搜索范围 | 说明 |
|---|---|---|---|
| 1 | `.hermes.md`, `HERMES.md` | CWD 走到 git root | Hermes 原生 |
| 2 | `AGENTS.md` | CWD | 通用 agent 指令文件 |
| 3 | `CLAUDE.md` | CWD | 兼容 Claude Code |
| 4 | `.cursorrules`, `.cursor/rules/*.mdc` | CWD | 兼容 Cursor |

所有 context file 都经过三道处理：
- **安全扫描** —— 检查 prompt injection（不可见 unicode、"ignore previous instructions"、偷 credential 的图案）；
- **截断** —— 上限 20k 字符，70/20 头尾比，中间塞截断标记；
- **YAML frontmatter 剥离** —— `.hermes.md` 的 frontmatter 会被去掉（预留给未来的 config override）。

### C.4 ephemeral 层：不进 cache 的东西

下面几样是**故意**不放到 cached 系统前缀里的：

- `ephemeral_system_prompt`（turn 级临时加的指令）；
- prefill messages；
- gateway 派生的 session context overlay；
- 后续 turn 里 Honcho 注入到 user message 的 recall 内容。

目的：**保持稳定的前缀稳定** —— 改了系统前缀就意味着所有后续请求 cache miss，一次几万 token 白花。

### C.5 SOUL.md 与 context files 的去重

`load_soul_md()` 读 `~/.hermes/SOUL.md` 作为 Layer 1 的身份。如果命中，`build_context_files_prompt(skip_soul=True)` 会阻止它在 Layer 8 再出现一次。

### C.6 什么时候该改代码 vs 改文件

- 想换 assistant 人格 → 改 `SOUL.md`；
- 想加仓库规则 → 改项目 context 文件；
- 想搞可复用操作流 → 加/改 skill；
- **想改所有人 prompt 的装配逻辑** → 改 `agent/prompt_builder.py`，这是产品级改动，不是用户配置。

---

## D. 上下文压缩 + Prompt Cache：两层防线

Hermes 这块是**双层压缩 + Anthropic prompt cache**。两套逻辑独立运行。

### D.1 双压缩系统

```
                     ┌──────────────────────────────┐
  新消息进来          │  Gateway Session Hygiene      │  85% 阈值
  ───────────────►   │  (agent 跑之前，粗估 token)   │  长 session 安全网
                     └──────────────┬────────────────┘
                                    ▼
                     ┌──────────────────────────────┐
                     │  Agent ContextCompressor      │  50% 阈值（可改）
                     │  (在 loop 内，用真实 token)   │  正常上下文管理
                     └──────────────────────────────┘
```

两个阈值故意错开：
- agent 内部 50% 就压是常态；
- gateway 85% 是安全网，防止「隔夜累积」或「agent 没压够」导致下一 turn API 直接失败。

如果两边都设 50%，长 gateway session 每 turn 都压，纯浪费 token。

### D.2 可插拔的 ContextEngine

```yaml
context:
  engine: "compressor"   # 默认 —— 内置有损摘要
  engine: "lcm"          # 示例 —— 插件提供的无损上下文
```

引擎要实现 `agent/context_engine.py` 这个 ABC：
- `should_compress()` —— 要不要压？
- `compress()` —— 压！
- 可选：暴露工具给 agent（比如 `lcm_grep`）；
- 追踪 API 返回的 token 统计。

解析顺序：`plugins/context_engine/<name>/` → 普通插件注册 → 内置 `ContextCompressor`。插件引擎**绝不**会被自动激活，必须在 config 明确写名字。

### D.3 压缩四阶段算法

**Phase 1: 剪老 tool result**（不调 LLM，纯本地）

保护尾之外、>200 字符的老 tool 输出，全部替换成：

```
[Old tool output cleared to save context space]
```

这是个廉价预处理，省的就是「文件全文」「终端大输出」「搜索结果」这些 token 大户。

**Phase 2: 定边界**

```
┌─────────────────────────────────────────────────┐
│  [0..2]    ← protect_first_n（system + 首次交互）│
│  [3..N]    ← 中段 turns → 被摘要                 │
│  [N..end]  ← 尾（按 token 预算 或 protect_last_n）│
└─────────────────────────────────────────────────┘
```

尾保护是 token-budget based：从末尾往前走，累积到预算耗尽停。如果 token 预算保护的条数还没 `protect_last_n` 多，就用后者兜底。

边界对齐规则：**绝不切开 tool_call/tool_result 对**。`_align_boundary_backward()` 会向前走过连续 tool result 找到 parent assistant message，保证整组完整。

**Phase 3: 结构化摘要（调 LLM）**

⚠️ **坑点**：**摘要模型的 context 必须 ≥ 主 agent 模型的 context**，否则整段中间被当作一个 prompt 塞进去就超了，`_generate_summary()` 捕获错误后返回 `None`，压缩器**会直接丢弃中段但不带摘要** —— 这是质量下降最常见的原因。

摘要模板是强结构化的：

```
## Goal / ## Constraints & Preferences / ## Progress
### Done / ### In Progress / ### Blocked
## Key Decisions / ## Relevant Files / ## Next Steps
## Critical Context
```

摘要预算：`content_tokens × 0.20`（`_SUMMARY_RATIO`），下限 2000，上限 `min(context_length × 0.05, 12000)`。

**Phase 4: 组装压缩后的 messages**

```
[ 头 messages（第一次压缩时给 system 加一条 note）]
[ 摘要 message（role 选得避开连续同 role）]
[ 尾 messages（原样）]
```

`_sanitize_tool_pairs()` 清孤儿：tool result 指向已消失的 call → 删；tool call 的 result 被删了 → 塞 stub result。

**迭代再压缩**：下一次压，老摘要作为 prompt 的一部分喂给 LLM，让它「update」而不是「from scratch」—— 条目从 In Progress 搬到 Done，过时的删掉。

### D.4 Anthropic Prompt Caching

`agent/prompt_caching.py`。多轮对话输入 token 成本降约 75%。

**策略：system_and_3**

Anthropic 每请求最多 4 个 `cache_control` 断点，Hermes 这么用：

```
Breakpoint 1: System prompt                        （跨 turn 都稳）
Breakpoint 2: 倒数第 3 个非 system message     ─┐
Breakpoint 3: 倒数第 2 个非 system message      ├─ 滚动窗口
Breakpoint 4: 最后一个非 system message        ─┘
```

**marker 在哪打取决于 content 类型**：

| Content 类型 | marker 位置 |
|---|---|
| string | 转成 `[{"type":"text","text":...,"cache_control":...}]` |
| list | 加到最后一个元素的 dict |
| None/空 | `msg["cache_control"]` |
| tool message | `msg["cache_control"]`（仅原生 Anthropic） |

**cache 命中的三条设计铁律**：

1. system prompt 要稳 —— breakpoint 1，跨 turn 能活；压缩只在**第一次**追加 note。
2. 消息顺序不能动 —— cache 靠前缀匹配，中间动一条后面全 miss。
3. 压缩后 cache：中段失效，system 存活，滚动 3 条窗口 1–2 个 turn 就重新建立。

TTL 选 `5m` 或 `1h`（长耗 session 选 1h）：

```yaml
prompt_caching:
  cache_ttl: "5m"
```

### D.5 没有 pressure warning

旧版本有「快到阈值了」的中间提示，现在删了 —— 原因是 **模型看到警告会提前放弃复杂任务**。现在就两档：50% 压、85% 安全网，不中间提醒。

---

## E. Provider 适配层：20+ 厂商怎么统一

Hermes 必须同时接 OpenAI / Anthropic / Gemini / OpenRouter / Nous / Z.AI / Kimi / MiniMax / DeepSeek / Copilot / Codex / Vercel AI Gateway / Alibaba DashScope / Hugging Face / 各种 custom ... 它的解法是**一个共享 runtime resolver + 每厂商一个 plugin profile**。

### E.1 核心文件

| 文件 | 职责 |
|---|---|
| `hermes_cli/runtime_provider.py` | credential 解析、`_resolve_custom_runtime()` |
| `hermes_cli/auth.py` | provider registry、`resolve_provider()` |
| `hermes_cli/model_switch.py` | `/model` 切换的共享 pipeline（CLI + gateway） |
| `agent/auxiliary_client.py` | 辅助模型路由 |
| `providers/` | ABC + 注册入口（`ProviderProfile`、`register_provider`、`get_provider_profile`、`list_providers`） |
| `plugins/model-providers/<name>/` | 每厂商一个 plugin，声明 `api_mode` / `base_url` / `env_vars` / `fallback_models` |
| `$HERMES_HOME/plugins/model-providers/<name>/` | 用户级同名覆盖 |

### E.2 解析优先级

```
1. 显式 CLI/runtime 请求        （最高）
2. config.yaml 里 model/provider 配置
3. 环境变量
4. provider 默认 / auto          （最低）
```

**为什么 config 高于 env**：防止一个陈年 `export` 偷偷覆盖用户上次在 `hermes model` 里选的端点。

### E.3 Resolver 返回结构

调一次 resolver 你拿到：

- `provider`
- `api_mode`
- `base_url`
- `api_key`
- `source`（说明这个 key 是哪来的：env / config / credential file）
- 厂商专属 metadata（expiry、refresh info 等）

### E.4 三条非同路径

**原生 Anthropic（不是 via OpenRouter）**：

- `api_mode = anthropic_messages`
- 用 `anthropic.Anthropic` 客户端
- `agent/anthropic_adapter.py` 翻译 message 格式
- credential 优先级：可刷新的 Claude Code credential file > `ANTHROPIC_TOKEN` / `CLAUDE_CODE_OAUTH_TOKEN` env
- 调用前 preflight 一次 refresh；401 后还会重建 client 重试一次。

**OpenAI Codex**：

- `api_mode = codex_responses`
- 独立的 credential 解析和 auth store。

**Custom OpenAI-compatible**：

- 关键点是**api key 按 base_url 隔离**，防泄漏：
  - `OPENROUTER_API_KEY` 只发到 `openrouter.ai`
  - `AI_GATEWAY_API_KEY` 只发到 `ai-gateway.vercel.sh`
  - `OPENAI_API_KEY` 给 custom 端点兜底
- 区分「用户真的配了 custom」和「只是 fallback 到 OpenRouter」，后者不会污染前者。

### E.5 加一个新厂商不用改 resolver

加 plugin 就行：

```
plugins/model-providers/my-provider/
    __init__.py       # 里面调 register_provider(ProviderProfile(
                      #   id="my-provider",
                      #   api_mode="chat_completions",
                      #   base_url="https://api.my-provider.com/v1",
                      #   env_vars=["MY_PROVIDER_API_KEY", "OPENAI_API_KEY"],
                      #   fallback_models=[...]
                      # ))
```

`get_provider_profile("my-provider")` 首次访问时触发 plugin 注册。resolver 里一行 `if` 都不用加。详细步骤见 `developer-guide/adding-providers.md`。

### E.6 Function call 格式差异全部在 adapter 里吃掉

三种 api_mode 的 tool call 长得完全不一样，但**内部统一 OpenAI 风格的 `role/content/tool_calls` dict**。每个 api_mode 在「进 API」和「出 API」两侧各做一次翻译。只有 `run_agent.py` + 三个 adapter 关心格式差异，别的模块（prompt_builder / tools / session）都不用知道。

### E.7 Fallback models

- `AIAgent.__init__` 里存 `fallback_model` dict，`_fallback_activated = False`；
- `_try_activate_fallback()` 在三处被触发：
  1. 无效响应重试超上限；
  2. 非重试性客户端错（401/403/404）；
  3. 瞬时错误（429/500/502/503）重试超上限；
- 激活流程：`resolve_provider_client()` 建新 client → 根据 provider 确定 api_mode（codex-openai → codex_responses；anthropic → anthropic_messages；其他 → chat_completions）→ in-place 换 `self.model/provider/base_url/api_mode/client` → 重评 prompt cache 是否可用 → `_fallback_activated = True`（只打一枪）→ retry count 归零继续 loop。
- **不支持 fallback 的地方**：subagent delegation、辅助任务（各有独立链）；
- **支持 fallback 的地方**：CLI、gateway、cron（cron 读 `fallback_providers` 或老的 `fallback_model` key）。

---

## F. 工具运行时：自注册 + toolset 分组 + 审批流

### F.1 自注册模型

每个 `tools/*.py` 在 **import 时** 调 `registry.register(...)`。签名：

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema={...},                   # OpenAI function-calling schema
    handler=handle_terminal,
    check_fn=check_terminal,        # 可选：可用性判断
    requires_env=["SOME_VAR"],      # 可选：UI 展示用
    is_async=False,
    description="Run commands",
    emoji="💻",
)
```

每次 register 产生一个 `ToolEntry`，存到单例 `ToolRegistry._tools`（key = tool name）。名字撞车了 warn 一下，后者赢。

### F.2 Discovery：AST 扫 + import

`discover_builtin_tools()` 扫 `tools/*.py`，用 AST 判断哪些文件含**顶层** `registry.register()` 调用（不匹配函数内的调用），然后 `importlib.import_module`。好处：加新 tool 文件**不用改任何清单**。

核心逻辑（简化）：

```python
def discover_builtin_tools(tools_dir=None):
    tools_path = Path(tools_dir) if tools_dir else Path(__file__).parent
    for path in sorted(tools_path.glob("*.py")):
        if path.name in {"__init__.py", "registry.py", "mcp_tool.py"}:
            continue
        if _module_registers_tools(path):       # AST 检查
            importlib.import_module(f"tools.{path.stem}")
```

某个可选 tool import 挂了（比如缺 `fal_client`）会被 catch 并记 log，不影响其他 tool。

核心 discovery 之后还会做：
1. **MCP 工具** —— `tools.mcp_tool.discover_mcp_tools()` 读 MCP server 配置，注册外部 server 暴露的 tool；
2. **Plugin 工具** —— `hermes_cli.plugins.discover_plugins()` 加载用户/项目/pip plugin。

### F.3 check_fn：可用性 gate

每个 tool 可以带一个 `check_fn`，返回 True 就放出来，False 就藏掉。典型用法：

- API key 在不在：`lambda: bool(os.environ.get("SERP_API_KEY"))`；
- 服务在不在（Honcho 配了没）；
- 二进制装了没（playwright 在不在）。

`registry.get_definitions()` 在拼 schema 列表时跑 `check_fn()`：

```python
if entry.check_fn:
    try:
        available = bool(entry.check_fn())
    except Exception:
        available = False        # 异常 = 不可用，fail-safe
    if not available:
        continue                  # 这 tool 压根不出现在 schema 里
```

结果**同一 call 内 cache**（多个 tool 共享 check_fn 时只跑一次）。

### F.4 Toolset：tool 的分组

Toolset 是 tool 的命名 bundle。解析方式：

- 显式 enabled / disabled 列表；
- 平台预设（`hermes-cli`、`hermes-telegram`、`hermes-discord` 等）；
- 动态 MCP toolset；
- 专用组，比如 `hermes-acp`。

`model_tools.get_tool_definitions(enabled, disabled, quiet_mode)` 的过滤规则：

1. 指定了 enabled → 只收那些 toolset 里的 tool；
2. 指定了 disabled → 从全集里减；
3. 都没指定 → 全部；
4. 交给 `registry.get_definitions()` 做 `check_fn` 过滤；
5. **动态 schema patch** —— `execute_code` 和 `browser_navigate` 的 schema 会被现场改写，只引用实际通过过滤的 tool 名，防止模型幻觉出不存在的子工具。

老名字（`web_tools`、`terminal_tools`）通过 `_LEGACY_TOOLSET_MAP` 兼容到新名字。

### F.5 Dispatch 流程

```
Model 响应里有 tool_call
    ↓
run_agent.py 主循环
    ↓
model_tools.handle_function_call(name, args, task_id, user_task)
    ↓
[ Agent-loop tool？ ]  yes → agent 直接处理（todo/memory/session_search/delegate_task）
    ↓ no
[ Plugin pre-hook ]   invoke_hook("pre_tool_call", ...)
    ↓
registry.dispatch(name, args, **kwargs)
    ↓
按 name 查 ToolEntry
    ↓
[ async handler？] → _run_async() 桥接
[ sync handler ]   → 直接调
    ↓
返回 result 字符串（或 JSON error）
    ↓
[ Plugin post-hook ]  invoke_hook("post_tool_call", ...)
```

**两层错误包装**：
- `registry.dispatch()` catch handler 的任何异常，返回 `{"error": "Tool execution failed: ..."}`；
- `handle_function_call()` 再套一层 try/except，返回 `{"error": "Error executing tool_name: ..."}`。

保证模型**永远**收到合法 JSON，不会被未处理异常打断对话。

### F.6 Async 桥接的三条路径

`_run_async()` 根据上下文选不同策略：

- **CLI（没 running loop）** —— 用一个持久事件循环，让缓存的 async client 活着；
- **Gateway（有 running loop）** —— 临时线程 + `asyncio.run()`；
- **worker 线程（并行 tool）** —— thread-local 的持久 loop。

### F.7 审批流：DANGEROUS_PATTERNS

`tools/approval.py` 是危险命令审批系统，主要给 `terminal_tool` 用。

1. **Pattern 检测** —— `DANGEROUS_PATTERNS` 是 `(regex, description)` 列表：
   - 递归删除（`rm -rf`）
   - 格式化（`mkfs`、`dd`）
   - SQL 破坏（`DROP TABLE`、无 `WHERE` 的 `DELETE FROM`）
   - 系统配置覆写（`> /etc/`）
   - 服务操作（`systemctl stop`）
   - 远程执行（`curl | sh`）
   - fork bomb、process kill...

2. **调用前检测** —— `detect_dangerous_command(command)` 跑一遍所有 pattern。

3. **审批提示**：
   - **CLI 模式** —— 交互提示：approve / deny / allow permanently；
   - **Gateway 模式** —— 异步 callback 发到消息平台让用户点；
   - **Smart approval**（可选）—— 辅助 LLM 给匹配 pattern 但实际低危的命令自动放行（比如 `rm -rf node_modules/` 匹配「递归删除」但明显安全）；

4. **Session 级状态** —— 批过「递归删除」后，同 session 里后续 `rm -rf` 不再重新问；

5. **永久白名单** —— 「allow permanently」写 `config.yaml` 的 `command_allowlist`，跨 session 生效。

### F.8 并发

当 model 一次给多个 tool_call：
- **单个** —— 主线程直接跑；
- **多个** —— `ThreadPoolExecutor` 并行；
  - 例外：标记为 interactive 的 tool（比如 `clarify`）强制串行；
  - 结果按 tool_call 原顺序回插，和完成顺序无关。

### F.9 结果回注

每个 tool 跑完，history 上 append：

```python
{"role": "tool", "tool_call_id": "...", "content": result_string}
```

然后跳回主循环第 5 步（「构造 API messages」），新一轮 request 带上这些 tool result 给 model 看。

---

## 过渡：下半部分在讲什么

读到这里，你应该能回答下面这些问题：
- Hermes 各个入口最终都汇聚到哪里？（`AIAgent.run_conversation`）
- 一个 turn 跑了哪 9 步？
- system prompt 10 层装配是哪 10 层？哪些进 cache，哪些不进？
- 什么时候压缩、压缩的 4 阶段分别在干嘛？
- 20+ 厂商怎么统一？加新 provider 要动哪些文件？
- tool 是怎么 auto-discover、怎么 dispatch 的？危险命令怎么审批？

但还有几块没讲：
- **Tool 后端**：terminal 的 7 种 backend（local / docker / ssh / modal / daytona / singularity / vercel sandbox）长啥样、怎么接第 8 种；
- **Session 存储**：SQLite schema、FTS5 搜索怎么建索引、session lineage 怎么跟踪；
- **Gateway 内部**：20 个平台 adapter、hook 系统、cross-session mirror、pairing 授权；
- **二次开发**：怎么把 Hermes 当库嵌进自己的 Python 项目？怎么写 plugin？怎么写 memory provider / context engine？

这些在 **part-3b-环境存储与扩展.md** 里继续讲。

---

## 导航表

| 章节 | 位置 | 关键文件 |
|---|---|---|
| A 总体架构 | 本文 | `run_agent.py`, 全仓库顶层 |
| B Agent 主循环 | 本文 | `run_agent.py`（`AIAgent`、`_interruptible_api_call`、`_try_activate_fallback`） |
| C Prompt 装配 | 本文 | `agent/prompt_builder.py`, `~/.hermes/SOUL.md` |
| D 压缩 + Cache | 本文 | `agent/context_engine.py`, `agent/context_compressor.py`, `agent/prompt_caching.py`, `gateway/run.py`（session hygiene） |
| E Provider 层 | 本文 | `hermes_cli/runtime_provider.py`, `hermes_cli/auth.py`, `providers/`, `plugins/model-providers/` |
| F 工具运行时 | 本文 | `tools/registry.py`, `model_tools.py`, `toolsets.py`, `tools/approval.py` |
| G 工具后端 | part-3b | `tools/environments/*`, `tools/terminal_tool.py` |
| H Session 存储 | part-3b | `hermes_state.py`, `gateway/session.py` |
| I Gateway 内部 | part-3b | `gateway/run.py`, `gateway/platforms/*`, `gateway/hooks.py` |
| J 二次开发 | part-3b | `plugins/`, 作为 Python library 嵌入 |
| 附录 | appendix-最佳应用场景与速查表.md | 12 场景 + 速查表（CLI/斜杠/配置/环境变量） |

> 读完上半部分你已经能改 Hermes 的核心行为了。下半部分主要是**扩展能力** —— 给它加新能力、新后端、新平台。
