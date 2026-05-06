---
来源: 基于 Hermes Agent v0.11.0 源码调研 + 官方文档
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, tutorial, intermediate, skills, memory, cron, gateway, delegation, mcp]
Ground Truth: ✅ 基于 website/docs/ 官方文档 136 章实读 + 源码交叉验证
主题: Hermes Agent 进阶篇 —— 把它用成生产力
系列: 三段式教程 Part 2 / 3
前置: part-1-入门篇.md
后续: part-3-源码深入.md（待续）
---

# Hermes Agent 进阶篇：把它用成生产力

> 系列定位：这是三段式教程的第二段。Part 1 让你会装、会聊、会调模型；
> 这一段要把它从「能用的小工具」变成「每天替你干活的助手」。
> 你会学到：工具集怎么切、技能怎么沉淀、记忆怎么管、定时任务怎么串、多平台怎么接、子 agent 怎么并行、MCP 怎么接外部工具。

---

## 1. 从「会用」到「用成生产力」

Part 1 结束时你应该已经能跑下面这几件事：

    hermes                              # 交互对话
    hermes chat -q "..."                # 一次性问答
    /new /model /tools /skill ...       # 斜杠命令

但那还只是"会用"。真正的生产力来自下面这些能力的组合：

    Skills     → 把重复工作流沉淀成剧本（下次直接 /skill-name）
    Memory     → 跨会话记住你、你的环境、你的项目
    Sessions   → 任何时间回到任何一次旧对话
    Cron       → 让 Hermes 定时自己干活（监控 / 汇报 / 清理）
    Gateway    → 把它搬进 Telegram / Discord，手机上也能用
    delegate   → 拆任务给 3 个子 agent 并行跑
    MCP        → 把第三方工具（GitHub / Stripe / 数据库）接进来

这一段一个一个讲。每节都按 **是什么 / 为什么 / 怎么用 / 踩坑** 结构，
不扯概念，直接给命令给配置。

---

## 2. Tools 与 Toolsets：细粒度权限控制

### 2.1 是什么

Hermes 里每个"功能"都是一个 **tool**（`read_file`、`terminal`、`web_search` 等）。
tool 被打包进 **toolset**（`file`、`terminal`、`web` 等核心包）。
再往上是 **platform toolset**（`hermes-cli`、`hermes-telegram`、`hermes-discord` ...）
——给每个运行环境规定默认带哪些工具集。

    tool  →  toolset  →  platform toolset

举例：`read_file` ∈ `file` toolset ∈ `hermes-cli` platform。

### 2.2 为什么要管

- **安全**：Telegram 里的 bot 面向外人，没必要开 `terminal`
- **省 token**：工具定义本身要占上下文，30 个工具 vs 10 个工具差几 K tokens
- **避免误操作**：cron 里跑的 agent 不需要 `clarify`、`delegate_task`

### 2.3 怎么用

三种粒度，从粗到细：

**① 命令行临时指定（本次会话）**

    hermes chat --toolsets web,file,terminal
    hermes chat --toolsets debugging          # 组合包：file + terminal + web
    hermes chat --toolsets all                # 全开

**② 交互 UI 配置（持久，推荐）**

    hermes tools

会打开一个 curses 界面，每个 platform（cli / telegram / discord / cron ...）
一列，挨个勾选。存回 `~/.hermes/config.yaml`。

**③ 会话内动态切**

    /tools list
    /tools disable browser
    /tools enable rl

### 2.4 常用 toolset 速查

| toolset | 包含的工具 | 用途 |
|---------|-----------|------|
| `file` | `read_file`, `write_file`, `patch`, `search_files` | 文件读写 |
| `terminal` | `terminal`, `process` | 执行命令 + 后台进程管理 |
| `web` | `web_search`, `web_extract` | 搜 + 抓网页 |
| `search` | `web_search` | 只搜不抓 |
| `browser` | `browser_navigate`, `browser_click`, `browser_snapshot` ... | 浏览器自动化 |
| `memory` | `memory` | 跨会话记忆 |
| `skills` | `skill_manage`, `skill_view`, `skills_list` | 技能 CRUD |
| `cronjob` | `cronjob` | 定时任务 |
| `delegation` | `delegate_task` | 子 agent |
| `code_execution` | `execute_code` | 沙盒 Python |
| `messaging` | `send_message` | 跨平台发消息 |
| `todo` | `todo` | 会话内待办清单 |
| `vision` | `vision_analyze` | 图像理解 |
| `image_gen` | `image_generate` | 文生图 |
| `tts` | `text_to_speech` | 文本转语音 |
| `safe` | 只读研究 + 生成类 | 无文件写、无 terminal |
| `debugging` | 组合：`file` + `terminal` + `web` | 调 bug 常用 |

### 2.5 Terminal 后端（值得单独说）

`terminal` 工具可以切换执行环境：

    local         本机（默认）
    docker        持久容器（一个进程共享一个）
    ssh           远端服务器（安全推荐）
    singularity   HPC 镜像
    modal         serverless 云
    daytona       云开发沙盒
    vercel_sandbox Vercel microVM

配置：

    # ~/.hermes/config.yaml
    terminal:
      backend: docker
      docker_image: python:3.11-slim
      container_cpu: 1
      container_memory: 5120      # MB
      container_persistent: true  # 会话之间保留 /workspace

**Docker 后端有个特别要点**：整个 Hermes 进程共用 *一个* 长寿容器
（`docker run -d ... sleep 2h`），所有 `terminal` / `read_file` / `execute_code` 
都走 `docker exec` 进这同一个容器。你在一个 tool call 里 `pip install foo`，
下一个 tool call 还能用。`/new` 和 `delegate_task` 也共享。

