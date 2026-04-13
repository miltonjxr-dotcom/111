# CBS 模块 AI 协作工作手册

> 实操指南: 如何利用 Gemini Pro / ChatGPT Plus / Cursor Claude Opus 4.6 高效落地 CBS 模块

---

## 1. 三模型定位总览

```
Gemini Pro       →  "眼睛" — 看世界、查数据、找信息
ChatGPT Plus     →  "大脑" — 想公式、做分析、定策略
Cursor Opus 4.6  →  "双手" — 写代码、搭架构、做集成
```

---

## 2. 逐阶段 AI 使用指南

### Phase 0: 基础建设

#### Task 0.1: 构建地址标签库

**Gemini Pro 执行:**

```markdown
Prompt 1 - 竞品调研:
"请详细对比 Arkham Intelligence, Nansen, Spot On Chain, Lookonchain,
DeBank 这5个平台的地址标签功能。包括:
1. 标签覆盖的区块链
2. 实体类型分类体系
3. API 可用性与定价
4. 数据更新频率
5. 标签准确度（如有公开评测）
请用表格对比，并给出 CBS 模块接入的优先级建议。"

Prompt 2 - 已知机构地址收集:
"请搜索以下 Web3 机构的已知链上地址:
[Jump Trading, Wintermute, a16z, Paradigm, Galaxy Digital,
Dragonfly, Polychain, Three Arrows Capital (历史)]
对于每个机构，列出:
- 已公开确认的地址
- 来源(Etherscan标签/Arkham/链上溯源)
- 地址类型(热钱包/冷存储/多签/交易执行)
- 最后活跃时间"
```

**ChatGPT Plus 执行:**

```markdown
Prompt - 质量评分模型:
"作为量化分析师，请设计一个地址质量评分模型。要求:

输入变量:
- 历史交易胜率 (win_rate)
- 平均每笔盈利 / 平均每笔亏损 (profit_loss_ratio)
- 总管理资产 (AUM)
- 近90天活跃度 (active_days_90)
- 标签来源数量 (label_sources)
- 信息先行度 (先于价格变动的次数 / 总交易次数)

请:
1. 定义归一化方法 (各变量单位不同)
2. 建议初始权重分配
3. 给出综合评分公式
4. 定义评分等级 (S/A/B/C/D)
5. 讨论可能的偏差和应对方案"
```

**Cursor Opus 4.6 执行:**

```markdown
任务: 实现地址标签数据库的数据模型和 API

请创建以下文件:
1. models/entity.py - 实体模型 (机构/个人/协议/交易所)
2. models/address.py - 地址模型 (含质量评分、标签、所属实体)
3. models/label.py - 标签模型 (来源、置信度、过期时间)
4. services/entity_resolver.py - 实体解析服务
5. api/labels.py - 标签查询 API

技术栈: Python + FastAPI + SQLAlchemy + PostgreSQL
要求:
- 支持多链地址 (EVM + Solana)
- 标签支持多来源 + 置信度权重
- 实体支持一对多地址关系
- 地址质量评分按以下公式: [粘贴 ChatGPT 生成的公式]
```

---

### Phase 1: 核心指标实现

#### Task 1.1: 实现"高质量资金净流入/流出"

**ChatGPT Plus — 设计计算规范:**

```markdown
Prompt:
"请为 CBS 的'高质量资金净流入/流出'指标设计完整的计算规范。

背景:
- 数据源: 链上 DEX Swap 事件 (Uniswap V2/V3, SushiSwap, Curve等)
- 高质量地址集合: 约5,000个已标记地址，带质量评分(0-100)
- 目标: 对每个 Token，计算一个加权净流入值

请提供:
1. 精确的数学公式
2. 时间窗口建议 (1h/4h/24h/7d)
3. 标准化方法 (使不同市值Token的指标可比)
4. 异常值处理 (如何处理极端大额交易)
5. 信号阈值定义 (什么值算'强流入'?)
6. 与 Token 市值的关系处理
7. 边界条件 (无交易、新Token、低流动性)

请用 Python 伪代码表示核心逻辑。"
```

**Cursor Opus 4.6 — 工程实现:**

```markdown
任务: 基于以下规范实现高质量资金净流入指标

规范: [粘贴 ChatGPT 生成的规范]

实现要求:
1. 数据采集: 监听 DEX Swap 事件 (先支持 Uniswap V3)
2. 实时计算: 每笔 swap 触发增量更新
3. 定时聚合: 每小时生成快照
4. 缓存: Redis 缓存最新值
5. API: GET /api/v1/cbs/netflow/{token}?window=24h
6. 存储: ClickHouse 存储历史数据

请实现完整的文件结构，包括:
- workers/dex_swap_listener.py
- services/netflow_calculator.py
- api/endpoints/netflow.py
- tests/test_netflow_calculator.py
```

**Gemini Pro — 数据验证:**

