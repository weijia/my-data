# 策略单按月归档格式

本文档描述**创建条件单 / 网格单**时，按**创建月份**归档的数据格式。归档数据作为后续写入数据库的数据源。

## 概述

- **触发时机**：用户在应用内创建条件单 / 网格单成功时自动归档。
- **归档粒度**：按**创建月份**分文件，同月多笔创建追加到同一文件（按 `id` 去重合并）。
- **数据来源**：创建接口的返回值（`id`）+ 创建时的实时价格（`startPrice`）+ 创建入参。
- **字段命名**：`id`、`startPrice`、`createTime` 为入库关键字段，其余专有字段按策略类型展开。

## 文件路径

| 类型 | 文件路径 |
|------|----------|
| 条件单 | `app_data/stocks/conditional_strategies/{YYYY-MM}.json` |
| 网格单 | `app_data/stocks/grid_strategies/{YYYY-MM}.json` |

- `{YYYY-MM}` 为创建月份（本地时区 `Asia/Shanghai`），如 `2026-10`。
- 该路径遵循 my-data 目录约定中的 `app_data/{项目名}/`，并与 my-quant 实际 WebDAV 路径一致。

---

## 文件结构

```json
{
  "type": "conditional_order_archive",
  "month": "2026-10",
  "orders": [ /* OrderRecord[] */ ],
  "lastUpdated": "2026-10-18T02:30:00.000Z"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `type` | `string` | 是 | 归档类型：`"conditional_order_archive"`（条件单）或 `"grid_order_archive"`（网格单） |
| `month` | `string` | 是 | 归档月份，格式 `YYYY-MM` |
| `orders` | `Array<OrderRecord>` | 是 | 当月创建的记录，见下节 |
| `lastUpdated` | `string` | 是 | 最后更新时间，ISO 8601 格式 |

---

## OrderRecord（公共字段）

条件单与网格单共用以下字段。

```json
{
  "id": "1880938628",
  "type": "condition",
  "provider": "founder",
  "stockCode": "600866",
  "stockName": "星湖科技",
  "accountType": "credit",
  "startPrice": 10.5,
  "createTime": "2026-10-18T10:30:00.000+08:00",
  "status": "active"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 策略唯一ID，取自创建接口返回值。条件单：`result.info.id`（兼容 `taskId`）；网格单：`taskId`。**后续按此 ID 查询策略状态** |
| `type` | `string` | 是 | 策略大类：`"condition"`（条件单）或 `"grid"`（网格单） |
| `provider` | `string` | 是 | 券商标识：`"founder"`（方正）/ `"pingan"`（平安） |
| `stockCode` | `string` | 是 | 股票代码 |
| `stockName` | `string` | 是 | 股票名称 |
| `accountType` | `string` | 是 | 账户类型：`"default"` / `"credit"` |
| `startPrice` | `number` | 是 | **起始价格**：创建该单时的实时价格，作为触发判定的起始基准价（网格单即基准价 `basicPrice`） |
| `createTime` | `string` | 是 | **起始时间**：创建该单的时间，ISO 8601 格式（含时区）。按月归档即取此字段的 `YYYY-MM` |
| `status` | `string` | 否 | 策略状态，创建时通常为 `"active"`；后续按 `id` 查询状态回填，取值见 [value-definitions](./value-definitions.md) |

---

## 条件单专有字段（`type` = `condition`）

```json
{
  "strategyType": "bounce_buy",
  "side": "buy",
  "bounceType": "percent",
  "deltaPercentage": 2.5,
  "deltaAmount": null,
  "tradeVolume": 100,
  "targetPrice": 10.76,
  "expiredTime": "2026-11-30"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `strategyType` | `string` | 否 | 交易类型，如 `"bounce_buy"`（反弹买入）、`"fall_sell"`（回落卖出） |
| `side` | `string` | 否 | 交易方向：`"buy"` / `"sell"` |
| `bounceType` | `string` | 否 | 涨跌方式：`"price"`（按金额）或 `"percent"`（按百分比） |
| `deltaPercentage` | `number` | 否 | 涨跌百分比（单位 `%`，如 `2.5`） |
| `deltaAmount` | `number` | 否 | 涨跌金额（元），`bounceType` 为 `"price"` 时使用 |
| `tradeVolume` | `number` | 否 | 委托数量 |
| `targetPrice` | `number` | 否 | 触发价（创建时若已指定） |
| `expiredTime` | `string` | 否 | 过期时间 |

---

## 网格单专有字段（`type` = `grid`）

```json
{
  "taskName": "星湖科技10格",
  "basicPrice": 10.5,
  "priceLow": 9.0,
  "priceHigh": 12.0,
  "offsetPrice": 0.3,
  "quantityBuy": 100,
  "maxFund": 50000,
  "expiredTime": "2026-12-31"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `taskName` | `string` | 否 | 策略名称 |
| `basicPrice` | `number` | 否 | 基准价，等于 `startPrice` |
| `priceLow` | `number` | 否 | 价格下限 |
| `priceHigh` | `number` | 否 | 价格上限 |
| `offsetPrice` | `number` | 否 | 网格间距 |
| `quantityBuy` | `number` | 否 | 每格买入数量 |
| `maxFund` | `number` | 否 | 最大占用资金 |
| `expiredTime` | `string` | 否 | 过期时间 |

---

## 起始时间与起始价格（触发判定）

条件单的触发由**起始时间 + 起始价格**共同决定：`createTime` 为起始时刻，`startPrice` 为创建时的实时价格，作为触发判定的起始基准价；再结合 `deltaPercentage` / `deltaAmount` 描述触发条件。

> 具体触发价由平台按策略类型计算，且会随行情变化调整，故本格式只记录触发判定的**起始基准**（`createTime` + `startPrice`），不固定触发价公式。

---

## 数据库入库映射

一条 `OrderRecord` 对应数据库一行，建议表名 `strategy_orders`。

| 归档字段 | 建议列类型 | 说明 |
|----------|-----------|------|
| `id` | `VARCHAR` / `TEXT` | 主键（与 `type` 组成唯一键），后续按此查询并回填状态 |
| `type` | `VARCHAR(16)` | 枚举：`condition` / `grid` |
| `provider` | `VARCHAR(16)` | 枚举：`founder` / `pingan` |
| `stockCode` | `VARCHAR(16)` | 建议索引 |
| `stockName` | `VARCHAR(64)` | |
| `accountType` | `VARCHAR(16)` | 枚举：`default` / `credit` |
| `startPrice` | `DECIMAL(12,3)` | 起始价格 |
| `createTime` | `TIMESTAMP` / `DATETIME` | 起始时间，建议索引或按月分区 |
| `status` | `VARCHAR(16)` | 枚举，可后续 update 回填 |
| 专有字段 | 见上 | 展开为独立列，或整体存入 `JSON` 列 |

## 相关文档

- [策略数据格式](./strategy-formats.md)
- [数据格式规范](./data-formats.md)