SSH 后端推荐给有代码洁癖的人——agent 改不到自己的源码：

    terminal:
      backend: ssh
    # ~/.hermes/.env
    TERMINAL_SSH_HOST=my-server.example.com
    TERMINAL_SSH_USER=myuser
    TERMINAL_SSH_KEY=~/.ssh/id_rsa

### 2.6 踩坑

- 工具禁用了但 agent 还在"尝试调"→ 清理会话缓存：`/new`
- Docker 后端启动慢 → 第一次 `terminal` 调用会卡 3-5 秒拉容器，之后就快了
- `--toolsets web,file` 写法里**逗号分隔不要空格**

---

## 3. Skills 系统：把重复工作流沉淀下来

### 3.1 是什么

Skill = 一个 Markdown 文档 + 可选的辅助文件，告诉 agent "遇到这类任务按这个剧本走"。
它不是代码、不是函数——就是自然语言写的 SOP。

所有 skill 放在 `~/.hermes/skills/<category>/<skill-name>/SKILL.md`。

### 3.2 为什么要用

- **省 token**：技能走渐进式加载——在系统 prompt 里只显示 `{name, description, category}`
  三元组（Level 0，约 3k tokens），只有 agent 决定要用才加载完整内容（Level 1）
- **复用**：你第一次折腾通了 "在 Fly.io 部署 Python 应用"，写成 skill 下次直接 `/deploy-fly`
- **agent 自己会写**：复杂任务完成后，`skill_manage` 工具让它自动把经验存成技能
- **官方生态**：skills.sh / OpenAI skills / Anthropic skills / ClawHub 都能一键 install

### 3.3 SKILL.md 格式

```markdown
---
name: my-skill
description: Brief description of what this skill does
version: 1.0.0
platforms: [macos, linux]          # 可选，限制 OS
metadata:
  hermes:
    tags: [python, automation]
    category: devops
    fallback_for_toolsets: [web]   # 当 web 不可用时才显示（可选）
    requires_toolsets: [terminal]  # 当 terminal 可用才显示（可选）
    config:                         # 可选，需要用户填的配置
      - key: my.setting
        description: "What this controls"
        default: "value"
        prompt: "Prompt for setup"
---

# Skill Title

## When to Use
触发条件。

## Procedure
1. 步骤一
2. 步骤二

## Pitfalls
- 已知失败模式 + 怎么修

## Verification
怎么验证成功。
```

### 3.4 日常命令

**浏览 / 搜索**：

    /skills                               # 会话内列表
    /skills search docker
    hermes skills list                    # CLI 下列表
    hermes skills search kubernetes       # 跨源搜

**使用**：每个 skill 都是斜杠命令：

    /plan Design a REST API for a todo app
    /github-pr-workflow Create a PR for the auth refactor
    /excalidraw                           # 不带参数：加载后让你说要啥

**从 Hub 装新的**：

    hermes skills browse                                      # 浏览
    hermes skills install official/research/arxiv             # 官方可选
    hermes skills install openai/skills/k8s                   # GitHub 直装
    hermes skills install skills-sh/vercel-labs/.../react --force
    hermes skills install https://sharethis.chat/SKILL.md     # 单文件 URL
    hermes skills check                                       # 检查更新
    hermes skills update                                      # 拉更新

所有 hub 安装都过 **安全扫描**（检查注入、外泄、破坏性命令）。
`--force` 能覆盖 warn 级发现，但 `dangerous` 一定阻断。

**信任等级**：

| 等级 | 来源 | 策略 |
|------|------|------|
| `builtin` | 随 Hermes 发布 | 永远信任 |
| `official` | `optional-skills/` in repo | builtin 级，无三方警告 |
| `trusted` | openai/skills、anthropics/skills 等 | 宽松 |
| `community` | skills.sh、custom GitHub、URL | 非危险可 `--force` 覆盖 |

### 3.5 自己写一个（3 分钟）

    mkdir -p ~/.hermes/skills/mytools/check-disk
    $EDITOR ~/.hermes/skills/mytools/check-disk/SKILL.md

SKILL.md 就按上面的模板填。放好后下一次新会话就自动可见，`/check-disk ...` 直接能用。
不需要注册、不需要 reload。

需要辅助文件？放 `references/`、`templates/`、`scripts/`，在 SKILL.md 里写：

    See: skill_view("check-disk", "references/api.md")

agent 会按需懒加载。

### 3.6 agent 自动创建的技能

复杂任务（5+ tool call）完成后，Hermes 经常会主动问 "要不要把这个做法存成 skill"。
说 yes——这些 agent 自己写的 skill 往往比你手写的还好，因为它把中间踩的坑也顺带记录了。

`skill_manage` 工具的动作：

| 动作 | 用途 |
|------|------|
| `create` | 从零新建 |
| `patch` | 小改（首选，省 token） |
| `edit` | 大改，整篇替换 |
| `delete` | 整个删 |
| `write_file` / `remove_file` | 增删辅助文件 |

### 3.7 踩坑

- **新装 skill 在当前会话看不到** → 需要 `/new`（或 `--now` 让缓存失效）
- **带环境变量的 skill**（`required_environment_variables`）在 Telegram 上不会弹窗问你 key——
  messaging 平台不暴露密钥输入，必须去本地跑 `hermes setup` 或改 `~/.hermes/.env`
- **编辑过 bundled skill** 后 `hermes update` 不会覆盖你的修改（用 hash 比对判定）。
  想还原：`hermes skills reset google-workspace --restore`
- **外部技能目录**：公司/团队共享一份 `~/.agents/skills/`？写进 config：

      skills:
        external_dirs:
          - ~/.agents/skills
          - /home/shared/team-skills

  外部目录**只读**，agent 写改永远写到 `~/.hermes/skills/`。同名时本地版覆盖外部版。

