# 📚 Full Knowledge Base Library

个人知识库 — 系统化整理学习笔记、分析报告、投资研究等内容。

---

## 📂 目录结构

```
docs/
├── 投资教育/          # 投资与交易学习笔记
│   └── 股票交易/      # 股票交易课程与策略
├── 市场分析/          # 市场研报、日报、宏观分析
└── 工具与平台/        # 交易工具、平台使用指南
```

## 🗂️ 文档索引

### 技术学习 / Hermes 系列

> ⚠️ **两个方向请勿混淆**（都出自 Nous Research，但目标不同）：
> - **`hermes-agents/`** — 聚焦 **Hermes 3 模型 + Function Calling**（模型层、ChatML、GOAP、微调）
> - **`hermes-agent-framework/`** — 聚焦 **Hermes Agent CLI/Gateway 框架**（应用层运行时、工具系统、跨平台网关）

#### A. ~~Hermes 3 模型与 Function Calling（模型层）~~ 已归档

> 原计划针对 Hermes 3 语言模型层写的三部曲，主角定位调整后统一聚焦 Agent 框架（见 B），本方向归档。
> 目录 `docs/技术学习/hermes-agents/` 保留一个废弃声明文件作历史记录。

#### B. Hermes Agent 框架（CLI / Gateway 运行时）✅ 全部完成

| 文档 | 内容 | 状态 |
|------|------|:---:|
| [🗺️ 从这里开始 / 学习路径](docs/技术学习/hermes-agent-framework/00-从这里开始.md) | 按画像推荐阅读路径 + 全文件一览 | ✅ |
| [📖 目录 README](docs/技术学习/hermes-agent-framework/README.md) | 三部曲索引 + 知识地图 | ✅ |
| [Part 1 · 入门篇](docs/技术学习/hermes-agent-framework/part-1-入门篇.md) | 十分钟上手：安装、配置、三种用法、斜杠命令、厂商选型、8 大错误速查 | ✅ |
| [Part 2 · 进阶篇](docs/技术学习/hermes-agent-framework/part-2-进阶篇.md) | Tools / Skills / Memory / Sessions / Cron / Gateway / delegate_task / MCP / Voice | ✅ |
| [Part 3A · 架构与主循环](docs/技术学习/hermes-agent-framework/part-3a-架构与主循环.md) | 总体架构 + Agent Loop + Prompt 装配 + 压缩+Cache + Provider 适配 + Tools Runtime | ✅ |
| [Part 3B · 环境存储与扩展](docs/技术学习/hermes-agent-framework/part-3b-环境存储与扩展.md) | 5 种 Environment + SQLite/FTS5 + Gateway + 二次开发（加 tool/provider/adapter/CLI） | ✅ |
| [附录 · 最佳应用场景与速查表](docs/技术学习/hermes-agent-framework/appendix-最佳应用场景与速查表.md) | 12 个场景（含配置片段）+ CLI/斜杠/配置/环境变量速查表 | ✅ |
| [合订版（历史归档）](docs/技术学习/hermes-agent-framework/01-三段式入门到深入教程.md) | 早期合订稿，已被新版取代，保留作归档 | 📦 |

---

### 投资教育 / 股票交易 — Adam Khoo 系列

