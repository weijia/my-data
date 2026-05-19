# My-Data 数据格式规范

本文档定义了 my-data 仓库中使用的所有数据文件格式规范。

## 目录结构

```
my-data/
├── README.md                          # 项目说明
├── docs/
│   └── formats/
│       ├── data-formats.md            # 本文件：数据格式规范
│       ├── kids-point-format.md         # Kids-Point 积分系统格式
│       ├── webdav-data-formats.md     # WebDAV 同步数据格式
│       └── my-quant-strategy-format.md # My-Quant 策略数据格式
├── stocks/                            # 股票基础数据
│   ├── stock_data.json                # 股票基础信息
│   └── watchlist.json                 # 关注列表
├── strategies/                        # 策略数据
│   ├── all_strategies.json            # 综合策略（单文件模式）
│   ├── conditional_strategies.json    # 条件单策略
│   ├── grid_strategies.json           # 网格策略
│   └── advanced_strategies.json       # 高级策略
├── holdings/                          # 持仓数据
│   └── holdings.default.{timestamp}.json  # 持仓历史
├── config/                            # 配置数据
│   └── window_config.json             # 窗口配置
└── trends/                            # 趋势判断数据
    └── trend_judgment_{name}_{date}.json
```

## 文件格式汇总

| 文件 | 用途 | 同步模式 |
|------|------|----------|
| `stocks/stock_data.json` | 股票基础信息 | 多文件模式 |
| `stocks/watchlist.json` | 关注列表 | WebDAVStockManager |
| `strategies/all_strategies.json` | 综合策略 | 单文件模式（默认） |
| `strategies/conditional_strategies.json` | 条件单策略 | 多文件模式 |
| `strategies/grid_strategies.json` | 网格策略 | 多文件模式 |
| `strategies/advanced_strategies.json` | 高级策略 | 多文件模式 |
| `trends/trend_judgment_{name}_{date}.json` | 趋势判断 | 独立同步 |
| `holdings/holdings.default.{timestamp}.json` | 持仓历史/导出 | 独立同步 |
| `config/window_config.json` | 窗口配置 | 独立同步 |
| `docs/formats/kids-point-format.md` | Kids-Point 积分系统 | 独立文档 |

> **同步模式说明**：单文件模式将所有策略数据合并到一个文件中同步（默认）；多文件模式将各类型策略分别保存到独立文件。

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
| `stockData` | `Array<StockItem>` | 股票基础信息数组（见第2节） |
| `conditionalStrategies` | `Array<ConditionalStrategy>` | 条件单策略数组（见第3节） |
| `gridStrategies` | `Array<GridStrategy>` | 网格策略数组（见第4节） |
| `advancedStrategies` | `Array<AdvancedStrategy>` | 高级策略数组（见第5节） |
| `lastUpdated` | `string` | 最后更新时间，ISO 8601 格式 |

---

## 2. 股票基础信息

**文件路径**: `stocks/stock_data.json`

### StockItem 结构

```json
{
  "id": "1700000000000abc12",
  "stockName": "星湖科技",
  "stockCode": "600866",
  "accountType": "default"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一ID，格式：`时间戳 + 随机5位字符` |
| `stockName` | `string` | 是 | 股票名称 |
| `stockCode` | `string` | 是 | 股票代码（如 "600866"） |
| `accountType` | `string` | 是 | 账户类型：`"default"`（普通）或 `"credit"`（信用），默认 `"default"` |

---

## 3. 条件单策略

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

## 4. 网格策略

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

## 5. 高级策略

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

---

## 6. 趋势判断

**文件路径**: `trends/trend_judgment_{name}_{YYYY-MM-DD_HH-MM-SS}.json`

文件名中的 `{name}` 为股票名称，日期为人类可读格式。每只股票每次保存会创建一个新文件。

### 趋势判断数据结构

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

### 趋势类型

| 趋势值 | 说明 |
|--------|------|
| `unset` | 未设置 |
| `unknown` | 未知 |
| `up` | 上涨 |
| `down` | 下跌 |
| `breakdown` | 下跌破位 |
| `oscillation` | 震荡 |
| `pullback` | 回踩 |

---

## 7. 持仓历史记录

**文件路径**: `holdings/holdings.default.{timestamp}.json`

文件名中的 `{timestamp}` 为 ISO 8601 格式时间戳。

### 持仓历史记录结构

```json
{
  "historyEntry": { ... },
  "exportTime": "2025-01-01T12:00:00.000Z",
  "version": "1.0",
  "type": "holdings_history"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `historyEntry` | `object` | 持仓历史条目 |
| `exportTime` | `string` | 导出时间，ISO 8601 格式 |
| `version` | `string` | 数据版本号 |
| `type` | `string` | 数据类型标识：`"holdings_history"` 或 `"holdings_export"` |

---

## 8. 窗口配置

**文件路径**: `config/window_config.json`

### 窗口配置数据结构

```json
{
  "top": "50vh",
  "left": "50vw",
  "width": "600px",
  "height": "200px"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `top` | `string` | 窗口顶部位置（CSS 值，如 `"50vh"`、`"0px"`） |
| `left` | `string` | 窗口左侧位置（CSS 值，如 `"50vw"`、`"0px"`、`"auto"`） |
| `width` | `string` | 窗口宽度（CSS 值，如 `"600px"`、`"100vw"`、`"300px"`） |
| `height` | `string` | 窗口高度（CSS 值，如 `"200px"`、`"100vh"`、`"40px"`） |

> **注意**: 停靠模式下可能包含 `right` 字段（如 `"0px"`）。

---

## 9. 关注列表

**文件路径**: `stocks/watchlist.json`

```json
{
  "watchList": [ "600866", "000001", "000002" ],
  "lastUpdated": "2025-01-01T12:00:00.000Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `watchList` | `Array<string>` | 关注的股票代码列表 |
| `lastUpdated` | `string` | 最后更新时间，ISO 8601 格式 |

---

## 相关文档

- [Kids-Point 积分系统格式](./kids-point-format.md) - Kids-Point 应用数据格式
- [WebDAV 数据格式](./webdav-data-formats.md) - WebDAV 同步的数据格式
- [My-Quant 策略格式](./my-quant-strategy-format.md) - My-Quant 应用策略数据格式

---

## 数据同步架构

```
                    ┌─────────────────────┐
                    │   WebDAV Server     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────┴──────┐  ┌─────┴──────┐  ┌──────┴─────────┐
    │ 单文件模式      │  │ 独立同步    │  │ 股票数据       │
    │ all_strategies │  │ (趋势/持仓  │  │ (data/watch)   │
    │ .json          │  │  /窗口)     │  │                │
    └────────────────┘  └────────────┘  └───────────────┘
```

**同步流程**：
1. **单文件模式（默认）**: 将所有策略数据合并上传到 `strategies/all_strategies.json`
2. **多文件模式**: 各类型策略分别保存到独立文件
3. **独立同步**: 趋势判断、持仓、窗口配置有独立的同步逻辑
4. **上传前自动备份**: 会在上传前将现有文件备份到 `.backup/` 目录（窗口配置文件除外）
