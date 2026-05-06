---
来源: 基于 Hermes Agent v0.11.0 源码调研 + 官方文档
作者: 沙漠绿洲 (Ma Ronnie 知识库)
日期: 2026-05-06
标签: [hermes-agent, tutorial, beginner, cli, nous-research]
Ground Truth: ✅ 基于 ~/.hermes/hermes-agent/ 本地源码 + setup/doctor 实测
主题: Hermes Agent 入门篇 —— 零基础十分钟上手
系列: 三段式教程 Part 1 / 3
前置: 无
后续: part-2-进阶篇.md（待续）、part-3-源码深入.md（待续）
---

# Hermes Agent 入门篇：十分钟让它跑起来

> 系列定位：这是三段式教程的第一段。读完这一段，你会装、会用、会调模型、遇到卡点知道去哪查。
> 不要求任何 agent/LLM 背景知识。Python、命令行用过就行。

---

## 0. 读前须知

三段式教程的分工：

    第一段【入门】（本篇）   ——  会装、会聊、会用基本工具
    第二段【进阶】            ——  Skills/Memory/Gateway/Cron，把它当生产力
    第三段【深入】            ——  源码级：agent 循环、provider 路由、插件

本篇目标，读完你能做到：

- 装好 Hermes Agent，并跑通第一句对话
- 知道三种日常使用方式（交互 / 一次性 / 后台）
- 会切模型、换厂商、看体检报告
- 踩到常见错误能自己排查

---

## 1. Hermes Agent 是什么

一句话：**它是一个能直接操作你电脑的 AI 助手**。

和 ChatGPT 网页版的区别：

    ChatGPT 网页：      你 ↔ LLM（只能聊天）
    Hermes Agent：     你 ↔ LLM ↔ 工具（读写文件、执行命令、搜网页、发消息…）

它属于 "agent CLI" 这一类产品，同类的有 Claude Code、OpenAI Codex CLI、Cursor Agent、opencode、gemini-cli 等。Hermes 的特点：

- 自研 agent 核心（不依赖 LangChain / AutoGen 等框架）
- 支持 20+ 个 LLM 厂商（OpenAI、Anthropic、OpenRouter、DeepSeek、Gemini、Bedrock、Groq、xAI……）
- 内置 Skills 系统 / 长期记忆 / 定时任务 / 多平台 Gateway（微信、Telegram、飞书、Discord、Slack…）
- 纯 Python，MIT 协议，~12k 行核心代码

工作原理（下面这 5 步要记住，后面全靠它）：

    1) 你输入一句话
    2) Hermes 把"你说的话 + 系统提示 + 可用工具列表"发给 LLM
    3) LLM 回一个"工具调用"（比如：请帮我 read_file("/tmp/a.py")）
    4) Hermes 真的去执行这个工具，把结果塞回上下文
    5) LLM 看到结果，决定继续调工具，还是给你最终答案
    （2→5 循环，直到没有新工具调用为止）

这就是 "agent loop"。

---

## 2. 安装

### 2.1 系统要求

- macOS / Linux / WSL2（原生 Windows 有已知坑，建议走 WSL2）
- Python 3.10+
- 至少一个 LLM 厂商的 API key（下文会给免费额度方案）

### 2.2 一行命令装

    curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

装好之后 `hermes` 命令应该已经在 PATH 里。

### 2.3 从源码装（想看/改源码）

    git clone https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent
    cd ~/.hermes/hermes-agent
    pip install -e .

开发版本推荐走这种方式，改完代码立即生效。

### 2.4 验证安装

    hermes --version
    hermes doctor

`hermes doctor` 是体检命令，把没配的项目一条条列出来。出现红色 ❌ 就挨个修，全绿就可以开始用。

---

## 3. 第一次配置：5 分钟跑通

首次运行会触发 setup 向导：

    hermes setup

向导会问四类问题：

1. **选 LLM 厂商**：新手建议填两个 —— 一个能用的主力（比如 OpenAI / Anthropic），一个便宜的备用（DeepSeek / OpenRouter 路由）
2. **API Key**：粘贴进去就行，存在 `~/.hermes/config.yaml`
3. **默认模型**：选一个便宜又聪明的做日常（比如 `gpt-5-mini` / `claude-sonnet-4.7` / `deepseek-chat`）
4. **消息平台**（可选）：先跳过，等第二段再配

