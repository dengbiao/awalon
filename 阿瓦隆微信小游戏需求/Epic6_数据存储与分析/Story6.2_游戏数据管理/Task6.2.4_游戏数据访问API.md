# Task 6.2.4: 游戏数据访问 API

## 任务描述

设计并实现游戏数据访问 API，提供统一、高效的接口用于访问游戏房间信息、历史记录、游戏状态以及统计数据，支持游戏客户端、管理后台及数据分析功能。

## 完成标准

1. 完成游戏房间查询和过滤 API 设计与实现
2. 完成游戏历史记录查询接口设计与实现
3. 完成游戏状态实时获取接口设计与实现
4. 完成游戏数据统计聚合接口设计与实现
5. 完成游戏回放数据获取 API 设计与实现
6. 所有 API 通过单元测试和集成测试
7. API 文档完整，包含示例请求和响应
8. 接口性能满足要求：读取响应时间 < 30ms，写入响应时间 < 80ms

## 技术要点

1. RESTful API 设计规范
2. WebSocket 实时通信
3. JWT 认证与权限控制
4. 缓存策略与数据一致性
5. 分页与过滤参数标准化
6. 错误处理与状态码规范

## 接口设计

### 1. 游戏房间查询和过滤 API

待完成

### 2. 游戏历史记录查询接口

待完成

### 3. 游戏状态实时获取接口

待完成

### 4. 游戏数据统计聚合接口

#### 4.1 接口概述

游戏数据统计聚合接口提供了一系列用于查询和分析游戏统计数据的 API。这些接口支持查询全局游戏统计、角色表现数据、玩家排行榜以及游戏趋势分析等。统计数据对于游戏平衡性调整、玩家行为分析和游戏体验优化具有重要价值。

主要功能包括：

- 获取游戏全局统计数据，如总游戏场次、活跃玩家数、胜率分布等
- 查询特定角色的表现统计，如使用率、胜率、有效性等
- 获取各类排行榜数据，如胜率排行、积分排行等
- 分析游戏数据趋势，如玩家活跃度变化、角色选择偏好变化等

#### 4.2 接口列表

| 接口名称             | HTTP 方法 | 接口路径                       | 描述                                                       |
| -------------------- | --------- | ------------------------------ | ---------------------------------------------------------- |
| 获取全局游戏统计数据 | GET       | /api/v1/stats/global           | 获取游戏整体统计数据，包括总游戏数、活跃玩家数、胜率分布等 |
| 获取角色统计数据     | GET       | /api/v1/stats/roles/{roleName} | 获取特定角色的使用情况、胜率、效能等统计数据               |
| 获取玩家排行榜       | GET       | /api/v1/stats/leaderboard      | 获取不同类型（胜率、场次、角色专精等）的玩家排行榜         |
| 获取游戏趋势分析     | GET       | /api/v1/stats/trends           | 获取游戏指标随时间变化的趋势数据                           |

#### 4.3 获取全局游戏统计数据

##### 接口描述

该接口用于获取阿瓦隆游戏的全局统计数据，包括总游戏数、活跃玩家数、胜率分布、角色使用情况等。支持不同时间段、不同游戏模式的数据筛选。

##### 请求方式

```
GET /api/v1/stats/global
```

##### 请求参数

| 参数名    | 类型   | 必填 | 描述                                                         |
| --------- | ------ | ---- | ------------------------------------------------------------ |
| period    | string | 否   | 统计周期（all/year/month/week/day），默认为 all              |
| gameMode  | string | 否   | 游戏模式（standard/avalon/custom），不传则返回所有模式的统计 |
| startDate | string | 否   | 开始日期（ISO 8601 格式：YYYY-MM-DD），与 endDate 配合使用   |
| endDate   | string | 否   | 结束日期（ISO 8601 格式：YYYY-MM-DD），与 startDate 配合使用 |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "stats": {
      "totalGames": 15782,
      "totalPlayers": 3456,
      "activePlayers": 1205,
      "averageGameDuration": 1845,
      "winRateByTeam": {
        "good": 0.52,
        "evil": 0.48
      },
      "popularRoles": [
        {
          "role": "merlin",
          "playCount": 12567,
          "winRate": 0.58
        },
        {
          "role": "assassin",
          "playCount": 11832,
          "winRate": 0.53
        }
      ],
      "gameModeDistribution": {
        "standard": 0.65,
        "avalon": 0.25,
        "custom": 0.1
      },
      "updateTimestamp": "2023-05-15T10:00:00Z"
    }
  }
}
```

##### 字段说明

| 字段                     | 类型    | 描述                                 |
| ------------------------ | ------- | ------------------------------------ |
| totalGames               | integer | 游戏总场次                           |
| totalPlayers             | integer | 总玩家数量                           |
| activePlayers            | integer | 活跃玩家数量（过去 30 天有游戏记录） |
| averageGameDuration      | integer | 平均游戏时长（秒）                   |
| winRateByTeam            | object  | 按阵营划分的胜率统计                 |
| winRateByTeam.good       | float   | 正义方胜率                           |
| winRateByTeam.evil       | float   | 邪恶方胜率                           |
| popularRoles             | array   | 角色使用情况排行                     |
| popularRoles[].role      | string  | 角色名称                             |
| popularRoles[].playCount | integer | 使用次数                             |
| popularRoles[].winRate   | float   | 角色胜率                             |
| gameModeDistribution     | object  | 游戏模式分布                         |
| gameModeDistribution.\*  | float   | 各游戏模式占比                       |
| updateTimestamp          | string  | 数据更新时间戳                       |

#### 4.4 获取角色统计数据

##### 接口描述

该接口用于获取特定角色的详细统计数据，包括使用频率、胜率、平均分数、效能评估等。可以帮助玩家了解不同角色的表现，也可用于游戏平衡性调整的参考。

##### 请求方式

```
GET /api/v1/stats/roles/{roleName}
```

##### 路径参数

| 参数名   | 类型   | 必填 | 描述                                       |
| -------- | ------ | ---- | ------------------------------------------ |
| roleName | string | 是   | 角色名称，如 merlin、assassin、percival 等 |

##### 查询参数

| 参数名    | 类型   | 必填 | 描述                                             |
| --------- | ------ | ---- | ------------------------------------------------ |
| period    | string | 否   | 统计周期（all/year/month/week/day），默认为 all  |
| gameMode  | string | 否   | 游戏模式，不传则返回所有模式下该角色的统计       |
| startDate | string | 否   | 开始日期（ISO 8601 格式），与 endDate 配合使用   |
| endDate   | string | 否   | 结束日期（ISO 8601 格式），与 startDate 配合使用 |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "role": "merlin",
    "stats": {
      "playCount": 12567,
      "winRate": 0.58,
      "averageScore": 125.5,
      "effectiveness": 0.67,
      "pickRate": 0.79,
      "counterRoles": [
        {
          "role": "assassin",
          "winRateAgainst": 0.42,
          "encounterRate": 0.85
        },
        {
          "role": "morgana",
          "winRateAgainst": 0.48,
          "encounterRate": 0.72
        }
      ],
      "synergies": [
        {
          "role": "percival",
          "winRateWith": 0.63,
          "encounterRate": 0.72
        },
        {
          "role": "loyal_servant",
          "winRateWith": 0.59,
          "encounterRate": 0.95
        }
      ],
      "performanceByPlayerCount": {
        "5": {
          "winRate": 0.6,
          "playCount": 1508
        },
        "6": {
          "winRate": 0.58,
          "playCount": 2305
        }
      },
      "updateTimestamp": "2023-05-15T10:00:00Z"
    }
  }
}
```

