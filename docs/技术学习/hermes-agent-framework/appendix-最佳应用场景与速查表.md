---
来源: 基于 Hermes Agent v0.11.0 官方文档（guides/ + reference/）整理
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, cookbook, reference, cheatsheet, automation, appendix]
Ground Truth: ✅ 基于 ~/.hermes/hermes-agent/ 官方 guides + reference 目录
主题: Hermes Agent 附录 —— 12 个最佳应用场景 + 全局速查表
系列: 三段式教程 附录 Appendix A+B
前置: part-1-入门篇.md（建议先过一遍）
后续: 无（工具书，按需查阅）
---

# 附录 A：最佳应用场景 12 例

> 这一章不讲原理，只讲「落地」。每个场景给你：谁适合用、痛点、怎么配、进阶技巧。
> 挑跟自己处境最像的那几个先跑通，剩下的当字典翻。

---

## 场景 1：个人知识管家（Obsidian 库 + 每日摘要）

**适合谁**：有 Obsidian/笔记库、每天产出碎片信息、希望自动归档和回顾的人。

**痛点**：每天存下几十条零散笔记、微信收藏、公众号链接，一周就变成信息坟场。想回顾的时候找不到、想提炼的时候懒得动手。

**落地方案**：

- 把 Hermes 的 CWD 指到笔记库目录（macOS/Linux：`cd ~/Obsidian/MyVault && hermes`）。
- 开启 `terminal + file + web` 三件套，让 agent 能读 md、grep、写摘要。
- 配 cron 每日定时跑（需要 `hermes gateway install` 把 gateway 装成服务）：

```
/cron add "0 22 * * *" "扫描 ~/Obsidian/MyVault/Inbox/ 下今天修改过的所有 md 文件，提取其中未完成的 todo、未消化的链接、关键概念。生成当日回顾笔记到 ~/Obsidian/MyVault/Daily/$(date +%Y-%m-%d).md，格式：# 今日回顾 / ## 关键产出 / ## 未完成 / ## 待读链接。如无新笔记输出 [SILENT]。" --name "daily-vault-digest" --deliver local
```

- 不想晚上打扰的话 `--deliver local`（只落盘），想推送到手机就 `--deliver telegram`。

**进阶 tips**：

- 加一条 `AGENTS.md` 到笔记库根，写清楚你的 PARA / Zettelkasten 规则；agent 每次进库会自动注入。
- 用 `/skills` 装一个 `gif-search` 或 `excalidraw` 之类的 skill，做图文并茂的周回顾。
- 用 `/memory` 把「我关注的长期主题：A 股、AI、宏观」存成长期记忆，以后摘要自动按这个维度分桶。

---

## 场景 2：A 股 / 美股 定时复盘助手（周报 cron）

**适合谁**：A 股零售、兼做美股、习惯用 Python 拉数据的技术型投资者。你已经在用脚本跑茅台周报？把它托管给 Hermes。

**痛点**：自己写的 cron 脚本只能输出固定模板，行情一变就要改代码；要的是「脚本拉数据 + LLM 负责读行情讲人话 + 直推到微信/Telegram」。

**落地方案**：

- 数据抓取用你已有的 Python 脚本（tushare/akshare/yfinance），丢到 `~/.hermes/scripts/zhou-bao.py`，它的 stdout 就是「本周数据块」。
- Hermes 的 cron 支持 `--script` 预处理 + agent 后处理：

```bash
hermes cron create "0 18 * * 5" \
  "下面是本周自选股的原始数据（来自脚本）。请按以下格式生成周报：
    1. 本周 MVP / 最大回撤
    2. 估值位分位数变化
    3. 基本面异动摘要（结合最新财报/公告）
    4. 下周关注点
  语气客观、数据优先，不要空话套话。最后附一句基于当前 PE 分位的操作倾向，不构成建议。" \
  --script zhou-bao.py \
  --name "weekly-stocks" \
  --deliver telegram
```

- 只要「本周跌破某阈值才提醒」这类纯告警，走 `--no-agent` 路径（场景 12），省 token。

**进阶 tips**：

