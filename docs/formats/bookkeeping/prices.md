# Bookkeeping 价格记录（PriceRecord）

价格记录是统一记录模型中 `type: 'price'` 的一条，用 **判别联合**（discriminated union）按 `kind` 字段区分三种类型。

> 旧版该数据位于 `BookkeepingData.prices` 数组；新版每条价格记录带顶层 `type: 'price'`，
> 与流水、订单共同组成统一的 `BookkeepingRecord[]`。

## 三种类型

| `kind` | 含义 | 代表价 |
|--------|------|--------|
| `item` | 单品 | `unitPrice`（单价） |
| `combo` | 套餐 | `unitPrice`（套餐价） |
| `voucher` | 代金券 | `cost`（实付金额） |

所有类型共享的字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | `'price'` | 是 | 记录类型，固定为 `price` |
| `id` | `string` | 是 | 唯一 ID（时间戳-随机串） |
| `kind` | `'item' \| 'combo' \| 'voucher'` | 是 | 子类型判别字段 |
| `date` | `string` | 否 | 日期，格式 `YYYY-MM-DD`；缺省/非法时回退当天 |
| `source` | `string` | 否 | 来源；订单提取的价格记录为 `京东订单` / `淘宝订单`，手动记录一般为商家名 |
| `createdAt` | `number` | 否 | 创建时间，毫秒级 Unix 时间戳；缺省为写入时刻 |

---

## 1. 单品（item）

```json
{
  "type": "price",
  "kind": "item",
  "id": "1746000000000",
  "date": "2026-10-20",
  "product": "鸡蛋",
  "unit": "斤",
  "unitPrice": 5.8,
  "source": "菜市场",
  "createdAt": 1746000000000
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `product` | `string` | 是 | 商品名 |
| `unit` | `string` | 否 | 单位（斤 / 个 / 粒 等） |
| `unitPrice` | `number` | 是 | 单价（必须为正数） |

---

## 2. 套餐（combo）

```json
{
  "type": "price",
  "kind": "combo",
  "id": "1746000000001",
  "date": "2026-10-20",
  "name": "宫保鸡丁套餐",
  "unitPrice": 68,
  "items": [
    { "product": "宫保鸡丁", "unit": "份", "quantity": 1 },
    { "product": "米饭", "unit": "碗", "quantity": 2 }
  ],
  "source": "XX饭店",
  "createdAt": 1746000000001
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | `string` | 是 | 套餐名 |
| `unitPrice` | `number` | 是 | 套餐价（必须为正数） |
| `items` | `ComboItem[]` | 否 | 套餐包含的菜品清单，缺省为空数组 |

**ComboItem 结构**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `product` | `string` | 是 | 菜品名（为空则该条被过滤掉） |
| `unit` | `string` | 否 | 单位 |
| `quantity` | `number` | 否 | 数量 |
| `unitPrice` | `number` | 否 | 单菜单价 |

---

## 3. 代金券（voucher）

```json
{
  "type": "price",
  "kind": "voucher",
  "id": "1746000000002",
  "date": "2026-10-20",
  "name": "超市券",
  "cost": 75,
  "faceValue": 100,
  "source": "XX超市",
  "createdAt": 1746000000002
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | `string` | 是 | 券名 |
| `cost` | `number` | 是 | 实付金额（必须为正数） |
| `faceValue` | `number` | 是 | 面值（必须为正数） |

---

## 导入容错规则

导入时通过 `normalizePriceRow()` 归一化：

- `kind` 不是 `item` / `combo` / `voucher` 之一的行视为无效并跳过（保证旧版本读新类型不崩）。
- `item` 缺 `product` 或 `unitPrice ≤ 0` → 跳过。
- `combo` 缺 `name` 或 `unitPrice ≤ 0` → 跳过；`items` 非数组时按空数组处理。
- `voucher` 缺 `name`、`cost`、`faceValue` 任一，或金额非正数 → 跳过。
- `date` 非 `YYYY-MM-DD` 时回退当天；`source` 为空时省略。

## 订单提取的价格记录

导入电商订单时，订单商品中「有单价」者（`price > 0.01`，过滤红包占位）会按商品名生成 `kind: 'item'`
的价格记录，`source` 为 `京东订单` / `淘宝订单`；同一商品多次出现时按商品名去重、保留最新报价。
详见 [orders.md §4](./orders.md#4-价格提取订单--报价)。

## 导出格式

导出时通过 `toPriceExportShape()` 生成，可选字段缺失时显式置为 `null`，字段顺序按 `kind` 区分：

```json
[
  { "kind": "item",    "date": "2026-10-20", "product": "鸡蛋", "unit": "斤", "unitPrice": 5.8, "source": null },
  { "kind": "combo",   "date": "2026-10-20", "name": "宫保鸡丁套餐", "unitPrice": 68,
    "items": [ { "product": "宫保鸡丁", "unit": "份", "quantity": 1 } ], "source": null },
  { "kind": "voucher", "date": "2026-10-20", "name": "超市券", "cost": 75, "faceValue": 100, "source": null }
]
```

[返回索引](../bookkeeping-format.md)
