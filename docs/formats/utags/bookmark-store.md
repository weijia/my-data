# BookmarksStore 结构

书签数据存储为单个 JSON 文件，顶层结构如下：

```json
{
  "data": {
    "<URL>": { ... },
    "<URL>": { ... }
  },
  "meta": {
    "databaseVersion": 3,
    "extensionVersion": "0.14.2",
    "created": 1700000000000,
    "updated": 1700000000000
  }
}
```

## 顶层字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `data` | `Record<string, BookmarkTagsAndMetadata>` | 以 URL 为 key 的书签集合 |
| `meta` | `StoreMeta` | 书签库元信息 |

## StoreMeta（`meta` 字段）

| 字段 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `databaseVersion` | `number` | 是 | 数据库 schema 版本号，当前为 **3** |
| `extensionVersion` | `string` | 否 | 扩展版本号 |
| `created` | `number` | 是 | 书签库创建时间（毫秒时间戳） |
| `updated` | `number` | 是 | 最后更新时间（毫秒时间戳） |
| `exported` | `number` | 否 | 最后导出时间（毫秒时间戳） |
| `stats` | `object` | 否 | 统计信息 |
| `lastUploadDevice` | `object` | 否 | 最后上传设备信息（deviceId、browser、os、deviceType、userAgent 等） |

每个 URL 对应的书签条目格式详见 [bookmark-entry.md](./bookmark-entry.md)。

[返回索引](../utags-bookmark-format.md)
