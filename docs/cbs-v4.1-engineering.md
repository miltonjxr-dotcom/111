# CBS（资金行为信号）— 产品需求文档（工程版）

> 版本: v4.1 | 最后更新: 2026-04-14

## 一、名词定义

| 术语 | 全称 | 定义 |
|------|------|------|
| CBS | Capital Behavior Signals | 资金行为信号，注意力引擎模块 2 |
| 实体 (Entity) | — | 一个独立的资金决策主体（机构/个人），可拥有多个链上地址 |
| 地址 (Address) | — | 单个链上钱包地址，归属于某个实体 |
| 白名单 (Whitelist) | — | 被纳入 CBS 持续监控的高质量实体集合 |
| 行为事件 (BehaviorEvent) | — | 经 Event Layer 清洗分类后的标准化链上事件 |
| 标准行为 | — | 五类行为之一：真实买入、真实卖出、迁移、结构、噪音 |
| CBS 信号 | — | 系统识别出的、关于某 Token 的资金异动信号 |
| 实体画像 (Profile) | — | 基于历史链上行为回测生成的实体特征数据集，含胜率/持仓/风格等 |
| 行为基线 (Baseline) | — | 某实体过去一段时间的行为统计均值，用于异常检测 |
| 供给路径 | — | 从链上原始事件到最终信号输出的完整数据流 |
| 行为匹配分 | — | 某次异动与该实体历史高胜率行为模式的匹配程度评分 |

## 二、系统目标

构建一个持续运行的自动化系统，监控链上高质量资金实体的行为，从中识别资金异动信号，当信号强度达到阈值后推送给投研分析师，帮助团队捕获早期投资机会或风险预警。

核心链路：**发现高质量实体 → 采集链上行为 → 构建实体画像 → 持续监控异动 → 与画像对比评估 → 计算指标与评分 → 达阈值推送通知 → 回测验证并更新画像。**

**与模块三（CT 监控）的类比：**

| 维度 | 模块三（CT 监控） | 模块二（CBS） |
|------|-----------------|-------------|
| 监控对象 | Smart CT 账号的推文 | 白名单实体的链上行为 |
| 信号来源 | 推文内容 + 传播度 | 链上交易 + 持仓变化 |
| 质量评分 | Smart 值（信誉+回测） | 实体画像（胜率+风格+有效性） |
| 信号累积 | Alpha 值累积到阈值发射 | CBS Score + 异动事件触发推送 |
| 回测闭环 | 推文 PnL 反馈到 Smart 值 | 信号 PnL 反馈到实体画像 |

## 三、系统架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                    CBS System                                     │
│                                                                    │
│ ┌──────────────┐ ┌──────────────┐ ┌────────────────────────┐     │
│ │ M1: 实体发现  │──▶│ M2: 实体画像  │──▶│ M3: 链上行为监控       │     │
│ │ & 白名单管理  │   │ & 评分管理    │   │ (Event Layer)         │     │
│ └──────────────┘ └──────────────┘ └───────────┬────────────┘     │
│       ▲                                        │                   │
│       │                                        ▼                   │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ M4: 指标计算 & 信号引擎                                      │   │
│ │ 行为→指标映射 → 评分 → 异动检测 → 推送                        │   │
│ └────────────────┬───────────────────────────────────────────┘   │
│                   │                                                │
│ ┌────────────────┴──────────────────────────────────────────┐   │
│ │ M5: 回测与画像更新引擎                                        │   │
│ │ 信号回测 → 胜率更新 → 画像刷新 ──反馈──▶ M2                   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                    │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ M6: 产品层（Dashboard / 异动推送 / API / 跨模块接口）        │   │
│ └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## 四、M1 — 实体发现与白名单管理

### 4.1 实体来源

| 来源 | 方式 | 执行频率 |
|------|------|---------|
| Arkham Intelligence API | 拉取已标记实体及其地址 | 每日增量 |
| Nansen Labels | 导入 Smart Money / Fund 标签地址 | 每日增量 |
| 链上溯源 | 从已知实体出发，通过 Gas Funder / 资金流聚类发现关联地址 | 每周 |
| 手动导入 | 投研团队手动添加已知机构/鲸鱼地址 | 按需 |
| 模式 B 发现 | 跨模块触发时发现的"疑似高质量"新地址 | 按需 |

