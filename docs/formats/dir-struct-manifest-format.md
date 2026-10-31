# 目录结构清单（Directory-Structure Manifest）数据格式

本文档定义了由 [FluentOS-On-Web](https://github.com/weijia/FluentOS-On-Web) 项目中的 **`dir-struct-gen`（目录结构清单生成器）** 生成的「本地目录结构清单」JSON 格式，以及它被 FluentOS 文件管理器读取侧消费的方式。

> 定位：该清单是 RemoteStorage 方案中的「**形态 B**（目录结构清单）」——只描述本地真实目录树（**仅文件夹，不含文件内容**），无需在 RemoteStorage 中真正建立所有文件夹。对应设计文档：`FluentOS-On-Web/remotestorage-fs-plan.md`。

## 用途

把一台电脑上「勾选过的真实目录树」序列化为一份轻量 JSON，便于：

- 在 FluentOS 文件管理器里以**只读**方式挂载、浏览这台电脑的目录结构（无需暴露真实文件内容）；
- 作为元数据被**任何程序**读取（格式是纯 JSON，按 `root.children` 递归即可还原整棵目录树）；
- 多台电脑各自生成、互不混淆（靠 `source` / `root.name` = 主机名命名空间隔离）。

## 存储与传输

格式本身是**独立的纯 JSON**，可存储在任何地方。默认通过 RemoteStorage 协议传输：

| 项目 | 说明 |
|------|------|
| **传输协议** | RemoteStorage（HTTP `GET` / `PUT`，`Authorization: Bearer <token>`） |
| **默认存储目录** | `/app_data/folders/`（生成器「基础路径」字段 `rsBase` 默认值） |
| **默认文件名** | 本机主机名（`os.hostname()`），如 `LAPTOP-91PR81Q2` |
| **完整上传目标** | `<服务器地址>/app_data/folders/<清单文件名>` |
| **鉴权** | `Bearer` Token（与 RemoteStorage 连接共用） |

> **文件名约定（重要）**：生成器默认把清单文件名设为「主机名」且**不带 `.json` 后缀**；而 FluentOS 文件管理器读取侧的「**清单文件留空 → 自动扫描 `basePath` 下所有 `.json`**」逻辑只识别 `.json` 文件。因此，若希望读取侧能自动发现，请把生成器的「清单文件名」显式写成 `<主机名>.json`（带后缀）；或在读取侧「清单文件」字段里手动填 `<主机名>`（无后缀，与上传名严格一致）。
>
> 生成器也支持**下载 / 复制**清单到本地任意位置（不依赖 RemoteStorage），此时建议同样保存为 `.json` 以便通用程序解析。

## 清单 JSON 结构

```json
{
  "version": 1,
  "generatedAt": "2026-10-24T12:00:00.000Z",
  "generator": "fluentos-dir-struct-gen",
  "source": "LAPTOP-91PR81Q2",
  "root": {
    "name": "LAPTOP-91PR81Q2",
    "type": "folder",
    "children": [
      {
        "name": "C:",
        "type": "folder",
        "updatedAt": 1793351520000,
        "children": [
          {
            "name": "Projects",
            "type": "folder",
            "tags": ["git", "node"],
            "updatedAt": 1793351520000,
            "children": []
          }
        ]
      }
    ]
  }
}
```

### 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `version` | `number` | 是 | 清单 schema 版本，当前固定为 `1` |
| `generatedAt` | `string` | 是 | 生成时间，ISO 8601（`YYYY-MM-DDTHH:mm:ss.sssZ`） |
| `generator` | `string` | 是 | 生成器标识，固定为 `"fluentos-dir-struct-gen"` |
| `source` | `string` | 是 | **计算机 ID**（主机名），用于命名空间隔离多台机器 |
| `root` | `object` | 是 | 根目录节点（见下） |

### 节点对象（Folder Node）

整棵树由嵌套的「文件夹节点」组成。每个文件夹节点：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | `string` | 是 | 文件夹名称（盘符如 `C:`，或目录名） |
| `type` | `string` | 是 | 固定为 `"folder"` |
| `children` | `array` | 否 | 子文件夹节点数组；叶子文件夹为空数组 `[]` 或省略 |
| `tags` | `array<string>` | 否 | 标签数组（自动探测 + 手动标签合并去重）；无标签时省略该字段 |
| `volumeId` | `string` | 否 | 仅在驱动层节点出现：该物理盘的稳定身份（`<盘根>/.dir-struct-id` 内容），供跨主机识别与合并 |
| `leaf` | `boolean` | 否 | 仅磁盘上确实无子目录时为 `true`（读取侧据此省略展开按钮） |
| `updatedAt` | `number` | 否 | 该节点**最后一次被扫描确认存在**的时间，毫秒级 Unix epoch（UTC），如 `1793351520000`；由生成器写入，用于多份清单合并时「按时间取新」（见下文 v2 合并规则） |

> **仅含文件夹**：清单描述「目录结构」，**不含文件**（文件内容/列表不在范围内）。读取侧会过滤掉 `type:"file"` 的节点，因此即使源数据包含文件节点也不会被展示。
>
> **时间字段说明**：顶层 `generatedAt` 保持 ISO 8601 字符串（历史契约不变）；新增的节点级 `updatedAt` 用**毫秒级 Unix epoch 数字**。读取侧两种都要能解析：数字直接采信，字符串按 ISO 解析（兼容早先写入的 ISO 值）；两者都无法解析时按合并规则的兜底策略处理。文件占位节点（`type:"file"`）在开启「包含文件」时同样可携带 `updatedAt`。

### 完整示例（含标签）

```json
{
  "version": 1,
  "generatedAt": "2026-10-24T12:00:00.000Z",
  "generator": "fluentos-dir-struct-gen",
  "source": "LAPTOP-91PR81Q2",
  "root": {
    "name": "LAPTOP-91PR81Q2",
    "type": "folder",
    "children": [
      {
        "name": "C:",
        "type": "folder",
        "children": [
          {
            "name": "Projects",
            "type": "folder",
            "tags": ["git", "node"],
            "children": [
              {
                "name": "FluentOS-On-Web",
                "type": "folder",
                "tags": ["git", "node"],
                "children": []
              }
            ]
          },
          {
            "name": "Tools",
            "type": "folder",
            "tags": ["rust", "docker"],
            "children": []
          }
        ]
      },
      {
        "name": "D:",
        "type": "folder",
        "children": []
      }
    ]
  }
}
```

> 为便于阅读，上面两个示例里的节点级 `updatedAt` 只在前面的示例中示范，未逐个列出；实际生成器会为**每个** folder / file 占位节点写入该字段（毫秒级 Unix epoch）。

## 目录标签（Tags）自动探测规则

生成器的本地 Node 服务（`serve.js` 的 `TAG_RULES`）按目录内是否含有特定文件/目录自动打标签，用户也可对任意目录手动加/删标签。当前自动规则如下（命中即打对应标签）：

| 命中特征（目录内存在） | 标签 |
|------------------------|------|
| `.git` / `.gitignore` / `.gitmodules` | `git` |
| `node_modules` | `node_modules` |
| `package.json` | `node` |
| `package-lock.json` | `npm` |
| `yarn.lock` | `yarn` |
| `pnpm-lock.yaml` / `pnpm-workspace.yaml` | `pnpm` |
| `Cargo.toml` / `Cargo.lock` | `rust` |
| `requirements.txt` / `pyproject.toml` / `setup.py` | `python` |
| `.venv` / `venv` | `venv` |
| `.svn` | `svn` |
| `.hg` | `hg` |
| `.idea` | `idea` |
| `.vscode` | `vscode` |
| `Dockerfile` / `docker-compose.yml` / `docker-compose.yaml` | `docker` |
| `go.mod` / `go.sum` | `go` |
| `Gemfile` / `Gemfile.lock` | `ruby` |
| `pom.xml` | `maven` |
| `build.gradle` / `build.gradle.kts` | `gradle` |
| `Makefile` | `make` |
| `CMakeLists.txt` | `cmake` |

> 自动标签与手动标签合并后去重写入 `tags`；一个目录可同时有多个标签（如同时含 `package.json` 与 `.git` ⇒ `["git","node"]`）。

## 被读取侧消费（FluentOS 文件管理器）

FluentOS 文件管理器（[`js/apps/files.js`](https://github.com/weijia/FluentOS-On-Web) + [`js/core/remotestorage-client.js`](https://github.com/weijia/FluentOS-On-Web)）的「远程存储」挂载流程：

1. 在「设置 → 远程存储」配置连接（`href` / `token` / `basePath` / `manifestFile` / `enabled`）。
2. 文件管理器根目录出现「远程存储」节点 → 展开连接 → 出现「目录清单」容器：
   - 若 `manifestFile` 留空：自动 `GET` 列出 `basePath` 下所有 `.json`（每个 json 生成一个清单节点）；
   - 若 `manifestFile` 指定：只 `GET` 该单个文件。
3. 清单被 `fetchManifest` 读取后**校验 `root.type === "folder"`**，通过则把整棵 `root.children` 渲染为只读目录树。
4. 远程节点全部**只读**：右键仅提供「打开 / 刷新 / 属性」，禁止新建 / 重命名 / 拖入。

## 其他程序如何加载

清单是**标准 JSON**，任何语言/框架均可消费：

- **走 RemoteStorage**：用 `GET <RS根>/app_data/folders/<文件名>`（带 `Authorization: Bearer <token>`）取回 JSON，递归遍历 `root.children` 即可还原目录树（按 `type==="folder"` 判定层级、按 `tags` 取标签）。
- **走本地文件**：直接读取保存好的 `.json` 文件，同样递归 `root.children`。
- 多台机器：每个清单 `source` 不同，可分别加载后按 `source` 分组，避免混淆。

## 数据来源

- **生成器**：`FluentOS-On-Web/dir-struct-gen/`（本地 Web App，`serve.js` 提供磁盘浏览与上传 API，`app.js` 负责勾选与生成 manifest）
- **读取侧**：`FluentOS-On-Web/js/apps/files.js`、`FluentOS-On-Web/js/core/remotestorage-client.js`；以及 onenav 文件浏览器 `src/components/apps/filebrowser/`
- **设计文档**：`FluentOS-On-Web/remotestorage-fs-plan.md`

## v2：卷级拆分、多分片合并与深挖标记

> 适用：当同一块物理盘可能被多台主机挂载（或同一块盘想按来源/用途分成多份清单）时启用。v1 单文件整树格式仍被读取侧兼容（见下方「读取侧拼装」）。

### 磁盘稳定身份 `.dir-struct-id`
- 生成器的 `serve.js` 在列盘时，对每块盘的根目录确保存在一个标记文件 `<盘根>/.dir-struct-id`，内容为 UUID v4。
- 首次见到盘时生成并写入；之后所有系统读到的都是同一串字节 → **跨 OS / 跨协议（含云盘）一致**，从根本上避免「同一块盘在不同主机被当成不同盘」导致的重复。
- 该标记文件被 `listDir` 过滤，不会进入目录树 / 清单；只读盘或写入失败时回退为 `null`（此时该盘按 `主机::盘符` 命名，不跨主机去重）。

### 清单拆分成「主机索引 + 卷文件（可分多份）」
`app_data/folders/` 下不再是单一整树文件，而是：

| 文件 | 内容 |
|------|------|
| `<host>.json` | 主机索引：`{ version:2, source, generatedAt, generator, mounts:[{ letter, volumeId, file, files? }] }` |
| `<volumeKey>.<partId>.json` | 某块盘的**分片**（默认每台主机一份）：`{ volumeId, name, type:'folder', part:{id,revision,generatedAt}, children:[...], sources:[<hostId>] }` |
| `<volumeKey>.json` | 旧版单文件卷（历史数据）：结构同分片但没有 `part`；读取侧会一并纳入合并，不必手工迁移 |

- `volumeKey` = 该盘 `<盘根>/.dir-struct-id` 的内容（UUID）；取不到时回退 `主机::盘符`。用作文件名时做安全化（`[^A-Za-z0-9._-]` → `_`）。
- `partId` = 分片身份，默认本机 ID（主机名，安全化后）；`part.revision` 为该分片自增版本号（生成器本地维护，每次成功上传 +1），`part.generatedAt` 为毫秒级 epoch 的生成时间。**同一块盘因此可以有多份目录结构文件**：每台主机写自己的一份，互不覆盖。
- `mounts[].file` = 主文件（不含 `.json` 的基名）：优先旧版单文件 `<volumeKey>`，否则取该卷分片里字典序第一份——供尚未支持 `files` 的读取侧兜底。
- `mounts[].files` = 该卷**全部分片**：`[{ file, partId }]`，`partId` 为 `null` 表示旧版单文件。生成器上传前会 `GET` 列出 `base` 目录，把同前缀（`<volumeKey>.`）的分片全部收进该数组。
- 上传只 `PUT` 自己的分片，**不再「读别人的文件 → 合并 → 写回同一文件」**：多台主机同时上传不会互相覆盖（旧实现在同一文件上读改写，并发即丢更新）。
- 分片文件名**不参与合并判定**——合并只看文件内容里的 `volumeId` 与 `part`，因此重命名分片不会打断合并，也便于按用途/来源自定义分片名。

### 读取侧拼装
读取侧读取 `app_data/folders/` 全部 `.json`，按形态分类：
- 含 `mounts` 数组 → 主机索引：对每个 `mounts` 项取出该卷的**全部分片**（优先 `files`，缺失则回退 `file`），按 `volumeId` 归组后按下面的合并规则合成一棵完整树；
- 含 `volumeId` + `children`（且无 `root`/`mounts`） → 分片/卷文件，收入卷映射；也可不依赖主机索引，直接把该目录下所有卷文件按 `volumeId` 分组后合并（这样手工放进目录的文件也能被纳入）；
- 含 `root` → legacy v1 整树清单，原样透传（向后兼容）。

驱动层节点携带 `volumeId`，供深挖标记命名空间与「同盘多主机」识别。

### 分片合并规则（同一卷合成一棵树）
把同一 `volumeId` 的全部分片（含旧版单文件）合并成一棵树时，按以下规则执行，**结果幂等且确定**：

1. `children`：**并集**（递归合并同名子节点）。某台机没扫到不代表目录不存在，绝不因「另一份更新」而删掉别人的目录。
2. 同名节点的属性：按**最后确认时间取较新**——节点 `updatedAt` 优先，缺失则回退所属文件的 `part.generatedAt` / 顶层 `generatedAt`。
3. `tags` / `sources`：**并集去重**（累积信息，不被时间覆盖）。
4. `leaf`：**重算**——合并后有子目录就不再是底部节点（否则读取侧不显示展开按钮，会把深挖出来的层藏住）；两侧都没见过子目录才算底部。
5. 同名类型冲突（`folder` vs `file`）：**folder 优先**，不看时间（`file` 只是占位、不读内容）。
6. 时间缺失、或两侧时间相差不超过 **5 分钟**（多机时钟未必同步）→ 按 `part.id` 字典序取大者兜底，保证同一组输入永远得到同一棵树。
7. `volumeId`：驱动层身份字段，两侧谁有取谁，不会凭空补 `null`。

合并结果只用于读取/展示；不再写回成「一份合并文件」，需要归档时由调用方另行处理（写回时不要保留多分片归并出来的 `part`）。

### 节点级时间戳 `updatedAt`（毫秒级 Unix epoch）
- 生成器为每个节点（folder 与 file 占位）写入 `updatedAt`，语义是「该目录/文件最后一次被扫描确认存在的时间」。
- 类型为 `number`（毫秒级 Unix epoch，UTC），如 `1793351520000`；**不要写本地时间**，否则跨时区比较会出错。
- 兼容：早先写入的 ISO 字符串（如 `"2026-10-24T12:00:00.000Z"`）读取侧也要能解析——数字直接采信，字符串按 ISO 解析；两者都缺失时走合并规则第 6 条兜底。
- `epoch` 只统一了时间表示，**并不能让各机时钟变准**，所以仍需 5 分钟容差 + `part.id` 兜底来保证确定性。

> **现状提示**：FluentOS 文件管理器读取侧（`js/apps/files.js` + `js/core/remotestorage-client.js`）目前只实现了「实时目录」与「单个清单文件（校验 `root.type === "folder"`）」两条路径，**尚未实现 v2 的主机索引与分片合并**。v2 的 `mounts` / `files` / 合并语义以本文档为准，读取侧接入时按本节规则实现。

### 深挖标记 `folder-deepen`
- 浏览器里给某目录打「继续解析下一层」标记 → 写入 `app_data/folder-deepen/<key>.json`：`{ marks:[ "E:/Photos/2024", ... ] }`。
- `key` = 该目录所属驱动层的 `volumeId`（跨主机共享）；回退为 `主机::盘符`。
- `marks` 中的路径为 host-relative（盘符 + `/` 连接的嵌套目录名）。
- 生成器下次「上传」时读取这些标记，对标记目录**按需多读一层真实磁盘**并入卷文件，然后**清空（消耗）**标记文件。标记仅下次生成生效一次。

