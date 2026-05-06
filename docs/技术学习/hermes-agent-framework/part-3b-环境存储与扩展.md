---
来源: 基于 Hermes Agent v0.11.0 源码 + 官方 developer-guide 文档
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, developer, environment, storage, gateway, plugin, nous-research]
Ground Truth: ✅ 基于 environments/, hermes_state.py, gateway/, tools/registry.py, hermes_cli/ 等子系统源码
主题: Hermes Agent 源码深入（下半篇）—— 环境、存储与扩展
系列: 三段式教程 Part 3B / 3
前置: part-3a-核心执行路径.md
后续: appendix-场景速查.md
---

# Part 3B：环境、存储与扩展

> 上半篇（3A）把 CLI、Agent Loop、Tool Registry、Provider Adapter 这条主线讲完了。
> 下半篇聚焦"周边基础设施"：隔离执行的 Environment、长期记忆用的 SQLite、连接外部世界的 Gateway，
> 以及最重要的 —— 你自己在不改 Hermes 源码的前提下，怎么往里加东西。

---

## G. 五种 Environment 后端：隔离执行的选择

Hermes 的 terminal / file / python_exec 这些"会改动真实文件系统"的工具，背后都走一个统一抽象：`Environment`。
通过 `TERMINAL_ENV` 环境变量或 `--env` 参数选择后端，同一份工具代码可以跑在本机、容器、远端主机、Modal 云上。

### G.1 五种后端一览

| 后端 | 隔离级别 | 启动速度 | 成本 | 典型用途 |
|---|---|---|---|---|
| `local` | 无（直接跑在你本机） | 毫秒级 | 免费 | 信任 agent 的日常开发、小脚本 |
| `docker` | 容器级（cap-drop、PID 限制、无特权升级） | 秒级 | 免费（本机资源） | 跑陌生代码、批量测试、sandbox 保护 |
| `ssh` | 远端主机 | 看网络 | 看你自己那台机器 | 把 agent 派到 GPU 服务器或 staging 环境做事 |
| `singularity` | HPC 容器 | 秒级 | 免费 | 大学/科研集群里跑，没有 root 也能用 |
| `modal` | 云 sandbox（每任务独立镜像） | 3–10 秒 | 按秒计费 | 大规模 benchmark（TB2、TBLite）、并行 rollout |

选择口诀：**本地开发 local，不信任代码 docker，跑 bench modal，在 HPC singularity，需要特定机器 ssh**。

### G.2 后端切换

三种方式，优先级从高到低：

```bash
# 1) CLI 参数（最高优先级）
hermes chat --env docker

# 2) 环境变量
export TERMINAL_ENV=modal

# 3) config.yaml
# environment: docker
```

在 Environment 训练/评测的 YAML 里，字段名是 `terminal_backend`：

```yaml
env:
  terminal_backend: "modal"
  terminal_timeout: 300      # 单条命令超时秒数
  terminal_lifetime: 3600    # 沙盒总寿命
```

### G.3 Docker 后端的硬化细节

Hermes 的 docker 后端默认启动参数等价于：

```
docker run --rm \
  --cap-drop=ALL --security-opt=no-new-privileges \
  --pids-limit=512 \
  -v <work_dir>:/workspace \
  <image>
```

所有 capability 都被 drop，不能提权，PID 被限制，防止 fork bomb。这是官方 contributing.md 里明确列出的"容器硬化"条款。

### G.4 Environment（RL/Eval 训练框架）

注意区分两个"Environment"：

- **Terminal Environment**：刚才讲的五种后端，用户工具执行时的沙盒。
- **Atropos Environment**（`environments/` 目录）：把 hermes-agent 接到 Nous 的 RL 训练框架 Atropos 上，给 TB2、TBLite、YC-Bench 这类 benchmark 用的。

继承链是 `BaseEnv ← HermesAgentBaseEnv ← 具体 Env（TerminalTestEnv / HermesSweEnv / TBLiteEvalEnv...）`。
每个具体 Env 实现五个方法：`setup / get_next_item / format_prompt / compute_reward / evaluate`。

三种运行模式：

| 子命令 | 用途 |
|---|---|
| `evaluate` | 跑 benchmark，出指标 |
| `process` | 生成 SFT 数据（打分 trajectory 存 JSONL） |
| `serve` | 连接 Atropos API 做 RL 训练 |

具体使用细节不在本文范围，想做 RL/Eval 的看官方 `developer-guide/environments.md`。

---