### 4.2 实体分类（一级/二级）

**5 大类 18 子类**（详见前文）。以 `entity_id` 为唯一标识，地址归属于实体。同一实体对不同 Token 可持有不同角色（通过 `entity_token_roles` 表实现）。

### 4.3 准入评级

硬性门槛，任一不满足则 `status = rejected`：

| 检查项 | 条件 | 数据来源 |
|--------|------|---------|
| 标签可信度 | 至少 1 个可靠来源标记 | Arkham/Nansen/手动 |
| 活跃度 | 近 90 天有交易记录 | 链上数据 |
| 规模门槛 | 管理资产估值 > `[TUNABLE: $100K]` | 持仓快照 |

### 4.4 白名单管理

- 达标实体写入 `whitelist_entities` 表，状态为 `active`
- 每月复审：底部 10% 进入 `review` 状态，投研确认后淘汰或保留
- 标签衰减：`[TUNABLE: 30d]` 无链上活动 → 置信度 -10%；`[TUNABLE: 180d]` → 降入 `dormant`

## 五、M2 — 实体画像构建与评分

**这是 v4.1 相比 v4.0 的核心新增模块。**

### 5.1 为什么需要画像

分类告诉你"这个实体是谁"。**画像告诉你"这个实体过去表现如何、擅长什么、这次行为对它来说是否异常"。**

画像不是人为标注的，而是**通过回测历史链上数据计算出来的**。系统先采集实体的全部历史有效买卖动作，然后计算一系列指标，最终生成画像。

### 5.2 画像构建流程

```
Step 1: 历史行为采集
  拉取该实体所有地址在过去 [TUNABLE: 6个月] 的链上事件
  → 通过 Event Layer 归类为五类行为
  → 提取所有"真实买入"和"真实卖出"事件

Step 2: 交易配对
  将买入和卖出匹配为"完整交易"（同 Token 的买入→卖出）
  未平仓的持仓标记为 open_position

Step 3: 逐笔计算
  每笔完整交易计算:
  - 盈亏 (PnL%)
  - 持仓时间 (days)
  - 仓位占比 (该笔金额 / 当时总持仓)
  - 买入时机 (价格相对 30d 高低点的位置)

Step 4: 聚合统计
  汇总所有完整交易，生成画像指标

Step 5: 标签生成
  基于聚合指标自动打标（A层/B层/C层）

Step 6: 快照存储
  生成 entity_profile 记录，每日更新
```

### 5.3 画像指标体系（实体级）

| 指标组 | 指标 | 计算方式 | 用途 |
|--------|------|---------|------|
| **胜率** | win_rate_all | 盈利交易数 / 总完成交易数 | 整体能力评估 |
| | win_rate_30d | 近 30 天胜率 | 近期状态 |
| | win_rate_90d | 近 90 天胜率 | 中期趋势 |
| **盈亏** | avg_pnl | 所有完整交易的平均 PnL% | 盈利能力 |
| | avg_win_pnl | 盈利交易的平均 PnL% | 赚钱时赚多少 |
| | avg_loss_pnl | 亏损交易的平均 PnL% | 亏钱时亏多少 |
| | profit_factor | 总盈利额 / 总亏损额 | 盈亏比 |
| **持仓** | avg_hold_days | 平均持仓天数 | 持仓风格 |
| | median_hold_days | 持仓天数中位数 | 更抗极端值 |
| | avg_position_pct | 平均仓位占比 | 下注习惯 |
| | max_position_pct | 最大仓位占比 | 最大单注 |
| **节奏** | avg_trades_per_month | 月均交易次数 | 活跃度 |
| | avg_entries_per_trade | 每笔交易的平均建仓次数 | 分批/一次性 |
| **择时** | avg_entry_position | 买入价在 30d 高低点中的位置 | 左侧/右侧 |
| | event_proximity_rate | 买入后 72h 内发生重大事件的比例 | 事件驱动度 |
| **偏好** | preferred_mcap_range | 交易胜率最高的市值区间 | 能力圈 |
| | preferred_chain | 交易胜率最高的链 | 生态偏好 |
| | sector_distribution | 交易过的 Token 板块分布 | 板块偏好 |