| # | 文档 | 主题 | Ground Truth | 状态 |
|---|------|------|:---:|:---:|
| 1 | [Lesson 01 — 交易风格与风险管理](docs/投资教育/股票交易/adam-khoo-lesson01-交易风格与风险管理.md) | Position/Swing/Day Trading + 风险管理 | ✅ | 完成 |
| 2 | [Lesson 02 — 支撑位与阻力位](docs/投资教育/股票交易/adam-khoo-lesson02-支撑位与阻力位.md) | Support & Resistance Zones | ✅ | 完成 |
| 3 | Lesson 03 — 趋势的力量 | The Power of Stock Market Trends | ⏳ | 待字幕 |
| 4 | Lesson 04 — 技术指标 | Technical Indicators for High Probability Trading | ⏳ | 待字幕 |
| 5 | Lesson 05 — 日内交易 | Stock Day Trading for Daily Profits | ⏳ | 待字幕 |
| 6 | Lesson 06 — 成为盈利交易员 | What It Takes to Be a Profitable Trader Part 2 | ⏳ | 待字幕 |
| 7 | [Lesson 07 — 像赌场一样交易](docs/投资教育/股票交易/adam-khoo-lesson07-像赌场一样交易.md) | 统计优势 + 概率思维 | ✅ | 完成 |
| 8 | [Lesson 08 — 创建盈利交易系统](docs/投资教育/股票交易/adam-khoo-lesson08-创建盈利交易系统.md) | 系统设计 + 期望值 + 交易日志 | ✅ | 完成 |
| 9 | Lesson 09 — 仓位管理 | Position Sizing & Stock Trading Strategies | ⏳ | 待字幕 |
| 10 | Lesson 10 — 赢家的秘密 | Secret of Winning Stock and Forex Traders | ⏳ | 待字幕 |

**扩展课程**（播放列表共 38 个视频）:

| # | 视频标题 | 状态 |
|---|----------|:---:|
| 11 | Stock Investing Versus Trading | ⏳ |
| 12 | How to Read Level 2 Quotes & Bid Ask | ⏳ |
| 13 | Screening for Stocks using Thinkorswim Scanner | ⏳ |
| 14 | Stocks Trading vs Forex vs Options | ⏳ |
| 15-16 | Secrets to Profitable Stock Investing (Part 1 & 2) | ⏳ |
| 17 | A Night in the Life of Adam Khoo | ⏳ |
| 18 | Trade The Strongest Momentum Stocks | ⏳ |
| 19-20 | Are You a Trader or a Gambler? (Part 1 & 2) | ⏳ |
| 21-22 | 10 Golden Rules of Winning Traders (Part 1 & 2) | ⏳ |
| 23-24 | How I Scan for High Momentum Stock Trades (Part 1 & 2) | ⏳ |
| 25 | Interactive Brokers TWS Tutorial | ⏳ |
| 26-28 | Trading for a Living (Part 1-3) | ⏳ |
| 29-30 | Not Profitable in Trading? 7 Reasons (Part 1 & 2) | ⏳ |
| 31-32 | What It Takes to Be Profitable Trader (Part 1 & 2) | ⏳ |
| 33 | Five Power Candlestick Patterns | ⏳ |
| 34-35 | Trading Your Way to Financial Freedom (Part 1 & 2) | ⏳ |
| 36 | Using Stock Market Cycles | ⏳ |
| 37 | 3 Ways to Profit from the Stock Market | ⏳ |
| 38 | Placing Trade Orders on Interactive Brokers | ⏳ |

> ⏳ = 需要提供视频字幕/transcript 才能基于 Ground Truth 完成分析

---

## 🏷️ 标签系统

- `#stock-trading` — 股票交易
- `#risk-management` — 风险管理
- `#swing-trading` — 波段交易
- `#technical-analysis` — 技术分析
- `#support-resistance` — 支撑与阻力
- `#trading-psychology` — 交易心理学
- `#trading-system` — 交易系统
- `#probability` — 概率与统计
- `#backtesting` — 回测
- `#fundamental-analysis` — 基本面分析
- `#index-investing` — 指数投资
- `#macro` — 宏观经济
- `#hermes` — Hermes 模型系列
- `#agents` — AI Agent 构建
- `#function-calling` — 函数调用
- `#llm` — 大语言模型

---

## 📝 贡献指南

1. 新文档放入对应分类目录
2. 文件名格式: `来源-课号-主题关键词.md`
3. 每个文档顶部需有元数据（来源、作者、日期、标签、Ground Truth 状态）
4. 更新 README 索引表
5. **Ground Truth 原则**: 所有分析必须基于作者实际输出，标注数据来源

---

*Last updated: 2026-05-06 — Hermes Agent 框架三部曲全部完成（Part 1/2/3A/3B + 附录 A/B + 导航页）*
