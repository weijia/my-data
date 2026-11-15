# 微信 4.x 聊天记录导出工具

统一入口：`C:\codes\my-data\tools\wechat_export.py`（默认只做 dry-run，加 `-Commit` 才落盘）。

## 前置条件

- 微信 4.1.15.13 处于登录/运行状态；工具只读 `db_storage` 下的库文件（含 `-wal`），不写原库、不结束微信进程。
- 密钥文件：`C:\codes\my-data\app_data\wechat-decrypted\moreconfident_5365\_keys.json`
  - 内含 18 个库的 raw key + salt + sha256 前 8 位。**该文件是当前版本唯一可用的密钥来源，务必异地备份**。
  - 若某库在 `_keys.json` 中缺失，工具会明确报出库名并停止/跳过，不会静默。
  - 若某库文件被微信重建（salt 变化），该库旧 key 失效，需要重新取 key；导出的核心 `message_*` 库不受影响时可继续。
- 图片密钥：`C:\codes\my-data\tools\chatlog\imgkeyscan\_imgkey_hits.json`（已存、已验证）。缺失时图片只会降级为文本占位。
- 依赖（均已就位，无需安装）：ffmpeg/ffprobe、Node 22、`silkwasm` 与 `silk24.mjs`、zstd sidecar、`imgkeyscan.exe`。

## 常用命令

```powershell
# 1) 看有哪些联系人可导出（含消息数与时间范围）
python C:\codes\my-data\tools\wechat_export.py -List

# 2) dry-run：只统计，不写文件
python C:\codes\my-data\tools\wechat_export.py -Alias mdd19910420

# 3) 正式导出（默认输出 chat-exports\mdd19910420）
python C:\codes\my-data\tools\wechat_export.py -Alias mdd19910420 -Commit

# 4) 限定月份 + 自定义输出目录
python C:\codes\my-data\tools\wechat_export.py -Alias mdd19910420 -Start 2026-07 -End 2026-09 -Out D:\chat-out -Commit

# 5) 按显示名查找（精确匹配备注/昵称/alias/username）
python C:\codes\my-data\tools\wechat_export.py -Name "我不说你不懂这就是距离" -Commit

# 6) 增量：只处理 2026-11-01 所在月及之后（默认原地更新当月 HTML，旧版自动备份）
python C:\codes\my-data\tools\wechat_export.py -Alias shen_yunxia -Since 2026-11-01 -Commit

# 7) 增量：只处理最近 30 天所在月及之后
python C:\codes\my-data\tools\wechat_export.py -Alias shen_yunxia -Recent 30 -Commit

# 8) 严格不覆盖：改动只写 chat_YYYY-MM.new.html，等人工确认
python C:\codes\my-data\tools\wechat_export.py -Alias shen_yunxia -Since 2026-11-01 -NoUpdate -Commit
```

参数：

| 参数 | 说明 |
|------|------|
| `-Alias` | 联系人微信号/别名，精确匹配 `username` 或 `alias` |
| `-Name` | 显示名，精确匹配 `remark` / `nick_name` / `alias` / `username`；与 `-Alias` 二选一 |
| `-Start` / `-End` | 月份范围 `YYYY-MM`（含端点） |
| `-Since` | 增量起点 `YYYY-MM-DD`（也接受 `YYYY-MM`）；只处理起点所在月及之后的月份 |
| `-Recent` | 增量起点为最近 N 天所在月；与 `-Since` 二选一，且不能与 `-Start` 同用 |
| `-NoUpdate` | 严格不覆盖已有 HTML：有差异时写 `chat_YYYY-MM.new.html`，不动原文件 |
| `-Out` | 输出目录；默认 `C:\codes\my-data\app_data\chat-exports\<Alias>` |
| `-List` | 列出有消息的会话及消息数（含群聊；同一会话分片在多个库时跨库汇总） |
| `-Commit` | 正式落盘；不加则只统计 |
| `-Keys` / `-DbDir` / `-Self` / `-Version` | 覆盖默认密钥、库目录、本人 wxid、页面版本号 |

