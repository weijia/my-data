# Kids-Point 数据格式规范

本文档定义了 [kids-point](https://github.com/weijia/kids-point) 项目的数据格式规范。

## 概述

Kids-Point 是一个儿童积分奖励管理系统，支持：
- 成员管理（家庭成员积分）
- 任务系统（日常/每周/一次性任务）
- 奖励商店（积分兑换奖励）
- 成就系统（完成任务解锁成就）
- WebDAV 同步（跨设备数据同步）
- 多语言支持（中/英文）

## 存储方式

数据使用 localStorage 存储在前端，通过 PouchDB + WebDAV 实现跨设备同步。

**localStorage 键名**：
- `kidpoints-settings` - 应用设置
- `kidpoints-members` - 成员数据
- `kidpoints-tasks` - 任务数据
- `kidpoints-rewards` - 奖励数据
- `kidpoints-achievements` - 成就数据
- `kidpoints-current-member` - 当前选中的成员 ID
- `kidpoints-locale` - 语言设置

## 文档索引

| 文档 | 说明 |
|------|------|
| [成员数据](./kids-point/members.md) | 家庭成员信息、积分、已完成任务 |
| [任务数据](./kids-point/tasks.md) | 日常/每周/一次性任务定义 |
| [奖励数据](./kids-point/rewards.md) | 可兑换的奖励项目 |
| [成就数据](./kids-point/achievements.md) | 成就定义和达成条件 |
| [应用设置](./kids-point/settings.md) | 全局设置、WebDAV 配置、语言 |

## 数据关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        kids-point                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐  │
│  │   Members   │────▶│   Tasks    │     │   Rewards   │  │
│  │  (成员)     │     │  (任务)     │     │  (奖励)     │  │
│  └─────────────┘     └─────────────┘     └─────────────┘  │
│        │                                       │           │
│        │ 完成/兑换                              │ 兑换       │
│        ▼                                       ▼           │
│  ┌─────────────┐                        ┌─────────────┐   │
│  │ Achievements│◀───────────────────────│   Members   │   │
│  │  (成就)     │                        │  (成员)     │   │
│  └─────────────┘                        └─────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Settings (设置)                     │   │
│  │  - adminPassword    - webdavSync                    │   │
│  │  - theme           - locale                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## PouchDB 数据同步

Kids-Point 使用 PouchDB 作为本地数据库，通过 `universal-sync-v2` 库与 WebDAV 服务器同步。

- **数据库名称**：`kidspoints`
- **WebDAV 同步路径**：`/kidspoints-data/`

## 相关文档

- [kids-point 项目](https://github.com/weijia/kids-point)
- [通用数据格式](./data-formats.md)
- [WebDAV 同步格式](./webdav-data-formats.md)
