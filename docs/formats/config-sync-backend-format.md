# 配置同步组（配置仓）后端配置字符串格式

> 适用项目：[zen-fs-config-admin](https://github.com/weijia/zen-fs-config-admin)（配置同步组的
> 管理界面），底层由 [zen-fs-config](https://github.com/weijia/zen-fs-config) 提供后端注册表。
>
> 代码出处（以第一项为准，文档主体按它核对）：
> - 序列化 / 反序列化：`FluentOS-On-Web/dir-struct-gen/vendor/zen-fs-config-ui.js`
>   中的 `serializeBackend(type, id, options, description)` / `deserializeBackend(str, metadataList)`
> - 后端类型元数据：同一 vendor 文件中的 `registerBackend(...)` 调用
> - 上游参考：`zen-fs-config-admin/src/backend-config-string.ts`（`serializeBackend` /
>   `deserializeBackend`）与 `zen-fs-config-admin/src/register-backends.ts`；上游实现与本
>   vendor 版存在少量差异，见各节标注
>
> 已对照上述 vendor 文件核对（2026-11-02）：WebDAV 字段为 `baseUrl` / `username` /
> `password` / `token` / `basePath` / `purgeLegacyMtime`（vendor 版不是上游 admin 的
> `url` / `rootPath`）；序列化还会额外过滤 `undefined` / `null`；反序列化不识别
> `accountBackendId`（见第 7 节）。

## 1. 这是什么

配置同步组（即配置仓 `ConfigRepo`）由一组**后端（backend）**构成，每个后端是一个同步副本
（如一个 GitHub 仓库、一个 WebDAV 目录、一个 RemoteStorage 账户）。每个后端在内部以
`BackendDescriptor` 结构存储：

```ts
interface BackendDescriptor {
  id: string;          // 组内唯一后端 ID（用户定义，自动生成形如 github-<时间戳>）
  type: string;        // 后端类型，必须已通过 registerBackend 注册
  options: Record<string, string>;  // 该后端类型的配置字段
  description?: string; // 可选的可读备注
}
```

**配置字符串（config string）** 就是上述 `BackendDescriptor` 的**一行文本化形式**，用于
在 UI 里复制 / 粘贴导入单个后端（例如把一台机器的配置仓后端原样迁到另一台）。它不是网络
协议，也不是持久化格式——持久化仍以 JSON（`backends` 数组）保存；配置字符串只是便捷交换格式。

## 2. 通用格式

```
type:id:key=value,key=value,...,desc=description
```

按分隔符拆解：

| 段 | 分隔符 | 说明 |
|----|--------|------|
| 第 1 段 `type` | 第一个 `:` | 后端类型，必须是已注册类型（见第 3 节）。未知类型反序列化报错。 |
| 第 2 段 `id` | 第二个 `:` | 组内唯一后端 ID，**必填**，为空则反序列化返回 `null`。 |
| 第 3 段 | 之后 | `key=value` 列表，项之间用 `,` 分隔。 |

- 第 3 段内部：`key=value` 用 `=` 拆分；`key === 'desc'` 被视为**备注**（description），
  不进入 `options`。其余 key 进入 `options`。
- 段与段之间**只**用 `:` 分隔：因此 value / key 里**不能包含 `:` `,` `=`** 字符
  （例如 `baseUrl` 里的 `https://` 也不允许——见下方"注意事项"）。
- 末尾若只有 `desc` 而无其它 options，第 3 段形如 `desc=...`。

### 序列化规则（`serializeBackend`）

1. 取 `Object.entries(options)`，过滤掉值为空字符串的项（空值不写出）。admin 的
   `backend-config-string.ts` 只判断 `v !== ''`；FluentOS-On-Web 的 vendor 版本还额外过滤
   `undefined` / `null`。
2. 拼接 `[type, id]`，若有 options 则追加 options 串，若有 `description` 则追加 `desc=...`。
3. 用 `:` 连接。

→ 因此 `InMemory` 这类没有有效 options 的后端，序列化结果就是 `InMemory:mem-1`
（没有尾随冒号）。

### 反序列化规则（`deserializeBackend`）

1. 按第一个 `:` 取 `type`；未知 `type` → 返回 `null`。
2. 剩余部分按第二个 `:` 拆成 `id` 与 `optionsStr`；无第二冒号时 `id=剩余全部`、`optionsStr=''`。
   `id` 为空 → 返回 `null`。
3. `optionsStr` 按 `,` 拆项，每项按 `=` 拆 `key`/`value`；`key==='desc'` 进 `description`，
   其余合并进 `options`，并与该类型的 `defaultOptions` 合并（反序列化侧会补全默认值）。

## 3. 各后端类型的字段表

下表为 vendor 中实际注册 / 核心内置的**所有后端类型**及其配置字段。"必填" 列依据元数据
`fields[].required`；"默认值" 列来自各后端的 `defaultOptions`（注册工厂可能另有兜底，见备注）。
`WebStorage` 行来自上游 admin 适配层，vendor 未注册该类型。

| 后端 `type` | 字段 key | 必填 | 默认值 | 含义 |
|------------|----------|------|--------|------|
| `WebStorage` | `storageType` | 否 | `localStorage` | 浏览器本地存储桶：`localStorage` 或 `sessionStorage` |
| `GitHub` | `owner` | 是 | `''` | 仓库所有者（用户名 / 组织） |
| | `repo` | 是 | `''` | 仓库名 |
| | `branch` | 否 | `main` | 分支名 |
| | `token` | 否 | `''` | Personal Access Token（`ghp_…`）；公开仓库可空 |
| | `baseUrl` | 否 | `''`（→ `https://api.github.com`） | GitHub API 基址（企业版 / 镜像可改） |
| `Gitee` | `owner` | 是 | `''` | 仓库所有者 |
| | `repo` | 是 | `''` | 仓库名 |
| | `branch` | 否 | `master` | 分支名 |
| | `token` | 否 | `''` | Gitee 私人令牌 |
| | `baseUrl` | 否 | `''`（→ `https://gitee.com/api/v5`） | Gitee API 基址 |
| `WebDAV` | `baseUrl` | 是 | `''` | WebDAV 服务器基础 URL（如 `https://dav.example.com/remote.php/dav/files/user/`） |
| | `username` | 否 | `''` | Basic Auth 用户名 |
| | `password` | 否 | `''` | Basic Auth 密码 |
| | `token` | 否 | `''` | 可选 Bearer Token |
| | `basePath` | 否 | `/` | 在该 WebDAV 下的根子路径（如 `/zen-fs-config/`） |
| | `purgeLegacyMtime` | 否 | `"false"` | 为布尔 `true` 时执行一次旧版 `.mtime` sidecar 清理；字段 UI 输出的是字符串 `"true"` / `"false"`，而 vendor 实现用严格 `=== true` 判断，故 UI 选 `"true"` 实际不会触发（见第 5 节） |
| `RemoteStorage` | `href` | 是 | `''` | 用户地址 `user@host`，或完整存储 URL（`https://…`） |
| | `token` | 是 | `''` | Bearer Token（`rs-…`） |
| | `basePath` | 否 | `/` | 账户内的子树路径（如 `/zen-fs-config/`） |
| `InMemory` | `maxSize` | 否 | `''` | 内存上限（字节数）；留空时注册工厂按 `100 * 1024 * 1024`（100 MiB）兜底 |
| | `label` | 否 | `''` | 标签；留空时注册工厂自动生成 `zen-fs-config-<n>` |
| `IndexedDB` | `storeName` | 否 | `''` | IndexedDB 存储库名（浏览器本地主副本常用） |
| `Folder` | `path` | 否 | `''` | Node.js 本地目录绝对路径（Node 环境主副本常用）；UI 未标 required，但为空会在运行时失败 |

> 说明：
> - `GitHub` / `Gitee` / `WebDAV` / `RemoteStorage` 在 admin 中被显式注册（覆盖核心内置的同名
>   类型，字段一致）；`InMemory` / `IndexedDB` / `Folder` 为 `zen-fs-config` 核心 / `folder-backend`
>   自带。最终可用类型集合 = `listBackendMetadata()` 的并集。
> - `WebStorage` 的 `storageType` 是 **admin 适配层**字段（`register-backends.ts` 把
>   `localStorage` / `sessionStorage` 映射成 `Storage` 对象后传给核心 `@zenfs/dom` 的
>   `WebStorage`；核心 `options` 实际是 `storage` 对象）。FluentOS-On-Web 的
>   `dir-struct-gen/vendor/zen-fs-config-ui.js` **没有注册** `WebStorage`，在那里粘贴
>   `WebStorage:...` 会因未知类型而反序列化失败；该 vendor UI 实际可用类型为
>   `InMemory` / `IndexedDB` / `GitHub` / `Gitee` / `RemoteStorage` / `WebDAV` / `Folder`。
> - 配置同步组里**通常的副本组合**：1 个本地主副本（浏览器=`IndexedDB`、Node=`Folder`）+ 若干
>   远端副本（GitHub / Gitee / WebDAV / RemoteStorage）。`WebStorage` / `InMemory` 为本地临时存储，
>   一般不当作跨机同步副本。

## 4. 各后端配置字符串示例

```
# GitHub（公开仓库，无 token）
GitHub:github-weijia:owner=weijia,repo=my-configs,branch=main,desc=my main config repo

# Gitee（带 token、自定义分支）
Gitee:gitee-weijia:owner=weijia,repo=my-configs,branch=master,token=gitee_xxxx,desc=gitee mirror

# WebDAV（teracloud 示例；baseUrl 含 :// 无法写入配置字符串，需在 UI 表单填写 baseUrl）
WebDAV:webdav-teracloud:username=yyy,password=zzz,basePath=/zen-fs-config/,desc=teracloud dav

# RemoteStorage（user@host 地址，自动 WebFinger 发现存储 URL）
RemoteStorage:rs-5apps:href=weijia@5apps.com,token=rs_xxxx,basePath=/zen-fs-config/,desc=5apps

# WebStorage（浏览器本地）
WebStorage:ws-local:storageType=localStorage

# IndexedDB（浏览器本地主副本）
IndexedDB:idb-1:storeName=zen-fs-config

# Folder（Node 本地主副本）
Folder:folder-local:path=/path/to/store

# InMemory（无有效 options，序列化为 type:id）
InMemory:mem-1
```

## 5. 注意事项 / 已知限制

- **value 不能含 `:` `,` `=`**：分隔符会被错误切分。最典型的是 `GitHub`/`Gitee` 的 `baseUrl`
  默认写的是完整 URL `https://api.github.com`——但序列化时该字段值为空字符串（用户未改）会被
  过滤掉；一旦用户填入真实 URL（含 `:` 与 `/`），**该配置字符串将无法正确反序列化**，需通过
  UI 表单填写或后续版本对 URL 做转义。当前实现未对 value 做 URL-encode，遇到真实 `baseUrl` 时
  请优先用 UI 表单而非粘贴字符串。
- **`purgeLegacyMtime` 的字符串/布尔差异**：字段是 `select`，值只能是字符串 `"true"` /
  `"false"`，但 vendor 注册工厂用 `if (options.purgeLegacyMtime === true)` 判断，因此从 UI 选择
  `"true"` 不会触发 `.mtime` sidecar 清理；只有以布尔 `true` 传给工厂时才生效。
- **`desc` 是保留 key**：options 里不要使用名为 `desc` 的字段，会被当成备注。
- **空值被过滤**：某个字段留空（空字符串）时不会出现在字符串里；反序列化侧用该类型
  `defaultOptions` 补全，因此空值 ≠ 缺失。
- **`id` 必须唯一**：同一配置同步组内两个后端不能共用一个 `id`，否则保存时报
  "Backend ID already exists"。
- 配置字符串仅描述**单个后端**；一个配置同步组由多个后端组成，对应 `backends` 数组里的多条
  记录，需逐条复制 / 导入。

## 6. 与持久化格式的关系

配置仓的持久化仍是以 JSON 保存的 `BackendDescriptor[]`（位于配置仓 meta 中，键 `backends`）：
`{ id, type, options, description }`。配置字符串只是其中**单条记录**的文本交换形式，二者字段
一一对应，可无损互转（空值 / `desc` 特殊处理见上）。

## 7. 数据同步组（data-sync group）的扩展

数据同步组（Data Groups，对应 `AppDataGroup`，描述符为 `AppDataBackendDescriptor`）**复用同一套
基础配置字符串格式**（`type:id:options...,desc=...`），描述符另有 `accountBackendId` 与
`createdAt` 两个字段，但它们在 vendor 版配置字符串中的处理与上游 admin 不同：

- **`accountBackendId`**：可选，声明该后端复用某个**配置同步组后端**的账号凭证
  （token / owner / …），这样数据组里只需填"存储位置"字段，不必重复填账号。
  - **上游 admin**（`backend-config-string.ts`）：作为**保留段**
    `accountBackendId=<id>`，与 `desc=` 同级（不进入 `options`），序列化时写出、反序列化时还原。
  - **FluentOS vendor**（`vendor/zen-fs-config-ui.js`）：`serializeBackend(type, id, options,
    description)` 不会写出该字段；`deserializeBackend(str, metadataList)` 只把 `desc` 当保留
    key，`accountBackendId` 会被当成普通 option。因此在 vendor UI 里粘贴带该段的字符串，需要
    再通过 UI / 持久化 JSON 设置描述符字段。
- **`createdAt`**：内部去重时间戳（毫秒）。**不进入配置字符串**——反序列化由调用方重置为
  `Date.now()`，原样搬运会导致去重逻辑误判（旧副本被优先保留）。

示例（仅在上游 admin 的导入框里有 `accountBackendId` 语义；vendor 版会把它当 options）：

```
# RemoteStorage 数据组后端，复用名为 gh-weijia 的配置同步组账号
RemoteStorage:rs-data:href=weijia@5apps.com,token=rs_xxxx,basePath=/my-data/,desc=my data,accountBackendId=gh-weijia
```

> 数据同步组的「复制配置字符串」与「从字符串导入」入口在
> `zen-fs-config-admin` 的 `DataGroupsPage`（组内后端列表的 ⧉ 按钮、加后端 / 建组弹窗的导入框），
> 与配置同步组 `BackendsPage` 的能力一致；底层都是 `backend-config-string.ts` 的
> `serializeBackend` / `deserializeBackend`（已兼容 `AppDataBackendDescriptor`）。
