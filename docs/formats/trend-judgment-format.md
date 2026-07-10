# 趋势判断数据格式

**文件路径**: `trends/trend_judgment_{name}_{YYYY-MM-DD_HH-MM-SS}.json`

文件名中的 `{name}` 为股票名称，日期为人类可读格式。每只股票每次保存会创建一个新文件。

## 数据结构

```json
{
  "name": "星湖科技",
  "stockCode": "600866",
  "createTime": "2025-01-01T12:00:00.000Z",
  "trendJudgment": "unset",
  "trendJudgmentUpdatedAt": "2025-01-01T12:00:00.000Z",
  "autoTrendJudgment": "up",
  "autoTrendJudgmentUpdatedAt": "2025-01-01T12:00:00.000Z",
  "volatilityMetrics": {
    "daily_price_volatility": 0.0152,
    "price_range_volatility": 0.0205,
    "close_position_volatility": 0.0180,
    "volatility_5d_ma": 0.0160,
    "volatility_10d_ma": 0.0170,
    "volatility_15d_ma": 0.0165,
    "max_high_price": 10.85,
    "current_price": 10.50,
    "price_drop_value": 0.35,
    "price_drop_ratio": 3.23
  },
  "volatilityMetricsUpdatedAt": "2025-01-01T12:00:00.000Z",
  "trendStrengthMetrics": {
    "adx": 32.50,
    "plus_di": 28.10,
    "minus_di": 14.30
  },
  "trendStrengthMetricsUpdatedAt": "2025-01-01T12:00:00.000Z",
  "fiveYearAverageDividendYield": 3.2,
  "fiveYearAverageDividendYieldUpdatedAt": "2025-01-01T12:00:00.000Z",
  "previousDayChangePercent": 1.25,
  "previousDayChangeDate": "2025-01-01T00:00:00.000Z",
  "stockAnalysis": {
    "name": "星湖科技",
    "price": 0,
    "changePercent": 0,
    "action": "观望",
    "reason": "暂无6月10日交易数据，生物科技需后续跟踪",
    "reportDate": "2026-06-11"
  },
  "stockAnalysisUpdatedAt": "2025-01-01T12:00:00.000Z"
}
```

## 字段说明

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `name` | `string` | 是 | 股票名称 |
| `stockCode` | `string` | 是 | 股票代码 |
| `createTime` | `string` | 是 | 创建时间，ISO 8601 格式 |
| `trendJudgment` | `string` | 否 | 手动趋势判断，取值见下方趋势类型表 |
| `trendJudgmentUpdatedAt` | `string` | 否 | 手动趋势判断更新时间，ISO 8601 格式 |
| `autoTrendJudgment` | `string` | 否 | 自动趋势判断，取值见下方趋势类型表 |
| `autoTrendJudgmentUpdatedAt` | `string` | 否 | 自动趋势判断更新时间，ISO 8601 格式 |
| `volatilityMetrics` | `object` | 否 | 波动率指标集合 |
| `volatilityMetrics.daily_price_volatility` | `number` | 否 | 当日价格波动率 |
| `volatilityMetrics.price_range_volatility` | `number` | 否 | 价格区间波动率 |
| `volatilityMetrics.close_position_volatility` | `number` | 否 | 收盘价相对位置波动率 |
| `volatilityMetrics.volatility_5d_ma` | `number` | 否 | 5天滚动平均波动率 |
| `volatilityMetrics.volatility_10d_ma` | `number` | 否 | 10天滚动平均波动率 |
| `volatilityMetrics.volatility_15d_ma` | `number` | 否 | 15天滚动平均波动率 |
| `volatilityMetrics.max_high_price` | `number` | 否 | 计算周期内（90天）的最高价 |
| `volatilityMetrics.current_price` | `number` | 否 | 当前收盘价 |
| `volatilityMetrics.price_drop_value` | `number` | 否 | 最高价到当前价格的下跌值 |
| `volatilityMetrics.price_drop_ratio` | `number` | 否 | 最高价到当前价格的下跌比率（%） |
| `volatilityMetricsUpdatedAt` | `string` | 否 | 波动率指标整体更新时间 |
| `trendStrengthMetrics` | `object` | 否 | ADX 趋势强度指标集合（见下方说明） |
| `trendStrengthMetrics.adx` | `number` | 否 | ADX（平均趋向指数）趋势强度，取值 0–100；`<20` 震荡、`20–25` 趋势初现、`25–50` 强趋势、`>50` 极强趋势；数据不足时为 `null` |
| `trendStrengthMetrics.plus_di` | `number` | 否 | +DI（上升方向指标），取值 0–100；高于 `minus_di` 表示上升动能占优；数据不足时为 `null` |
| `trendStrengthMetrics.minus_di` | `number` | 否 | −DI（下降方向指标），取值 0–100；高于 `plus_di` 表示下降动能占优；数据不足时为 `null` |
| `trendStrengthMetricsUpdatedAt` | `string` | 否 | 趋势强度指标更新时间 |
| `fiveYearAverageDividendYield` | `number` | 否 | 5年平均股息率（%） |
| `fiveYearAverageDividendYieldUpdatedAt` | `string` | 否 | 5年平均股息率更新时间 |
| `previousDayChangePercent` | `number` | 否 | 前一交易日涨跌百分比（%） |
| `previousDayChangeDate` | `string` | 否 | 前一交易日日期 |
| `stockAnalysis` | `object` | 否 | 股票分析报告（来自 stock-reports） |
| `stockAnalysis.name` | `string` | 否 | 股票名称（来自分析报告） |
| `stockAnalysis.price` | `number` | 否 | 分析时价格 |
| `stockAnalysis.changePercent` | `number` | 否 | 分析时涨跌幅（%，如 "+1.01%" 对应数值 1.01） |
| `stockAnalysis.action` | `string` | 否 | 操作建议，取值见下方 action 值表 |
| `stockAnalysis.reason` | `string` | 否 | 分析理由（文本描述） |
| `stockAnalysis.reportDate` | `string` | 否 | 分析报告日期（来自 latest.json 顶层 date 字段） |
| `stockAnalysisUpdatedAt` | `string` | 否 | 分析报告更新时间，ISO 8601 格式 |

### 操作建议值（stockAnalysis.action）

来自 stock-reports/latest.json 的分析报告，实际出现的取值：

| 值 | 含义 |
|----|------|
| `买入关注` | 建议重点关注并考虑买入 |
| `持有` | 建议继续持有 |
| `观望` | 建议观望，方向待确认 |
| `回避` | 建议回避，趋势不利 |
| `卖出` | 建议卖出 |
| `买入` | 建议买入 |
| `减仓` | 建议减少持仓 |

## 趋势类型

| 趋势值 | 说明 |
|--------|------|
| `unset` | 未设置 |
| `unknown` | 未知 |
| `up` | 上涨 |
| `down` | 下跌 |
| `breakdown` | 下跌破位 |
| `oscillation` | 震荡 |
| `pullback` | 回踩 |
