# 成交记录格式

本文档描述**成交记录**（实际发生的买卖成交明细）的数据格式。成交可能由条件单执行、网格操作执行产生，也可能来自其他系统。

## 概述

- **数据来源多样**：一条成交记录可能来自条件单触发成交、网格单触发成交，或其他外部系统导入，因此用 `source` 字段标识来源。
- **可回链策略单**：由条件单/网格单产生的成交，用 `orderId` 关联对应的策略单（见 [策略单按月归档格式](./strategy-order-archive-format.md)）。
- **归档粒度**：按**成交月份**分文件，同月多笔成交追加到同一文件（按 `id` 去重合并）。

## 文件路径

```
app_data/stocks/trade_records/{YYYY-MM}.json
```

- `{YYYY-MM}` 为成交月份（本地时区 `Asia/Shanghai`），如 `2026-10`。
- 遵循 my-data 目录约定中的 `app_data/{项目名}/`。

---

## 文件结构

```json
{
  "type": "trade_records",
  "month": "2026-10",
  "trades": [ /* TradeRecord[] */ ],
  "lastUpdated": "2026-10-18T06:10:00.000Z"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `type` | `string` | 是 | 数据类型标识，固定 `"trade_records"` |
| `month` | `string` | 是 | 归档月份，格式 `YYYY-MM` |
| `trades` | `Array<TradeRecord>` | 是 | 当月成交记录，见下节 |
| `lastUpdated` | `string` | 是 | 最后更新时间，ISO 8601 格式 |

---

## TradeRecord 结构

```json
{
  "id": "20261018-600866-0001",
  "source": "condition_order",
  "orderId": "1880938628",
  "orderType": "condition",
  "provider": "founder",
  "stockCode": "600866",
  "stockName": "星湖科技",
  "market": "SH",
  "accountType": "credit",
  "side": "buy",
  "tradePrice": 10.76,
  "tradeVolume": 100,
  "tradeAmount": 1076.0,
  "fee": 5.0,
  "feeDetail": {
    "commission": 5.0,
    "stampTax": 0.0,
    "transferFee": 0.0,
    "other": 0.0
  },
  "tradeTime": "2026-10-18T14:05:00.000+08:00",
  "settledDate": "2026-10-19",
  "externalRef": null,
  "notes": ""
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 成交唯一ID。优先取券商成交编号，无则由应用生成（格式 `时间戳 + 随机5位字符`） |
| `source` | `string` | 是 | 成交来源，见下方 source 值表 |
| `orderId` | `string` | 否 | 关联的策略单ID（对应 OrderRecord.id）。`source` 为 `external` 时可为空 |
| `orderType` | `string` | 否 | 关联策略大类：`"condition"` / `"grid"`。无关联时为空 |
| `provider` | `string` | 否 | 券商标识：`"founder"` / `"pingan"` / 其他；`external` 来源按实际填写 |
| `stockCode` | `string` | 是 | 股票代码 |
| `stockName` | `string` | 是 | 股票名称 |
| `market` | `string` | 否 | 交易市场：`"SH"` / `"SZ"` |
| `accountType` | `string` | 是 | 账户类型：`"default"` / `"credit"` |
| `side` | `string` | 是 | 交易方向：`"buy"` / `"sell"` |
| `tradePrice` | `number` | 是 | 成交价（均价），精确到分 |
| `tradeVolume` | `number` | 是 | 成交数量（股） |
| `tradeAmount` | `number` | 是 | 成交金额（元）= `tradePrice × tradeVolume` |
| `fee` | `number` | 否 | 成交总费用（元） |
| `feeDetail` | `object` | 否 | 费用明细，见下节 |
| `tradeTime` | `string` | 是 | 成交时间，ISO 8601 格式（含时区）。按月归档即取此字段的 `YYYY-MM` |
| `settledDate` | `string` | 否 | 交割日期，格式 `YYYY-MM-DD` |
| `externalRef` | `string` | 否 | 外部系统引用标识，`source` 为 `external` 时使用 |
| `notes` | `string` | 否 | 备注 |

### `source` 值表

| 取值 | 含义 |
|------|------|
| `condition_order` | 条件单触发成交 |
| `grid_order` | 网格单触发成交 |
| `external` | 其他系统导入 |

### `feeDetail` 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `commission` | `number` | 佣金（元） |
| `stampTax` | `number` | 印花税（元） |
| `transferFee` | `number` | 过户费（元） |
| `other` | `number` | 其他费用（元） |

---

## 与策略单的关联

| 场景 | `source` | `orderId` / `orderType` |
|------|----------|-------------------------|
| 条件单触发成交 | `condition_order` | 填对应条件单的 `id`，`orderType = "condition"` |
| 网格单触发成交 | `grid_order` | 填对应网格单的 `id`，`orderType = "grid"` |
| 其他系统产生 | `external` | 通常为空，可用 `externalRef` 记录来源 |

> 通过 `orderId` 可把成交记录回链到 [策略单按月归档格式](./strategy-order-archive-format.md) 中的创建记录，从而还原"创建（起始时间 + 起始价）→ 触发 → 成交"的完整链路。

---

## 数据库入库映射

一条 `TradeRecord` 对应数据库一行，建议表名 `trade_records`。

| 归档字段 | 建议列类型 | 说明 |
|----------|-----------|------|
| `id` | `VARCHAR` / `TEXT` | 主键 |
| `source` | `VARCHAR(16)` | 枚举，建议索引 |
| `orderId` | `VARCHAR` / `TEXT` | 外键，关联 `strategy_orders.id`，建议索引 |
| `orderType` | `VARCHAR(16)` | 枚举：`condition` / `grid` |
| `provider` | `VARCHAR(16)` | |
| `stockCode` | `VARCHAR(16)` | 建议索引 |
| `stockName` | `VARCHAR(64)` | |
| `market` | `VARCHAR(8)` | 枚举：`SH` / `SZ` |
| `accountType` | `VARCHAR(16)` | 枚举：`default` / `credit` |
| `side` | `VARCHAR(8)` | 枚举：`buy` / `sell` |
| `tradePrice` | `DECIMAL(12,3)` | |
| `tradeVolume` | `INT` | |
| `tradeAmount` | `DECIMAL(16,3)` | |
| `fee` | `DECIMAL(12,3)` | |
| `feeDetail` | `JSON` | 或拆为独立列 |
| `tradeTime` | `TIMESTAMP` / `DATETIME` | 建议索引或按月分区 |
| `settledDate` | `DATE` | |
| `externalRef` | `VARCHAR` | |
| `notes` | `VARCHAR` / `TEXT` | |

## 相关文档

- [策略单按月归档格式](./strategy-order-archive-format.md)
- [策略数据格式](./strategy-formats.md)
- [数据格式规范](./data-formats.md)
