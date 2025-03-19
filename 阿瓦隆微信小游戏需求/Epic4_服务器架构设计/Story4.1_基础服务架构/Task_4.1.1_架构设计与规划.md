# Task 4.1.1: 架构设计与规划

## 描述

设计并规划阿瓦隆微信小游戏的整体服务器架构，确定技术栈选型，定义系统各个组件之间的关系和通信方式，以及制定基础设施部署方案。

## 验收标准

1. 完成详细的架构设计文档，包括：
   - 系统架构图
   - 组件关系图
   - 数据流图
   - 部署架构图
2. 定义清晰的技术栈选型，并提供选型理由
3. 确定系统的扩展性、可用性和容错性策略
4. 制定服务器资源需求评估和扩展规划
5. 完成基础设施即代码(IaC)配置文件设计

## 详细内容

### 架构设计文档

设计文档应包含以下内容：

1. **整体架构**

   - 分层架构设计
   - 各层级职责划分
   - 模块化设计原则
   - 组件间依赖关系

   #### 详细架构图

   ```plantuml
   @startuml
   !theme plain
   skinparam linetype ortho

   cloud "微信小游戏客户端" as client {
      [UI层] as ClientUI
      [游戏逻辑层] as ClientLogic
      [网络层] as ClientNetwork
   }

   node "阿瓦隆服务器架构" {
      node "负载均衡层" {
         [Nginx反向代理] as Nginx
         [负载均衡器] as LB
         [SSL终结] as SSL
      }

      node "应用服务层" {
         [NestJS应用网关] as Gateway

         node "核心服务" {
            [用户服务] as UserService
            [认证服务] as AuthService
            [房间服务] as RoomService
            [游戏服务] as GameService
         }

         node "支持服务" {
            [配置服务] as ConfigService
            [日志服务] as LoggingService
            [监控服务] as MonitoringService
         }
      }

      node "通信层" {
         [Socket.IO服务器] as SocketServer
         [WebSocket管理器] as WSManager
         [会话管理器] as SessionManager
      }

      node "数据层" {
         database "MongoDB集群" as MongoDB {
            [用户数据] as UserDB
            [游戏数据] as GameDB
            [日志数据] as LogDB
         }

         database "Redis集群" as Redis {
            [会话存储] as SessionCache
            [游戏状态] as GameStateCache
            [实时数据] as RealtimeData
         }
      }

      node "监控与运维" {
         [Prometheus] as Prometheus
         [Grafana] as Grafana
         [Sentry] as Sentry
         [ELK堆栈] as ELK
      }
   }

   client --> Nginx : HTTPS请求
   ClientNetwork --> SocketServer : WebSocket连接

   Nginx --> Gateway : 路由API请求
   Nginx --> SocketServer : 路由WebSocket请求

   Gateway --> UserService
   Gateway --> AuthService
   Gateway --> RoomService
   Gateway --> GameService

   UserService --> MongoDB
   RoomService --> MongoDB
   GameService --> MongoDB
   AuthService --> MongoDB

   UserService --> Redis
   RoomService --> Redis
   GameService --> Redis
   AuthService --> Redis

   SocketServer --> WSManager
   WSManager --> SessionManager
   SessionManager --> Redis

   UserService --> LoggingService
   RoomService --> LoggingService
   GameService --> LoggingService
   AuthService --> LoggingService
   SocketServer --> LoggingService

   LoggingService --> ELK
   MonitoringService --> Prometheus
   Prometheus --> Grafana
   LoggingService --> Sentry

   @enduml
   ```

   #### 系统组件依赖关系

   ```plantuml
   @startuml
   !theme plain

   package "阿瓦隆服务组件依赖图" {
       [用户模块] as UserModule
       [认证模块] as AuthModule
       [房间模块] as RoomModule
       [游戏模块] as GameModule
       [WebSocket模块] as WSModule

       package "共享基础设施" {
           [核心模块] as CoreModule
           [配置模块] as ConfigModule
           [数据库模块] as DatabaseModule
           [缓存模块] as CacheModule
           [日志模块] as LoggingModule
           [事件模块] as EventModule
       }
   }

   AuthModule --> CoreModule
   AuthModule --> ConfigModule
   AuthModule --> DatabaseModule
   AuthModule --> CacheModule
   AuthModule --> LoggingModule

   UserModule --> CoreModule
   UserModule --> ConfigModule
   UserModule --> DatabaseModule
   UserModule --> CacheModule
   UserModule --> LoggingModule
   UserModule --> AuthModule

   RoomModule --> CoreModule
   RoomModule --> ConfigModule
   RoomModule --> DatabaseModule
   RoomModule --> CacheModule
   RoomModule --> LoggingModule
   RoomModule --> UserModule
   RoomModule --> EventModule

   GameModule --> CoreModule
   GameModule --> ConfigModule
   GameModule --> DatabaseModule
   GameModule --> CacheModule
   GameModule --> LoggingModule
   GameModule --> RoomModule
   GameModule --> EventModule

   WSModule --> CoreModule
   WSModule --> ConfigModule
   WSModule --> CacheModule
   WSModule --> LoggingModule
   WSModule --> EventModule
   WSModule --> AuthModule
   WSModule --> GameModule

   @enduml
   ```