- cron 是**无状态新会话**，所以 prompt 要把「自选股名单、分析维度、纪律」全写死，不能依赖上次会话记忆。
- 配 Fallback provider（主 Anthropic + 备 OpenRouter），周五 18:00 美东半夜限速时不至于发不出周报。
- 用 `execute_code` 工具让 agent 在周报里顺手画 matplotlib 图，落盘 png 再让 Telegram adapter 发图。

---

## 场景 3：GitHub PR 自动审查机器人（webhook 触发）

**适合谁**：小团队 maintainer、PR 堆到没人看、希望 AI 先筛一遍再上人眼的技术 lead。

**痛点**：PR 在队列里躺三天，junior 把 bug 合进 main；自己一早上全在读 diff 不在写代码。

**落地方案**：

- 有公网 URL（或 ngrok）→ 用 webhook，**秒级触发**。没公网 → 走场景 5 的 cron 轮询版。
- 开 webhook 平台（`~/.hermes/config.yaml`）：

```yaml
platforms:
  webhook:
    enabled: true
    extra:
      port: 8644
      routes:
        github-pr-review:
          secret: "强口令，必须和 GitHub 里完全一致"
          events: [pull_request]
          prompt: |
            PR #{number}: {pull_request.title}
            作者: {pull_request.user.login}
            先跑: gh pr diff {number} --repo {repository.full_name}
            按 code-review skill 的规则审查；结论 APPROVE / REQUEST_CHANGES / COMMENT。
            审完用 gh pr review 回帖。
```

- 新建 `~/.hermes/skills/code-review/SKILL.md`，写清楚你们团队的 review checklist（bug / 安全 / 性能 / 测试覆盖）。
- 在 GitHub repo 的 Settings → Webhooks 加 `https://your-host:8644/webhook/github-pr-review`，Secret 填同一串。

**进阶 tips**：

- **注意 prompt injection**：PR 标题/正文是攻击面，gateway 一定跑在 Docker backend (`TERMINAL_BACKEND=docker`)。
- 用 `hermes cron create` 配合 `hermes webhook subscribe --events pull_request` 也可以，webhook 订阅是动态的，不用重启。
- CI 失败事件也能接（`events: [check_run]`），让 agent 直接读失败日志归因（场景 7）。

---

## 场景 4：邮件分类 + 回复草稿生成

**适合谁**：收件箱每天几百封、需要一个「读邮件→分类→打草稿」的助理、但不放心让 AI 直接回信。

**痛点**：Gmail 过滤器规则写到爆炸还是漏；重要客户邮件被埋在 newsletter 里。

**落地方案**：

- Hermes 有 Email messaging adapter（IMAP/SMTP），配到 `~/.hermes/.env`：`EMAIL_IMAP_HOST / EMAIL_USER / EMAIL_PASSWORD` 等。
- 思路：**agent 只读 + 生成草稿，不自动发**。草稿落到 `~/.hermes/outbox/` 目录由你审后再手动发。
- cron 每 30 分钟扫一次：

```
/cron add "*/30 * * * *" "读取最近 30 分钟未读邮件。对每封邮件输出 JSON：{subject, sender, category(工作/账单/newsletter/垃圾), priority(P0-P3), suggested_reply}。P0-P1 写出完整回复草稿保存到 ~/.hermes/outbox/{timestamp}-{subject}.md。只有 P0 邮件要推送摘要到 Telegram；其他一律 [SILENT]。"
```

**进阶 tips**：

- 把常客/家人的邮箱存到 memory，让 agent 永远认得「这是 VIP」。
- 回复草稿要求 agent 先用 `/compress` 或显式引用原文最多 3 句，避免把整封邮件甩回去浪费 token。
- 垃圾邮件不仅标签，还可以让 agent 直接调 IMAP move 到垃圾箱（注意给接口做权限收敛，别误伤）。

---

## 场景 5：多 agent 并行研究（delegate_task orchestrator）

**适合谁**：做行业研究、技术选型、竞品对比的人——一次要同时跑 3~10 个小调研。

**痛点**：串行问「去查 A、再查 B、再查 C」，一个长对话把 context 撑爆；并行手动开三个窗口又难合并。

**落地方案**：

- 用 `delegate_task` 工具，每个子 agent 独立 context、独立 terminal、只回总结。
- Prompt 直接写：

