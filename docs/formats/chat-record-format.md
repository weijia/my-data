# 聊天记录保存格式（home_cloud）

本文档定义由 `home_cloud` 项目的 `scripts/download_chat_media.py` 生成的**聊天记录离线存档**的数据格式。该脚本将聊天室的文本、图片、视频消息拉取下来，按年月组织目录，媒体文件落盘为本地文件、HTML 使用相对路径引用，并借助状态文件实现增量下载。

> **源码参考**（home_cloud）：
> - 脚本：`scripts/download_chat_media.py`
> - 设计文档：`scripts/download_chat_media_design.md`
> - 复用配置：`scripts/chat_download_config.json`（与旧版 `download_chat_v2.py` 通用）

---

## 1. 概述

- **目标**：将聊天记录（文本 + 图片 + 视频）离线保存为可本地浏览的 HTML，媒体文件独立存储，便于增量更新与单独管理。
- **数据来源**：Supabase `messages` 表（经邀请码定位 `chat_rooms`），按 `created_at` 分页拉取。
- **存储方式**：本地文件目录（默认 `./chat_exports`，集成到 my-data 时置于 `app_data/chat-exports/`）。
- **增量机制**：通过 `download_state.json`（根目录，跨月）记录上次下载时间/消息 ID；媒体索引 `media_index.json` **按月拆分**存放于各月度目录，记录当月媒体 URL→本地路径映射，避免单文件随年份无限膨胀。
- **非目标**：不做并发下载、不做 Hash 完整性校验（见设计文档 §1.3）。

> **实现状态**：本文档为 my-data 侧的**目标格式规范**。当前 `home_cloud` 脚本（`scripts/download_chat_media.py`）仍按「根目录单份 `media_index.json`」实现；落地「按月拆分索引」需同步调整其 `StateManager` 的 `load_media_index` / `save_media_index`（改为按月份读写），`local_path` 由相对存档根目录改为相对月度目录。

---

## 2. 存储路径

| 层级 | 载体 | 路径 / 命名 | 说明 |
|------|------|-------------|------|
| 存档根目录 | 文件目录 | `app_data/chat-exports/`（或脚本 `output_dir` 指定） | 默认 `./chat_exports`；集成 my-data 时遵循 `app_data/{项目名}/` 约定 |
| 月度目录 | 子目录 | `{YYYY-MM}/` | 如 `2024-01/`，按月切分 |
| 图片目录 | 子目录 | `{YYYY-MM}/images/` | 当月图片存储 |
| 视频目录 | 子目录 | `{YYYY-MM}/videos/` | 当月视频存储 |
| 月度 HTML | 文件 | `{YYYY-MM}/chat_{YYYY-MM}.html` | 当月聊天记录浏览页 |
| 下载状态 | 文件 | `download_state.json` | 进度与游标（**存档根目录**，跨月，体积小） |
| 媒体索引 | 文件 | `{YYYY-MM}/media_index.json` | 当月媒体 URL→本地路径映射（**按月拆分**，避免单文件过大） |

---

## 3. 目录结构

```
chat_exports/
├── 2024-01/
│   ├── images/
│   │   ├── img_001_a1b2c3d.jpg
│   │   ├── img_002_e4f5g6h.png
│   │   └── ...
│   ├── videos/
│   │   ├── video_001_x1y2z3.mp4
│   │   └── ...
│   ├── media_index.json          # 当月媒体索引（URL→本地路径映射）
│   └── chat_2024-01.html
├── 2024-02/
│   ├── images/
│   ├── videos/
│   ├── media_index.json
│   └── chat_2024-02.html
├── ...
└── download_state.json            # 下载进度/游标（跨月，留在根目录）
```

---

## 4. 文件命名规则

| 类型 | 命名格式 | 示例 |
|------|----------|------|
| 图片 | `img_{序号}_{消息ID前8位}.{扩展名}` | `img_001_a1b2c3d.jpg` |
| 视频 | `video_{序号}_{消息ID前8位}.{扩展名}` | `video_001_x1y2z3.mp4` |
| HTML | `chat_{年}-{月}.html` | `chat_2024-01.html` |

