演示地址：https://shinny-xuyida.github.io/test-mcp-site/

为了兼容“低阶/非编程用户”，我们将产品形态从单纯的“API/MCP插件”升级为**“FinSight AI 智能投研终端” (Web Application)**。

这个 Web 应用的后端依然由 MCP 服务驱动，但前端提供了一个“开箱即用”的图形化界面，封装了 Prompt 工程、代码执行环境和图表渲染引擎。

---

# FinSight AI 智能投研终端 - 产品设计方案 (Web版)

## 1. 产品概述与定位变化

*   **产品形态**：Web SaaS 应用（网站）
*   **核心定位**：面向**个人交易者**的“对话式”量化验证工具。
*   **核心理念**：**“你的想法，数据的答案。”** (Your Idea, Data's Answer.)
*   **解决痛点**：用户不懂代码，但需要用真实历史数据验证交易直觉（如价差、简单的日内策略、均线规律）。

---

## 2. 用户体验流程 (User Journey)

### 2.1 首页：灵感激发 (Inspiration)
用户打开网站，首先看到的不是空白的对话框，而是**“灵感卡片墙”**（预设模版）。
*   **针对小白用户**：点击卡片直接运行。
*   **针对进阶用户**：在对话框输入自定义问题。

### 2.2 交互：对话与执行
1.  用户输入（或点击）：“螺纹钢和铁矿石过去三年的价格走势对比，并画出价差图。”
2.  **语义解析 (Thinking)**：LLM 根据预置知识库（System Prompt），将“螺纹钢”映射为 `KQ.m@SHFE.rb`，将“铁矿石”映射为 `KQ.m@DCE.i`。
3.  **数据获取 (Fetching)**：调用 MCP 工具获取这两个合约的日线数据。
4.  **结果呈现**：
    *   **主视觉**：一张交互式折线图（可缩放、拖拽）。
    *   **文字结论**：AI 总结的分析（例如：“两者相关性系数为 0.85，当前价差处于历史 90% 分位，属于高估区间”）。

### 2.3 深度探索：透明化 (Glass Box)
*   界面默认隐藏代码。
*   提供**“查看分析逻辑” (Show Code)** 按钮。点击后侧边栏展开，高亮显示 Python 代码，方便用户学习或核对逻辑。

---

## 3. 核心功能模块设计

### 3.1 首页“灵感库” (Preset Examples & Execution Logic)
这是降低门槛的关键。我们需要展示“自然语言”是如何一步步转化为“代码执行”的。

#### **类别 A：相关性与价差分析 (Spread & Correlation)**
*   **卡片标题**：RB vs I 黑色系联动
*   **用户指令**：`获取过去3年螺纹钢(RB)和铁矿石(I)的主力合约收盘价，计算它们的相关性系数，并绘制归一化的价格走势对比图，以及它们的价差序列图。`
*   **幕后执行逻辑 (Behind the Scenes)**：
    1.  **LLM 映射**：识别 "螺纹钢" -> `KQ.m@SHFE.rb` (主力连续), "铁矿石" -> `KQ.m@DCE.i`。
    2.  **MCP 调用**：
        *   `tq_get_kline(symbol="KQ.m@SHFE.rb", duration_seconds=86400, data_length=750)`
        *   `tq_get_kline(symbol="KQ.m@DCE.i", duration_seconds=86400, data_length=750)`
    3.  **Python 计算**：Pandas 读取 CSV -> `df.corr()` 计算相关性 -> Matplotlib 绘图。

#### **类别 B：跨期/跨品种套利 (Arbitrage)**
*   **卡片标题**：玻璃(FG)跨期套利分布
*   **用户指令**：`获取玻璃期货当前的主力合约(2501)与次主力合约(2505)的价差数据。统计价差的均值、标准差，绘制价差分布直方图。`
*   **幕后执行逻辑 (Behind the Scenes)**：
    1.  **LLM 映射**：识别 "玻璃" -> `CZCE.FG`。识别 "2501" -> `CZCE.FG501`, "2505" -> `CZCE.FG505`。
    2.  **MCP 调用**：
        *   `tq_get_quote("CZCE.FG501")` (获取合约乘数，计算盈亏用)
        *   `tq_get_kline("CZCE.FG501", ...)`
        *   `tq_get_kline("CZCE.FG505", ...)`
    3.  **Python 计算**：`spread = close1 - close2` -> `seaborn.histplot(spread)`。

#### **类别 C：简单策略回测 (Simple Backtest)**
*   **卡片标题**：股指期货“隔夜持仓”效应
*   **用户指令**：`获取IF（沪深300）股指期货过去5年的日线数据。假设策略为：每天收盘价买入，次日开盘价卖出。请计算累计收益率曲线。`
*   **幕后执行逻辑 (Behind the Scenes)**：
    1.  **LLM 映射**：识别 "IF/沪深300" -> `KQ.m@CFFEX.IF`。
    2.  **MCP 调用**：
        *   `tq_get_kline("KQ.m@CFFEX.IF", duration_seconds=86400, data_length=1200)`
    3.  **Python 计算**：`df['ret'] = df['open'].shift(-1) - df['close']` -> `df['cum_ret'] = df['ret'].cumsum()`。

---

## 4. 界面原型设计 (Wireframe Descriptions)

### 4.1 布局结构
采用类似 **Claude Artifacts** 或 **Jupyter Notebook** 的左右/上下分栏布局。

*   **左侧 (30%)**：对话流 (Chat Stream)
    *   用户输入框。
    *   历史对话气泡。
    *   简短的文字结论。
*   **右侧 (70%)**：画布 (Canvas/Artifact)
    *   **顶部**：图表展示区（核心视觉）。
    *   **中部**：详细数据表格（Top 5 行预览，支持下载完整 CSV）。
    *   **底部（可折叠）**：Python 代码控制台。

### 4.2 关键交互细节
1.  **加载状态**：当后台在跑 Python 时，显示进度条：“正在获取 IF 主力数据 (30%)” -> “正在计算策略收益 (60%)” -> “正在生成图表 (90%)”。
2.  **错误处理**：如果代码报错（如数据缺口），AI 需自动自我修复（Self-Correction）并重试。

---

## 5. MCP 工具协议标准 (MCP Tool Protocol)

为了简化 Python 编程复杂度，我们将 TQSDK 的复杂功能封装为**“原子化数据工具”**。LLM 不需要学习如何建立 TQSDK 连接，只需要调用以下工具即可获得 DataFrame 就绪的数据。

### 5.1 通用设计原则
*   **Symbol 格式**：严格遵循 TQSDK 格式。
    *   主力连续：`KQ.m@<Exchange>.<Product>` (如 `KQ.m@SHFE.rb`)
    *   具体合约：`<Exchange>.<Product><YearMonth>` (如 `SHFE.rb2510`)
*   **数据返回**：所有数据接口默认返回 **CSV 格式字符串**，包含表头，方便 Python `pandas.read_csv(io.StringIO(res))` 直接读取。

### 5.2 核心工具定义

#### Tool 1: `tq_get_quote` (获取合约元数据)
*   **描述**：获取合约的基础信息，如合约乘数、最小变动价位等。用于计算盈亏金额或验证合约是否存在。
*   **参数 Parameters**:
    *   `symbol` (string, required): 合约代码 (e.g., "SHFE.rb2510", "KQ.m@SHFE.rb")
*   **返回 Return (JSON)**:
    ```json
    {
        "symbol": "SHFE.rb2510",
        "product_name": "螺纹钢",
        "volume_multiple": 10,      // 合约乘数 (1手10吨)
        "price_tick": 1.0,          // 最小变动价位
        "margin_ratio": 0.08,       // 保证金比例
        "expired": false
    }
    ```

#### Tool 2: `tq_get_kline` (获取历史K线)
*   **描述**：获取指定合约的历史 K 线数据。
*   **参数 Parameters**:
    *   `symbol` (string, required): 合约代码。
    *   `duration_seconds` (integer, required): K线周期（秒）。
        *   60: 1分钟
        *   3600: 1小时
        *   86400: 日线
    *   `data_length` (integer, optional): 数据长度，默认 200，最大 5000。
*   **返回 Return (CSV String)**:
    ```csv
    datetime,open,high,low,close,volume,open_interest
    2024-01-01 09:00:00,3800.0,3820.0,3790.0,3810.0,15000,500000
    2024-01-02 09:00:00,3810.0,3850.0,3800.0,3835.0,18000,505000
    ...
    ```

---

## 6. 技术架构方案 (Technical Architecture)

为了支持“模糊意图”到“精确执行”的转化，我们在 LLM 层增加了**System Prompt 知识库**。

```mermaid
graph TD
    User[Web用户] -- "看看螺纹钢" --> Frontend
    Frontend -- "System Prompt + User Query" --> LLM
    
    subgraph "Step 1: 语义映射 (LLM Thinking)"
        LLM -- 查表: 螺纹钢=SHFE.rb --> SymbolResolved[确定代码: KQ.m@SHFE.rb]
    end
    
    subgraph "Step 2: 数据获取 (MCP)"
        SymbolResolved -- Tool Call --> MCP_Server[FinSight MCP (TQSDK)]
        MCP_Server -- TQSDK API --> FutureMarket[期货交易所]
        FutureMarket -- Returns CSV --> LLM
    end
    
    subgraph "Step 3: 代码执行 (Sandbox)"
        LLM -- 生成 Python 代码 --> CodeSandbox[Python 容器]
        CodeSandbox -- 执行并绘图 --> Result[JSON/Image]
    end
    
    Result --> Frontend
```

*   **System Prompt 注入**：
    我们需要在 Prompt 中维护一份核心**品种代码映射表**（例如：螺纹=SHFE.rb, 铁矿=DCE.i, 沪深300=CFFEX.IF）。这样 LLM 就能在不请求后端搜索接口的情况下，自行完成“自然语言”到“合约代码”的转换。

---

## 7. MVP 开发计划 (基于 Web 方案)

### 第一阶段：核心闭环 (Week 1-4)
*   **数据**：支持 System Prompt 中预置的 10-20 个主流品种。
*   **功能**：
    *   实现 `tq_get_kline` 和 `tq_get_quote` 两个 MCP 工具。
    *   前端对话框 + Matplotlib 静态图展示。

### 第二阶段：交互增强 (Week 5-8)
*   **数据**：全量国内期货。
*   **功能**：
    *   前端渲染 ECharts 交互图表。
    *   增加“灵感卡片”首页。

---

## 8. 总结

通过**“LLM 语义映射 + MCP 原子化工具”**的组合，我们极大降低了系统的复杂度。MCP 服务只需要做好一件事：**忠实地执行 TQSDK 的数据查询**，而将复杂的意图理解和代码生成交给 LLM 处理。
