# 策略数据格式

本文档描述 my-data 仓库中各类交易策略的数据格式。

---

## 1. 综合策略单文件 `strategies/all_strategies.json`

**说明**: 默认同步模式，将所有策略数据合并为一个文件。

```json
{
  "stockData": [ ... ],
  "conditionalStrategies": [ ... ],
  "gridStrategies": [ ... ],
  "advancedStrategies": [ ... ],
  "lastUpdated": "2025-01-01T12:00:00.000Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `stockData` | `Array<StockItem>` | 股票基础信息数组（见 [stock-data.md](./stock-data.md)） |
| `conditionalStrategies` | `Array<ConditionalStrategy>` | 条件单策略数组（见第2节） |
| `gridStrategies` | `Array<GridStrategy>` | 网格策略数组（见第3节） |
| `advancedStrategies` | `Array<AdvancedStrategy>` | 高级策略数组（见第4节） |
| `lastUpdated` | `string` | 最后更新时间，ISO 8601 格式 |

---

## 2. 条件单策略

**文件路径**: `strategies/conditional_strategies.json`

### ConditionalStrategy 结构

```json
{
  "id": "1700000000000abc12",
  "stockName": "星湖科技",
  "stockCode": "600866",
  "deltaPercentage": "5",
  "deltaAmount": "0.500",
  "bounceType": "percentage",
  "tradeVolume": "100",
  "targetPrice": "10.000",
  "notes": "",
  "accountType": "credit",
  "strategyType": "bottom_buy",
  "side": "MARGINBUY",
  "createDate": "01-15 10:30:00",
  "expiredTime": "2025-03-15",
  "status": "active"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一ID |
| `stockName` | `string` | 是 | 股票名称 |
| `stockCode` | `string` | 是 | 股票代码 |
| `deltaPercentage` | `string` | 否 | 涨跌百分比 |
| `deltaAmount` | `string` | 否 | 涨跌金额，精确到小数点后3位 |
| `bounceType` | `string` | 否 | 反弹类型：`"percentage"` 或 `"amount"` |
| `tradeVolume` | `string` | 否 | 交易数量 |
| `targetPrice` | `string` | 否 | 目标价格，精确到小数点后3位 |
| `notes` | `string` | 否 | 备注 |
| `accountType` | `string` | 是 | 账户类型：`"default"` 或 `"credit"` |
| `strategyType` | `string` | 是 | 策略类型：`"bottom_buy"` 或 `"top_sell"` |
| `side` | `string` | 否 | 交易方向（见下方 side 值表） |
| `createDate` | `string` | 否 | 创建时间，格式：`"MM-dd HH:mm:ss"` |
| `expiredTime` | `string` | 否 | 过期时间，格式：`"YYYY-MM-DD"` |
| `status` | `string` | 否 | 状态，默认 `"active"` |

### side 值表

| 账户类型 | 买入方向 | 卖出方向 |
|---------|---------|---------|
| 普通账户 | `BUY` | `SELL` |
| 信用账户-担保品 | `COLLABUY` | `COLLASELL` |
| 信用账户-融资融券 | `MARGINBUY` | `MARGINSELL` |

---

## 3. 网格策略

**文件路径**: `strategies/grid_strategies.json`

### GridStrategy 结构

```json
{
  "id": "1700000000000abc12",
  "stockName": "星湖科技",
  "stockCode": "600866",
  "priceRange": "10-20",
  "gridSpacing": "0.5",
  "gridCount": "10",
  "gridAmount": "10000",
  "quantityBuy": "100",
  "quantitySell": "100",
  "totalAmount": "50000",
  "accountType": "default",
  "notes": ""
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一ID，格式：`时间戳 + 随机5位字符` |
| `stockName` | `string` | 是 | 股票名称 |
| `stockCode` | `string` | 是 | 股票代码 |
| `priceRange` | `string` | 否 | 价格区间（如 "10-20"） |
| `gridSpacing` | `string` | 否 | 网格间距 |
| `gridCount` | `string` | 否 | 网格数量 |
| `gridAmount` | `string` | 否 | 每格金额 |
| `quantityBuy` | `string` | 否 | 买入数量 |
| `quantitySell` | `string` | 否 | 卖出数量 |
| `totalAmount` | `string` | 否 | 总金额 |
| `accountType` | `string` | 是 | 账户类型：`"default"` 或 `"credit"` |
| `notes` | `string` | 否 | 备注 |

---

## 4. 高级策略

**文件路径**: `strategies/advanced_strategies.json`

### AdvancedStrategy 结构

```json
{
  "id": "1700000000000",
  "name": "星湖科技",
  "isMarginAccount": false,
  "accountType": "default",
  "stockCode": "600866",
  "marketValue": "50000",
  "fiveYearAvgDividendYield": "3.2",
  "trendJudgment": "unset",
  "expiryDate": "",
  "oscillationGridSize": "0.5",
  "oscillationTradeAmount": "100",
  "breakoutGridSize": "1.0",
  "breakoutTradeAmount": "200",
  "decreaseSide": "COLLASELL",
  "decreaseStrategies": [],
  "increaseStrategies": [],
  "netPosition": 1000,
  "price": "10.50",
  "notes": "自动生成",
  "manualNotes": "",
  "createdAt": "2025-01-01T00:00:00.000Z",
  "updatedAt": "2025-01-01T12:00:00.000Z"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一ID，格式：时间戳字符串 |
| `name` | `string` | 是 | 策略名称（通常为股票名称） |
| `isMarginAccount` | `boolean` | 否 | 是否融资融券账户，默认 `false` |
| `accountType` | `string` | 否 | 账户类型：`"default"` 或 `"credit"` |
| `stockCode` | `string` | 否 | 股票代码 |
| `marketValue` | `string` | 否 | 市值 |
| `fiveYearAvgDividendYield` | `string` | 否 | 5年平均股息率 |
| `trendJudgment` | `string` | 否 | 趋势判断：`"unset"` / `"unknown"` / `"up"` / `"down"` / `"oscillation"` / `"pullback"` |
| `expiryDate` | `string` | 否 | 过期日期 |
| `oscillationGridSize` | `string` | 否 | 震荡网格大小 |
| `oscillationTradeAmount` | `string` | 否 | 震荡交易数量 |
| `breakoutGridSize` | `string` | 否 | 突破网格大小 |
| `breakoutTradeAmount` | `string` | 否 | 突破交易数量 |
| `decreaseSide` | `string` | 否 | 下跌卖出方向，如 `"COLLASELL"` |
| `decreaseStrategies` | `Array` | 否 | 下跌策略列表 |
| `increaseStrategies` | `Array` | 否 | 上涨策略列表 |
| `netPosition` | `number` | 否 | 净持仓量 |
| `price` | `string` | 否 | 当前价格 |
| `notes` | `string` | 否 | 备注（系统生成） |
| `manualNotes` | `string` | 否 | 手动备注 |
| `createdAt` | `string` | 是 | 创建时间，ISO 8601 格式 |
| `updatedAt` | `string` | 否 | 更新时间，ISO 8601 格式 |