2. **技术选型**

   | 组件     | 技术选择                | 选型理由                                                                        |
   | -------- | ----------------------- | ------------------------------------------------------------------------------- |
   | 后端框架 | NestJS                  | 提供完整的企业级框架结构，支持 TypeScript，模块化设计，依赖注入，易于维护和扩展 |
   | 数据库   | MongoDB                 | 文档型数据库适合游戏数据的半结构化存储，支持高并发读写，便于水平扩展            |
   | 缓存     | Redis                   | 高性能内存数据库，支持发布/订阅，适合存储游戏状态和会话数据                     |
   | 实时通信 | Socket.IO               | 提供可靠的 WebSocket 抽象，自动降级，支持房间和命名空间概念                     |
   | API 文档 | Swagger                 | 自动生成 API 文档，支持交互式测试                                               |
   | 日志管理 | Winston + ELK           | Winston 提供灵活的日志记录，ELK 提供集中式日志管理和分析                        |
   | 监控     | Prometheus + Grafana    | 时序数据库适合收集性能指标，Grafana 提供可视化面板                              |
   | 容器化   | Docker + Docker Compose | 标准化部署环境，简化开发和生产环境一致性                                        |
   | CI/CD    | GitHub Actions          | 与代码仓库紧密集成，自动化测试、构建和部署流程                                  |
   | 负载均衡 | Nginx                   | 高性能 Web 服务器，支持 WebSocket 代理和 SSL 终结                               |

   #### 实时通信技术选型分析

   ```plantuml
   @startuml
   !theme plain

   card Socket.IO [
     <b>Socket.IO</b>
     ----
     优点:
     - 自动降级支持
     - 断线重连机制
     - 房间和命名空间
     - 二进制数据支持
     - 跨平台兼容性

     缺点:
     - 消息大小略大于原生WS
     - 额外协议开销
   ]

   card "原生WebSocket" [
     <b>原生WebSocket</b>
     ----
     优点:
     - 更低的协议开销
     - 更小的消息体积
     - 无额外依赖

     缺点:
     - 无自动重连
     - 无降级支持
     - 需要自行实现房间
     - 兼容性问题
   ]

   card "SignalR" [
     <b>SignalR</b>
     ----
     优点:
     - 强类型支持
     - .NET集成良好
     - 自动降级支持

     缺点:
     - Node.js支持较弱
     - 客户端库较重
     - 学习曲线陡
   ]

   @enduml
   ```

3. **API 设计**

   - RESTful API 设计原则
   - WebSocket 事件设计
   - 数据交换格式定义

   #### API 接口结构

   ```
   /api
   ├── /auth
   │   ├── POST /login          # 使用微信code登录
   │   ├── POST /refresh        # 刷新访问令牌
   │   └── POST /logout         # 登出用户
   ├── /users
   │   ├── GET /me              # 获取当前用户信息
   │   ├── PUT /me              # 更新用户信息
   │   └── GET /:id/statistics  # 获取用户游戏统计
   ├── /rooms
   │   ├── POST /               # 创建游戏房间
   │   ├── GET /                # 获取房间列表
   │   ├── GET /:id             # 获取房间详情
   │   ├── POST /:code/join     # 通过房间码加入
   │   ├── POST /:id/leave      # 离开房间
   │   ├── PUT /:id/settings    # 更新房间设置
   │   └── DELETE /:id          # 关闭房间
   ├── /games
   │   ├── POST /rooms/:id/start # 开始游戏
   │   ├── GET /:id              # 获取游戏状态
   │   ├── GET /:id/history      # 获取游戏历史
   │   └── POST /:id/actions     # 执行游戏动作
   └── /system
       ├── GET /health           # 健康检查
       └── GET /version          # API版本信息
   ```

   #### WebSocket 事件设计

   ```javascript
   // 客户端事件 (客户端触发)
   socket.emit('auth', { token: 'jwt_token' });              // 认证
   socket.emit('room:join', { roomId: 'room_id' });          // 加入房间
   socket.emit('room:leave');                                // 离开房间
   socket.emit('room:ready', { ready: true });               // 准备状态
   socket.emit('game:action', { type: 'vote', data: {...} }); // 游戏动作

   // 服务器事件 (服务器触发)
   socket.on('auth:success', (data) => { ... });            // 认证成功
   socket.on('auth:error', (error) => { ... });             // 认证失败
   socket.on('room:joined', (roomData) => { ... });         // 成功加入房间
   socket.on('room:updated', (roomData) => { ... });        // 房间信息更新
   socket.on('room:playerJoined', (playerData) => { ... }); // 新玩家加入
   socket.on('room:playerLeft', (playerData) => { ... });   // 玩家离开
   socket.on('game:started', (gameData) => { ... });        // 游戏开始
   socket.on('game:updated', (gameState) => { ... });       // 游戏状态更新
   socket.on('game:ended', (result) => { ... });            // 游戏结束
   socket.on('error', (error) => { ... });                  // 一般错误
   ```