没有 API key 怎么办？三条路：

- **OpenRouter 免费额度**：注册送 $1，`google/gemini-2.0-flash:free` 等免费模型可用
- **DeepSeek**：新用户送 token，国内可直连，便宜
- **本地 Ollama**：完全离线，但需要自己的显卡，新手不推荐

配好之后跑：

    hermes chat -q "你好，你能用哪些工具？"

收到回答就通了。

### 3.1 随时修改配置

    hermes setup          # 整个向导重跑
    hermes model          # 只换模型/厂商
    hermes tools          # 启用/禁用工具
    hermes config get     # 查看所有配置
    hermes config set <key> <value>   # 改单项

配置文件在 `~/.hermes/config.yaml`，直接编辑也行。

---

## 4. 日常三种用法

### 4.1 交互模式（最常用）

    hermes

进入一个带彩色边框的对话框。直接说话，它会自动决定要不要调工具。

快捷键：

    Ctrl+C          中断当前回复
    Ctrl+D          退出
    ↑ / ↓           翻历史
    Shift+Enter     换行不提交

### 4.2 一次性问答（脚本里用）

    hermes chat -q "把当前目录下所有 py 文件行数统计一下"

不进交互界面，答完就退。适合写 shell 脚本、CI、cron 里用。

可以加 `--no-tools` 禁用工具（只聊天）；加 `--model xxx` 临时换模型。

### 4.3 后台长任务

    hermes chat -q "把 ~/myapp 的测试跑一遍，失败的修一下" > run.log 2>&1 &

结合 cron 工具还能让 agent 定时跑（第二段会讲）。

---

## 5. 斜杠命令速查

进入 `hermes` 交互模式后，以 `/` 开头的命令不走 LLM，直接控制会话。新手只要记住 6 个：

    /new              开新会话（清空上下文）
    /help             所有命令 + 简介
    /model            切模型
    /tools            启/禁工具
    /skill <名字>     加载一个技能（第二段细讲）
    /exit             退出

其他常用（知道存在就行，用到再查）：

    /rollback         回滚最近一次文件修改（agent 手抖时救命）
    /compress         手动压缩上下文（token 快满时用）
    /voice on         开启语音对话（需配 TTS）
    /cost             看当前会话消耗了多少钱
    /session list     看历史会话

---

## 6. 三个实战例子（跟着敲）

### 例 1：让它读代码

    hermes
    > 读一下 ~/myproj/main.py，告诉我它干嘛的

它会调 `read_file` 工具拿文件内容，再用 LLM 总结。

### 例 2：让它跑命令

    > 我电脑磁盘哪里占用最多？

它会调 `terminal` 工具，跑类似 `du -sh ~/* | sort -h` 的命令，返回结果并解释。

⚠️ **审批机制**：遇到危险命令（`rm -rf`、`sudo`、覆盖文件等）Hermes 默认会停下问你 "是否允许"。按 `y` 允许、`n` 拒绝、`a` 以后同类都允许。

### 例 3：让它搜网页

    > 搜一下 Claude 4.7 Opus 的最新评测，写 300 字总结

前提：配过搜索 provider（默认用 Bing / SerpAPI / Tavily 之一，`hermes doctor` 会提示）。

到这里你就能用 Hermes 干正事了。

---

## 7. 常见错误排查（碰到先看这里）

**① `hermes: command not found`**
→ 安装脚本没把 bin 目录加到 PATH。`export PATH="$HOME/.local/bin:$PATH"` 写进 `~/.bashrc` 或 `~/.zshrc`。

**② `authentication_error` / `401 Unauthorized`**
→ API key 错或过期。`hermes config get providers` 检查，或重跑 `hermes setup`。

**③ `rate_limit_exceeded` / `429`**
→ 免费额度打完了，换厂商：`hermes model` 切到 OpenRouter / DeepSeek。

**④ `context_length_exceeded`**
→ 会话太长超上下文窗口。输入 `/compress` 手动压缩，或 `/new` 开新会话。

**⑤ 工具调用一直失败 / 一直在转圈**
→ 看 `~/.hermes/logs/hermes.log`。常见是 provider 不支持 function calling（比如某些 Ollama 模型），换个模型解决。

**⑥ 中文输入乱码 / 符号错乱**
→ 终端 locale 问题，`export LANG=en_US.UTF-8` 或 `zh_CN.UTF-8`。

