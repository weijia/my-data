# 文件列表（File List）数据格式

## 用途

把「用户显式选中的目录下的文件元数据」序列化成轻量 JSON，供查询、筛选、排序或别的程序消费。
与[目录结构清单](./dir-struct-manifest-format.md)（以下简称 `struct/`）并列，但两者定位不同：

| | `struct/` 清单 | `file-list/` 文件列表 |
|---|---|---|
| 描述对象 | **目录树**（仅文件夹） | **文件**（扁平列表） |
| 内容来源 | 用户勾选的目录 + 自动标签 + 体积汇总 | 用户勾选的目录下的文件元数据 |
| 写入语义 | 扫描分片累积、`children` 取并集 | **当前快照覆盖写**，移除必须传播 |
| 读取侧 | FluentOS 文件管理器已实现 | 目前无读取侧，契约由本文档定义 |

> 文件**绝不**作为 `type:"file"` 节点写进 `struct/`：格式文档第 94 行规定清单仅含文件夹，
> onenav `normalize.ts` 会过滤掉 `type:"file"` 节点。本格式用独立目录承载，因此不改动既有约定。

## 存储与传输

基目录 `<base>/` 默认 `app_data/folders/`（与 `struct/` 一致，由数据同步组后端的 `basePath` 推导）：

```
<base>/
  struct/          主机索引 + 卷分片      ← 不改动
  folder-deepen/   深挖标记                ← 不改动
  file-tags/       用户标签                ← 只读，用于给列表补 tags
  file-list/       ★ 本格式
```

`file-list/` 与三者并列。onenav `fetch.ts` 目前只消费 `struct/`、`folder-deepen/`、`file-tags/`，
**不会读 `file-list/`**，因此新增文件对现有界面零影响。

## 目录布局：原始目录层级镜像

**一个用户选中的目录 = 一个远端目录，里面放一份 `files.json`。** `volumeId` 作根段，
其后直接是本机路径的目录层级，**没有 `<host>` 层**。

```
<base>/file-list/
  <volumeId>/
    C/Photos/2024/files.json          ← 选中 C:\Photos\2024
    C/Photos/2024/RAW/files.json     ← 同时选中子目录，天然共存
    D/工作/报表/files.json           ← 中文目录名原样保留
    _picked.json                     ← 手工「+ 列表」加入的散文件 / zip 内文件
    <dirKey>.json                   ← 路径超限时的哈希回退片
```

- `<volumeId>` 与 `file-tags/<key>.json`、`folder-deepen/<key>.json` **同一命名空间**：
  驱动层 `volumeId`（有 `.dir-struct-id` 时即 UUID，否则卷序列号），取不到时回退 `主机::盘符`。
  该段必须过 `sanitizeKey`（`[^A-Za-z0-9._-]` → `_`），因为回退值含 `::` 与 `:`。
- **不加 `<host>` 层**：`volumeId` 的语义就是「同一块物理盘跨主机一致」（格式文档 v2 §磁盘稳定身份），
  同一卷同一目录本应只有一份当前快照。v2 的卷分片文件名 `<volumeKey>.<createdTime>.json` 同样不含主机名，
  主机信息放在片内 `part.id` / `sources[]`。主机信息在本格式里由片内 `source` / `part.id` 承载。
- 盘符段 `C:` → `C`（去掉冒号，冒号在 URL / 远端路径里不合法）。片内 `dir` 保留完整
  host-relative 路径（如 `C:/Photos/2024`），**读取侧以 `dir` 为权威字段，不从文件路径反推**。
- `_picked.json` 位于卷根，与单字母盘符段并列，不会撞。

### 逐段安全化

不能用 `sanitizeKey` 压扁整条路径（中文会变成 `_`）。逐段处理：

1. 本机绝对路径 → host-relative：盘符大写、`\` → `/`、去掉尾部斜杠，保留原始大小写。
2. 首段盘符去掉冒号（`C:` → `C`）。
3. 其余段**保留 Unicode**（含中文、空格），仅替换真正不合法字符：
   `/ \ : * ? " < > |`、控制字符（U+0000–U+001F）、段首/段尾的空格与点；
   `.` 与 `..` 段直接丢弃（防越界）；空段丢弃。
4. 若替换后段为空 → 该目录走哈希回退片。
5. 选择集按 `rel.toLowerCase()` 去重（Windows 大小写不敏感，远端 Git / RemoteStorage 敏感）；
   路径保留首次出现的原始大小写。

### 回退条件

以下情况不镜像路径，改写卷根下的哈希片 `file-list/<volumeId>/<dirKey>.json`：

- 路径段数超过 **24** 层，或拼出的远端路径总长超过 **512** 字符；
- 任一段安全化后为空、或含代理项/不可编码字符；
- zip 内文件（伪路径 `zip:C:/a.zip#inner/x.png` 含 `#` 与 `:`）——统一归入 `_picked.json`。

