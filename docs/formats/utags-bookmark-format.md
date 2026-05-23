# utags WebDAV 书签数据格式

本文档描述 [utags](https://github.com/weijia/utags) 在 WebDAV 上保存书签的数据格式和路径规则，供其他程序读取已打标签的书签数据。

> **源码参考**（utags v0.14.x）：
> - 类型定义：`packages/webapp/src/types/bookmarks.ts`
> - WebDAV 适配器：`packages/webapp/src/sync/webdav-sync-adapter.ts`
> - 路径构建：`packages/webapp/src/sync/sync-path-builder.ts`
> - 常量定义：`packages/webapp/src/config/constants.js`

---

## 1. WebDAV 文件路径

### 1.1 路径生成规则

书签文件路径由 `buildSyncPath(targetPath, scope)` 函数生成，规则如下：

| 用户配置的 `target.path` | scope | 实际 WebDAV 文件路径 |
|---|---|---|
| `/` 或空或未设置 | `all`（默认） | `/utags-bookmarks.json` |
| `/utags/` | `all` | `/utags/utags-bookmarks.json` |
| `/bookmarks/data/` | `all` | `/bookmarks/data/utags-bookmarks.json` |
| `/my-bookmarks` | `all` | `/my-bookmarks.json`（文件名优先） |
| `/data/utags.json` | `all` | `/data/utags.json`（已有 `.json` 后缀则保留） |
| `/` 或空或未设置 | `abc123` | `/utags-collection-abc123.json` |
| `/utags/` | `abc123` | `/utags/utags-collection-abc123.json` |

**规则说明**：
- 如果 `target.path` 以 `.json` 结尾，则直接使用该文件名（忽略 scope）
- 如果 `target.path` 为目录路径（以 `/` 结尾或包含多级），则在该目录下生成文件
- 文件名默认为 `utags-bookmarks.json`（scope=all）或 `utags-collection-{scope}.json`

### 1.2 WebDAV 上的文件布局

同步时会在 WebDAV 上创建/更新 **两个文件**：

```
{target.path 所在目录}/
├── utags-bookmarks.json        # 书签数据（BookmarksStore JSON）
└── sync-settings.json           # 同步配置（过滤后的同步设置）
```

**示例**：如果 `target.path` 为 `/utags/`，则：

```
/utags/
├── utags-bookmarks.json
└── sync-settings.json
```

### 1.3 WebDAV 连接配置

| 配置项 | 说明 |
|---|---|
| 认证方式 | HTTP Basic Auth（`username:password`） |
| Content-Type | `application/octet-stream` |
| 并发控制 | 使用 ETag + `If-Match` 头实现乐观锁 |

---

## 2. 书签数据格式（BookmarksStore）

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

### 2.1 顶层字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `data` | `Record<string, BookmarkTagsAndMetadata>` | 以 URL 为 key 的书签集合 |
| `meta` | `StoreMeta` | 书签库元信息 |

### 2.2 StoreMeta（`meta` 字段）

| 字段 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `databaseVersion` | `number` | 是 | 数据库 schema 版本号，当前为 **3** |
| `extensionVersion` | `string` | 否 | 扩展版本号 |
| `created` | `number` | 是 | 书签库创建时间（毫秒时间戳） |
| `updated` | `number` | 是 | 最后更新时间（毫秒时间戳） |
| `exported` | `number` | 否 | 最后导出时间（毫秒时间戳） |
| `stats` | `object` | 否 | 统计信息 |
| `lastUploadDevice` | `object` | 否 | 最后上传设备信息（deviceId、browser、os、deviceType、userAgent 等） |

---

## 3. 书签条目格式（BookmarkTagsAndMetadata）

每个 URL 对应一个书签条目，完整结构如下：

```json
{
  "tags": ["标签1", "标签2"],
  "meta": {
    "url": "https://example.com/page1",
    "title": "页面标题",
    "shortTitle": "短标题",
    "description": "描述信息",
    "note": "个人笔记",
    "favicon": "https://example.com/favicon.ico",
    "coverImage": "https://example.com/cover.jpg",
    "mainUrl": "https://example.com",
    "created": 1700000000000,
    "updated": 1700000000000,
    "updated2": 1700000000000,
    "deleted": 1700000000000,
    "lang": "zh-CN",
    "rating": 5,
    "read": true,
    "source": "import"
  },
  "hilights": [
    {
      "created": 1700000000000,
      "updated": 1700000000000,
      "text": "高亮文本",
      "color": "yellow",
      "note": "高亮笔记",
      "type": "highlight"
    }
  ],
  "deletedMeta": {
    "deleted": 1700000000000,
    "actionType": "DELETE"
  },
  "importedMeta": {
    "imported": 1700000000000,
    "source": "chrome",
    "type": "html"
  }
}
```

### 3.1 tags（标签数组）

| 字段 | 类型 | 说明 |
|---|---|---|
| `tags` | `string[]` | 书签的标签列表，至少包含一个标签 |

**特殊标签（系统保留）**：

| 标签 | 常量名 | 说明 |
|---|---|---|
| `._DELETED_` | `DELETED_BOOKMARK_TAG` | 已软删除的书签 |
| `._STARRED_` | `STARRED_BOOKMARK_TAG` | 已收藏/加星的书签 |
| `._ARCHIVED_` | `ARCHIVED_BOOKMARK_TAG` | 已归档的书签 |

### 3.2 meta（书签元数据）

| 字段 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `url` | `string` | 否 | 书签主 URL |
| `title` | `string` | 否 | 书签标题 |
| `shortTitle` | `string` | 否 | 短标题 |
| `description` | `string` | 否 | 描述信息 |
| `note` | `string` | 否 | 个人笔记 |
| `favicon` | `string` | 否 | favicon URL |
| `coverImage` | `string` | 否 | 封面图 URL |
| `mainUrl` | `string` | 否 | 主 URL（用于 URL 归一化） |
| `created` | `number` | 否 | 创建时间（毫秒时间戳） |
| `updated` | `number` | 否 | 最后手动编辑时间（毫秒时间戳） |
| `updated2` | `number` | 否 | 最后修改时间（含自动修改，毫秒时间戳） |
| `deleted` | `number` | 否 | 软删除时间（毫秒时间戳） |
| `lang` | `string` | 否 | 内容语言（如 `"zh-CN"`、`"en"`） |
| `rating` | `number` | 否 | 评分 |
| `read` | `boolean` | 否 | 是否已读 |
| `source` | `string` | 否 | 来源（如 `"import"`） |

### 3.3 hilights（高亮标注，可选）

高亮标注数组，每项结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `created` | `number` | 创建时间（毫秒时间戳） |
| `updated` | `number` | 更新时间（毫秒时间戳） |
| `text` | `string` | 高亮文本内容 |
| `color` | `string` | 高亮颜色（如 `"yellow"`、`"green"`、`"blue"`、`"red"`） |
| `note` | `string` | 高亮笔记 |
| `type` | `string` | 标注类型，通常为 `"highlight"` |

### 3.4 deletedMeta（删除元数据，可选）

| 字段 | 类型 | 说明 |
|---|---|---|
| `deleted` | `number` | 删除时间（毫秒时间戳） |
| `actionType` | `string` | 删除操作类型 |

**actionType 取值**：

| 值 | 说明 |
|---|---|
| `DELETE` | 手动删除 |
| `IMPORT` | 导入时删除 |
| `SYNC` | 同步时删除 |
| `BATCH_DELETE_BOOKMARKS` | 批量删除 |
| `BATCH_REMOVE_TAGS` | 批量移除标签 |
| `LAST_TAG_REMOVED` | 最后一个标签被移除 |

### 3.5 importedMeta（导入元数据，可选）

| 字段 | 类型 | 说明 |
|---|---|---|
| `imported` | `number` | 导入时间（毫秒时间戳） |
| `source` | `string` | 导入来源（如 `"chrome"`、`"edge"`） |
| `type` | `string` | 导入文件类型（如 `"html"`） |

---

## 4. 如何获取打过标签的书签

### 4.1 基本读取步骤

1. **下载书签文件**：从 WebDAV 获取 `utags-bookmarks.json`（或对应路径）
2. **解析 JSON**：得到 `BookmarksStore` 对象
3. **遍历 `data` 对象**：每个 key 是 URL，value 是书签条目
4. **过滤有效书签**：排除带有 `._DELETED_` 标签的条目
5. **读取标签**：从 `tags` 数组获取该书签的所有标签

### 4.2 示例代码（JavaScript）

```javascript
// 1. 从 WebDAV 下载并解析
const response = await fetch(webdavUrl + '/utags-bookmarks.json', {
  headers: { Authorization: 'Basic ' + btoa(username + ':' + password) }
})
const store = await response.json()

// 2. 获取所有打过标签的书签（排除已删除）
const taggedBookmarks = Object.entries(store.data)
  .filter(([, entry]) => !entry.tags.includes('._DELETED_'))
  .map(([url, entry]) => ({
    url,
    title: entry.meta?.title || url,
    tags: entry.tags.filter(t => !t.startsWith('._')),
    note: entry.meta?.note || '',
    created: entry.meta?.created,
    updated: entry.meta?.updated,
    hilights: entry.hilights || []
  }))

// 3. 按标签查找书签
function findByTag(tagName) {
  return taggedBookmarks.filter(b => b.tags.includes(tagName))
}
```

### 4.3 注意事项

- **软删除**：被删除的书签不会从 `data` 中移除，而是添加 `._DELETED_` 标签并设置 `deletedMeta`。读取时应过滤掉这些条目。
- **系统标签**：以 `._` 开头的标签为系统保留标签（`._DELETED_`、`._STARRED_`、`._ARCHIVED_`），通常应排除。
- **URL 作为 key**：`data` 对象的 key 就是书签的 URL，但 `meta.url` 字段也可能存在，两者通常一致。
- **时间戳**：所有时间字段均为毫秒级 Unix 时间戳（`number`），非 ISO 字符串。
- **数据库版本**：读取时应检查 `meta.databaseVersion`，当前版本为 **3**。
