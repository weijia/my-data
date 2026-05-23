# Kids-Point 成就数据

**localStorage 键名**: `kidpoints-achievements`

## 数据结构

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

## 字段说明

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

## requirement.type 类型说明

| 值 | 说明 | 计算方式 |
|----|------|----------|
| `taskCount` | 完成任务数 | 成员 `completedTasks` 数组长度 |
| `pointsTotal` | 累计获得积分 | 成员历史积分总和（需要额外追踪） |
| `rewardsRedeemed` | 已兑换奖励数 | 成员 `redeemedRewards` 数组长度 |
| `custom` | 自定义 | 使用 `taskType` 字段指定特定任务 |

[返回索引](../kids-point-format.md)