---

## 4. Memory：跨会话的长期记忆

### 4.1 是什么

两个文件在 `~/.hermes/memories/`：

| 文件 | 干嘛 | 字符上限 |
|------|------|---------|
| `MEMORY.md` | agent 的笔记——环境、项目、学到的教训 | 2,200 (~800 tokens) |
| `USER.md` | 用户画像——你的偏好、沟通风格、身份 | 1,375 (~500 tokens) |

每次新会话启动，它们被原样塞进系统 prompt 顶部：

    ═══════════════════════════════════════
    MEMORY (your personal notes) [67% — 1,474/2,200 chars]
    ═══════════════════════════════════════
    User's project is a Rust web service at ~/code/myapi using Axum + SQLx
    §
    This machine runs Ubuntu 22.04, has Docker and Podman installed
    §
    User prefers concise responses, dislikes verbose explanations

条目之间用 `§` 分隔。

### 4.2 为什么限得这么死

因为**系统 prompt 的 KV cache 要稳定**——一个会话内 memory 一改，缓存就失效，每个 tool call 都重新算前缀 = 烧钱。
所以 Hermes 的策略是：

- memory 在**会话开始那一刻**冻结进系统 prompt
- 会话中途 agent 调用 `memory` 工具增删，**立即写盘**，但**下次会话才出现在 prompt**
- 当前会话里 agent 通过 tool 返回值看到"最新 memory"的真实状态

这个设计叫 **frozen snapshot pattern**，是 Part 3 源码篇会拆的重点之一。

### 4.3 memory 工具怎么用（给 agent 看，也能你自己引导）

三个 action：

    memory(action="add", target="memory", content="...")
    memory(action="replace", target="memory", old_text="...", content="...")
    memory(action="remove", target="memory", old_text="...")

没有 `read`——因为 memory 已经在系统 prompt 里了。
`old_text` 是**子串匹配**，不需要完整原文，只要能唯一识别就行。

### 4.4 该存 vs 不该存

**该存**：

- 用户偏好：`"User prefers TypeScript over JavaScript"` → `user`
- 环境事实：`"Server runs Debian 12, PostgreSQL 16"` → `memory`
- 纠正过的错：`"Docker commands don't need sudo here"` → `memory`
- 项目约定：`"Uses tabs, 120-char lines, Google docstrings"` → `memory`
- 完成里程碑：`"Migrated MySQL → PostgreSQL on 2026-01-15"` → `memory`

**别存**：

- 能搜到的常识（"Python 3.12 supports f-string nesting"）
- 原始数据 dump（日志、大段代码）
- 一次性上下文（临时文件路径、这次的 debug 细节）
- 已经在 SOUL.md / AGENTS.md 里的

### 4.5 满了怎么办

加不进去时，工具返回：

    {
      "success": false,
      "error": "Memory at 2,100/2,200 chars. Adding this entry (250 chars) would exceed the limit. Replace or remove existing entries first.",
      "usage": "2,100/2,200"
    }

agent 正确的反应：读现有条目 → 合并/替换相关的几条 → 再 add。
用到 80% 就应该开始主动 consolidate。

配置在 `~/.hermes/config.yaml`：

    memory:
      memory_enabled: true
      user_profile_enabled: true
      memory_char_limit: 2200
      user_char_limit: 1375

### 4.6 session_search：记忆的另一半

内置 memory 只能存精华（~1,300 tokens）。历史会话全量存在 `~/.hermes/state.db` 的
SQLite + **FTS5 全文索引** 里，agent 调用 `session_search` 工具查过去：

| 维度 | memory | session_search |
|------|--------|----------------|
| 容量 | ~1,300 tokens 上限 | 无限（所有历史） |
| 速度 | 即时（在 prompt 里） | 需查询 + LLM 摘要 |
| 用途 | 关键事实时刻在线 | "上周我们聊过 X 吗？" |
| 管理 | agent 手工维护 | 自动全存 |
| 成本 | 每会话固定 ~1,300 tokens | 按需 |

FTS5 语法标准：`docker deployment`、`"exact phrase"`、`docker OR kubernetes`、`deploy*`。

### 4.7 外部 memory provider

想要真正的"知识图谱式"跨会话记忆？装外部 provider：

    hermes memory setup    # 交互挑 provider
    hermes memory status

支持的：Honcho、OpenViking、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Supermemory。
它们**和**内置 memory 并存，不替换。具体对比见 `memory-providers.md`。

---

## 5. Sessions 与上下文压缩

### 5.1 是什么

每一次对话都是一个 **session**，双重存储：

1. **SQLite** (`~/.hermes/state.db`) — 元数据 + 全文 + FTS5
2. **JSONL** (`~/.hermes/sessions/`) — gateway 场景下的原始 transcript

每个 session 有：id、平台来源、user_id、**标题**、模型、系统 prompt 快照、完整消息、token 统计、`parent_session_id`（压缩时的链接）。

session id 格式：`YYYYMMDD_HHMMSS_<hex>`，如 `20250305_091523_a1b2c3d4`。

### 5.2 常用命令

**列表 / 搜索**：

    hermes sessions list                           # 最近 20 个
    hermes sessions list --source telegram         # 按平台
    hermes sessions list --limit 50
    hermes sessions stats                          # 统计

**恢复**：

    hermes -c                                      # 最近一次 CLI session
    hermes --continue
    hermes -c "my project"                         # 按标题恢复（系列自动挑最新）
    hermes -r 20250305_091523_a1b2c3d4             # 按 id
    hermes --resume "refactoring auth"             # 按标题

**命名**：auto 的 + 手动的

    /title my research project          # 会话内设
    hermes sessions rename <id> "refactoring auth module"