```
并行研究以下 3 个方向，各 1 个子 agent：
1. PostgreSQL pgvector 方案的生产经验
2. Qdrant 在亿级向量下的表现
3. Milvus 2.x 的坑点
每个子任务只给 web toolset，重点输出：成熟度 / 运维成本 / 典型坑 / 推荐场景。
汇总到一张对比表，最后给出推荐。
```

- 背后 agent 会自动拆成 3 个子任务，每个子 agent 只看到自己的 goal/context（完全不知道你的聊天历史）。

**进阶 tips**：

- 默认最多 3 并发，改 `~/.hermes/config.yaml`：

```yaml
delegation:
  max_concurrent_children: 10
  max_spawn_depth: 2   # 允许子 agent 再派子 agent
```

- **子 agent 不认得上下文**，goal 和 context 里要把「项目路径、错误信息、约束」全写死。
- 机械数据抓取用 `execute_code`（批量调 web_search，便宜），复杂推理再丢给 delegate（贵但深）。

---

## 场景 6：Telegram 群管家（多用户 + 权限分级）

**适合谁**：3~20 人小团队、用 Telegram 沟通、想共享一个带终端权限的 AI 助手。

**痛点**：每人装一套 Hermes 贵又乱；但开公共 bot 又怕 `TERMINAL_BACKEND=local` 给人删库。

**落地方案**：

- 一台 $5/月 VPS，装 Hermes + gateway + Docker。
- `~/.hermes/.env`：

```bash
TELEGRAM_BOT_TOKEN=...
TELEGRAM_ALLOWED_USERS=123456789,987654321   # 静态白名单
TELEGRAM_HOME_CHANNEL=-1001234567890         # 群 ID
TERMINAL_BACKEND=docker                      # 强制容器，保护宿主
```

- 新成员加入走**DM pairing**：对方 DM 机器人拿到一次性码 `XKGH5N7P`，发给管理员，管理员执行：

```bash
hermes pairing approve telegram XKGH5N7P
```

- 进阶权限分级：`TELEGRAM_GROUP_ALLOWED_USERS`（只在群里响应的成员）vs `TELEGRAM_ALLOWED_USERS`（可 DM）。

**进阶 tips**：

- `hermes gateway install` 装成 systemd 服务，`sudo loginctl enable-linger $USER` 关 SSH 也不掉。
- 用 `profiles` 给不同团队开不同 Hermes 实例，各自独立 memory/skills/config。
- 绝对不要设 `GATEWAY_ALLOW_ALL_USERS=true`——带 terminal 工具的 bot 等于公开 shell。

---

## 场景 7：CI/CD 智能诊断（失败日志 → 根因归因）

**适合谁**：维护 GitHub Actions / GitLab CI pipeline、每天被红 badge 轰炸的 DevOps。

**痛点**：build 失败 1000 行日志 95% 是噪音，要人肉 grep 找那句 `AssertionError`；新同事看不懂 stack trace。

**落地方案**：

- webhook 订阅 `check_run` 或 `workflow_run` 事件。
- 路由 prompt：

```
CI job 失败：
 repo: {repository.full_name}
 workflow: {workflow_run.name}
 run URL: {workflow_run.html_url}

步骤：
1. gh run view {workflow_run.id} --repo {repository.full_name} --log-failed
2. 定位第一个 FAIL 的测试或编译错误，引用 5 行上下文
3. 参考 memory 中的 known issues（如果 hit 到就标注）
4. 给出：根因假设、验证命令、最小修复 diff

输出格式：Markdown，**最多 200 行**。如是已知的 flaky 测试输出 [SILENT]。
```

- Deliver 到 `github_comment`，直接评在 commit/PR 上。

**进阶 tips**：

- 把团队的「常见失败模式」存到 memory：`remember: 当你看到 "Connection reset" 通常是 test-db 容器冷启动，重跑即可`。这样 agent 能去噪。
- 用 skills 封装一套 `gh`/`kubectl`/`terraform` 命令规范，保证不同仓库的诊断流程一致。
- 成本控制：大模型只处理失败 job，成功的 job 直接 `[SILENT]`。

---

## 场景 8：本地离线 + Ollama（隐私优先）

**适合谁**：处理客户合同、医疗记录、内部代码等敏感数据、不能让数据出本机的人。

