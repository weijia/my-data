# Bookkeeping 交易记录（transactions）

交易记录是 `BookkeepingData.transactions` 数组的元素，表示一条收支流水。

## 数据结构

```json
[
  {
    "id": "1746000000000",
    "type": "expense",
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
]
```

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一 ID（时间戳字符串，由 `randId()` 生成） |
| `type` | `'income' \| 'expense'` | 否 | 收支类型，缺省按 `expense` 处理 |
| `amount` | `number` | 是 | 金额；若缺失但提供了 `quantity` 与 `unitPrice`，会自动按 `quantity × unitPrice` 计算 |
| `category` | `string` | 否 | 消费分类，缺省为 `"其他"` |
| `note` | `string` | 否 | 备注，缺省为空字符串 |
| `date` | `string` | 否 | 日期，格式 `YYYY-MM-DD`；缺省/非法时回退为当天 |
| `createdAt` | `number` | 否 | 创建时间，毫秒级 Unix 时间戳；缺省为写入时刻 |
| `product` | `string` | 否 | 关联商品名，用于「上次报价带入」 |
| `quantity` | `number` | 否 | 数量 |
| `unit` | `string` | 否 | 单位（如 粒 / 斤 / 个 / 件） |
| `unitPrice` | `number` | 否 | 单价 |

## 分类建议值

`餐饮` / `交通` / `购物` / `居家` / `娱乐` / `医疗` / `工资` / `理财` / `其他`

> 分类仅为建议，实际接受任意字符串。

## 导入容错规则

导入时通过 `normalizeRow()` 归一化，规则如下：

- `type` 兼容写法：`income` / `收入` / `in` / `+` 均识别为收入，其余一律按支出处理。
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

> 导出文件名为 `bookkeeping-tx-{YYYY-MM-DD}.json`（仅含当前记录 Tab 的数据）。

[返回索引](../bookkeeping-format.md)