auto title 在第一次对话后用 fast aux model 后台生成，不加延时。

**导出 / 删除 / 清理**：

    hermes sessions export backup.jsonl
    hermes sessions export telegram.jsonl --source telegram
    hermes sessions delete <id> --yes
    hermes sessions prune --older-than 30 --yes

auto-prune 默认 **关闭**（你的历史就是 session_search 的燃料，悄悄删容易误伤）：

    sessions:
      auto_prune: true
      retention_days: 90
      vacuum_after_prune: true
      min_interval_hours: 24

### 5.3 上下文压缩：/compress

长会话迟早塞满 context window。遇到 `context_length_exceeded` 前主动：

    /compress

Hermes 会：
1. 摘要当前对话
2. **开一个新 session**（parent_session_id 指向老的）
3. 新 session 标题自动编号：`my project` → `my project #2` → `#3`

之后 `hermes -c "my project"` 自动挑最新那个。auto-compress 也会触发同样的 lineage。

### 5.4 踩坑

- `/compress` 会丢细节——把关键信息先存到 memory 或文件里再压
- session 太多了查得慢？384 MB / ~1000 session 时 FTS5 插入开始慢，这时才打开 `auto_prune`
- 群聊 session：默认 `group_sessions_per_user: true`，每个人自己的上下文。
  想要"一个房间一个大脑"设为 `false`

---

## 6. Cron：让 Hermes 定时干活

### 6.1 是什么

`cronjob` 工具让 Hermes 自己 schedule 任务。gateway daemon 每 60 秒 tick 一次，
到期的 job 在**全新的 agent session**里跑，结果投递到你指定的地方。

### 6.2 为什么要用

- 每天早上 9 点推 HN AI 新闻摘要
- 每 5 分钟看服务器内存，超阈值报警
- 每周日总结本周 git commits 发到 Slack
- 多级 pipeline：Job A（抓数据）→ Job B（筛选）→ Job C（发稿）

### 6.3 创建 job

**聊天里 `/cron` 快捷版**：

    /cron add 30m "Remind me to check the build"
    /cron add "every 2h" "Check server status"
    /cron add "every 1h" "Summarize new feed items" --skill blogwatcher
    /cron add "every 1h" "Use both skills" --skill blogwatcher --skill maps

**独立 CLI**：

    hermes cron create "0 9 * * *" "Audit open PRs, summarize CI" \
      --workdir /home/me/projects/acme \
      --skill pr-audit \
      --name "Morning PR audit"

**自然语言最省事**（Hermes 内部自动调 `cronjob` 工具）：

    每天早上 9 点从 Hacker News 抓 AI 新闻，总结后发到 Telegram。

### 6.4 schedule 格式

    # 一次性（相对）
    30m         → 30 分钟后
    2h          → 2 小时后
    1d          → 1 天后

    # 循环
    every 30m   → 每 30 分钟
    every 2h
    every 1d

    # 标准 cron
    0 9 * * *       → 每天 9:00
    0 9 * * 1-5     → 工作日 9:00
    0 */6 * * *     → 每 6 小时
    30 8 1 * *      → 每月 1 号 8:30
    0 0 * * 0       → 每周日 0:00

    # 绝对时间戳
    2026-03-15T09:00:00

### 6.5 生命周期管理

    /cron list
    /cron pause <job_id>
    /cron resume <job_id>
    /cron run <job_id>          # 立即触发下一 tick
    /cron remove <job_id>
    /cron edit <job_id> --schedule "every 4h" --prompt "..." --skill foo

（`hermes cron ...` 是同一套 CLI 版）

### 6.6 结果投递

`deliver` 参数决定把 agent 最终回复发哪儿：

| 值 | 说明 |
|-----|------|
| `"origin"` | 创建处（messaging 平台默认） |
| `"local"` | 只存本地 `~/.hermes/cron/output/`（CLI 默认） |
| `"telegram"` | telegram home 频道 |
| `"telegram:123456"` | 指定 chat id |
| `"telegram:-100123:17585"` | topic `chat_id:thread_id` |
| `"discord:#engineering"` | 指定频道名 |
| `"slack"` / `"whatsapp"` / `"signal"` / `"matrix"` / ... | 对应平台 home |

agent 的 **最终回复**自动投递——**不要**在 prompt 里再写 `send_message` 到同一个目标，Hermes 会去重。

**silent 模式**：agent 回复以 `[SILENT]` 开头 → 只存本地、不投递。做 watchdog 最爽：

    Check if nginx is running. If healthy, respond with only [SILENT]. Otherwise report the issue.

### 6.7 context_from 链式 job

cron job 默认完全隔离，每次一个新 session 不记得上次干过啥。
**但** `context_from=<job_id>` 让 Job B 自动拿到 Job A 最近一次的输出：

    # Job 1: 收集
    cronjob(action="create",
            schedule="0 7 * * *",
            prompt="Fetch top 10 AI/ML stories from HN. Save to ~/.hermes/data/raw.md.",
            name="AI News Collector")
    # → 假设返回 job_id = a1b2c3d4

    # Job 2: 筛选（自动拿到 Job 1 的输出）
    cronjob(action="create",
            schedule="30 7 * * *",
            context_from="a1b2c3d4",
            prompt="Score each story 1-10. Output top 5 to ~/.hermes/data/ranked.md.",
            name="AI News Triage")
    # → 返回 job_id = e5f6g7h8

    # Job 3: 输出
    cronjob(action="create",
            schedule="0 8 * * *",
            context_from="e5f6g7h8",
            prompt="Write 3 tweet drafts, deliver to telegram:7976161601.",
            name="AI News Brief")

`context_from` 也支持列表：`context_from=["job_a", "job_b"]`，输出按列表顺序拼。

