# MQTT 连接配置字符串格式（mqtt-config-string）

> 适用项目：[mqtthub](https://github.com/weijia/mqtthub)（MQTT 聊天示例，字符串的"发源地"）与
> [mqtt-crypto-client](https://github.com/weijia/mqtt-crypto-client)（加密 MQTT 客户端库，本格式的
> 解析/序列化实现方）。
>
> 代码出处（以实际核对为准）：
> - 序列化 / 反序列化（原始实现）：`mqtthub/src/chat.js` 中的 `serializeConfig()` /
>   `parseConfig()`
> - 解析 / 序列化（本库）：`mqtt-crypto-client/src/config-string.js` 中的 `parseConfigString()` /
>   `serializeConfigString()`，并在 `src/options.js` 的 `resolveOptions()` 中直接支持字符串输入
>   （`createClient(str)` 即可）
> - 加密互通：`mqtt-crypto-client/src/crypto.js`（与 `CryptoJS.AES` 逐字节兼容）

## 1. 这是什么

MQTT 连接配置字符串是一种**一行文本化的 MQTT 连接 + 加密配置**，用于在系统之间便捷地交换
"连哪个 broker、哪个 topic、用什么口令加密"。它是**便捷交换格式**，不是持久化格式——持久化仍建议
用 JSON / 对象保存；本字符串只是方便：

- 在设置页的输入框里**复制 / 粘贴导入**（把一台机器的配置原样迁到另一台或另一个系统）；
- 编码进 **URL 查询参数**（如 `?broker=...&topic=...&password=...`），通过链接分享配置。

> 与 [配置同步组后端配置字符串](./config-sync-backend-format.md) 不是同一套：那里是
> `type:id:key=value,…`（按 `:` / `,` 分隔，未做 URL 编码）；本格式是 `key=value&…`
> 按 `&` 分隔、value 经 `encodeURIComponent` 编码，二者**不互通**，请勿混用。

## 2. 通用格式

```
key=value&key=value&...
```

- 每一项 `key=value`，**value 必须（且已）经 `encodeURIComponent` 编码**，因此 value 里可以安全
  包含 `&`、`=`、`#`、`:`、空格、中文、emoji 等任意字符。
- 项之间用 `&` 连接（与 URL 查询串完全一致）。
- 字段顺序不敏感。

### 已知字段

| key | 必填 | 含义 | 备注 |
|-----|------|------|------|
| `broker` | 是 | MQTT broker URL（MQTT over WebSocket 常用 `wss://host:port/mqtt`，也支持 `ws://` / `mqtt://` / `mqtts://` / `tcp://` / `tls://`） | 必须是合法 URL 且协议受支持 |
| `topic` | 是 | 订阅 / 发布的 topic（业务 topic，**不应含** `+` / `#` 通配符） | |
| `password` | 否 | AES 口令；**对应 mqtt-crypto-client 的 `secret`**。为空表示不加密（`encrypt=false`） | 这是跨系统的"口令"字段名，**不是** broker 的登录密码（broker 鉴权另由 `auth` 等机制提供） |
| `name` | 否 | 可读标签，仅用于 UI 展示；**不参与连接** | mqtthub 在缺失时会用 broker 主机名兜底生成 |

> 除上表 4 个已知 key 外，**其它任何 key 都会被原样保留为扩展字段**（不参与连接，但可被 UI 或上层
> 逻辑使用，例如 `clientIdPrefix`、`clientIdSuffix` 等）。这保证了格式向前兼容。

### 序列化规则（`serializeConfigString` / mqtthub `serializeConfig`）

1. 取已知字段：`name`（可选）、`broker`、`topic`、`password`（库的 `secret` 仅在 `encrypt !== false`
   时映射为 `password`；`encrypt=false` 时 `password` 为空，表示不加密）。
2. 每个 value 经 `encodeURIComponent` 编码后用 `=` 拼接；项之间用 `&` 连接。
3. mqtthub 固定输出顺序为 `name & broker & topic & password`；本库 `serializeConfigString` 同理
   （name 在前）。顺序不影响解析。
4. 扩展字段在已知字段之后逐个写出（排除已映射的 `broker`/`topic`/`secret`/`encrypt`/`name`）。

### 反序列化规则（`parseConfigString` / mqtthub `parseConfig`）

1. 按 `&` 拆项；每项按第一个 `=` 拆 `key` / `value`，value 经 `decodeURIComponent` 解码。
2. 缺少 `=` 的片段被容忍并跳过；key 为空的片段跳过。
3. **缺少 `broker` 或 `topic` 视为非法，返回 `null`（mqtthub）/ 抛出 `InvalidConfigError`
   （mqtt-crypto-client）**。
4. `password` 非空 → `{ secret: password, encrypt: true }`；`password` 为空或缺失 →
   `{ secret: '', encrypt: false }`。
5. `name` 原样保留；其余未知 key 透传为扩展字段。

## 3. 与 mqtt-crypto-client 的字段映射

| 字符串 key | mqtt-crypto-client options | 说明 |
|-----------|---------------------------|------|
| `broker` | `broker` | 直传 |
| `topic` | `topic` | 直传 |
| `password` | `secret` | **改名**：库内称"密钥 / secret"，字符串里称"口令 / password" |
| （派生） | `encrypt` | 由 `password` 是否非空推导：`password` 非空 → `true`，否则 `false` |
| `name` | （不进入连接） | 仅作标签，被 `resolveOptions` 忽略 |
| 其它扩展 key | 原样透传 | 例如 `clientIdPrefix`，可被库识别并使用 |

在 mqtt-crypto-client 里，下列写法等价：

```js
import { createClient } from 'mqtt-crypto-client';

// 写法一：直接传字符串（最便捷，适合粘贴/URL）
const client = createClient(
  'name=我的频道&broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fdemo&password=shared-secret'
);

// 写法二：先解析再合并（适合需要覆盖默认参数）
import { parseConfigString } from 'mqtt-crypto-client';
const client2 = createClient({
  ...parseConfigString(str),
  clientIdPrefix: 'hub_',   // 额外覆盖
});
```

## 4. 加密互通说明（重要）

mqtt-crypto-client 的加密与 mqtthub（`CryptoJS.AES.encrypt(payload, passphrase)`）**逐字节兼容**：

- 格式：OpenSSL `"Salted__"` 头（8 字节）+ 8 字节随机 salt + AES-256-CBC 密文，整体 base64。
- 密钥派生：EVP_BytesToKey(MD5, 1 次迭代)，key 32 字节 + IV 16 字节——与 CryptoJS 的 OpenSSLKDF 一致。
- 填充：PKCS#7。

因此：**同一 `password`，在 mqtthub 与本库之间可以互加密 / 互解密**（只要连的是同一个 broker/topic
且口令相同）。但注意——

> ⚠️ **消息体语义各系统不同**：mqtthub 发送的是聊天消息体
> `{ id, msgId, user, msg, time }`；mqtt-crypto-client 发送的是命令信封
> （`msg` 为 `JSON.stringify({ action, data })`）。本配置字符串只统一了**连接 + 加密口令**，
> 不统一消息负载结构。两个系统用同一字符串连上同一个 broker/topic 后，能互相"读得出密文并解开"，
> 但业务层能否理解对方消息取决于各自的消息 schema。

## 5. 示例

```
# 最简：加密聊天频道（broker/topic/password 三项即可）
broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fmqtthub%2Fchat%2F2026&password=Your_Secret_Password

# 带 name 标签（便于 UI 显示）
name=默认&broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fmqtthub%2Fchat%2F2026&password=Your_Secret_Password

# 不加密（password 为空）
broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fdemo

# 带扩展字段（如指定 clientId 前缀）
name=demo&broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fdemo&password=secret&clientIdPrefix=hub_

# 作为 URL 查询串分享
https://example.com/setup?broker=wss%3A%2F%2Fbroker.emqx.io%3A8084%2Fmqtt&topic=secure%2Fdemo&password=secret
```

## 6. 注意事项 / 已知限制

- **value 必须 URL 编码**：`broker` / `topic` 里的 `:`、`/`、中文、空格，`password` 里的 `&`、`=` 等
  都必须经 `encodeURIComponent`。未编码的 `&` / `=` 会被错误切分字段；未编码的 `+` 在某些解码场景下
  会被当作空格。分享前请用 `serializeConfigString()`（或 `encodeURIComponent`）生成，**不要手敲**。
- **`password` 是加密口令，不是 broker 登录密码**：broker 本身若需要用户名/密码登录，走 MQTT 的
  `auth`（CONNECT 报文 username/password）或 TLS 客户端证书，与本字符串的 `password`（AES 口令）是
  两回事，别混淆。
- **空 `password` 即不加密**：同一 broker/topic 上，加密端发的密文，未设 `password` 的端会按明文处理
  而解不开；务必两端口令一致。
- **这是交换格式，不是持久化格式**：长期保存请用对象 / JSON；本字符串可能随扩展字段变化而变长。
- **`name` 不参与连接**：它只是展示标签，重复 / 缺失都不影响连接。
- **与配置同步组后端配置字符串不互通**：见第 1 节，分隔符与编码规则不同。
- **消息负载 schema 不统一**：见第 4 节。

## 7. 代码出处与版本

- 字符串"发源地"：`mqtthub/src/chat.js` 的 `serializeConfig()` / `parseConfig()`（聊天示例，UI 导入/导出）。
- 本库解析/序列化：`mqtt-crypto-client` ≥ **0.2.3** 提供 `parseConfigString()` /
  `serializeConfigString()`，`resolveOptions()` / `createClient()` 可直接吃字符串。