### 5.4 基于画像的风格标签自动生成

| 标签层 | 标签 | 触发条件 |
|--------|------|---------|
| A层 | 事件驱动型 | event_proximity_rate > `[TUNABLE: 30%]` |
| A层 | 趋势配置型 | avg_hold_days > `[TUNABLE: 14]` 且 avg_entry_position > `[TUNABLE: 0.5]` |
| A层 | 早期发现型 | preferred_mcap_range 中位数 < `[TUNABLE: $10M]` |
| B层 | 阶梯建仓 | avg_entries_per_trade > `[TUNABLE: 3]` |
| B层 | 一次性重仓 | avg_entries_per_trade < `[TUNABLE: 1.5]` 且 avg_position_pct > `[TUNABLE: 10%]` |
| B层 | 左侧入场 | avg_entry_position < `[TUNABLE: 0.2]` |
| B层 | 右侧入场 | avg_entry_position > `[TUNABLE: 0.5]` |
| C层 | 近期有效 | win_rate_30d > `[TUNABLE: 55%]` |
| C层 | 近期衰减 | win_rate_30d < `[TUNABLE: 40%]` |

### 5.5 画像更新机制

| 触发事件 | 更新内容 | 频率 |
|---------|---------|------|
| 新的完成交易（卖出平仓） | 增量更新：重算受影响指标 | 实时 |
| 每日快照 | 全量重算所有指标 + 标签 | 每日 UTC 00:00 |
| 手动触发 | 运营后台重算某实体 | 按需 |

### 5.6 行为匹配分（关键设计）

**当某实体发生新的异动时，系统将该次行为与其画像进行对比，计算"行为匹配分"——该次行为与其历史高胜率模式的匹配程度。**

```
场景: A3 实体 "SM_0042" 买入 $XYZ

该实体画像:
  win_rate_90d: 68%
  avg_hold_days: 23天
  avg_position_pct: 12%
  avg_entries_per_trade: 3.2 (阶梯建仓)
  B层标签: 阶梯建仓 + 右侧确认 + 中线持有

本次行为:
  买入金额: $800K
  仓位占比: 15% (高于均值12%)
  建仓方式: 一次性 (偏离惯常的阶梯建仓)
  买入时机: 价格在30d高点附近 (右侧确认✓)

行为匹配分计算:
  仓位维度: 15% vs avg 12% → 高于均值 → 该实体认真下注 → +
  建仓维度: 一次性 vs 习惯阶梯 → 偏离基线 → 紧迫感更强 → 标记异常
  时机维度: 右侧 → 符合惯常风格 → ✓
  
输出:
  行为匹配标签: HIGH_CONVICTION (高确信度操作)
  原因: 仓位高于历史均值 + 放弃惯常分批策略一次性建仓
  参考: 该实体历史上类似高仓位+一次性建仓的交易胜率 = 78% (高于整体68%)
```

**这直接回答了"买入金额对应到过往行为模式中，是否像高胜率动作"这个问题。**

### 5.7 行为匹配分的技术实现

```python
def calc_behavior_match(entity_profile, current_event):
    """计算本次行为与实体历史高胜率模式的匹配度"""
    
    position_pct = current_event.amount / entity_profile.total_holdings
    
    match_signals = []
    
    # 仓位对比
    if position_pct > entity_profile.avg_position_pct * 1.5:
        match_signals.append("HIGH_POSITION")
    elif position_pct < entity_profile.avg_position_pct * 0.3:
        match_signals.append("LOW_POSITION")
    
    # 建仓方式对比
    if entity_profile.has_label("阶梯建仓"):
        if current_event.is_single_entry:
            match_signals.append("UNUSUAL_SINGLE_ENTRY")
    
    # 查询历史上类似模式的胜率
    similar_trades = query_similar_trades(
        entity_id=entity_profile.id,
        position_range=(position_pct * 0.8, position_pct * 1.2),
        entry_style=current_event.entry_style
    )
    similar_win_rate = calc_win_rate(similar_trades)
    
    return BehaviorMatch(
        signals=match_signals,
        similar_win_rate=similar_win_rate,
        overall_conviction=determine_conviction(match_signals, similar_win_rate)
    )
```

