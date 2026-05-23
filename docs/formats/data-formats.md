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
│   ├── trend-judgment-format.md       # 趋势判断数据
│   ├── holdings-format.md             # 持仓历史记录
│   ├── config-formats.md              # 应用配置（窗口/WebDAV）
│   ├── webdav-data-formats.md         # WebDAV 同步架构
│   ├── utags-bookmark-format.md       # utags 书签格式索引
│   ├── kids-point-format.md           # kids-point 格式索引
│   └── my-quant-strategy-format.md    # my-quant 策略格式索引
├── app_data/                          # 应用数据（推荐）
│   └── {项目名}/                      # 每个应用一个子目录
├── stocks/                            # 股票基础数据
├── strategies/                        # 策略数据
├── holdings/                          # 持仓数据
├── config/                            # 全局配置
└── trends/                            # 趋势判断数据
```

## 目录约定

- **`app_data/{项目名}/`**（推荐）：各应用的独立数据存放目录。新应用建议将数据放在此目录下，按项目名创建子目录。
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
| `trends/trend_judgment_{name}_{date}.json` | 趋势判断 | [trend-judgment-format.md](./trend-judgment-format.md) |
| `holdings/holdings.default.{timestamp}.json` | 持仓历史 | [holdings-format.md](./holdings-format.md) |
| `config/window_config.json` | 窗口配置 | [config-formats.md](./config-formats.md#1-窗口配置) |
| `config/webdav_config.json` | WebDAV 连接配置 | [config-formats.md](./config-formats.md#2-webdav-连接配置) |

## 第三方应用数据格式

| 应用 | 说明 | 格式文档 |
|------|------|----------|
| [utags](https://github.com/weijia/utags) | WebDAV 书签标签管理 | [utags-bookmark-format.md](./utags-bookmark-format.md) |
| [kids-point](https://github.com/weijia/kids-point) | 儿童积分奖励系统 | [kids-point-format.md](./kids-point-format.md) |
| my-quant | 策略管理与条件单 | [my-quant-strategy-format.md](./my-quant-strategy-format.md) |

## WebDAV 同步

所有数据通过 WebDAV 同步，详见 [webdav-data-formats.md](./webdav-data-formats.md)。
