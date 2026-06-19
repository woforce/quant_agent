# Claude Code + Skills + Loops 项目的可维护性与可审计性模式

> 调研范围：3 个平行 agent，覆盖 Claude Code 生态、软件工程/医疗/法律 agent 系统、可观测性/审计工具链

---

## 一、目录结构：你应该怎么组织 `.claude/`

```
quant-agent/
├── CLAUDE.md                          # 项目级规范 (< 200 行，@import 分包)
│
├── .claude/
│   ├── settings.json                  # 权限、hooks、环境变量
│   │
│   ├── rules/                         # 自动加载的行为规则
│   │   ├── data-integrity.md          # 无 paths → 每次 session 加载
│   │   └── a-share-conventions.md     # paths: "data/a_share/**" → 按需加载
│   │
│   ├── skills/                        # 投研角色 Skill
│   │   ├── fundamental-analyst/
│   │   │   ├── SKILL.md               # 必须：metadata + 指令
│   │   │   ├── scripts/               # 辅助脚本
│   │   │   └── references/            # 按需加载的参考文档
│   │   ├── sentiment-analyst/
│   │   ├── macro-analyst/
│   │   ├── bull-researcher/
│   │   ├── bear-researcher/
│   │   └── portfolio-manager/
│   │
│   ├── commands/                      # 手动 slash 命令
│   │   ├── daily-scan.md              # /daily-scan → 一键跑全流程
│   │   └── review-position.md         # /review-position 茅台
│   │
│   ├── agents/                        # 专用 sub-agent 定义
│   │   ├── data-validator.md          # 独立验证数据质量
│   │   └── hallucination-checker.md   # 检查其他 agent 的输出
│   │
│   └── hooks/                         # 或内联在 settings.json
│       └── validate-output.sh         # 确保 JSON schema 合规
│
├── docs/
│   ├── skills-index.yaml              # 权威 Skill 注册表（机器可读）
│   └── audit-trail/                   # 决策日志
│
└── data/
    ├── raw/                           # 原始数据
    └── cache/                         # 缓存
```

### 核心原则

- **CLAUDE.md < 200 行**：超过这个长度 Claude 会开始选择性忽略。用 `@import docs/xxx.md` 分包
- **Skill 一个只做一件事**：如果 SKILL.md 超过 200-500 行，拆成多个
- **`skills-index.yaml` 是唯一权威注册表**：README、文档、CLAUDE.md 都从它自动生成

---

## 二、CLAUDE.md：装什么、不装什么

### 应该装

- 构建/测试命令（Claude 无法从 pyproject.toml 猜到的）
- **非默认**的代码风格和架构约定
- 项目特有的坑（"fundamentals_agent 不要直接信 yfinance 返回的 ROE"）
- 策略迭代纪律（"每次修改因子权重后必须先跑 backtest.py"）

### 不应该装

- Claude 自己能读懂的（package.json、pyproject.toml 里的命令）
- 标准语言惯例（PEP 8 不需要写）
- **经常变的东西**——过时的指令比没有更糟
- 格式化规则——用 hooks 强制（hooks 是确定性的，CLAUDE.md 是建议性的）

### 自检

对每一行问：**"删掉这行，Claude 会犯错吗？"** 不会就删。

---

## 三、Skill 设计：三个跨领域通用原则

### 3.1 Progressive Disclosure（渐进披露）

元数据先加载 → SKILL.md body 在触发时加载 → references 条件性加载（只加载当前任务相关的） → scripts 按需执行

这样即使有 10+ Skill，每次 session 的 token 开销也是可控的。

### 3.2 数据契约（Inter-Skill Data Contracts）

每个 Skill 的输出必须定义 JSON Schema。Skill A 的输出 schema 必须和 Skill B 的输入 schema 兼容。

`tradermonty/claude-trading-skills` 有一个专门的 `skill-integration-tester`，自动验证跨 Skill 的 schema 兼容性。

### 3.3 分离生成与治理（FERZ 模式，来自法律 agent 系统）