##### 字段说明

| 字段                          | 类型    | 描述                         |
| ----------------------------- | ------- | ---------------------------- |
| role                          | string  | 角色名称                     |
| playCount                     | integer | 使用次数                     |
| winRate                       | float   | 角色胜率                     |
| averageScore                  | float   | 平均得分                     |
| effectiveness                 | float   | 角色效能评估（0-1）          |
| pickRate                      | float   | 角色选择率                   |
| counterRoles                  | array   | 克制该角色的其他角色列表     |
| counterRoles[].role           | string  | 克制角色名称                 |
| counterRoles[].winRateAgainst | float   | 对抗胜率                     |
| counterRoles[].encounterRate  | float   | 遭遇率                       |
| synergies                     | array   | 与该角色协同效果好的角色列表 |
| synergies[].role              | string  | 协同角色名称                 |
| synergies[].winRateWith       | float   | 协同胜率                     |
| synergies[].encounterRate     | float   | 同场率                       |
| performanceByPlayerCount      | object  | 不同玩家人数下的表现         |
| performanceByPlayerCount.\*   | object  | 各玩家人数配置的统计数据     |
| updateTimestamp               | string  | 数据更新时间戳               |

#### 4.5 获取玩家排行榜

##### 接口描述

该接口用于获取阿瓦隆游戏的玩家排行榜数据，支持多种排行类型，包括胜率排行、游戏场次排行、积分排行以及特定角色专精排行等。可以按照不同的统计周期和游戏模式进行筛选。

##### 请求方式

```
GET /api/v1/stats/leaderboard
```

##### 查询参数

| 参数名   | 类型    | 必填 | 描述                                           |
| -------- | ------- | ---- | ---------------------------------------------- |
| type     | string  | 是   | 排行榜类型（winRate/score/games/roleSpecific） |
| role     | string  | 否   | 角色名称（当 type=roleSpecific 时必填）        |
| team     | string  | 否   | 阵营（good/evil），可选过滤条件                |
| period   | string  | 否   | 统计周期（all/year/month/week），默认为 month  |
| gameMode | string  | 否   | 游戏模式，不传则计算所有模式                   |
| page     | integer | 否   | 分页页码，默认为 1                             |
| limit    | integer | 否   | 每页数量，默认为 20，最大为 100                |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "leaderboard": {
      "type": "winRate",
      "period": "month",
      "totalPlayers": 3456,
      "rankings": [
        {
          "rank": 1,
          "userId": "user123456",
          "nickname": "战术大师",
          "avatarUrl": "https://example.com/avatars/user123456.jpg",
          "winRate": 0.92,
          "gamesPlayed": 135,
          "favoriteRole": "merlin"
        },
        {
          "rank": 2,
          "userId": "user789012",
          "nickname": "谎言专家",
          "avatarUrl": "https://example.com/avatars/user789012.jpg",
          "winRate": 0.89,
          "gamesPlayed": 158,
          "favoriteRole": "assassin"
        }
      ],
      "updateTimestamp": "2023-05-15T10:00:00Z"
    },
    "pagination": {
      "total": 3456,
      "page": 1,
      "limit": 20,
      "pages": 173
    }
  }
}
```

##### 字段说明

| 字段                                | 类型    | 描述                            |
| ----------------------------------- | ------- | ------------------------------- |
| leaderboard.type                    | string  | 排行榜类型                      |
| leaderboard.period                  | string  | 统计周期                        |
| leaderboard.totalPlayers            | integer | 参与排名的总玩家数              |
| leaderboard.rankings                | array   | 排名列表                        |
| leaderboard.rankings[].rank         | integer | 玩家排名                        |
| leaderboard.rankings[].userId       | string  | 玩家 ID                         |
| leaderboard.rankings[].nickname     | string  | 玩家昵称                        |
| leaderboard.rankings[].avatarUrl    | string  | 玩家头像 URL                    |
| leaderboard.rankings[].winRate      | float   | 胜率（仅在 winRate 类型时提供） |
| leaderboard.rankings[].score        | integer | 积分（仅在 score 类型时提供）   |
| leaderboard.rankings[].gamesPlayed  | integer | 游戏场次                        |
| leaderboard.rankings[].favoriteRole | string  | 最常使用角色                    |
| leaderboard.updateTimestamp         | string  | 数据更新时间戳                  |
| pagination                          | object  | 分页信息                        |
| pagination.total                    | integer | 总记录数                        |
| pagination.page                     | integer | 当前页码                        |
| pagination.limit                    | integer | 每页记录数                      |
| pagination.pages                    | integer | 总页数                          |

#### 4.6 获取游戏趋势分析

##### 接口描述

该接口用于获取游戏数据随时间变化的趋势分析，支持多种分析指标，包括玩家活跃度、游戏场次、胜率分布、游戏时长以及角色选择偏好等。可用于分析游戏健康度、平衡性变化以及玩家行为演变。

##### 请求方式

```
GET /api/v1/stats/trends
```

##### 查询参数

| 参数名    | 类型   | 必填 | 描述                                                        |
| --------- | ------ | ---- | ----------------------------------------------------------- |
| metric    | string | 是   | 分析指标（players/games/winRate/duration/roleDistribution） |
| interval  | string | 否   | 时间间隔（day/week/month），默认为 week                     |
| gameMode  | string | 否   | 游戏模式，不传则分析所有模式                                |
| startDate | string | 否   | 开始日期（ISO 8601 格式），默认为 3 个月前                  |
| endDate   | string | 否   | 结束日期（ISO 8601 格式），默认为当前日期                   |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "trend": {
      "metric": "winRate",
      "interval": "week",
      "gameMode": "standard",
      "startDate": "2023-03-01T00:00:00Z",
      "endDate": "2023-05-15T00:00:00Z",
      "points": [
        {
          "period": "2023-03-01",
          "values": {
            "good": 0.51,
            "evil": 0.49
          },
          "gamesCount": 1245
        },
        {
          "period": "2023-03-08",
          "values": {
            "good": 0.52,
            "evil": 0.48
          },
          "gamesCount": 1356
        },
        {
          "period": "2023-03-15",
          "values": {
            "good": 0.53,
            "evil": 0.47
          },
          "gamesCount": 1450
        }
      ],
      "updateTimestamp": "2023-05-15T10:00:00Z"
    }
  }
}
```