## 六、M3 — 链上行为监控（Event Layer）

### 6.1 数据采集

| 链 | 数据源 | 延迟 | 优先级 |
|---|--------|:----:|:------:|
| Ethereum | Allium/Goldsky + 备用 RPC | 分钟级 | P0 |
| Base | 同上 | 分钟级 | P0 |
| Arbitrum | 同上 | 分钟级 | P0 |
| Solana | Helius + 备用 RPC | 分钟级 | P0 |
| BSC | 同上 | 分钟级 | P1 |
| Hyperliquid | API | 小时级 | P1 (Perp数据) |

架构设计为"链无关"——每条链仅需一份 `ChainConfig`（RPC + DEX 合约 + CEX 地址 + 标签）。

### 6.2 Event Layer 处理（六步）

详见 v4.0 第 5 章。核心流程：

原始事件 → D类过滤 → 结构穿透 → 白名单匹配 → 五类行为分类 → 标准化输出

五类行为链上路径：真实买入(7种) + 真实卖出(7种) + 迁移(8种) + 结构(7种) + 噪音(6种)

### 6.3 双触发运行模式

**模式 A（白名单扫描，常驻）**：每小时扫描白名单全部地址 → Event Layer → 指标更新 → CBS Score → 异动检测

**模式 B（跨模块响应，按需）**：其他模块通知 → 白名单维度扫描 + Token 维度扫描（可发现新地址）

## 七、M4 — 指标计算与信号引擎

### 7.1 指标体系（7+1）

7 个指标不分组，扁平排列，各自输出有向值 [-100, +100]。

| # | 名称 | 权重 | 时间窗口 |
|---|------|:----:|---------|
| 1 | 高质量资金净流入/流出 | 0.25 | 24h/7d/30d |
| 2 | 连续增持/减持天数 | 0.15 | 自身时间指标 |
| 3 | 高质量资金参与动态 | 0.15 | 24h/7d/30d |
| 4 | 交易所净流量 | 0.15 | 24h/7d/30d |
| 5 | 持仓集中度变化 | 0.10 | 7d/30d |
| 6 | 结构性资金事件 (层1量化+层2事件流) | 0.10 | 事件触发 |
| 7 | LP行为与流动性动态 | 0.10 | 7d/30d |
| Adj | Perp 仓位修正 | 乘数 | 快照 |

各指标详细定义见 v4.0 第 7 章。

### 7.2 CBS Score 计算

```
CBS_Score = [Σ(wi × Ii_normalized)] × PerpModifier

窗口共振: 某指标 24h/7d/30d 全同向 → 该指标权重 ×1.2
```

### 7.3 异动检测与推送触发

| 触发条件 | 推送优先级 |
|---------|:---------:|
| A 类绝对确认买入 > `[TUNABLE: $500K]` | 高 |
| 多个独立高质量实体 4h 内同向操作同一 Token | 高 |
| 做市商异常行为（撤出/反向/多做市商同向） | 高 |
| 连续增持天数 > `[TUNABLE: 7天]` | 中 |
| A1 机构首次参与某 Token | 高 |
| 撤池 + 深度恶化 | 高 (风险) |
| 冷→热钱包大额转移 | 中 |

### 7.4 异动推送内容（核心输出）

```
CBS 异动报告 | $XYZ | 2026-04-14 14:32 UTC

触发: A3匿名高手 "SM_0042" 买入 $XYZ $800K

实体画像快照:
  一级分类: A3 匿名高质量买方
  A层风格: 趋势配置型
  B层风格: 阶梯建仓 / 右侧确认 / 中线持有
  近期状态: 近期有效 (30D胜率 68%)
  
本次行为 vs 画像对比:
  仓位: 15% (高于历史均值 12%) → 认真下注
  建仓: 一次性建仓 (偏离惯常阶梯风格) → 紧迫感
  时机: 右侧确认 → 符合惯常风格
  行为匹配: HIGH_CONVICTION
  历史类似模式胜率: 78% (高于整体 68%)
  
CBS 指标快照:
  净流入(1): 24h +$2.3M | 7d +$5.1M
  增持天数(2): 5天
  参与动态(3): +3实体 | 87%看多
  
建议: 高优先级关注。该实体的本次操作符合其历史高胜率行为模式。
  → 查 ECS: 是否有催化事件临近？
  → 查 NDS: 叙事层面是否有扩散？
```