## H. Session 存储：SQLite + FTS5

### H.1 为什么是 SQLite 而不是 JSONL

早期 Hermes 每个 session 存一份 JSONL。问题来了：gateway 开着 14 个平台同时写、CLI 和 gateway 还要互相 `/resume` 彼此的会话、全文搜索要扫所有文件 —— JSONL 撑不住。

v0.10+ 起切到 SQLite，单文件 `~/.hermes/state.db`：

- **WAL 模式**：一个写入者 + 多个读取者并发，gateway 多平台不互相卡。
- **FTS5 全文索引**：`/search docker build` 在几万条消息里 10ms 出结果。
- **Session lineage**：用 `parent_session_id` 链起"压缩后分裂的新 session"，保留历史。
- **来源 tag**：每条 session 有 `source=cli|telegram|discord|...`，过滤方便。

### H.2 核心 Schema

两张主表 + 两张 FTS 虚表：

```sql
sessions(
  id TEXT PK, source TEXT, user_id, model, model_config,
  system_prompt, parent_session_id,  -- 关键：会话血统
  started_at, ended_at, end_reason,
  message_count, tool_call_count, api_call_count,
  input_tokens, output_tokens, cache_read_tokens, cache_write_tokens,
  reasoning_tokens, estimated_cost_usd, actual_cost_usd,
  title   -- 唯一索引（允许多个 NULL）
);

messages(
  id PK, session_id FK, role, content,
  tool_call_id, tool_calls (JSON), tool_name,
  timestamp, token_count, finish_reason,
  reasoning, reasoning_content, reasoning_details (JSON),
  codex_reasoning_items, codex_message_items
);

messages_fts           -- FTS5 默认 tokenizer
messages_fts_trigram   -- trigram，专门处理 CJK/子串搜索
```

`messages` 插入/更新/删除都由三个 trigger 同步到 FTS 虚表，**你不需要手动维护索引**。

schema 版本当前是 11，迁移链用 `schema_version` 单行表跟踪。新加列走"声明式"`_reconcile_columns()` 自动对齐；涉及数据搬运或索引重建的才上版本号。

### H.3 写入争用的处理

多个 hermes 进程共用一个 `state.db`（gateway 进程 + 两个 CLI 会话 + 若干 worktree agent），SQLite 默认锁策略会产生 convoy effect。Hermes 的做法：

- SQLite 锁超时缩到 **1 秒**（默认 30 秒）
- 应用层重试最多 15 次，**20–150ms 随机 jitter**
- `BEGIN IMMEDIATE` 立刻暴露冲突
- 每 50 次成功写入做一次 PASSIVE checkpoint

这些常量都在 `hermes_state.py` 顶部：`_WRITE_MAX_RETRIES=15`, `_WRITE_RETRY_MIN_S=0.020`, `_WRITE_RETRY_MAX_S=0.150`, `_CHECKPOINT_EVERY_N_WRITES=50`。

### H.4 全文搜索

```python
db.search_messages("docker deployment")                    # 隐式 AND
db.search_messages('"exact phrase"')                       # 短语
db.search_messages("docker OR kubernetes")                 # OR
db.search_messages("python NOT java")                      # 排除
db.search_messages("deploy*")                              # 前缀
db.search_messages("error", source_filter=["cli"])         # 只 CLI 会话
db.search_messages("bug", exclude_sources=["telegram"])    # 排除 telegram
```

返回每条结果带 `snippet`（FTS5 自带 `>>>match<<<` 高亮）和上下文（前后各一条消息，截断到 200 字）。

`_sanitize_fts5_query()` 会自动包裹带连字符的词（`chat-send` → `"chat-send"`），去掉落单的布尔操作符（`hello AND` → `hello`），别担心用户输错 query 会 crash。

### H.5 导入 / 导出 / 清理

```python
data = db.export_session("sess_abc123")
all_data = db.export_all(source="cli")
db.prune_sessions(older_than_days=90)               # 只删已结束的
db.prune_sessions(older_than_days=30, source="telegram")
db.clear_messages("sess_abc123")                    # 清消息留壳
db.delete_session("sess_abc123")
```

trajectory 格式就是 OpenAI chat conversation 的超集，附加 tool_calls / reasoning / token_count 等列。导出后可以直接喂给训练脚本或迁移到别的工具。

---

## I. Gateway 架构：把 14 个消息平台统一成一个接口

### I.1 组件清单

`gateway/` 下的核心文件：

