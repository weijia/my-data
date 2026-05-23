# WebDAV 数据同步格式

本文档描述 my-data 仓库中各应用通过 WebDAV 进行数据同步的架构和机制。

## 同步架构

```
                    ┌─────────────────────┐
                    │   WebDAV Server     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────┴──────┐  ┌─────┴──────┐  ┌──────┴─────────┐
    │ 单文件模式      │  │ 独立同步    │  │ 股票数据       │
    │ all_strategies │  │ (趋势/持仓  │  │ (data/watch)   │
    │ .json          │  │  /窗口)     │  │                │
    └────────────────┘  └────────────┘  └───────────────┘
```

## 同步模式说明

| 模式 | 说明 | 涉及文件 |
|------|------|----------|
| 单文件模式（默认） | 所有策略数据合并到一个文件 | `strategies/all_strategies.json` |
| 多文件模式 | 各类型策略分别保存到独立文件 | `conditional_strategies.json`、`grid_strategies.json`、`advanced_strategies.json` |
| 独立同步 | 各数据类型独立同步逻辑 | 趋势判断、持仓、窗口配置 |

## 同步流程

1. **单文件模式（默认）**: 将所有策略数据合并上传到 `strategies/all_strategies.json`
2. **多文件模式**: 各类型策略分别保存到独立文件
3. **独立同步**: 趋势判断、持仓、窗口配置有独立的同步逻辑
4. **上传前自动备份**: 会在上传前将现有文件备份到 `.backup/` 目录（窗口配置文件除外）

## WebDAV 连接

所有应用共享相同的 WebDAV 连接配置，详见 [config-formats.md](./config-formats.md#2-webdav-连接配置)。

认证方式为 HTTP Basic Auth（`username:password`）。

## 各应用 WebDAV 路径

| 应用 | WebDAV 路径 | 数据格式文档 |
|------|-------------|-------------|
| my-quant（策略） | `strategies/` | [strategy-formats.md](./strategy-formats.md) |
| my-quant（趋势） | `trends/` | [trend-judgment-format.md](./trend-judgment-format.md) |
| my-quant（持仓） | `holdings/` | [holdings-format.md](./holdings-format.md) |
| my-quant（配置） | `config/` | [config-formats.md](./config-formats.md) |
| kids-point | `/kidspoints-data/` | [kids-point-format.md](./kids-point-format.md) |
| utags | 由 `target.path` 配置决定 | [utags-bookmark-format.md](./utags-bookmark-format.md) |
