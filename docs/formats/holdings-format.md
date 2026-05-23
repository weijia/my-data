# 持仓历史记录格式

**文件路径**: `holdings/holdings.default.{timestamp}.json`

文件名中的 `{timestamp}` 为 ISO 8601 格式时间戳。

## 数据结构

```json
{
  "historyEntry": { ... },
  "exportTime": "2025-01-01T12:00:00.000Z",
  "version": "1.0",
  "type": "holdings_history"
}
```

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `historyEntry` | `object` | 持仓历史条目 |
| `exportTime` | `string` | 导出时间，ISO 8601 格式 |
| `version` | `string` | 数据版本号 |
| `type` | `string` | 数据类型标识：`"holdings_history"` 或 `"holdings_export"` |