| 文件 | 职责 |
|---|---|
| `run.py` | `GatewayRunner`：主循环、slash 命令、消息分发（~12k 行） |
| `session.py` | `SessionStore`：session key 构建、会话持久化 |
| `delivery.py` | 出站消息投递（直接回复 / home channel / 跨平台） |
| `pairing.py` | DM 配对流程（管理员授权新用户） |
| `hooks.py` | 生命周期 hook 的发现和调度 |
| `mirror.py` | `send_message` 工具的跨会话镜像 |
| `status.py` | 多 profile 场景下的 token lock |
| `platforms/` | 每个平台一个 adapter（telegram/discord/slack/...） |

### I.2 消息流

一条 Telegram 消息从 app 里发出，到 agent 开始思考，要穿过这几层：

```
Telegram Bot API → TelegramAdapter.on_message()
  → 规范化成 MessageEvent
  → BaseAdapter 检查 _active_sessions（Level 1 guard）
     · agent 还在跑？→ 塞进 _pending_messages 队列 + 置 interrupt event
     · /approve /deny /stop → 放行
  → GatewayRunner._handle_message()
     · _session_key_for_source() 算 session key
     · 授权检查
     · 是 slash 命令？→ 命令 handler
     · Level 2 guard：_running_agents 拦截 /stop /new /queue /status
     · 都不是？→ 创建 AIAgent 跑对话
  → SessionStore 持久化消息（走 hermes_state.py）
  → 响应走 adapter.send() 回 Telegram
```

**两级 guard** 是故意设计的：Level 1 在 adapter 层拦住普通消息，Level 2 在 runner 层保留让 `/approve` 这类命令仍能进入内部状态。`/approve` 用 `await self._message_handler(event)` **inline 派发**，绕开后台任务系统，避免竞态。

### I.3 Session Key 格式

永远长这样：

```
agent:main:{platform}:{chat_type}:{chat_id}
```

比如 `agent:main:telegram:private:123456789`。

对于支持线程的平台（Telegram 论坛主题、Discord threads、Slack threads），chat_id 里会编入 thread ID。**永远别手搓 session key**，用 `build_session_key()`。这样保证 CLI/gateway/cron 各路代码对同一会话达成共识。

### I.4 授权链

按顺序检查，任一通过即放行：

1. 平台级允许所有用户（`TELEGRAM_ALLOW_ALL_USERS=1`）
2. 平台级白名单（`TELEGRAM_ALLOWED_USERS=id1,id2`）
3. DM pairing（已授权用户 `/pair` 生成码，新用户发码完成配对）
4. 全局 `GATEWAY_ALLOW_ALL_USERS`
5. 默认拒绝

配对状态持久化在 `gateway/pairing.py` 里，重启也在。

### I.5 Hook 事件

gateway 生命周期可以挂 hook（Python 模块，放在 `~/.hermes/hooks/<name>/`）：

| 事件 | 触发时机 |
|---|---|
| `gateway:startup` | gateway 进程启动 |
| `session:start / end / reset` | 会话开启 / 结束 / `/new` 重置 |
| `agent:start / step / end` | agent 开始 / 每步工具调用 / 结束 |
| `command:*` | 任何 slash 命令执行 |

每个 hook 是一个目录，包含 `HOOK.yaml` 清单 + `handler.py`。这是给你做"每条消息发企业微信通知运维"或"agent 结束后自动 commit git"这类旁路逻辑用的。

---

## J. 二次开发入口：四种扩展

这是本篇最实用的部分。Hermes 刻意做成高度可扩展，**大多数场景不需要 fork 源码**，而是放插件目录。

### J.1 加新 tool（两种路径）

**路径 A：Plugin（推荐）—— 不改 Hermes 源码**

放一个目录到 `~/.hermes/plugins/my-tool/`，带一个 `PLUGIN.yaml` 清单和 `__init__.py` 里调 `register_tool()`。启动时自动发现，没踩到任何核心代码。

**路径 B：内置 tool（只在给 Hermes 上游贡献时用）**

触 2 个文件：

1. `tools/your_tool.py` — handler + schema + `check_fn` + 顶层 `registry.register()` 调用
2. `toolsets.py` — 把 tool 名加到 `_HERMES_CORE_TOOLS` 或新 toolset

骨架：