## 八、M5 — 回测与画像更新引擎

### 8.1 信号级回测

以异动推送时间为 T0，在 `[TUNABLE: +24h / +7d / +30d]` 回测价格表现：

| 窗口 | 权重 | 设计理由 |
|------|:----:|---------|
| 24h | 0.35 | 主力窗口，短线验证 |
| 7d | 0.40 | 核心窗口，大部分信号在此验证 |
| 30d | 0.25 | 中线参考 |

### 8.2 画像反馈闭环

```
异动推送 → 价格回测 → 该笔交易标记为盈/亏
  → 更新该实体的画像指标（胜率/PnL/持仓时间等）
  → 标签可能变化（如"近期有效"→"近期衰减"）
  → 下次该实体异动时，画像对比结果更准确
```

### 8.3 系统级参数自动调优（Phase 3）

定期分析所有推送信号的回测结果，自动调整：
- 各指标权重
- 异动推送阈值
- 行为匹配分的触发条件

## 九、核心数据模型

### 9.1 entities（实体表）

```
id              BIGINT PK
entity_name     VARCHAR             -- 如 "Paradigm", "SM_0042"
entity_class    ENUM('A1','A2','A3','B1','B2','C1','C2','C3','C4',
                     'D1','D2','D3','D4','E1','E2','E3','E4')
is_whitelist    BOOLEAN DEFAULT FALSE
whitelist_status ENUM('active','review','dormant','rejected')
source          ENUM('arkham','nansen','onchain','manual','discovered')
confidence      FLOAT               -- 标签置信度 0-1
total_addresses INT                 -- 关联地址数
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### 9.2 addresses（地址表）

```
id              BIGINT PK
address         VARCHAR UNIQUE
chain           VARCHAR             -- ethereum/base/arbitrum/solana/bsc
entity_id       BIGINT FK -> entities.id
address_type    ENUM('hot','cold','multisig','contract','unknown')
first_seen_at   TIMESTAMP
last_active_at  TIMESTAMP
created_at      TIMESTAMP
```

### 9.3 entity_profiles（实体画像）

```
id              BIGINT PK
entity_id       BIGINT FK -> entities.id UNIQUE
-- 胜率
win_rate_all    FLOAT NULL
win_rate_30d    FLOAT NULL
win_rate_90d    FLOAT NULL
-- 盈亏
avg_pnl         FLOAT NULL
profit_factor   FLOAT NULL
-- 持仓
avg_hold_days   FLOAT NULL
avg_position_pct FLOAT NULL
max_position_pct FLOAT NULL
-- 节奏
avg_trades_per_month FLOAT NULL
avg_entries_per_trade FLOAT NULL
-- 择时
avg_entry_position FLOAT NULL
event_proximity_rate FLOAT NULL
-- 偏好
preferred_mcap_range JSONB NULL
preferred_chain VARCHAR NULL
-- 标签
label_a         VARCHAR NULL        -- A层主标签
label_b         JSONB NULL          -- B层标签数组
label_c         JSONB NULL          -- C层标签数组
-- 元数据
total_trades    INT DEFAULT 0
profile_window_months INT DEFAULT 6
last_updated_at TIMESTAMP
snapshot_date   DATE
```

### 9.4 behavior_events（行为事件）

```
id              BIGINT PK
entity_id       BIGINT FK -> entities.id
address         VARCHAR
chain           VARCHAR
token_address   VARCHAR
token_symbol    VARCHAR
behavior_type   ENUM('confirmed_buy','confirmed_sell','inferred_buy',
                     'inferred_sell','lp_implied_buy','lp_implied_sell',
                     'migration','structural','noise')