**痛点**：Claude/GPT 好用但合规部门不让用；自己用 vLLM/Ollama 部署，又没有趁手的 agent 框架能调工具。

**落地方案**：

- `curl -fsSL https://ollama.com/install.sh | sh`
- `ollama pull gemma4:31b`（**必须选支持 tool calling 的模型**，3B/7B 大多不行）
- `hermes setup` → 选 Custom Endpoint → base_url `http://localhost:11434/v1`，API key 留空或 `no-key`，model `gemma4:31b`。
- `~/.hermes/config.yaml` 加大 timeout（CPU 推理可能 30s+/query）：

```yaml
agent:
  api_timeout: 1800
model:
  default: gemma4:31b
  provider: custom
  base_url: http://localhost:11434/v1
```

- 扩展上下文（Ollama 默认 2048 tokens，太小）：

```bash
cat > /tmp/Modelfile <<'EOF'
FROM gemma4:31b
PARAMETER num_ctx 16384
EOF
ollama create gemma4-16k -f /tmp/Modelfile
```

**进阶 tips**：

- 配 fallback：90% 简单任务走本地免费，难任务自动切 Claude：

```yaml
fallback_providers:
  - provider: openrouter
    model: anthropic/claude-sonnet-4
```

- Mac 上可用 MLX 或 llama.cpp，M 系芯片 31B 也能跑到 15 tok/s。
- 24h keep-alive，不然每次启动要等 30s 加载权重：`OLLAMA_KEEP_ALIVE=24h`。

---

## 场景 9：团队共享 Gateway（Slack/飞书 + profiles 多实例）

**适合谁**：公司有多个小组（前端/后端/数据），每组都想要自己的 Hermes，但又共享一个 Slack workspace。

**痛点**：一个 Hermes 实例配一套 API key/skills/memory，多个团队共用会互相污染（前端组的 memory 里全是数据组的 SQL）。

**落地方案**：

- `hermes profile create frontend-team`、`hermes profile create data-team`，每个 profile 有独立的 `HERMES_HOME` 目录、独立的 config/memory/skills。
- 同一台机器跑多个 gateway 实例，用 `HERMES_HOME` 环境变量切换：

```bash
HERMES_HOME=~/.hermes-frontend hermes gateway install
HERMES_HOME=~/.hermes-data hermes gateway install
```

- 每个 profile 配自己的 Slack bot token（不同 app）或不同 Channel。
- 飞书/企微走对应 messaging adapter（`user-guide__messaging__feishu.md` / `wecom.md`）。

**进阶 tips**：

- 不同 profile 用不同 provider——研发组走 Claude Code OAuth 吃 plan 额度，数据组走 DeepSeek 省钱。
- Kanban (`hermes kanban`) 跨 profile，适合做团队任务板。
- `/config` 可以看当前 profile，免得在终端搞混。

---

## 场景 10：MCP 跨工具桥接（Slack ↔ Jira / Notion / 内部 API）

**适合谁**：公司有一堆内部 API / SaaS，想让 AI 不写代码就能跨系统查询。

**痛点**：手写一堆 tool 插件太累；直接给 agent 开全集 MCP server 又怕它删数据。

**落地方案**：

- `~/.hermes/config.yaml`：

```yaml
mcp_servers:
  github:
    command: npx
    args: [-y, "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "***"
    tools:
      include: [list_issues, create_issue, search_code]   # 白名单
      prompts: false
      resources: false
  jira:
    url: https://mcp.internal.example.com/jira
    headers:
      Authorization: Bearer ***
    tools:
      exclude: [delete_issue, delete_project]             # 黑名单危险操作
  notion:
    command: npx
    args: [-y, "@notionhq/notion-mcp-server"]
    env:
      NOTION_API_KEY: "secret_..."
```

- 改完 `/reload-mcp`，无需重启。
- 典型 prompt：「把 Jira 上所有『bug、无人认领、超 7 天』的 issue 同步一份汇总到 Notion 页面 XXX」——agent 会自动跨 3 个 MCP server 串起来。

**进阶 tips**：

