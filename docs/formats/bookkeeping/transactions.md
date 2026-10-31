# Bookkeeping 交易记录（TxRecord）

交易记录是统一记录模型中 `type: 'tx'` 的一条，表示一条收支流水（收入/支出）。

> 旧版该数据位于 `BookkeepingData.transactions` 数组、用 `type: 'income' | 'expense'` 表达方向；
> 新版顶层 `type` 固定为 `'tx'`，收支方向改由 **`direction`** 字段表达。

## 数据结构

```json
{
  "type": "tx",
  "id": "1746000000000-a1b2c3",
  "direction": "expense",
  "amount": 35,
  "category": "餐饮",
  "note": "午餐",
  "date": "2026-10-20",
  "createdAt": 1746000000000,
  "product": "麻辣烫",
  "quantity": 1,
  "unit": "份",
  "unitPrice": 35
}
```

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | `'tx'` | 是 | 记录类型，固定为 `tx` |
| `id` | `string` | 是 | 唯一 ID（`randId()` 生成：时间戳-随机串） |
| `direction` | `'income' \| 'expense'` | 是 | 收支方向（导入时由 `type`/兼容值映射而来） |
| `amount` | `number` | 是 | 金额；若缺失但提供了 `quantity` 与 `unitPrice`，会自动按 `quantity × unitPrice` 计算 |
| `category` | `string` | 否 | 消费分类，缺省为 `"其他"` |
| `note` | `string` | 否 | 备注，缺省为空字符串 |
| `date` | `string` | 否 | 日期，格式 `YYYY-MM-DD`；缺省/非法时回退为当天 |
| `createdAt` | `number` | 否 | 创建时间，毫秒级 Unix 时间戳；缺省为写入时刻 |
| `product` | `string` | 否 | 关联商品名，用于「上次报价带入」 |
| `quantity` | `number` | 否 | 数量 |
| `unit` | `string` | 否 | 单位（如 粒 / 斤 / 个 / 件） |
| `unitPrice` | `number` | 否 | 单价 |
| `source` | `string` | 否 | 数据来源（手动记录一般不写） |

## 分类建议值

`餐饮` / `交通` / `购物` / `居家` / `娱乐` / `医疗` / `工资` / `理财` / `其他`

> 分类仅为建议，实际接受任意字符串。

## 导入容错规则

导入时通过 `normalizeRow()` 归一化，规则如下：

- `direction` 兼容写法：`type` 为 `income` / `收入` / `in` / `+` 均识别为收入，其余一律按支出处理。
- 金额解析失败（非有限正数）记为 `0`；若 `amount` 为 0 但 `quantity` 与 `unitPrice` 有效，则取二者乘积（保留两位小数）。
- `amount` 最终为 0 的行会被视为无效并跳过。
- `date` 非 `YYYY-MM-DD` 格式时回退为当天。
- 其余可选字段缺失时省略（`undefined`）。

## 导出格式

导出时通过 `toExportShape()` 生成，可选字段缺失时显式置为 `null`：

```json
{
  "date": "2026-10-20",
  "type": "expense",
  "category": "餐饮",
  "amount": 35,
  "note": "午餐",
  "product": "麻辣烫",
  "quantity": 1,
  "unit": "份",
  "unitPrice": 35
}
```

> 导出时 `type` 写回 `income`/`expense` 语义（与导入兼容）；订单一律导出为 `expense`，
> 并附 `order_id` / `platform` 等字段保证回灌时被识别为订单（见 [orders.md](./orders.md)）。

[返回索引](../bookkeeping-format.md)
