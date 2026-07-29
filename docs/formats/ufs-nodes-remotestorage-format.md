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
| `capabilities` | `array` | 否 | `[]` | 该节点对外暴露的可被控制能力清单，详见[控制能力](#控制能力-capabilities) |
| `inbox` | `array` | 否 | `[]` | 发往本节点的命令队列，由控制端写入、被控端读取执行，详见[命令队列](#命令队列-inbox) |
| `mqtt` | `object` | 否 | - | MQTT 远程命令通道配置；节点启用 MQTT 时上报，供控制端定位发布地址与加密密钥，详见[MQTT 远程命令通道](#mqtt-远程命令通道-mqtt) |

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

## 设备控制 (Device Control)

在原有"节点上报状态"的基础上，扩展出**节点间互相下发指令**的能力：一个节点（控制端）可以向另一个节点（被控端）发送控制命令，被控端执行后回填状态与结果，实现双向回执。

命令通道复用节点自身的 JSON 文件：控制端向目标节点的 `inbox` 追加命令，被控端在自己的刷新周期读取并执行，结果写回同一条命令记录。

> 设计取舍：命令直接写在 `{uuid}.json` 内（无需新增目录），但意味着控制端的写操作与被控端的心跳/遥测写操作会落在**同一个文件**上。双方都必须遵守"读取-合并-写回"原则（见[并发与一致性](#并发与一致性)），不可整文件覆盖。

### 控制能力 (Capabilities)

`capabilities` 为被控端对外声明的**可被执行的能力清单**，控制端据此知道能下发哪些动作。数组中每个元素：

```json
{
  "id": "relay1",
  "type": "switch",
  "label": "继电器1",
  "params": { "state": ["on", "off"] }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 能力唯一标识，命令的 `action` 必须与之匹配 |
| `type` | `string` | 否 | 能力类型，如 `switch`(开关) / `action`(一次性动作) / `range`(数值范围) / `trigger`(触发) |
| `label` | `string` | 否 | UI 展示名称 |
| `params` | `object` | 否 | 该能力接受的参数 schema（如枚举值、取值范围），供控制端构造命令与 UI 渲染 |

> **标准能力命名约定（建议）**：为让不同节点互相识别，建议对常见能力使用固定的 `id`。已约定的有：
>
> | `id` | `type` | `label` | `params` | 说明 |
> |------|--------|---------|----------|------|
> | `power` | `action` | 电源控制 | `{"mode":["sleep","hibernate","shutdown","restart"]}` | 电源管理；`mode="hibernate"` 即休眠，`"sleep"` 即睡眠（挂起） |
> | `relayN` | `switch` | 继电器N | `{"state":["on","off"]}` | 受控开关，N 为编号 |
> | `snapshot` | `action` | 拍照 | - | 触发一次拍照/截图 |
>
> 其它能力可自行扩展 `id`，但建议全局唯一且语义稳定。

### 命令队列 (Inbox)

`inbox` 为发往本节点的命令数组。每条命令对象：

```json
{
  "cmd_id": "7f3b1c2e-8a3d-4f5a-9b2c-1d2e3f4a5b6c",
  "from": "550e8400-e29b-41d4-a716-446655440000",
  "action": "relay1",
  "params": { "state": "on" },
  "created_at": 1719876543210,
  "status": "pending",
  "result": null,
  "updated_at": 1719876600000
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `cmd_id` | `string` | 是 | 命令唯一标识（UUID v4），用于幂等去重与回执查询 |
| `from` | `string` | 是 | 控制端节点的 `uuid` |
| `action` | `string` | 是 | 要执行的能力 `id`，必须存在于目标节点 `capabilities` |
| `params` | `object` | 否 | 命令参数，需符合该能力 `params` 约定 |
| `created_at` | `number` | 是 | 命令创建时间（时间戳） |
| `status` | `string` | 是 | 命令状态，状态机见下 |
| `result` | `any` | 否 | 执行结果；`pending`/`executing` 时为 `null`，`done` 时为结果值，`failed` 时为错误原因 |
| `updated_at` | `number` | 否 | 状态最后变更时间（时间戳） |

#### 命令状态机

```
控制端写入        -> status: "pending"
被控端开始执行   -> status: "executing"
执行成功         -> status: "done"      (填写 result)
执行失败         -> status: "failed"    (result 为错误原因)
动作不存在/无权限 -> status: "rejected"
```

#### 完整节点对象示例（含控制字段）

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "客厅控制板",
  "status": "online",
  "last_access": 1719876543210,
  "capabilities": [
    { "id": "relay1", "type": "switch", "label": "继电器1", "params": { "state": ["on", "off"] } },
    { "id": "snapshot", "type": "action", "label": "拍照" }
  ],
  "inbox": [
    {
      "cmd_id": "7f3b1c2e-8a3d-4f5a-9b2c-1d2e3f4a5b6c",
      "from": "660f9500-e29b-41d4-a716-446655440001",
      "action": "relay1",
      "params": { "state": "on" },
      "created_at": 1719876543210,
      "status": "done",
      "result": { "state": "on" },
      "updated_at": 1719876600000
    }
  ]
}
```

#### Windows 节点示例（电源控制 / 休眠）

Windows 节点通常用**统一的 `power` 能力**暴露电源管理，以 `mode` 参数区分睡眠、休眠、关机、重启，避免为每种状态各建一条能力：

`capabilities` 中声明：

```json
{ "id": "power", "type": "action", "label": "电源控制", "params": { "mode": ["sleep", "hibernate", "shutdown", "restart"] } }
```

控制端下发**休眠**命令（`mode: "hibernate"`）：

```json
{
  "cmd_id": "9a2c1d3e-8a3d-4f5a-9b2c-1d2e3f4a5b6c",
  "from": "controller-uuid",
  "action": "power",
  "params": { "mode": "hibernate" },
  "created_at": 1719876543210,
  "status": "pending",
  "result": null,
  "updated_at": null
}
```

被控端执行完成后回填：

```json
{
  "status": "done",
  "result": { "mode": "hibernate", "ok": true },
  "updated_at": 1719876600000
}
```

> 说明：Windows 下休眠对应 Hibernate（S4，需 `powercfg -h on` 启用），由被控端以具备权限的进程执行（如 `shutdown /h` 触发休眠）；若只需挂起内存可用 `mode: "sleep"`（S3，如 `rundll32.exe powrprof.dll,SetSuspendState 0,1,0`）。被控端实现 `execInbox` 时按 `mode` 映射到底层系统命令即可。

### 控制流程

1. **控制端下发命令**：读取目标节点 `{uuid}.json` → 在 `inbox` 追加一条 `status:"pending"` 的命令（以 `cmd_id` 去重，避免重复下发）→ 写回文件。
2. **被控端轮询执行**：被控端在本节点的刷新周期（建议复用 `updateTimer` 或新增 `commandTimer`）扫描 `inbox` 中 `status:"pending"` 的命令，按 `action` 匹配 `capabilities` 执行；执行中置 `executing`，完成后置 `done`/`failed` 并填 `result`、`updated_at`，写回自身文件。
3. **控制端查询回执**：控制端按 `cmd_id` 读取目标节点 `inbox` 中对应命令的 `status`/`result` 获取执行结果（双向回执）。

### 并发与一致性

由于命令与被控端心跳/遥测写入同一文件，**任意一方都不得整文件覆盖**：

- **控制端追加命令**：先读取现有节点文件，仅向 `inbox` 追加新 `cmd_id`，保留原 `capabilities`、其它 `inbox` 项及遥测字段，再写回。
- **被控端更新自身**：更新 `last_access`/`status` 及执行回执时，必须保留 `inbox`、`capabilities` 中由控制端写入的内容。
- **幂等去重**：以 `cmd_id` 去重，重复写入同一 `cmd_id` 应被忽略。
- **冲突处理**：若出现并发写冲突（last-writer-wins），以 `updated_at` / `cmd_id` 较晚者为准；业务层可对关键命令加重试。

## MQTT 远程命令通道 (mqtt)

节点可额外暴露一个基于 MQTT 的**远程命令通道**：控制端将命令以 AES 加密后发布到 MQTT 主题，被控端订阅主题、解密后在本地 shell 执行。节点将其 MQTT 连接信息随心跳上报到 RemoteStorage（**仅启用时**，未启用则字段整体省略），使控制端无需额外约定即可知道往何处发布、用什么密钥加密。

> 与 `inbox` 的区别：`inbox` 走 RemoteStorage 文件（双向回执、轮询），`mqtt` 通道走 MQTT Broker（近实时、单向推送命令）。两者可并存：控制端既能在 `inbox` 留痕，也能经 MQTT 即时下发。

节点 `mqtt` 字段结构（位于节点对象根级）：

```json
{
  "mqtt": {
    "enabled": true,
    "broker": "tcp://broker.emqx.io:1883",
    "topic": "ufsd/cmd",
    "username": "device-001",
    "password": "broker-pass",
    "secret": "aes-decrypt-key",
    "client_id": "go-daemon-550e8400"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | `boolean` | 是 | 是否启用 MQTT 远程命令通道；`false` 时本字段整体省略 |
| `broker` | `string` | 启用时必填 | MQTT Broker 地址，如 `tcp://host:1883`、`ssl://host:8883`、`wss://host:8084/mqtt` |
| `topic` | `string` | 启用时必填 | 控制端发布命令、被控端订阅的主题 |
| `username` | `string` | 否 | Broker 登录用户名 |
| `password` | `string` | 否 | Broker 登录密码（明文，注意存储安全） |
| `secret` | `string` | 启用时必填 | AES 解密密钥，对应发送端 `CryptoJS.AES.encrypt(text, password)` 中的 `password`；控制端须用同一密钥加密命令 |
| `client_id` | `string` | 否 | 自定义 MQTT ClientID；留空时由被控端按节点 UUID 自动生成 |

### 消息格式与执行

控制端向 `topic` 发布的消息为 **CryptoJS 兼容的 OpenSSL 格式密文**（前缀 `Salted__` + 随机 salt + Base64，内部为 AES-256-CBC，密钥由 `EVP_BytesToKey`(MD5) 从 `secret` 派生）。被控端解密后：

- 若明文为 JSON 且含 `msg` 字段，则取 `msg` 作为命令执行；
- 否则整段明文作为命令执行。

被控端在本地 shell 执行命令：`sh -c "<cmd>"`（Linux）/ `cmd /C "<cmd>"`（Windows）。

> 安全提示：`mqtt` 中的 `password` 与 `secret` 均为敏感明文，随节点文件持久化于 RemoteStorage。请确保 RemoteStorage 的访问权限（token / ACL）严格受限，否则拿到存储的人即可向该节点下发任意命令。建议 Broker 启用 TLS（`ssl://` 或 `wss://`）并配置订阅 ACL。

### 完整节点对象示例（含 MQTT 通道）

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "客厅控制板",
  "status": "online",
  "last_access": 1719876543210,
  "capabilities": [
    { "id": "power", "type": "action", "label": "电源控制", "params": { "mode": ["sleep","hibernate","shutdown","restart"] } }
  ],
  "mqtt": {
    "enabled": true,
    "broker": "tcp://broker.emqx.io:1883",
    "topic": "ufsd/cmd",
    "username": "device-001",
    "password": "broker-pass",
    "secret": "aes-decrypt-key",
    "client_id": "go-daemon-550e8400"
  },
  "inbox": []
}
```

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
| 下发命令 | `sendCommand(targetUuid, cmd)` | 向目标节点 `inbox` 追加一条 `pending` 命令（按 `cmd_id` 去重） |
| 查询回执 | `pollCommand(targetUuid, cmdId)` | 按 `cmd_id` 读取目标节点 `inbox` 中命令的 `status` / `result` |
| 执行命令队列 | `execInbox(this)` | 被控端扫描自身 `inbox` 的 `pending` 命令并执行，回填 `status` / `result` |

## 自动刷新机制

| 定时器 | 间隔 | 职责 |
|--------|------|------|
| `refreshTimer` | 5 分钟 | 重新拉取所有节点列表 |
| `updateTimer` | 15 分钟 | 更新自身节点的 `last_access` |
| `commandTimer` | 5 分钟 | 被控端执行自身 `inbox` 中的 `pending` 命令并回填结果（可与 `refreshTimer` 合并） |

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
