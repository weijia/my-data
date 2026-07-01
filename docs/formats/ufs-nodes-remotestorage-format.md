# UFS 节点 RemoteStorage 数据格式

本文档描述 [vue-element-admin](https://github.com/weijia/vue-element-admin) 项目中 `nodes.vue` 页面通过 RemoteStorage 保存的节点数据结构。

## 存储路径

| 项目 | 说明 |
|------|------|
| **存储目录** | `/ufs-nodes/` |
| **单文件命名** | `{uuid}.json` |
| **Scope** | `remoteStorage.scope('/ufs-nodes/')` |

## 节点数据对象 (Node Object)

每个节点对应一个 JSON 文件，数据结构如下：

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My Node Name",
  "status": "online",
  "last_access": 1719876543210
}
```

### 字段说明

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `uuid` | `string` | 是 | - | 节点唯一标识符，使用 UUID v4 格式 |
| `name` | `string` | 否 | `"New Client"` | 节点显示名称，可在 UI 中编辑 |
| `status` | `string` | 否 | `"active"` | 节点状态。存储时初始为 `active`，运行时被判断为 `online` / `offline` |
| `last_access` | `number` / `string` | 是 | `Date.now()` | 最后访问时间。存储时为时间戳(`number`)，读取后转换为浏览器本地时间字符串 |

### 状态流转

```
创建节点 -> status: "active"
定期更新 -> status: "online"  (最后访问时间在 30 分钟内)
超时离线 -> status: "offline" (最后访问时间超过 30 分钟)
```

### 前端运行时扩展字段

以下字段仅在 Vue 组件运行时使用，**不会**持久化到 RemoteStorage：

| 字段 | 类型 | 说明 |
|------|------|------|
| `connecting` | `boolean` | 是否正在尝试连接该节点，用于 UI 加载状态展示 |

## 本地存储 (localStorage)

当前节点自身的标识保存在浏览器 `localStorage` 中：

| Key | 类型 | 说明 |
|-----|------|------|
| `web-uuid` | `string` | 当前浏览器/节点的 UUID，首次访问时自动生成并持久化 |

> 获取方式：`localStorage.getItem('web-uuid')`，生成方式：`uuidv4()`

## 核心操作 API

| 操作 | 函数 | 说明 |
|------|------|------|
| 获取/创建自身 UUID | `getOrCreateWebUUID()` | 从 localStorage 读取或生成新 UUID |
| 获取或注册节点 | `fetchClients(this)` | 读取自身节点文件，不存在则创建 |
| 列出所有节点 | `listAllClients()` | 遍历 `/ufs-nodes/` 目录下所有 `.json` 文件 |
| 更新节点 | `updateClient(node)` | 更新节点名称和最后访问时间 |
| 更新最后访问 | `updateClientLastAccess()` | 仅更新自身节点的 `last_access` 和 `status` |
| 删除节点 | `remove(filePath)` | 从 RemoteStorage 删除指定节点文件 |

## 自动刷新机制

| 定时器 | 间隔 | 职责 |
|--------|------|------|
| `refreshTimer` | 5 分钟 | 重新拉取所有节点列表 |
| `updateTimer` | 15 分钟 | 更新自身节点的 `last_access` |

## 在线状态判定

节点在线状态由前端根据 `last_access` 计算：

```javascript
const diffMinutes = (now - lastAccessTime) / (1000 * 60)
status = diffMinutes <= 30 ? 'online' : 'offline'
```

即：最后访问时间在 **30 分钟以内** 视为在线，否则视为离线。

## 数据来源

- **源码文件**: `src/views/login-db/nodes.vue`
- **RemoteStorage 模块**: `src/views/login-db/remotestorage.js`
- **Git Commit**: `f3bc6ac37e0f2eae4079edc29944947a35705c93`
