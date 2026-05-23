# 数据同步格式

## 单文件模式（all_strategies.json）

```json
{
  "stockData": [
    {
      "name": "贵州茅台",
      "stockCode": "600519",
      "accountType": "default"
    }
  ],
  "conditionalStrategies": [],
  "gridStrategies": [],
  "advancedStrategies": [
    {
      "id": "strategy_1750000000000_abc123",
      "name": "贵州茅台",
      "stockCode": "600519",
      "accountType": "default",
      "isMarginAccount": false,
      "netPosition": 100,
      "marketValue": "168000",
      "trendJudgment": "trend_up",
      "decreaseStrategies": [],
      "increaseStrategies": [],
      "createdAt": "2025-06-15T10:30:00.000Z",
      "updatedAt": "2025-06-15T14:20:00.000Z"
    }
  ],
  "lastUpdated": "2025-06-15T14:20:00.000Z"
}
```

## 字段映射

| my-quant 字段 | WebDAV 同步字段 | 说明 |
|---------------|-----------------|------|
| `name` | `name` / `stockName` | 策略名称 |
| `stockCode` | `stockCode` | 股票代码 |
| `accountType` | `accountType` | 账户类型 |
| `isMarginAccount` | `isMarginAccount` | 是否融资融券 |
| `netPosition` | `netPosition` | 净持仓 |
| `marketValue` | `marketValue` | 市值 |
| `trendJudgment` | `trendJudgment` | 趋势判断 |
| `decreaseStrategies` | `decreaseStrategies` | 下跌策略 |
| `increaseStrategies` | `increaseStrategies` | 上涨策略 |
| `manualNotes` | `manualNotes` / `notes` | 手动备注 |

## 趋势判断数据注入

策略加载时，从 WebDAV 趋势数据注入以下字段：

| 注入字段 | 来源字段 | 说明 |
|----------|----------|------|
| `currentPrice` | `volatilityMetrics.current_price` | 当前价格 |
| `price_drop_ratio` | `volatilityMetrics.price_drop_ratio` | 下跌百分比 |
| `trendJudgment` | `autoTrendJudgment` / `trendJudgment` | 趋势判断 |

[返回索引](../my-quant-strategy-format.md)