`dirKey = sanitizeKey(basename) + '-' + fnv1a64Hex(rel.toLowerCase()).slice(0, 12) + '.json'`，
哈希对 `TextEncoder` 的 UTF-8 字节计算（同步、Node/浏览器结果一致）。片内仍写完整 `dir`。

## 每片 JSON 结构（v1）

```json
{
  "version": 1,
  "generator": "zip-browser",
  "source": "Richard",
  "volumeId": "A401-6B8F",
  "letter": "C:",
  "dir": "C:/Photos/2024",
  "recursive": true,
  "part": { "id": "Richard", "revision": 3, "generatedAt": 1763110000000 },
  "generatedAt": 1763110000000,
  "incomplete": false,
  "truncated": false,
  "fileCount": 2,
  "copyRefs": { "files": 1, "refs": 1 },
  "files": [
    {
      "path": "C:/Photos/2024/IMG_001.jpg",
      "name": "IMG_001.jpg",
      "size": 2048,
      "sizeOnDisk": 4096,
      "mtime": 1761800000000,
      "tags": ["旅行"],
      "archive": false,
      "copies": [
        { "id": "cp_8f3a", "kind": "url", "value": "https://.../IMG_001.jpg",
          "label": "图床", "note": "待上传", "addedAt": 1763110000000 }
      ]
    },
    {
      "path": "zip:C:/a.zip#inner/x.png",
      "name": "x.png",
      "size": 1024,
      "sizeOnDisk": 1024,
      "mtime": 1761800000001,
      "tags": [],
      "archive": true
    }
  ]
}
```

| 字段 | 说明 |
|---|---|
| `version` | 固定 `1` |
| `source` / `part.id` | 主机名（`os.hostname()`），路径里没有 host 层，来源信息必须进片内 |
| `volumeId` / `letter` | 卷稳定身份与盘符 |
| `dir` | 选中目录的 host-relative 路径，与 `file-tags` 的 key 同口径，可直接 join |
| `recursive` | `true` 递归含子孙目录；`false` 仅本层直接文件。切换该选项仍是**同一片覆盖写** |
| `part.revision` | 该片本地自增版本号，每次成功写入 +1；跨主机各自维护，不保证连续 |
| `incomplete` | 有目录读不到（权限 / 超时 / 符号链接环 / 深度上限）时为 `true`；**读不到的分支跳过，绝不记成 0** |
| `truncated` | 单目录超过 `maxFiles`（默认 20000）时截断并置 `true` |
| `files[].path` | host-relative；zip 内文件用 `zip:<zip 路径>#<包内路径>` 伪路径 + `archive:true` |
| `files[].sizeOnDisk` | 占用空间；Windows 下 Node 不提供 `st_blocks`，按逻辑大小 `size` 替代 |
| `files[].tags` | 从 `file-tags/<key>.json` 只读合并，不回写 |
| `files[].mtime` | 毫秒 epoch |
| `files[].copies` | 可选：**预留的网络副本引用**，见下节；无引用时不写该字段 |
| `copyRefs` | 片级统计 `{files, refs}`：本片有多少个文件带引用、共几条引用 |

`files` 按 `path` 去重（同一文件被多个选中目录覆盖时只留一条）、按 `path` 字典序排序。
只写元数据，**不读文件内容、不上传文件本体**。

## 网络副本引用（仅预留，不创建副本）

`copies` 里存的是用户为**单个文件**手工登记的 Web 链接或远端服务路径。它表达「将来/别处可能有一份网络副本」，
**不表示副本已存在、不表示内容已上传、不表示链接可达**。生成器不会 `fetch`、不会校验、不会上传文件内容，
读取侧也不得据此字段发起下载。

```json
"copies": [
  { "id": "cp_8f3a", "kind": "url",  "value": "https://.../a.jpg",
    "label": "图床", "note": "待上传", "addedAt": 1763110000000 },
  { "id": "cp_91bc", "kind": "path", "value": "app_data/drive/2024/a.jpg",
    "label": "WebDAV", "note": "", "addedAt": 1763110000001 }
]
```

| 字段 | 说明 |
|---|---|
| `id` | 引用稳定 ID，用于编辑 / 删除；同一文件内唯一 |
| `kind` | `url`（Web 链接）/ `path`（远端服务路径、UNC 等）/ `other`；仅提示，不强制协议 |
| `value` | 链接或路径本体。允许 `http(s)://`、`app_data/…`、`dav://…`、`\\nas\share\…`；拒绝 `javascript:` / `data:` / `vbscript:` / `file:` 与本机盘符绝对路径 `C:\…`（避免把本地路径误当远端、避免泄露本机路径）。单条 ≤ 2048 字符 |
| `label` / `note` | 可选的备注标签与说明 |
| `addedAt` | 首次登记时间（毫秒 epoch），编辑时保留原值 |

