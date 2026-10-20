# Bookkeeping 数据格式规范

本文档定义了 [onenav](https://github.com/weijia/onenav) 内置小应用 **记账（bookkeeping）** 的数据格式规范，包括存储格式与导入/导出格式。

> **源码参考**（onenav）：
> - 应用组件：`src/components/apps/BookkeepingApp.tsx`
> - 数据存取层：`src/lib/app-data.ts`
> - PouchDB 层：`src/lib/pouchdb.ts`

## 概述

Bookkeeping 是一个轻量的记账 / 报价记录应用，包含两个相互独立的数据集合：

- **交易记录（transactions）**：日常收支流水（支出/收入）
- **价格记录（prices）**：商品单价、套餐、代金券等报价信息

支持能力：
- 本地 PouchDB 存储 + RemoteStorage 跨设备同步 + WebDAV 落盘
- JSON 导入/导出（支持 Ctrl+V 粘贴、文件选择）
- 导入时自动识别每条记录是「交易」还是「价格」，支持混合数组
- AI 提示词模板（复制给 AI 生成 JSON）

## 存储方式

| 层级 | 载体 | 标识 | 说明 |
|------|------|------|------|
| 本地数据库 | PouchDB | 数据库名 `onenav`，文档 `_id = app:bookkeeping` | 前端主存储，IndexedDB `_pouch_onenav` |
| WebDAV 落盘 | 文件 | `app_data/onenav/apps/bookkeeping.json` | 每个应用一个 JSON 文件 |
| 跨设备同步 | RemoteStorage | 数据库 `onenav` | 通过 PouchDB 整库同步 |

> 详细路径与文档结构见 [bookkeeping/storage.md](./bookkeeping/storage.md)。

## 顶层数据结构

`data` 字段（即 PouchDB 文档的 `data`、WebDAV 文件的 `data`）统一为：

```json
{
  "transactions": [],
  "prices": []
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `transactions` | `Tx[]` | 交易记录（流水）数组 |
| `prices` | `PriceRecord[]` | 价格记录数组 |

## 文档索引

| 文档 | 说明 |
|------|------|
| [存储格式](./bookkeeping/storage.md) | PouchDB 文档结构、WebDAV 路径、RemoteStorage 同步 |
| [交易记录](./bookkeeping/transactions.md) | `transactions` 数组：支出/收入流水字段 |
| [价格记录](./bookkeeping/prices.md) | `prices` 数组：item / combo / voucher 三种变体 |
| [导入导出格式](./bookkeeping/import-export.md) | JSON 导入/导出格式、自动识别规则、AI 提示词模板 |

## 数据关系图

```
┌─────────────────────────────────────────────────────────────┐
│                     bookkeeping (记账)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐      ┌─────────────────────────┐  │
│  │   transactions      │      │        prices           │  │
│  │   (交易记录/流水)    │      │       (价格记录)         │  │
│  │  type: income/expense│      │  kind: item/combo/voucher│ │
│  └─────────────────────┘      └─────────────────────────┘  │
│           │                              │                  │
│           │                              │                  │
│           └──────────────┬───────────────┘                  │
│                          ▼                                  │
│              ┌─────────────────────────┐                    │
│              │  BookkeepingData         │                    │
│              │  { transactions, prices } │                    │
│              └─────────────────────────┘                    │
│                          │                                  │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│   ┌──────────┐    ┌───────────┐    ┌────────────┐          │
│   │ PouchDB  │    │  WebDAV   │    │RemoteStorage│          │
│   │ app:记账 │    │ app_data/ │    │  整库同步   │          │
│   └──────────┘    └───────────┘    └────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 相关文档

- [onenav 项目](https://github.com/weijia/onenav)
- [通用数据格式](./data-formats.md)
- [WebDAV 同步格式](./webdav-data-formats.md)
