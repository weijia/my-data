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
