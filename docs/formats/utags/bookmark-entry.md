# 书签条目格式（BookmarkTagsAndMetadata）

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

## tags（标签数组）

| 字段 | 类型 | 说明 |
|---|---|---|
| `tags` | `string[]` | 书签的标签列表，至少包含一个标签 |

**特殊标签（系统保留）**：

| 标签 | 常量名 | 说明 |
|---|---|---|
| `._DELETED_` | `DELETED_BOOKMARK_TAG` | 已软删除的书签 |
| `._STARRED_` | `STARRED_BOOKMARK_TAG` | 已收藏/加星的书签 |
| `._ARCHIVED_` | `ARCHIVED_BOOKMARK_TAG` | 已归档的书签 |

## meta（书签元数据）

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

## hilights（高亮标注，可选）

高亮标注数组，每项结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `created` | `number` | 创建时间（毫秒时间戳） |
| `updated` | `number` | 更新时间（毫秒时间戳） |
| `text` | `string` | 高亮文本内容 |
| `color` | `string` | 高亮颜色（如 `"yellow"`、`"green"`、`"blue"`、`"red"`） |
| `note` | `string` | 高亮笔记 |
| `type` | `string` | 标注类型，通常为 `"highlight"` |

## deletedMeta（删除元数据，可选）

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

## importedMeta（导入元数据，可选）

| 字段 | 类型 | 说明 |
|---|---|---|
| `imported` | `number` | 导入时间（毫秒时间戳） |
| `source` | `string` | 导入来源（如 `"chrome"`、`"edge"`） |
| `type` | `string` | 导入文件类型（如 `"html"`） |

[返回索引](../utags-bookmark-format.md)
