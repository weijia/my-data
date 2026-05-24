# utags WebDAV 路径规则

## 缺省路径

根据 my-data 目录约定，utags 书签数据缺省存放在：

```
app_data/utags/bookmarks.json
```

## 路径生成规则

书签文件路径由 `buildSyncPath(targetPath, scope)` 函数生成，规则如下：

| 用户配置的 `target.path` | scope | 实际 WebDAV 文件路径 |
|---|---|---|
| `/` 或空或未设置 | `all`（默认） | `/utags-bookmarks.json` |
| `/utags/` | `all` | `/utags/utags-bookmarks.json` |
| `/bookmarks/data/` | `all` | `/bookmarks/data/utags-bookmarks.json` |
| `/my-bookmarks`（无末尾斜杠） | `all` | `/my-bookmarks.json`（文件名优先） |
| `/data/utags.json`（已有 `.json` 后缀） | `all` | `/data/utags.json` |
| `/` 或空或未设置 | `abc123` | `/utags-collection-abc123.json` |
| `/utags/` | `abc123` | `/utags/utags-collection-abc123.json` |

**规则说明**：
- 如果 `target.path` 以 `.json` 结尾，则直接使用该文件名
- 如果 `target.path` 为目录路径（以 `/` 结尾），则在该目录下生成文件
- 默认文件名：`bookmarks.json`（符合 my-data 约定）或 `utags-bookmarks.json`（历史兼容）

## WebDAV 上的文件布局

同步时会在 WebDAV 上创建/更新 **两个文件**：

```
{target.path 所在目录}/
├── bookmarks.json              # 书签数据（BookmarksStore JSON）
└── sync-settings.json          # 同步配置（过滤后的同步设置）
```

**my-data 缺省布局**：

```
app_data/
└── utags/
    ├── bookmarks.json          # 书签数据
    └── sync-settings.json      # 同步配置
```

## WebDAV 连接配置

| 配置项 | 说明 |
|---|---|
| 认证方式 | HTTP Basic Auth（`username:password`） |
| Content-Type | `application/octet-stream` |
| 并发控制 | 使用 ETag + `If-Match` 头实现乐观锁 |

[返回索引](../utags-bookmark-format.md)
