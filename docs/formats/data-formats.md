# My-Data 数据格式规范

本文档是 my-data 仓库数据格式的总索引。

## 目录结构

```
my-data/
├── README.md                          # 项目说明
├── docs/formats/                      # 数据格式文档
│   ├── data-formats.md                # 本文件：格式总索引
│   ├── stock-data.md                  # 股票基础信息 + 关注列表
│   ├── strategy-formats.md            # 策略数据（综合/条件单/网格/高级）
│   ├── strategy-order-create-format.md # 条件单/网格单创建时保存格式
│   ├── trade-record-format.md          # 成交记录格式
│   ├── trend-judgment-format.md       # 趋势判断数据
│   ├── holdings-format.md             # 持仓历史记录
│   ├── config-formats.md              # 应用配置（窗口/WebDAV）
│   ├── webdav-data-formats.md         # WebDAV 同步架构
│   ├── utags-bookmark-format.md       # utags 书签格式索引
│   ├── kids-point-format.md           # kids-point 格式索引
│   ├── my-quant-strategy-format.md    # my-quant 策略格式索引
│   ├── host-disk-info-format.md       # 主机与磁盘信息格式（home_cloud 采集）
│   ├── dir-struct-manifest-format.md  # 目录结构清单（dir-struct-gen 生成 / FluentOS 文件管理器读取）
│   ├── bookkeeping-format.md          # onenav 记账应用格式索引
│   └── chat-record-format.md         # 聊天记录保存格式（home_cloud download_chat_media）
├── app_data/                          # 应用数据（推荐）
│   └── {项目名}/                      # 每个应用一个子目录
├── stocks/                            # 股票基础数据
├── strategies/                        # 策略数据
├── holdings/                          # 持仓数据
├── config/                            # 全局配置
└── trends/                            # 趋势判断数据
```

## 目录约定

- **`app_data/{项目名}/`**（缺省）：所有应用的数据默认存放在此目录下，按项目名创建子目录。
- **`config/`**：全局共享配置（如 WebDAV 连接配置）。
- **`stocks/`、`strategies/`、`holdings/`、`trends/`**：my-quant 应用的数据目录（历史遗留）。

## 文件格式汇总

| 文件 | 用途 | 格式文档 |
|------|------|----------|
| `stocks/stock_data.json` | 股票基础信息 | [stock-data.md](./stock-data.md) |
| `stocks/watchlist.json` | 关注列表 | [stock-data.md](./stock-data.md#关注列表) |
| `strategies/all_strategies.json` | 综合策略（单文件模式） | [strategy-formats.md](./strategy-formats.md) |
| `strategies/conditional_strategies.json` | 条件单策略 | [strategy-formats.md](./strategy-formats.md#2-条件单策略) |
| `strategies/grid_strategies.json` | 网格策略 | [strategy-formats.md](./strategy-formats.md#3-网格策略) |
| `strategies/advanced_strategies.json` | 高级策略 | [strategy-formats.md](./strategy-formats.md#4-高级策略) |
| `app_data/stocks/orders/{YYYY-MM}.json` | 条件单/网格单创建时保存 | [strategy-order-create-format.md](./strategy-order-create-format.md) |
| `app_data/stocks/trade_records/{YYYY-MM}.json` | 成交记录 | [trade-record-format.md](./trade-record-format.md) |
| `trends/trend_judgment_{name}_{date}.json` | 趋势判断 | [trend-judgment-format.md](./trend-judgment-format.md) |
| `holdings/holdings.default.{timestamp}.json` | 持仓历史 | [holdings-format.md](./holdings-format.md) |
| `config/window_config.json` | 窗口配置 | [config-formats.md](./config-formats.md#1-窗口配置) |
| `config/webdav_config.json` | WebDAV 连接配置 | [config-formats.md](./config-formats.md#2-webdav-连接配置) |
| `ufs-nodes/{uuid}.json`（主机状态）/ `O:\app_data\disks\*` | 主机与磁盘信息（home_cloud 采集） | [host-disk-info-format.md](./host-disk-info-format.md) |
| `app_data/chat-exports/{YYYY-MM}/`（含 `images/`、`videos/`、`chat_{YYYY-MM}.html` + 根目录 `download_state.json` / `media_index.json`） | 聊天记录离线存档（home_cloud download_chat_media） | [chat-record-format.md](./chat-record-format.md) |

## 第三方应用数据格式

| 应用 | 说明 | 格式文档 |
|------|------|----------|
| [utags](https://github.com/weijia/utags) | WebDAV 书签标签管理 | [utags-bookmark-format.md](./utags-bookmark-format.md) |
| [kids-point](https://github.com/weijia/kids-point) | 儿童积分奖励系统 | [kids-point-format.md](./kids-point-format.md) |
| [onenav](https://github.com/weijia/onenav) | 内置记账小应用（流水 + 价格 + 电商订单，统一 type 模型） | [bookkeeping-format.md](./bookkeeping-format.md) |
| my-quant | 策略管理与条件单 | [my-quant-strategy-format.md](./my-quant-strategy-format.md) |
| [FluentOS-On-Web](https://github.com/weijia/FluentOS-On-Web)（dir-struct-gen / 文件管理器） | 本地目录结构清单（RemoteStorage 形态 B manifest，文件夹树 + 标签，只读） | [dir-struct-manifest-format.md](./dir-struct-manifest-format.md) |

## WebDAV 同步

所有数据通过 WebDAV 同步，详见 [webdav-data-formats.md](./webdav-data-formats.md)。