4. **数据库设计**

   - 主要实体关系设计
   - MongoDB 集合设计
   - Redis 数据结构设计
   - 数据访问层策略

   #### 实体关系图

   ```plantuml
   @startuml
   !theme plain

   entity "User" as user {
     * _id : ObjectId
     --
     * openId : String
     * nickname : String
     * avatarUrl : String
     * gender : Number
     * createdAt : Date
     * lastLogin : Date
     * gameStats : Object
   }

   entity "Room" as room {
     * _id : ObjectId
     --
     * roomCode : String
     * creatorId : ObjectId
     * status : String
     * createTime : Date
     * players : Array<ObjectId>
     * settings : Object
     * currentGame : ObjectId
   }

   entity "Game" as game {
     * _id : ObjectId
     --
     * roomId : ObjectId
     * players : Array<Object>
     * roles : Object
     * missions : Array<Object>
     * votes : Array<Object>
     * result : Object
     * startTime : Date
     * endTime : Date
     * logs : Array<Object>
   }

   entity "GameHistory" as history {
     * _id : ObjectId
     --
     * gameId : ObjectId
     * roomId : ObjectId
     * summary : Object
     * createdAt : Date
   }

   user }|--o{ room : creates/joins
   room ||--o{ game : hosts
   game ||--|| history : archived as
   user }|--o{ game : participates in

   @enduml
   ```

   #### Redis 数据结构设计

   ```
   # 用户会话
   session:{sessionId} -> hash {
     userId: "user_id",
     openId: "wx_open_id",
     expires: timestamp,
     permissions: ["user", "admin"]
   }

   # 用户在线状态
   user:online:{userId} -> hash {
     connectionId: "socket_id",
     roomId: "room_id",
     lastActive: timestamp,
     clientInfo: { device: "iOS", version: "1.0.1" }
   }

   # 房间缓存
   room:{roomId} -> hash {
     roomCode: "123456",
     status: "waiting",
     playerCount: 5,
     settings: { encoded_json }
   }

   # 活跃房间索引
   rooms:active -> sorted set [
     { score: timestamp, value: "room_id_1" },
     { score: timestamp, value: "room_id_2" }
   ]

   # 游戏状态缓存
   game:state:{gameId} -> hash {
     phase: "team_building",
     round: 2,
     leader: "user_id",
     selectedPlayers: ["user_id_1", "user_id_2"],
     stateVersion: 35
   }

   # 游戏事件队列
   queue:game:events -> list [
     { type: "player_vote", data: { ... }, timestamp },
     { type: "mission_result", data: { ... }, timestamp }
   ]
   ```