- **序号**：当月从 `1` 起自增，三位零填充（`001`、`002`…）。
- **消息 ID 前 8 位**：取自消息 `id` 的前 8 个字符；缺失时回退为 `unknown`。
- **扩展名**：优先取 URL 路径扩展名（限 `jpg/jpeg/png/gif/webp/mp4/mov/avi`），否则按 `Content-Type` 推断，仍无法确定时默认 `jpg`。

---

## 5. 状态跟踪

### 5.1 download_state.json

记录下载进度与游标，结构对应 `DownloadState` dataclass。

```json
{
  "room_id": "xxx",
  "room_name": "聊天室名称",
  "last_download_time": "2024-03-15T10:30:00Z",
  "last_message_id": "msg_uuid_xxx",
  "completed_months": ["2024-01", "2024-02"],
  "total_messages": 1500,
  "total_images": 320,
  "total_videos": 15,
  "last_run": "2024-03-15T12:00:00Z",
  "status": "completed",
  "created_at": "2024-03-10T08:00:00.123456",
  "updated_at": "2024-03-15T12:00:00.123456"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `room_id` | `string` | 是 | 聊天室 ID（Supabase `chat_rooms.id`） |
| `room_name` | `string` | 是 | 聊天室名称 |
| `last_download_time` | `string` | 否 | 上次下载到的消息时间（ISO 8601，`Z` 表示 UTC）；增量下载游标 |
| `last_message_id` | `string` | 否 | 上次下载到的最后一条消息 ID |
| `completed_months` | `array<string>` | 是 | 已处理月份的 `YYYY-MM` 列表 |
| `total_messages` | `number` | 是 | 累计消息数（增量累加） |
| `total_images` | `number` | 是 | 累计新增图片数（仅本次真正下载的） |
| `total_videos` | `number` | 是 | 累计新增视频数（仅本次真正下载的） |
| `last_run` | `string` | 否 | 最后一次运行时间 |
| `status` | `string` | 是 | 运行状态：`initialized` / `downloading` / `completed` |
| `created_at` | `string` | 是 | 状态文件创建时间 |
| `updated_at` | `string` | 是 | 状态文件最后更新时间 |

> 时间格式统一：`last_download_time` 内部规整为 `+00:00` → `Z`（见脚本 `_normalize_time_format`）。

### 5.2 {YYYY-MM}/media_index.json

媒体文件索引，避免**当月**重复下载，按月份拆分存储，结构对应 `MediaIndex` dataclass。每个月度目录各持有一份 `media_index.json`，避免随年份累积导致单文件过大。

```json
{
  "media_files": {
    "https://storage.supabase.co/xxx/abc.jpg": {
      "local_path": "images/img_001_a1b2c3.jpg",
      "message_id": "a1b2c3d...",
      "downloaded_at": "2024-03-10T10:00:00.123456",
      "file_size": 102400
    },
    "https://storage.supabase.co/xxx/video.mp4": {
      "local_path": "videos/video_001_x1y2z3.mp4",
      "message_id": "x1y2z3...",
      "downloaded_at": "2024-03-10T11:00:00.123456",
      "file_size": 5242880
    }
  },
  "created_at": "2024-03-10T08:00:00.123456",
  "updated_at": "2024-03-15T12:00:00.123456"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `media_files` | `object` | 是 | 键为媒体原始 URL，值为该媒体的元数据 |
| `media_files.{url}.local_path` | `string` | 是 | 相对**月度目录**的本地路径（不含 `YYYY-MM/` 前缀），如 `images/img_001_a1b2c3.jpg` |
| `media_files.{url}.message_id` | `string` | 是 | 关联的消息 ID |
| `media_files.{url}.downloaded_at` | `string` | 是 | 下载时间（`datetime.now().isoformat()`） |
| `media_files.{url}.file_size` | `number` | 是 | 文件字节大小 |
| `created_at` | `string` | 是 | 索引创建时间 |
| `updated_at` | `string` | 是 | 索引最后更新时间 |

**重复下载判定（当月）**：下载前先查 `media_url` 是否在本月 `media_index.json` 的 `media_files` 中（命中则直接用 `local_path`）；同时若本地文件已存在也跳过。

> **取舍说明（按月拆分索引）**：索引拆分后，去重范围从「全量」缩小为「当月」。同一媒体 URL 若跨月份再次出现（如同一张图在 1 月和 3 月都发过），会各自下载到对应月份目录、不再全局去重。这是为「单文件体积有界」做的取舍——跨月重复媒体在实践中很少见，且即便重复下载，单条媒体的磁盘代价远小于一个无限膨胀的索引文件。如需保留跨月去重，可在下载时选择性读入其它月份的索引（仅用于查重，不合并写入）。

---

## 6. HTML 相对路径引用

HTML 存放在月度目录下，与 `images/`、`videos/` 同层，因此用相对路径引用媒体：

```html
<!-- 图片 -->
<img src="images/img_001_a1b2c3.jpg" class="media-image"
     onerror="this.onerror=null;this.src='https://storage.supabase.co/xxx/abc.jpg'">

<!-- 视频 -->
<video controls class="media-video"
       onerror="this.onerror=null;this.src='https://storage.supabase.co/xxx/video.mp4'">
  <source src="videos/video_001_x1y2z3.mp4" type="video/mp4">
  您的浏览器不支持视频播放
</video>
```

- 默认使用本地相对路径；`onerror` 兜底回退到服务器原始 URL（离线打开时若本地文件缺失仍可显示）。
- 每条消息渲染为 `.message` 卡片：`.message-header`（发送者 + 时间戳）、`.message-content`（文本/媒体 + 可选 `caption`）。
- 已删除消息（`deleted_at` 非空）在生成时被跳过。
- 时间戳统一转换为 `YYYY-MM-DD HH:MM:SS`（本地时区）展示。

---

## 7. 配置文件

复用 `scripts/chat_download_config.json`（JSON 文件形式）：

```json
{
  "supabase_url": "https://xxx.supabase.co",
  "supabase_key": "...",
  "publishable_key": "...",
  "jwt_secret": "...",
  "invite_code": "...",
  "output_dir": "./chat_exports"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `supabase_url` | `string` | 是 | Supabase 项目地址 |
| `supabase_key` | `string` | 是 | Supabase anon/service key |
| `publishable_key` | `string` | 否 | 公开发布 key（默认同 `supabase_key`） |
| `jwt_secret` | `string` | 否 | 用于本地签发 JWT（HS256），提供时走 `Authorization: Bearer <jwt>` |
| `invite_code` | `string` | 否 | 聊天室邀请码，用于定位 `room_id` / `room_name` |
| `output_dir` | `string` | 否 | 输出根目录，默认 `./chat_exports`；集成 my-data 建议 `app_data/chat-exports` |

---

## 8. 命令行用法

```bash
python download_chat_media.py              # 增量下载（默认，读 last_download_time 之后）
python download_chat_media.py --full       # 全量下载
python download_chat_media.py --year 2024 --month 1   # 指定月份
python download_chat_media.py --start 2024-01-01 --end 2024-03-31  # 时间范围
python download_chat_media.py --force-media # 强制重新下载媒体
python download_chat_media.py --status     # 查看下载状态
```

---

## 9. 数据来源（源码）

| 模块 | 函数 | 生成内容 |
|------|------|----------|
| `download_chat_media.py` → `StateManager` | `load_state()` / `save_state()`（根目录）/ `load_media_index(month)` / `save_media_index(month)` | `download_state.json`（根目录）、各月度目录下的 `media_index.json` |
| `download_chat_media.py` → `DirectoryManager` | `ensure_month_directory()` / `get_html_path()` / `get_relative_media_path()` | 月度目录与相对路径 |
| `download_chat_media.py` → `MediaDownloader` | `download_media()` / `get_file_extension()` / `generate_filename()` | 媒体落盘与命名 |
| `download_chat_media.py` → `HTMLGenerator` | `generate_html()` / `_generate_message_content()` | 月度 HTML 生成 |
| `download_chat_media.py` → `ChatAPIClient` | `get_room_by_invite_code()` / `get_all_messages()` | Supabase 拉取聊天室与消息 |
