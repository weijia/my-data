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
        "children": [
          {
            "name": "Projects",
            "type": "folder",
            "tags": ["git", "node"],
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

> **仅含文件夹**：清单描述「目录结构」，**不含文件**（文件内容/列表不在范围内）。读取侧会过滤掉 `type:"file"` 的节点，因此即使源数据包含文件节点也不会被展示。

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
- **读取侧**：`FluentOS-On-Web/js/apps/files.js`、`FluentOS-On-Web/js/core/remotestorage-client.js`
- **设计文档**：`FluentOS-On-Web/remotestorage-fs-plan.md`