5. **基础设施配置**

   - 服务器规格和配置
   - 网络架构设计
   - 安全策略和访问控制
   - 容器编排策略

   #### 容器编排配置示例

   ```yaml
   # docker-compose.yml 示例
   version: "3.8"

   services:
     nginx:
       image: nginx:alpine
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - ./nginx/conf:/etc/nginx/conf.d
         - ./nginx/ssl:/etc/nginx/ssl
         - ./nginx/logs:/var/log/nginx
       depends_on:
         - api
         - socket
       networks:
         - frontend

     api:
       build:
         context: ./
         dockerfile: Dockerfile.api
       environment:
         - NODE_ENV=production
         - MONGODB_URI=mongodb://mongodb:27017/avalon
         - REDIS_HOST=redis
         - REDIS_PORT=6379
       volumes:
         - ./logs:/app/logs
       depends_on:
         - mongodb
         - redis
       deploy:
         replicas: 2
         update_config:
           parallelism: 1
           delay: 10s
         restart_policy:
           condition: on-failure
       networks:
         - frontend
         - backend

     socket:
       build:
         context: ./
         dockerfile: Dockerfile.socket
       environment:
         - NODE_ENV=production
         - MONGODB_URI=mongodb://mongodb:27017/avalon
         - REDIS_HOST=redis
         - REDIS_PORT=6379
       volumes:
         - ./logs:/app/logs
       depends_on:
         - mongodb
         - redis
       deploy:
         replicas: 2
         update_config:
           parallelism: 1
           delay: 10s
         restart_policy:
           condition: on-failure
       networks:
         - frontend
         - backend

     mongodb:
       image: mongo:5
       volumes:
         - mongodb_data:/data/db
       environment:
         - MONGO_INITDB_ROOT_USERNAME=admin
         - MONGO_INITDB_ROOT_PASSWORD=password
       ports:
         - "27017:27017"
       networks:
         - backend

     redis:
       image: redis:6-alpine
       command: redis-server --requirepass password
       volumes:
         - redis_data:/data
       ports:
         - "6379:6379"
       networks:
         - backend

     prometheus:
       image: prom/prometheus
       volumes:
         - ./prometheus:/etc/prometheus
         - prometheus_data:/prometheus
       ports:
         - "9090:9090"
       networks:
         - monitoring

     grafana:
       image: grafana/grafana
       volumes:
         - grafana_data:/var/lib/grafana
       ports:
         - "3000:3000"
       depends_on:
         - prometheus
       networks:
         - monitoring

   networks:
     frontend:
     backend:
     monitoring:

   volumes:
     mongodb_data:
     redis_data:
     prometheus_data:
     grafana_data:
   ```

### 资源需求评估

1. **计算资源**

   | 服务           | CPU    | 内存 | 实例数量（初始） | 自动扩展策略                            |
   | -------------- | ------ | ---- | ---------------- | --------------------------------------- |
   | API 服务       | 2 vCPU | 2 GB | 2                | CPU > 70% 时增加实例，最大 5 个         |
   | WebSocket 服务 | 2 vCPU | 4 GB | 2                | 每 500 并发连接增加 1 个实例，最大 8 个 |
   | MongoDB        | 4 vCPU | 8 GB | 1 主 + 2 副本    | 存储利用率 > 70% 时扩容                 |
   | Redis          | 2 vCPU | 4 GB | 1 主 + 2 副本    | 内存利用率 > 70% 时扩容                 |
   | Nginx          | 2 vCPU | 2 GB | 2                | 请求数 > 1000/s 时增加实例              |

2. **存储资源**

   | 数据类型   | 初始容量 | 增长预测             | 备份策略              |
   | ---------- | -------- | -------------------- | --------------------- |
   | 用户数据   | 5 GB     | 每月 0.5 GB          | 每日增量，每周完整    |
   | 游戏数据   | 10 GB    | 每月 2 GB            | 每日增量，每周完整    |
   | 日志数据   | 20 GB    | 每月 5 GB            | 每周完整，保留 3 个月 |
   | Redis 缓存 | 2 GB     | 根据并发用户动态调整 | RDB 持久化 + AOF 日志 |

3. **网络资源**

   | 场景     | 估计带宽 | 连接数上限      |
   | -------- | -------- | --------------- |
   | 普通时段 | 10 Mbps  | 1,000 并发连接  |
   | 峰值时段 | 50 Mbps  | 5,000 并发连接  |
   | 活动期间 | 100 Mbps | 10,000 并发连接 |

### 扩展性和高可用性设计

