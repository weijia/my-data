# My-Quant 策略数据格式规范

本文档定义 my-quant 应用中策略数据的存储格式。

## 概述

My-quant 使用 PouchDB（IndexedDB）在浏览器本地存储策略数据。策略数据可通过以下方式同步到 WebDAV：
- **单文件模式**（默认）：所有策略合并到 `all_strategies.json`
- **多文件模式**：各类型策略分别保存到独立文件

## 文档索引

| 文档 | 说明 |
|------|------|
| [策略基础结构](./my-quant/strategy-structure.md) | Strategy 接口定义、TrendType、SideType 等 |
| [策略模板格式](./my-quant/strategy-template.md) | StrategyTemplate 接口和 MQTT 条件单消息格式 |
| [MQTT 消息格式](./my-quant/mqtt-message.md) | MQTTMessage 和 OrderCommand 接口 |
| [数据同步格式](./my-quant/sync-format.md) | 单文件模式结构、字段映射、趋势数据注入 |

## 相关文档

- [WebDAV 数据格式](./webdav-data-formats.md) - WebDAV 同步的数据格式
- [数据格式规范](./data-formats.md) - my-data 仓库整体数据规范
- [策略数据格式](./strategy-formats.md) - WebDAV 上的策略文件格式
