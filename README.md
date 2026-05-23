# My-Data

个人数据仓库，用于存储和管理个人应用相关的数据文件。

## 目录结构

```
my-data/
├── README.md                          # 项目说明
├── docs/formats/                      # 数据格式文档
│   ├── data-formats.md                # 格式总索引
│   ├── stock-data.md                  # 股票基础信息 + 关注列表
│   ├── strategy-formats.md            # 策略数据格式
│   ├── trend-judgment-format.md       # 趋势判断格式
│   ├── holdings-format.md             # 持仓历史格式
│   ├── config-formats.md              # 应用配置格式
│   ├── webdav-data-formats.md         # WebDAV 同步架构
│   ├── utags-bookmark-format.md       # utags 书签格式
│   ├── kids-point-format.md           # kids-point 格式
│   └── my-quant-strategy-format.md    # my-quant 策略格式
├── app_data/                          # 应用数据（推荐）
│   └── {项目名}/                      # 每个应用一个子目录
├── stocks/                            # 股票基础数据
├── strategies/                        # 策略数据
├── holdings/                          # 持仓数据
├── config/                            # 全局配置
└── trends/                            # 趋势判断数据
```

## 目录约定

- **`app_data/{项目名}/`**（推荐）：各应用的独立数据存放目录，每个应用在 `app_data/` 下创建以项目名命名的子目录，存放该应用专属的配置和数据文件。
- **`config/`**：全局共享配置，如 WebDAV 连接配置。
- **`stocks/`、`strategies/`、`holdings/`、`trends/`**：my-quant 应用的数据目录（历史遗留，新应用建议使用 `app_data/`）。

## 数据格式

详细的数据格式规范请参考 [docs/formats/data-formats.md](docs/formats/data-formats.md)。

## 使用说明

1. **应用数据** (`app_data/`): 各应用的独立数据，按项目名分目录存放
2. **股票数据** (`stocks/`): 存储股票基础信息和关注列表
3. **策略数据** (`strategies/`): 存储各种交易策略，支持单文件模式和多文件模式
4. **持仓数据** (`holdings/`): 存储持仓历史记录
5. **配置数据** (`config/`): 存储全局共享配置，如 WebDAV 连接等
6. **趋势数据** (`trends/`): 存储股票趋势判断数据

## 同步模式

- **单文件模式**: 所有策略数据合并到 `all_strategies.json`
- **多文件模式**: 各类型策略分别保存到独立文件
- **独立同步**: 趋势判断、持仓、窗口配置独立同步

## 许可证

MIT