##### 字段说明

| 字段                      | 类型    | 描述                                   |
| ------------------------- | ------- | -------------------------------------- |
| trend.metric              | string  | 分析指标                               |
| trend.interval            | string  | 时间间隔                               |
| trend.gameMode            | string  | 游戏模式                               |
| trend.startDate           | string  | 开始日期                               |
| trend.endDate             | string  | 结束日期                               |
| trend.points              | array   | 趋势数据点列表                         |
| trend.points[].period     | string  | 时间段（格式根据 interval 不同而变化） |
| trend.points[].values     | object  | 指标值，结构根据 metric 不同而变化     |
| trend.points[].gamesCount | integer | 该时间段的游戏场次                     |
| trend.updateTimestamp     | string  | 数据更新时间戳                         |

#### 4.7 错误处理

##### 错误响应格式

所有游戏数据统计聚合 API 在发生错误时，将返回统一格式的错误信息：

```json
{
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述信息",
    "details": {
      "param1": "错误详细信息"
    }
  }
}
```

##### 常见错误代码

| 错误代码             | HTTP 状态码 | 描述                 |
| -------------------- | ----------- | -------------------- |
| INVALID_PARAMETER    | 400         | 无效的请求参数       |
| ROLE_NOT_FOUND       | 404         | 角色不存在           |
| INSUFFICIENT_DATA    | 404         | 数据量不足以进行统计 |
| DATE_RANGE_TOO_LARGE | 400         | 日期范围过大         |
| NOT_AUTHORIZED       | 401         | 未授权访问           |
| STATS_UNAVAILABLE    | 503         | 统计数据暂时不可用   |

##### 示例错误响应

```json
{
  "status": "error",
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "无效的请求参数",
    "details": {
      "metric": "不支持的分析指标"
    }
  }
}
```

#### 4.8 数据缓存策略

##### 服务端缓存

为提高游戏数据统计 API 的响应速度，降低数据库负载，系统采用以下缓存策略：

1. **全局统计数据**：每小时更新一次，使用 Redis 缓存
2. **角色统计数据**：每天更新一次，使用 Redis 缓存
3. **排行榜数据**：热门排行榜（如月度胜率、总场次等）每小时更新一次，其他排行榜每天更新一次
4. **趋势数据**：每天凌晨进行计算和更新

##### 缓存标识

所有 API 响应中都包含`updateTimestamp`字段，表示数据的最后更新时间，客户端可根据此字段判断数据的新鲜度。

##### HTTP 缓存控制

API 响应头部包含以下缓存控制信息：

```
Cache-Control: max-age=3600, public
Expires: Wed, 15 May 2023 11:00:00 GMT
Last-Modified: Wed, 15 May 2023 10:00:00 GMT
ETag: "a1b2c3d4e5f6"
```

##### 缓存失效

在以下情况下会触发缓存提前失效：

1. 游戏规则发生重大调整
2. 角色属性发生变化
3. 统计算法优化升级
4. 管理员手动触发缓存刷新

#### 4.9 性能要求

游戏数据统计聚合接口的性能要求：

1. 全局统计数据和角色统计 API 平均响应时间< 30ms
2. 排行榜 API 平均响应时间< 50ms
3. 趋势分析 API 平均响应时间< 80ms
4. API 峰值处理能力：100 QPS
5. 每天定时统计处理能力：处理过去 24 小时内 30 万场游戏的统计数据<10 分钟

### 5. 游戏回放数据获取 API

#### 5.1 接口概述

游戏回放数据获取 API 提供了一系列用于获取和处理游戏回放数据的接口。这些接口支持玩家查看历史游戏的详细过程、重要时刻，以及游戏中各种事件的发生顺序。回放功能不仅能让玩家重温精彩游戏，也能作为学习策略和改进技巧的重要工具。

主要功能包括：

- 获取游戏回放元数据，如游戏基本信息、参与玩家、游戏结果等
- 按时间顺序获取游戏中的事件序列
- 获取游戏关键时刻的快照数据
- 支持分段获取回放数据，优化网络传输
- 提供数据格式转换，支持多种客户端回放方式

#### 5.2 接口列表

| 接口名称           | HTTP 方法 | 接口路径                                       | 描述                                               |
| ------------------ | --------- | ---------------------------------------------- | -------------------------------------------------- |
| 获取游戏回放元数据 | GET       | /api/v1/replays/{gameId}/metadata              | 获取特定游戏的回放元数据，包括基本信息、参与玩家等 |
| 获取游戏事件序列   | GET       | /api/v1/replays/{gameId}/events                | 获取游戏中的完整事件序列，支持分页和过滤           |
| 获取游戏状态快照   | GET       | /api/v1/replays/{gameId}/snapshots/{timestamp} | 获取游戏在特定时间点的状态快照                     |
| 获取游戏关键时刻   | GET       | /api/v1/replays/{gameId}/highlights            | 获取游戏中的关键时刻列表                           |
| 获取回放数据包     | GET       | /api/v1/replays/{gameId}/package               | 获取完整的游戏回放数据包，适用于客户端离线回放     |

#### 5.3 获取游戏回放元数据

##### 接口描述

该接口用于获取特定游戏的回放元数据，包括游戏的基本信息、参与玩家、游戏结果等。这些元数据是回放功能的基础，客户端可以据此构建回放界面和控制逻辑。

##### 请求方式

```
GET /api/v1/replays/{gameId}/metadata
```

##### 路径参数

| 参数名 | 类型   | 必填 | 描述           |
| ------ | ------ | ---- | -------------- |
| gameId | string | 是   | 游戏唯一标识符 |

##### 查询参数

