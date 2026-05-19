# My-Quant 策略数据格式规范

本文档定义 my-quant 应用中策略数据的存储格式。

## 概述

My-quant 使用 PouchDB（IndexedDB）在浏览器本地存储策略数据。策略数据可通过以下方式同步到 WebDAV：
- **单文件模式**（默认）：所有策略合并到 `all_strategies.json`
- **多文件模式**：各类型策略分别保存到独立文件

---

## 1. 策略基础结构

### 1.1 核心字段

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

// 趋势类型
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

// 下跌策略
interface DecreaseStrategy {
  deltaPercentage: string;  // 触发百分比
  tradeVolume: string;      // 交易数量
  side: SideType;           // 交易方向
}

// 上涨策略
interface IncreaseStrategy {
  deltaPercentage: string;  // 触发百分比
  tradeVolume: string;      // 交易数量
  side: SideType;           // 交易方向
}

// 交易方向
type SideType =
  | "BUY"        // 普通买入
  | "SELL"       // 普通卖出
  | "COLLABUY"   // 担保品买入（信用账户）
  | "COLLASELL"  // 担保品卖出（信用账户）
  | "MARGINBUY"  // 融资买入
  | "MARGINSELL"; // 融券卖出
```

### 1.2 完整示例

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

---

## 2. 策略模板格式（SettingsView 策略管理器）

策略管理器中定义的策略模板，用于生成 MQTT 条件单消息。

### 2.1 模板结构

```typescript
interface StrategyTemplate {
  name: string;              // 模板名称（如"默认买入"）
  action: "buy" | "sell";    // 操作类型
  percentage: string;        // 触发百分比（如 "0.1" 或 "0.5"）
  orderMode: "amount" | "volume";  // 下单方式
  amount?: string;           // 金额（orderMode=amount 时）
  volume?: string;           // 数量（orderMode=volume 时）
  provider: string;          // 券商（如 "pingan"）
  accountType: "default" | "credit";  // 账户类型
  useMargin: boolean;        // 是否使用融资
}
```

### 2.2 存储位置

```
localStorage key: "orderStrategyTemplates"
```

### 2.3 示例

```json
[
  {
    "name": "0.1%小额买入",
    "action": "buy",
    "percentage": "0.1",
    "orderMode": "amount",
    "amount": "20000",
    "provider": "pingan",
    "accountType": "default",
    "useMargin": false
  },
  {
    "name": "0.5%大额卖出",
    "action": "sell",
    "percentage": "0.5",
    "orderMode": "volume",
    "volume": "500",
    "provider": "pingan",
    "accountType": "credit",
    "useMargin": true
  }
]
```

---

## 3. MQTT 条件单消息格式

策略执行时生成的 MQTT 消息格式。

### 3.1 消息结构

```typescript
interface MQTTMessage {
  id: string;        // 客户端ID（如 "myquant_abc123"）
  msgId: string;     // 消息唯一ID（如 "1750000000000_abcd"）
  user: string;      // 用户名（固定 "myquant"）
  msg: string;       // 加密的消息内容（JSON字符串）
  time: number;      // 时间戳（毫秒）
}

// msg 字段解密后的结构
interface OrderCommand {
  action: "buy" | "sell" | "add" | "create";
  data: {
    stockCode: string;       // 股票代码
    stockName: string;       // 股票名称
    tradeVolume?: number;    // 交易数量
    tradeAmount?: number;    // 交易金额（可选）
    percentage: number;      // 触发百分比
    provider: string;        // 券商
    accountType: string;     // 账户类型
    side?: string;           // 交易方向（不使用融资时）
    endDate?: string;        // 条件单结束日期（格式 "YYYY-MM-DD"），不传则使用默认过期逻辑
  };
}
```

### 3.2 完整示例

```json
{
  "id": "myquant_abc123",
  "msgId": "1750000000000_abcd",
  "user": "myquant",
  "msg": "{\"action\":\"buy\",\"data\":{\"stockCode\":\"600519\",\"stockName\":\"贵州茅台\",\"tradeVolume\":200,\"percentage\":0.5,\"provider\":\"pingan\",\"accountType\":\"default\",\"side\":\"COLLABUY\",\"endDate\":\"2026-05-30\"}}",
  "time": 1750000000000
}
```

### 3.3 Side 字段规则

| 操作 | useMargin=true | useMargin=false |
|------|----------------|-----------------|
| 买入 | 不发送 side | `COLLABUY` |
| 卖出 | 不发送 side | `COLLASELL` |

---

## 4. 数据同步格式

### 4.1 单文件模式（all_strategies.json）

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

### 4.2 字段映射

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

---

## 5. 趋势判断数据注入

策略加载时，从 WebDAV 趋势数据注入以下字段：

| 注入字段 | 来源字段 | 说明 |
|----------|----------|------|
| `currentPrice` | `volatilityMetrics.current_price` | 当前价格 |
| `price_drop_ratio` | `volatilityMetrics.price_drop_ratio` | 下跌百分比 |
| `trendJudgment` | `autoTrendJudgment` / `trendJudgment` | 趋势判断 |

---

## 6. 相关文档

- [WebDAV 数据格式](./webdav-data-formats.md) - WebDAV 同步的数据格式
- [数据格式规范](./data-formats.md) - my-data 仓库整体数据规范
