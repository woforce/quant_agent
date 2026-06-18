# 投研选股 Agent 构建方案：综合调研报告

> 调研日期：2026-06-19  
> 覆盖范围：中英文网页 + GitHub 开源仓库 + 学术论文

---

## 一、投行/机构交易员如何选股

### 1.1 基本面分析：估值模型体系

机构的核心选股锚是 **DCF 估值**：
- 构建 5 年三表财务预测 → 计算无杠杆自由现金流 → 以 WACC 折现 → 加终值 → 得目标价
- DCF 总是与**可比公司分析**（EV/EBITDA、P/E、PEG）和**先例交易分析**交叉验证，形成"足球场图"估值区间

各投行有专有增强模型：

| 机构 | 模型 | 特点 |
|------|------|------|
| Credit Suisse | HOLT CFROI | 现金回报率框架，资产通胀调整 |
| Morgan Stanley | ModelWare | "盈利树"连接经营活动和内在价值 |
| UBS | VCAM + EGQ | ROIC 利差 × 投入资本增长识别低估/高估 |
| Deutsche Bank | Monte Carlo FCFF | 随机建模输出公允价值概率分布 |

**A 股实践（招商证券 FCF-ROE 框架）**：高 ROE + 高自由现金流占比的企业更容易超额收益，三波估值修复路径为：FCFR 改善 → 盈利企稳/ROE 提升 → 美债收益率下行。

