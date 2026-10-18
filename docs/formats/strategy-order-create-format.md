# 策略单创建时保存格式

本文档描述**创建条件单 / 网格单**时保存的数据格式（按**创建月份**分文件）。保存数据作为后续写入数据库的数据源。

## 概述

- **保存时机**：用户在应用内创建条件单 / 网格单时，随创建流程一并保存（创建接口返回后写入，以便带上服务端 `id`）。
- **单一系列**：条件单与网格单写入**同一系列**文件（按月），用记录内的 `type` 区分。
- **保存粒度**：按**创建月份**分文件，同月多笔创建追加到同一文件（按 `id` 去重合并）。
- **数据来源**：创建接口的返回值（`id`）+ 创建时的实时价格（`startPrice`）+ 创建入参。
- **字段命名**：`id`、`startPrice`、`createTime` 为入库关键字段，其余专有字段按策略类型展开。

## 文件路径

**文件路径**: `app_data/stocks/orders/{YYYY-MM}.json`

- 条件单与网格单**合并为同一系列**，每月一个文件，用记录内的 `type`（`condition` / `grid`）区分。
- `{YYYY-MM}` 为创建月份（本地时区 `Asia/Shanghai`），如 `2026-10`。
- 该路径遵循 my-data 目录约定中的 `app_data/{项目名}/`，并与 my-quant 实际 WebDAV 路径一致。

---

## 文件结构

```json
{
  "type": "order_create_records",
  "month": "2026-10",
  "orders": [ /* OrderRecord[] */ ],
  "lastUpdated": "2026-10-18T02:30:00.000Z"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `type` | `string` | 是 | 数据类型标识，固定 `"order_create_records"` |
| `month` | `string` | 是 | 保存月份，格式 `YYYY-MM` |
| `orders` | `Array<OrderRecord>` | 是 | 当月创建的记录（条件单与网格单混存，用 `type` 区分），见下节 |
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
| `createTime` | `string` | 是 | **起始时间**：创建该单的时间，ISO 8601 格式（含时区）。按月分文件即取此字段的 `YYYY-MM` |
| `status` | `string` | 否 | 策略状态，创建时通常为 `"active"`；后续按 `id` 查询状态回填，取值见 [状态查询与状态取值](#状态查询与状态取值) |

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

## 状态查询与状态取值

记录的 `id` 用于后续向券商服务器接口查询策略状态，回填 `status`。

### 方正（`provider` = `founder`，基础 URL `https://quant.foundersc.com`）

| 查询接口 | 方法/路径 | 说明 |
|----------|-----------|------|
| 条件单列表 | `GET /qt/task/getTaskList?sName=condition&statusList=0&statusList=1&statusList=2&page=1&pageSize=2000` | 返回 `info.content[]`，含 `id`、`status` |
| 任务详情 | `GET /qt/task/detail?id={id}&isMock=false` | 按 `id` 查单条详情（网格策略详情走此接口） |
| 策略列表（全量） | `GET /qt/strategy/list` | `content[]` 含 `id`、`status`、`sname`（`GridTrading` = 网格，其余为条件单） |

- 状态取值（`/qt/strategy/list`、`/qt/task/getTaskList`）：`RUNNING`（运行中）/ `STOPPED`（已停止）/ `STOPPED_MANUAL`（手动停止）。
- 另有条件单状态枚举：`active`（激活中）/ `triggered`（已触发）/ `completed`（已完成）/ `canceled`（已取消）/ `expired`（已过期）/ `failed`（创建失败）。
- 停止接口：`POST /qt/task/stop`。

### 平安（`provider` = `pingan`）

| 查询接口 | 方法/路径 | 说明 |
|----------|-----------|------|
| 条件单/网格列表 | `POST https://m.stock.pingan.com/restapi/past/queryMyYmdForPage` | 一个接口同时覆盖条件单与网格 |

- 入参：`search_status`（`"0,1"` = 全部）、`strategy_ids`（`8,34,23,35,9,7,12,21,15,39,22,38`，其中 `12`/`34`/`35` 为网格）、`date_type`、`stock_code`、`page_size`、`order_column` 等。
- 返回 `data[].ymd_base`：含 `ymd_id`、`uuid`、`strategy_id`、`ymd_status`、`create_datetime`、`expiry_datetime`、`error_no`。
- 状态取值：`ymd_status` = `"1"`（running，运行中）/ `"0"`（paused，已暂停）；`error_no` 为错误码（`0` 表示成功）。

> **`id` 映射说明**：创建接口返回的 ID 与查询接口的标识字段未必同名。方正创建返回 `info.id`，对应 `/qt/strategy/list` 的 `content[].id` 与 `/qt/task/detail` 的 `id`；平安创建返回 `data`（订单号），而查询接口主键为 `ymd_base.ymd_id` / `ymd_base.uuid`。入库时以 `id` 为准，查询时按各接口对应字段匹配。

---

## 数据库入库映射

一条 `OrderRecord` 对应数据库一行，建议表名 `strategy_orders`。

| 字段 | 建议列类型 | 说明 |
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
