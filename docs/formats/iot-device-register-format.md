# 设备注册与上报格式（iot_device_register）

本文档定义德雨科技物联网平台"方案 A：设备自助注册 + 注册令牌"的**设备端入网注册接口契约**与**MQTT 上报格式**。适用对象：平台后端（`/prod-api/registration-token*`、`/prod-api/device/register`）与设备端 `cpp-daemon` 的 `iot_report` 插件。

> 占位约定：本文所有示例一律使用占位值（`xxxPK` / `xxxDEV` / `xxxTOKEN` / `xxxSECRET`），**禁止出现真实凭据**；admin 账号信息不写入本文档。接口契约以本文档为设计基线，若后续实现有出入，**以代码为准**。

## 数据来源

| 侧 | 模块 / 接口 | 生成内容 | 存储位置 |
|---|---|---|---|
| 平台 | `POST /prod-api/registration-token` | 注册令牌（明文仅当次返回） | 平台库 `iot_registration_token.token_hash` |
| 平台 | `POST /prod-api/device/register` | 设备三元组 `{productKey, deviceNum, secret}`（secret 明文仅当次返回） | 平台库 `iot_device`（secret 只存 hash） |
| 设备端 | `cpp-daemon iot_report` 插件 | 本地持久化的三元组 | 设备端本地凭据文件（路径以代码为准） |
| 设备端 | MQTT 上行 | 属性/事件上报报文 | Broker → 平台摄取管线 → TDengine |

> 与 [host-disk-info-format.md](./host-disk-info-format.md) 的关系：那份文档描述 home_cloud 采集的主机/磁盘遥测数据的落盘格式；本文档描述**这些遥测数据如何经 MQTT 上报到 IoT 平台**的注册与报文契约。设备端 `iot_report` 上报的身份字段（`serial_number` / `hostname` / 磁盘序列号 / 卷标）与 host-disk-info-format.md 中采集的字段同源，但在 MQTT payload 中按本文第 3.4 节脱敏约定输出。

---

## 1. 注册令牌接口（管理端）

### 1.1 生成注册令牌

- **方法/路径**：`POST /prod-api/registration-token`
- **鉴权**：管理端 JWT
- **请求**：

```json
{
  "productId": 1,
  "remark": "xxxREMARK"
}
```

- **响应**（`token` 明文仅本次返回）：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "id": 1,
    "productId": 1,
    "token": "xxxTOKEN_ONCE"
  }
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `productId` | number | 是 | 令牌绑定的产品 ID |
| `remark` | string | 否 | 用途备注（如"车间网关批量注册用"） |
| `data.id` | number | 是 | 令牌记录 ID（用于列表/吊销） |
| `data.token` | string | 是 | 令牌明文，**仅当次返回**；平台库只存 hash |

### 1.2 注册令牌列表

- **方法/路径**：`GET /prod-api/registration-token/list`
- **鉴权**：管理端 JWT
- **响应**（不回显 token 明文）：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "rows": [
      {
        "id": 1,
        "productId": 1,
        "productKey": "xxxPK",
        "remark": "xxxREMARK",
        "status": "ACTIVE",
        "createdAt": "2026-10-25T10:00:00"
      }
    ],
    "total": 1
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | string | `ACTIVE`（有效）/ `REVOKED`（已吊销） |

### 1.3 吊销注册令牌

- **方法/路径**：`POST /prod-api/registration-token/{id}/revoke`
- **鉴权**：管理端 JWT
- **响应**：`{ "code": 200, "msg": "操作成功" }`
- 吊销后，设备端持该令牌调用 `/device/register` 一律 401 拒绝。

---

## 2. 设备自助注册接口（设备端免登录）

### 2.1 接口契约

- **方法/路径**：`POST /prod-api/device/register`
- **鉴权**：**免登录**，请求 body 携带 `token` 字段（注册令牌明文）
- **请求体**：