confidence      FLOAT               -- 置信度 0-1
amount_usd      DECIMAL(20,2)
tx_hash         VARCHAR
block_number    BIGINT
event_timestamp TIMESTAMP
labels          JSONB               -- 如 ["STABLECOIN_ACTIVATION","HIGH_CONVICTION"]
raw_data        JSONB
created_at      TIMESTAMP
```

### 9.5 metric_snapshots（指标快照）

```
id              BIGINT PK
token_address   VARCHAR
token_symbol    VARCHAR
chain           VARCHAR
snapshot_time   TIMESTAMP
-- 7个指标值
indicator_1     FLOAT               -- 净流入
indicator_2     FLOAT               -- 增持天数
indicator_3     FLOAT               -- 参与动态
indicator_4     FLOAT               -- 交易所流量
indicator_5     FLOAT               -- 持仓集中度
indicator_6     FLOAT               -- 结构性事件(层1)
indicator_7     FLOAT               -- LP动态
perp_modifier   FLOAT DEFAULT 1.0
cbs_score       FLOAT
window_pattern  VARCHAR NULL        -- BULL_ALIGNED / SHORT_REVERSAL 等
created_at      TIMESTAMP
```

### 9.6 signal_alerts（异动推送）

```
id              BIGINT PK
token_address   VARCHAR
token_symbol    VARCHAR
chain           VARCHAR
trigger_type    ENUM('large_buy','multi_entity','mm_anomaly',
                     'streak','first_participation','lp_risk','cold_hot')
