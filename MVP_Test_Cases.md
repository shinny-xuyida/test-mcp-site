# FinSight AI - MVP 最小实现用例集 (Test Cases)

本文档定义了 FinSight AI 最小可行产品 (MVP) 阶段的核心功能用例。这些用例旨在验证“自然语言 -> 语义映射 -> MCP 数据获取 -> Python 分析 -> 前端呈现”这一完整链路。

**适用范围**：后端逻辑开发、Prompt 优化、前端交互测试。

---

## 1. 基础功能用例 (Functional Cases)

### TC01: 基础单品种数据查询与可视化
**目标**：验证最简单的全链路打通，确保 Prompt 能正确映射品种并调用 MCP。

*   **用户输入 (User Prompt)**:
    > "看一下螺纹钢过去一年的价格走势。"
*   **前置条件**:
    *   System Prompt 中包含螺纹钢映射规则 (`RB` -> `KQ.m@SHFE.rb`).
*   **预期行为 (Expected Behavior)**:
    1.  **语义映射**: 识别实体 "螺纹钢" -> `KQ.m@SHFE.rb`。
    2.  **MCP 调用**: `tq_get_kline(symbol="KQ.m@SHFE.rb", duration_seconds=86400, data_length=300)` (约1年交易日)。
    3.  **Python 逻辑**: 读取 CSV，将 `datetime` 转为对象，绘制 `close` 价格曲线。
    4.  **输出结果**:
        *   图表：简单的折线图 (Line Chart)。
        *   结论：无需复杂分析，仅展示“这是螺纹钢主力连续过去一年的日线走势”。

### TC02: 跨品种相关性分析 (Core Scenario)
**目标**：验证多工具调用、数据对齐 (Data Alignment) 与统计计算。

*   **用户输入**:
    > "分析螺纹钢和铁矿石过去3年的价格相关性，并画出价差图。"
*   **预期行为**:
    1.  **语义映射**:
        *   螺纹钢 -> `KQ.m@SHFE.rb`
        *   铁矿石 -> `KQ.m@DCE.i`
    2.  **MCP 调用**:
        *   `tq_get_kline(symbol="KQ.m@SHFE.rb", duration_seconds=86400, ...)`
        *   `tq_get_kline(symbol="KQ.m@DCE.i", duration_seconds=86400, ...)`
    3.  **Python 逻辑**:
        *   **关键点**: 使用 `pd.merge` 按 `datetime` 对齐两个数据框 (Inner Join)。
        *   计算 `corr = df['close_rb'].corr(df['close_i'])`。
        *   计算价差 `spread = close_rb - close_i` (或按主力合约乘数比调整)。
    4.  **输出结果**:
        *   图表：双轴图 (Dual Axis) 或 标准化对比图 + 价差子图。
        *   结论：包含具体的相关系数 (e.g., 0.85) 和当前价差水平描述。

### TC03: 跨期价差分布 (Arbitrage)
**目标**：验证具体合约映射与统计分布绘图。

*   **用户输入**:
    > "玻璃 2501 和 2505 的价差分布怎么样？"
*   **预期行为**:
    1.  **语义映射**:
        *   识别品种 "玻璃" -> `CZCE.FG`
        *   识别月份 "2501" -> `CZCE.FG501`, "2505" -> `CZCE.FG505`
    2.  **MCP 调用**: 分别获取两个具体合约的日线数据。
    3.  **Python 逻辑**:
        *   数据对齐。
        *   计算 `spread = close_01 - close_05`。
        *   计算均值 (Mean) 和标准差 (Std)。
    4.  **输出结果**:
        *   图表：直方图 (Histogram) 或 密度图 (KDE)，并标记“当前价差”在分布中的位置。
        *   结论：当前价差处于历史的 % 分位（如：偏高/偏低/正常）。

### TC04: 简单策略回测 (Backtest)
**目标**：验证较复杂的 Python 逻辑 (Shift, Cumsum) 与 Glass Box 透明度。

*   **用户输入**:
    > "回测一下沪深300期货：每天收盘买入，第二天开盘卖出，看看过去5年收益如何。"
*   **预期行为**:
    1.  **语义映射**: 沪深300 -> `KQ.m@CFFEX.IF`。
    2.  **MCP 调用**: 获取 5 年日线数据。
    3.  **Python 逻辑**:
        *   `df['ret'] = df['open'].shift(-1) - df['close']` (注意滑点/手续费可先忽略或设为0)。
        *   `df['cum_ret'] = df['ret'].cumsum()`。
    4.  **输出结果**:
        *   图表：资金曲线 (Cumulative Return)。
        *   代码展示：必须清晰展示 `shift(-1)` 的逻辑，证明没有用到未来函数。

