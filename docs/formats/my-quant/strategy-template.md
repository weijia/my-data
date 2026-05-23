# 策略模板格式

策略管理器中定义的策略模板，用于生成 MQTT 条件单消息。

## 模板结构

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

## 存储位置

```
localStorage key: "orderStrategyTemplates"
```

## 示例

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

[返回索引](../my-quant-strategy-format.md)