```
LLM 生成提案 → 独立的确定性层验证 → 验证通过才放行
```

在你的场景里：
- 基本面 Agent 写分析 → **独立的 data-validator agent** 检查每个数字是否在数据源范围内
- 情绪 Agent 打分 → **独立的 hallucination-checker agent** 验证引用的新闻是否真实存在

**LLM 提案，框架裁决。**

---

## 四、审计：每个决策必须可追溯到源

### 4.1 最少必须记录的内容

对投研 Agent 的每个决策，日志里必须包含：

```
{
  "session_id": "2026-06-19-001",
  "timestamp": "...",
  "agent": "fundamentals-analyst",
  "ticker": "600519.SH",
  "decision": "茅台 ROE 32%，行业排名第 2，评级 Overweight",
  
  "data_sources": [
    {"source": "akshare stock_financial_abstract", "field": "roe", "value": 0.32, "as_of": "2025Q4"},
    {"source": "akshare stock_sector_pe", "field": "industry_median_roe", "value": 0.12}
  ],
  
  "tool_calls": [
    {"tool": "get_fundamentals", "args": {"symbol": "600519.SH"}, "duration_ms": 234}
  ],
  
  "alternatives_considered": ["Hold", "Buy"],
  "confidence": 0.85,
  
  "prompt_hash": "sha256:...",
  "model": "claude-sonnet-4-6",
  "temperature": 0.0,
  "seed": 42
}
```

这来自医疗系统的 **MEP (Model Evaluation Packet)** 模式、法律系统的 **Proof-Carrying Decisions** 模式、和金融审计的 **TraceLedger** 工具。

### 4.2 用 Git 做审计日志是可行的——但有条件

**适用**于你这场景（日频投研，不是高频交易）：

```
每次投研流程产出 → 
  JSON 决策日志 → 
    docs/audit-trail/2026-06-19/
      ├── fundamentals-report.json
      ├── sentiment-report.json
      ├── bull-bear-debate.json
      └── final-decision.json
  → git commit（带 hash chain 引用父决策）
```

Git 的 Merkle DAG 天然提供了防篡改、内容寻址、O(1) 快照恢复。

**不适用**于微秒级高频决策——那种需要专用 event store。

### 4.3 确定性：T=0 可以保证，但工程上不等于可复现

`temperature=0 + fixed seed` 在数学上保证了 greedy decoding 的确定性——给定相同的 prompt、相同的模型权重、相同的硬件，输出应该是确定的。这不是模型大小的问题。

但在工程实践中，"可复现"会遇到几个实际问题：

- **硬件层面**：GPU 浮点运算的非结合性可能导致 `matmul` 在不同 run 间有微小差异，这些差异经过多层 transformer 后可能被放大
- **批处理效应**：batch size 不同可能导致不同的内存布局和并行调度，影响浮点累加顺序
- **推理引擎差异**：同一个模型通过不同 provider（OpenAI API vs Azure vs 自部署）可能因为不同的优化路径产生不同输出
- **推理 token 被截断**：如果 provider 在返回的 response 对象中去掉了部分 reasoning tokens，prompt 上下文虽然相同但实际输入不完全一致

所以对你的项目：
- `temperature=0` 是必要的，但不保证跨 provider 可复现
- 用 `Pass^k` 指标——同一 provider 下 k 次运行应产出相同决策——来捕捉 provider 内部的不确定性
- 跨 provider 验证：同一个 prompt 在至少两个 provider 上跑，输出差异大 → 说明 prompt 本身不够稳定
- 关键决策的 prompt 应该是**确定性的函数调用格式**（JSON Schema），而不是自由文本推理

---

## 五、可维护性：是什么和不是什么

### 五个信号表明你的项目可维护