### 6.8 脚本注入 & no-agent 模式

**预检脚本**：让 cron 先跑一段脚本，再决定要不要唤醒 agent：

    # script 最后一行输出 {"wakeAgent": false} → 本次 tick 不唤醒 agent
    import json, sys
    latest = fetch_latest_issue_count()
    if latest <= last_seen:
        print(json.dumps({"wakeAgent": False}))

超时配置：

    cron:
      script_timeout_seconds: 300   # 默认 120

**no-agent 模式**：根本不需要 LLM 推理的纯 watchdog：

    hermes cron create "every 5m" \
      --no-agent \
      --script memory-watchdog.sh \
      --deliver telegram \
      --name "memory-watchdog"

语义：
- stdout（非空）→ 作为消息**原样**投递
- stdout 为空 → **静默**，不投递（就是 watchdog 语义："没事别吵我"）
- 非零退出 / 超时 → 错误告警总会投递
- **完全不烧 token**

脚本必须放在 `~/.hermes/scripts/`（沙盒规则），`.sh`/`.bash` 用 bash，其他用 Python。

### 6.9 cron 的 toolset 策略

默认用你在 `hermes tools` 里给 `cron` 平台配的那套——**不是** CLI 的全家桶。
原因：每个工具定义都在 prompt 里占位置，cron job 跑得频繁，10 → 30 工具
能把月度账单吃掉两位数美金。

per-job 精确控制：

    cronjob(action="create", name="weekly-news",
            schedule="every sunday 9am",
            enabled_toolsets=["web", "file"],       # 只要这俩
            prompt="Summarize this week's AI news")

优先级：job 级 `enabled_toolsets` > `hermes tools` cron 平台配置 > 内置默认。

### 6.10 gateway 启动（cron 必需）

cron scheduler 在 gateway daemon 里跑。一次性装好：

    hermes gateway install              # 用户级 service（mac = launchd / linux = user systemd）
    sudo hermes gateway install --system  # Linux VPS：boot 系服务

状态检查：

    hermes gateway status
    hermes cron list
    hermes cron status

### 6.11 踩坑

- cron session 里**禁用 `cronjob` 管理工具**（防递归创建自己）
- `context_from` 指向的 job 必须**已经跑过**至少一次，否则只有 prompt 没有注入
- 用 `workdir` 的 job 是**串行**跑的（`TERMINAL_CWD` 是进程全局），无 workdir 的 job 还是并行
- provider 失败会按你配的 `fallback_providers` + credential pool 自动切，cron 天生更耐日间高峰限流

---

## 7. Gateway：多平台接入

### 7.1 是什么

Gateway 是一个常驻后台进程，同时监听 Telegram / Discord / Slack / WhatsApp / Signal / Email / SMS / 
Matrix / Mattermost / DingTalk / Feishu / WeCom / Weixin / BlueBubbles / QQ / Yuanbao / Teams / 
Home Assistant / API Server / Webhook 的消息，把每条消息路由给 `AIAgent` 跑，再投递回原 chat。

### 7.2 快速配置

    hermes gateway setup     # 交互向导，挨个平台配
    hermes gateway           # 前台跑（调试）
    hermes gateway install   # 装 service
    hermes gateway start
    hermes gateway status
    hermes gateway stop

日志在：
- Linux: `journalctl --user -u hermes-gateway -f`
- macOS: `tail -f ~/.hermes/logs/gateway.log`

### 7.3 Telegram（最常用，展开）

1. 去 @BotFather 申请 bot token
2. `hermes gateway setup` → 选 Telegram → 粘 token
3. **必填**：`TELEGRAM_ALLOWED_USERS=<your_user_id>`（不填默认拒绝所有人）
4. 给 bot 发 `/start`，Hermes 就在手机上了

`.env` 示例：

    TELEGRAM_BOT_TOKEN=123456:ABC...
    TELEGRAM_ALLOWED_USERS=123456789,987654321
    TELEGRAM_HOME_CHANNEL=123456789        # cron 默认投递地

### 7.4 Discord（次常用）

1. discord.com/developers 建应用 + bot，拿 token
2. 邀请 bot 进你的 server（权限：Read/Send Messages、Embed Links、Read History、Attach Files）
3. `hermes gateway setup` → Discord → 粘 token
4. `DISCORD_ALLOWED_USERS=<your_user_id>`

Discord 独享能力（`hermes-discord` toolset 自动加 `discord` + `discord_admin`）：
- 发 embed、管频道、踢人/改 role（需要 bot 在 discord 那边也有对应权限）
- 语音频道：`/voice join` 让 bot 进语音，可以直接语音对话

### 7.5 其他平台速览

| 平台 | toolset | 特色 |
|------|---------|------|
| Slack | `hermes-slack` | 线程回复、typing 指示 |
| WhatsApp | `hermes-whatsapp` | via WhatsApp Web Bridge |
| Signal | `hermes-signal` | 端到端，允许列表用手机号 |
| SMS | `hermes-sms` | Twilio 后端，纯文本 |
| Email | `hermes-email` | IMAP 入 + SMTP 出 |
| Matrix | `hermes-matrix` | 支持 e2ee 房间 |
| Feishu/Lark | `hermes-feishu` | 额外 `feishu_doc_read` / `feishu_drive_*` |
| DingTalk / WeCom / Weixin | 对应 | 中国企业场景 |
| BlueBubbles | `hermes-bluebubbles` | macOS 服务器中转 iMessage |
| Teams | `hermes-teams` | Azure AD 集成 |
| Home Assistant | `hermes-homeassistant` | 加 `ha_*` 工具控家居 |

### 7.6 session 隔离

gateway session 的 key 是确定性的：

