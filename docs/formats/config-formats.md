# 应用配置格式

---

## 1. 窗口配置

**文件路径**: `config/window_config.json`

### 数据结构

```json
{
  "top": "50vh",
  "left": "50vw",
  "width": "600px",
  "height": "200px"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `top` | `string` | 窗口顶部位置（CSS 值，如 `"50vh"`、`"0px"`） |
| `left` | `string` | 窗口左侧位置（CSS 值，如 `"50vw"`、`"0px"`、`"auto"`） |
| `width` | `string` | 窗口宽度（CSS 值，如 `"600px"`、`"100vw"`、`"300px"`） |
| `height` | `string` | 窗口高度（CSS 值，如 `"200px"`、`"100vh"`、`"40px"`） |

> **注意**: 停靠模式下可能包含 `right` 字段（如 `"0px"`）。

---

## 2. WebDAV 连接配置

**文件路径**: `config/webdav_config.json`

**localStorage 变量名**: `webDAVConfig`

### 数据结构

```json
{
  "url": "https://xxx.teracloud.jp/dav",
  "username": "yyy",
  "password": "zzz"
}
```

| 字段 | 类型 | 必选 | 说明 |
|------|------|------|------|
| `url` | `string` | 是 | WebDAV 服务器基础 URL（不含末尾斜杠，保存时自动去除） |
| `username` | `string` | 是 | HTTP Basic Auth 用户名 |
| `password` | `string` | 是 | HTTP Basic Auth 密码 |

> **注意**: 该配置同时存储在 WebDAV（`config/webdav_config.json`）和页面 localStorage（变量名 `webDAVConfig`）中，格式一致。

---

## 3. 共享 localStorage 约定（推荐）

为方便同一用户在浏览器中使用多个应用时避免重复配置 WebDAV 连接信息，推荐各应用遵循以下 localStorage 约定：

### 共享 Key

| localStorage Key | 类型 | 说明 |
|---|---|---|
| `webDAVConfig` | `WebDAVConfig` | WebDAV 连接配置（推荐共享） |

### 使用方式

1. **写入**：任何应用在用户配置 WebDAV 连接后，将配置写入 `localStorage['webDAVConfig']`
2. **读取**：其他应用启动时优先检查 `localStorage['webDAVConfig']`，若已存在则自动填充，无需用户重复输入
3. **格式**：与 `config/webdav_config.json` 完全一致

### 示例

```javascript
// 读取（其他应用启动时）
const shared = localStorage.getItem('webDAVConfig')
if (shared) {
  const config = JSON.parse(shared) // { url, username, password }
  // 自动填充，跳过配置步骤
}

// 写入（用户配置完成后）
localStorage.setItem('webDAVConfig', JSON.stringify({
  url: 'https://xxx.teracloud.jp/dav',
  username: 'yyy',
  password: 'zzz'
}))
```

> **注意**: 这是推荐约定，非强制要求。各应用可选择使用自己的 key，但建议同时写入 `webDAVConfig` 以便其他应用复用。