```python
# tools/weather_tool.py
import json, os
from tools.registry import registry

def check_weather_requirements() -> bool:
    return bool(os.getenv("WEATHER_API_KEY"))

def weather_tool(location: str, units: str = "metric") -> str:
    try:
        # ...调 API...
        return json.dumps({"location": location, "temp": 22, "units": units})
    except Exception as e:
        return json.dumps({"error": str(e)})

WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather for a location.",
    "parameters": {
        "type": "object",
        "properties": {
            "location": {"type": "string"},
            "units": {"type": "string", "enum": ["metric", "imperial"], "default": "metric"},
        },
        "required": ["location"],
    },
}

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool(args.get("location", ""), args.get("units", "metric")),
    check_fn=check_weather_requirements,
    requires_env=["WEATHER_API_KEY"],
)
```

**三条硬规定**：

- Handler **必须返回 JSON 字符串**（用 `json.dumps()`），不能返回 dict
- 错误返回 `{"error": "..."}`，**不要 raise**
- `check_fn=False` 时工具在 schema 组装阶段被静默剔除（LLM 根本看不到）

异步 handler 加 `is_async=True`，registry 会自己桥接，你不用写 `asyncio.run()`。需要 `task_id` 做多会话状态隔离的，从 `**kwargs` 里取。

### J.2 加新 provider

优先走 **plugin 路径**：`plugins/model-providers/<name>/` 放 `plugin.yaml` + `__init__.py`，顶层调 `register_provider(profile)`。简单 API-key provider 这样就够了 —— `auth.py` / `models.py` / `runtime_provider.py` / `main.py` 全部自动接线。

只有以下情况才走全量 built-in 路径（要改 8+ 个文件）：

- OAuth / token 刷新（Nous Portal、Codex、Gemini、Qwen Portal、Copilot）
- 非 OpenAI 协议（Anthropic Messages、Codex Responses）需要新 adapter
- 需要 endpoint 探测或多区域探活（z.ai、Kimi）
- 需要在 `hermes model` 菜单里展示专门的 UX

全量路径的关键词：`PROVIDER_REGISTRY`（auth.py）→ `_PROVIDER_MODELS/_PROVIDER_LABELS/_PROVIDER_ALIASES`（models.py）→ `resolve_runtime_provider()` 加分支 → `select_provider_and_model()` / `_model_flow_xxx()` → `_API_KEY_PROVIDER_AUX_MODELS`（auxiliary_client.py）→ `model_metadata.py` 加 context length → 如果是 native 协议还要 `agent/<provider>_adapter.py` 加上 `run_agent.py` 里每个 `api_mode` 分支审一遍。

**一个核心抽象：`api_mode`**。三种已存在的值：`chat_completions`（绝大多数）、`anthropic_messages`、`codex_responses`。引入新协议 = 加新 `api_mode`。

### J.3 加新平台 adapter（消息收发）

同样 **plugin 路径**优先。目录 `~/.hermes/plugins/my-platform/`，放：

```yaml
# PLUGIN.yaml
name: my-platform
version: 1.0.0
requires_env: [MY_PLATFORM_TOKEN]
```

```python
# adapter.py
from gateway.platforms.base import BasePlatformAdapter, SendResult, MessageEvent, MessageType
from gateway.config import Platform, PlatformConfig

class MyPlatformAdapter(BasePlatformAdapter):
    def __init__(self, config):
        super().__init__(config, Platform("my_platform"))
    async def connect(self) -> bool: self._mark_connected(); return True
    async def disconnect(self): self._mark_disconnected()
    async def send(self, chat_id, content, reply_to=None, metadata=None):
        return SendResult(success=True, message_id="...")
    async def get_chat_info(self, chat_id):
        return {"name": chat_id, "type": "dm"}

def register(ctx):
    ctx.register_platform(
        name="my_platform", label="My Platform",
        adapter_factory=lambda cfg: MyPlatformAdapter(cfg),
        check_fn=lambda: bool(os.getenv("MY_PLATFORM_TOKEN")),
        required_env=["MY_PLATFORM_TOKEN"],
        allowed_users_env="MY_PLATFORM_ALLOWED_USERS",
        allow_all_env="MY_PLATFORM_ALLOW_ALL_USERS",
        max_message_length=4000,
        platform_hint="You are chatting via My Platform...",
        emoji="💬",
    )
```

调 `ctx.register_platform()` 自动接线的 16 个点，官方文档里列表形式给得很清楚：gateway adapter 创建、config 解析、授权、cron 投递、send_message 工具、webhook 跨平台、`/update` 访问、频道目录、系统 prompt 提示、分片、PII redaction、`hermes status` 显示、`hermes gateway setup` 菜单、`hermes tools/skills` 每平台配置、token lock、孤儿配置告警 —— 这些你全都**不用手写**。