| chat 类型 | 默认 key |
|-----------|---------|
| Telegram DM | `agent:main:telegram:dm:<chat_id>` |
| Discord DM | `agent:main:discord:dm:<chat_id>` |
| 群 | `agent:main:<platform>:group:<chat_id>:<user_id>` |
| 线程 | `agent:main:<platform>:group:<chat_id>:<thread_id>` |

默认 `group_sessions_per_user: true`——群里每人自己的 session，互不干扰。
想要"房间共享大脑" → `config.yaml` 改 `false`。

### 7.7 reset 策略

消息 session 按配置自动 reset，不然上下文无限涨：

    # ~/.hermes/gateway.json
    {
      "reset_by_platform": {
        "telegram": { "mode": "idle", "idle_minutes": 240 },
        "discord":  { "mode": "idle", "idle_minutes": 60 },
        "slack":    { "mode": "daily", "hour": 4 }
      }
    }

模式：`idle` / `daily` / `both` / `none`。reset 前 agent 有一次机会把要事存进 memory。
**有后台进程**的 session 永不 auto-reset。

### 7.8 消息里的斜杠命令

    /new /reset            新开会话
    /model gpt-5-mini      切模型
    /stop                  打断当前任务
    /compress              压缩上下文
    /title "my project"    起名
    /resume "my project"   恢复
    /sethome               把当前 chat 设为 home（cron 默认投递到这）
    /voice on              开语音回复
    /background <prompt>   起后台 session 干活，结果送回来
    /rollback              回滚文件修改
    /<skill-name> ...      跑 skill

### 7.9 busy-input 三种模式

消息 agent 正忙时你又发消息，默认是**打断**。配置可改：

    display:
      busy_input_mode: steer    # 或 queue / interrupt（默认）
      busy_ack_enabled: true

- `interrupt`：杀掉当前任务、重启一轮（默认）
- `queue`：排队到当前任务完再跑
- `steer`：塞进当前 run 下一个 tool call 之后（不打断、不新轮）——不适合 "完全换任务"，适合 "补充细节"

### 7.10 踩坑

- **默认拒绝所有人**是安全默认。务必先配 `*_ALLOWED_USERS` 或用 DM 配对：

      # 陌生人 DM bot → 收到 pairing code "XKGH5N7P"
      hermes pairing approve telegram XKGH5N7P
      hermes pairing list
      hermes pairing revoke telegram 123456789

- 装完新工具（ffmpeg / 新 Node）要 `hermes gateway install` 一次，重刷 launchd plist 的 PATH
- 一台机器别同时装 user + system service，会冲突
- Telegram 消息超长：Hermes 自动分片或用 streaming 编辑更新

---

## 8. delegate_task：子 agent 并行

### 8.1 是什么

`delegate_task` 工具产生一个**全新的** AIAgent 实例，独立对话、独立工具集、独立 terminal session。
只有它的**最终总结**进入父 agent 的上下文。

默认并发 3（可配置），**没有硬顶**。

### 8.2 为什么要用

- **并行省时**：研究 3 个话题同时跑，串行的话 3×
- **保护父上下文**：refactor 200 个文件会把父 context 塞爆；扔给子 agent，只要总结回来
- **隔离失败**：子 agent 崩了不影响父

### 8.3 怎么用

**单任务**：

    delegate_task(
        goal="Debug why tests fail",
        context="Error: assertion in test_foo.py line 42",
        toolsets=["terminal", "file"]
    )

**并行批量**：

    delegate_task(tasks=[
        {"goal": "Research WebAssembly 2025",   "toolsets": ["web"]},
        {"goal": "Research RISC-V 2025",        "toolsets": ["web"]},
        {"goal": "Research quantum 2025",       "toolsets": ["web"]},
    ])

### 8.4 子 agent 一无所知

**最关键的坑**：子 agent 的对话从 0 开始，不知道父对话发生过啥。
`goal` + `context` 必须把子 agent 需要的一切塞进去：

    # BAD - subagent 不知道 "the error" 指啥
    delegate_task(goal="Fix the error")

    # GOOD
    delegate_task(
        goal="Fix the TypeError in api/handlers.py",
        context="""The file api/handlers.py has a TypeError on line 47:
        'NoneType' object has no attribute 'get'.
        process_request() receives a dict from parse_body(),
        which returns None when Content-Type is missing.
        Project at /home/user/myproject, Python 3.11."""
    )

### 8.5 leaf vs orchestrator（多级委托）

默认**扁平**：父（depth 0）→ 子（depth 1），子**不能再委托**。

多级场景（研究 → 综合、分形并行）：

    delegate_task(
        goal="Survey three code review approaches",
        role="orchestrator",      # 允许此子再派工
        context="...",
    )

- `role="leaf"`（默认）：不能委托
- `role="orchestrator"`：保留 `delegation` toolset，但受 `delegation.max_spawn_depth` 限制
- `max_spawn_depth: 1`（默认）= 扁平，orchestrator 等于 no-op
- 调到 `2` 允许 orchestrator 再生 leaf 孙子；`3` 是上限（3 级）

**成本警告**：`max_spawn_depth=3` + `max_concurrent_children=3` = 最多 27 个并发 leaf。
全局熔断：`orchestrator_enabled: false` 强制所有 child = leaf。

### 8.6 leaf 子 agent 被封禁的工具

无论 toolset 怎么传，leaf 永远不能用：

- `delegate_task`（防无穷递归）
- `clarify`（不能问人话）
- `memory`（不能写共享 memory）
- `code_execution`（子要逐步推理）
- `send_message`（不能跨平台发消息）

orchestrator 保留 `delegate_task`，其他 4 个依旧禁。

