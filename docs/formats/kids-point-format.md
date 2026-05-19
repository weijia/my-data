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

---

## 1. 成员数据 `kidpoints-members`

**说明**: 存储家庭成员信息，包括积分、已完成任务、已兑换奖励。

```json
[
  {
    "id": "1746000000000",
    "name": "小明",
    "color": "#FF6B6B",
    "points": 150,
    "createdAt": 1746000000000,
    "completedTasks": ["task_001", "task_002"],
    "redeemedRewards": ["reward_001"]
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID，格式为时间戳字符串 |
| `name` | `string` | 成员名称 |
| `color` | `string` | 头像颜色，十六进制颜色码 |
| `points` | `number` | 当前积分余额 |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |
| `completedTasks` | `string[]` | 已完成任务 ID 列表 |
| `redeemedRewards` | `string[]` | 已兑换奖励 ID 列表 |

---

## 2. 任务数据 `kidpoints-tasks`

**说明**: 存储任务定义，包括日常任务、每周任务和一次性任务。

```json
[
  {
    "id": "task_001",
    "title": "整理房间",
    "description": "每天整理自己的房间",
    "points": 10,
    "icon": "🧹",
    "frequency": "daily",
    "assignedTo": [],
    "createdAt": 1746000000000
  },
  {
    "id": "task_002",
    "title": "洗碗",
    "description": "帮忙洗碗",
    "points": 5,
    "icon": "🍽️",
    "frequency": "weekly",
    "assignedTo": ["1746000000001"],
    "createdAt": 1746000000000
  },
  {
    "id": "task_003",
    "title": "期中考试优秀",
    "description": "期中考试获得优秀成绩",
    "points": 50,
    "icon": "🏆",
    "frequency": "once",
    "assignedTo": [],
    "createdAt": 1746000000000
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID |
| `title` | `string` | 任务标题 |
| `description` | `string` | 任务描述 |
| `points` | `number` | 完成奖励积分 |
| `icon` | `string` | 任务图标（emoji） |
| `frequency` | `string` | 频率类型：`"daily"` / `"weekly"` / `"once"` |
| `assignedTo` | `string[]` | 分配的成员 ID 列表，空数组表示所有成员 |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |

### frequency 类型说明

| 值 | 说明 | 重置规则 |
|----|------|----------|
| `daily` | 每日任务 | 每天午夜自动重置 |
| `weekly` | 每周任务 | 每周一自动重置 |
| `once` | 一次性任务 | 完成后不重置 |

---

## 3. 奖励数据 `kidpoints-rewards`

**说明**: 存储可兑换的奖励项目。

```json
[
  {
    "id": "reward_001",
    "title": "冰淇淋",
    "description": "美味的冰淇淋一个",
    "icon": "🍦",
    "points": 30,
    "createdAt": 1746000000000
  },
  {
    "id": "reward_002",
    "title": "玩游戏时间",
    "description": "额外30分钟游戏时间",
    "icon": "🎮",
    "points": 50,
    "createdAt": 1746000000000
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID |
| `title` | `string` | 奖励名称 |
| `description` | `string` | 奖励描述 |
| `icon` | `string` | 奖励图标（emoji） |
| `points` | `number` | 兑换所需积分 |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |

---

## 4. 成就数据 `kidpoints-achievements`

**说明**: 存储成就定义和达成条件。

```json
[
  {
    "id": "ach_001",
    "title": "任务达人",
    "description": "完成10个任务",
    "icon": "🏅",
    "requirement": {
      "type": "taskCount",
      "count": 10,
      "taskType": ""
    },
    "createdAt": 1746000000000
  },
  {
    "id": "ach_002",
    "title": "积分达人",
    "description": "累计获得100积分",
    "icon": "⭐",
    "requirement": {
      "type": "pointsTotal",
      "count": 100,
      "taskType": ""
    },
    "createdAt": 1746000000000
  },
  {
    "id": "ach_003",
    "title": "兑换达人",
    "description": "兑换3个奖励",
    "icon": "🎁",
    "requirement": {
      "type": "rewardsRedeemed",
      "count": 3,
      "taskType": ""
    },
    "createdAt": 1746000000000
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID |
| `title` | `string` | 成就名称 |
| `description` | `string` | 成就描述 |
| `icon` | `string` | 成就图标（emoji） |
| `requirement` | `object` | 达成条件 |
| `requirement.type` | `string` | 条件类型，见下方类型表 |
| `requirement.count` | `number` | 所需数量 |
| `requirement.taskType` | `string` | 任务类型（可选，用于特定类型任务计数） |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |

### requirement.type 类型说明

| 值 | 说明 | 计算方式 |
|----|------|----------|
| `taskCount` | 完成任务数 | 成员 `completedTasks` 数组长度 |
| `pointsTotal` | 累计获得积分 | 成员历史积分总和（需要额外追踪） |
| `rewardsRedeemed` | 已兑换奖励数 | 成员 `redeemedRewards` 数组长度 |
| `custom` | 自定义 | 使用 `taskType` 字段指定特定任务 |

---

## 5. 应用设置 `kidpoints-settings`

**说明**: 存储应用全局设置，包括管理员密码、WebDAV 配置等。

```json
{
  "adminPassword": "123456",
  "isAuthenticated": false,
  "lastDailyReset": 1746000000000,
  "lastWeeklyReset": 1746000000000,
  "notificationsEnabled": true,
  "theme": "light",
  "webdavSync": {
    "url": "https://example.com/dav/",
    "username": "user",
    "password": "pass",
    "enabled": true,
    "lastSyncTime": 1746000000000
  },
  "locale": "zh"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `adminPassword` | `string \| null` | 管理员密码，`null` 表示未设置 |
| `isAuthenticated` | `boolean` | 当前是否已登录 |
| `lastDailyReset` | `number` | 上次每日任务重置时间 |
| `lastWeeklyReset` | `number` | 上次每周任务重置时间 |
| `notificationsEnabled` | `boolean` | 是否启用通知 |
| `theme` | `string` | 主题：`"light"` / `"dark"` |
| `webdavSync` | `WebDAVSyncConfig \| null` | WebDAV 同步配置 |
| `locale` | `string` | 语言：`"en"` / `"zh"` |

### WebDAVSyncConfig 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | `string` | WebDAV 服务器 URL |
| `username` | `string` | WebDAV 用户名 |
| `password` | `string` | WebDAV 密码 |
| `enabled` | `boolean` | 是否启用自动同步 |
| `lastSyncTime` | `number \| null` | 上次同步时间 |

---

## 6. 语言设置 `kidpoints-locale`

**说明**: 存储用户选择的语言偏好。

**值**: `"en"` 或 `"zh"`

---

## 7. PouchDB 数据同步

Kids-Point 使用 PouchDB 作为本地数据库，通过 `universal-sync-v2` 库与 WebDAV 服务器同步。

### PouchDB 数据库名称
- `kidspoints` - 主数据库

### WebDAV 同步路径
- 默认：`/kidspoints-data/`

### 同步数据结构

PouchDB 中的数据以 JSON 文件形式存储在 WebDAV 服务器上，包含所有成员、任务、奖励、成就等数据的快照。

---

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

---

## 相关文档

- [kids-point 项目](https://github.com/weijia/kids-point)
- [通用数据格式](./data-formats.md)
