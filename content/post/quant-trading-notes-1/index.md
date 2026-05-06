---
title: "量化交易笔记：1"
date: 2026-05-07
tags: ["AI协作", "加密货币", "量化交易", "Qlib"]
categories: ["量化交易"]
---

# 量化交易笔记 1：做加密货币交易的技术选择

> 基于 Qlib (Microsoft) 框架，面向加密货币短线交易

---

## 一、Qlib 项目结构速览

### examples/ 目录分类

| 目录 | 类型 | 说明 |
|------|------|------|
| `benchmarks/` | 模型对比实验 | LightGBM、LSTM、GRU、Transformer、TRA、HIST 等，统一数据集横向对比 IC/Sharpe |
| `benchmarks_dynamic/` | 动态市场适应 | 测试模型在市场分布偏移时的表现（DDG-DA 等） |
| `highfreq/` | 高频特征工程 | 日内/tick 级别自定义算子、Handler、Processor |
| `nested_decision_execution/` | 嵌套执行器 | 日频策略 + 分钟频执行同时模拟 |
| `rl_order_execution/` | 强化学习执行 | 用 RL 做订单拆单（TWAP/VWAP 类问题） |
| `online_srv/` | 在线服务 | 模型上线后的滚动再训练、在线预测更新 |
| `portfolio/` | 组合管理 | 增强指数配置示例 |
| `model_rolling/` | 滚动更新 | walk-forward 框架 |

---

## 二、重要概念区分

### Alpha158 是特征集，不是模型

这是最容易混淆的地方：

| 层次 | 例子 | 作用 |
|------|------|------|
| **特征集（Handler）** | Alpha158、Alpha360 | 决定用什么因子作为模型输入 |
| **预测模型** | XGBoost、LightGBM、LSTM... | 决定怎么从特征预测收益 |

两者是正交关系，可以任意组合：LightGBM + Alpha158、LightGBM + Alpha360 都是合法的。

### Alpha158 的 158 个因子构成

```
9  个 K 线形态因子（KMID, KLEN, KMID2, KUP, KUP2, KLOW, KLOW2, KSFT, KSFT2）
4  个原始价格因子（OPEN/HIGH/LOW/VWAP 各 1 个，window=0）
30 个 rolling 指标 × 5 个窗口（5/10/20/30/60天）= 145 个
   包括：ROC, MA, STD, BETA, RSQR, RESI, MAX, MIN, QTLU, QTLD, RSV,
         IMAX, IMIN, IMXD, CORR, CORD, CNTP, CNTN, CNTD,
         SUMP, SUMN, SUMD, VMA, VSTD, WVMA, VSUMP, VSUMN, VSUMD
```

本质是**同一套技术指标在 5 个时间窗口的展开**，存在大量冗余。

### Alpha158 vs Alpha360 的本质差异

| | Alpha158 | Alpha360 |
|-|----------|----------|
| 内容 | 人工设计的 rolling 因子 | 原始 OHLCV × 60 天 = 360 个特征 |
| 适合模型 | 树模型（LightGBM/XGBoost） | 时序模型（LSTM/GRU） |
| 人工先验 | 多 | 少 |
| 竞争壁垒 | 低（用的人多） | 相对较高 |

### benchmark 表关键数据（CSI300，Alpha158）

| 模型 | 年化收益 | IR | 特点 |
|------|----------|-----|------|
| DoubleEnsemble | 11.6% | 1.34 | 最强，基于 LightGBM 集成 |
| LightGBM | 9.0% | 1.02 | 稳定，门槛低，推荐起点 |
| XGBoost | 7.8% | 0.91 | 略弱于 LightGBM |
| MLP | 8.9% | 1.14 | 神经网络里性价比高 |

---

## 三、加密货币短线交易指南

### 时间级别选择

| 级别 | 持仓时长 | 门槛 |
|------|----------|------|
| 高频做市 | 毫秒~秒 | 需要 co-location，极高门槛 |
| **日内趋势/反转** | **分钟~小时** | **推荐起点** |
| 隔夜短摆 | 小时~天 | 最容易入手 |

### 订单簿核心信号（你的直觉是对的）

**1. Order Book Imbalance (OBI)**
```
OBI = (bid_volume - ask_volume) / (bid_volume + ask_volume)
OBI > 0 → 买压大 → 价格倾向上涨
```

**2. Microprice（比 mid-price 更精确）**
```
microprice = (ask_price × bid_vol + bid_price × ask_vol) / (bid_vol + ask_vol)
```
通常领先成交价几秒到几十秒。