- 敏感系统**一律用白名单 `include`**，不要信 `exclude`（新加的危险工具不会自动拦）。
- `tools.prompts: false / resources: false` 关掉非必要 wrapper，减少 tool 数量、省 token。
- WSL + Windows Chrome 场景：用 `chrome-devtools-mcp` 从 WSL 内跨界操控本机浏览器（免登录）。

---

## 场景 11：语音对话助手（voice mode 日常交互）

**适合谁**：通勤/散步时想让 AI 当研究搭子、或打字不便的场合。

**痛点**：想用 AI 但手头在忙；打字慢；希望像用 Siri 一样但又要有完整的 agent 工具能力。

**落地方案**：

- 装依赖：`pip install "hermes-agent[voice]"` + `brew install portaudio ffmpeg opus espeak-ng`（Linux 用 apt 对应包）。
- `~/.hermes/config.yaml`：

```yaml
voice:
  record_key: "ctrl+b"
  silence_threshold: 200
  silence_duration: 3.0
stt:
  provider: local       # 本地 whisper，免费
  local: { model: base }
tts:
  provider: edge        # Edge TTS 免费够用
  edge: { voice: "zh-CN-XiaoxiaoNeural" }   # 中文女声
```

- `hermes` 进 CLI 后 `/voice on` → 按 `Ctrl+B` 说话，自动转写+调用 agent +朗读回复。

**进阶 tips**：

- 手机端：Telegram 里发语音 → bot 自动 STT → agent 回复 → `/voice on` 朗读回去。真·移动 AI。
- Discord 语音频道：`/voice join`，bot 直接进语音房间听大家说话（先配好 STT 和权限）。
- 高质量组合：STT 用 Groq whisper-large-v3（几乎实时），TTS 用 ElevenLabs（自然度封神，但收费）。

---

## 场景 12：长周期后台任务（cron + webhook + 推送混合）

**适合谁**：需要跑数小时甚至几天的任务（大规模爬虫、批量文件处理、训练监控）；不希望卡在 CLI 前等结果。

**痛点**：`delegate_task` 是同步的，会话一断就没；`nohup` 又没法中间给反馈。

**落地方案**：三条路径按需组合：

**a) 无 LLM 看门狗（零 token）** ——磁盘/内存/GPU 告警：

```bash
cat > ~/.hermes/scripts/gpu-watch.sh <<'EOF'
#!/usr/bin/env bash
util=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits | head -1)
[ "$util" -gt 95 ] && echo "🔥 GPU util ${util}% on $(hostname)"
EOF
chmod +x ~/.hermes/scripts/gpu-watch.sh

hermes cron create "every 5m" --no-agent --script gpu-watch.sh \
  --deliver telegram --name "gpu-watchdog"
```

空输出=静默，有输出=Telegram 推一条。每 tick 0 token。

**b) 带 LLM 的周期总结**：场景 1-3 的路子，`--deliver telegram` + `[SILENT]` 机制过滤无事件 tick。

**c) 真正的后台 agent**：`terminal(background=True, notify_on_complete=True)` 把 agent 自己的一个 turn 放后台，完成后回 Telegram 通知。父会话即使断开也不影响。

**进阶 tips**：

- 系统级监控（监控 Hermes 自己时）用 OS cron + `hermes send` 一条，不依赖 gateway 存活。
- 组合拳：webhook 接外部事件 → 启动长任务 → 完成后 `hermes send` 回同一个 Telegram thread。
- 用 `hermes kanban` 把长任务做成任务卡片，状态、日志、产出一体化。

---

## 场景选型对照表

| 你想要的效果 | 推荐场景 | 必备功能模块 |
|---|---|---|
| 每天自动整理笔记 | 1 | Skills + Cron + Memory |
| 自动周报推送 | 2 | Cron + Gateway + --script |
| PR 秒级审查 | 3 | Webhook + Skills + Docker backend |
| 邮件辅助分流 | 4 | Email adapter + Cron + Memory |
| 多方向并行调研 | 5 | Delegation (orchestrator) |
| 团队共享 AI | 6 | Gateway + Pairing + Docker |
| CI 失败根因 | 7 | Webhook + Skills + Memory |
| 敏感数据不出本机 | 8 | Ollama + Fallback + 大 context |
| 多团队隔离实例 | 9 | Profiles + Gateway + Multi-messaging |
| 跨 SaaS 联动 | 10 | MCP (include 白名单) |
| 语音/移动交互 | 11 | Voice Mode + Messaging |
| 长周期后台 | 12 | Cron --no-agent + background terminal |