| 参数名               | 类型    | 必填 | 描述                                 |
| -------------------- | ------- | ---- | ------------------------------------ |
| includePlayerDetails | boolean | 否   | 是否包含详细的玩家信息，默认为 false |
| includeGameSettings  | boolean | 否   | 是否包含详细的游戏设置，默认为 false |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "metadata": {
      "gameId": "game789012",
      "roomId": "room123456",
      "roomName": "阿瓦隆欢乐局",
      "gameMode": "standard",
      "startedAt": "2023-05-15T09:00:00Z",
      "endedAt": "2023-05-15T09:35:27Z",
      "duration": 2127,
      "winner": "good",
      "players": [
        {
          "userId": "user789012",
          "nickname": "游戏大师",
          "role": "merlin",
          "team": "good",
          "result": "win",
          "avatarUrl": "https://example.com/avatars/user789012.jpg"
        },
        {
          "userId": "user345678",
          "nickname": "策略高手",
          "role": "assassin",
          "team": "evil",
          "result": "lose",
          "avatarUrl": "https://example.com/avatars/user345678.jpg"
        }
      ],
      "rounds": 5,
      "totalEvents": 128,
      "highlights": 8,
      "availableSnapshots": [
        "2023-05-15T09:00:00Z",
        "2023-05-15T09:05:12Z",
        "2023-05-15T09:12:45Z",
        "2023-05-15T09:20:30Z",
        "2023-05-15T09:35:27Z"
      ],
      "replayVersion": "1.2",
      "createdAt": "2023-05-15T09:40:15Z"
    }
  }
}
```

##### 字段说明

| 字段                | 类型    | 描述                         |
| ------------------- | ------- | ---------------------------- |
| gameId              | string  | 游戏唯一标识符               |
| roomId              | string  | 游戏房间 ID                  |
| roomName            | string  | 游戏房间名称                 |
| gameMode            | string  | 游戏模式                     |
| startedAt           | string  | 游戏开始时间                 |
| endedAt             | string  | 游戏结束时间                 |
| duration            | integer | 游戏持续时间（秒）           |
| winner              | string  | 胜利方阵营                   |
| players             | array   | 参与玩家列表                 |
| players[].userId    | string  | 玩家 ID                      |
| players[].nickname  | string  | 玩家昵称                     |
| players[].role      | string  | 玩家角色                     |
| players[].team      | string  | 玩家阵营                     |
| players[].result    | string  | 玩家游戏结果                 |
| players[].avatarUrl | string  | 玩家头像 URL                 |
| rounds              | integer | 游戏总回合数                 |
| totalEvents         | integer | 游戏事件总数                 |
| highlights          | integer | 关键时刻数量                 |
| availableSnapshots  | array   | 可用的游戏状态快照时间点列表 |
| replayVersion       | string  | 回放数据格式版本             |
| createdAt           | string  | 回放数据创建时间             |

#### 5.4 获取游戏事件序列

##### 接口描述

该接口用于获取游戏中的事件序列，这些事件按时间顺序排列，共同构成了游戏的完整过程。客户端可以根据这些事件重现游戏的每一步操作和状态变化。支持分页获取和按事件类型过滤，以优化数据传输和展示。

##### 请求方式

```
GET /api/v1/replays/{gameId}/events
```

##### 路径参数

| 参数名 | 类型   | 必填 | 描述           |
| ------ | ------ | ---- | -------------- |
| gameId | string | 是   | 游戏唯一标识符 |

##### 查询参数

| 参数名    | 类型    | 必填 | 描述                                                                    |
| --------- | ------- | ---- | ----------------------------------------------------------------------- |
| startTime | string  | 否   | 事件开始时间（ISO 8601 格式），默认为游戏开始时间                       |
| endTime   | string  | 否   | 事件结束时间（ISO 8601 格式），默认为游戏结束时间                       |
| types     | string  | 否   | 事件类型过滤，多个类型用逗号分隔（如：ROUND_START,VOTE,MISSION_RESULT） |
| page      | integer | 否   | 分页页码，默认为 1                                                      |
| limit     | integer | 否   | 每页事件数量，默认为 50，最大为 200                                     |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "events": [
      {
        "eventId": "event12345",
        "type": "GAME_START",
        "timestamp": "2023-05-15T09:00:00Z",
        "data": {
          "roles": {
            "user789012": "merlin",
            "user345678": "assassin"
          },
          "firstLeader": "user789012"
        },
        "visibleTo": ["all"]
      },
      {
        "eventId": "event12346",
        "type": "ROUND_START",
        "timestamp": "2023-05-15T09:01:30Z",
        "data": {
          "roundNumber": 1,
          "leader": "user789012"
        },
        "visibleTo": ["all"]
      },
      {
        "eventId": "event12347",
        "type": "TEAM_SELECTION",
        "timestamp": "2023-05-15T09:02:45Z",
        "data": {
          "roundNumber": 1,
          "leader": "user789012",
          "selectedTeam": ["user789012", "user345678", "user567890"]
        },
        "visibleTo": ["all"]
      }
    ],
    "pagination": {
      "total": 128,
      "page": 1,
      "limit": 50,
      "pages": 3
    },
    "timeRange": {
      "start": "2023-05-15T09:00:00Z",
      "end": "2023-05-15T09:35:27Z"
    }
  }
}
```

##### 字段说明

| 字段               | 类型    | 描述                               |
| ------------------ | ------- | ---------------------------------- |
| events             | array   | 游戏事件列表                       |
| events[].eventId   | string  | 事件唯一标识符                     |
| events[].type      | string  | 事件类型                           |
| events[].timestamp | string  | 事件发生时间                       |
| events[].data      | object  | 事件详细数据，结构因事件类型而异   |
| events[].visibleTo | array   | 事件可见范围（all 表示所有人可见） |
| pagination         | object  | 分页信息                           |
| pagination.total   | integer | 总事件数                           |
| pagination.page    | integer | 当前页码                           |
| pagination.limit   | integer | 每页事件数                         |
| pagination.pages   | integer | 总页数                             |
| timeRange          | object  | 事件时间范围                       |
| timeRange.start    | string  | 起始时间                           |
| timeRange.end      | string  | 结束时间                           |

##### 事件类型说明

| 事件类型       | 描述         |
| -------------- | ------------ |
| GAME_START     | 游戏开始     |
| ROUND_START    | 回合开始     |
| TEAM_SELECTION | 队伍选择     |
| TEAM_VOTE      | 队伍投票     |
| VOTE_RESULT    | 投票结果     |
| MISSION_START  | 任务开始     |
| MISSION_VOTE   | 任务投票     |
| MISSION_RESULT | 任务结果     |
| ROLE_ABILITY   | 角色能力使用 |
| ASSASSIN_GUESS | 刺客猜测     |
| GAME_END       | 游戏结束     |

#### 5.5 获取游戏状态快照

##### 接口描述

该接口用于获取游戏在特定时间点的状态快照，包括当时的游戏状态、玩家状态、任务状态等。客户端可以通过快照直接定位到游戏的某个时刻，无需重放前面的所有事件，便于快速跳转和定位。

##### 请求方式

```
GET /api/v1/replays/{gameId}/snapshots/{timestamp}
```

##### 路径参数

| 参数名    | 类型   | 必填 | 描述                                            |
| --------- | ------ | ---- | ----------------------------------------------- |
| gameId    | string | 是   | 游戏唯一标识符                                  |
| timestamp | string | 是   | 快照时间点（ISO 8601 格式），需是可用快照点之一 |

##### 查询参数