### 8.7 超时与诊断

    delegation:
      max_iterations: 50                   # 单 child tool 轮上限
      max_concurrent_children: 3
      max_spawn_depth: 1
      orchestrator_enabled: true
      child_timeout_seconds: 600           # 10 分钟静默才杀，reasoning 模型够用
      # 可覆盖子 agent 模型：
      model: "google/gemini-flash-2.0"
      provider: "openrouter"

timer 在 child 每次 API / tool call 时重置——真没动静才杀。
如果 child 在**零次 API call** 就超时（通常是 provider 不通、鉴权错），
Hermes 写诊断：`~/.hermes/logs/subagent-timeout-<session>-<ts>.log`。

### 8.8 /agents TUI 总览

交互 TUI 里敲 `/agents`（别名 `/tasks`）：

- 实时树视图
- 各 branch 的 cost / tokens / 动过的文件
- 单独 kill / pause 某个 subagent
- 事后回放每个 subagent 的 turn-by-turn

### 8.9 delegate_task 不是后台队列

    ⚠️ delegate_task 是**同步**的，跑在父的当前轮。

父被打断（新消息、`/stop`、`/new`）→ 所有 active 子也一起取消。子**不会**在父轮结束后继续。

要**耐久**的长任务：

- `cronjob(action="create", ...)` — 独立调度，免疫父打断
- `terminal(background=true, notify_on_complete=true)` — 长命令后台跑

### 8.10 delegate_task vs execute_code

| 维度 | delegate_task | execute_code |
|------|--------------|--------------|
| 推理 | 完整 LLM loop | 纯执行 Python |
| 上下文 | 新鲜独立会话 | 无会话 |
| 工具 | 几乎全部（除禁用的） | 7 个 RPC 工具，无推理 |
| 并行 | 默认 3 并发 | 单脚本 |
| 场景 | 需要判断、多步 | 机械数据处理 |
| token | 高 | 低 |

**Rule of thumb**：需要"判断"时用 `delegate_task`；纯"把 JSON 转表格" 用 `execute_code`。

---

## 9. MCP：接入外部工具生态

### 9.1 是什么

MCP（Model Context Protocol）= 一个开放协议，让 LLM 使用住在外部服务里的工具。
GitHub、Stripe、数据库、FS、内网 API ... 只要有对应 MCP server，Hermes 就能用它们的工具。

Hermes 同时支持 **两种**：

- **stdio server**：本地子进程，走 stdin/stdout
- **HTTP server**：远程 HTTP 端点

### 9.2 配置示例

`~/.hermes/config.yaml`：

    mcp_servers:
      filesystem:
        command: "npx"
        args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]

      github:
        command: "npx"
        args: ["-y", "@modelcontextprotocol/server-github"]
        env:
          GITHUB_PERSONAL_ACCESS_TOKEN: "***"
        tools:
          include: [create_issue, list_issues, search_code]
          prompts: false                 # 不注册 list_prompts/get_prompt

      stripe:
        url: "https://mcp.stripe.com"
        headers:
          Authorization: "Bearer ***"
        tools:
          exclude: [delete_customer, refund_payment]

      legacy:
        url: "https://mcp.legacy.internal"
        enabled: false                   # 完全跳过

### 9.3 工具命名

Hermes 统一加前缀，不和内置冲突：

    mcp_<server_name>_<tool_name>

举例：`mcp_filesystem_read_file`、`mcp_github_create_issue`。
你不用手动叫这些名字——agent 自己从工具列表里选。

每个有至少一个工具的 server 还自动生成一个运行时 toolset：`mcp-<server>`。
`--toolsets mcp-github` 就能只开 GitHub 的工具。

### 9.4 过滤（必学的安全控制）

- `enabled: false` — 整个 server 停掉
- `tools.include: [...]` — 白名单，优先级最高
- `tools.exclude: [...]` — 黑名单
- `tools.prompts: false` / `tools.resources: false` — 关掉对应的 utility wrapper

同时有 include 和 exclude → **include 赢**。

### 9.5 运行时

- 启动时扫描、自动注册
- server 改了自己的工具列表会推送 `notifications/tools/list_changed`，
  Hermes 自动 refresh，不用手动 reload
- 改了 config？聊天里 `/reload-mcp`

### 9.6 Sampling（server 反向请求 LLM）

MCP server 能通过 `sampling/createMessage` 让 Hermes 代它调 LLM。
默认开，每 server 可配：

    mcp_servers:
      my_server:
        command: "my-mcp-server"
        sampling:
          enabled: true
          model: "openai/gpt-4o"
          max_tokens_cap: 4096
          timeout: 30
          max_rpm: 10
          max_tool_rounds: 5
          allowed_models: []

有速率、token 上限、工具循环深度限流，防止 server 跑飞你的账单。

### 9.7 把 Hermes 自己当 MCP server

反向——让 Claude Code / Cursor / Codex 通过 MCP 使用 Hermes 的 messaging 能力：

    hermes mcp serve

Claude Code 配置：

    {
      "mcpServers": {
        "hermes": {
          "command": "hermes",
          "args": ["mcp", "serve"]
        }
      }
    }

暴露 10 个工具：`conversations_list`、`conversation_get`、`messages_read`、
`attachments_fetch`、`events_poll`、`events_wait`、`messages_send` 等。
效果：Claude Code 能通过 Hermes 读写你所有消息平台。

### 9.8 踩坑

- stdio server 启动失败 → `node --version && npx --version`，`uv pip install -e ".[mcp]"`
- 工具没出现 → 可能被 `include`/`exclude` 过滤掉了，或者 server 根本没这个 capability
- stdio server 的 env：Hermes **不会**透传你完整 shell env，只传 `env:` 里明说的 + 安全基线（防密钥泄漏）

---

## 10. Voice Mode / Personality（知道存在即可）

### 10.1 Voice Mode

`/voice on` 开语音对话，具体支持：