---

# 附录 B：全局速查表

## B.1 常用 CLI 命令

### 启动 / 会话
```
hermes                            # 进交互
hermes chat -q "问题" --provider openrouter --model xxx    # 一次性
hermes -z "问题"                  # 纯净一次性（脚本调用，只输出最终回答）
hermes --continue / -c            # 接着上次聊
hermes --resume <id> / -r         # 指定会话恢复
hermes --worktree / -w            # 开 git worktree 隔离跑
```

### 配置 / 体检
```
hermes setup                      # 全向导
hermes model                      # 加 provider / 换模型 / 跑 OAuth
hermes auth                       # 凭据管理
hermes tools                      # 按平台配工具开关
hermes doctor                     # 体检（依赖/API key/网络）
hermes dump                       # 导出整体配置摘要（贴 issue 用）
hermes config show / set k v / migrate
hermes logs --tail 100 / --follow
hermes status                     # agent + auth + platforms 状态
```

### Gateway / Cron
```
hermes gateway                    # 前台跑
hermes gateway install            # 装成用户 systemd/launchd
sudo hermes gateway install --system   # Linux 开机自启
hermes gateway start/stop/status
hermes cron list / create / edit / pause / resume / run <id> / remove <id>
hermes cron status                # 调度器状态
```

### Skills / Memory / MCP
```
hermes skills list / install <name> / publish
hermes curator status / run / pin <name>
hermes memory                     # 配外部记忆（Honcho/Supermemory/...）
hermes mcp add / list / test <name>
hermes webhook subscribe / list / remove
hermes pairing list / approve <platform> <code> / revoke / clear-pending
```

### 其他
```
hermes kanban list / create / dispatch
hermes sessions list / export / prune / rename
hermes insights                   # 30 天 token/成本/活动
hermes backup / import
hermes update [--check] [--backup]
hermes dashboard                  # 启本地 Web 管理台
hermes profile create/use/list    # 多实例隔离
```

## B.2 核心斜杠命令（CLI + Messaging）

| 命令 | 作用 | 典型用法 |
|---|---|---|
| `/new`（别名 `/reset`）| 起新会话 | 清上下文重来 |
| `/model [name]` | 查看/切模型 | `/model claude-sonnet-4 --global` |
| `/config` | 查当前配置（CLI） | 快速 debug |
| `/usage` | token 用量 + 成本 + provider 额度 | 看钱花哪 |
| `/compress [focus]` | 手动压缩上下文 | 长会话瘦身 |
| `/stop` | 打断 agent 正在跑的任务 | 跑飞了救命 |
| `/queue <prompt>` / `/q` | 排队下一个 prompt，不打断 | 边跑边攒 |
| `/steer <prompt>` | 在下个 tool call 后注入提示 | 中途微调方向 |
| `/goal <text>` | 设定跨 turn 目标（Ralph loop）| `/goal 把测试全部修绿` |
| `/background <prompt>` / `/bg` | 丢后台会话跑 | 不占当前 session |
| `/tools list/disable/enable` | 管工具开关（CLI） | 临时关危险工具 |
| `/skills` | 搜/装/查 skill | `/skills search security` |
| `/cron list/add/remove` | 管定时任务 | `/cron add "0 8 * * *" "..."` |
| `/voice on/off/tts/join/leave` | 语音模式 | Discord VC 必用 |
| `/browser connect` | 连本地 Chrome CDP | 自动化 |
| `/reload-mcp` | 改完 MCP 配置热加载 | 免重启 |
| `/sethome` | 标当前聊天为默认推送频道（Messaging） | cron 结果自动进这里 |
| `/yolo` | 跳过所有危险操作确认 | 谨慎使用 |
| `/approve session/always` / `/deny` | 批准/拒绝待决的危险命令（Messaging） | 远程管机器必备 |
| `/update` / `/restart` | 升级/重启 gateway（Messaging） | 手机端也能做 |

## B.3 关键配置字段（~/.hermes/config.yaml）

