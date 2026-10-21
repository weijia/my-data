# Bookkeeping 电商订单（OrderRecord）

电商订单是统一记录模型中 `type: 'order'` 的一条，来源于**京东 + 淘宝统一订单 schema** 的导入
（`all_orders_unified.json`，每条一行 JSON 对象）。

> 旧版订单被并入 `transactions` 数组、只能靠 `platform`/`orderId` 间接识别；新版独立为 `type: 'order'`，
> 电商字段集中在 `OrderRecord` 上。样例文件见 onenav 仓库 `docs/bookkeeping-order-sample.json`。

## 1. 导入 schema（输入行结构）

```json
{
  "platform": "taobao",
  "account": "weijiagg",
  "order_id": "2211993344556677",
  "order_time": "2026-09-02 00:00:00",
  "year": "2026",
  "shop": "良品铺子官方旗舰店",
  "status": "交易成功",
  "items": [
    { "name": "每日坚果礼盒 750g", "spec": "混合味", "price": 69.9, "qty": 2, "amount": 139.8 }
  ],
  "total_amount": 159.7,
  "payment": "信用卡支付",
  "is_master_split": false,
  "receiver_name": null,
  "receiver_address": null,
  "remark": null,
  "freight": 0,
  "logistics": "已签收",
  "refund_note": null,
  "is_phone_order": true
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `platform` | `'jd' \| 'taobao'` | 是 | 平台（京东 / 淘宝）；其他值导入时置 `undefined`，无 `order_id` 则整行跳过 |
| `account` | `string` | 否 | 下单账号 |
| `order_id` | `string` | 是 | 订单号（与 `platform`+`account` 构成复合主键） |
| `order_time` | `string` | 否 | 下单时间 `YYYY-MM-DD HH:mm:ss`；`date` 取前 10 位 |
| `year` | `string` | 否 | 年份 `YYYY`；`order_time` 缺失/非法时回退为 `${year}-01-01` |
| `shop` | `string` | 否 | 店铺名 |
| `status` | `string` | 否 | 订单状态（如 付款成功 / 交易成功） |
| `items` | `OrderItem[]` | 否 | 商品清单 |
| `total_amount` | `number` | 否 | 订单总额 → 记录的 `amount` |
| `payment` | `string` | 否 | 支付方式 |
| `is_master_split` | `boolean` | 否 | **京东拆单主单标记**，`true` 时整行剔除（见 §3） |
| `receiver_name` / `receiver_address` / `remark` / `freight` / `logistics` / `refund_note` / `is_phone_order` | — | 否 | 收货人、地址、备注、运费、物流、退款说明、手机订单等附加字段，**不落入记录**（仅 schema 保留） |

**OrderItem 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 商品名 |
| `spec` | `string \| null` | 规格 |
| `price` | `number \| null` | 单价（京东多为 `null`，淘宝有单价） |
| `qty` | `number \| null` | 数量 |
| `amount` | `number \| null` | 小计金额 |

## 2. 归一化规则（normalizeOrderRow）

导入时一条订单行被归一化为：

```json
{
  "type": "order",
  "id": "1746000000001-d4e5f6",
  "platform": "taobao",
  "account": "weijiagg",
  "orderId": "2211993344556677",
  "status": "交易成功",
  "shop": "良品铺子官方旗舰店",
  "payment": "信用卡支付",
  "amount": 159.7,
  "category": "购物",
  "note": "淘宝 · @weijiagg · 交易成功",
  "date": "2026-09-02",
  "createdAt": 1746000000001,
  "product": "每日坚果礼盒 750g",
  "items": [ { "name": "每日坚果礼盒 750g", "spec": "混合味", "price": 69.9, "qty": 2, "amount": 139.8 } ]
}
```

| 记录字段 | 来源 | 说明 |
|------|------|------|
| `type` | 固定 | 恒为 `order` |
| `platform` / `account` / `orderId` / `status` / `shop` / `payment` / `items` | 同名字段 | `orderId ← order_id` |
| `amount` | `total_amount` | 订单总额 |
| `category` | 固定 | 恒为 `购物` |
| `note` | 拼接 | `平台 · @账号 · 状态`（账号/状态缺失时省略对应段） |
| `date` | `order_time` | 取前 10 位；非法时回退 `${year}-01-01`，再无则当天 |
| `product` | 派生 | 第一个商品名；无商品时取 `shop` |
| `createdAt` | `createdAt` | 缺省为写入时刻 |

## 3. 拆单主单过滤

京东存在拆单：一个父单拆成多个子单时，父单以 `is_master_split: true` 标记。
按 schema §5.1，统计真实交易时**自动剔除**这类主单（`is_master_split === true` 的行直接跳过），
既不计入有效订单数、也不计入跳过数。导出时订单记录的 `is_master_split` 恒写 `false`。

## 4. 价格提取（订单 → 报价）

订单商品中**有单价**者（`price > 0.01`）会额外生成 `type: 'price'`（`kind: 'item'`）记录，
使导入的订单同时沉淀到价格页：

- 过滤 `price ≤ 0.01` 的占位（淘宝大量订单单价为 0.01 的红包占位）；
- 京东单价多为 `null`，因此主要来自淘宝订单；
- 提取的价格记录带 `source: '京东订单' | '淘宝订单'`；
- 按商品名去重、保留最新报价（`mergePrices`，以 `date`/`createdAt` 新者胜）。

## 5. 导入识别

`detectRowKind()` 判定顺序（详见 [import-export.md](./import-export.md)）：

1. 含 `order_id`，或含 `platform` + `total_amount` → **订单（order）**
2. 含 `kind` → 价格（price）
3. 含 `amount` 且含 `type` 或 `category` → 流水（tx）
4. 含 `unitPrice` 且不含 `type` → 价格（price）

## 6. 导出格式

导出时通过 `toExportShape()` 生成（在流水字段基础上追加订单字段，保证回灌可再识别）：

```json
{
  "date": "2026-09-02",
  "type": "expense",
  "category": "购物",
  "amount": 159.7,
  "note": "淘宝 · @weijiagg · 交易成功",
  "product": "每日坚果礼盒 750g",
  "quantity": null,
  "unit": null,
  "unitPrice": null,
  "platform": "taobao",
  "account": "weijiagg",
  "order_id": "2211993344556677",
  "order_time": "2026-09-02 00:00:00",
  "year": "2026",
  "shop": "良品铺子官方旗舰店",
  "status": "交易成功",
  "payment": "信用卡支付",
  "items": [ { "name": "每日坚果礼盒 750g", "spec": "混合味", "price": 69.9, "qty": 2, "amount": 139.8 } ],
  "is_master_split": false,
  "total_amount": 159.7
}
```

> `total_amount` 与 `amount` 同值（回灌识别用）；普通流水记录**不写**这些订单字段，
> 避免被误判为订单（修复早期"普通记录被误判为订单"的回环 bug）。

[返回索引](../bookkeeping-format.md)
