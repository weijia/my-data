# 策略基础结构

## 核心字段

```typescript
interface Strategy {
  // 系统字段（由数据库自动生成）
  _id: string;           // 文档ID，格式: "strategy_{timestamp}_{random}"
  _rev?: string;         // 文档版本（PouchDB 内部使用）
  id?: string;           // 与 _id 相同，用于前端兼容
  createdAt: string;     // 创建时间，ISO 8601 格式
  updatedAt: string;     // 更新时间，ISO 8601 格式

  // 基础信息
  name: string;          // 策略名称（通常为股票名称）
  stockCode: string;     // 股票代码（如 "600519"）
  accountType: "default" | "credit";  // 账户类型
  isMarginAccount: boolean;  // 是否融资融券账户

  // 持仓信息
  netPosition: number;   // 净持仓量
  marketValue: string;   // 市值（字符串格式，如 "168000"）

  // 财务指标
  fiveYearAvgDividendYield: string;  // 5年平均股息率

  // 趋势判断
  trendJudgment: TrendType;  // 趋势判断值

  // 网格策略参数
  oscillationGridSize: string;      // 震荡网格大小
  oscillationTradeAmount: string;   // 震荡交易数量
  breakoutGridSize: string;         // 突破网格大小
  breakoutTradeAmount: string;      // 突破交易数量

  // 下跌卖出配置
  decreaseSide: "COLLSELL" | "COLLABUY" | "SELL" | "BUY";
  decreaseStrategies: DecreaseStrategy[];  // 下跌策略列表

  // 上涨买入配置
  increaseStrategies: IncreaseStrategy[];  // 上涨策略列表

  // 其他
  expiryDate: string;    // 过期日期
  notes: string;         // 系统备注
  manualNotes: string;   // 用户手动备注

  // 运行时注入字段（从趋势数据获取）
  currentPrice?: number;     // 当前价格
  price_drop_ratio?: number; // 下跌百分比
  decreasePercentage?: number; // 下跌百分比（旧字段，已废弃）
}
```

## 趋势类型

```typescript
type TrendType =
  | "unset"           // 未设置
  | "trend_unknown"   // 未知趋势
  | "trend_up"        // 上涨趋势
  | "trend_down"      // 下跌趋势
  | "trend_breakdown" // 下跌破位
  | "trend_oscillation" // 震荡趋势
  | "trend_pullback"  // 回踩趋势
  | "high_volatility" // 高波动率
  | "medium_volatility" // 中等波动率
  | "low_volatility"; // 低波动率
```

## 下跌/上涨策略

```typescript
interface DecreaseStrategy {
  deltaPercentage: string;  // 触发百分比
  tradeVolume: string;      // 交易数量
  side: SideType;           // 交易方向
}

interface IncreaseStrategy {
  deltaPercentage: string;  // 触发百分比
  tradeVolume: string;      // 交易数量
  side: SideType;           // 交易方向
}
```

## 交易方向

```typescript
type SideType =
  | "BUY"        // 普通买入
  | "SELL"       // 普通卖出
  | "COLLABUY"   // 担保品买入（信用账户）
  | "COLLASELL"  // 担保品卖出（信用账户）
  | "MARGINBUY"  // 融资买入
  | "MARGINSELL"; // 融券卖出
```

## 完整示例

```json
{
  "_id": "strategy_1750000000000_abc123",
  "id": "strategy_1750000000000_abc123",
  "createdAt": "2025-06-15T10:30:00.000Z",
  "updatedAt": "2025-06-15T14:20:00.000Z",

  "name": "贵州茅台",
  "stockCode": "600519",
  "accountType": "default",
  "isMarginAccount": false,

  "netPosition": 100,
  "marketValue": "168000",
  "fiveYearAvgDividendYield": "2.5",

  "trendJudgment": "trend_up",
  "expiryDate": "",

  "oscillationGridSize": "2.5",
  "oscillationTradeAmount": "100",
  "breakoutGridSize": "5.0",
  "breakoutTradeAmount": "200",

  "decreaseSide": "COLLSELL",
  "decreaseStrategies": [
    {
      "deltaPercentage": "5",
      "tradeVolume": "200",
      "side": "SELL"
    },
    {
      "deltaPercentage": "10",
      "tradeVolume": "300",
      "side": "SELL"
    }
  ],

  "increaseStrategies": [
    {
      "deltaPercentage": "8",
      "tradeVolume": "300",
      "side": "BUY"
    }
  ],

  "notes": "",
  "manualNotes": "长期持有"
}
```

[返回索引](../my-quant-strategy-format.md)