**⑦ `doctor` 报 git / rg / fd 找不到**
→ 装一下：`brew install git ripgrep fd`（mac）/ `apt install git ripgrep fd-find`（ubuntu）。

**⑧ 想看 agent 到底发了什么给 LLM**
→ `HERMES_DEBUG=1 hermes chat -q "..."`，或者进交互后 `/debug on`。会把 raw request/response 打到日志里。

---

## 8. LLM 厂商速查表（新手怎么选）

| 厂商 | 代表模型 | 日常用途 | 优点 | 缺点 |
|------|---------|---------|------|------|
| Anthropic | claude-sonnet-4.7 / opus-4.7 | 编码、长文、Agent 任务 | 工具调用最稳、长上下文 | 贵；国内不能直连 |
| OpenAI | gpt-5 / gpt-5-mini | 通用 | 生态最广 | 工具调用有时懒 |
| OpenRouter | 路由 300+ 模型 | 试新模型、备用 | 一个 key 通吃 | 加 5% 手续费 |
| DeepSeek | deepseek-chat / deepseek-reasoner | 中文、代码、便宜 | 便宜到离谱、国内直连 | 工具调用偶尔不稳 |
| Google | gemini-2.5-pro | 长上下文（1M+） | 上下文大、有免费额度 | 工具调用格式特殊 |
| Groq | llama-3.3-70b | 要求极快响应 | 毫秒级首 token | 模型选择少 |
| AWS Bedrock | Claude / Llama / Mistral | 企业内部 | 合规、billing 走 AWS | 配置最复杂 |
| Ollama | 本地模型 | 离线 / 隐私 | 完全免费、不联网 | 要显卡、工具调用弱 |

**推荐组合**：
- 完全新手 → OpenRouter（一个 key 试所有模型）
- 国内日常 → DeepSeek + 备用 Anthropic
- 老外 / 企业 → Anthropic + OpenAI
- 有显卡 + 隐私敏感 → Ollama (qwen2.5-coder 32B) + 备用云端

---

## 9. 本地目录速查

装完之后这些路径要知道：

    ~/.hermes/config.yaml          主配置（API key、模型、工具开关）
    ~/.hermes/logs/hermes.log      运行日志（排错第一站）
    ~/.hermes/state.db             SQLite：会话、记忆、FTS5 全文索引
    ~/.hermes/skills/              你自己的技能（Markdown）
    ~/.hermes/audio_cache/         语音/TTS 缓存
    ~/.hermes/hermes-agent/        源码（如果源码安装）

备份建议：`~/.hermes/` 整个压缩备份就行，可以在另一台机器 100% 还原。

---

## 10. 学完这段你能做什么 / 下段学什么

学完本段你会：

- ✅ 安装、配置、切模型
- ✅ 三种日常用法 + 斜杠命令
- ✅ 读文件 / 跑命令 / 搜网页
- ✅ 常见报错自查
- ✅ 厂商选型

下段【进阶篇】会教：

- Skills（技能）：把重复任务沉淀成可复用的"剧本"
- Memory（长期记忆）：跨会话记住你是谁、你的偏好
- Cron：让 Hermes 定时干活（比如每周一推送股市复盘）
- Gateway：把 Hermes 装进 Telegram / 微信 / 飞书 / Discord
- 多 agent 并行（delegate_task）
- MCP 接入第三方工具

---

## 附录 A：一页纸速查卡

    # 安装
    curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

    # 体检
    hermes doctor

    # 配置
    hermes setup            hermes model           hermes tools
    hermes config get       hermes config set k v

    # 用起来
    hermes                                  # 交互
    hermes chat -q "..."                    # 一次性
    hermes chat -q "..." --model gpt-5      # 临时换模型
    hermes chat -q "..." --no-tools         # 禁工具

    # 交互内斜杠（新手六件套）
    /new   /help   /model   /tools   /skill <x>   /exit

    # 救命
    /rollback     文件误操作了
    /compress     上下文太长
    HERMES_DEBUG=1 hermes ...    看 agent 发了啥

    # 排错第一站
    ~/.hermes/logs/hermes.log

---

> 📍 本篇位置：`docs/技术学习/hermes-agent-framework/part-1-入门篇.md`
> 📍 上级目录大纲：`docs/技术学习/hermes-agent-framework/README.md`
> 📍 下一篇（待续）：`part-2-进阶篇.md`
