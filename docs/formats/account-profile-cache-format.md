# 账号属性缓存数据格式（account-profile-cache）

本文档描述跨平台账号公开属性缓存的统一格式，用于在多次检索同一账号时复用已核查的性别、地区等信息，避免重复访问主页。格式不绑定具体平台，新增平台时只需在 `accounts[].platform` 填入对应标识。

## 适用场景

- 检索/浏览各社区、论坛、社交平台的帖子或账号时，缓存其公开可见的个人属性。
- 同一账号再次出现时，先查缓存；仅在缓存缺失或需要更新时才访问主页。
- 区分"已核实条目"（`accounts`）与"信息不全待补条目"（`pending`）。

## 数据文件

| 路径 | 说明 |
|------|------|
| `O:\app_data\{平台}\account-profiles.json` | 外部盘实际数据落点 |
| `app_data/{平台}/account-profiles.json` | 仓库内同步副本（纳入版本管理 / WebDAV 同步） |

> `{平台}` 为平台小写标识，如 `douban`。外部盘与仓内副本结构一致，以其中一份为准、定期互相同步。

## 顶层结构

```json
{
  "schemaVersion": 1,
  "kind": "account-profile-cache",
  "description": "跨平台账号公开属性缓存，条目按 platform + accountId 唯一。",
  "lastUpdated": "2026-11-13T14:30:00+08:00",
  "accounts": [],
  "pending": []
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `schemaVersion` | number | 格式版本号，当前为 `1` |
| `kind` | string | 固定为 `account-profile-cache`，便于程序识别文件类型 |
| `description` | string | 人类可读说明 |
| `lastUpdated` | string | 最近一次写入时间，ISO 8601 带时区 |
| `accounts` | array | 已核实、属性较完整的账号条目 |
| `pending` | array | 已部分采集但信息不全、待补全的账号条目 |

## 账号条目字段

### `accounts[]`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `platform` | string | 是 | 平台标识，如 `douban`；格式通用，可扩展 |
| `accountId` | string | 是 | 平台内唯一账号 ID（数字 ID 或主页 slug）；与 `platform` 组成唯一键 |
| `nickname` | string | 否 | 主页显示名；未知留空字符串 |
| `gender` | string | 是 | 性别枚举：`female` / `male` / `unknown`（未设置或无标识） |
| `region` | string | 否 | 公开显示的地区/IP 属地；未知留空字符串。属地不等于籍贯 |
| `verifiedAt` | string | 是 | 本次核查日期，`YYYY-MM-DD` |
| `sourceRefs` | string[] | 否 | 来源帖子/内容 ID，便于回溯 |
| `note` | string | 否 | 补充说明，如"主页无性别标识" |

### `pending[]`

字段与 `accounts[]` 基本一致，额外包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `todo` | string[] | 待补字段名，如 `["nickname", "body"]` |

`pending` 条目在补全并核实后，应迁移到 `accounts`，并移除 `todo`。

## 性别取值约定

| 取值 | 含义 |
|------|------|
| `female` | 主页明确标识为女性 |
| `male` | 主页明确标识为男性 |
| `unknown` | 主页未设置性别、无标识，或无法核实；不得仅凭昵称、正文措辞推断 |

> 不同平台的性别标识来源不同（如页面 class、资料字段），采集时以该平台公开可见的标识为准；无标识一律记 `unknown`。

## 唯一性与更新规则

- 唯一键：`platform + accountId`，重复条目应合并而非追加。
- 再次核查同一账号时，更新对应条目的 `gender`、`region`、`nickname`、`verifiedAt`，并把新的 `sourceRefs` 并入。
- 缓存只存公开属性，不保存密码、私信、Cookie、会话等敏感信息。
- 地区/属地可能随登录环境变化，长期未核查的条目可标注 `verifiedAt` 供判断时效。

## 读取示例（伪代码）

```javascript
function findAccount(cache, platform, accountId) {
  return cache.accounts.find(
    (a) => a.platform === platform && a.accountId === accountId
  );
}

// 仅取已核实女性账号
const verifiedFemales = cache.accounts.filter(
  (a) => a.gender === "female"
);
```
