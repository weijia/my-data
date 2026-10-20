# Bookkeeping 存储格式

本文档描述 Bookkeeping 应用数据在本地与云端的存储结构。

**源码参考**：`src/lib/app-data.ts`、`src/lib/pouchdb.ts`

## 1. 本地存储（PouchDB）

应用数据存储在名为 `onenav` 的 PouchDB 数据库中，每个应用对应一个文档：

- **数据库名**：`onenav`
- **IndexedDB 名称**：`_pouch_onenav`
- **文档 `_id`**：`app:bookkeeping`（前缀 `app:` + 应用 ID）

### 文档结构

```json
{
  "_id": "app:bookkeeping",
  "_rev": "1-xxxxxxxx",
  "type": "app-data",
  "appId": "bookkeeping",
  "data": {
    "transactions": [],
    "prices": []
  },
  "updatedAt": 1746000000000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | `string` | PouchDB 文档 ID，固定为 `app:bookkeeping` |
| `_rev` | `string` | PouchDB 修订版本（由数据库维护） |
| `type` | `string` | 固定为 `app-data` |
| `appId` | `string` | 应用 ID，固定为 `bookkeeping` |
| `data` | `BookkeepingData` | 应用业务数据，见 [顶层数据结构](../bookkeeping-format.md#顶层数据结构) |
| `updatedAt` | `number` | 最后更新时间，毫秒级 Unix 时间戳 |

## 2. WebDAV 落盘格式

除本地 PouchDB 外，应用数据会额外写入 WebDAV 文件（在配置目录同级）：

- **文件路径**：`app_data/onenav/apps/bookkeeping.json`

### 文件结构

```json
{
  "appId": "bookkeeping",
  "data": {
    "transactions": [],
    "prices": []
  },
  "updatedAt": 1746000000000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `appId` | `string` | 应用 ID，固定为 `bookkeeping` |
| `data` | `BookkeepingData` | 应用业务数据 |
| `updatedAt` | `number` | 最后更新时间，毫秒级 Unix 时间戳 |

> **注意**：WebDAV 文件多一层 `{ appId, data, updatedAt }` 包装，实际业务数据在 `data` 字段内。读取时应取 `parsed.data`。

## 3. 读取与写入顺序

**写入**（`save`）：
1. 写入本地 PouchDB（`app:bookkeeping` 文档）
2. 若已配置 WebDAV → 写入 `app_data/onenav/apps/bookkeeping.json`
3. 若已配置 RemoteStorage → 触发自动同步（push）

**读取**（`load`）：
1. 优先读本地 PouchDB
2. 本地缺失时回退读 WebDAV，并写回本地 PouchDB 缓存

## 4. RemoteStorage 跨设备同步

- 应用数据以 PouchDB 文档形式存储，**整库**（数据库 `onenav`）通过 RemoteStorage 同步引擎跨设备同步。
- 保存后由自动同步调度触发 push，无需额外配置应用级路径。

## 目录布局

```
app_data/
└── onenav/
    └── apps/
        └── bookkeeping.json      # 记账应用数据
```

[返回索引](../bookkeeping-format.md)
