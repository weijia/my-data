# Bookkeeping 数据格式规范

本文档定义了 [onenav](https://github.com/weijia/onenav) 内置小应用 **记账（bookkeeping）** 的数据格式规范，包括存储格式与导入/导出格式。

> **源码参考**（onenav）：
> - 应用组件：`src/components/apps/BookkeepingApp.tsx`
> - 数据存取层：`src/lib/bookkeeping-store.ts`
> - 设计文档：`docs/bookkeeping-data-model.md`

## 概述

Bookkeeping 是一个轻量的记账 / 报价记录应用，采用**统一 `type` 模型**：所有数据（流水 / 价格 / 电商订单）
都是一个 `BookkeepingRecord[]` 的元素，用顶层 `type` 字段区分三类记录：

| `type` | 含义 | 对应文档 |
|---|---|---|
| `'tx'` | 记账 / 流水（收入或支出，内部方向字段为 `direction`） | [transactions.md](./bookkeeping/transactions.md) |
| `'price'` | 价格（单品 / 套餐 / 代金券，子类型字段为 `kind`） | [prices.md](./bookkeeping/prices.md) |
| `'order'` | 电商订单（京东 / 淘宝统一订单 schema） | [orders.md](./bookkeeping/orders.md) |

支持能力：
- 本地 PouchDB 存储（每条记录一个独立文档）+ RemoteStorage 跨设备同步 + WebDAV 落盘兜底
- JSON 导入/导出（支持 Ctrl+V 粘贴、文件选择），导入时自动识别 tx / price / order，支持混合数组
- 旧版单文档 `app:bookkeeping`（`{ transactions, prices }`）首次加载时自动迁移为多文档
- AI 提示词模板（复制给 AI 生成 JSON）
- 统一时间线 UI（流水 / 订单 / 报价同列，顶部筛选 chip 过滤，按页翻页）

## 存储方式

| 层级 | 载体 | 标识 | 说明 |
|------|------|------|------|
| 本地数据库 | PouchDB | 数据库名 `onenav`，每条记录一个文档 `_id = bk:<type>:<id>` | 前端主存储，IndexedDB `_pouch_onenav` |
| WebDAV 落盘 | 文件 | `app_data/onenav/apps/bookkeeping-records.json` | 全部记录聚合的兜底文件 |
| 跨设备同步 | RemoteStorage | 数据库 `onenav` | 通过 PouchDB 整库同步 |

> 旧版单文档 `app:bookkeeping` 已被取代：首次加载检测到旧文档且无 `bk:` 文档时自动迁移，迁移后删除旧文档（幂等）。
> 详细路径与文档结构见 [bookkeeping/storage.md](./bookkeeping/storage.md)。

## 顶层数据结构

统一记录模型（判别联合）：

```ts
type BookkeepingType = 'tx' | 'price' | 'order'

interface BaseRecord {
  id: string          // 唯一 ID（randId() 生成：时间戳-随机串）
  type: BookkeepingType
  date: string        // YYYY-MM-DD（订单取 order_time 前 10 位）
  createdAt: number   // 毫秒级 Unix 时间戳
  source?: string     // 数据来源，如 '京东订单' / '淘宝订单' / 手动
}
```

- `TxRecord = BaseRecord & { type: 'tx'; direction: 'income' | 'expense'; amount; category; ... }`
- `PriceRecord = BaseRecord & { type: 'price'; kind: 'item' | 'combo' | 'voucher'; ... }`
- `OrderRecord = BaseRecord & { type: 'order'; platform; account; orderId; ... }`

> 命名注意：流水的收支方向字段名为 `direction`（旧版的 `Tx.type: 'income' | 'expense'` 已改名，
> 避免与顶层 `type` 语义冲突）；导入 JSON 里的 `type: 'income' | 'expense'` 仍兼容，导入时映射为 `direction`。

## 文档索引

| 文档 | 说明 |
|------|------|
| [存储格式](./bookkeeping/storage.md) | PouchDB `bk:` 多文档结构、旧单文档迁移、WebDAV 路径、RemoteStorage 同步 |
| [交易记录](./bookkeeping/transactions.md) | `type: 'tx'`：支出/收入流水字段、导入容错、导出格式 |
| [电商订单](./bookkeeping/orders.md) | `type: 'order'`：京东/淘宝统一订单 schema、归一化规则、拆单主单过滤、价格提取 |
| [价格记录](./bookkeeping/prices.md) | `type: 'price'`：item / combo / voucher 三种变体 |
| [导入导出格式](./bookkeeping/import-export.md) | JSON 导入/导出格式、自动识别规则、AI 提示词模板 |

## 数据关系图

```
┌─────────────────────────────────────────────────────────────┐
│                  bookkeeping (记账)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐ ┌─────────────────────────┐       │
│  │   TxRecord          │ │      PriceRecord        │       │
│  │  type: 'tx'         │ │   type: 'price'         │       │
│  │  direction: income/ │ │   kind: item/combo/     │       │
│  │  expense            │ │   voucher               │       │
│  └─────────────────────┘ └─────────────────────────┘       │
│  ┌─────────────────────────────────────────────┐           │
│  │   OrderRecord（京东 / 淘宝订单）              │           │
│  │  type: 'order'，platform/account/orderId/…  │           │
│  └─────────────────────────────────────────────┘           │
│           │                    │                   │        │
│           └────────────────────┼───────────────────┘        │
│                                ▼                            │
│              ┌──────────────────────────────┐              │
│              │  BookkeepingRecord[]         │              │
│              │  （顶层 type 区分的判别联合）  │              │
│              └──────────────────────────────┘              │
│                                │                            │
│         ┌──────────────────────┼──────────────────┐        │
│         ▼                      ▼                  ▼        │
│  PouchDB bk:<type>:<id>   WebDAV 聚合兜底    RemoteStorage  │
│  （每条一个文档）           records.json       整库同步      │
└─────────────────────────────────────────────────────────────┘
```

## 相关文档

- [onenav 项目](https://github.com/weijia/onenav)
- [通用数据格式](./data-formats.md)
- [WebDAV 同步格式](./webdav-data-formats.md)
