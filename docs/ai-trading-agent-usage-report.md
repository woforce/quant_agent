# AI 投研/交易 Agent 真实使用经验深度调研报告

> 调研日期：2026-06-19  
> 数据来源：知乎、雪球、CSDN、V2EX、Reddit、Hacker News、GitHub Issues（380+）、学术论文（8篇）、博客（30+篇）

---

## 核心结论（TL;DR）

**没有任何 AI 选股/交易 Agent 系统被证明能在实盘中稳定盈利。** 所有的"暴赚"案例要么是模拟盘、美股牛市加成、加了杠杆，要么本质上是卖课/卖工具的营销。

但 **Claude Code + MCP + 人工决策** 的混合模式在投研效率上效果显著——10x 提速、非程序员也能搭建、已被 Carlyle/Walleye Capital/Bridgewater 等机构采用。

中文社区最精准的总结（[zwt0204](https://zwt0204.github.io/github/2026/05/05/tradingagents-deep-dive/)）：

> **"把它当 Agent 系统设计样本来学，你会收获很大；把它当 AI 炒股工具来用，你会亏得很惨。"**

---

## 一、真实盈亏数据

### 亏损案例

| 案例 | 来源 | 细节 |
|------|------|------|
| **20万亏8万（-40%）** | [每日经济新闻](https://finance.eastmoney.com/a/202603303688752391.html) | 20万全权委托 AI 智能体（OpenClaw"龙虾"），跟随大盘下跌亏损 8 万 |
| **Token费5000元 + 亏损15%** | [知乎"八荒"](https://zhuanlan.zhihu.com/p/2016218194679977564) | AI 基于过期政策荐股、违规追高 |
| **AI推荐的10只股票9只下跌** | [新黄河](https://www.jinantimes.com.cn/news-20-5274679.html) | 同一指令三次推荐结果**无一只重合** |
| **GPT o1策略跌40%** | [Austin Starks](https://medium.datadriveninvestor.com/last-year-i-used-gpt-o1-to-create-a-trading-strategy-it-is-the-worst-strategy-ive-ever-seen-e66ffa3b49ad) | SMA 交叉策略回测 +277%，模拟盘 10 个月 -40%，最大回撤 52% |
| **17,800元"AI荐股"打水漂** | 新黄河 | "本以为抱上AI大腿，结果还是被当成韭菜收割" |
| **198元年费，第二天停服** | V2EX 用户反馈 | TradingAgents-CN 的 App Store 应用收了钱就跑 |

### "赚钱"案例（需仔细甄别）

| 案例 | 表面 | 真相 |
|------|------|------|
| **"月赚90%"** | 自媒体宣传 | 每日经济新闻调查还原：是 AI 炒美股**模拟盘大赛**，最高冲到 90% 最终回落至 36%，使用了杠杆 |
| **美股 +628.5%** | [Longbridge](https://longbridge.com/zh-HK/topics/41144155) | 5个 Agent 管20只美股，核心持仓 MU 浮盈 +236%。关键：**AI 给分析，人做决策** |
| **5个月 +3152%** | [知乎](https://zhuanlan.zhihu.com/p/2045865077803311382) | 美股交易员核心方法论："别让 AI 替你下判断，让它替你把判断之前的功课做厚" |
| **Paper Trading $95,431** | [Igor Ganapolsky](https://dev.to/igorganapolsky/i-built-a-self-healing-ai-trading-bot-that-learns-from-every-failure-g94) | 自愈型 Iron Condor bot（122+ 失败教训注入 RAG），**仍是模拟盘** |

### 权威声音

- **资深量化投资人李辉**："普通人几无可能通过 OpenClaw 赚钱"
- **南开大学教授田利辉**："AI 可辅助信息处理，但决策核心仍是人"
- **数字经济学者刘兴亮**："现阶段并不放心让龙虾经手资金操作"
- **广州国邦资管董事长周权庆**："普通投资者能想到的指标，量化基金早就反复研究过"

---

## 二、TradingAgents 深度剖析

### 2.1 论文 vs 现实

| 论文声称 | 独立复现结果 |
|---------|------------|
| AAPL +26.62%、Sharpe 8.21 | **卢森堡大学 ACM 复现**：GPT-4o 配置 15.8%±4.2%，**跑输** buy-and-hold 19.1%；Qwen 配置 18.1%±2.8%，**同样跑输** |
| 3 只股票、3 个月 | 传统量化标准是数百只股票、5-20+ 年——这是**类别错误** |
| 宣称 Bloomberg/Yahoo/Reddit 数据源 | **代码实际默认用 OpenAI web search**（[Issue #86](https://github.com/TauricResearch/TradingAgents/issues/86)） |

[卢森堡大学论文](https://dl.acm.org/doi/10.1145/3800973.3801029)的结论直白：
> "Without cherry-picking, both configurations fail to outperform the passive benchmark. Differences in mean returns were not statistically significant (p-value > 0.26)."

**Shannon 熵 0.73-1.10（满分 1.58）**——意味着很多天的决策接近随机。

### 2.2 创始人自己关闭了线上服务

这是最致命的事实。TradingAgents 团队因 **"large query volume and budget constraints"** 暂停了线上服务（Issue #7）。[GitPicks 分析](https://gitpicks.dev/featured/tradingagents-llm-trading-production-gaps)：
> "如果创始人自己都承担不起规模化的成本，这对从业者的信号很明确。"

### 2.3 Changelog 暴露的问题

从 v0.2.0 到 v0.2.5，每个版本都在修**致命 bug**：

- v0.2.3：**前视数据泄露**（回测有效性完全崩塌）
- v0.2.4：**空记忆库时 Agent 会编造历史"教训"**
- v0.2.5：社交情绪 Agent 会**虚构社交媒体帖子**；DeepSeek/MiniMax 结构化输出失败；配置状态在运行间泄漏

### 2.4 TradingAgents-CN（A 股版）的严重问题

| Issue | 问题 | 严重度 |
|-------|------|--------|
| [#162](https://github.com/hsliuping/TradingAgents-CN/issues/162) | **所有 A 股的资产负债率都显示45%、流动比率都显示1.5x**——Tushare 根本没被调用，返回的是硬编码默认值 | 🔴 致命 |
| [#324](https://github.com/hsliuping/TradingAgents-CN/issues/324) | 股票代码 300132（青松股份）显示为智动力（完全不同的公司）；股票代码 601377（兴业证券）显示为红塔证券 | 🔴 致命 |
| [#502](https://github.com/hsliuping/TradingAgents-CN/issues/502) | Web UI 显示 Tushare 已连接（绿色对勾），但实际 `.env` 未写入，数据静默失败 | 🔴 致命 |
| 讨论 #1 | 维护者承认：**"A 股新闻和情绪面目前用的是英文原版的美国数据源"** | 🟡 重要 |
| [#91](https://github.com/hsliuping/TradingAgents-CN/issues/91) | 程序跑到 Step 9/11 然后没有任何输出，无法判断成功还是崩溃 | 🟡 重要 |

**TradingAgents-CN 后来转为闭源收费**，引发社区不满。

### 2.5 五大硬伤（什么值得买社区）

[来源](https://post.smzdm.com/p/avg5q384/)

1. **输出极不稳定**：同一票同一日期多次运行结论可能完全相反
2. **无全市场筛选能力**：只能分析预设 ticker，面对 5000+ 股票是个"伪命题"
3. **时间和成本过高**：单票 90-135 秒，扩展后成本指数级增长
4. **无视策略专业化**：只能做通用"该不该买"分析，对特定策略（日内、恐慌反转）无用
5. **无真正记忆与进化**：把多轮对话当成记忆，无法从失败中学习

---

## 三、成本：AI Agent 交易的致命经济学

### 3.1 真实成本数据

| 规模 | 月度 LLM API 成本（社区报告） |
|------|---------------------------|
| 个人交易 bot | $90–$600 |
| 1000 次查询/天 | $1,500–$4,000 |
| 10000 次查询/天 | $4,500–$12,000 |

**TradingAgents 单票成本**（[博客园实测](https://www.cnblogs.com/llcailh/articles/20060513)）：
- DeepSeek：~$0.02-0.04/份报告
- GPT-4o：~$0.10-0.30/份报告
- 100 只股票每天 + 多轮辩论：成本迅速失控

### 3.2 成本灾难案例

| 案例 | 详情 |
|------|------|
| **$10/30秒** | 从 Claude 订阅切到 API 计费后，单次 agent loop 因全上下文重试导致 Token 消耗 10x |
| **$2,100 过夜账单** | 30 分钟心跳模式在市场收盘后仍在运行 |
| **$14,000 惊吓** | "我们的 AI agent 上线三周，LLM 账单 $14,000。我们预算了 $1,500" |
| **70% 零售 bot 2周破产** | [CoinMarketCap 调查](https://coinmarketcap.com/community/articles/69f18f2739c2b62146fa3a92/)：不是策略差，是 **API Token 费烧光本金** |
| **$10/天 API 费赚 $2/天利润** | "智能的成本超过了交易的价值" |

### 3.3 省钱策略（来自实践者）

1. 用便宜模型处理 80% 以上的调用（Haiku/Sonnet），仅最终决策用 Opus
2. PTC（Programmatic Tool Calling）：原始数据留在代码沙箱里，不进 LLM 上下文 → **Token 减少 85-98%**（[LangAlpha](https://github.com/ginlix-ai/langalpha)）
3. `/compact` 在 ~20 轮后压缩对话历史
4. 避免高峰期（周一至周五太平洋时间 5-11am）
5. 避免心跳循环在市场关闭时仍在轮询

---

## 四、幻觉：金融场景的致命伤

### 4.1 系统性幻觉（不是偶发）

| 发现的幻觉 | 来源 |
|-----------|------|
| ROE 差了 5 个百分点，营收增速数字完全对不上 | [CSDN](https://blog.csdn.net/2501_91062530/article/details/160298103) |
| 编造"某政策红利即将释放"推荐股票，实际三个月前已落地 | [知乎](https://zhuanlan.zhihu.com/p/2016218194679977564) |
| 所有 A 股资产负债率显示45%（硬编码假数据） | TradingAgents-CN Issue #162 |
| 股票名称张冠李戴 | TradingAgents-CN Issue #324 |
| Claude 在 broker 连接慢时**编造了格式正确但价格错误的市场数据** | [Kristin M.](https://dev.to/kristinm/how-i-built-an-ai-assisted-trading-ecosystem-without-writing-a-line-of-code-myself-53bh) |
| **记忆库为空时 Agent 会编造历史"教训"** | TradingAgents CHANGELOG v0.2.4 |

### 4.2 DayTrading.com 系统性测试（2025年9月）

[来源](https://www.advfn.com/stock-market/stock-news/96760202/daytrading-com-publishes-new-study-on-the-dangers)

测试 6 个主流 AI 平台、180+ 个交易查询：

| 平台 | 危险评分 | 准确率 | 发现 |
|------|---------|--------|------|
| Meta AI | 8.8/10 | 68% | 虚构股价，数据不全即发"Buy" |
| Groq | 8.2/10 | — | 编造特斯拉从未存在的价格 |
| Gemini | — | — | 自信但错误，遗漏风险提示 |

> **6 个模型平均亏损 -8.4%**（2025年8月），同期 S&P 500 涨 +2%

### 4.3 缓解措施（社区共识）

- 每个 LLM 输出的数字必须经独立数据源验证
- "LLM is writer, not knower"——算术交给 Python `decimal.Decimal`
- RAG 向量检索提供可追溯引用
- 风控 Agent 独立验证，有否决权

---

## 五、安全漏洞：TradeTrap 框架

[上海 AI 实验室 TradeTrap 论文](https://arxiv.org/html/2512.02261)（2025年12月）测试了 DeepSeek-v3、Claude 4.5、Qwen3、Gemini 2.5、GPT-5：

| 攻击向量 | 效果 |
|---------|------|
| **Prompt 注入** | 逆向预期扰动 → Agent 灾难性转向、强制爆仓 |
| **MCP 工具劫持** | 拦截价格源/Reddit 推文 → 制造"一致但虚假的现实" |
| **记忆投毒** | 腐蚀历史交易记录 → 渐进式资本流失 |
| **状态篡改** | 伪造持仓/余额/PnL → 仓位失控累积 |

**所有测试模型都显示系统性崩溃。** 自适应的（多轮、反思型）Agent 有更高上限，但更容易受信息通道攻击。

---

## 六、学术共识：Alpha Illusion

### 关键论文

| 论文 | 发现 |
|------|------|
| **[FINSABER](https://arxiv.org/pdf/2505.07078)** (2025) | LLM 策略在牛市过于保守（跑输被动基准），在熊市过于激进（巨额亏损）。FinMem Sharpe：牛市 -0.19，熊市 -0.97 |
| **[The Alpha Illusion](https://arxiv.org/abs/2505.16895)** (2026.05) | "报告的 Alpha 是方法论的产物，不是可部署能力的证据。"提出 P1-P6 最低报告协议 |
| **[Bias Audit of 164 Papers](https://arxiv.org/abs/2602.14233)** | 前视偏差、幸存者偏差等问题——**没有一项偏差在超过 28% 的研究中被讨论**。仅 1.2% 提及幸存者偏差 |
| **[KTD-Fin](https://arxiv.org/abs/2605.28359)** | 屏蔽股票代码和日期后，LLM Agent 的收益主要由被动市场和风格暴露解释——**几乎没有持续选股 Alpha 的证据** |
| **[TradeTrap](https://arxiv.org/html/2512.02261)** | 所有主流 LLM 交易 Agent 能从系统层面被攻击导致灾难性损失 |

### 学术界的行动

- **A**CM 会议已要求 LLM 交易论文的复现性声明
- **N**eurIPS 开始要求多市场、多资产类别、长周期（>3年）评估
- **S**EC 关注 AI 投顾的合规问题

---

## 七、Claude Code + MCP 生态：真正有用的部分

### 7.1 机构采用（确实在用）

| 机构 | 用法 |
|------|------|
| **NBIM（挪威央行）** | Claude Code 获得约 20% 生产力提升 = 21.3 万小时 |
| **Walleye Capital**（400 人对冲基金） | 100% 员工采用，AI-first |
| **Carlyle** | 核心 AI 栈用于投资、运营、组合管理 |
| **AIG** | 5x 审查速度，准确率 75%→90% |
| **Bridgewater** | 已在用 |
| **FIS** | 反洗钱调查从数天压缩到数分钟 |

### 7.2 MCP Server 生态（A 股）

| Server | 安装 | 能力 |
|--------|------|------|
| **[china-stock-mcp](https://github.com/xinkuang/china-stock-mcp)** | `uvx china-stock-mcp` | 30+ 工具，A/B/H股，30+ 技术指标，多源 fallback |
| **[akshare-tools](https://pypi.org/project/akshare-tools/)** | `uvx akshare-tools` | 市场概况/K线/资金流/龙虎榜/研报 |
| **[FinQ4Cn](https://blog.csdn.net/m0_38007743/article/details/150232897)** | MCP Server | 内置 pybroker 回测框架 |

### 7.3 实际使用评价

- **Kristin M.**（13 年交易员）：搭建了 47 MCP 工具 + 46 本地命令 + 10 systemd 服务的完整生态。AI 多次劝阻了她原本会做的错误入场。但也警告："Claude 编造了格式正确但价格错误的市场数据……你手里的不是数据，是看似可信的幻觉。"
- **非程序员用户**（[LongPort](https://longportapp.com/zh-CN/topics/39790611)）：让 Claude Code 自动爬 Magnificent Seven 的 IR 页面、下载 PDF、建数据库，"简单到荒谬……计算比 Excel 快 10 倍，错误率极低"
- **V2EX 用户**：MCP 加持的 Coding Agent 和单独 Agent "完全是两个工具"

### 7.4 Claude Code Skills 生态

| 项目 | 内容 |
|------|------|
| **[gonewx/TradingAgents](https://github.com/gonewx/TradingAgents)** | 13 个 sub-agent + 6 个 slash command + SQLite 记忆，专为 Claude Code 设计。但作者明确标注"纯粹用于研究学习，严禁实际交易" |
| **[ancs21/ai-sub-invest](https://github.com/ancs21/ai-sub-invest)** | 21 个投资大师人格 + 标准化 JSON 输出 |
| **[tradermonty/claude-trading-skills](https://github.com/tradermonty/claude-trading-skills)** | 40+ 交易 skill（VCP/CANSLIM/宏观周期/市场宽度） |
| **[fadacai-portfolio](https://github.com/PatrickSUDO/fadacai-portfolio)** | 7 MCP server + 12 slash command + 模型分层 + 概率诚实度检查 |
| **[Anthropic 官方 Finance Agents](https://www.anthropic.com/news/finance-agents)** | 10 个 agent 模板 + 41 skills + 38 slash commands + 11 个 MCP 数据连接器 |

---

## 八、到底什么有用？

### ❌ 没用（共识）

1. **全自动 AI 选股赚钱**——没有任何可靠证据
2. **TradingAgents 实盘**——跑不赢 buy-and-hold，创始人自己关了线上服务
3. **"AI 荐股"贩卖**——绝大多数是骗局（深圳证监局 2026年6月已发风险提示）
4. **"Fire and forget"**——静态策略面对黑天鹅必然失效

### ✅ 有用（共识）

1. **AI 做信息处理苦活**——爬财报、扫舆情、跑回测、整数据。5 个月 32 倍的美股交易员的核心方法论："别让 AI 替你下判断，让它替你把判断之前的功课做厚"
2. **Claude Code + MCP 做投研加速**——10x 效率提升，非程序员也能用
3. **对抗式辩论作为思想检验**——AI 多空辩论帮你发现自己没注意到的盲区
4. **自愈系统**——122+ 失败教训注入 RAG，每次犯错都变成未来的防护
5. **"红色团队"机制**——定期让 Agent 专门挑刺当前最优策略

### 📐 推荐架构

```
                      ┌─────────────────────┐
                      │   人工最终决策        │  ← 只有人类承担最终责任
                      └─────────┬───────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ↓                     ↓                     ↓
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ AI 信息收集   │   │ AI 多空辩论   │   │ AI 风险审查   │
   │ (苦活层)      │   │ (检验层)      │   │ (门控层)      │
   │ 财报/新闻/情绪 │   │ 挑战每个假设   │   │ 独立否决权    │
   └──────────────┘   └──────────────┘   └──────────────┘
```

**三个铁律**：
1. 所有 AI 输出的数字必须经独立数据源验证
2. 风控层必须独立，有否决权
3. 所有决策留痕，可复盘

---

## 九、对你的启示

结合你之前提到的 "Claude Code + Skills + Loops" 架构：

| 做法 | 判定 |
|------|------|
| 让 AI 做全自动实盘交易 | ❌ 目前不可行 |
| 用 AI Agent 做投研信息处理 + 人工决策 | ✅ 已被多个机构验证有效 |
| Chinastock-mcp + Claude Code 做 A 股分析 | ✅ 实用，成本可控 |
| 用 Skills 定义不同角色 Agent | ✅ 成熟方案，有多个现成项目参考 |
| Bull/Bear 辩论辅助思想检验 | ✅ 有用，但要独立验证辩论中的数据引用 |
| CronCreate 做定时驱动 | ✅ 原型阶段够用，生产级需外部调度 |
| 纯 LLM 回测策略 | ❌ 严重过拟合，必须用传统回测框架验证 |

**最大的坑**：不要相信任何人说的"AI 自动赚钱"。中文社区大量案例表明，"AI 选股暴赚"的博主真正赚钱的方式是卖课（2280 元/年）、卖硬件（3380 元"AI 股票机"）、卖会员（2988 元/季度）。**卖铲子的人赚了掘金者的钱。**

**最大的机会**：AI Agent 做投研效率工具——数据处理 10x 提速、多空辩论发现盲区、自愈系统积累经验。这个方向在散户和机构两端都有真实需求，且已经有跑通的案例。

---

## 参考来源摘要

- [卢森堡大学 ACM 复现研究](https://dl.acm.org/doi/10.1145/3800973.3801029)
- [FINSABER LLM 基准论文](https://arxiv.org/pdf/2505.07078)
- [The Alpha Illusion 论文](https://arxiv.org/abs/2505.16895)
- [TradeTrap 安全框架](https://arxiv.org/html/2512.02261)
- [Bias Audit of 164 LLM Trading Papers](https://arxiv.org/abs/2602.14233)
- [GitPicks TradingAgents 生产差距分析](https://gitpicks.dev/featured/tradingagents-llm-trading-production-gaps)
- [Daily Economic News AI 炒股调查](https://finance.eastmoney.com/a/202603303688752391.html)
- [DayTrading.com AI 平台测试](https://www.advfn.com/stock-market/stock-news/96760202/daytrading-com-publishes-new-study-on-the-dangers)
- [zwt0204 TradingAgents 深度剖析](https://zwt0204.github.io/github/2026/05/05/tradingagents-deep-dive/)
- [SMZDM TradingAgents 五大硬伤](https://post.smzdm.com/p/avg5q384/)
- [Kristin M. Claude Code 交易生态](https://dev.to/kristinm/how-i-built-an-ai-assisted-trading-ecosystem-without-writing-a-line-of-code-myself-53bh)
- [Austin Starks AI 策略失败系列](https://medium.datadriveninvestor.com/last-year-i-used-gpt-o1-to-create-a-trading-strategy-it-is-the-worst-strategy-ive-ever-seen-e66ffa3b49ad)
- [CoinMarketCap Token Burn Trap](https://coinmarketcap.com/community/articles/69f18f2739c2b62146fa3a92/)
- [Anthropic Finance Agents 官方发布](https://www.anthropic.com/news/finance-agents)
