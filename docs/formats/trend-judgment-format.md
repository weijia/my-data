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
  "fiveYearAverageDividendYield": 3.2,
  "fiveYearAverageDividendYieldUpdatedAt": "2025-01-01T12:00:00.000Z",
  "previousDayChangePercent": 1.25,
  "previousDayChangeDate": "2025-01-01T00:00:00.000Z"
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
| `fiveYearAverageDividendYield` | `number` | 否 | 5年平均股息率（%） |
| `fiveYearAverageDividendYieldUpdatedAt` | `string` | 否 | 5年平均股息率更新时间 |
| `previousDayChangePercent` | `number` | 否 | 前一交易日涨跌百分比（%） |
| `previousDayChangeDate` | `string` | 否 | 前一交易日日期 |

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