**3. Trade Flow Imbalance**
主动买入量 vs 主动卖出量的比值，反映"聪明钱"方向。

**4. Order Book Depth Gradient**
前 5 档挂单分布——上方挂单稀薄说明阻力小，易突破。

### 加密市场独有因子

| 因子 | 来源 | 信号含义 |
|------|------|----------|
| **资金费率（Funding Rate）** | 永续合约，每 8 小时结算 | 持续为正 → 多头拥挤 → 反转信号 |
| **期现价差（Basis）** | 现货 vs 期货 | 情绪溢价指标 |
| **清算数据（Liquidation）** | Binance/Bybit API | 大量多单清算 → 短暂超跌后反弹 |
| **鲸鱼大单流** | 超阈值成交 | 机构行为有延续性 |

### 模型选择逻辑

```
数据是表格特征（OBI、资金费率、技术指标...）
    → LightGBM / XGBoost  ← 最稳，先从这里开始

数据是时间序列（订单簿逐帧快照）
    → LSTM / Transformer

数据是订单流事件序列（每一笔成交）
    → TCN 或专门的 LOB 模型（DeepLOB 等）
```

---

## 四、工具选择：自己写 vs Qlib vs RD-Agent

### 三者的定位

```
自己写代码          → 灵活，但要从零造轮子
    ↓
Qlib               → 给你造好了 ML 量化的标准流水线
    ↓
RD-Agent           → 用 LLM 自动帮你跑 Qlib、挖因子、调模型
```

**它们是叠加关系，不是竞争关系。**

### Qlib 真正节省时间的地方

| 模块 | 价值 |
|------|------|
| **回测防护** | Look-ahead bias 防护、Point-in-Time 数据、手续费/滑点模型内置 |
| **实验管理** | MLflow 封装，每次实验的参数、指标、模型文件全部自动归档 |
| **标准评估报告** | IC/ICIR、年化收益、最大回撤、Sharpe，一行代码出完整报告 |
| **滚动训练** | walk-forward 框架，模型每月重训，在线预测自动更新 |

### Qlib 的短板（对加密短线）

| 短板 | 说明 |
|------|------|
| 分钟级以下支持弱 | 订单簿、逐笔成交没有标准化支持 |
| 加密支持有限 | 只有 CoinGecko 日线，没有 Binance 分钟线接口 |
| 因子库是 A 股视角 | rolling 窗口和加密市场 24h 不休盘不匹配 |

### RD-Agent 是什么

微软在 Qlib 之上造的 **LLM 自动化研究员**：
- 自动读论文、提取因子公式、生成 Qlib 表达式
- 自动跑实验、看 IC、决定是否保留因子
- 前提：你已有稳定的 Qlib 流水线。它是加速器，不是替代品。

### 针对加密短线的工具分工建议

| 模块 | 用什么 | 理由 |
|------|--------|------|
| 数据获取 | 自己写（Binance API） | Qlib 没有现成支持 |
| 特征工程 | 自己写（pandas/numpy） | 订单簿特征 Qlib 没有 |
| 模型训练 | LightGBM 直接调用 | 不需要 Qlib 包装 |
| **回测** | **用 Qlib** | 手续费模型、防 look-ahead，省大量时间 |
| **实验记录** | **用 Qlib（MLflow）** | 跑 100 次实验时不会乱 |
| 因子挖掘（后期） | RD-Agent | 有稳定流水线后再上 |

---

## 五、完整实践路线图

### 分钟级加密短线（推荐路径）

```
第一步：获取历史数据
  → 从 data.binance.vision 下载 BTC/USDT 1分钟K线 + aggTrades
  → 存成 parquet 格式

第二步：构建特征
  → OBI（前5档订单簿不平衡）
  → Microprice 偏差
  → Trade Flow Imbalance
  → 资金费率（Binance API）
  → 1/5/15 分钟 VWAP 偏差、波动率

第三步：标注 Label
  → 预测未来 5 分钟 mid-price 涨跌（二分类）
  → 或预测涨跌幅（回归）

第四步：训练模型
  → LightGBM，walk-forward 验证（时间序列不能随机 split）
  → 观察 Rank IC 和方向准确率

第五步：实验管理（接入 Qlib）
  → 每次实验自动归档到 MLflow
  → 产出：params.pkl（模型）、pred.pkl（预测分数表）、ic.pkl 等

第六步：选模型 + 上线
  → 从 MLflow 里选出 ICIR 稳定的实验
  → OnlineToolR.reset_online_tag() 标记为在线模型
  → 每根 K 线结束后调用 update_online_pred() 生成新预测分数

第七步：纸交易验证（2-4 周）
  → 验证回测 IC vs 实盘 IC 的差距（差距 > 30% 说明有问题）
  → 验证手续费后是否还盈利
  → 观察模型衰减速度

第八步：实盘（小仓位开始）
```

