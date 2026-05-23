# My-Data

个人数据仓库，用于存储和管理股票交易相关的数据文件。

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
├── stocks/                            # 股票基础数据
├── strategies/                        # 策略数据
├── holdings/                          # 持仓数据
├── config/                            # 配置数据
└── trends/                            # 趋势判断数据
```

## 数据格式

详细的数据格式规范请参考 [docs/formats/data-formats.md](docs/formats/data-formats.md)。

## 使用说明

1. **股票数据** (`stocks/`): 存储股票基础信息和关注列表
2. **策略数据** (`strategies/`): 存储各种交易策略，支持单文件模式和多文件模式
3. **持仓数据** (`holdings/`): 存储持仓历史记录
4. **配置数据** (`config/`): 存储应用配置，如窗口位置、WebDAV 连接等
5. **趋势数据** (`trends/`): 存储股票趋势判断数据

## 同步模式

- **单文件模式**: 所有策略数据合并到 `all_strategies.json`
- **多文件模式**: 各类型策略分别保存到独立文件
- **独立同步**: 趋势判断、持仓、窗口配置独立同步

## 许可证

MIT
