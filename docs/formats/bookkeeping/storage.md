# Bookkeeping 存储格式

本文档描述 Bookkeeping 应用数据在本地与云端的存储结构。

**源码参考**：`src/lib/bookkeeping-store.ts`、`src/lib/pouchdb.ts`

> 本格式为**统一 `type` 模型 + 每条记录一个独立 PouchDB 文档**。旧版"单文档 `app:bookkeeping` 存
> `{ transactions, prices }`"的形态已被取代，首次加载时自动迁移（见 [§5](#5-旧版迁移)）。

## 1. 本地存储（PouchDB）

应用数据存储在名为 `onenav` 的 PouchDB 数据库中，**每条记账记录对应一个文档**：

- **数据库名**：`onenav`
- **IndexedDB 名称**：`_pouch_onenav`
- **文档 `_id`**：`bk:<type>:<id>`（前缀 `bk:` + 记录类型 + 记录 ID）

### 文档结构

```json
{
  "_id": "bk:tx:1746000000000-a1b2c3",
  "_rev": "1-xxxxxxxx",
  "type": "tx",
  "id": "1746000000000-a1b2c3",
  "direction": "expense",
  "amount": 35,
  "category": "餐饮",
  "note": "午餐",
  "date": "2026-10-20",
  "createdAt": 1746000000000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | `string` | PouchDB 文档 ID，格式 `bk:<type>:<id>` |
| `_rev` | `string` | PouchDB 修订版本（由数据库维护） |
| `type` | `'tx' \| 'price' \| 'order'` | 记录类型（body 与 `_id` 冗余，兼作查询键） |
| 其余字段 | — | 记录本体，见各类型文档（[transactions](./transactions.md) / [orders](./orders.md) / [prices](./prices.md)） |

### 查询

```ts
// 全部记录（按 _id 字典序）
allDocs({ startkey: 'bk:', endkey: 'bk:￿', include_docs: true })

// 按类型范围查询（如仅订单）
allDocs({ startkey: 'bk:order:', endkey: 'bk:order:￿' })
```

排序（按 `date` 倒序的统一时间线）在内存中对查询结果做，跨类型统一展示。

### 软删除

删除记录时置 `deleted: true`（不物理删除，便于同步恢复）；加载时过滤掉 `deleted` 的文档。

## 2. WebDAV 落盘格式

除本地 PouchDB 外，写入时会聚合全部记录写入 WebDAV 文件：

- **文件路径**：`app_data/onenav/apps/bookkeeping-records.json`

### 文件结构

```json
{
  "updatedAt": 1746000000000,
  "records": [
    { "type": "tx", "id": "1746000000000-a1b2c3", "direction": "expense", "amount": 35, "date": "2026-10-20" },
    { "type": "order", "id": "1746000000001-d4e5f6", "platform": "jd", "orderId": "3658473011154524", "amount": 248.03, "date": "2026-10-21" }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `updatedAt` | `number` | 最后更新时间，毫秒级 Unix 时间戳 |
| `records` | `BookkeepingRecord[]` | 全部记录（统一 `type` 模型的数组） |

> 旧版 WebDAV 文件为 `app_data/onenav/apps/bookkeeping.json`（`{ appId, data: { transactions, prices } }`），
> 新版改用 `bookkeeping-records.json`；本地无 `bk:` 文档时会回退读 WebDAV 文件并写回 PouchDB。

## 3. 读取与写入顺序

**写入**（`putRecords` / `removeRecord(s)` 之后）：
1. 写入本地 PouchDB（`bk:<type>:<id>` 文档）
2. 若已配置 WebDAV → 聚合全部记录写入 `app_data/onenav/apps/bookkeeping-records.json`
3. 若已配置 RemoteStorage → 触发自动同步（push）

**读取**（`loadAllRecords`）：
1. 检测并迁移旧版 `app:bookkeeping` 单文档（见 §5）
2. 读本地 PouchDB 全部 `bk:` 文档
3. 本地无记录时回退读 WebDAV `bookkeeping-records.json`，并写回本地 PouchDB 缓存

## 4. RemoteStorage 跨设备同步

- 记录以 PouchDB 文档形式存储，**整库**（数据库 `onenav`）通过 RemoteStorage 同步引擎跨设备同步。
- 每条记录是独立文档，单条变更只同步该文档（细粒度）。
- 保存后由自动同步调度触发 push，无需额外配置应用级路径。

## 5. 旧版迁移

加载时检测旧 shape：`app:bookkeeping` 文档存在、且 `bk:` 前缀无文档时执行迁移（幂等，迁移后删除旧文档）：

1. 旧 `transactions` 中**无** `platform`/`orderId` → `TxRecord`（`direction = 原 type`）。
2. 旧 `transactions` 中**有** `platform`/`orderId` → `OrderRecord`（拆出电商字段）。
3. 旧 `prices` → `PriceRecord`（`kind` 不变，补 `type: 'price'`）。
4. 逐条写为 `bk:<type>:<id>` 文档。

## 目录布局

```
app_data/
└── onenav/
    └── apps/
        └── bookkeeping-records.json   # 记账全部记录（聚合兜底）
```

[返回索引](../bookkeeping-format.md)