```yaml
model:
  default: "claude-sonnet-4.6"
  provider: "anthropic"             # 或 openrouter/custom/...
  base_url: "..."                   # 自定义端点时

fallback_providers:
  - { provider: openrouter, model: anthropic/claude-sonnet-4 }

agent:
  api_timeout: 1800                 # 秒；本地慢模型调大
  max_turns: 90

delegation:
  max_concurrent_children: 3        # 并行子 agent 数
  max_spawn_depth: 1                # 子 agent 是否能继续派
  orchestrator_enabled: true

terminal:
  backend: docker                   # local / docker / ssh / modal / ...
  container_memory: 5120            # MB
  container_persistent: true

platforms:
  telegram: { enabled: true, token: "..." }
  discord:  { enabled: true, token: "..." }
  webhook:
    enabled: true
    extra:
      port: 8644
      routes:
        my-route: { secret: "...", events: [...], prompt: "..." }

mcp_servers:
  name:
    command: npx
    args: [-y, "..."]
    env: { KEY: value }
    tools:
      include: [tool1, tool2]       # 白名单（推荐敏感系统）
      exclude: [dangerous1]
      prompts: false
      resources: false

display:
  tool_progress: new                # off / new / all / verbose
  tool_progress_command: true       # 让 messaging 也能用 /verbose

voice:
  record_key: "ctrl+b"
stt: { provider: local, local: { model: base } }
tts: { provider: edge, edge: { voice: "zh-CN-XiaoxiaoNeural" } }

quick_commands:
  deploy: { type: exec, command: scripts/deploy.sh }
  inbox:  { type: alias, target: "/gmail unread" }

model_aliases:
  fav:  { model: claude-sonnet-4.6, provider: anthropic }
  grok: { model: grok-4,            provider: x-ai }
```

## B.4 核心环境变量（~/.hermes/.env）

| 变量 | 作用 |
|---|---|
| `HERMES_HOME` | 切 Hermes 根目录（多实例隔离的总开关）|
| `HERMES_INFERENCE_PROVIDER` | 全局 provider 覆盖（`anthropic/openrouter/custom/...`）|
| `HERMES_INFERENCE_MODEL` | 全局模型覆盖 |
| `HERMES_DEBUG` / `HERMES_DUMP_REQUESTS` | 调试：打开 verbose 日志 / dump API payload |
| `HERMES_TIMEZONE` | 时区（IANA，如 `Asia/Shanghai`）|
| `OPENROUTER_API_KEY` / `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `DEEPSEEK_API_KEY` | 主流 provider key |
| `OPENAI_BASE_URL` | 自定义 OpenAI 兼容端点（vLLM / Ollama / SGLang）|
| `TERMINAL_BACKEND` | `local` / `docker` / `ssh` / `modal` / `daytona` / `vercel_sandbox` |
| `TERMINAL_DOCKER_IMAGE` | 容器镜像（默认 python-nodejs）|
| `SUDO_PASSWORD` | 非交互 sudo（谨慎）|
| `TELEGRAM_BOT_TOKEN` / `TELEGRAM_ALLOWED_USERS` / `TELEGRAM_HOME_CHANNEL` | TG 三件套 |
| `DISCORD_BOT_TOKEN` / `DISCORD_ALLOWED_USERS` / `DISCORD_ALLOWED_ROLES` | Discord 三件套 |
| `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` | Slack（Socket Mode）|
| `FIRECRAWL_API_KEY` / `TAVILY_API_KEY` / `EXA_API_KEY` | 网页搜索/抓取 |
| `GROQ_API_KEY` / `ELEVENLABS_API_KEY` / `VOICE_TOOLS_OPENAI_KEY` | 语音 STT/TTS |
| `BROWSER_CDP_URL` | 外接 Chrome 调试端口 |
| `GITHUB_TOKEN` / `GH_TOKEN` | GitHub API（`gh` 命令也认）|

完整列表看 `reference__environment-variables.md`——一共 500+ 行，覆盖了几十家 provider、全部 messaging 平台、容器/SSH 后端、观测（Langfuse）、RL 训练等。真用到哪一块再翻。

---

## 一句话收尾

**入门记 part-1，进阶翻 part-2，实战查本附录。** 碰到具体命令拿不准，先 `hermes doctor` + `hermes dump`，90% 的坑能自救。

---
