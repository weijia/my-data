# utags WebDAV 书签数据格式

本文档描述 [utags](https://github.com/weijia/utags) 在 WebDAV 上保存书签的数据格式和路径规则，供其他程序读取已打标签的书签数据。

> **源码参考**（utags v0.14.x）：
> - 类型定义：`packages/webapp/src/types/bookmarks.ts`
> - WebDAV 适配器：`packages/webapp/src/sync/webdav-sync-adapter.ts`
> - 路径构建：`packages/webapp/src/sync/sync-path-builder.ts`
> - 常量定义：`packages/webapp/src/config/constants.js`

## 文档索引

| 文档 | 说明 |
|------|------|
| [WebDAV 路径规则](./utags/webdav-path.md) | 文件路径生成规则、WebDAV 上的文件布局 |
| [BookmarksStore 结构](./utags/bookmark-store.md) | 顶层 JSON 结构和 StoreMeta 字段 |
| [书签条目格式](./utags/bookmark-entry.md) | 每个书签的 tags、meta、hilights 等详细字段 |

## 快速示例：获取打过标签的书签

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

## 注意事项

- **软删除**：被删除的书签不会从 `data` 中移除，而是添加 `._DELETED_` 标签。读取时应过滤。
- **系统标签**：以 `._` 开头的标签为系统保留（`._DELETED_`、`._STARRED_`、`._ARCHIVED_`），通常应排除。
- **时间戳**：所有时间字段均为毫秒级 Unix 时间戳（`number`），非 ISO 字符串。
- **数据库版本**：读取时应检查 `meta.databaseVersion`，当前版本为 **3**。
