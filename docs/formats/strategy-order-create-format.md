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
| `startPrice` | `number` | 是* | **起始价格**：创建该单时的实时价格，作为触发判定的起始基准价（网格单即基准价 `basicPrice`）。`origin` 为 `"server"` 的条件单记录无此值，为 `null`（服务端任务里没有"创建时实时价"，无法还原） |
| `createTime` | `string` | 是* | **起始时间**：创建该单的时间，ISO 8601 格式（含时区）。按月分文件即取此字段的 `YYYY-MM`。服务端任务缺 `time` 时补建记录为 `null` |
| `status` | `string` | 否 | 策略状态，创建时通常为 `"active"`；后续按 `id` 查询状态回填，取值见 [状态查询与状态取值](#状态查询与状态取值) |
| `origin` | `string` | 是 | **记录来源**：`"plugin"` = 本插件创建时写入；`"server"` = 「自动同步全部」时按服务端任务补建（非本插件创建，或创建时保存失败） |
| `rawTask` | `object` | 否 | 服务端原始任务 JSON 快照（来自 `getTaskList` 的 `content[]` 项或 `task/detail` 的 `info.taskInfo` 整条任务对象），作为后续扩展字段与跨系统核对的参考。`origin="plugin"` 创建时该字段为 `null`，首次 `refreshStatus` 回填时写入；**不参与业务字段映射** |

> **成交/委托明细（entrusts）**：记录的「实际成交」由 `todayEntrust` 接口按 `taskId` 拉取（方正 `getTaskList` 的任务对象里**不含成交明细**，之前误用的 `buyTimes/saleTimes/profit` 等任务级字段并非真实成交），写入为 `entrusts` 数组，见下方「成交/委托记录（entrusts）」。创建时（含 `origin="plugin"`）该字段为 `null`，仅在「自动同步全部」每轮 `refreshStatus` 回填时写入。
>
> **原始成交/委托记录（rawEntrusts）**：与 `entrusts` **并存**的「原样」字段。两者数据同源（`todayEntrust` 的 `info.content[]`），但 `entrusts` 只保留归一化后的 13 个挑选字段，`rawEntrusts` 则把接口返回的每一条记录**原样**（不挑字段、不做字段重命名、时间不转 ISO）整体保存，便于以后核对字段、应对接口字段变化。同样按委托 `id` 跨同步累积合并；创建时为 `null`，仅在 `refreshStatus` 回填时写入。

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

## 成交/委托记录（entrusts，来自 todayEntrust）

记录的「实际成交 / 委托」明细由方正 `todayEntrust` 接口按 `taskId` 返回（见 [状态查询与状态取值](#状态查询与状态取值) 的接口表），**不在** `getTaskList` 的任务对象里（任务对象上虽有 `buyTimes/saleTimes/profit` 等字段，但那不是成交明细，不得作为实际成交使用）。`refreshStatus` 每轮按 `id` 拉取后，以 `entrusts` 数组写入 / 累积合并进对应记录；普通创建保存（`origin="plugin"`）时该字段为 `null`。

> ⚠️ `todayEntrust` 只返回**当日**委托。不同步之间靠委托 `id` 去重合并，因此只要每天都跑一次「自动同步全部」，历史成交会被逐步累积保留；如需一次性补齐很久以前的历史成交，需另接历史委托接口（当前未实现）。

单条委托记录字段（来自 `todayEntrust` 的 `info.content[]` 项）：

```json
{
  "id": "4623392568",
  "taskId": "1881433463",
  "time": "2026-10-19T11:30:12.000+08:00",
  "stockCode": "002327",
  "stockName": "富安娜",
  "price": 6.04,
  "priceType": "限价",
  "amount": 3200.0,
  "direction": 2,
  "side": "sell",
  "status": 55,
  "statusName": "已成",
  "orderId": "1881433463-1931020-1792373412.5"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `id` | `string` | 否 | 委托/成交记录 ID（按此去重合并） |
| `taskId` | `string` | 否 | 所属策略任务 ID |
| `time` | `string` | 否 | 委托时间，epoch 毫秒转 ISO 8601（含时区）；无则为 `null` |
| `stockCode` | `string` | 否 | 股票代码 |
| `stockName` | `string` | 否 | 股票名称 |
| `price` | `number` | 否 | 委托/成交价格 |
| `priceType` | `string` | 否 | 价格类型，如 `"限价"` |
| `amount` | `number` | 否 | 委托数量 |
| `direction` | `number` | 否 | 委托方向原始值（服务端 `entrust_direction`）：`1`=买入 / `2`=卖出 |
| `side` | `string` | 否 | 方向语义：`"buy"` / `"sell"`（由 `direction` 映射，其它值 `null`） |
| `status` | `number` | 否 | 委托状态原始值 |
| `statusName` | `string` | 否 | 委托状态名，如 `"已成"` / `"已撤"` |
| `orderId` | `string` | 否 | 委托订单号 |

> 原始任务对象（来自 `getTaskList` / `task/detail`）仍作为 `rawTask` 快照保留在公共字段里，供扩展与核对，但它**不含**上述成交明细。

### 原始成交/委托记录（rawEntrusts，todayEntrust 原样）

`rawEntrusts` 与 `entrusts` 同源（都来自 `todayEntrust` 的 `info.content[]`），但它是**原始响应原样保留**的一份，未经 `normalizeEntrust` 的字段挑选与重命名：

- `entrusts`：归一化数组，只含 `id`/`taskId`/`time`/`stockCode`/`stockName`/`price`/`priceType`/`amount`/`direction`/`side`/`status`/`statusName`/`orderId` 共 13 个字段，`time` 已转本地 ISO 8601。
- `rawEntrusts`：接口返回的原始对象数组（如 `stock_code`、`entrust_direction`、`time` 为 epoch 毫秒、`priceType`、以及任何接口新增/未知的字段都原样保留），仅按 `id` 去重合并，不做其它处理。

```json
{
  "entrusts": [ { "id": "4623392568", "side": "sell", "time": "2026-10-19T11:30:12.000+08:00", "...": "..." } ],
  "rawEntrusts": [
    {
      "id": 4623392568,
      "taskId": 1881433463,
      "stock_code": "002327",
      "stock_name": "富安娜",
      "price": 6.04,
      "priceType": "限价",
      "amount": 3200.0,
      "entrust_direction": 2,
      "time": 1760849412000,
      "status": 55,
      "statusName": "已成",
      "orderId": "1881433463-1931020-1792373412.5"
    }
  ]
}
```

> 两者按**同一套 `id`** 合并（都来自 `info.content[].id`），因此 `rawEntrusts[i]` 与 `entrusts[i]` 通常一一对应；若接口字段映射或类型有出入，以 `rawEntrusts` 为准做核对。`rawEntrusts` 不参与任何业务字段映射，纯属「留底 / 扩展」用途。

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
| 策略列表（全量） | `GET /qt/strategy/list` | `content[]` 含 `id`、`status`、`sname` |
| 今日委托/成交 | `GET /qt/user/todayEntrust?pageNum=1&pageSize=200&taskId={id}` | 按 `taskId` 返回**当日**委托/成交明细（`info.content[]`）；**这是「实际成交」的唯一真实来源**，`getTaskList` 任务对象里不含成交 |

**网格单怎么认**：服务端任务用 `sname` 区分，网格实测有两种取值 `GridTrading`（`data/grid-example.md`）与 `AdvancedGridTrading`（`data/grid_strategy_for_credit_account.json`），两者都算网格；其余 `sname`（如 `strategy_bottom_buy` / `strategy_top_sell`）为条件单。注意**网格单没有条件单那样的列表接口**（`getTaskList` 只返回 `sName=condition`），只能按已知 `id` 调 `task/detail` 逐条取。

- **服务器原始值**（`/qt/strategy/list`、`/qt/task/getTaskList`、`/qt/task/detail` 返回同一套）：`RUNNING`（运行中）/ `STOPPED`（已停止）/ `STOPPED_MANUAL`（手动停止）—— 样本已见；另有 `COMPLETED`（映射表已定义，样本未直接见到）。
- **入库口径**：`status` **不原样存服务器值**，按 `vue-dialog-userscript` 的 `docs/reference/value-definitions.md` §3.2 映射为内部枚举：`RUNNING`→`active`、`STOPPED`→`stopped`、`STOPPED_MANUAL`→`stopped_manual`、`COMPLETED`→`completed`，未知值兜底 `unknown`。创建时写入的 `"active"` 即 `RUNNING`，与回填口径一致。
- 内部枚举另有 `triggered`（已触发）/`canceled`（已取消）/`expired`（已过期）/`failed`（失败），但**无对应的服务器原始值样本**，回填时遇到未知值一律走 `unknown` 兜底，不猜。
- 停止接口：`POST /qt/task/stop`。

> **实现状态**：方正侧已实现（`vue-dialog-userscript` 在「自动同步全部」每轮解析完成后，用本轮已获取到的服务端任务回填/补建 `orders/` 记录：`id` 命中则更新 `status`、并按下述 `todayEntrust` 回填 `entrusts`（成交/委托明细，归一化）与 `rawEntrusts`（接口原样并存）、以及原始任务快照 `rawTask`；未命中则补建一条并标记 `origin="server"`（同样带 `entrusts`、`rawEntrusts` 与 `rawTask`）；获取不到的不动。详见其 `docs/api/OrderCreateSaver-API.md`）。平安侧**未实现**（等 id 映射确认后再说）。

### 平安（`provider` = `pingan`）

| 查询接口 | 方法/路径 | 说明 |
|----------|-----------|------|
| 条件单/网格列表 | `POST https://m.stock.pingan.com/restapi/past/queryMyYmdForPage` | 一个接口同时覆盖条件单与网格 |

- 入参：`search_status`（`"0,1"` = 全部）、`strategy_ids`（`8,34,23,35,9,7,12,21,15,39,22,38`，其中 `12`/`34`/`35` 为网格）、`date_type`、`stock_code`、`page_size`、`order_column` 等。
- 返回 `data[].ymd_base`：含 `ymd_id`、`uuid`、`strategy_id`、`ymd_status`、`create_datetime`、`expiry_datetime`、`error_no`。
- 状态取值（**服务器原始值**）：`ymd_status` = `"1"`（running，运行中）/ `"0"`（paused，已暂停）；`error_no` 为错误码（`0` 表示成功）。
- **入库口径**：`"1"`→`active`、`"0"`→`stopped`（`"0"` 原文语义是「暂停」，与内部 `stopped` 对齐；若后续需要区分「可恢复的暂停」，需先在 `value-definitions.md` 的内部枚举里新增 `paused`，不要擅自映射成其它值）。

> **`id` 映射说明**：创建接口返回的 ID 与查询接口的标识字段未必同名。
> - 方正：创建返回 `info.id`，对应 `/qt/strategy/list` 的 `content[].id` 与 `/qt/task/detail` 的 `id`。**已由仓库样本确认**：`data/create_condition_post.md` 的创建响应 `info.id`、`data/getTaskList.json` 的 `content[].id`、`/qt/task/stop` 的取参 id 与 `/qt/task/detail` 的 `info.taskInfo.id` 属同一 id 命名空间。
> - 平安：**不确定（待确认）** —— 创建返回 `data`（订单号），查询接口主键为 `ymd_base.ymd_id` / `ymd_base.uuid`，两者是否一致尚未核对。
>
> 入库时以 `id` 为准，查询时按各接口对应字段匹配。

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
| 成交/委托明细 | 见「成交/委托记录（entrusts）」节 | `entrusts` 建议存为 `JSON`/`TEXT` 列（数组，含每条委托的明细）；如需分析可把关键字段（`side`/`amount`/`price`/`statusName`/`time`）展开为独立列 |
| 原始成交/委托明细 | 见「原始成交/委托记录（rawEntrusts）」节 | `rawEntrusts` 建议存为 `JSON`/`TEXT` 列（接口原样数组），与 `entrusts` 并存留底；不参与索引与字段映射 |
| `rawTask` | `JSON` / `TEXT` | 服务端原始任务快照，整体存入，不参与索引 |

## 相关文档

- [策略数据格式](./strategy-formats.md)
- [数据格式规范](./data-formats.md)