| 参数名      | 类型   | 必填 | 描述                                                               |
| ----------- | ------ | ---- | ------------------------------------------------------------------ |
| perspective | string | 否   | 查看视角，可选值：omniscient（上帝视角）、specific（指定玩家视角） |
| userId      | string | 否   | 当 perspective=specific 时，指定视角的玩家 ID                      |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "snapshot": {
      "gameId": "game789012",
      "timestamp": "2023-05-15T09:12:45Z",
      "gameState": {
        "phase": "mission",
        "round": 2,
        "currentLeader": "user567890",
        "voteTrack": 1,
        "missionTrack": [
          {
            "roundNumber": 1,
            "result": "success",
            "votesRequired": 2,
            "successVotes": 3,
            "failVotes": 0
          }
        ],
        "remainingMissions": 4
      },
      "players": [
        {
          "userId": "user789012",
          "role": "merlin",
          "team": "good",
          "isActive": true,
          "hasVoted": true,
          "isOnCurrentTeam": true
        },
        {
          "userId": "user345678",
          "role": "assassin",
          "team": "evil",
          "isActive": true,
          "hasVoted": false,
          "isOnCurrentTeam": true
        }
      ],
      "currentTeam": ["user789012", "user345678", "user234567"],
      "visibilityMask": {
        "revealedRoles": {
          "user789012": ["user345678", "user123456"],
          "user345678": ["user123456"]
        }
      }
    },
    "perspective": "omniscient",
    "nextEventId": "event12367"
  }
}
```

##### 字段说明

| 字段                                 | 类型    | 描述                    |
| ------------------------------------ | ------- | ----------------------- |
| snapshot                             | object  | 游戏状态快照            |
| snapshot.gameId                      | string  | 游戏唯一标识符          |
| snapshot.timestamp                   | string  | 快照时间点              |
| snapshot.gameState                   | object  | 游戏状态信息            |
| snapshot.gameState.phase             | string  | 当前游戏阶段            |
| snapshot.gameState.round             | integer | 当前回合数              |
| snapshot.gameState.currentLeader     | string  | 当前回合领袖            |
| snapshot.gameState.voteTrack         | integer | 当前投票轮次            |
| snapshot.gameState.missionTrack      | array   | 已完成任务列表          |
| snapshot.gameState.remainingMissions | integer | 剩余任务数              |
| snapshot.players                     | array   | 玩家状态列表            |
| snapshot.players[].userId            | string  | 玩家 ID                 |
| snapshot.players[].role              | string  | 玩家角色                |
| snapshot.players[].team              | string  | 玩家阵营                |
| snapshot.players[].isActive          | boolean | 玩家是否仍在游戏中      |
| snapshot.players[].hasVoted          | boolean | 玩家是否已投票          |
| snapshot.players[].isOnCurrentTeam   | boolean | 玩家是否在当前队伍中    |
| snapshot.currentTeam                 | array   | 当前队伍玩家 ID 列表    |
| snapshot.visibilityMask              | object  | 玩家间信息可见性        |
| perspective                          | string  | 当前查看视角            |
| nextEventId                          | string  | 该快照后的下一个事件 ID |

#### 5.6 获取游戏关键时刻

##### 接口描述

该接口用于获取游戏中的关键时刻列表，包括重要事件发生的时间点和简要描述。这些关键时刻通常包括任务结果公布、重要投票、刺客猜测等对游戏走向有重大影响的事件，便于用户快速定位和回看游戏中的精彩瞬间。

##### 请求方式

```
GET /api/v1/replays/{gameId}/highlights
```

##### 路径参数

| 参数名 | 类型   | 必填 | 描述           |
| ------ | ------ | ---- | -------------- |
| gameId | string | 是   | 游戏唯一标识符 |

##### 查询参数

| 参数名 | 类型    | 必填 | 描述                                                                                |
| ------ | ------- | ---- | ----------------------------------------------------------------------------------- |
| type   | string  | 否   | 关键时刻类型过滤，可选值：mission_result/vote_result/assassin_guess/all，默认为 all |
| limit  | integer | 否   | 返回的关键时刻数量限制，默认为 10，最大为 50                                        |

##### 响应结果

```json
{
  "status": "success",
  "data": {
    "highlights": [
      {
        "id": "highlight123",
        "type": "mission_result",
        "timestamp": "2023-05-15T09:08:45Z",
        "title": "第1轮任务成功",
        "description": "3人任务团队，全部投了成功票",
        "eventId": "event12358",
        "importance": 8
      },
      {
        "id": "highlight124",
        "type": "vote_result",
        "timestamp": "2023-05-15T09:15:20Z",
        "title": "第2轮队伍投票失败",
        "description": "5人中3人投了反对票，团队组建失败",
        "eventId": "event12370",
        "importance": 6
      },
      {
        "id": "highlight125",
        "type": "assassin_guess",
        "timestamp": "2023-05-15T09:34:50Z",
        "title": "刺客猜测失败",
        "description": "刺客(user345678)猜测梅林是user123456，但实际梅林是user789012",
        "eventId": "event12425",
        "importance": 10
      }
    ],
    "count": 3,
    "gameId": "game789012"
  }
}
```

##### 字段说明

| 字段                     | 类型    | 描述               |
| ------------------------ | ------- | ------------------ |
| highlights               | array   | 关键时刻列表       |
| highlights[].id          | string  | 关键时刻唯一标识符 |
| highlights[].type        | string  | 关键时刻类型       |
| highlights[].timestamp   | string  | 发生时间           |
| highlights[].title       | string  | 关键时刻标题       |
| highlights[].description | string  | 关键时刻描述       |
| highlights[].eventId     | string  | 对应的事件 ID      |
| highlights[].importance  | integer | 重要性评分（1-10） |
| count                    | integer | 返回的关键时刻数量 |
| gameId                   | string  | 游戏 ID            |

#### 5.7 获取回放数据包

##### 接口描述

该接口用于获取完整的游戏回放数据包，包括游戏元数据、事件序列和关键状态快照，便于客户端离线回放和分析。数据包可以根据需求选择不同的格式和内容范围，支持压缩以减少传输量。

##### 请求方式

```
GET /api/v1/replays/{gameId}/package
```

##### 路径参数

| 参数名 | 类型   | 必填 | 描述           |
| ------ | ------ | ---- | -------------- |
| gameId | string | 是   | 游戏唯一标识符 |

##### 查询参数

| 参数名            | 类型    | 必填 | 描述                                                     |
| ----------------- | ------- | ---- | -------------------------------------------------------- |
| format            | string  | 否   | 数据包格式，可选值：json/binary/protobuf，默认为 json    |
| compressed        | boolean | 否   | 是否压缩数据，默认为 true                                |
| includeSnapshots  | boolean | 否   | 是否包含状态快照，默认为 true                            |
| includeVisibility | boolean | 否   | 是否包含可见性信息，默认为 true                          |
| perspective       | string  | 否   | 查看视角，可选值：omniscient/specific，默认为 omniscient |
| userId            | string  | 否   | 当 perspective=specific 时，指定视角的玩家 ID            |

##### 响应结果

```
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="game789012_replay.json.gz"
Content-Length: 25678
```

##### 响应数据结构（解压后）

```json
{
  "version": "1.2",
  "metadata": {
    "gameId": "game789012",
    "roomId": "room123456",
    "startedAt": "2023-05-15T09:00:00Z",
    "endedAt": "2023-05-15T09:35:27Z",
    "gameMode": "standard",
    "players": [
      {
        "userId": "user789012",
        "nickname": "游戏大师",
        "role": "merlin",
        "team": "good"
      },
      {
        "userId": "user345678",
        "nickname": "策略高手",
        "role": "assassin",
        "team": "evil"
      }
    ],
    "winner": "good",
    "rounds": 5
  },
  "events": [
    {
      "eventId": "event12345",
      "type": "GAME_START",
      "timestamp": "2023-05-15T09:00:00Z",
      "data": {
        "roles": {
          "user789012": "merlin",
          "user345678": "assassin"
        },
        "firstLeader": "user789012"
      },
      "visibleTo": ["all"]
    }
    // ... 更多事件 ...
  ],
  "snapshots": {
    "2023-05-15T09:00:00Z": {
      // 游戏开始时的状态快照
    },
    "2023-05-15T09:12:45Z": {
      // 中间状态快照
    },
    "2023-05-15T09:35:27Z": {
      // 游戏结束时的状态快照
    }
  },
  "highlights": [
    {
      "id": "highlight123",
      "type": "mission_result",
      "timestamp": "2023-05-15T09:08:45Z",
      "title": "第1轮任务成功",
      "eventId": "event12358"
    }
    // ... 更多关键时刻 ...
  ],
  "visibilityMatrix": {
    // 信息可见性矩阵
  },
  "createdAt": "2023-05-15T09:40:15Z"
}
```

##### 字段说明

| 字段             | 类型   | 描述                         |
| ---------------- | ------ | ---------------------------- |
| version          | string | 回放数据格式版本             |
| metadata         | object | 游戏元数据                   |
| events           | array  | 完整的游戏事件序列           |
| snapshots        | object | 游戏状态快照集合，键为时间戳 |
| highlights       | array  | 游戏关键时刻列表             |
| visibilityMatrix | object | 游戏信息可见性矩阵           |
| createdAt        | string | 回放数据包创建时间           |

#### 5.8 错误处理

##### 错误响应格式

所有游戏回放数据获取 API 在发生错误时，将返回统一格式的错误信息：

```json
{
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述信息",
    "details": {
      "param1": "错误详细信息"
    }
  }
}
```

##### 常见错误代码

| 错误代码                  | HTTP 状态码 | 描述                   |
| ------------------------- | ----------- | ---------------------- |
| GAME_NOT_FOUND            | 404         | 游戏 ID 不存在         |
| REPLAY_NOT_AVAILABLE      | 404         | 回放数据不可用         |
| SNAPSHOT_NOT_FOUND        | 404         | 指定的快照时间点不存在 |
| INVALID_PARAMETER         | 400         | 无效的请求参数         |
| NOT_AUTHORIZED            | 401         | 未授权访问             |
| REPLAY_PROCESSING         | 409         | 回放数据仍在处理中     |
| REPLAY_FORMAT_UNSUPPORTED | 400         | 不支持的回放格式       |

#### 5.9 数据保留策略

游戏回放数据的保留策略如下：

1. 标准游戏模式的回放数据保留 30 天
2. 竞技模式和锦标赛的回放数据保留 90 天
3. 系统自动标记的精彩游戏回放保留 180 天
4. 玩家可以在游戏结束后手动收藏回放，收藏的回放将永久保存
5. 所有超过保留期限的回放数据会进行匿名化处理，用于游戏数据分析

### 6. 错误码与状态码规范

为确保 API 调用方能够正确理解和处理各种响应情况，我们定义了统一的错误码和 HTTP 状态码规范，适用于所有游戏数据访问 API。

#### 6.1 HTTP 状态码使用规范

| HTTP 状态码               | 使用场景                               |
| ------------------------- | -------------------------------------- |
| 200 OK                    | 请求成功处理并返回预期结果             |
| 201 Created               | 成功创建新资源（如新建游戏房间）       |
| 204 No Content            | 请求成功处理但无返回内容（如删除操作） |
| 400 Bad Request           | 请求参数错误或不完整                   |
| 401 Unauthorized          | 未提供身份验证或身份验证无效           |
| 403 Forbidden             | 身份验证有效但权限不足                 |
| 404 Not Found             | 请求的资源不存在                       |
| 409 Conflict              | 请求冲突（如创建已存在的资源）         |
| 429 Too Many Requests     | 请求频率超过限制                       |
| 500 Internal Server Error | 服务器内部错误                         |
| 503 Service Unavailable   | 服务暂时不可用（如维护或过载）         |

#### 6.2 统一错误响应格式

所有 API 在发生错误时，都应返回统一格式的错误信息：

```json
{
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "人类可读的错误描述信息",
    "details": {
      "field1": "字段1的错误详情",
      "field2": "字段2的错误详情"
    },
    "requestId": "req-12345678"
  }
}
```

#### 6.3 通用错误码

以下通用错误码适用于所有游戏数据访问 API：

| 错误码                  | HTTP 状态码 | 描述                  |
| ----------------------- | ----------- | --------------------- |
| INVALID_REQUEST         | 400         | 无效的请求格式或结构  |
| INVALID_PARAMETER       | 400         | 无效的请求参数        |
| MISSING_PARAMETER       | 400         | 缺少必需的参数        |
| AUTHENTICATION_REQUIRED | 401         | 需要身份验证          |
| INVALID_AUTHENTICATION  | 401         | 无效的身份验证凭证    |
| PERMISSION_DENIED       | 403         | 权限不足              |
| RESOURCE_NOT_FOUND      | 404         | 请求的资源不存在      |
| RATE_LIMIT_EXCEEDED     | 429         | 超过 API 调用频率限制 |
| INTERNAL_ERROR          | 500         | 服务器内部错误        |
| SERVICE_UNAVAILABLE     | 503         | 服务暂时不可用        |

#### 6.4 业务特定错误码

除了通用错误码外，不同的 API 模块还有其特定的业务错误码：

##### 游戏房间相关错误码

| 错误码                  | HTTP 状态码 | 描述                       |
| ----------------------- | ----------- | -------------------------- |
| ROOM_NOT_FOUND          | 404         | 游戏房间不存在             |
| ROOM_FULL               | 409         | 游戏房间已满               |
| INVALID_ROOM_STATUS     | 400         | 游戏房间状态不允许当前操作 |
| INVALID_INVITATION_CODE | 400         | 无效的邀请码               |
| PLAYER_ALREADY_IN_ROOM  | 409         | 玩家已在其他房间中         |

##### 游戏记录相关错误码

| 错误码                    | HTTP 状态码 | 描述               |
| ------------------------- | ----------- | ------------------ |
| GAME_NOT_FOUND            | 404         | 游戏记录不存在     |
| GAME_HISTORY_UNAVAILABLE  | 404         | 游戏历史记录不可用 |
| INSUFFICIENT_PLAY_RECORDS | 404         | 玩家游戏记录不足   |

##### 游戏状态相关错误码

| 错误码                     | HTTP 状态码 | 描述                      |
| -------------------------- | ----------- | ------------------------- |
| GAME_STATE_UNAVAILABLE     | 404         | 游戏状态不可用            |
| WEBSOCKET_CONNECTION_LIMIT | 429         | 达到 WebSocket 连接数限制 |
| INVALID_SUBSCRIPTION       | 400         | 无效的订阅请求            |

##### 数据统计相关错误码

| 错误码               | HTTP 状态码 | 描述                 |
| -------------------- | ----------- | -------------------- |
| STATS_NOT_AVAILABLE  | 404         | 请求的统计数据不可用 |
| INSUFFICIENT_DATA    | 404         | 数据量不足以进行统计 |
| ROLE_NOT_FOUND       | 404         | 角色不存在           |
| DATE_RANGE_TOO_LARGE | 400         | 日期范围过大         |

##### 游戏回放相关错误码

| 错误码                    | HTTP 状态码 | 描述                   |
| ------------------------- | ----------- | ---------------------- |
| REPLAY_NOT_AVAILABLE      | 404         | 回放数据不可用         |
| SNAPSHOT_NOT_FOUND        | 404         | 指定的快照时间点不存在 |
| REPLAY_PROCESSING         | 409         | 回放数据仍在处理中     |
| REPLAY_FORMAT_UNSUPPORTED | 400         | 不支持的回放格式       |

#### 6.5 错误处理最佳实践

1. **提供明确的错误信息**：错误消息应清晰描述问题和可能的解决方案
2. **包含唯一的请求 ID**：便于跟踪和调试问题
3. **适当的日志级别**：根据错误严重性记录不同级别的日志
4. **不泄露敏感信息**：错误信息不应包含敏感数据或内部实现细节
5. **客户端重试策略**：提供适当的重试指导（如 Retry-After 头部）
6. **多语言支持**：错误消息应支持国际化和本地化

### 7. API 性能与优化措施

为确保游戏数据访问 API 能够满足高并发、低延迟的性能要求，同时保持系统资源的高效利用，我们采用以下性能优化措施。

#### 7.1 缓存策略

##### 多级缓存

1. **内存缓存（Redis）**

   - 热点数据（活跃游戏房间、当日排行榜等）
   - 访问频率高的只读数据
   - 缓存时间：根据数据更新频率设置，从 30 秒到 1 小时不等

2. **CDN 缓存**

   - 静态资源（游戏配置、角色图片等）
   - 公共数据（全局统计信息、历史排行榜等）
   - 缓存时间：静态资源最长 7 天，公共数据最长 1 小时

3. **客户端缓存**
   - 用户个人数据
   - 游戏记录摘要
   - 缓存策略通过 HTTP Cache-Control 头控制

##### 缓存一致性保障

1. 基于 TTL（Time-To-Live）的过期策略
2. 写操作触发的缓存更新机制
3. 缓存键设计包含版本号，支持版本化缓存
4. 使用消息队列进行缓存失效通知

#### 7.2 数据库优化

1. **读写分离**

   - 主库处理写入操作
   - 只读副本处理查询操作，降低主库负载

2. **分片策略**

   - 按游戏 ID 对游戏记录和回放数据进行水平分片
   - 按用户 ID 对用户游戏历史数据进行分片
   - 使用一致性哈希算法确保数据分布均匀

3. **索引优化**

   - 为常用查询条件创建合适的索引
   - 复合索引优化多条件查询
   - 定期分析慢查询并优化

4. **连接池管理**
   - 动态调整连接池大小
   - 监控连接使用情况
   - 设置合理的连接超时和重试策略

#### 7.3 API 设计优化

1. **批量操作支持**

   - 提供批量获取 API，减少网络往返
   - 支持部分更新操作，减少传输数据量

2. **字段筛选**

   - 通过`fields`参数允许客户端指定需要返回的字段
   - 减少不必要的数据传输

3. **压缩传输**

   - 启用 gzip/deflate 压缩
   - 大型响应（如回放数据包）自动压缩

4. **分页与限流**
   - 所有列表类 API 支持分页
   - 基于用户级别的 API 调用频率限制
   - 针对敏感资源的访问限流

#### 7.4 异步处理

1. **消息队列**

   - 使用 RabbitMQ 处理非实时写入操作
   - 数据统计和分析任务异步处理

2. **计划任务**

   - 排行榜数据定时计算更新
   - 历史数据归档和清理

3. **WebSocket 优化**
   - 连接池管理
   - 心跳机制降低无效连接
   - 消息批处理和优先级队列

#### 7.5 监控与预警

1. **性能指标监控**

   - API 响应时间
   - 错误率
   - 缓存命中率
   - 数据库负载

2. **自动扩缩容**

   - 基于负载指标的服务自动扩缩容
   - 读库自动扩展

3. **智能预警**
   - 基于历史数据的异常检测
   - 多级别预警策略
   - 自动恢复机制

#### 7.6 性能要求指标

| API 类别     | 平均响应时间 | 95%响应时间 | 99%响应时间 | 最大并发 |
| ------------ | ------------ | ----------- | ----------- | -------- |
| 游戏房间查询 | < 20ms       | < 50ms      | < 100ms     | 500 QPS  |
| 游戏历史记录 | < 30ms       | < 80ms      | < 150ms     | 300 QPS  |
| 实时状态获取 | < 10ms       | < 30ms      | < 50ms      | 1000 QPS |
| 数据统计查询 | < 50ms       | < 120ms     | < 200ms     | 200 QPS  |
| 回放数据获取 | < 100ms      | < 300ms     | < 500ms     | 100 QPS  |

## 实现细节

### 技术栈选择

1. **后端框架**：Node.js + Express.js

   - 适合处理高并发 I/O 密集型 API
   - 非阻塞异步模型
   - 丰富的中间件生态

2. **数据库**：

   - 主数据库：MongoDB
     - 适合存储复杂的游戏数据结构
     - 支持水平扩展
     - 灵活的查询能力
   - 缓存：Redis
     - 高性能内存数据库
     - 支持多种数据结构
     - 原子操作支持

3. **实时通信**：

   - Socket.IO
     - 提供可靠的 WebSocket 抽象
     - 自动降级和重连机制
     - 支持房间和命名空间

4. **消息队列**：

   - RabbitMQ
     - 可靠的消息传递
     - 支持多种消息模式
     - 高吞吐量

5. **监控和日志**：
   - Prometheus + Grafana
   - ELK Stack (Elasticsearch, Logstash, Kibana)

### 核心模块设计

#### API 网关层

1. **请求验证与鉴权**

   - JWT 验证
   - 访问控制列表(ACL)
   - 请求体积和速率限制

2. **路由分发**

   - API 版本管理
   - 服务发现与负载均衡
   - 请求追踪

3. **响应处理**
   - 统一响应格式
   - 错误处理
   - 内容协商（JSON, MessagePack）

#### 数据访问层

1. **数据模型**

   - 游戏房间模型
   - 游戏记录模型
   - 用户统计模型
   - 回放数据模型

2. **查询优化器**

   - 智能索引选择
   - 查询重写
   - 结果缓存

3. **数据聚合服务**
   - 实时统计
   - 历史数据分析
   - 趋势计算

#### 实时同步层

1. **连接管理**

   - 会话状态维护
   - 自动重连处理
   - 负载均衡

2. **事件处理**

   - 事件过滤
   - 事件优先级
   - 批量事件发送

3. **状态同步**
   - 增量状态更新
   - 冲突解决
   - 断线重连恢复

### 代码组织结构

```
/src
  /api
    /v1
      /rooms        # 游戏房间API
      /history      # 游戏历史记录API
      /state        # 游戏状态API
      /stats        # 游戏统计API
      /replays      # 游戏回放API
  /models           # 数据模型
  /services         # 业务逻辑
  /utils            # 工具函数
  /middleware       # 中间件
  /config           # 配置文件
  /socket           # WebSocket处理
  /workers          # 后台工作进程
  /db               # 数据库连接
  /caches           # 缓存管理
  /queues           # 消息队列