```json
{
  "token": "xxxTOKEN_ONCE",
  "device_num": "xxxDEV0001",
  "name": "xxxGateway"
}
```

> `device_num` 与 `name` 均为可选；`device_num` 不传时由服务端生成。

- **响应**（新建设备）：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "productKey": "xxxPK",
    "deviceNum": "xxxDEV0001",
    "secret": "xxxSECRET_ONCE"
  }
}
```

- **响应**（device_num 已存在，重置 secret）：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "productKey": "xxxPK",
    "deviceNum": "xxxDEV0001",
    "secret": "xxxSECRET_RESET"
  }
}
```

#### 请求/响应字段

| 字段 | 位置 | 类型 | 必填 | 说明 |
|------|------|------|------|------|
| `token` | Body | string | 是 | 注册令牌明文（管理端按产品生成） |
| `device_num` | Body | string | 否 | 设备端本机生成的唯一编号；不传则服务端生成 |
| `name` | Body | string | 否 | 设备显示名 |
| `data.productKey` | Response | string | 是 | 令牌绑定产品的 productKey（设备端不透传） |
| `data.deviceNum` | Response | string | 是 | 回显请求的 device_num（或服务端生成值） |
| `data.secret` | Response | string | 是 | 设备密钥明文，**仅当次返回**；库内只存 hash |

### 2.2 服务端处理顺序

1. 取 Body `token` → 计算 hash → 查 `iot_registration_token`；不存在、`status=REVOKED` 或已过期 → 返回 401；
2. 按令牌绑定 `productId` 查产品；产品缺失时按绑定信息自动建产品（具体以代码为准）；
3. 按 `device_num` 在该产品下查重（未传则服务端生成新 device_num）：
   - 已存在 → 重置 `deviceSecret`，返回新 secret；
   - 不存在 → 创建设备记录，生成新 secret；
4. `deviceSecret` 落库前只存 hash，明文仅写入本次 HTTP 响应。

### 2.3 错误码

| HTTP | code | msg | 含义 | 设备端动作 |
|------|------|-----|------|------------|
| 200 | 200 | 操作成功 | 注册/重置成功 | 持久化三元组，转 MQTT 接入 |
| 401 | 401 | 注册令牌无效或已吊销/过期 | token 缺失、hash 不匹配、已吊销或已过期 | 不重试，告警人工介入 |
| 400 | 400 | 参数错误 | device_num 格式非法等 | 检查本机 device_num 生成逻辑 |
| 500 | 500 | 服务器内部错误 | 产品/设备落库异常 | 指数退避重试 |

### 2.4 静态路径注册顺序注意事项

`/prod-api/device/register` 为**免登录静态路由**，部署时须满足：

1. Nginx `/prod-api` 反代不拦截该路径（与 `/prod-api/emqx/auth|acl|events` 同列白名单）；
2. FastAPI 路由中该路径**不挂 JWT 鉴权依赖**；
3. 路由注册须排在 `/prod-api/device/{id}` 等动态路由之前，或使用独立路径段，避免被动态路由捕获吞掉；
4. 生产环境建议该路径仅允许 TLS，并对源 IP 做基础限流。

---

## 3. MQTT 上报规范

### 3.1 连接参数

| 项 | 值 |
|---|---|
| 协议 | MQTT over TLS |
| 端口 | `8883` |
| `client_id` | `{productKey}@{deviceNum}`，如 `xxxPK@xxxDEV0001` |
| `username` | `{deviceNum}` |
| `password` | `{secret}`（注册接口当次返回、设备端本地持久化的密钥明文） |
| QoS | `1` |

> 设备端启动流程：本地已有 `deviceNum+secret` → 直接连 MQTT；本地无凭据 → 先调 `/device/register` 取得并持久化后再连 MQTT。**不重复注册**。

### 3.2 主题（Topic）

