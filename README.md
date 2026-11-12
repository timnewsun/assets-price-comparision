# Assets Price Comparision Skill

这是一个可开源的 Codex Skill，用于比较多个资产或市场指数在不同期限内的价格表现和历史波动率。Skill 从 Yahoo Finance 获取日度数据，保留可复核的 CSV 数据，生成带统计说明的图表，并汇总为严格两页的 PDF 报告。

Skill 名称在界面中显示为 `Assets Price Comparision Skill`，目录名遵循 Codex Skill 的命名规则：`assets-price-comparision-skill`。

## 1. 项目解决什么问题

不同资产的价格单位和绝对数值通常不可直接比较。例如，纳斯达克综合指数、标普 500、股票和 Bitcoin 的价格尺度不同，仅把原始价格放在同一张图中会产生误导。

本项目解决以下问题：

- 从统一的数据源获取不同资产或指数的日度价格；
- 比较最近一周、一个月、半年和一年的价格变化；
- 用年化历史波动率衡量不同资产的价格波动程度；
- 将不同价格尺度的资产按共同起点归一化为 100，比较实际走势；
- 同时保存原始数据、统计结果、图表和报告，便于复核和二次分析。

结果是历史数据分析，不构成投资建议。

## 2. 主要功能

- 支持输入资产名称、Yahoo Finance ticker，或两者混合；例如 `Bitcoin`、`BTC-USD`、`^IXIC`、`^GSPC`、`AAPL`。
- 使用 Yahoo Finance 日度数据，并记录实际使用的 ticker、数据源、价格字段和获取时间。
- 分析四个日历窗口：最近 7 天、最近 1 个日历月、最近 6 个日历月和最近 1 个日历年。
- 计算期初价格、期末价格、绝对变动、百分比变动、观测数量和年化历史波动率。
- 为每个资产生成四张单资产价格折线图；图中标注价格变动和年化波动率。
- 为每个期限生成一张归一化比较图；所有资产在该图的第一个共同日期从 100 开始。
- 保存每个资产的原始价格 CSV、合并价格 CSV、四个归一化 CSV 和统计汇总 CSV。
- 生成严格两页的 `asset_comparison_report.pdf`：第 1 页为数据、统计表和方法说明，第 2 页为四张比较图和简短解读。
- 单个 ticker 下载失败或某个期限数据不足时，尽量保留其他结果，并在汇总中标记问题。

波动率使用日简单收益率的样本标准差年化计算：

```text
annualized volatility = daily return std(ddof=1) × sqrt(annualization days)
```

交易日资产和指数使用 252 天；全年交易的资产（例如加密货币）使用 365 天。

## 3. 使用方法

### 在 Codex 中调用

将 `assets-price-comparision-skill` 放到 Codex 可发现的 Skill 目录，然后在对话中使用：

```text
使用 $assets-price-comparision-skill 比较纳斯达克综合指数和标普 500 指数，生成 CSV、图表和两页 PDF 报告。
```

也可以直接提供 ticker：

```text
使用 $assets-price-comparision-skill 比较 ^IXIC、^GSPC、AAPL 和 BTC-USD。
```

对于可能对应多个 ticker 的名称，应在结果中说明选择的 ticker；如果不同解释会显著改变结果，应先请求澄清。

### 运行环境

执行分析需要能够访问 Yahoo Finance，并安装常用 Python 依赖：

```bash
python3 -m pip install yfinance pandas numpy matplotlib pypdf
```

Skill 默认使用一次性的 Python 执行过程，不要求在 Skill 目录中添加永久分析脚本，也不需要 Yahoo Finance 凭证。

## 4. 输入输出示例

### 输入示例

```text
比较纳斯达克综合指数和标普 500 指数，分析最近一周、一个月、半年和一年的价格变化与波动率，并生成完整报告。
```

本例解析为：

```text
Nasdaq Composite -> ^IXIC
S&P 500          -> ^GSPC
```

### 输出目录示例

每次未指定输出目录时，Skill 使用新的运行目录，避免覆盖历史结果：

```text
asset_analysis_<description>_<date>/
├── csv_data/
│   ├── <asset>.csv
│   ├── all_assets_close_prices.csv
│   ├── normalized_1_week.csv
│   ├── normalized_1_month.csv
│   ├── normalized_6_months.csv
│   └── normalized_1_year.csv
├── single_asset_charts/
│   ├── <asset>_1_week.png
│   ├── <asset>_1_month.png
│   ├── <asset>_6_months.png
│   └── <asset>_1_year.png
├── comparison_charts/
│   ├── comparison_1_week.png
│   ├── comparison_1_month.png
│   ├── comparison_6_months.png
│   └── comparison_1_year.png
├── summary_statistics.csv
├── run_metadata.json
└── asset_comparison_report.pdf
```

已验证的实际运行示例位于：

`asset_analysis_nasdaq_composite_vs_sp500_2026-11-12/`

其中 `summary_statistics.csv` 的每一行包含资产、ticker、期限、实际起止日期、起止价格、绝对变动、百分比变动、年化波动率、观测数量和数据状态。归一化 CSV 的首行中，每个资产的数值均为 100。

两页报告的内容为：

1. 数据来源、获取时间、四个期限的“价格变动 / 年化波动率”汇总表、计算方法和限制；
2. 四张归一化价格比较图，以及表现强弱和波动率的简短解读。

### 开发验证

验证 Skill 结构：

```bash
python3 <codex-skills-root>/skill-creator/scripts/quick_validate.py assets-price-comparision-skill
```

验证实际结果时，应确认每个成功 ticker 都有非空 CSV、四个期限的统计数据、四张比较图，以及页数恰好为 2 的 PDF。
