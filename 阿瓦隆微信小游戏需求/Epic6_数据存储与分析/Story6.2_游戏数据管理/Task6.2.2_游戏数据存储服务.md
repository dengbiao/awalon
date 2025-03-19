# Task 6.2.2: 游戏数据存储服务

## 1. 任务描述

本任务旨在实现阿瓦隆游戏的数据存储服务，负责处理游戏数据的创建、更新、存储和查询等核心功能。该服务将作为游戏数据管理的基础设施，支持游戏房间管理、游戏进程数据存储、历史记录归档等功能，并确保数据的一致性、可靠性和高效访问。

## 2. 功能需求

### 2.1 游戏数据创建和更新服务

- **房间创建服务**：
  - 创建新的游戏房间记录
  - 生成唯一房间标识
  - 设置房间初始配置
  - 验证房间创建请求的合法性
- **房间更新服务**：

  - 更新房间状态（等待中、游戏中、已结束）
  - 管理房间成员信息
  - 修改房间配置
  - 处理房间锁定/解锁

- **游戏状态更新服务**：
  - 记录游戏进程中的状态变化
  - 保存玩家操作和系统事件
  - 维护游戏当前状态
  - 确保状态更新的原子性

### 2.2 游戏房间管理功能

- **房间生命周期管理**：

  - 创建房间
  - 启动游戏
  - 游戏进行中状态维护
  - 游戏结束和房间关闭
  - 超时房间清理

- **房间成员管理**：

  - 玩家加入房间
  - 玩家离开房间
  - 房主转移
  - 成员角色分配和变更

- **房间配置管理**：
  - 游戏规则设置
  - 房间可见性控制
  - 进入密码设置
  - 房间容量限制

### 2.3 游戏进程数据的增量存储

- **事件流存储**：

  - 记录游戏中的每个关键事件
  - 确保事件按时间顺序存储
  - 支持事件的批量写入
  - 提供事件查询接口

- **状态快照管理**：

  - 定期创建游戏状态快照
  - 根据事件和快照重建状态
  - 优化存储空间和恢复效率
  - 实现增量与全量备份策略

- **并发控制机制**：
  - 实现乐观锁控制
  - 版本号冲突检测
  - 合并策略定义
  - 事务一致性保障

### 2.4 游戏历史记录归档系统

- **游戏完成归档**：

  - 游戏结束后自动归档
  - 压缩和优化存储格式
  - 计算并存储游戏统计数据
  - 关联玩家档案

- **批量归档处理**：

  - 历史数据定期批量归档
  - 冷热数据分离存储
  - 归档数据完整性校验
  - 归档任务调度和监控

- **归档数据索引**：
  - 创建高效查询索引
  - 支持多维度检索
  - 优化聚合查询性能
  - 索引定期维护机制

### 2.5 高效数据查询机制

- **优化查询模式**：

  - 设计复合索引
  - 查询缓存策略
  - 分页和游标支持
  - 投影和字段过滤

- **实时查询优化**：

  - 实时游戏状态快速查询
  - 缓存预热策略
  - 读写分离机制
  - 查询性能监控

- **聚合查询支持**：
  - 玩家统计数据计算
  - 游戏结果聚合分析
  - 批量数据导出功能
  - 复杂查询条件支持

## 3. 技术规格

### 3.1 数据库架构

- **主数据库**：MongoDB 4.4+

  - 副本集配置：1 主 2 从
  - 数据分片策略：按游戏 ID 和时间范围
  - 索引设计：根据常用查询模式优化
  - 写入关注级别：majority

- **缓存层**：Redis 6.0+

  - 集群模式：Redis Sentinel
  - 数据持久化：RDB + AOF
  - 缓存策略：LRU + TTL
  - 过期清理：异步定期清理

- **存储类型划分**：
  - 活跃游戏数据：Redis
  - 当前游戏状态：MongoDB (实时集合)
  - 历史游戏记录：MongoDB (归档集合)
  - 统计聚合数据：MongoDB (专用集合)

### 3.2 API 规格

- **数据服务 API**：

  - 基于 RESTful 设计
  - JSON 数据格式
  - 支持批量操作
  - 遵循 OpenAPI 3.0 规范

- **权限控制**：

  - 基于角色的访问控制
  - API 密钥认证
  - 操作审计日志
  - 资源级权限粒度

- **性能要求**：
  - 读取操作: P95 < 50ms
  - 写入操作: P95 < 100ms
  - 批量操作: 吞吐量 > 1000 ops/s
  - 并发支持: > 500 同时连接

### 3.3 代码规范与架构

- **代码组织**：

  - 分层架构：Repository -> Service -> Controller
  - 依赖注入模式
  - 单一职责原则
  - 接口隔离

