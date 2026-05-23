# Kids-Point 奖励数据

**localStorage 键名**: `kidpoints-rewards`

## 数据结构

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

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID |
| `title` | `string` | 奖励名称 |
| `description` | `string` | 奖励描述 |
| `icon` | `string` | 奖励图标（emoji） |
| `points` | `number` | 兑换所需积分 |
| `createdAt` | `number` | 创建时间，时间戳（毫秒） |

[返回索引](../kids-point-format.md)