**事实源不在 `file-list/` 片里**：片是导出产物、手动扫描会全量重建覆盖。引用持久化在 zen-fs-config
配置仓 `/file-copy-refs`（结构 `{v:1, refs:{"C:/Photos/a.jpg":[…]}}`，localStorage 兜底），按
host-relative 路径索引、大小写不敏感；生成片时才按 `path` 只读合并进 `files[].copies`，
与 `tags` 从 `file-tags` 合并完全同构。每文件最多 20 条引用。

文件被删除 / 改名后引用仍留在配置仓，但因为引用挂在 `files[]` 上，**该片里不再出现**；
面板标注「源文件已不存在」并允许清理，不自动生成孤儿占位片。

### 幂等与轻量回写

`fileListSignature` 除 `path + size + mtime` 外，还包含规范化后的 `copies`
（`id/kind/value/label/note/addedAt`），否则「改了引用」会被误判为内容未变而跳过写入。

引用变更走独立的 **C 通道**：按已生成片列表读取远端现有片 → 按 `path` 合并 / 删除 `copies` →
覆盖写，**零扫盘、不重建 `struct/`、不写主机索引**。片不存在（还没生成过）时跳过，
等下次手动扫描生成时自然带上。同步组未连接时引用仍保存在本机配置仓，仅提示远端未更新。

> B 通道（手动扫描）与 C 通道必须调用同一个 `applyCopyRefs()`，保证两边写出的结构逐字节一致，
> 否则下一次扫描会因签名不同而重复写入。

## 写入语义

- **固定名覆盖写**：每次生成全量重建该片；内容由当前选择集决定，移除文件 / 取消勾选会传播。
  这与 `struct/` 的「历次选择累积 + `children` 并集」刻意相反 —— 文件列表不能并集合并，
  否则已删除的文件会从旧快照里复活。
- **幂等**：以 `path + size + mtime` 的有序集合算签名；与远端该片签名相同则零写入。
- **取消选择 / 目录改名**：优先 `unlink` 对应片；后端不支持删除时写
  `{ version:1, removed:true, files:[] }` 空片，读取侧忽略 `removed:true`。
  目录改名会产生新路径、旧片成为孤儿，由生成器提供「清理孤儿片」手动入口，**不自动删**。
- **跨主机**：两台机写同一卷同一路径时 last-write-wins；`source` / `generatedAt` 用于诊断。
  若将来需要保留多机各自快照，应通过片内 `sources[]` + 读取侧按 `generatedAt` 裁决，**不得**在路径里加 `<host>` 层。

## 选择集

「哪些目录进文件列表」完全由用户勾选决定，服务端不自动添加目录。选择集存 zen-fs-config：

```
/file-list-selection = { v:1, dirs:[{path, recursive}], files:[path, ...] }
```

与目录清单的 `/manifest-selection` 相互独立（可能只想传某目录的文件列表，不想把它加进 `struct` 树）。
网络副本引用另有独立键 `/file-copy-refs`，与两套选择集并列，刷新页面可恢复。
一次物理扫描同时喂给 `struct/` 与 `file-list/`，不重复扫盘。

## 读取侧约定（尚未实现）

1. 枚举 `file-list/<volumeId>/` 下的目录层级与卷根的哈希片；
2. 按片内 `dir` 过滤（不从路径反推），忽略 `removed:true` 的空片；
3. 合并时按 `files[].path` 去重，按 `path` 排序；
4. `incomplete` / `truncated` 任一片为真时，合并结果同样标记；
5. `tags` 从 `file-tags/` 覆盖式读取，不使用片内 `tags` 作为事实源；
6. `copies` 同理只是派生视图：事实源是配置仓 `/file-copy-refs`，**不得当作可下载内容或已存在的副本**；
   读取侧若要展示，只做文本/链接呈现，不发起可达性探测。

## 安全边界

- 所有本机 API 沿用 `X-Local-Token` + `normalizeLocalPath` / `isPathAllowed`；
  递归下钻带深度上限、超时与符号链接环检测，只从「用户显式勾选且已浏览过的根」下钻。
- 只 `stat` 元数据，不读文件内容；不写盘根、不创建 `.dir-struct-id`。
- 卷身份只读：优先读已有 `.dir-struct-id`，回退卷序列号，再回退 `主机::盘符`。
- 网络副本是**纯字符串**：不读文件内容、不上传、不 `fetch`、不校验链接；面板打开链接仅用
  `target="_blank" rel="noopener noreferrer"`，且只对 `http(s)://` 渲染可点击 `<a>`。
  引用会写进用户配置的网络数据组，UI 需提示一次「内容会离开本机」。
