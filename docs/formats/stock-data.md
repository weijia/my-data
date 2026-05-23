# 股票基础信息

**文件路径**: `stocks/stock_data.json`

## StockItem 结构

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

## 关注列表

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