| 信号 | 含义 |
|------|------|
| **可以从日志重放任何一次过去的 agent 运行** | 你有 append-only event log |
| **加一个新 Skill 不需要改已有 Skill** | 松耦合，通过 JSON Schema 对接 |
| **不需要读对话记录就能理解发生了什么** | 结构化状态，不是隐式的聊天历史 |
| **每个循环有显式退出条件** | 不是只靠 `max_iterations` |
| **每个输出声明都能追溯到源数据** | source-level attribution |

### 五个信号表明项目要失控

| 信号 | 含义 |
|------|------|
| Agent 路由由 LLM 自己决定（不是框架决定） | 不可复现的 workflow |
| 状态活在非结构化的聊天历史里 | 无法审计 |
| 错误处理就是"让 LLM 再试一次" | 无限循环 + token 费爆炸 |
| 生成的代码跑在宿主机文件系统上 | 安全灾难 |
| 你回答不了"agent 做那个决策时看到了什么数据" | 审计失败 |

---

## 六、人机接口：把"展示什么"当成设计问题

从医疗系统的 GRADE 证据分级、法律系统的 HalluGraph 实体溯源、软件工程系统的 Aider 架构/编辑分离中得到启发：

### 不要展示的

- ❌ AI 的最终结论（避免锚定效应）

### 要展示的

- ✅ 原始数据（可点击追溯到源）
- ✅ 异常值高亮（"这个 ROE 比上季度跳了 40%，为什么？"）
- ✅ 多空分歧热力图（哪些维度共识，哪些在打架）
- ✅ 历史相似场景（上次这类模式出现时，后续 30 天收益分布）
- ✅ AI 草拟的多空论据（**不标注"AI认为哪边对"**——让人自己判断）

### 然后在人形成判断后

> "我倾向于做多，请用反方视角挑战我的逻辑。"

这是认知工程——**信息架构设计决定了决策质量**，比模型选择更重要。

---

## 七、从 0 到 1 的搭建顺序

```
Phase 1（本周）:
  ├── 写好 CLAUDE.md（< 200 行，只写非默认规范）
  ├── 建好 .claude/ 目录结构
  ├── 写 1 个 Skill（fundamental-analyst）跑通
  └── 配置 data-validator agent 做数字范围检查

Phase 2（下周）:
  ├── 加 3 个 Skill（sentiment / macro / portfolio-manager）
  ├── 建立 skills-index.yaml 注册表
  ├── 每个 Skill 的 JSON output schema 确定
  └── 写 hallucination-checker agent

Phase 3:
  ├── 加 Bull/Bear 辩论 Skill（带回合协议）
  ├── 搭建 git-based 决策日志
  ├── 配置 hooks 做 JSON schema 验证
  └── 接入 LangFuse 做 trace

Phase 4:
  ├── 人类审查接口（分歧热力图 + 异常高亮）
  ├── 可证伪条件自动回溯
  └── Pass^k 确定性测试
```

---

## 参考来源

- [code.claude.com/docs/en/claude-directory](https://code.claude.com/docs/en/claude-directory) — 官方 .claude/ 目录文档
- [tradermonty/claude-trading-skills](https://github.com/tradermonty/claude-trading-skills) — 40+ Skills with canonical index
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) — Command→Agent→Skill 架构
- [OpenHands event stream architecture](https://arxiv.org/abs/2407.16741) — append-only event log
- [FERZ Proof-Carrying Decisions](https://zenodo.org/records/18072966) — separation of intelligence from governance
- [IBM Output Drift research](https://github.com/ibm-client-engineering/output-drift-financial-llms) — T=0 not enough
- [TraceLedger](https://pypi.org/project/traceledger/0.1.0/) — SHA-256 hash-chained decision logging
- [GridSeal](https://www.npmjs.com/package/@gridseal/core) — 24-field proof chain entries
- [ICLR 2026 Replayable Financial Agents](https://ar5iv.labs.arxiv.org/html/2601.15322) — Pass^k metric, bi-temporal audit
- [Agentic AI Frameworks survey](https://ar5iv.labs.arxiv.org/html/2508.10146) — LangGraph/CrewAI/AutoGen 对比