```

### 部署架构

1. **多区域部署**

   - 主要区域：华东、华南、华北
   - 跨区域数据同步
   - 智能流量路由

2. **容器化部署**

   - Kubernetes 编排
   - Docker 容器
   - Helm Charts

3. **自动化运维**

   - CI/CD 流水线
   - 自动化测试
   - 蓝绿部署

4. **伸缩策略**
   - 水平伸缩
   - 自动伸缩组
   - 冷热数据分层

## 测试计划

### 单元测试

1. **测试覆盖率目标**: 代码覆盖率 > 90%
2. **测试框架**: Jest
3. **测试范围**:
   - 数据模型验证
   - 服务层业务逻辑
   - 工具函数
   - 中间件

### 集成测试

1. **API 测试**

   - REST API 端点测试
   - WebSocket 接口测试
   - 使用 Supertest 和 Socket.IO-client

2. **数据流测试**

   - 数据库操作
   - 缓存交互
   - 消息队列集成

3. **认证与授权测试**
   - 权限控制
   - 令牌验证
   - 访问限制

### 性能测试

1. **负载测试**

   - 工具: k6, Artillery
   - 场景:
     - 普通用户流量 (100 RPS)
     - 节假日峰值 (500 RPS)
     - 极限负载 (1000 RPS)

2. **耐久性测试**

   - 持续运行 24 小时
   - 模拟真实用户行为
   - 监控资源使用

3. **扩展性测试**
   - 集群扩展测试
   - 数据库分片效果测试
   - 跨区域部署测试

### 安全测试

1. **渗透测试**

   - API 端点安全
   - 认证机制
   - 数据访问控制

2. **数据验证测试**

   - 输入验证
   - 敏感数据处理
   - 输出过滤

3. **依赖项安全扫描**
   - NPM 审计
   - Docker 镜像扫描
   - 第三方库漏洞检查

### 验收测试

1. **功能验收测试**

   - 全流程用户场景
   - 边界条件测试
   - 错误处理测试

2. **兼容性测试**

   - 不同客户端版本
   - 不同网络条件
   - 跨平台测试

3. **监控与观测性测试**
   - 日志记录完整性
   - 指标收集准确性
   - 告警触发有效性

### 测试环境

1. **开发环境**: 本地开发测试
2. **测试环境**: 功能和集成测试
3. **预生产环境**: 性能和负载测试
4. **生产环境**: 灰度发布和验证

### 测试自动化

1. **CI/CD 集成**

   - 提交触发单元测试
   - 合并触发集成测试
   - 发布前运行性能测试

2. **自动化报告**

   - 测试覆盖率报告
   - 性能测试报告
   - 安全扫描报告

3. **测试数据管理**
   - 测试数据生成器
   - 环境重置脚本
   - 数据快照和恢复

## 扩展性考虑

### 横向扩展能力

1. **无状态设计**

   - API 服务无本地状态
   - 会话状态存储在 Redis
   - 支持任意节点扩展

2. **分片策略**

   - 游戏数据按游戏 ID 分片
   - 用户数据按用户 ID 分片
   - 支持动态重平衡

3. **流量控制**
   - 负载均衡器智能路由
   - 服务发现自动注册
   - 流量调节和限流

### 版本兼容性

1. **API 版本控制**

   - URL 路径版本 (/api/v1/...)
   - 支持多版本并行运行
   - 版本废弃策略

2. **向后兼容**

   - 新增字段默认值处理
   - 保留废弃字段支持
   - 客户端版本检测

3. **数据迁移**
   - 增量数据模型更新
   - 后台数据迁移服务
   - 双写过渡期

### 功能扩展预留

1. **可插拔式设计**

   - 模块化 API 集合
   - 中间件扩展点
   - 基于配置的功能切换

2. **事件驱动架构**

   - 基于消息的服务间通信
   - 发布/订阅模式
   - 事件溯源支持

3. **第三方集成**
   - 身份认证服务集成
   - 分析和监控工具集成
   - 支付和通知系统集成

### 数据增长规划

1. **存储扩展策略**

   - 冷热数据分离
   - 数据归档机制
   - 时序数据优化

2. **查询性能保障**

   - 索引策略定期评估
   - 查询复杂度限制
   - 大批量查询保护

3. **统计数据策略**
   - 预计算统计聚合
   - 采样数据分析
   - 历史数据压缩

### 未来技术适配

1. **新平台支持**

   - 小程序平台更新适配
   - 新游戏客户端支持
   - 第三方应用集成

2. **新通信协议**

   - HTTP/3 支持准备
   - gRPC 服务支持
   - GraphQL 查询扩展

3. **新分析能力**
   - 游戏行为分析
   - AI 辅助策略推荐
   - 个性化内容

### 全球化支持

1. **多语言支持**

   - 错误消息国际化
   - 时区处理
   - 本地化格式

2. **全球部署**

   - 多区域数据中心
   - 地理位置路由
   - GDPR 合规支持

3. **跨境数据传输**
   - 数据本地化存储
   - 区域隔离选项
   - 数据主权合规