```markdown
Prompt:
"请帮我验证以下 Token 的资金流数据是否合理:

Token: [ETH/USDT]
我的系统计算结果:
- 过去24h高质量资金净流入: $12.5M
- 参与高质量地址数: 47
- 最大单笔: $2.3M (地址: 0x...)

请:
1. 在 Arkham/Nansen 上查找该时段的大额交易确认
2. 与 Spot On Chain 的'聪明钱'数据对比
3. 检查是否有遗漏的重大交易
4. 评估结果的合理性"
```

---

### Phase 2: 指标优化与扩展

#### Task 2.1: 使用 ChatGPT Code Interpreter 做回测

```markdown
Prompt (附带CSV数据文件):
"我上传了CBS模块6个月的历史指标数据(CSV)，包含:
- daily_netflow: 每日净流入
- exchange_flow: 交易所净流量
- new_participants: 新增高质量地址
- concentration_change: 集中度变化
- lp_behavior: LP行为评分
- price_change_24h: 对应Token 24h后的价格变化

请:
1. 计算每个指标与24h价格变化的相关系数
2. 测试不同权重组合的预测能力(Sharpe Ratio)
3. 用遗传算法或网格搜索找最优权重
4. 绘制各指标的累计收益曲线
5. 分析哪些指标在牛市/熊市中表现更好
6. 提出最终的综合CBS评分权重建议"
```

#### Task 2.2: 使用 Gemini Pro 做异动案例分析

```markdown
Prompt:
"请搜索2024-2025年间最著名的10个'聪明钱先于市场行动'的案例
(例如: FTX崩盘前的资金撤出、某Token暴涨前的鲸鱼积累等)

对每个案例，请分析:
1. 事件描述
2. 聪明钱的具体链上行为(时间、金额、地址)
3. 行为发生到市场反应的时间差
4. 该行为对应CBS的哪个指标
5. 如果当时有CBS系统，能否提前发出信号

这些案例将用于:
- 验证CBS指标体系的完备性
- 作为回测验证的标记数据
- 作为产品推广的案例素材"
```

---

## 3. 日常运营中的 AI 使用

### 每日流程

| 时间 | 任务 | 使用模型 | 说明 |
|------|------|---------|------|
| 09:00 | 异动扫描 | Gemini Pro | 检查过去24h的重大链上异动 |
| 10:00 | 数据质量检查 | Cursor | 运行数据一致性检查脚本 |
| 11:00 | 信号验证 | ChatGPT | 分析当日信号与市场走势的匹配度 |
| 14:00 | 标签更新 | Gemini Pro | 搜索新发现的机构地址 |
| 16:00 | 代码迭代 | Cursor | 实现新功能/修复问题 |
| 18:00 | 日报生成 | Gemini Pro | 生成当日 CBS 信号摘要报告 |

### 每周流程

| 任务 | 使用模型 | 说明 |
|------|---------|------|
| 指标回测 | ChatGPT + Code Interpreter | 上传一周数据，分析信号准确率 |
| 竞品跟踪 | Gemini Pro | 检查竞品是否有新功能上线 |
| 代码重构 | Cursor | 根据周回测结果优化计算逻辑 |
| 新模式发现 | ChatGPT | 分析回测异常数据，发现新行为模式 |

---

## 4. Prompt 工程最佳实践

### 4.1 上下文传递模板

在不同 AI 之间传递结果时，使用统一格式：

```markdown
## CBS 上下文传递
- 来源模型: [Gemini/ChatGPT/Cursor]
- 任务ID: [Phase.Task]
- 日期: [YYYY-MM-DD]
- 关键输出:
  [具体结果]
- 下游任务:
  [这个输出将用于什么]
- 待验证假设:
  [这个输出中有哪些未验证的假设]
```

### 4.2 角色设定

每个模型的最佳角色设定：

**Gemini Pro:**
> 你是 Web3 链上数据研究分析师，擅长跨平台信息搜索、数据交叉验证和趋势分析。你的工作服务于 CBS (Capital Behavior Signals) 模块。

**ChatGPT Plus:**
> 你是量化研究员兼产品策略师，擅长金融指标设计、统计建模和数据分析。你在设计 CBS 模块的指标体系和评分模型。

**Cursor Opus 4.6:**
> 你是 CBS 模块的首席工程师，正在实现一个实时链上资金行为信号系统。技术栈: Python/FastAPI + ClickHouse + Redis + Kafka。

### 4.3 迭代优化闭环

```
Gemini 发现新的异动模式
  → ChatGPT 将其抽象为可量化指标
    → Cursor 实现计算逻辑
      → ChatGPT 用回测数据验证有效性
        → Gemini 用真实案例交叉验证
          → Cursor 部署到生产环境
            → 回到第一步
```

---

## 5. 成本控制建议

| 模型 | 月度预算建议 | 优化策略 |
|------|------------|---------|
| Gemini Pro | ~$20-50 | 批量查询、缓存搜索结果 |
| ChatGPT Plus | $20/月 | 利用 Code Interpreter 减少 API 调用 |
| Cursor | 按订阅 | 利用 `.cursorrules` 减少上下文重复 |

**关键省钱技巧**: 将 Gemini 的搜索结果和 ChatGPT 的分析结果持久化到文档中，避免重复查询。

---

*本手册随 CBS 模块迭代持续更新*