- **CLI 麦克风模式**：按键说话、agent TTS 回读
- **messaging 平台**：发语音消息会被自动转写；你也可以让它 TTS 回语音
- **Discord 语音频道**：`/voice join` 让 bot 直接加入语音频道对话

10 种 TTS provider 可选：
Edge TTS（免费）、ElevenLabs、OpenAI TTS、MiniMax、Mistral Voxtral、Gemini、xAI、NeuTTS、KittenTTS、Piper——加自定义命令 provider 接任何本地 TTS CLI。

具体见 `user-guide/features/voice-mode.md`。

### 10.2 Personality

`SOUL.md` = 系统 prompt 里**第一**个文件，定 agent 的基本人设。
Hermes 带一套预设（默认 `hermes` / `nous` 等）+ 你能自定义：

    /personality hermes
    /personality pirate
    /personality show

自定义：`~/.hermes/personalities/<name>.md`。具体见 `personality.md`。

---

## 11. Context Files：项目级上下文自动注入

### 11.1 是什么

Hermes 启动时自动扫当前目录（以及父目录），找到这些文件就塞进系统 prompt：

| 文件 | 来源 |
|------|------|
| `.hermes.md` | Hermes 原生 |
| `AGENTS.md` | OpenAI Codex 规范 |
| `CLAUDE.md` | Claude Code 规范 |
| `SOUL.md` | 人设 |
| `.cursorrules` | Cursor IDE |

同一套 agent 可以在 Cursor / Claude Code / Hermes 里跨工具复用项目规则。

### 11.2 `@` 引用

聊天里打 `@` 可以快速塞东西进消息：

    @file:src/main.py         整个文件
    @dir:src/                 整个目录 tree
    @git-diff                 当前 git diff
    @url:https://...          抓网页

Hermes 展开后 inline 拼到你的消息末尾。

---

## 12. 小结 + 引出 Part 3

到这里你应该能：

- ✅ 按平台 / 会话 / 任务三个粒度切 toolset
- ✅ 从 Hub 装技能、自己写技能、让 agent 自己沉淀技能
- ✅ 管理 memory / user profile 双库，理解 frozen snapshot 为何而存
- ✅ 找回/恢复/压缩会话，起系列标题
- ✅ 写 cron job，串 context_from pipeline，no-agent watchdog
- ✅ 把 Hermes 装进 Telegram / Discord 等 20 个平台
- ✅ 用 delegate_task 并行跑 3 件事，区分 leaf/orchestrator
- ✅ 接 MCP server，过滤到需要的子集
- ✅ 知道 voice / personality / context files 的存在

**Part 3【源码深入】**会打开黑盒，带你读 ~12k 行核心 Python：

- `AIAgent` 主循环：消息 → 系统 prompt → LLM → 工具 → 再循环的真实路径
- Provider 路由 + fallback + credential pool 的决策树
- Frozen snapshot 是怎么实现的、prefix cache 怎么保住
- Tool schema 生成、工具调用解析的跨 provider 兼容层
- Skills loader 的渐进式披露机制
- Gateway 的 asyncio 多平台架构
- Plugin 系统、hook 机制、如何自己写插件
- 数据库 schema 全览（sessions / messages / FTS5 / cron jobs / checkpoints）

想深挖、改源码、写扩展 → Part 3 见。
只是日常用？看到这里已经够了——这本书的 Part 1 + Part 2 覆盖你 95% 的使用场景。

---

## 附录 A：进阶速查卡

    # Toolsets
    hermes tools                              # 交互 UI
    hermes chat --toolsets web,file,terminal
    /tools list | /tools enable X | /tools disable X

    # Skills
    /skills | /skills search X
    /<skill-name> [args]
    hermes skills browse | install | check | update | reset
    ~/.hermes/skills/<category>/<name>/SKILL.md

    # Memory
    memory(action="add"|"replace"|"remove", target="memory"|"user", ...)
    ~/.hermes/memories/MEMORY.md   # 2,200 char
    ~/.hermes/memories/USER.md     # 1,375 char

    # Sessions
    hermes -c | hermes -c "title" | hermes -r <id>
    hermes sessions list | stats | export | prune | rename
    /title X | /compress | /resume "X"

    # Cron
    /cron add "every 1h" "..." --skill X
    hermes cron create | list | pause | resume | run | edit | remove
    schedule: 30m | every 2h | "0 9 * * 1-5" | 2026-03-15T09:00:00
    deliver: origin | local | telegram[:id] | discord[:#ch] | slack | ...
    context_from="<job_id>" | context_from=["a","b"]
    --no-agent --script foo.sh     # watchdog 纯脚本

    # Gateway
    hermes gateway setup | install | start | stop | status
    TELEGRAM_ALLOWED_USERS=... | DISCORD_ALLOWED_USERS=...
    hermes pairing approve <platform> <code>
    /sethome | /background <prompt>

    # Delegate
    delegate_task(goal=..., context=..., toolsets=[...])
    delegate_task(tasks=[{...}, {...}])
    role="orchestrator" + max_spawn_depth:2
    /agents         # TUI 总览

    # MCP
    mcp_servers:
      name: {command, args, env}         # stdio
      name: {url, headers}               # HTTP
      name: {tools: {include, exclude, prompts, resources}}
    /reload-mcp

---

## 导航

    上一篇： part-1-入门篇.md              （十分钟上手）
    本篇 ：  part-2-进阶篇.md              （把它用成生产力）← 你在这里
    下一篇： part-3-源码深入.md            （架构、主循环、Provider 路由、插件）

大纲：`docs/技术学习/hermes-agent-framework/README.md`

---

> 📍 本篇位置：`docs/技术学习/hermes-agent-framework/part-2-进阶篇.md`
