# Kids-Point 任务数据

**localStorage 键名**: `kidpoints-tasks`

## 数据结构

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

## 字段说明

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

## frequency 类型说明

| 值 | 说明 | 重置规则 |
|----|------|----------|
| `daily` | 每日任务 | 每天午夜自动重置 |
| `weekly` | 每周任务 | 每周一自动重置 |
| `once` | 一次性任务 | 完成后不重置 |

[返回索引](../kids-point-format.md)
