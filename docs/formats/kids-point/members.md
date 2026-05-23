# Kids-Point 成员数据

**localStorage 键名**: `kidpoints-members`

## 数据结构

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

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID，格式为时间戳字符串 |
| `name` | `string` | 成员名称 |
| `color` | `string` | 头像颜色，十六进制颜色码 |
| `points` | `number` | 当前积分余额 |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |
| `completedTasks` | `string[]` | 已完成任务 ID 列表 |
| `redeemedRewards` | `string[]` | 已兑换奖励 ID 列表 |

[返回索引](../kids-point-format.md)
