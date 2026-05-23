# utags WebDAV 路径规则

## 路径生成规则

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

## WebDAV 上的文件布局

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

## WebDAV 连接配置

| 配置项 | 说明 |
|---|---|
| 认证方式 | HTTP Basic Auth（`username:password`） |
| Content-Type | `application/octet-stream` |
| 并发控制 | 使用 ETag + `If-Match` 头实现乐观锁 |

[返回索引](../utags-bookmark-format.md)