| 方向 | Topic | QoS | 说明 |
|---|---|---|---|
| 属性上报 | `/{productKey}/{deviceNum}/property/post` | 1 | 遥测属性 |
| 事件上报 | `/{productKey}/{deviceNum}/event/post` | 1 | 故障/告警事件 |
| 属性下发 | `/{productKey}/{deviceNum}/property/set` | 1 | 平台下行期望值 |
| 服务调用 | `/{productKey}/{deviceNum}/service/invoke` | 1 | 远程控制 |

### 3.3 Payload 结构

属性上报报文：

```json
{
  "payload": {
    "temperature": { "value": 36.5 },
    "hostname": { "value": "xxxABCD" },
    "serial_number": { "value": "xxx1234" },
    "uuid": { "value": "550e8400-e29b-41d4-a716-446655440000" }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `payload` | object | 属性 map，key 为物模型 identifier |
| `<identifier>.value` | any | 属性值；温度等数值型直接给数值 |

> **硬约束**：payload 中**不得包含任何凭据**（`productKey` / `deviceNum` / `secret` / 注册令牌明文）；身份字段按 3.4 节脱敏。

### 3.4 脱敏约定

| 字段 | 脱敏规则 | 示例 |
|------|----------|------|
| `serial_number`（主板/设备序列号） | `xxx` + 原值后 4 位 | `ABC12345678` → `xxx6789` |
| `device_name`（设备名） | `xxx` + 后 4 位 | `m10-gateway-01` → `xxx-01` |
| `hostname`（系统主机名） | `xxx` + 后 4 位 | `DESKTOP-ABC1234` → `xxx1234` |
| 磁盘序列号（`physical_disks[].SerialNumber`） | `xxx` + 后 4 位 | `S5GXNF0R123456` → `xxx3456` |
| 卷标（`logical_disks[].VolumeName`） | `xxx` + 后 4 位 | `System-Data-01` → `xxx-01` |
| `uuid`（设备 UUID） | **保留明文，不脱敏** | `550e8400-e29b-41d4-a716-446655440000` |

> "后 4 位"取原值去掉空白与分隔符后的最后 4 个字符；不足 4 位时全部保留并前缀 `xxx`。

---

## 4. 与 host-disk-info-format.md 的字段对齐

`cpp-daemon iot_report` 插件上报的身份/环境字段，与 `host-disk-info-format.md` 中 home_cloud 采集字段同源对齐如下：

| MQTT payload identifier | host-disk-info-format.md 来源字段 | 脱敏 |
|---|---|---|
| `uuid` | `host_info.device_uuid` / 记录根 `uuid` | 保留 |
| `hostname` | `host_info.hostname` | xxx + 后 4 位 |
| `serial_number` | `host_info.serial_number` | xxx + 后 4 位 |
| `device_name` | `host_info.device_name` | xxx + 后 4 位 |
| `disk_serial`（可选） | `physical_disks[].SerialNumber` | xxx + 后 4 位 |
| `volume_name`（可选） | `logical_disks[].VolumeName` | xxx + 后 4 位 |

---

## 5. 安全约束汇总

1. 注册令牌与 admin 密码**不得写死进代码、文档、日志**；
2. 注册令牌与设备密钥在平台库中**只存 hash**，明文仅当次 HTTP 响应可见；
3. 设备端本地持久化的三元组文件权限建议 `0600`，不含注册令牌明文（注册成功后设备端不再需要令牌）；
4. MQTT payload 不含任何凭据，身份字段按第 3.4 节脱敏；
5. `/device/register` 免登录接口生产环境强制 TLS + 限流。

---

## 6. 数据来源（源码）

- **平台注册令牌接口**：`backend/app/api/v1/endpoints/registration_token.py`（路径以代码为准）
- **平台设备自助注册接口**：`backend/app/api/v1/endpoints/device.py` → `POST /device/register`
- **设备端注册与上报**：`cpp-daemon` 的 `iot_report` 插件，配置项 `platform/register_token`
- **MQTT 上报主题**：平台摄取管线订阅 `+/property/post`、`+/event/post`（对应 FR-MQTT-02）