## 增量模式

- 微信 4.x 会把同一张 `Msg_<md5>` 表按时间拆到 `message_0/1/2.db`。工具会自动找出**所有**含该表的分片库，按 `server_id`（缺失时用 `local_id+create_time`）去重、按 `created_at` 合并排序，不再只读最后一个匹配库。
- 增量按**整月**重建：`-Since 2026-11-01` 会重建 2026-11；`-Since 2026-10-15` 会重建 2026-10 与 2026-11。原因是 HTML 以月为页，只追加单条消息无法保证页内顺序与计数正确。
- 增量只解压/转码窗口内的消息；图片还原只扫描受影响月份（额外带上一个月，覆盖跨月 `.dat`）。
- 新消息按 `created_at` 插入该月正确位置；月内媒体序号从 1 连续重排，因此结果与全量导出一致。
- 受影响月份的 `media_index` 条目整月重建；不再被引用的旧媒体（序号漂移或降级产生的孤儿）会自动删除。
- 幂等：同一窗口重复执行不会重复消息、不会重复复制媒体、不会生成多余备份。
- 注意：若某会话的 `Msg_<md5>` 表已被微信轮转清理出当前库（如 `mdd19910420`），增量无法读取新消息；已有存档不受影响，但需等表重新出现或改用历史快照。

## HTML 更新策略

| 场景 | 默认行为 | 加 `-NoUpdate` |
|------|----------|----------------|
| 内容与现有 HTML 相同 | `unchanged`，不动文件 | 同左 |
| 内容有差异且已有 HTML | 先备份 `chat_YYYY-MM.bak.html`，再原地覆盖，状态 `updated` | 写 `chat_YYYY-MM.new.html`，原 HTML 不动 |
| 该月尚无 HTML | 直接写 `chat_YYYY-MM.html`，状态 `written` | 同左 |
| 内容有差异且已有 `.new.html` | 覆盖后删除该 `.new.html` | 覆盖 `.new.html`，状态 `rewritten_new` |

- 备份命名固定为 `chat_YYYY-MM.bak.html`，每月只保留一份；内容不变时不会产生新备份。

## 输出与规则

- 严格遵循 `C:\codes\my-data\docs\formats\chat-record-format.md`：按月目录、`images/`、`videos/`、`audios/`、`originals/`、`media_index.json`、`download_state.json`。
- 图片：`.dat` 用已存 img key 还原；`wxgf` 主图在导出期用 ffmpeg 转 JPEG（`-frames:v 1 -update 1 -q:v 2`），缩略图另存；少数主图 `.dat` 本身即 JPEG，直接作为主图使用；转码失败/主图缺失不用缩略图冒充，降级为 `[图片]` 并列出原因。
- 语音：SILK → 24 kHz / 16-bit / 单声道 WAV；`.silk` 归档到 `originals/audios/`。
- 视频：复制 mp4 与封面；mp4 缺失时降级为 `[视频]` 文本并在汇总中列出。
- 幂等：已存在媒体不重复复制、不覆盖；HTML 内容不变标记 `unchanged`，变化默认原地更新并保留一份 `.bak.html`（`-NoUpdate` 时写 `.new.html`）。
- `download_state.json` 的媒体计数按最终 `media_index.json` 统计，增量后不会漏算已存在媒体；`total_messages` 为会话总消息数（增量时为含起点之前月份的全量口径）。
- 运行结束自动清理临时解密目录，不留中间产物。

## 失败时的表现

- 联系人 0 条或多条：直接报错退出，不猜测。
- `Msg_<md5>` 落在缺密钥的库：报出库名与缺失项并停止。
- 单个图片/语音/视频转码失败：该条降级为文本占位，其余照常导出，失败项在汇总里逐条列出。