### 日线加密（快速验证，用 Qlib 现成工具）

```bash
cd scripts/data_collector/crypto
python collector.py download_data \
  --source_dir ~/.qlib/crypto_data/source/1d \
  --start 2020-01-01 --end 2024-12-31 \
  --interval 1d
```

---

## 六、实验产出详解

### 一次实验归档的文件

```
实验记录（MLflow）
├── params.pkl              ← 训练好的 LightGBM 模型（决策树结构 + 权重）
├── dataset（对象）          ← 特征配置和数据 pipeline
├── pred.pkl                ← 模型对测试集的打分结果（DataFrame）
├── label.pkl               ← 测试集对应的真实涨跌幅
├── sig_analysis/
│   ├── ic.pkl              ← 每天的 IC 时间序列
│   └── ric.pkl             ← 每天的 Rank IC 时间序列
└── portfolio_analysis/
    ├── report_normal.pkl   ← 回测收益曲线、持仓记录
    └── port_analysis.pkl   ← Sharpe、最大回撤、年化收益
```

### pred.pkl 的内容（不是模型文件！）

```
                        score
datetime   instrument
2024-01-02 BTC/USDT    0.0312   ← 模型认为会涨 3.12%
           ETH/USDT   -0.0089   ← 模型认为会跌 0.89%
2024-01-03 BTC/USDT    0.0156
...
```

**params.pkl 才是真正的模型文件**，包含所有决策树结构。

### 关系
```
params.pkl（模型）+ 新的市场数据（特征）→ model.predict() → 输出新的预测分数
```

---

## 七、信号层与策略层分离（核心设计思想）

```
【信号层】LightGBM 模型
    输入：特征（OBI、资金费率等）
    输出：每个标的的预测分数（实数，如 +0.73, -0.45）
    只回答"哪个更可能涨"，不管买多少、何时卖

            ↓ pred.pkl

【策略层】TopkDropoutStrategy（或自定义规则）
    输入：所有标的的预测分数
    规则示例（加密短线）：
      score > 0.6  → 开多仓，仓位 = score × 最大仓位
      score < -0.6 → 开空仓
      否则          → 不操作
    输出：具体买卖指令（Order）

            ↓

【执行层】Executor + Exchange
    模拟成交：手续费（Binance 0.05%）、滑点
    记录每天持仓和盈亏

            ↓

【评估层】PortAnaRecord
    输出：Sharpe、最大回撤、年化收益...
```

**最大回撤是策略 + 执行层模拟出来的，不是模型本身产生的。**
模型打分再准，策略设计不好（如全仓单一标的）回撤也会很大。

### 类比理解

| 概念 | 类比 |
|------|------|
| `params.pkl` | 会打分的"分析师" |
| `pred.pkl` | 分析师写的历史打分报告 |
| `TopkDropoutStrategy` | 基金经理，决定怎么操作 |
| `Exchange` | 交易员，负责真实下单 |
| `PortAnaRecord` | 风控部门算出来的收益/风险报告 |

---

## 八、常见坑

| 坑 | 说明 |
|----|------|
| **手续费侵蚀** | 日内高频：0.05% × 每天 20 次 = 1% 成本，IC 必须够高才能覆盖 |
| **Look-ahead bias** | 特征计算时用了"未来数据"，回测完美但实盘亏损 |
| **过拟合** | 订单簿数据噪声极大，walk-forward 验证是必须的，不能随机 split |
| **市场微观结构变化** | 加密市场变化快，去年的模型今年往往失效，模型需持续更新 |
| **只选 IC 最高的模型** | 往往是过拟合。要选跨时间段都稳定的，而非单段最优 |

---

## 九、关键判断标准

- **IC > 0.03**：基准线，低于此无太大意义
- **ICIR > 0.3**：IC 稳定性合格
- **IC 正值占比 > 55%**：大多数时候模型有效
- **回测 vs 实盘 IC 差距 < 30%**：说明没有严重的 look-ahead 或数据问题
- **纸交易 2-4 周后再上实盘**：观察真实信号质量

---

*笔记整理自 Qlib 框架学习对话，2026-05-06*