built-in 路径要改 20+ 文件，只有核心贡献者为官方支持的平台才走。

入站消息的写法（三种典型模式）：

- **长轮询**（Telegram / Weixin）：`asyncio.create_task(self._poll_loop())`
- **webhook/callback**（WeCom、Home Assistant）：起 `aiohttp.web.Application()` 接 POST
- **WebSocket**（Slack Socket Mode、DingTalk、Feishu）：SDK 回调里 forward 到 `self.handle_message(event)`

对响应时效要求苛刻的（如 WeCom 5 秒回调限制），务必立即 ACK，然后异步用 API 主动推回复 —— agent 一轮动辄 3–30 分钟，不可能 inline。

### J.4 扩展 CLI（TUI 定制）

不想 fork `cli.py` 那一千多行 `run()`？继承 `HermesCLI`，override 五个扩展点：

| Hook | 干嘛 |
|---|---|
| `_get_extra_tui_widgets()` | 插入持久 UI 元素（状态面板、mini-player） |
| `_register_extra_tui_keybindings(kb, *, input_area)` | 加快捷键 |
| `_build_tui_layout_children(**widgets)` | 完全重排布局（慎用） |
| `process_command(cmd)` | 加 `/mycommand` |
| `_build_tui_style_dict()` | 自定义 prompt_toolkit 样式 |

一个 F2 切面板的最小例子：

```python
from cli import HermesCLI
from prompt_toolkit.layout import Window, FormattedTextControl
from prompt_toolkit.filters import Condition

class MyCLI(HermesCLI):
    def __init__(self, **kw):
        super().__init__(**kw)
        self._panel_visible = False

    def _get_extra_tui_widgets(self):
        return [Window(
            FormattedTextControl(lambda: "📊 My custom panel"),
            height=1,
            filter=Condition(lambda: self._panel_visible),
        )]

    def _register_extra_tui_keybindings(self, kb, *, input_area):
        @kb.add("f2")
        def _(event): self._panel_visible = not self._panel_visible

    def process_command(self, cmd):
        if cmd.strip() == "/panel":
            self._panel_visible = not self._panel_visible
            return True
        return super().process_command(cmd)

if __name__ == "__main__":
    MyCLI().run()
```

避开和内置键绑定冲突：`Enter`（提交）、`Esc+Enter`（换行）、`Ctrl-C`（打断）、`Ctrl-D`（退出）、`Tab`（接受建议）。F2+ 和 Ctrl-组合基本安全。状态变了记得 `self._invalidate()` 触发重绘。

---

## K. 调试技巧

Agent 框架调 bug 痛点是"黑盒 + 异步 + 多进程"。以下是实测有效的招数：

**K.1 开 debug 日志**

```bash
export HERMES_DEBUG=1
hermes chat -q "..."
```

会打开 verbose 日志，LLM 请求原文、tool 参数、retry 次数全都可见。日志写到 `~/.hermes/logs/hermes.log`（按大小 rotate）。

**K.2 看 trajectory 复盘**

任何一个会话都能从 SQLite 里整段导出：

```python
from hermes_state import SessionDB
db = SessionDB()
for m in db.get_messages("sess_abc123"):
    print(m["role"], "|", (m.get("content") or "")[:200])
    if m.get("tool_calls"):
        print("  tool_calls:", m["tool_calls"])
    if m.get("reasoning"):
        print("  reasoning:", m["reasoning"][:200])
```

或直接 `sqlite3 ~/.hermes/state.db` 手动查。`messages_fts_trigram` 对中文和子串搜索特别好使。

**K.3 打印 provider raw request**

改 `run_agent.py` 在 `_build_api_kwargs()` 返回前加一行：

```python
if os.getenv("HERMES_DEBUG_RAW"):
    logger.warning("RAW REQUEST: %s", json.dumps(kwargs, default=str)[:2000])
```

然后 `export HERMES_DEBUG_RAW=1`。这招能一眼看出"tool schema 有没有发过去"、"system prompt 被谁动了"、"reasoning_effort 传没传"。

**K.4 `hermes doctor`**

诊断命令，检查 venv、Python 版本、各 provider key、sqlite、`~/.hermes` 目录权限、browser binary 等，比自己肉眼查 `.env` 快得多。

**K.5 拆分最小复现**