来源：*Equity Valuation: Models from Leading Investment Banks* | [招商证券 FCF-ROE](https://stock.finance.sina.com.cn/stock/go.php/vReport_Show/kind/strategy/rptid/769856906967/index.phtml)

### 1.2 量化筛选：多因子模型

| 因子类别 | 典型指标 | 逻辑 |
|----------|----------|------|
| 价值 | PE、PB、FCF Yield | 捕捉低估 |
| 动量 | 6-12 月收益率、均线交叉 | 趋势持续 |
| 质量 | ROE、ROA、FCFR、盈利稳定性 | 盈利安全 |
| 规模 | 总市值 | A 股小盘效应显著 |
| 低波动 | 日/周收益率标准差 | 防御属性 |
| 成长 | 营收/净利润增速、盈余超预期 | 增长溢价 |

**关键实践**：
- **行业中性化**：截面回归去行业效应后再比较
- **因子检验标准**：IC > 0.05、t 值 > 2、ICIR 显著、分层回测单调
- **动态多因子**：基于滚动 IC 和拥挤度动态调整因子权重，显著优于静态等权
- **机器学习增强**：XGBoost/LightGBM 捕捉因子非线性交互，SHAP 值可解释；Oxford 的 GAT 图注意力网络利用分析师共同覆盖网络构建动量信号，年化收益 29.44%，Sharpe 4.06

来源：[MSCI Core Multi-Factor](https://www.msci.com/research-and-insights/paper/efficient-multi-factor-indexing-and-concentrated-markets-introducing-the-msci-core-multiple-factor-indexes) | [Oxford Analyst Network Alpha](https://ar5iv.labs.arxiv.org/html/2410.20597)

### 1.3 宏观择时与行业轮动

**美林时钟中国改良版**（货币+信用框架）：

| 周期 | 货币信用状态 | 推荐资产 | 正确率 |
|------|-------------|----------|--------|
| 衰退 | 宽货币+紧信用 | 债券 | 83% |
| 复苏 | 宽货币+宽信用 | 股票 | 100% |
| 过热 | 紧货币+宽信用 | 商品 | 57% |
| 滞胀 | 紧货币+紧信用 | 现金 | 43% |

**行业轮动信号**：EPFR 的 Flow Momentum (FloMo) —— 总美元流入/总 AUM，ETF 信号回看 30-110 天，主动基金约 70 天。

来源：[新浪改良时钟](https://finance.sina.cn/fund/jjgdxw/2021-03-08/detail-ikkntiak6358998.d.html) | [EPFR 行业轮动](https://isimarkets.com/quants-corner/epfrs-sector-rotation-strategy-taking-a-look-from-the-bottom-up/)

### 1.4 信息优势来源

- **卖方研究**：天风证券"四位一体"（政策+专家圈+调研问卷+数据科技）
- **专家网络**：按小时付费连接行业专家，交叉验证卖研，覆盖中小标的
- **另类数据**：信用卡消费、卫星图像、供应链数据、社交媒体情绪、期权数据
- **分析师共同覆盖网络**：构建股票-分析师二部图，信息溢出效应可作为动量增强信号

### 1.5 风险管理核心

**仓位公式**（Deep Learning A 股算法）：
```
wi = Score × sqrt(MarketCap) × Momentum^0.2 / (ADV^0.3 × Volatility^0.5) × λ
```

**止损机制**（中金公司多步止损）：嵌套多层安全垫，不同止损线对应不同安全资产比例。实证：4股6债组合叠加多步止损，累计净值从 1.94 提升至 2.93，最大回撤从 35.24% 降至 11.26%。

**退出纪律**（Resonanz Capital）：
1. 论点锚定点——入场时即归档退出条件
2. 回撤触发审查——20% 从高点回落 → 强制重新评估
3. "升级或退出"测试——如果今天不会买，就退出或换仓

来源：[中金公司详解止损机制](http://stock.finance.sina.com.cn/stock/go.php/vReport_Show/kind/lastest/rptid/774634811189/index.phtml) | [Resonanz Capital](https://resonanzcapital.com/insights/position-sizing-sell-discipline-a-modern-allocators-framework)

---

## 二、小型交易团队管理

### 2.1 三种主流组织架构

| 模式 | 结构 | 优势 | 劣势 | 代表 |
|------|------|------|------|------|
| **Multi-PM 制** | 多组独立 PM | 策略启动快，归属清晰 | 重复造轮子，赛马压力 | 鸣石早期 |
| **流水线制** | 因子→AI→优化→风控→交易 | 效率高，全局最优 | 工作"螺丝钉化" | 鸣石"五环多核" |
| **大课题-小项目制** | 负责人牵头大课题 + 个人小项目 | 兼顾深度和广度 | 需要强 PM | 量派投资 |

**3-10 人团队核心角色**：
- 策略研究员/PM：1-2 人
- 数据工程师：0.5-1 人
- 量化开发者：1-2 人
- 风控/交易执行：1 人（必须独立于投研！）
- 组合经理：由资深研究员兼任

来源：[量派投资孙林](https://www.cs.com.cn/sylm/jsbd/202601/t20260106_6531939.html) | [启林投资王鸿勇](https://finance.eastmoney.com/a/202604153704938191.html)

### 2.2 标准研发流程（七阶段）

```
①策略构思 → ②数据获取与预处理 → ③策略编写与回测
    → ④策略评估与筛选 → ⑤模拟盘测试(1-3月)
    → ⑥小资金实盘(10%-20%资金) → ⑦实盘监控与迭代
```

**AI 增强工作流**（Jonathan Kinlay 四 Agent 架构）：
- **Proposer**：发可证伪假说（无代码执行权，无权访问价格数据）
- **Implementer**：将获批假说写成 notebook（无权查看之前结果，防锚定偏差）
- **Critic**：对抗性审查（只能记录问题不能修复，25 个预设缺陷 notebook 中达 80% 捕获率）
- **Replicator**：独立重新实现，Ablation 测试

**关键效果**：12 周内通过人类审查的策略数翻倍，假说测试量从 11 增至 38（3.5 倍），假说到第一次回测从 ~2 天缩短至 ~3 小时。

来源：[Jonathan Kinlay](https://jonathankinlay.com/2026/05/agentic-workflows-for-alpha-research/) | [叩富网](https://licai.cofool.com/user/guide_view_3385709.html)

### 2.3 决策框架：三种模式对比

| 维度 | PM 驱动 | 模型驱动 | 委员会制 |
|------|---------|---------|---------|
| 决策主体 | 个人 PM | 模型输出 | 多角色辩论+PM 审批 |
| 关键人风险 | 高 | 低 | 中 |
| 问责难度 | 低 | 高（易"决策洗白"） | 中 |
| 创新性 | 高 | 受框架限制 | 结构化创新 |

**AUIM 案核心教训**：模型驱动 ≠ 免除问责。必须**文档化 PM 自由裁量权的边界**——什么情况下可以推翻模型信号，如何记录，谁来监督。

**推荐混合模式**：
- 核心信号由模型生成，PM 保留有边界的人为否决权
- 建立"红色团队"：定期指定一人专门"杀死当前最佳策略"
- 投决过程全程留痕：模型输出、PM 修改理由、事后归因均记录

来源：[FinCom arXiv](https://arxiv.org/html/2606.00939v1) | [TradingAgents](https://www.e-com-net.com/article/2051606700095496192.htm)

### 2.4 激励机制设计

**行业薪酬结构**：
- 月薪 3-8 万，年终奖 8-20 个月工资
- 超额收益提取 20% 作为业绩报酬
- 研究员"推票奖金池"按 PM 打分权重分配
- 头部私募将 10% 净利润纳入股权池，分三年解锁

**核心原则**：短中长期激励配合 + 递延发放 + 清晰透明的分润公式

来源：[每日经济新闻](https://finance.eastmoney.com/a/202601083613058512.html)

### 2.5 风险控制体系

**三道防线**：

| 防线 | 机制 | 要点 |
|------|------|------|
| 事前（准入） | 极端压力测试 | 2008/2015/2018/2020 行情+放大交易成本，"坚决不上"未通过策略 |
| 事中（运行） | 行为监控 | 频繁撤单/指令骤增 = 策略失效早期信号；单票≤15%、日内回撤熔断 3% |
| 事后（复盘） | 亏损复盘 | "不是追责，是为升级系统"；建立策略"病历本" |

**每个策略的"体检报告"**追踪三个生命体征：
- **波动率体温**：突然放大 = 策略"生病"
- **相关性血压**：与基准相关性异常跳升 = 暴露于未预料的单一风险
- **容量心率**：管理规模增加后业绩稳定性

**灵均投资教训**（2024年2月被限制交易）：投研与风控由同一合伙人统管，当进攻性与风控冲突时无人把关。改革：风控从投研体系中独立，提升至公司级战略位置。

来源：[BigQuant 风控框架](https://mf.bigquant.com/wiki/doc/4B5cBW5H2S) | [灵均投资](https://paper.cnstock.com/html/2026-03/18/content_2189397.htm) | [SOA Lean MRM](https://www.soa.org/communities/investment-and-risk-management/newsletter-articles/2026/january/2026-01-ir-levine/)

### 2.6 常见陷阱 TOP 10

1. **过拟合** — 回测年化 80%，实盘变"渣"
2. **拥挤交易** — 大量产品集中同一赛道
3. **跳过模拟盘** — 直接回测→实盘
4. **风控与投研同一人管理** — 冲突时无人把关
5. **忽视交易成本** — 回测不包含佣金、滑点
6. **止损逻辑死循环** — 止损后仍满足买入条件再次买入
7. **并发发单触发流控** — 大面积废单
8. **情绪驱动过度优化** — 回撤后"狗尾续貂"
9. **规模与策略不匹配** — 冲击成本失控
10. **模型驱动但无人问责** — PM 干预边界未文档化

**应对：设立"冷静日"**——较大回撤时研究员当天禁止提交代码。

---

## 三、如何构建投研选股 Agent

### 3.1 标杆开源项目全景

#### 国际项目

| 项目 | Stars | 核心架构 | 技术栈 | 仓库 |
|------|-------|---------|--------|------|
| **TradingAgents** | 87k+ | 9 Agent 分工（分析师→多空辩论→交易员→风控→PM） | LangGraph + 12+ LLM | [链接](https://github.com/TauricResearch/TradingAgents) |
| **AI Hedge Fund** | 60k+ | 6 分析师 + 12 投资大师人格 | Python + LangGraph | [链接](https://github.com/DanisHack/ai-hedge-fund) |
| **FinRobot** | 4k+ | 多层架构：AI Agent 层→LLM 算法层→LLMOps | LangChain | [链接](https://github.com/AI4Finance-Foundation/FinRobot) |
| **Swarm Trader** | — | 20 个 Agent（13 LLM 人格 + 7 数据专家） | 13 种 LLM + Alpaca | [链接](https://github.com/zhound420/swarm-trader) |
| **QuantAgent** | — | 4 Agent（Indicator/Pattern/Trend/Decision） | LangChain + LangGraph | [链接](https://github.com/irsath/quantagent) |
| **Orallexa** | — | 8 源信号 + Bull/Bear/Judge 辩论 + 20 Agent MC 收敛 | Claude Opus 4.7 | [链接](https://github.com/alex-jb/orallexa-ai-trading-agent) |

#### A 股项目

| 项目 | 核心技术 | 特点 | 仓库 |
|------|---------|------|------|
| **AlphaSift** | LLM 三层漏斗 + 多源降级（Tushare→新浪→AKShare） | Apache 2.0，A 股全市场筛选 | [链接](https://github.com/ZhuLinsen/alphasift) |
| **Fin-Agent** | DeepSeek + Tushare + Function Calling | pip 安装，CLI 交互 | [链接](https://github.com/YUHAI0/fin-agent) |
| **EasyQuant** | LLM Agent + 聚宽回测 + Playwright 自动化 | 28 个已验证策略库 | [链接](https://github.com/HiRenyi/EasyQuant) |
| **AI Stock V3.0** | Tushare + DeepSeek/GLM/Qwen/Kimi | 端到端全流程+飞书通知 | [链接](https://github.com/bazingamc/AI_Stock_V3.0) |

#### 学术前沿

| 项目/论文 | 核心创新 | 关键结果 |
|-----------|---------|---------|
| **QuantAgents** (EMNLP 2025) | 4 Agent + 双重奖励 | NASDAQ-100 3年回报~300% |
| **GuruAgents** (CIKM '25) | 5 投资大师人格 + 确定性推理 | 巴菲特 Agent 42.2% CAGR |
| **AlphaAgents** (BlackRock) | 3 类 LLM Agent + AutoGen 多轮辩论 | 基本面/情绪/估值融合 |
| **Mass** (北大+正仁量化) | 512 Agent 大规模模拟 | Agent 数量指数增长超额收益更显著 |
| **ATLAS** | 动态 Prompt 优化 | 跨 LLM 家族均优于固定 Prompt |

**论文追踪**：[Awesome-LLM-Quantitative-Trading-Papers](https://github.com/Tom-roujiang/Awesome-LLM-Quantitative-Trading-Papers)

### 3.2 推荐架构：分层流水线

```
┌─────────────────────────────────────────┐
│         数据获取层 (Data Layer)           │
│  yfinance / Tushare / AKShare / EDGAR   │
│  + 新闻 API / 另类数据                    │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│      分析师团队 (Analyst Team) 并行       │
│  ┌────────┐ ┌────────┐ ┌──────┐ ┌────┐  │
│  │基本面   │ │技术    │ │情绪  │ │宏观│  │
│  │Agent    │ │Agent   │ │Agent │ │Agent│ │
│  └────────┘ └────────┘ └──────┘ └────┘  │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│      对抗式辩论 (Bull vs Bear Debate)     │
│   多空研究员多轮结构化论战 + 证据引用       │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│      交易员/基金经理 (Trader/PM)          │
│   综合报告 → BUY/SELL/HOLD + 仓位大小     │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         风控层 (Risk Gate)               │
│   暴露度/止损/相关性检查 → 最终否决权     │
└─────────────────────────────────────────┘
```

### 3.3 三种 LLM-量化结合模式

| 模式 | 代表 | LLM 角色 | 优势 | 劣势 |
|------|------|---------|------|------|
| **A: 分析型** | AI Hedge Fund 规则模式 | LLM 只做分析，规则引擎决策 | 幻觉风险可控 | 未释放 LLM 能力 |
| **B: 全代理型** | TradingAgents, TradingGroup | LLM 驱动全流程 | 端到端推理完整 | 成本高、需要精心 Prompt |
| **C: 混合型**（推荐） | AlphaSift, EasyQuant | 传统量化大规模筛选 + LLM 精筛 | 务实、高效 | 需要设计两层接口 |

**推荐混合型**：先用传统多因子模型从全市场筛出 Top 50-100，再由 LLM Agent 对候选池做深度分析和排序。

### 3.4 决策融合：多 Agent 意见如何聚合

| 方法 | 效果 | 代表 | 原理 |
|------|------|------|------|
| 置信度加权投票 | 优于简单投票 | Orallexa | Agent 输出置信度，按置信度加权 |
| 多轮辩论至共识 | 深层推理 | TradingAgents | Bull/Bear 多轮辩论，Judge 仲裁 |
| 元分类器 | **最优** | Kirtac et al. (UCL) | XGBoost/LogReg 学习"何时信任哪个 Agent" |
| Shapley 动态权重 | 适合非平稳市场 | MRC | 计算每个 Agent 在所有联盟中的边际贡献 |

**关键发现**：
- **Agent 多样性比单个强度更重要** — 融合 3 个不同 Agent 显著优于最强单 Agent
- **市场状态感知** — 牛市时基本面 Agent 权重更高，波动市时技术 Agent 权重更高
- **分歧是信号，不是问题** — 高冲突场景下聚合增益最大

**推荐融合架构**：
```
原始信号池 → 质量过滤(丢弃低置信度) → 辩论/对齐(冲突信号→多空辩论)
    → 元聚合器(XGBoost学习"何时信任谁") → 风控门控(有否决权) → 最终决策
```

### 3.5 数据源方案

#### A 股推荐组合（三足鼎立）

| 数据源 | 定位 | 费用 | 获取 |
|--------|------|------|------|
| **Tushare Pro** | 专业深度数据 | 免费积分有限 | tushare.pro 注册 |
| **AKShare** | 覆盖面最广（100+ 接口含舆情） | 完全免费 | pip install akshare |
| **Baostock** | 稳定备选 | 免费 | baostock.com |

**多源降级策略**（AlphaSift 方案）：
```
全市场快照: tushare → sina → efinance → akshare_em → em_datacenter
日K线增强:  tushare → tencent → akshare → baostock
全部失效 → 回退缓存数据（标注 stale 和 fallback）
```

#### 新闻/情绪数据

| 数据源 | 覆盖 | 适用场景 |
|--------|------|---------|
| StockData.org | 全球 5000+ 信源，实体级情绪 | 量化情绪因子 |
| Finnhub | 美股+加密，NLP 情绪 | 实时新闻情绪 |
| AKShare 新闻接口 | A 股东方财富/新浪等 | A 股舆情 |
| Tiingo News | 美股，专为算法情绪设计 | 量化情绪因子 |

### 3.6 编排框架与回测方案

**编排框架选型**：推荐 **LangGraph**（生态最大、案例最多、支持条件分支和循环）

**回测框架**：

| 框架 | 速度 | 适用场景 |
|------|------|---------|
| VectorBT | ★★★★★ | 大规模向量化回测，12秒/500只×10年 |
| Zipline-Reloaded | ★★★☆☆ | 学术因子研究、事件驱动 |
| Backtrader | ★★☆☆☆ | 学习与中频策略，中文教程丰富 |

**A 股回测**：Backtrader + AKShare/Tushare 数据，或 EasyQuant 的 Playwright 自动化对接聚宽平台。

### 3.7 Prompt 工程最佳实践

| 模式 | 技术 | 来源 |
|------|------|------|
| **角色扮演** | "You are Warren Buffett, a value investor..." | AI Hedge Fund, GuruAgents |
| **思维链 (CoT)** | "Let's think step by step... First, analyze financials..." | TradingAgents |
| **结构化输出** | JSON 格式：`{"score": 7.5, "reasoning": "...", "risks": [...], "evidence": [...]}` | TradingAgents, AlphaSift |
| **对抗式辩论** | "You are the BULL analyst. Now debate the BEAR..." | TradingAgents, Orallexa |
| **RAG 增强** | 嵌入财报/新闻/法规段落作为上下文 | Stock Picker Agent |
| **动态 Prompt 优化** | 根据实时市场反馈自动调整 Prompt | ATLAS |

**关键安全警示**：LLM Agent 存在"推荐漂移"——当工具输出含有篡改数据时，Agent 在 80% 案例中忠实复述错误数据（"Sell Me This Stock" arXiv:2603.12564）。缓解：多数据源交叉验证 + RAG 可追溯引用 + 风控 Agent 独立验证。

### 3.8 实践路线图

| 阶段 | 周期 | 核心任务 |
|------|------|---------|
| **一：基础设施** | 1-2 周 | 选数据源（AKShare+Tushare / yfinance+StockData）；选编排（LangGraph）；选 LLM（至少 2-3 提供商）；选回测（VectorBT） |
| **二：单 Agent 验证** | 2-3 周 | 实现基本面+技术+情绪 Agent；独立回测；结构化输出稳定；幻觉检测机制 |
| **三：多 Agent 协作** | 2-4 周 | 并行分析流程；Bull/Bear 辩论；决策融合；风控门控 |
| **四：迭代优化** | 持续 | 长期记忆系统；自我反思机制；动态 Prompt 优化；扩展另类数据；实盘部署 |

**学习路径**：TradingAgents → AlphaSift（A 股） → AI Hedge Fund → GuruAgents → ContestTrade

---

## 核心结论

1. **选股方法可以系统化**：投行的基本面 + 量化 + 宏观三层体系完全可以抽象为 Agent 工作流，传统多因子模型做粗筛、LLM Agent 做精筛和分析是目前最务实的混合路径。

2. **团队管理的关键**：风控必须独立于投研（灵均教训）；架构设计比模型选择更重要（Kinlay）；设立"冷静日"和"红色团队"对抗确认偏差和情绪驱动决策。

3. **Agent 构建的核心设计决策**：
   - 分层流水线架构（数据层→分析师层→辩论层→决策层→风控层）
   - 混合模式：传统量化 + LLM，前者做海选，后者做精筛
   - 多 Agent 意见融合时，元分类器（XGBoost 学习"何时信任谁"）优于简单投票
   - 必须有独立的风控门控层，拥有最终否决权
   - 多数据源交叉验证 + 降级链是数据可靠性的保障

4. **"架构是承重部件——不是提示词，不是模型选择"**：角色分离（Proposer/Implementer/Critic/Replicator）、类型化交接、不可变日志——这些结构化设计比任何单一工具的先进程度更重要。