- **错误处理**：

  - 统一错误码体系
  - 详细错误信息
  - 错误日志记录
  - 优雅降级策略

- **测试要求**：
  - 单元测试覆盖率 > 80%
  - 集成测试覆盖关键路径
  - 性能基准测试
  - 容错性测试

## 4. 实现细节

### 4.1 数据模型实现

```javascript
// 游戏房间模型
{
  roomId: "R123456789",           // 房间唯一标识
  name: "阿瓦隆-欢乐局",          // 房间名称
  createdAt: ISODate("2023-08-01T10:00:00Z"), // 创建时间
  updatedAt: ISODate("2023-08-01T10:30:00Z"), // 更新时间
  status: "PLAYING",              // 房间状态: WAITING, PLAYING, ENDED
  createdBy: "U987654321",        // 创建者ID
  owner: "U987654321",            // 房主ID
  password: "******",             // 可选房间密码(已加密)
  capacity: 8,                    // 房间容量
  playerCount: 6,                 // 当前玩家数
  players: [                      // 玩家列表
    {
      userId: "U987654321",
      nickname: "玩家A",
      joinedAt: ISODate("2023-08-01T10:00:00Z"),
      status: "ONLINE",           // 玩家状态
      role: "OWNER"               // 房间角色
    },
    // ... 其他玩家
  ],
  gameSettings: {                 // 游戏设置
    roles: ["MERLIN", "ASSASSIN", "LOYAL1", "LOYAL2", "MINION1", "MINION2"],
    options: {
      enableLady: true,
      enablePercival: false
    }
  },
  currentGameId: "G123456789",    // 当前或最近一局游戏ID
  gameHistory: ["G123456780", "G123456781"], // 历史游戏ID列表
  metadata: {                     // 元数据，用于扩展
    visibility: "PUBLIC",
    tags: ["快速局", "新手友好"]
  }
}

// 游戏状态模型
{
  gameId: "G123456789",           // 游戏唯一标识
  roomId: "R123456789",           // 关联的房间ID
  startedAt: ISODate("2023-08-01T10:10:00Z"), // 开始时间
  endedAt: ISODate("2023-08-01T10:40:00Z"),   // 结束时间,进行中为null
  status: "PLAYING",              // 游戏状态: SETUP, PLAYING, ENDED
  version: 42,                    // 状态版本号，用于乐观锁
  currentRound: 3,                // 当前回合
  currentPhase: "TEAM_SELECTION", // 当前阶段
  players: [                      // 玩家详情
    {
      userId: "U987654321",
      nickname: "玩家A",
      role: "MERLIN",             // 角色
      team: "GOOD",               // 阵营
      isLeader: true,             // 是否为当前队长
      status: "ONLINE"            // 在线状态
    },
    // ... 其他玩家
  ],
  rounds: [                       // 回合记录
    {
      roundNumber: 1,
      leader: "U987654321",
      team: ["U987654321", "U987654322"],
      votes: {"U987654321": true, "U987654322": true, /*...*/},
      approved: true,
      missionResult: true,        // 任务成功为true
      ladyOfLake: null
    },
    // ... 其他回合
  ],
  scores: {                       // 得分情况
    good: 1,
    evil: 0
  },
  result: null,                   // 游戏结果,进行中为null
  activeTimers: {                 // 活动计时器
    phase: "TEAM_SELECTION",
    endsAt: ISODate("2023-08-01T10:42:00Z")
  },
  lastEventId: "E987654321",      // 最后一个事件ID，用于同步
  lastUpdateAt: ISODate("2023-08-01T10:35:00Z") // 最后更新时间
}

// 游戏事件模型
{
  eventId: "E987654321",          // 事件唯一ID
  gameId: "G123456789",           // 关联的游戏ID
  timestamp: ISODate("2023-08-01T10:15:30Z"), // 事件发生时间
  sequence: 23,                   // 事件序号，确保顺序
  type: "VOTE_CAST",              // 事件类型
  data: {                         // 事件数据
    playerId: "U987654321",
    vote: true,
    roundNumber: 2
  },
  version: 41,                    // 对应的游戏状态版本
  metadata: {                     // 元数据
    origin: "CLIENT",             // 事件来源
    clientInfo: {
      ip: "192.168.1.100",
      device: "iPhone",
      version: "1.2.3"
    }
  }
}
```

### 4.2 数据存储服务实现

