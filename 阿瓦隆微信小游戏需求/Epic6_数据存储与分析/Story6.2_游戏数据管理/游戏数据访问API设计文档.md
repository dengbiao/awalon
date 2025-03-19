# 游戏数据访问 API 设计文档

## 文档概述

本文档详细描述阿瓦隆微信小游戏数据访问 API 的设计与实现方案，包括游戏房间查询、历史记录获取、实时状态同步、数据统计以及游戏回放功能的 API 设计。

## 目录

1. [简介](#简介)
2. [游戏房间查询和过滤 API](#游戏房间查询和过滤-api)
3. [游戏历史记录查询接口](#游戏历史记录查询接口)
4. [游戏状态实时获取接口](#游戏状态实时获取接口)
5. [游戏数据统计聚合接口](#游戏数据统计聚合接口)
6. [游戏回放数据获取 API](#游戏回放数据获取-api)
7. [API 安全与性能考量](#api-安全与性能考量)
8. [附录](#附录)

## 简介

### 背景

阿瓦隆是一款基于微信小游戏平台的多人在线角色扮演游戏，需要高效地管理和访问游戏数据，包括房间信息、游戏进度、历史记录等。本文档描述了游戏数据访问 API 的设计，这些 API 将支持游戏的核心功能，确保游戏体验的流畅性和数据的一致性。

### 目标

1. 提供统一、安全的游戏数据访问接口
2. 支持高效的游戏房间创建、查询和过滤
3. 确保游戏历史记录的完整保存和高效检索
4. 实现游戏状态的实时同步和获取
5. 支持游戏数据的统计分析与可视化
6. 提供游戏回放功能所需的数据支持

### 适用范围

本 API 设计适用于阿瓦隆微信小游戏的前端界面、后端服务以及管理后台。所有与游戏数据相关的交互都应通过本文档定义的 API 进行。

### API 设计原则

1. **RESTful 架构**：API 遵循 REST 规范设计，使用标准 HTTP 方法
2. **轻量级交互**：JSON 格式数据交换，优化传输效率
3. **版本控制**：API 路径包含版本号，确保向后兼容性
4. **安全性**：采用 HTTPS、令牌验证和权限控制
5. **可扩展性**：设计支持未来功能扩展的结构
6. **可测试性**：提供明确的接口规范，便于测试和验证

## 游戏房间查询和过滤 API

### 概述

游戏房间查询和过滤 API 提供了一套完整的接口，用于创建、查询、过滤和管理游戏房间。这些接口支持玩家快速找到合适的游戏房间，或者创建新的游戏房间。

### API 端点

#### 1. 获取游戏房间列表

**请求**:

```
GET /api/v1/rooms
```

**查询参数**:

- `status`: 房间状态（waiting/playing/finished）
- `playerCount`: 房间当前玩家数量
- `capacity`: 房间容量
- `gameMode`: 游戏模式
- `createdBy`: 创建者 ID
- `page`: 分页页码 (默认: 1)
- `limit`: 每页数量 (默认: 20，最大: 50)
- `sort`: 排序方式 (例如: createdAt:desc)

**响应**:

```json
{
  "status": "success",
  "data": {
    "rooms": [
      {
        "roomId": "room123456",
        "name": "阿瓦隆欢乐局",
        "status": "waiting",
        "playerCount": 5,
        "capacity": 8,
        "gameMode": "standard",
        "createdAt": "2023-05-15T08:30:45Z",
        "createdBy": "user789012",
        "tags": ["新手友好", "语音"],
        "isPrivate": false
      }
      // 更多房间...
    ],
    "pagination": {
      "total": 120,
      "page": 1,
      "limit": 20,
      "pages": 6
    }
  }
}
```

#### 2. 获取单个游戏房间详情

**请求**:

```
GET /api/v1/rooms/{roomId}
```

**路径参数**:

- `roomId`: 游戏房间唯一标识符

**响应**:

```json
{
  "status": "success",
  "data": {
    "room": {
      "roomId": "room123456",
      "name": "阿瓦隆欢乐局",
      "status": "waiting",
      "playerCount": 5,
      "capacity": 8,
      "gameMode": "standard",
      "createdAt": "2023-05-15T08:30:45Z",
      "updatedAt": "2023-05-15T08:45:12Z",
      "createdBy": "user789012",
      "tags": ["新手友好", "语音"],
      "isPrivate": false,
      "password": null,
      "players": [
        {
          "userId": "user789012",
          "nickname": "游戏大师",
          "role": "host",
          "joinedAt": "2023-05-15T08:30:45Z",
          "status": "ready"
        }
        // 更多玩家...
      ],
      "settings": {
        "timeLimit": 60,
        "roleConfig": {
          "merlin": true,
          "percival": true,
          "morgana": true
          // 更多角色配置...
        }
      }
    }
  }
}
```

#### 3. 创建游戏房间

**请求**:

```
POST /api/v1/rooms
```

**请求体**:

```json
{
  "name": "阿瓦隆欢乐局",
  "capacity": 8,
  "gameMode": "standard",
  "isPrivate": false,
  "password": null,
  "tags": ["新手友好", "语音"],
  "settings": {
    "timeLimit": 60,
    "roleConfig": {
      "merlin": true,
      "percival": true,
      "morgana": true
      // 更多角色配置...
    }
  }
}
```

**响应**:

```json
{
  "status": "success",
  "data": {
    "roomId": "room123456",
    "joinCode": "ABCDEF",
    "createdAt": "2023-05-15T08:30:45Z"
  }
}
```

#### 4. 快速加入游戏房间

**请求**:

```
POST /api/v1/rooms/quickjoin
```

**请求体**:

```json
{
  "gameMode": "standard",
  "preferredCapacity": 8,
  "excludePrivate": true
}
```

**响应**:

```json
{
  "status": "success",
  "data": {
    "roomId": "room123456",
    "name": "阿瓦隆欢乐局",
    "playerCount": 5,
    "capacity": 8
  }
}
```

#### 5. 使用邀请码加入游戏房间

**请求**:

```
POST /api/v1/rooms/join
```

**请求体**:

```json
{
  "joinCode": "ABCDEF",
  "password": "optional-password"
}
```

**响应**:

```json
{
  "status": "success",
  "data": {
    "roomId": "room123456",
    "name": "阿瓦隆欢乐局",
    "playerCount": 6,
    "capacity": 8
  }
}
```

### 错误处理

所有游戏房间 API 遵循统一的错误响应格式：

```json
{
  "status": "error",
  "error": {
    "code": "ROOM_NOT_FOUND",
    "message": "找不到指定的游戏房间",
    "details": {
      "roomId": "invalid-room-id"
    }
  }
}
```

**常见错误代码**:

- `ROOM_NOT_FOUND`: 游戏房间不存在
- `ROOM_FULL`: 游戏房间已满
- `INVALID_PASSWORD`: 私密房间密码错误
- `ROOM_CLOSED`: 游戏房间已关闭
- `INVALID_PARAMETERS`: 请求参数无效
- `UNAUTHORIZED`: 未授权操作

## 游戏历史记录查询接口

### 概述

游戏历史记录查询接口提供了一套完整的 API，用于查询和检索游戏历史记录。这些接口使玩家能够查看自己的游戏历史、特定游戏的详细信息、游戏统计数据等。历史记录查询对于玩家分析自己的游戏表现、学习游戏策略以及回顾精彩时刻非常重要。

### API 端点

#### 1. 获取用户游戏历史列表

**请求**:

```
GET /api/v1/history/user/{userId}
```

**路径参数**:

- `userId`: 用户唯一标识符

**查询参数**:

- `gameMode`: 游戏模式
- `result`: 游戏结果（win/lose/draw）
- `role`: 玩家角色
- `startDate`: 开始日期（ISO 8601 格式）
- `endDate`: 结束日期（ISO 8601 格式）
- `page`: 分页页码 (默认: 1)
- `limit`: 每页数量 (默认: 20，最大: 50)
- `sort`: 排序方式 (例如: playedAt:desc)

**响应**:

```json
{
  "status": "success",
  "data": {
    "games": [
      {
        "gameId": "game789012",
        "roomId": "room123456",
        "roomName": "阿瓦隆欢乐局",
        "gameMode": "standard",
        "startedAt": "2023-05-15T09:00:00Z",
        "endedAt": "2023-05-15T09:35:27Z",
        "playerCount": 8,
        "userRole": "merlin",
        "result": "win",
        "team": "good",
        "score": 3,
        "playerRank": 1
      }
      // 更多游戏记录...
    ],
    "pagination": {
      "total": 85,
      "page": 1,
      "limit": 20,
      "pages": 5
    }
  }
}
```

#### 2. 获取特定游戏记录详情

**请求**:

```
GET /api/v1/history/games/{gameId}
```

**路径参数**:

- `gameId`: 游戏唯一标识符

**响应**:

```json
{
  "status": "success",
  "data": {
    "game": {
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
          "score": 125,
          "actions": 18
        }
        // 更多玩家...
      ],
      "rounds": [
        {
          "roundNumber": 1,
          "leader": "user789012",
          "team": ["user789012", "user345678", "user567890"],
          "votes": {
            "user789012": true,
            "user345678": true
            // 更多投票...
          },
          "result": "success",
          "duration": 320
        }
        // 更多回合...
      ],
      "events": [
        {
          "timestamp": "2023-05-15T09:02:15Z",
          "type": "ROUND_START",
          "data": {
            "roundNumber": 1,
            "leader": "user789012"
          }
        }
        // 更多事件...
      ]
    }
  }
}
```

#### 3. 获取用户游戏统计数据

**请求**:

```
GET /api/v1/history/user/{userId}/stats
```

**路径参数**:

- `userId`: 用户唯一标识符

**查询参数**:

- `gameMode`: 游戏模式
- `period`: 统计周期（all/month/week）
- `startDate`: 开始日期（ISO 8601 格式）
- `endDate`: 结束日期（ISO 8601 格式）

**响应**:

```json
{
  "status": "success",
  "data": {
    "stats": {
      "totalGames": 127,
      "winRate": 0.68,
      "averageScore": 115.8,
      "totalPlayTime": 156840,
      "favoriteRole": "merlin",
      "roleStats": {
        "merlin": {
          "played": 35,
          "winRate": 0.74,
          "averageScore": 125.3
        },
        "assassin": {
          "played": 28,
          "winRate": 0.61,
          "averageScore": 108.7
        }
        // 更多角色统计...
      },
      "teamStats": {
        "good": {
          "played": 72,
          "winRate": 0.65,
          "averageScore": 119.2
        },
        "evil": {
          "played": 55,
          "winRate": 0.72,
          "averageScore": 111.5
        }
      },
      "achievements": [
        {
          "id": "perfect_game",
          "name": "完美游戏",
          "count": 5,
          "lastEarned": "2023-05-10T14:23:18Z"
        }
        // 更多成就...
      ]
    }
  }
}
```

#### 4. 获取游戏房间的历史记录

**请求**:

```
GET /api/v1/history/rooms/{roomId}
```

**路径参数**:

- `roomId`: 房间唯一标识符

**查询参数**:

- `page`: 分页页码 (默认: 1)
- `limit`: 每页数量 (默认: 20，最大: 50)
- `sort`: 排序方式 (例如: playedAt:desc)

**响应**:

```json
{
  "status": "success",
  "data": {
    "room": {
      "roomId": "room123456",
      "name": "阿瓦隆欢乐局",
      "createdAt": "2023-05-15T08:30:45Z",
      "createdBy": "user789012",
      "totalGames": 15
    },
    "games": [
      {
        "gameId": "game789012",
        "gameMode": "standard",
        "startedAt": "2023-05-15T09:00:00Z",
        "endedAt": "2023-05-15T09:35:27Z",
        "playerCount": 8,
        "winner": "good",
        "duration": 2127
      }
      // 更多游戏...
    ],
    "pagination": {
      "total": 15,
      "page": 1,
      "limit": 20,
      "pages": 1
    }
  }
}
```

### 错误处理

所有游戏历史记录查询 API 遵循统一的错误响应格式：

```json
{
  "status": "error",
  "error": {
    "code": "HISTORY_NOT_FOUND",
    "message": "找不到指定的游戏历史记录",
    "details": {
      "gameId": "invalid-game-id"
    }
  }
}
```

**常见错误代码**:

- `HISTORY_NOT_FOUND`: 游戏历史记录不存在
- `USER_NOT_FOUND`: 用户不存在
- `ROOM_NOT_FOUND`: 游戏房间不存在
- `INVALID_PARAMETERS`: 请求参数无效
- `UNAUTHORIZED`: 未授权操作
- `FORBIDDEN`: 禁止访问他人私密游戏记录

## 游戏状态实时获取接口

### 概述

游戏状态实时获取接口提供了一套实时访问和更新游戏状态的 API。这些接口使游戏客户端能够获取最新的游戏状态、接收实时更新通知，以及确保在网络波动或断线重连场景下游戏状态的一致性。本节 API 设计同时考虑了 HTTP 轮询和 WebSocket 推送两种实现方式。

### API 端点

#### 1. 获取当前游戏状态

**请求**:

```
GET /api/v1/games/{gameId}/state
```

**路径参数**:

- `gameId`: 游戏唯一标识符

**查询参数**:

- `version`: 客户端当前持有的游戏状态版本号，用于增量更新

**响应**:

```json
{
  "status": "success",
  "data": {
    "gameState": {
      "gameId": "game789012",
      "version": 35,
      "status": "playing",
      "currentRound": 3,
      "currentPhase": "team_selection",
      "leader": "user789012",
      "remainingTime": 45,
      "players": [
        {
          "userId": "user789012",
          "nickname": "游戏大师",
          "role": "merlin",
          "isLeader": true,
          "status": "active"
        }
        // 更多玩家...
      ],
      "rounds": [
        {
          "roundNumber": 1,
          "result": "success",
          "leader": "user345678",
          "team": ["user345678", "user567890", "user789012"]
        },
        {
          "roundNumber": 2,
          "result": "fail",
          "leader": "user567890",
          "team": ["user567890", "user123456", "user789012"]
        }
      ],
      "currentTeam": [],
      "votes": {},
      "missionResults": [true, false],
      "visibilityMap": {
        "user789012": ["user123456", "user456789"]
        // 更多可见性规则...
      },
      "gameSpecificState": {
        // 游戏特定状态数据...
      }
    }
  }
}
```

#### 2. 订阅游戏状态更新 (WebSocket)

**连接**:

```
WebSocket: /ws/games/{gameId}?token={authToken}
```

**路径参数**:

- `gameId`: 游戏唯一标识符

**查询参数**:

- `token`: 身份验证令牌

**初始化消息** (服务器->客户端):

```json
{
  "type": "CONNECTED",
  "data": {
    "gameId": "game789012",
    "userId": "user789012",
    "timestamp": "2023-05-15T09:05:30Z"
  }
}
```

**状态更新消息** (服务器->客户端):

```json
{
  "type": "STATE_UPDATE",
  "data": {
    "version": 36,
    "changes": [
      {
        "path": "currentPhase",
        "value": "team_voting"
      },
      {
        "path": "currentTeam",
        "value": ["user789012", "user345678", "user567890"]
      },
      {
        "path": "remainingTime",
        "value": 60
      }
    ],
    "timestamp": "2023-05-15T09:06:15Z"
  }
}
```

**客户端行动消息** (客户端->服务器):

```json
{
  "type": "ACTION",
  "data": {
    "action": "VOTE",
    "payload": {
      "vote": true
    },
    "gameVersion": 36,
    "timestamp": "2023-05-15T09:06:40Z"
  }
}
```

#### 3. 游戏状态事件通知

**WebSocket 事件类型**:

| 事件类型        | 描述         | 方向             |
| --------------- | ------------ | ---------------- |
| `CONNECTED`     | 连接建立     | 服务器 -> 客户端 |
| `STATE_UPDATE`  | 游戏状态更新 | 服务器 -> 客户端 |
| `ACTION`        | 玩家行动     | 客户端 -> 服务器 |
| `ACTION_RESULT` | 行动结果     | 服务器 -> 客户端 |
| `GAME_EVENT`    | 游戏事件     | 服务器 -> 客户端 |
| `CHAT_MESSAGE`  | 聊天消息     | 双向             |
| `ERROR`         | 错误通知     | 服务器 -> 客户端 |
| `PING`          | 心跳检测     | 双向             |
| `RECONNECT`     | 重连请求     | 客户端 -> 服务器 |

**游戏事件消息** (服务器->客户端):

```json
{
  "type": "GAME_EVENT",
  "data": {
    "eventType": "ROLE_REVEAL",
    "payload": {
      "targetUserId": "user123456",
      "role": "assassin"
    },
    "timestamp": "2023-05-15T09:15:22Z"
  }
}
```

**行动结果消息** (服务器->客户端):

```json
{
  "type": "ACTION_RESULT",
  "data": {
    "actionId": "action123456",
    "status": "success",
    "action": "VOTE",
    "result": {
      "voteAccepted": true,
      "voteCount": {
        "approve": 5,
        "reject": 3
      },
      "teamApproved": true
    },
    "timestamp": "2023-05-15T09:06:45Z"
  }
}
```

#### 4. 断线重连状态恢复

**请求**:

```
POST /api/v1/games/{gameId}/reconnect
```

**路径参数**:

- `gameId`: 游戏唯一标识符

**请求体**:

```json
{
  "lastKnownVersion": 25,
  "disconnectTime": "2023-05-15T09:00:15Z"
}
```

**响应**:

```json
{
  "status": "success",
  "data": {
    "currentVersion": 35,
    "stateUpdates": [
      {
        "version": 26,
        "changes": [
          {
            "path": "currentRound",
            "value": 2
          },
          {
            "path": "leader",
            "value": "user567890"
          }
        ],
        "timestamp": "2023-05-15T09:01:10Z"
      }
      // 更多中间更新...
    ],
    "fullState": {
      // 完整当前状态，同获取当前游戏状态 API
    },
    "missedEvents": [
      {
        "type": "GAME_EVENT",
        "data": {
          "eventType": "MISSION_RESULT",
          "payload": {
            "roundNumber": 1,
            "result": "success"
          },
          "timestamp": "2023-05-15T09:01:05Z"
        }
      }
      // 更多错过的事件...
    ]
  }
}
```

### WebSocket 连接管理策略

1. **心跳机制**：客户端和服务器之间每 30 秒交换一次 PING 消息，确认连接活跃
2. **自动重连**：连接断开后，客户端应实现指数退避重连策略
3. **会话保持**：服务器应在一定时间内（默认 5 分钟）保留断开连接的会话状态
4. **状态同步**：重连后，服务器发送断开期间的增量更新或完整状态

### 错误处理

WebSocket 错误消息格式：

```json
{
  "type": "ERROR",
  "data": {
    "code": "INVALID_ACTION",
    "message": "当前阶段不允许该操作",
    "details": {
      "action": "VOTE",
      "currentPhase": "team_selection"
    },
    "timestamp": "2023-05-15T09:07:02Z"
  }
}
```

**常见错误代码**:

- `INVALID_ACTION`: 不合法的操作
- `UNAUTHORIZED`: 未授权操作
- `VERSION_CONFLICT`: 版本冲突
- `GAME_NOT_FOUND`: 游戏不存在
- `CONNECTION_LIMIT`: 连接数量限制
- `RATE_LIMIT`: 请求频率限制

## 游戏数据统计聚合接口

### 概述

游戏数据统计聚合接口提供了一系列用于查询和分析游戏统计数据的 API。这些接口支持查询全局游戏统计、角色表现数据、玩家排行榜以及游戏趋势分析等。统计数据对于游戏平衡性调整、玩家行为分析和游戏体验优化具有重要价值。

### API 端点

#### 1. 获取全局游戏统计数据

**请求**:

```
GET /api/v1/stats/global
```

**查询参数**:

- `period`: 统计周期（all/year/month/week/day）
- `gameMode`: 游戏模式
- `startDate`: 开始日期（ISO 8601 格式）
- `endDate`: 结束日期（ISO 8601 格式）

**响应**:

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

## 游戏回放数据获取 API

待完成

## API 安全与性能考量

待完成

## 附录

待完成
