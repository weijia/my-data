# Kids-Point 应用设置

## 应用全局设置

**localStorage 键名**: `kidpoints-settings`

### 数据结构

```json
{
  "adminPassword": "123456",
  "isAuthenticated": false,
  "lastDailyReset": 1746000000000,
  "lastWeeklyReset": 1746000000000,
  "notificationsEnabled": true,
  "theme": "light",
  "webdavSync": {
    "url": "https://example.com/dav/",
    "username": "user",
    "password": "pass",
    "enabled": true,
    "lastSyncTime": 1746000000000
  },
  "locale": "zh"
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `adminPassword` | `string \| null` | 管理员密码，`null` 表示未设置 |
| `isAuthenticated` | `boolean` | 当前是否已登录 |
| `lastDailyReset` | `number` | 上次每日任务重置时间 |
| `lastWeeklyReset` | `number` | 上次每周任务重置时间 |
| `notificationsEnabled` | `boolean` | 是否启用通知 |
| `theme` | `string` | 主题：`"light"` / `"dark"` |
| `webdavSync` | `WebDAVSyncConfig \| null` | WebDAV 同步配置 |
| `locale` | `string` | 语言：`"en"` / `"zh"` |

### WebDAVSyncConfig 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | `string` | WebDAV 服务器 URL |
| `username` | `string` | WebDAV 用户名 |
| `password` | `string` | WebDAV 密码 |
| `enabled` | `boolean` | 是否启用自动同步 |
| `lastSyncTime` | `number \| null` | 上次同步时间 |

---

## 语言设置

**localStorage 键名**: `kidpoints-locale`

**值**: `"en"` 或 `"zh"`

[返回索引](../kids-point-format.md)