Gateway 偶发卡住？先在 CLI 单进程里复现。复现不了说明是 gateway 并发/race 问题，再去看 `_active_sessions` / `_running_agents` / `_pending_messages` 三者的状态。大多数 "agent 无响应" 事后看都是两级 guard 里消息被 queue 住了没触发 interrupt。

**K.6 Environment/benchmark 调试**

- `process` 模式跑一条 item，看 JSONL 里的 `messages` 字段，人眼看 trajectory 是否符合预期
- `compute_reward` 里 print `ctx.terminal("pwd && ls -la")`，确认沙盒状态
- Modal 卡住？先 `modal run` 手动起一下，确认 token 和镜像没坏

---

## L. 贡献流程（精华版）

想把修复/功能提 PR 回 Nous 上游？几个关键点，详细规则见 contributing.md。

**优先级**（Nous 明确列的）：bug fix > 跨平台兼容（macOS/WSL2/不同发行版） > 安全硬化 > 性能/健壮性 > 新 skill > 新 tool（大多数能力应当是 skill，不是 tool） > 文档。

**开发环境**：

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[all,dev]"
uv pip install -e "./tinker-atropos"
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
echo 'OPENROUTER_API_KEY=***' >> ~/.hermes/.env
```

**代码风格**：PEP 8（不强卡行长）、注释只解释非显然的意图、异常尽量抓具体类型用 `exc_info=True` 记日志、**永远不要硬编码 `~/.hermes`**（用 `get_hermes_home()` 和 `display_hermes_home()`，否则破坏多 profile 支持）。

**跨平台硬规矩**：

- `termios`/`fcntl` Unix-only，`try: ... except (ImportError, NotImplementedError):`
- `.env` 偶尔有非 UTF-8，`load_dotenv` fallback 到 `latin-1`
- `os.setsid/setpgid/killpg`、信号处理在 Windows 不同，先 `if platform.system() != "Windows":`
- 路径用 `pathlib.Path`，别字符串拼 `/`

**安全代码的守则**：shell 命令用 `shlex.quote()`、访问控制前 `os.path.realpath()` 解 symlink、别 log 秘密、tool 执行边界抓宽异常。Hermes 已有多层防护：sudo 密码 shlex.quote、危险命令正则 + 审批流、cron prompt-injection 扫描、write deny list、skills 安全扫描、子进程剥离 API key、Docker 能力全 drop。

**PR 流程**：分支名 `fix/` / `feat/` / `docs/` / `refactor/`；提交前 `pytest tests/ -v`；Conventional Commits 格式 `<type>(<scope>): <description>`；scope 常用 `cli / gateway / tools / skills / agent / install / security`。

**社区**：Discord `discord.gg/NousResearch`，GitHub Discussions 适合设计讨论，Skills Hub 可以上传专业技能共享给所有用户。协议 MIT。

---

## 小结：三段教程走完了

把 Part 1 / 3A / 3B 串起来看，你手里现在应当有三把钥匙：

- **入门（Part 1）**：十分钟装完、选好 provider、跑通第一条 chat。
- **主线（Part 3A）**：知道一次对话在 CLI → Agent Loop → Tool Registry → Provider Adapter 之间怎么流动，能读懂核心路径的 20% 代码解释 80% 的行为。
- **外围（Part 3B / 本篇）**：沙盒怎么换、会话怎么查、消息平台怎么接、二次开发怎么不改源码就上线。

剩下两件事：

1. **附录 A（场景速查）**：常见任务的最短 workflow，"我想 …… 应该用什么命令/配置"。
2. **附录 B（命令速查）**：hermes 子命令 / 环境变量 / 配置字段 / 调试技巧的速查表。

看完附录，这个知识库对 Hermes 的覆盖就到"能独立开发和运维"的深度了。

---

## 导航

| 章节 | 位置 | 说明 |
|---|---|---|
| Part 1 入门篇 | `part-1-入门篇.md` | 十分钟上手 |
| Part 2 进阶篇 | `part-2-进阶篇.md` | 日常使用、多平台 |
| Part 3A 核心执行路径 | `part-3a-核心执行路径.md` | CLI / Agent Loop / Registry / Adapter |
| **Part 3B 环境存储与扩展** | **本篇** | Environment / SQLite / Gateway / 扩展点 |
| 附录 A 场景速查 | `appendix-a-场景速查.md` | 按任务查操作 |
| 附录 B 命令速查 | `appendix-b-命令速查.md` | 命令/环境变量/配置字段 |

---