### TC04-B: 统计套利回测 (Statistical Arbitrage)
**目标**：验证基于统计规律（均值回归）的交易信号生成与收益回测。

*   **用户输入**:
    > "分析螺纹钢和铁矿石过去3年的价差。策略：当价差超过2倍标准差时做空价差（卖螺纹买铁矿），回归均值时平仓；反之做多。回测这个策略的收益。"
*   **预期行为**:
    1.  **语义映射**: 识别螺纹钢 (`KQ.m@SHFE.rb`) 和 铁矿石 (`KQ.m@DCE.i`)。
    2.  **MCP 调用**: 获取两者过去3年日线数据。
    3.  **Python 逻辑**:
        *   **数据对齐与价差计算**: `df['spread'] = df['close_rb'] - k * df['close_i']` (k需估算或由Prompt指定)。
        *   **统计指标**: 计算价差的滚动均值 (`rolling_mean`) 和标准差 (`rolling_std`)。
        *   **信号生成**:
            *   Short Signal: `spread > mean + 2*std`
            *   Long Signal: `spread < mean - 2*std`
            *   Exit Signal: `spread` 回归 `mean`
        *   **收益计算**: 模拟持仓期间的每日盈亏 (`position * spread_change`) 并累加。
    4.  **输出结果**:
        *   图表：
            1.  价差序列与 Bollinger Bands (上下轨)。
            2.  策略累计收益曲线。
        *   结论：报告总收益、最大回撤及交易次数。

---

## 2. 边界与异常处理用例 (Edge Cases)

### TC05: 模糊品种识别 (Ambiguity)
**目标**：测试 LLM 在用户输入不明确时的默认行为。

*   **用户输入**:
    > "看看纯碱的价格。"
*   **预期行为**:
    *   系统应默认映射到 **主力连续合约** (`KQ.m@CZCE.SA`)，而不是某个即将过期的具体合约。
    *   或者，系统在结论中注明：“已为您展示纯碱**主力连续**合约的走势。”

### TC06: 数据获取失败/不足 (Data Failure)
**目标**：测试 Python 沙箱的容错性 (Self-Correction)。

*   **用户输入**:
    > "分析一个不存在的合约，比如 SHFE.rb9999"
*   **模拟场景**: MCP 工具返回 Error 或空 CSV。
*   **预期行为**:
    1.  Python 脚本在读取 CSV 时捕获异常 (EmptyDataError)。
    2.  **自我修复**:
        *   方案 A: 尝试调用 `tq_get_quote` 确认合约是否存在。
        *   方案 B: 如果是主力合约数据为空，尝试缩短时间范围重试。
    3.  **最终输出**: 如果无法修复，向用户友善报错：“无法获取指定合约数据，请检查合约代码是否正确。”（而不是直接抛出 Python Traceback）。

### TC07: 数据对齐失败 (Alignment Issue)
**目标**：测试多品种分析时的健壮性。

*   **场景**: 用户请求对比“螺纹钢”和“标普500指数”（假设系统支持）。
*   **问题**: 两者交易时间完全不同（时区、假期）。
*   **预期行为**:
    *   Python 代码必须包含 `dropna()` 逻辑，剔除无法对齐的日期，并在结论中提示：“已剔除 15% 无法对齐的交易日数据。”

---

## 3. 验收标准 (Acceptance Criteria)

所有用例必须满足以下核心标准才能视为“通过”：

1.  **JSON 协议合规**: 所有输出必须符合 `LLM_提示词_FinSightAI.md` 中定义的 JSON 结构，包含 `semantic_mapping`, `python_sandbox_code` 等字段。
2.  **代码可复现 (Reproducibility)**: 用户复制 `python_sandbox_code` 中的代码到本地 Python 环境（安装 pandas/matplotlib 后），配合相同的 CSV 数据，**必须** 能跑通并生成相同的图表。
3.  **零幻觉数据**: 图表和结论中的数值（如相关系数 0.85）必须是 Python 代码真实计算的结果，严禁 LLM 直接编造数值。
4.  **Glass Box**: 必须提供数据预览（Top 5 rows）和完整代码，确保用户可核查。

