# MQTT 条件单消息格式

策略执行时生成的 MQTT 消息格式。

## 消息结构

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

## 完整示例

```json
{
  "id": "myquant_abc123",
  "msgId": "1750000000000_abcd",
  "user": "myquant",
  "msg": "{\"action\":\"buy\",\"data\":{\"stockCode\":\"600519\",\"stockName\":\"贵州茅台\",\"tradeVolume\":200,\"percentage\":0.5,\"provider\":\"pingan\",\"accountType\":\"default\",\"side\":\"COLLABUY\",\"endDate\":\"2026-05-30\"}}",
  "time": 1750000000000
}
```

## Side 字段规则

| 操作 | useMargin=true | useMargin=false |
|------|----------------|-----------------|
| 买入 | 不发送 side | `COLLABUY` |
| 卖出 | 不发送 side | `COLLASELL` |

[返回索引](../my-quant-strategy-format.md)