1. **水平扩展策略**

   - **API 服务**：

     - 无状态设计，可直接增加实例
     - 使用 Nginx 负载均衡，支持轮询和 IP 哈希策略
     - 基于 CPU 和内存使用率的自动扩缩容

   - **WebSocket 服务**：

     - 使用 Redis 适配器实现集群支持
     - 基于连接数和消息吞吐量的自动扩缩容
     - 会话亲和性保证用户连接到同一服务器

   - **数据库扩展**：
     - MongoDB 复制集提高读取性能和可用性
     - 数据分片支持（按用户 ID 或游戏 ID）
     - Redis 主从复制和哨兵模式

   #### 扩展架构图

   ```plantuml
   @startuml
   !theme plain
   skinparam linetype ortho

   cloud "用户" as Users

   node "负载均衡" as LB {
     [Nginx 1] as N1
     [Nginx 2] as N2
   }

   node "API服务集群" as API {
     [API实例 1] as API1
     [API实例 2] as API2
     [API实例 3] as API3
     [API实例 ...] as APIn
   }

   node "WebSocket服务集群" as WS {
     [WS实例 1] as WS1
     [WS实例 2] as WS2
     [WS实例 3] as WS3
     [WS实例 ...] as WSn
   }

   node "Redis集群" as Redis {
     [Redis主节点] as RM
     [Redis从节点1] as RS1
     [Redis从节点2] as RS2
     [Redis哨兵1] as RSen1
     [Redis哨兵2] as RSen2
     [Redis哨兵3] as RSen3
   }

   node "MongoDB集群" as Mongo {
     [MongoDB主节点] as MM
     [MongoDB副本1] as MS1
     [MongoDB副本2] as MS2
     [MongoDB配置服务器] as MConfig
   }

   Users --> LB

   LB --> API
   LB --> WS

   API --> Redis
   API --> Mongo

   WS --> Redis
   WS --> Mongo

   RM --> RS1
   RM --> RS2

   RSen1 --> RM
   RSen1 --> RS1
   RSen1 --> RS2

   RSen2 --> RM
   RSen2 --> RS1
   RSen2 --> RS2

   RSen3 --> RM
   RSen3 --> RS1
   RSen3 --> RS2

   MM --> MS1
   MM --> MS2

   @enduml
   ```

2. **高可用性措施**

   - **多区域部署**：

     - 主要区域：腾讯云华东区域
     - 备份区域：腾讯云华北区域
     - 使用云负载均衡实现区域间流量分配

   - **故障转移机制**：

     - MongoDB 自动故障转移（副本集选举）
     - Redis 哨兵模式实现主从自动切换
     - 后端服务健康检查和自动重启

   - **灾难恢复计划**：
     - 数据库定时快照和备份
     - 跨区域数据同步
     - 恢复时间目标（RTO）：30 分钟
     - 恢复点目标（RPO）：5 分钟

   #### 故障恢复流程图

   ```plantuml
   @startuml
   !theme plain

   start

   :监控系统检测到服务异常;

   if (是单个实例故障?) then (是)
     :标记实例为不健康;
     :从负载均衡中移除实例;
     :尝试重启实例;

     if (重启成功?) then (是)
       :恢复健康检查;
       :重新加入负载均衡;
     else (否)
       :启动新实例替代故障实例;
     endif

   else (否)
     if (是区域级故障?) then (是)
       :激活跨区域故障转移;
       :将流量切换到备用区域;
       :启动备用区域的全部服务;
     else (否)
       :执行服务组件级恢复;

       fork
         :数据库故障恢复流程;
         if (主节点故障?) then (是)
           :触发副本集选举;
           :提升新的主节点;
         else (否)
           :重启故障的副本节点;
         endif
       fork again
         :缓存故障恢复流程;
         if (Redis主节点故障?) then (是)
           :哨兵触发主从切换;
           :更新客户端连接配置;
         else (否)
           :重启故障的从节点;
         endif
       fork again
         :应用服务故障恢复;
         :启动足够数量的新实例;
         :更新服务发现注册表;
       end fork
     endif
   endif

   :验证系统功能恢复;
   :通知运维团队;
   :记录事故报告;

   stop

   @enduml
   ```

3. **性能优化策略**

   - **缓存策略**：

     - 热点数据多级缓存
     - 房间和游戏状态缓存在 Redis
     - 用户会话和权限缓存

   - **数据索引优化**：

     - MongoDB 复合索引设计
     - 常用查询条件索引覆盖
     - 定期索引使用分析和优化

   - **连接池配置**：
     - MongoDB 连接池：初始 10，最大 50
     - Redis 连接池：初始 5，最大 30
     - HTTP 连接池：初始 20，最大 100

## 工作量估计

3 人天

## 技术关键点

1. 确保架构设计满足实时游戏的低延迟需求
2. 设计弹性架构以应对用户量波动
3. 保证数据一致性的同时最大化系统可用性
4. 考虑微信平台特有的技术约束和要求

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)

## 依赖关系

- 无上游依赖
- 下游依赖:
  - Task 4.1.2: 核心服务框架搭建
  - Task 4.1.3: 数据库与缓存配置
  - Task 4.1.4: 认证与授权服务
  - Task 4.1.5: 日志与监控系统
  - Task 4.1.6: 服务部署与配置