trigger_entity_id BIGINT FK NULL
trigger_details JSONB
-- 画像对比
behavior_match_score FLOAT NULL
behavior_match_label VARCHAR NULL    -- HIGH_CONVICTION / NORMAL / LOW_CONVICTION
behavior_match_details JSONB NULL
-- CBS 指标快照
cbs_score_at_alert FLOAT
indicators_snapshot JSONB
-- 回测
backtest_24h_pnl FLOAT NULL
backtest_7d_pnl FLOAT NULL
backtest_30d_pnl FLOAT NULL
-- 推送
pushed_at       TIMESTAMP NULL
push_channel    VARCHAR NULL
alert_priority  ENUM('high','medium','low')
created_at      TIMESTAMP
```

## 十、外部依赖与数据源

| 依赖 | 用途 | 备注 |
|------|------|------|
| Arkham API | 实体标签、地址归属 | 需订阅 |
| Nansen API | Smart Money 标签 | 需订阅 |
| Allium / Goldsky | 结构化链上数据(EVM) | 按量计费 |
| Helius | Solana 链上数据 | 按量计费 |
| Hyperliquid API | Perp 仓位数据 | 免费 |
| DexScreener API | 代币价格、池子数据 | 免费,有限流 |
| CoinGecko API | 代币价格备用源 | 免费/付费 |
| ECS 模块接口 | 事件日历(解锁/上所) | 内部 |
| PostgreSQL | 主数据库 | — |
| Redis | 缓存+任务调度 | — |

## 十一、边界条件与风险

| 场景 | 处理方式 |
|------|---------|
| 实体使用 Fresh Wallet | Gas Funder 溯源 + 时序分析关联到已知实体 |
| Router/Bridge 中转 | 穿透到原始 EOA |
| CEX 后续行为不可见 | 接受局限，只推测不确认 |
| VC 未解锁做空 | 链上 Perp 可监测；CEX 不可监测 |
| 做市商常规做市噪音 | C1 行为四分类(常规过滤/异常纳入) |
| Token 无法获取价格 | 回测标记为 failed，不计入画像 |
| 代币改名/迁移 | 以 contract_address + chain 为唯一标识 |
| 标签数据源冲突 | 多源交叉验证，取置信度最高者 |

## 十二、产品输出

### 12.1 Token CBS 面板（Dashboard）

每小时更新，展示所有被追踪 Token 的 CBS Score + 7 指标 + 异动事件。

### 12.2 异动推送（Slack/飞书）

即时触发，含：触发事件 + 实体画像 + 行为匹配分 + CBS 指标 + 跨模块引导。

### 12.3 实体画像卡

每个白名单实体的画像页面：基础信息 + 画像指标 + 风格标签 + 历史交易列表 + 胜率走势。

### 12.4 跨模块 API

供其他模块查询某 Token 的 CBS 数据（模式 B 响应接口）。

## 十三、分期建议

### P0（MVP）

- 实体发现 + 白名单(≥3000 实体) + 一级分类
- Event Layer(五类行为识别 + LP判定 + 做市商分类) + D类过滤
- 多链数据管道(ETH/Base/Arb/Sol)
- 7 指标全部实现 + CBS Score
- **实体画像 v1**（基于历史数据的胜率/持仓/节奏等核心指标）
- Dashboard + 异动推送(Slack/飞书)
- 双触发运行模式

### P1

- BSC/Optimism 链扩展
- Hyperliquid 数据采集
- **行为匹配分**（异动 vs 画像对比）
- A/B/C 层标签自动化(P0标签)
- Contrast Layer（自我/同类/标的三维对比）
- 信号回测框架

### P2

- Perp 修正项正式启用
- Profile Layer 完整画像卡
- ECS 协同接口
- 参数回测自动优化
- P1/P2 标签扩展

## 十四、可配置参数清单

### 白名单参数

| 参数 | 初始值 | 说明 |
|------|--------|------|
| MIN_ASSET_VALUE | $100K | 准入最低资产 |
| DORMANT_THRESHOLD_DAYS | 180 | 休眠认定天数 |
| REVIEW_BOTTOM_PCT | 10% | 每月复审比例 |

### 画像参数

| 参数 | 初始值 | 说明 |
|------|--------|------|
| PROFILE_WINDOW_MONTHS | 6 | 画像计算回溯窗口 |
| EVENT_PROXIMITY_HOURS | 72 | 事件驱动判定窗口 |
| TREND_HOLD_DAYS_MIN | 14 | 趋势配置最低持仓天数 |
| EARLY_MCAP_THRESHOLD | $10M | 早期发现市值阈值 |
| LADDER_ENTRY_MIN | 3 | 阶梯建仓最低建仓次数 |
| EFFECTIVE_WIN_RATE | 55% | "近期有效"胜率阈值 |

### 指标参数

| 参数 | 初始值 | 说明 |
|------|--------|------|
| W1-W7 | 0.25/0.15/0.15/0.15/0.10/0.10/0.10 | 指标权重 |
| WINDOW_RESONANCE_BOOST | 1.2 | 窗口共振加成 |
| CONCENTRATION_WHALE_PCT | 0.1% | 鲸鱼层持仓阈值 |

### 推送参数

| 参数 | 初始值 | 说明 |
|------|--------|------|
| LARGE_BUY_THRESHOLD | $500K | 大额买入推送阈值 |
| STREAK_ALERT_DAYS | 7 | 连续增持推送天数 |
| MAX_ALERTS_PER_HOUR | 3 | 每小时最大推送数 |

### 回测参数

| 参数 | 初始值 | 说明 |
|------|--------|------|
| BACKTEST_24H_WEIGHT | 0.35 | 24h回测权重 |
| BACKTEST_7D_WEIGHT | 0.40 | 7d回测权重 |
| BACKTEST_30D_WEIGHT | 0.25 | 30d回测权重 |

## 十五、待确认问题

1. **历史数据回溯深度**：构建实体画像需要回溯多久的历史？6 个月 vs 全历史？全历史数据量和成本如何？
2. **白名单规模**：初期目标监控多少实体？3,000 / 5,000 / 10,000？直接影响计算量和成本。
3. **价格数据源**：回测和画像计算需要精确的历史价格。DexScreener 历史数据 API 是否充足？是否需要自建价格数据库？
4. **跨模块接口规范**：与 MTS/NDS/ECS/BAS 的接口格式需要统一定义。
5. **链上数据成本**：按当前白名单规模和链覆盖，Allium/Goldsky 的月度成本预估？
6. **画像冷启动**：新入白名单的实体无历史数据时如何处理？最小历史数据量要求？
7. **MTS 流动性边界**：指标 7（LP 动态）的深度变化子维度与 MTS 的流动性指标如何明确边界？建议 CBS 侧重变化速度，MTS 侧重绝对值。

---

*本文档由注意力引擎 CBS 模块产研团队维护*