```typescript
// 游戏房间存储服务接口
interface RoomRepository {
  createRoom(roomData: RoomCreateDTO): Promise<Room>;
  findRoomById(roomId: string): Promise<Room | null>;
  findRooms(filter: RoomFilter, options: QueryOptions): Promise<Room[]>;
  updateRoom(
    roomId: string,
    updates: Partial<Room>,
    version?: number
  ): Promise<Room>;
  addPlayerToRoom(roomId: string, player: PlayerDTO): Promise<Room>;
  removePlayerFromRoom(roomId: string, userId: string): Promise<Room>;
  deleteRoom(roomId: string): Promise<boolean>;
  countActiveRooms(filter?: RoomFilter): Promise<number>;
}

// 游戏状态存储服务接口
interface GameStateRepository {
  createGameState(gameData: GameCreateDTO): Promise<GameState>;
  findGameStateById(gameId: string): Promise<GameState | null>;
  findGameStateByRoomId(roomId: string): Promise<GameState | null>;
  updateGameState(
    gameId: string,
    updates: Partial<GameState>,
    version: number
  ): Promise<GameState>;
  saveGameEvent(event: GameEvent): Promise<GameEvent>;
  findGameEvents(
    gameId: string,
    options: EventQueryOptions
  ): Promise<GameEvent[]>;
  createSnapshot(gameId: string): Promise<GameStateSnapshot>;
  findLatestSnapshot(gameId: string): Promise<GameStateSnapshot | null>;
  archiveGame(gameId: string): Promise<ArchivedGame>;
  findArchivedGames(
    filter: ArchiveFilter,
    options: QueryOptions
  ): Promise<ArchivedGame[]>;
}

// MongoDB实现示例 - 游戏房间存储
class MongoRoomRepository implements RoomRepository {
  constructor(private readonly db: Db) {}

  async createRoom(roomData: RoomCreateDTO): Promise<Room> {
    const room = {
      ...roomData,
      roomId: generateUniqueId(),
      createdAt: new Date(),
      updatedAt: new Date(),
      status: "WAITING",
      playerCount: roomData.players?.length || 0,
      gameHistory: [],
    };

    await this.db.collection("rooms").insertOne(room);
    return room;
  }

  async findRoomById(roomId: string): Promise<Room | null> {
    return this.db.collection("rooms").findOne({ roomId });
  }

  async updateRoom(
    roomId: string,
    updates: Partial<Room>,
    version?: number
  ): Promise<Room> {
    const query: any = { roomId };
    if (version !== undefined) {
      // 乐观锁控制
      query.version = version;
    }

    const updateDoc = {
      $set: { ...updates, updatedAt: new Date() },
      $inc: { version: 1 },
    };

    const result = await this.db
      .collection("rooms")
      .findOneAndUpdate(query, updateDoc, { returnDocument: "after" });

    if (!result.value) {
      throw new Error("Room update failed: version conflict or room not found");
    }

    return result.value;
  }

  // ... 其他方法实现
}

// Redis实现示例 - 活跃游戏状态缓存
class RedisGameStateCache {
  constructor(private readonly redis: Redis) {}

  async cacheGameState(
    gameId: string,
    state: GameState,
    ttl: number = 3600
  ): Promise<void> {
    const key = `game:state:${gameId}`;
    await this.redis.setex(key, ttl, JSON.stringify(state));
  }

  async getGameState(gameId: string): Promise<GameState | null> {
    const key = `game:state:${gameId}`;
    const data = await this.redis.get(key);
    if (!data) return null;
    return JSON.parse(data);
  }

  async updateGameStateField(
    gameId: string,
    field: string,
    value: any
  ): Promise<void> {
    const key = `game:state:${gameId}`;
    const state = await this.getGameState(gameId);
    if (!state) throw new Error("Game state not found in cache");

    state[field] = value;
    state.lastUpdateAt = new Date();
    state.version += 1;

    await this.cacheGameState(gameId, state);
  }

  async invalidateGameState(gameId: string): Promise<void> {
    const key = `game:state:${gameId}`;
    await this.redis.del(key);
  }

  // ... 其他方法实现
}
```

### 4.3 游戏数据归档策略

1. **实时游戏完成归档流程**:

   - 游戏结束时触发归档
   - 计算和汇总游戏统计数据
   - 压缩不必要的中间状态
   - 存储到归档集合
   - 更新玩家和房间的历史记录

2. **定期批量归档任务**:

   - 每日低峰期执行
   - 识别长时间未活动的游戏
   - 强制归档长时间未结束的游戏
   - 清理过期的临时数据
   - 生成归档报告

3. **冷热数据分离**:

   - 活跃游戏: Redis + 主数据库
   - 近期游戏(7 天内): 主数据库
   - 历史游戏: 归档数据库
   - 统计数据: 专用分析集合

4. **数据压缩策略**:
   - 事件流压缩算法
   - 仅保留关键状态变更
   - JSON 字段优化
   - 二进制格式存储选项

### 4.4 高效查询设计

1. **索引设计**:

```javascript
// 房间集合索引
db.rooms.createIndex({ status: 1, updatedAt: -1 });
db.rooms.createIndex({ "players.userId": 1 });
db.rooms.createIndex({ createdBy: 1 });
db.rooms.createIndex({ "metadata.visibility": 1, status: 1, updatedAt: -1 });

// 游戏状态集合索引
db.gameStates.createIndex({ roomId: 1 });
db.gameStates.createIndex({ status: 1, startedAt: -1 });
db.gameStates.createIndex({ "players.userId": 1, status: 1 });

// 游戏事件集合索引
db.gameEvents.createIndex({ gameId: 1, sequence: 1 });
db.gameEvents.createIndex({ gameId: 1, timestamp: 1 });
db.gameEvents.createIndex({ gameId: 1, type: 1 });

// 归档游戏集合索引
db.archivedGames.createIndex({ endedAt: -1 });
db.archivedGames.createIndex({ "players.userId": 1, endedAt: -1 });
db.archivedGames.createIndex({ result: 1, endedAt: -1 });
```

2. **分页查询优化**:

```typescript
interface QueryOptions {
  limit: number;
  skip?: number;
  sort?: Record<string, 1 | -1>;
  fields?: string[]; // 投影字段
  useCursor?: boolean; // 是否使用游标
  cursorId?: string; // 游标ID
}

// 房间查询示例
async function findRoomsWithPagination(
  filter: RoomFilter,
  options: QueryOptions
): Promise<PaginatedResult<Room>> {
  const query = buildRoomQuery(filter);

  // 使用计数器缓存优化count查询
  const totalCount = await getCachedCount("rooms", query, 60);

  const cursor = db
    .collection("rooms")
    .find(query)
    .sort(options.sort || { updatedAt: -1 })
    .limit(options.limit);

  if (options.skip) {
    cursor.skip(options.skip);
  }

  if (options.fields && options.fields.length > 0) {
    cursor.project(
      options.fields.reduce((acc, field) => {
        acc[field] = 1;
        return acc;
      }, {})
    );
  }

  const rooms = await cursor.toArray();

  return {
    items: rooms,
    totalCount,
    page: Math.floor((options.skip || 0) / options.limit) + 1,
    pageSize: options.limit,
    hasMore: (options.skip || 0) + rooms.length < totalCount,
  };
}
```

3. **缓存查询策略**:

```typescript
class CachedGameRepository {
  constructor(
    private readonly repository: GameStateRepository,
    private readonly cache: RedisGameStateCache,
    private readonly logger: Logger
  ) {}

  async findGameStateById(gameId: string): Promise<GameState | null> {
    try {
      // 尝试从缓存获取
      const cachedState = await this.cache.getGameState(gameId);
      if (cachedState) {
        return cachedState;
      }

      // 缓存未命中，从数据库加载
      const gameState = await this.repository.findGameStateById(gameId);

      // 如果找到了，放入缓存
      if (gameState) {
        await this.cache.cacheGameState(gameId, gameState);
      }

      return gameState;
    } catch (error) {
      this.logger.error(`Error finding game state: ${error.message}`, {
        gameId,
        error,
      });

      // 降级到直接从数据库读取
      return this.repository.findGameStateById(gameId);
    }
  }

  // ... 其他方法实现，采用类似的缓存策略
}
```

## 5. 测试计划

### 5.1 单元测试

- 数据模型验证测试
- 存储服务接口测试
- 缓存策略测试
- 索引效果测试

### 5.2 集成测试

- 数据创建-读取-更新-删除周期测试
- 并发操作测试
- 事务一致性测试
- 版本冲突解决测试

### 5.3 性能测试

- 单实例读写基准测试
- 高并发读写测试（500 QPS）
- 大数据量查询性能测试
- 缓存效率测试

### 5.4 容错性测试

- 网络故障恢复测试
- 数据库宕机恢复测试
- 数据备份恢复测试
- 边界条件压力测试

## 6. 交付标准

- 所有数据存储服务 API 实现并通过单元测试
- 数据库索引和缓存策略优化完成
- 批量数据归档机制正常工作
- 查询性能满足要求（读<30ms，写<80ms）
- 并发测试证明可支持 500 同时在线玩家
- 完整的 API 文档和示例代码
- 代码符合项目规范并通过代码评审

## 7. 工作量估计

| 子任务                 | 工作量（人天） |
| ---------------------- | -------------- |
| 数据模型和存储接口设计 | 3              |
| MongoDB 存储服务实现   | 5              |
| Redis 缓存层实现       | 3              |
| 游戏事件存储与查询     | 4              |
| 归档系统开发           | 4              |
| 查询优化与测试         | 4              |
| 文档编写和代码整理     | 2              |
| **总计**               | **25**         |
