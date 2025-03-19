# Task 6.2.3: 游戏状态同步机制

## 1. 任务描述

本任务旨在设计和实现阿瓦隆游戏的实时状态同步机制，确保游戏进程中的所有玩家能够实时获取游戏状态变更，保持状态一致性，并支持断线重连时的状态恢复。该机制是游戏多人协作体验的核心支撑，将直接影响游戏的流畅度和用户体验。

游戏状态同步机制需要解决以下关键问题：

- 在多玩家环境下如何保持游戏状态的实时同步
- 如何处理网络延迟和不稳定连接导致的状态不一致
- 玩家断线重连后如何快速恢复游戏状态
- 多个客户端同时操作时如何避免冲突和保持数据一致性
- 如何在保证游戏公平性的前提下优化同步性能

通过实现高效、可靠的状态同步机制，使玩家能够沉浸在流畅的游戏体验中，即使在网络环境不佳的情况下也能保持游戏的连贯性。

## 2. 功能需求

### 2.1 实时游戏状态同步协议

实时游戏状态同步协议是游戏客户端与服务器之间交换游戏状态信息的标准化通信规则，需要满足以下功能需求：

- **协议基础要求**：

  - 定义基于 WebSocket 的二进制或 JSON 消息格式
  - 支持全量状态同步和增量状态更新
  - 包含消息序列号和时间戳，确保顺序性
  - 支持消息确认机制，实现可靠传输
  - 内置心跳机制，检测连接健康状态

- **状态消息类型**：

  - 游戏初始状态（全量）：包含完整游戏配置和初始状态
  - 状态变更消息（增量）：仅包含变更的状态字段
  - 玩家操作消息：客户端发送的游戏操作
  - 系统事件消息：服务端产生的游戏事件
  - 同步控制消息：用于协调同步过程的控制信息

- **状态粒度控制**：

  - 按角色权限过滤状态信息（如：邪恶方玩家不应看到梅林身份）
  - 支持细粒度的状态访问控制
  - 区分公开状态和私有状态
  - 提供状态变更范围控制，减少网络传输量

- **优先级与延迟处理**：

  - 为不同类型的状态更新定义优先级
  - 关键操作（如投票）具有高优先级
  - 支持延迟补偿策略
  - 实现平滑状态过渡机制

- **版本与兼容性**：
  - 支持协议版本号，确保向后兼容
  - 客户端与服务端版本协商机制
  - 支持协议升级过程中的平滑过渡
  - 兼容不同设备和网络条件的差异化处理

### 2.2 游戏状态变更推送机制

游戏状态变更推送机制负责将游戏状态的变化实时传递给所有相关玩家，确保游戏体验的连贯性和一致性，需要满足以下功能需求：

- **推送触发机制**：

  - 玩家操作引起的状态变更自动触发推送
  - 系统事件（如计时器到期）触发的状态变更推送
  - 管理员干预操作导致的状态变更推送
  - 定期状态同步检查触发的推送

- **推送目标选择**：

  - 基于玩家角色的定向推送
  - 全房间广播机制
  - 按状态变更相关性进行推送过滤
  - 支持观战者模式的特殊状态推送

- **推送策略优化**：

  - 状态变更批量处理和合并推送
  - 基于网络质量的自适应推送频率
  - 低优先级状态更新的延迟发送
  - 关键状态变更的立即推送

- **推送质量保障**：

  - 关键状态变更的确认回执机制
  - 推送消息的超时重试策略
  - 消息丢失检测与恢复机制
  - 推送执行状态监控和日志记录

- **推送流量控制**：
  - 单个客户端的推送速率限制
  - 推送队列优先级管理
  - 网络拥塞时的降级推送策略
  - 大型状态变更的分片推送

### 2.3 断线重连状态恢复功能

断线重连状态恢复功能确保玩家因网络问题临时断开连接后，能够快速、无缝地重新加入游戏并恢复当前游戏状态，需要满足以下功能需求：

- **会话保持机制**：

  - 客户端断开连接后的会话保持时间（至少 5 分钟）
  - 会话标识符生成与验证
  - 长时间断线的会话过期策略
  - 角色临时 AI 托管选项

- **重连身份验证**：

  - 快速重连认证流程
  - 断线前状态校验机制
  - 防止非授权玩家接管
  - 重连尝试次数限制与冷却时间

- **状态恢复策略**：

  - 根据断线时长选择恢复策略（增量/全量）
  - 状态快照与事件日志结合的恢复机制
  - 优先恢复关键游戏状态
  - 分批次有序恢复完整状态

- **游戏连续性保障**：

  - 重连过程中游戏状态的临时冻结
  - 其他玩家的重连状态通知
  - 重连后的游戏进度对齐
  - 玩家重连后的操作时间补偿

- **多设备重连处理**：

  - 同一账号多设备重连的冲突解决
  - 设备切换时的状态迁移
  - 不同设备类型的状态适配
  - 设备能力差异下的降级恢复策略

- **重连失败处理**：
  - 多次重连失败后的备选方案
  - 长时间无法重连的游戏影响最小化
  - 异常情况下的手动恢复机制
  - 重连过程的完整日志记录

### 2.4 多端状态一致性保障机制

多端状态一致性保障机制确保在不同终端设备、不同网络环境下的玩家能够享有一致的游戏状态和体验，防止状态偏差和不同步问题，需要满足以下功能需求：

- **状态权威认定**：

  - 服务器作为唯一状态权威源
  - 客户端状态预测与服务器状态校正机制
  - 状态变更的确认与验证流程
  - 异常状态的检测与纠正机制

- **状态同步检查点**：

  - 定期全量状态校验点设置
  - 关键游戏阶段的强制同步
  - 状态哈希值快速比对机制
  - 差异检测时的自动恢复流程

- **时间同步机制**：

  - 服务器与客户端的时钟同步
  - 网络延迟的测量与补偿
  - 操作时序的统一处理
  - 时间敏感操作的特殊处理策略

- **多端适配策略**：

  - 不同终端设备的状态表示一致性
  - UI 呈现差异下的状态映射规则
  - 设备性能差异的兼容处理
  - 新旧客户端版本的状态兼容性保障

- **状态同步监控**：

  - 实时监测客户端状态一致性
  - 状态偏差程度的量化评估
  - 异常客户端的标记与处理
  - 系统级别的同步健康度报告

- **降级保障策略**：
  - 严重网络波动时的最小状态同步保障
  - 部分状态不一致时的游戏体验保障
  - 非关键状态的延迟同步处理
  - 极端条件下的服务降级方案

### 2.5 状态冲突解决策略

状态冲突解决策略处理多个客户端同时操作或网络延迟导致的状态冲突问题，确保游戏状态的一致性和公平性，需要满足以下功能需求：

- **冲突检测机制**：

  - 基于版本号的冲突检测
  - 时间戳与序列号结合的判定策略
  - 并发操作的识别与标记
  - 状态偏差程度的评估方法

- **冲突类型分类**：

  - 同一资源的并发修改冲突
  - 操作顺序依赖的时序冲突
  - 客户端状态预测与服务器状态不一致
  - 玩家意图与游戏规则冲突

- **冲突解决策略**：

  - 基于时间优先的解决策略（先到先得）
  - 基于角色权重的优先级策略
  - 游戏规则约束下的有效性判定
  - 服务器最终裁决机制

- **状态回滚与重播**：

  - 冲突检测后的状态回滚机制
  - 基于事件日志的状态重播
  - 最小影响范围的局部回滚
  - 回滚操作的平滑过渡处理

- **用户体验保障**：

  - 冲突解决过程对用户的透明度控制
  - 冲突发生时的用户提示机制
  - 降低冲突感知对游戏体验的影响
  - 特殊冲突情况下的补偿措施

- **异常处理机制**：
  - 无法自动解决的冲突上报机制
  - 严重冲突下的游戏状态保护
  - 长时间冲突的超时处理
  - 冲突解决失败的备选方案

## 3. 技术规格

### 3.1 WebSocket 通信架构

阿瓦隆游戏的实时状态同步将基于 WebSocket 技术实现高效、稳定的双向通信，通信架构技术规格如下：

#### 3.1.1 整体架构

- **服务端架构**：

  - 采用 Node.js + Socket.IO 或 ws 库实现 WebSocket 服务
  - 支持水平扩展的集群模式部署
  - 每个游戏房间独立的通信通道
  - 使用 Redis Pub/Sub 实现跨节点消息同步

- **通信层次结构**：

  ```
  ┌─────────────────────────────────┐
  │           客户端应用层           │
  │  (游戏状态管理、UI渲染、用户交互)  │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │          客户端通信层            │
  │ (WebSocket客户端、消息序列化/反序列化) │
  └───────────────┬─────────────────┘
                  │
                  ▼
          ┌───────────────┐
          │   微信小游戏API  │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │   互联网/NAT    │
          └───────┬───────┘
                  │
                  ▼
  ┌───────────────────────────────────┐
  │          负载均衡层              │
  │      (Nginx/HAProxy)            │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │         WebSocket服务器集群      │
  │ (连接管理、消息路由、会话维护)      │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │           游戏逻辑层             │
  │ (状态处理、事件处理、业务逻辑)      │
  └───────────────┬─────────────────┘
                  │
  ┌───────────────▼─────────────────┐
  │           数据持久层             │
  │ (MongoDB存储、Redis缓存)         │
  └─────────────────────────────────┘
  ```

- **连接管理**：
  - 最大并发连接数：每服务节点支持 10,000+连接
  - 单房间最大连接数：标准 8 人+观战者（最多 20 人）
  - 连接状态监控：活跃/闲置/断开/重连中
  - 房间连接分片：相同房间的连接尽量路由到同一节点

#### 3.1.2 连接规格

- **WebSocket 配置**：

  - 协议：标准 WebSocket 协议（RFC 6455）
  - 子协议：'avalon-game-v1'
  - 压缩：支持 Per-message Deflate 扩展
  - 最大消息大小：256KB
  - 心跳间隔：20 秒

- **连接安全性**：

  - 传输加密：TLS 1.3
  - 连接认证：JWT 令牌验证
  - 消息完整性：包含校验和
  - 防重放攻击：包含随机 nonce 值

- **连接可靠性**：
  - 自动重连策略：指数退避算法（初始 100ms，最大 10s）
  - 重连最大尝试次数：10 次
  - 连接保活机制：双向心跳包
  - 异常检测：30 秒无响应判定为异常

#### 3.1.3 性能指标

- **延迟要求**：

  - 消息传输平均延迟：< 100ms
  - 消息处理时间：< 50ms
  - 状态同步延迟：< 200ms
  - 重连恢复时间：< 2 秒

- **吞吐量指标**：

  - 单房间消息吞吐量：50 条/秒
  - 单服务器节点消息处理能力：10,000 条/秒
  - 单客户端发送限制：10 条/秒
  - 单客户端接收能力：20 条/秒

- **资源占用**：
  - 客户端内存占用：< 10MB
  - 客户端 CPU 占用：< 5%
  - 单连接服务端内存：约 50KB
  - 网络带宽消耗：平均 100KB/min/用户

#### 3.1.4 故障恢复机制

- **服务端故障恢复**：

  - 节点故障自动转移：< 10 秒
  - 会话状态持久化：Redis
  - 集群健康检查频率：5 秒
  - 自动扩缩容触发条件：CPU > 70% 或 内存 > 80%

- **网络故障处理**：
  - 网络抖动容忍度：500ms 内的抖动不触发重连
  - 短暂断线（<30 秒）：自动重连并恢复
  - 长时间断线：会话保持并等待重连
  - 极端网络条件：降级同步策略启动

### 3.2 状态同步协议设计

阿瓦隆游戏的状态同步协议定义了游戏状态数据的交换格式、同步策略和消息处理机制，以确保所有客户端能够保持一致的游戏状态，技术规格如下：

#### 3.2.1 消息格式规范

- **基本消息结构**：

  ```json
  {
    "header": {
      "msgId": "m123456789", // 消息唯一ID
      "msgType": "STATE_UPDATE", // 消息类型
      "gameId": "G987654321", // 游戏ID
      "roomId": "R123456789", // 房间ID
      "senderId": "U123456789", // 发送者ID
      "timestamp": 1623456789123, // 发送时间戳
      "sequence": 42, // 消息序列号
      "version": 2 // 协议版本
    },
    "payload": {
      // 消息主体内容
      // 根据msgType不同，结构不同
    },
    "meta": {
      // 元数据
      "compression": "none", // 压缩方式
      "encryption": "none", // 加密方式
      "priority": 1, // 优先级(0-9)
      "ttl": 30, // 生存时间(秒)
      "nonce": "abc123def456" // 防重放随机值
    }
  }
  ```

- **消息类型定义**：

  | 消息类型      | 描述         | 方向            | 优先级 |
  | ------------- | ------------ | --------------- | ------ |
  | HANDSHAKE     | 连接握手消息 | 双向            | 9      |
  | ACK           | 消息确认回执 | 双向            | 9      |
  | HEARTBEAT     | 心跳包       | 双向            | 8      |
  | FULL_STATE    | 全量游戏状态 | 服务器 → 客户端 | 7      |
  | STATE_UPDATE  | 状态增量更新 | 服务器 → 客户端 | 6      |
  | GAME_EVENT    | 游戏事件通知 | 服务器 → 客户端 | 5      |
  | PLAYER_ACTION | 玩家操作     | 客户端 → 服务器 | 6      |
  | STATE_REQUEST | 状态请求     | 客户端 → 服务器 | 4      |
  | SYNC_CHECK    | 同步状态检查 | 双向            | 3      |
  | ERROR         | 错误通知     | 双向            | 7      |
  | SYSTEM_NOTICE | 系统通知     | 服务器 → 客户端 | 2      |

- **状态更新格式**：
  ```json
  // STATE_UPDATE消息payload示例
  {
    "stateVersion": 103, // 状态版本号
    "baseVersion": 100, // 基准版本号
    "updateType": "incremental", // 更新类型
    "changes": [
      // 变更集合
      {
        "path": "currentRound", // 变更路径
        "op": "replace", // 操作类型
        "value": 2 // 新值
      },
      {
        "path": "players.1.status", // 嵌套路径
        "op": "replace",
        "value": "READY"
      },
      {
        "path": "rounds[1]", // 数组操作
        "op": "add",
        "value": {
          /* round object */
        }
      }
    ],
    "events": [
      // 关联事件
      {
        "eventId": "E123456789",
        "type": "ROUND_STARTED",
        "timestamp": 1623456789123,
        "data": {
          /* event data */
        }
      }
    ]
  }
  ```

#### 3.2.2 同步策略

- **全量同步策略**：

  - 触发条件：
    - 玩家初次加入游戏
    - 玩家断线重连超过阈值时间
    - 状态不一致检测到严重偏差
    - 关键游戏阶段转换点
  - 实现方式：
    - 一次性传输完整游戏状态
    - 可选分块传输（大状态）
    - 包含状态版本号和时间戳
    - 替换客户端整个状态

- **增量同步策略**：

  - 触发条件：
    - 游戏状态发生变化
    - 客户端请求特定状态更新
    - 定期增量同步检查点
  - 实现方式：
    - 仅传输变更部分
    - 使用 JSON Patch 格式描述变更
    - 基于版本号的增量计算
    - 变更路径支持通配符

- **定向同步策略**：

  - 根据玩家角色和权限筛选状态
  - 私有信息仅发送给授权玩家
  - 观战者接收特殊处理后的状态
  - 支持状态字段级别的可见性控制

- **自适应同步策略**：
  - 根据网络条件调整同步频率
  - 高延迟环境下合并非关键更新
  - 弱网环境启用状态压缩
  - 网络质量分层推送策略

#### 3.2.3 序列化与编码

- **序列化选项**：

  - 主要格式：JSON（兼容性好，调试方便）
  - 可选格式：MessagePack（体积小，处理快）
  - 大型二进制数据单独传输
  - 内嵌类型信息确保正确解析

- **消息编码优化**：

  - 字段名压缩：常用字段使用短名称
  - 枚举值数字化：状态常量使用数字代码
  - 数值压缩：使用合适的数据类型
  - 增量编码：仅包含变更部分

- **数据压缩**：
  - 大于 1KB 的消息自动启用压缩
  - 支持 gzip 和 deflate 压缩算法
  - 可配置的压缩级别（速度 vs 压缩比）
  - 客户端声明支持的压缩能力

#### 3.2.4 消息处理流程

- **发送流程**：

  1. 状态变更触发发送请求
  2. 根据接收者过滤状态内容
  3. 增量计算需要发送的变更
  4. 序列化并可选压缩消息
  5. 添加消息头和元数据
  6. 通过 WebSocket 发送消息

- **接收流程**：

  1. 接收 WebSocket 消息
  2. 验证消息完整性和来源
  3. 解压缩并反序列化消息
  4. 检查消息序列号确保顺序
  5. 应用状态变更或处理事件
  6. 发送确认回执（如需要）

- **确认机制**：
  - 关键消息需要接收方确认
  - 确认超时后自动重发策略
  - 批量确认减少消息数量
  - 支持显式和隐式确认方式

### 3.3 状态恢复机制

阿瓦隆游戏的状态恢复机制负责确保玩家断线后能快速恢复游戏状态，提供无缝的游戏体验，技术规格如下：

#### 3.3.1 会话管理与保持

- **会话标识与存储**：

  - 会话标识符格式：`sessionId = uuid-v4`
  - 会话数据存储：Redis 中使用哈希结构
  - 键名格式：`avalon:session:{sessionId}`
  - 过期时间：默认 5 分钟，可配置

- **会话数据内容**：

  ```json
  {
    "sessionId": "1a2b3c4d-5e6f-7g8h-9i0j",
    "userId": "U123456789",
    "gameId": "G987654321",
    "roomId": "R123456789",
    "deviceId": "D123456789",
    "connectionInfo": {
      "ip": "192.168.1.1",
      "userAgent": "...",
      "lastActiveTime": 1623456789123,
      "connectionStatus": "DISCONNECTED"
    },
    "gameState": {
      "lastStateVersion": 103,
      "lastSequenceReceived": 42,
      "lastSequenceSent": 65,
      "playerRole": "MERLIN",
      "inGameStatus": "ACTIVE",
      "disconnectTime": 1623456789123
    }
  }
  ```

- **会话刷新策略**：

  - 活跃会话自动续期（heartbeat 触发）
  - 重要游戏阶段强制会话刷新
  - 会话合并策略（多设备登录）
  - 会话清理时机：游戏结束或过期

- **AI 托管转换**：
  - 会话无响应超过 2 分钟启动 AI 托管
  - AI 托管角色的行为策略定义
  - 托管状态显示与通知机制
  - 玩家重连后的托管解除流程

#### 3.3.2 状态存储与快照

- **状态快照策略**：

  - 生成粒度：
    - 关键游戏节点强制快照
    - 时间间隔自动快照（30 秒）
    - 状态变更累积阈值快照（>10 次变更）
  - 存储结构：
    ```json
    {
      "snapshotId": "S123456789",
      "gameId": "G987654321",
      "roomId": "R123456789",
      "timestamp": 1623456789123,
      "stateVersion": 103,
      "globalState": {
        /* 全局状态 */
      },
      "playerStates": {
        "U123456789": {
          /* 玩家状态 */
        }
        // 其他玩家...
      },
      "gamePhase": "TEAM_BUILDING",
      "roundNumber": 2,
      "checksum": "abc123def456"
    }
    ```
  - 存储位置：Redis + MongoDB 冷备份
  - 淘汰策略：仅保留最近 5 个快照，游戏结束后 12 小时删除

- **增量事件日志**：

  - 格式定义：
    ```json
    {
      "eventId": "E123456789",
      "gameId": "G987654321",
      "roomId": "R123456789",
      "timestamp": 1623456789123,
      "sourceStateVersion": 100,
      "targetStateVersion": 101,
      "eventType": "PLAYER_VOTE",
      "originatorId": "U123456789",
      "payload": {
        /* 事件数据 */
      },
      "stateChanges": [
        { "path": "votes.round2.U123456789", "op": "add", "value": true }
      ]
    }
    ```
  - 存储策略：按房间分区的时序数据库
  - 索引优化：基于游戏 ID、房间 ID、时间范围
  - 查询模式：范围查询（开始版本到结束版本）

- **二级缓存架构**：
  - 热数据层：内存缓存，保存当前活跃游戏状态
  - 温数据层：Redis，保存近期快照和事件日志
  - 冷数据层：MongoDB，保存历史游戏记录
  - 数据流向：内存 →Redis→MongoDB（单向）

#### 3.3.3 重连流程与协议

- **重连握手协议**：

  ```json
  // 客户端重连请求
  {
    "header": {
      "msgId": "m123456789",
      "msgType": "RECONNECT_REQUEST",
      "timestamp": 1623456789123
    },
    "payload": {
      "sessionId": "1a2b3c4d-5e6f-7g8h-9i0j",
      "userId": "U123456789",
      "gameId": "G987654321",
      "roomId": "R123456789",
      "lastStateVersion": 95,
      "lastSequenceReceived": 40,
      "deviceInfo": {
        "deviceId": "D123456789",
        "platform": "iOS",
        "networkType": "4G"
      }
    }
  }

  // 服务器重连响应
  {
    "header": {
      "msgId": "m987654321",
      "msgType": "RECONNECT_RESPONSE",
      "timestamp": 1623456789456
    },
    "payload": {
      "status": "SUCCESS",
      "sessionId": "1a2b3c4d-5e6f-7g8h-9i0j",
      "currentStateVersion": 103,
      "recoveryStrategy": "INCREMENTAL",
      "baseStateVersion": 95,
      "gameStatus": "IN_PROGRESS",
      "missingEventCount": 8
    }
  }
  ```

- **状态恢复策略选择**：

  - 决策矩阵：
    | 断线时长 | 状态变化量 | 选择策略 |
    |---------|----------|---------|
    | <30s | <10 个变更 | 增量恢复 |
    | <2min | <50 个变更 | 增量恢复 |
    | <2min | >=50 个变更 | 快照+增量 |
    | >=2min | 任意 | 全量恢复 |

  - 增量恢复：仅发送断线期间的事件和状态变更
  - 快照+增量：发送最近快照和后续变更
  - 全量恢复：发送当前完整游戏状态

- **多阶段恢复流程**：

  1. **初始化阶段**：

     - 传输基本连接参数
     - 认证会话有效性
     - 协商恢复策略

  2. **数据传输阶段**：

     - 根据选定策略传输状态数据
     - 优先传输关键游戏状态
     - 非关键数据后台异步加载

  3. **同步完成阶段**：
     - 状态一致性校验
     - 确认重连完成
     - 通知房间内其他玩家

- **游戏状态冻结机制**：
  - 触发条件：关键角色玩家（如当前队长）断线
  - 冻结范围：特定游戏阶段的操作
  - 最大冻结时间：30 秒
  - 解冻条件：玩家重连或超时

#### 3.3.4 恢复质量保障

- **状态完整性校验**：

  - 校验方法：JSON Schema 验证 + 业务规则校验
  - 关键状态字段双重检查（服务器和客户端）
  - 状态哈希值比对（SHA-256）
  - 增量状态应用后的一致性验证

- **恢复性能指标**：

  - 重连认证时间：<500ms
  - 状态恢复总时间：<2s（标准网络）
  - 关键状态恢复时间：<1s（优先恢复）
  - 后台资源加载时间：<5s

- **降级恢复策略**：

  - 弱网环境：优先恢复最小可玩状态
  - 高延迟网络：增加批处理，减少往返次数
  - 异常情况：恢复静态观战模式
  - 严重网络问题：提供游戏总结信息

- **恢复过程监控**：
  - 实时恢复进度展示
  - 分阶段恢复状态日志
  - 恢复异常主动上报
  - 关键性能指标收集

### 3.4 冲突检测与解决

阿瓦隆游戏的冲突检测与解决机制确保在高并发操作和网络延迟情况下，游戏状态保持一致性，技术规格如下：

#### 3.4.1 冲突类型与检测

- **版本冲突**：

  - 检测方式：基于状态版本号比对
  - 判定条件：当前操作的基础版本号 != 服务器最新版本号
  - 严重度分级：
    - 轻微：版本差异 <= 2 且不涉及关键状态
    - 中等：2 < 版本差异 <= 5 或涉及半关键状态
    - 严重：版本差异 > 5 或涉及关键状态
  - 检测位置：服务器端和客户端双重检测

- **并发操作冲突**：

  - 检测方式：操作时间戳+目标资源路径组合分析
  - 冲突窗口期：200ms 内的同一资源操作视为并发
  - 顺序确定算法：

    ```typescript
    function determineOperationOrder(op1, op2) {
      // 优先级比较
      if (op1.priority !== op2.priority) {
        return op1.priority > op2.priority ? op1 : op2;
      }

      // 时间戳比较（考虑到时钟偏差的容错）
      const timeDiff = Math.abs(op1.timestamp - op2.timestamp);
      if (timeDiff > CLOCK_TOLERANCE_MS) {
        // 通常为50ms
        return op1.timestamp < op2.timestamp ? op1 : op2;
      }

      // 角色权重比较
      if (op1.roleWeight !== op2.roleWeight) {
        return op1.roleWeight > op2.roleWeight ? op1 : op2;
      }

      // 最后使用操作ID的哈希值作为决胜因素
      return hashCode(op1.operationId) < hashCode(op2.operationId) ? op1 : op2;
    }
    ```

- **客户端预测冲突**：

  - 检测方式：服务器实际执行结果与客户端预测结果比对
  - 判定标准：结果状态差异路径集合非空
  - 差异计算：
    ```typescript
    function calculateStateDifference(predictedState, actualState) {
      return {
        additions: findPaths(actualState).filter(
          (p) => !hasPath(predictedState, p)
        ),
        deletions: findPaths(predictedState).filter(
          (p) => !hasPath(actualState, p)
        ),
        modifications: findPaths(actualState).filter(
          (p) =>
            hasPath(predictedState, p) &&
            getValue(predictedState, p) !== getValue(actualState, p)
        ),
      };
    }
    ```
  - 阈值设置：可忽略的微小差异（如动画状态、UI 状态）

- **状态不一致检测**：
  - 检测触发：
    - 定期检查点（30 秒一次）
    - 关键游戏阶段转换
    - 玩家显式请求同步检查
    - 客户端异常行为检测
  - 检测方法：
    - 轻量级：状态校验和比对（SHA-256）
    - 完整性：关键状态字段值比对
    - 深度检查：完整状态树递归比对

#### 3.4.2 冲突解决策略

- **版本冲突解决**：

  - 基本原则：服务器状态为准
  - 解决流程：

    1. 轻微冲突：自动合并非冲突字段，服务器裁决冲突字段
    2. 中等冲突：客户端重新应用操作到最新状态，服务器验证结果
    3. 严重冲突：放弃客户端操作，获取服务器最新状态，提示用户重试

  - 合并策略矩阵：
    | 状态字段类型 | 轻微冲突 | 中等冲突 | 严重冲突 |
    |------------|---------|---------|---------|
    | UI 状态 | 客户端优先 | 客户端优先 | 服务器优先 |
    | 游戏进度 | 服务器优先 | 服务器优先 | 服务器优先 |
    | 玩家操作 | 智能合并 | 重新应用 | 废弃并重试 |
    | 系统状态 | 服务器优先 | 服务器优先 | 服务器优先 |

- **并发操作冲突解决**：

  - 时间优先策略：先到先得，基于服务器时间戳
  - 角色权重策略：特定场景下角色优先级
    - 队长行动优先级 > 普通队员
    - 当前轮次行动玩家 > 其他玩家
    - 梅林角色特殊阶段优先级最高
  - 操作类型优先级：
    - 系统操作 (10) > 管理员操作 (9) > 游戏规则操作 (8) > 玩家关键操作 (7) > 普通操作 (5) > 聊天/表情 (3)
  - 冲突操作处理：
    - 非互斥操作：尽可能同时应用
    - 互斥操作：应用高优先级，拒绝低优先级
    - 部分冲突：拆分操作，应用非冲突部分

- **状态回滚与重播**：

  - 回滚条件：严重冲突且无法前向解决
  - 回滚深度限制：最多回滚到 5 个版本前
  - 回滚过程：
    1. 确定回滚目标版本
    2. 保存当前临时状态
    3. 加载目标版本状态
    4. 按时间顺序重放有效事件
    5. 验证最终状态一致性
  - 重播优化：跳过非关键或已过期事件

- **客户端调整策略**：
  - 预测结果错误处理：
    - 无痕修正：用户无感知的小差异自动修正
    - 动画过渡：有感知差异通过动画平滑过渡
    - 提示并修正：显著差异提示用户并修正
  - 乐观锁重试机制：
    - 重试最大次数：3 次
    - 重试间隔：指数退避（初始 200ms）
    - 重试前状态更新：与服务器最新状态同步

#### 3.4.3 特殊冲突场景处理

- **投票冲突处理**：

  - 场景描述：多人同时投票，最后一票关键决策
  - 特殊规则：
    - 所有投票必须基于相同的游戏状态版本
    - 投票有效时间窗口设定（通常 10 秒）
    - 投票一旦提交不可更改
    - 系统强制等待所有玩家投票或超时
  - 实现机制：

    ```typescript
    class VotingProtocol {
      constructor(baseStateVersion, votingPlayers, timeoutSeconds) {
        this.baseStateVersion = baseStateVersion;
        this.requiredVoters = new Set(votingPlayers);
        this.votes = new Map();
        this.expirationTime = Date.now() + timeoutSeconds * 1000;
        this.locked = false;
      }

      submitVote(playerId, vote, playerStateVersion) {
        if (this.locked) return { success: false, reason: "VOTING_LOCKED" };
        if (!this.requiredVoters.has(playerId))
          return { success: false, reason: "NOT_AUTHORIZED" };
        if (this.baseStateVersion !== playerStateVersion)
          return { success: false, reason: "VERSION_MISMATCH" };
        if (this.votes.has(playerId))
          return { success: false, reason: "ALREADY_VOTED" };

        this.votes.set(playerId, vote);
        return { success: true };
      }

      isComplete() {
        return (
          this.votes.size === this.requiredVoters.size ||
          Date.now() > this.expirationTime
        );
      }

      lockVoting() {
        this.locked = true;
      }

      getResult() {
        if (!this.isComplete()) return null;
        // 计算投票结果
        const result = {
          /* ... */
        };
        return result;
      }
    }
    ```

- **角色技能冲突**：

  - 场景描述：特殊角色技能使用顺序和效果冲突
  - 优先级定义：
    - 防御类技能 > 攻击类技能
    - 先发动的防御类技能 > 后发动的防御类技能
    - 高等级角色技能 > 低等级角色技能
  - 冲突解决流程：
    1. 确定所有同时发动的技能
    2. 按优先级排序
    3. 逐个应用技能效果
    4. 每次应用后检查状态有效性
    5. 忽略因前序技能导致无效的技能

- **网络极端情况**：
  - 场景描述：严重的网络延迟、丢包或分区
  - 处理策略：
    - 多数派原则：当超过半数玩家状态一致时采纳
    - 超时降级：等待响应超过 5 秒自动降级处理
    - 强制同步点：异常状态比例超过 20%时强制同步
    - 部分功能降级：关闭高实时性要求的功能
  - 恢复机制：
    - 网络恢复后立即强制同步
    - 分批次恢复完整状态
    - 优先恢复游戏核心功能

#### 3.4.4 冲突监控与诊断

- **冲突数据收集**：

  - 收集维度：
    - 冲突类型和频率
    - 冲突发生游戏阶段
    - 冲突解决策略效果
    - 冲突对用户体验的影响
  - 存储结构：
    ```json
    {
      "conflictId": "C123456789",
      "gameId": "G987654321",
      "roomId": "R123456789",
      "timestamp": 1623456789123,
      "conflictType": "VERSION_CONFLICT",
      "severity": "MEDIUM",
      "involvedPlayers": ["U123456789", "U987654321"],
      "stateVersions": {
        "server": 105,
        "client1": 103,
        "client2": 105
      },
      "resolutionStrategy": "CLIENT_RETRY",
      "resolutionSuccess": true,
      "resolutionTime": 320,
      "impactedGamePhase": "TEAM_BUILDING"
    }
    ```
  - 冲突率计算：冲突操作数 / 总操作数

- **实时监控指标**：

  - 关键指标：
    - 冲突发生率（每分钟）
    - 冲突解决成功率
    - 平均冲突解决时间
    - 状态一致性水平（百分比）
  - 阈值告警：
    - 冲突率 > 5%触发警告
    - 解决失败率 > 1%触发警告
    - 任何严重冲突立即通知
  - 监控维度：按游戏房间、游戏阶段、客户端版本

- **诊断工具**：

  - 冲突重放工具：模拟冲突场景验证解决方案
  - 状态比对器：可视化显示状态树差异
  - 操作时序分析：重建操作时间线并分析依赖
  - 网络条件模拟：测试不同网络环境下的冲突处理

- **优化反馈循环**：
  - 冲突热点识别算法
  - 自适应调整冲突解决策略
  - A/B 测试不同冲突解决方案
  - 基于历史数据预测冲突风险

## 4. 实现细节

### 4.1 WebSocket 服务实现

阿瓦隆游戏的 WebSocket 服务是实时状态同步的核心组件，负责维护客户端与服务器之间的持久连接，处理消息的路由和分发。具体实现需要考虑以下关键方面：

#### 4.1.1 服务架构

- **多层架构设计**：

  - 连接层：负责 WebSocket 连接的建立、维护和关闭
  - 会话层：管理用户会话状态和认证
  - 消息处理层：处理消息的序列化/反序列化和路由
  - 业务逻辑层：实现游戏状态的处理和更新
  - 持久化层：负责状态的存储和恢复

- **集群部署考量**：

  - 无状态设计：核心业务逻辑无状态化，便于水平扩展
  - 会话亲和性：同一玩家的连接尽量路由到同一节点
  - 房间分片：基于房间 ID 的一致性哈希分片
  - 节点间通信：使用 Redis Pub/Sub 实现跨节点消息转发

- **服务发现与负载均衡**：

  - 服务注册：WebSocket 服务节点自动注册到服务中心
  - 健康检查：定期心跳检测节点健康状态
  - 负载均衡算法：考虑节点负载和网络延迟的加权轮询
  - 会话保持：确保客户端请求路由到指定节点

- **WebSocket 服务 UML 类图**：

```plantuml
@startuml WebSocket服务类图

package "连接层" {
  class WebSocketServer {
    - port: number
    - connections: Map<string, WebSocketConnection>
    + start(): void
    + stop(): void
    + broadcastToAll(message: Message): void
    + getConnectionCount(): number
  }

  class WebSocketConnection {
    - id: string
    - socket: WebSocket
    - sessionId: string | null
    - userId: string | null
    - roomId: string | null
    - status: ConnectionStatus
    - lastActivity: Date
    - pingTimeout: NodeJS.Timeout | null
    + send(message: Message): boolean
    + close(code: number, reason: string): void
    + isAlive(): boolean
    + updateActivity(): void
  }

  enum ConnectionStatus {
    CONNECTING
    AUTHENTICATING
    ACTIVE
    INACTIVE
    RECONNECTING
    CLOSED
  }

  WebSocketServer "1" *-- "0..*" WebSocketConnection
  WebSocketConnection -- ConnectionStatus
}

package "会话层" {
  class SessionManager {
    - sessions: Map<string, Session>
    + createSession(userId: string, connectionId: string): Session
    + getSession(sessionId: string): Session
    + removeSession(sessionId: string): boolean
    + linkConnectionToSession(connectionId: string, sessionId: string): boolean
  }

  class Session {
    - id: string
    - userId: string
    - connectionIds: string[]
    - createdAt: Date
    - lastAccess: Date
    - data: SessionData
    + isExpired(): boolean
    + refresh(): void
    + setData(key: string, value: any): void
    + getData(key: string): any
  }

  class AuthenticationService {
    + authenticate(token: string): Promise<User>
    + generateToken(user: User): string
    + validateSession(sessionId: string, userId: string): boolean
  }

  SessionManager "1" *-- "0..*" Session
}

package "消息处理层" {
  class MessageRouter {
    - handlers: Map<string, MessageHandler>
    + registerHandler(type: string, handler: MessageHandler): void
    + route(message: Message): Promise<void>
    + broadcastToRoom(roomId: string, message: Message): void
    + sendToUser(userId: string, message: Message): void
  }

  interface MessageHandler {
    + handle(message: Message, connection: WebSocketConnection): Promise<void>
  }

  class MessageSerializer {
    + serialize(message: Message): Buffer
    + deserialize(data: Buffer): Message
    + validate(message: Message): boolean
  }

  class Message {
    + header: MessageHeader
    + payload: any
    + meta: MessageMetadata
  }

  MessageRouter -- MessageHandler
  MessageRouter -- Message
  MessageSerializer -- Message
}

package "房间管理" {
  class RoomManager {
    - rooms: Map<string, GameRoom>
    + createRoom(roomId: string, config: RoomConfig): GameRoom
    + getRoom(roomId: string): GameRoom
    + removeRoom(roomId: string): boolean
    + addUserToRoom(roomId: string, userId: string): boolean
    + removeUserFromRoom(roomId: string, userId: string): boolean
  }

  class GameRoom {
    - id: string
    - userIds: string[]
    - state: GameState
    - createdAt: Date
    - status: RoomStatus
    + broadcastMessage(message: Message): void
    + broadcastToRole(role: PlayerRole, message: Message): void
    + getState(): GameState
    + updateState(changes: StateChange[]): void
  }

  enum RoomStatus {
    CREATED
    WAITING
    IN_PROGRESS
    FINISHED
    ARCHIVED
  }

  RoomManager "1" *-- "0..*" GameRoom
  GameRoom -- RoomStatus
}

package "状态管理" {
  class GameStateManager {
    - stateCache: Map<string, GameState>
    + getState(roomId: string): GameState
    + applyChanges(roomId: string, changes: StateChange[]): GameState
    + saveSnapshot(roomId: string): string
    + restoreFromSnapshot(snapshotId: string): GameState
  }

  class GameState {
    - version: number
    - global: object
    - players: Map<string, PlayerState>
    - timestamp: Date
    + applyChanges(changes: StateChange[]): GameState
    + getVersion(): number
    + calculateDiff(otherState: GameState): StateChange[]
    + validate(): boolean
  }

  class StateChange {
    + path: string
    + operation: ChangeOperation
    + value: any
    + previousValue: any
  }

  enum ChangeOperation {
    ADD
    REPLACE
    REMOVE
    MOVE
  }

  GameStateManager "1" *-- "0..*" GameState
  GameState -- StateChange
  StateChange -- ChangeOperation
}

package "持久化" {
  class StateStorageService {
    + saveState(roomId: string, state: GameState): Promise<string>
    + loadState(roomId: string): Promise<GameState>
    + saveEvent(event: GameEvent): Promise<void>
    + getEvents(roomId: string, fromVersion: number, toVersion: number): Promise<GameEvent[]>
  }

  class GameEvent {
    + id: string
    + roomId: string
    + type: string
    + data: any
    + sourceVersion: number
    + targetVersion: number
    + timestamp: Date
    + createdBy: string
  }

  StateStorageService -- GameEvent
  StateStorageService -- GameState
}

' 关系连接
WebSocketServer -- SessionManager
WebSocketConnection -- Session
SessionManager -- AuthenticationService
WebSocketServer -- MessageRouter
RoomManager -- MessageRouter
GameStateManager -- RoomManager
StateStorageService -- GameStateManager

@enduml
```

#### 4.1.2 连接生命周期管理

- **连接建立流程**：

  1. 握手阶段：验证 WebSocket 协议与子协议
  2. 认证阶段：验证 JWT 令牌的有效性和权限
  3. 初始化阶段：创建连接上下文和会话关联
  4. 加入房间：将连接加入到相应的游戏房间通道
  5. 状态同步：发送初始游戏状态

- **连接维护策略**：

  - 心跳机制：客户端 20 秒发送一次心跳，40 秒无心跳判定断开
  - 连接状态监控：实时监控连接健康状态和延迟
  - 连接恢复优化：断线后快速恢复而非重新建立
  - 网络条件适应：根据网络质量调整消息发送策略

- **连接关闭处理**：

  - 正常关闭：客户端主动关闭或会话结束
  - 异常关闭：网络中断或服务器故障
  - 关闭后处理：保留会话状态，启动重连等待机制
  - 资源清理：释放连接相关资源，但保留必要状态数据

- **连接监控与统计**：

  - 实时指标：活跃连接数、消息吞吐量、平均延迟
  - 异常监控：连接错误率、异常关闭率、重连失败率
  - 性能分析：连接建立时间、消息处理时间
  - 资源占用：内存、CPU、网络带宽使用情况

- **WebSocket 服务核心实现示例**：

```typescript
// WebSocket服务器核心实现示例（基于Node.js和ws库）

import * as WebSocket from "ws";
import * as http from "http";
import { v4 as uuidv4 } from "uuid";
import { authenticate } from "./auth-service";
import { SessionManager } from "./session-manager";
import { MessageRouter } from "./message-router";
import { RoomManager } from "./room-manager";
import { Logger } from "./logger";

enum ConnectionStatus {
  CONNECTING = "connecting",
  AUTHENTICATING = "authenticating",
  ACTIVE = "active",
  INACTIVE = "inactive",
  RECONNECTING = "reconnecting",
  CLOSED = "closed",
}

class WebSocketConnection {
  private id: string;
  private socket: WebSocket;
  private sessionId: string | null;
  private userId: string | null;
  private roomId: string | null;
  private status: ConnectionStatus;
  private lastActivity: Date;
  private pingTimeout: NodeJS.Timeout | null;

  constructor(socket: WebSocket, req: http.IncomingMessage) {
    this.id = uuidv4();
    this.socket = socket;
    this.sessionId = null;
    this.userId = null;
    this.roomId = null;
    this.status = ConnectionStatus.CONNECTING;
    this.lastActivity = new Date();
    this.pingTimeout = null;

    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.socket.on("message", this.handleMessage.bind(this));
    this.socket.on("close", this.handleClose.bind(this));
    this.socket.on("error", this.handleError.bind(this));
    this.socket.on("pong", this.heartbeat.bind(this));

    // 设置心跳检测定时器
    this.schedulePing();
  }

  private async handleMessage(data: WebSocket.Data): Promise<void> {
    this.updateActivity();

    try {
      const message = JSON.parse(data.toString());
      Logger.debug(`Received message from connection ${this.id}`, {
        messageType: message.header?.msgType,
      });

      // 处理不同消息类型
      switch (this.status) {
        case ConnectionStatus.CONNECTING:
          if (message.header?.msgType === "HANDSHAKE") {
            await this.handleHandshake(message);
          } else {
            this.sendError("Expected HANDSHAKE message");
          }
          break;

        case ConnectionStatus.AUTHENTICATING:
          if (message.header?.msgType === "AUTHENTICATE") {
            await this.handleAuthentication(message);
          } else {
            this.sendError("Expected AUTHENTICATE message");
          }
          break;

        case ConnectionStatus.ACTIVE:
        case ConnectionStatus.RECONNECTING:
          // 路由消息到对应的处理器
          await WebSocketServer.messageRouter.route(message, this);
          break;

        case ConnectionStatus.INACTIVE:
          if (message.header?.msgType === "RECONNECT_REQUEST") {
            await this.handleReconnect(message);
          } else {
            // 非活跃状态下，只处理重连请求
            this.sendError("Connection inactive, reconnect required");
          }
          break;

        case ConnectionStatus.CLOSED:
          Logger.warn(`Received message for closed connection ${this.id}`);
          break;
      }
    } catch (error) {
      Logger.error(`Error handling message for connection ${this.id}`, error);
      this.sendError("Failed to process message");
    }
  }

  private async handleHandshake(message: any): Promise<void> {
    try {
      // 验证子协议版本兼容性
      const clientProtocolVersion = message.payload?.protocolVersion;
      if (!isCompatibleVersion(clientProtocolVersion)) {
        return this.sendError("Incompatible protocol version", true);
      }

      // 发送握手确认
      this.send({
        header: {
          msgId: uuidv4(),
          msgType: "HANDSHAKE_RESPONSE",
          timestamp: Date.now(),
        },
        payload: {
          status: "SUCCESS",
          serverVersion: SERVER_VERSION,
          sessionTimeout: SESSION_TIMEOUT_MS,
          heartbeatInterval: HEARTBEAT_INTERVAL_MS,
        },
        meta: {
          compression: "none",
          encryption: "none",
        },
      });

      // 更新状态为等待认证
      this.status = ConnectionStatus.AUTHENTICATING;
    } catch (error) {
      Logger.error(`Handshake failed for connection ${this.id}`, error);
      this.sendError("Handshake failed", true);
    }
  }

  private async handleAuthentication(message: any): Promise<void> {
    try {
      const authToken = message.payload?.token;
      if (!authToken) {
        return this.sendError("Authentication token required");
      }

      // 验证认证令牌
      const authResult = await authenticate(authToken);
      if (!authResult.success) {
        return this.sendError(`Authentication failed: ${authResult.reason}`);
      }

      this.userId = authResult.userId;

      // 检查是否有现有会话
      const existingSession =
        await WebSocketServer.sessionManager.findSessionByUserId(this.userId);
      if (existingSession) {
        this.sessionId = existingSession.id;
        await WebSocketServer.sessionManager.linkConnectionToSession(
          this.id,
          this.sessionId
        );
      } else {
        // 创建新会话
        const session = await WebSocketServer.sessionManager.createSession(
          this.userId
        );
        this.sessionId = session.id;
        await WebSocketServer.sessionManager.linkConnectionToSession(
          this.id,
          this.sessionId
        );
      }

      // 获取用户要加入的房间ID
      this.roomId = message.payload?.roomId;
      if (this.roomId) {
        // 将用户加入到游戏房间
        const joinResult = await WebSocketServer.roomManager.addUserToRoom(
          this.roomId,
          this.userId,
          this.id
        );
        if (!joinResult.success) {
          return this.sendError(`Failed to join room: ${joinResult.reason}`);
        }
      }

      // 认证成功响应
      this.send({
        header: {
          msgId: uuidv4(),
          msgType: "AUTHENTICATE_RESPONSE",
          timestamp: Date.now(),
        },
        payload: {
          status: "SUCCESS",
          sessionId: this.sessionId,
          userId: this.userId,
        },
        meta: {
          compression: "none",
          encryption: "none",
        },
      });

      this.status = ConnectionStatus.ACTIVE;

      // 如果加入了房间，发送当前游戏状态
      if (this.roomId) {
        await this.sendInitialGameState();
      }
    } catch (error) {
      Logger.error(`Authentication failed for connection ${this.id}`, error);
      this.sendError("Authentication process failed");
    }
  }

  private async sendInitialGameState(): Promise<void> {
    if (!this.roomId || !this.userId) return;

    try {
      // 获取当前游戏状态
      const gameState = await WebSocketServer.roomManager.getGameState(
        this.roomId
      );
      if (!gameState) {
        return this.sendError("Game state not available");
      }

      // 根据玩家角色过滤状态
      const filteredState = WebSocketServer.stateFilter.filterForUser(
        gameState,
        this.userId
      );

      // 发送全量游戏状态
      this.send({
        header: {
          msgId: uuidv4(),
          msgType: "FULL_STATE",
          gameId: gameState.gameId,
          roomId: this.roomId,
          senderId: "SERVER",
          timestamp: Date.now(),
          sequence: 0,
          version: 1,
        },
        payload: filteredState,
        meta: {
          compression: gameState.isLarge ? "gzip" : "none",
          encryption: "none",
          priority: 7,
        },
      });
    } catch (error) {
      Logger.error(
        `Failed to send initial game state for connection ${this.id}`,
        error
      );
      this.sendError("Failed to retrieve game state");
    }
  }

  private async handleReconnect(message: any): Promise<void> {
    try {
      const sessionId = message.payload?.sessionId;
      if (!sessionId) {
        return this.sendError("Session ID required for reconnection");
      }

      // 验证会话是否存在且有效
      const session = await WebSocketServer.sessionManager.getSession(
        sessionId
      );
      if (!session) {
        return this.sendError("Invalid or expired session");
      }

      // 验证会话所有者
      if (session.userId !== this.userId) {
        return this.sendError("Session ownership mismatch");
      }

      this.sessionId = sessionId;
      this.status = ConnectionStatus.RECONNECTING;

      // 更新连接与会话的关联
      await WebSocketServer.sessionManager.linkConnectionToSession(
        this.id,
        this.sessionId
      );

      // 获取最新的房间ID
      this.roomId = session.getData("roomId");

      // 恢复状态
      if (this.roomId) {
        // 重新添加到房间
        await WebSocketServer.roomManager.reconnectUserToRoom(
          this.roomId,
          this.userId,
          this.id
        );

        // 获取断线期间错过的状态更新
        const lastStateVersion = message.payload?.lastStateVersion || 0;
        const recoveryStrategy = determineRecoveryStrategy(
          lastStateVersion,
          session.getData("lastDisconnectTime"),
          Date.now()
        );

        // 发送重连响应
        this.send({
          header: {
            msgId: uuidv4(),
            msgType: "RECONNECT_RESPONSE",
            timestamp: Date.now(),
          },
          payload: {
            status: "SUCCESS",
            recoveryStrategy,
            currentStateVersion:
              await WebSocketServer.roomManager.getCurrentStateVersion(
                this.roomId
              ),
          },
          meta: {
            compression: "none",
            encryption: "none",
          },
        });

        // 根据恢复策略发送相应的状态数据
        await this.sendRecoveryState(recoveryStrategy, lastStateVersion);
      } else {
        // 没有加入房间的情况
        this.send({
          header: {
            msgId: uuidv4(),
            msgType: "RECONNECT_RESPONSE",
            timestamp: Date.now(),
          },
          payload: {
            status: "SUCCESS",
            recoveryStrategy: "NONE",
          },
          meta: {
            compression: "none",
            encryption: "none",
          },
        });
      }

      this.status = ConnectionStatus.ACTIVE;
    } catch (error) {
      Logger.error(`Reconnection failed for connection ${this.id}`, error);
      this.sendError("Reconnection process failed");
    }
  }

  private async sendRecoveryState(
    strategy: string,
    lastStateVersion: number
  ): Promise<void> {
    if (!this.roomId || !this.userId) return;

    try {
      switch (strategy) {
        case "FULL":
          // 发送全量状态
          await this.sendInitialGameState();
          break;

        case "SNAPSHOT_PLUS_DELTA":
          // 发送最近快照和增量更新
          const snapshotId =
            await WebSocketServer.roomManager.getNearestSnapshotId(
              this.roomId,
              lastStateVersion
            );
          const snapshot = await WebSocketServer.roomManager.getStateSnapshot(
            snapshotId
          );

          if (snapshot) {
            // 发送快照状态
            this.send({
              header: {
                msgId: uuidv4(),
                msgType: "SNAPSHOT_STATE",
                gameId: snapshot.gameId,
                roomId: this.roomId,
                senderId: "SERVER",
                timestamp: Date.now(),
                sequence: 0,
                version: 1,
              },
              payload: WebSocketServer.stateFilter.filterForUser(
                snapshot,
                this.userId
              ),
              meta: {
                compression: snapshot.isLarge ? "gzip" : "none",
                encryption: "none",
                priority: 7,
              },
            });

            // 发送快照之后的增量更新
            const events = await WebSocketServer.roomManager.getEventsSince(
              this.roomId,
              snapshot.version
            );
            for (const event of events) {
              this.send({
                header: {
                  msgId: uuidv4(),
                  msgType: "STATE_UPDATE",
                  gameId: event.gameId,
                  roomId: this.roomId,
                  senderId: "SERVER",
                  timestamp: event.timestamp,
                  sequence: event.sequence,
                  version: 1,
                },
                payload: {
                  stateVersion: event.targetVersion,
                  baseVersion: event.sourceVersion,
                  updateType: "incremental",
                  changes: WebSocketServer.stateFilter.filterChangesForUser(
                    event.changes,
                    this.userId
                  ),
                  events: [event],
                },
                meta: {
                  compression: "none",
                  encryption: "none",
                  priority: 6,
                },
              });
            }
          }
          break;

        case "DELTA":
          // 只发送增量更新
          const events = await WebSocketServer.roomManager.getEventsSince(
            this.roomId,
            lastStateVersion
          );
          for (const event of events) {
            this.send({
              header: {
                msgId: uuidv4(),
                msgType: "STATE_UPDATE",
                gameId: event.gameId,
                roomId: this.roomId,
                senderId: "SERVER",
                timestamp: event.timestamp,
                sequence: event.sequence,
                version: 1,
              },
              payload: {
                stateVersion: event.targetVersion,
                baseVersion: event.sourceVersion,
                updateType: "incremental",
                changes: WebSocketServer.stateFilter.filterChangesForUser(
                  event.changes,
                  this.userId
                ),
                events: [event],
              },
              meta: {
                compression: "none",
                encryption: "none",
                priority: 6,
              },
            });
          }
          break;
      }
    } catch (error) {
      Logger.error(
        `Failed to send recovery state for connection ${this.id}`,
        error
      );
      this.sendError("Failed to retrieve recovery state");
    }
  }

  private handleClose(code: number, reason: string): void {
    Logger.info(`Connection ${this.id} closed: ${code} - ${reason}`);

    // 清理心跳检测定时器
    if (this.pingTimeout) {
      clearTimeout(this.pingTimeout);
      this.pingTimeout = null;
    }

    // 记录断开连接时间
    if (this.sessionId) {
      WebSocketServer.sessionManager.updateSessionData(
        this.sessionId,
        "lastDisconnectTime",
        Date.now()
      );
    }

    // 如果处于活动状态，更新状态为非活跃
    if (this.status === ConnectionStatus.ACTIVE) {
      this.status = ConnectionStatus.INACTIVE;

      // 通知房间用户断开连接
      if (this.roomId && this.userId) {
        WebSocketServer.roomManager.notifyUserDisconnected(
          this.roomId,
          this.userId
        );
      }
    }

    // 延迟移除用户，以便支持快速重连
    setTimeout(() => {
      if (this.status === ConnectionStatus.INACTIVE) {
        this.status = ConnectionStatus.CLOSED;

        // 从房间中移除用户
        if (this.roomId && this.userId) {
          WebSocketServer.roomManager.removeUserFromRoom(
            this.roomId,
            this.userId
          );
        }

        // 解除连接与会话的绑定
        if (this.sessionId) {
          WebSocketServer.sessionManager.unlinkConnectionFromSession(
            this.id,
            this.sessionId
          );
        }

        // 从连接池中移除
        WebSocketServer.removeConnection(this.id);
      }
    }, CONNECTION_CLEANUP_DELAY);
  }

  private handleError(error: Error): void {
    Logger.error(`Error on connection ${this.id}`, error);

    // 根据错误类型决定是否关闭连接
    if (isFatalError(error)) {
      this.close(1011, "Internal server error");
    }
  }

  private heartbeat(): void {
    this.updateActivity();

    // 重置心跳检测超时
    if (this.pingTimeout) {
      clearTimeout(this.pingTimeout);
    }
    this.schedulePing();
  }

  private schedulePing(): void {
    this.pingTimeout = setTimeout(() => {
      // 如果连接仍然开放
      if (this.socket.readyState === WebSocket.OPEN) {
        // 发送ping
        this.socket.ping();

        // 设置等待pong的超时，超时后关闭连接
        setTimeout(() => {
          if (Date.now() - this.lastActivity.getTime() > HEARTBEAT_TIMEOUT_MS) {
            Logger.warn(`Connection ${this.id} heartbeat timeout`);
            this.close(1008, "Heartbeat timeout");
          }
        }, HEARTBEAT_WAIT_MS);
      }
    }, HEARTBEAT_INTERVAL_MS);
  }

  public send(message: any): boolean {
    if (this.socket.readyState !== WebSocket.OPEN) {
      Logger.warn(
        `Attempted to send message to non-open connection ${this.id}`
      );
      return false;
    }

    try {
      // 序列化消息
      const serialized = JSON.stringify(message);

      // 根据消息大小决定是否压缩
      if (
        serialized.length > COMPRESSION_THRESHOLD &&
        message.meta.compression === "none"
      ) {
        message.meta.compression = "gzip";
        const compressed = compressMessage(serialized);
        this.socket.send(compressed);
      } else {
        this.socket.send(serialized);
      }

      return true;
    } catch (error) {
      Logger.error(`Failed to send message to connection ${this.id}`, error);
      return false;
    }
  }

  public sendError(errorMessage: string, close: boolean = false): void {
    const errorResponse = {
      header: {
        msgId: uuidv4(),
        msgType: "ERROR",
        timestamp: Date.now(),
      },
      payload: {
        error: errorMessage,
      },
      meta: {
        compression: "none",
        encryption: "none",
      },
    };

    this.send(errorResponse);

    if (close) {
      this.close(1008, errorMessage);
    }
  }

  public close(code: number = 1000, reason: string = "Normal closure"): void {
    if (this.socket.readyState === WebSocket.OPEN) {
      this.socket.close(code, reason);
    }
  }

  public updateActivity(): void {
    this.lastActivity = new Date();
  }

  public isAlive(): boolean {
    return (
      this.status === ConnectionStatus.ACTIVE ||
      this.status === ConnectionStatus.RECONNECTING
    );
  }

  // Getters
  public getId(): string {
    return this.id;
  }

  public getSessionId(): string | null {
    return this.sessionId;
  }

  public getUserId(): string | null {
    return this.userId;
  }

  public getRoomId(): string | null {
    return this.roomId;
  }

  public getStatus(): ConnectionStatus {
    return this.status;
  }
}

class WebSocketServer {
  private static instance: WebSocketServer;
  private server: WebSocket.Server;
  private connections: Map<string, WebSocketConnection>;
  private httpServer: http.Server;

  public static sessionManager: SessionManager;
  public static roomManager: RoomManager;
  public static messageRouter: MessageRouter;
  public static stateFilter: StateFilter;

  private constructor(
    httpServer: http.Server,
    options: WebSocket.ServerOptions
  ) {
    this.httpServer = httpServer;
    this.server = new WebSocket.Server({ ...options, server: httpServer });
    this.connections = new Map();

    this.setupEventHandlers();
  }

  public static getInstance(
    httpServer: http.Server,
    options: WebSocket.ServerOptions
  ): WebSocketServer {
    if (!WebSocketServer.instance) {
      WebSocketServer.instance = new WebSocketServer(httpServer, options);
    }
    return WebSocketServer.instance;
  }

  private setupEventHandlers(): void {
    this.server.on("connection", this.handleConnection.bind(this));
    this.server.on("error", this.handleServerError.bind(this));

    // 设置定期清理无效连接的任务
    setInterval(
      this.cleanupConnections.bind(this),
      CONNECTION_CLEANUP_INTERVAL
    );
  }

  private handleConnection(socket: WebSocket, req: http.IncomingMessage): void {
    // 验证子协议
    const protocol = req.headers["sec-websocket-protocol"];
    if (!protocol || !protocol.includes("avalon-game-v1")) {
      socket.close(1002, "Protocol not supported");
      return;
    }

    // 创建新的连接对象
    const connection = new WebSocketConnection(socket, req);

    // 将连接添加到连接池
    this.connections.set(connection.getId(), connection);

    Logger.info(`New WebSocket connection established: ${connection.getId()}`);
  }

  private handleServerError(error: Error): void {
    Logger.error("WebSocket server error", error);
  }

  private cleanupConnections(): void {
    const now = Date.now();
    for (const [id, connection] of this.connections.entries()) {
      // 关闭长时间非活跃的连接
      if (connection.getStatus() === ConnectionStatus.INACTIVE) {
        const lastActivity = connection["lastActivity"].getTime();
        if (now - lastActivity > SESSION_TIMEOUT_MS) {
          Logger.info(`Closing inactive connection ${id} due to timeout`);
          connection.close(1001, "Session timeout");
        }
      }
    }
  }

  public static removeConnection(connectionId: string): void {
    WebSocketServer.instance.connections.delete(connectionId);
  }

  public broadcast(
    message: any,
    filter?: (connection: WebSocketConnection) => boolean
  ): void {
    for (const connection of this.connections.values()) {
      if (connection.isAlive() && (!filter || filter(connection))) {
        connection.send(message);
      }
    }
  }

  public broadcastToRoom(roomId: string, message: any): void {
    this.broadcast(message, (conn) => conn.getRoomId() === roomId);
  }

  public sendToUser(userId: string, message: any): void {
    for (const connection of this.connections.values()) {
      if (connection.isAlive() && connection.getUserId() === userId) {
        connection.send(message);
        break; // 只发送给该用户的一个连接
      }
    }
  }

  public getConnectionCount(): number {
    return this.connections.size;
  }

  public getActiveConnectionCount(): number {
    let count = 0;
    for (const connection of this.connections.values()) {
      if (connection.isAlive()) {
        count++;
      }
    }
    return count;
  }

  public start(): void {
    Logger.info(
      `WebSocket server started on port ${
        (this.httpServer.address() as any).port
      }`
    );
  }

  public stop(): Promise<void> {
    return new Promise((resolve, reject) => {
      // 关闭所有连接
      for (const connection of this.connections.values()) {
        connection.close(1001, "Server shutting down");
      }

      // 关闭WebSocket服务器
      this.server.close((err) => {
        if (err) {
          Logger.error("Error closing WebSocket server", err);
          reject(err);
        } else {
          Logger.info("WebSocket server stopped");
          resolve();
        }
      });
    });
  }
}

// 辅助函数
function isCompatibleVersion(clientVersion: string): boolean {
  if (!clientVersion) return false;

  // 简单版本兼容性检查
  const serverMajorVersion = parseInt(SERVER_VERSION.split(".")[0]);
  const clientMajorVersion = parseInt(clientVersion.split(".")[0]);

  return clientMajorVersion === serverMajorVersion;
}

function determineRecoveryStrategy(
  lastStateVersion: number,
  disconnectTime: number,
  currentTime: number
): string {
  const timeDiff = currentTime - disconnectTime;

  if (timeDiff > 120000) {
    // 2分钟以上
    return "FULL";
  }

  if (timeDiff > 30000 || !lastStateVersion) {
    // 30秒以上或无版本信息
    return "SNAPSHOT_PLUS_DELTA";
  }

  return "DELTA";
}

function isFatalError(error: Error): boolean {
  // 判断是否为致命错误
  return (
    error.message.includes("ECONNRESET") ||
    error.message.includes("EPIPE") ||
    error.message.includes("ETIMEDOUT")
  );
}

function compressMessage(message: string): Buffer {
  // 压缩消息实现
  // 此处使用gzip或其他压缩算法
  return Buffer.from(message); // 简化实现
}

// 常量定义
const SERVER_VERSION = "1.0.0";
const SESSION_TIMEOUT_MS = 300000; // 5分钟
const HEARTBEAT_INTERVAL_MS = 20000; // 20秒
const HEARTBEAT_TIMEOUT_MS = 40000; // 40秒
const HEARTBEAT_WAIT_MS = 5000; // 5秒
const CONNECTION_CLEANUP_DELAY = 60000; // 1分钟
const CONNECTION_CLEANUP_INTERVAL = 300000; // 5分钟
const COMPRESSION_THRESHOLD = 1024; // 1KB

export { WebSocketServer, WebSocketConnection, ConnectionStatus };
```

#### 4.1.3 房间管理与消息路由

- **房间生命周期**：

  - 房间创建：游戏开始时创建房间通信通道
  - 房间状态管理：维护房间内玩家连接和游戏状态
  - 房间销毁：游戏结束后延迟销毁（保留历史记录）
  - 房间恢复：从持久化存储恢复房间状态

- **消息路由策略**：

  - 点对点路由：特定玩家间的私有消息
  - 房间广播：发送给房间内所有玩家的消息
  - 角色组播：只发送给特定角色玩家的消息
  - 选择性路由：根据消息内容和玩家权限决定路由

- **消息队列与优先级**：

  - 多级队列：基于优先级的消息队列
  - 流量整形：高峰期消息发送速率控制
  - 批量处理：非关键消息的批量发送
  - 超时处理：长时间未发送成功的消息处理策略

- **房间数据一致性保障**：

  ```plantuml
  @startuml 房间状态同步流程

  actor "玩家1" as Player1
  actor "玩家2" as Player2
  participant "WebSocket服务" as WSS
  participant "房间管理器" as RM
  participant "游戏状态管理器" as GSM
  participant "消息路由器" as MR
  database "状态存储" as SS

  == 房间创建 ==
  Player1 -> WSS: 创建游戏房间
  WSS -> RM: createRoom(roomConfig)
  RM -> GSM: initializeState(roomId)
  GSM -> SS: saveInitialState(roomId, state)
  RM --> WSS: 房间创建成功
  WSS --> Player1: 房间创建响应

  == 玩家加入 ==
  Player2 -> WSS: 加入房间请求
  WSS -> RM: addUserToRoom(roomId, userId)
  RM -> GSM: getState(roomId)
  GSM --> RM: 当前游戏状态
  RM -> MR: broadcastToRoom(roomId, JOIN_NOTIFICATION)
  MR --> Player1: 玩家加入通知
  RM --> WSS: 加入成功
  WSS --> Player2: 加入响应 + 全量游戏状态

  == 游戏操作与状态同步 ==
  Player1 -> WSS: 游戏操作请求
  WSS -> MR: route(PLAYER_ACTION)
  MR -> RM: handleAction(roomId, action)
  RM -> GSM: applyChanges(roomId, changes)
  GSM -> SS: saveEvent(event)
  GSM -> SS: updateState(roomId, newState)
  GSM --> RM: 更新后的状态
  RM -> MR: broadcastToRoom(roomId, STATE_UPDATE)
  MR --> Player1: 状态更新
  MR --> Player2: 状态更新

  == 房间销毁 ==
  Player1 -> WSS: 结束游戏请求
  WSS -> RM: finishGame(roomId)
  RM -> GSM: finalizeState(roomId)
  GSM -> SS: archiveState(roomId)
  RM -> RM: scheduleRoomCleanup(roomId, delay)
  RM -> MR: broadcastToRoom(roomId, GAME_ENDED)
  MR --> Player1: 游戏结束通知
  MR --> Player2: 游戏结束通知

  @enduml
  ```

- **房间管理器实现示例**：

```typescript
// 房间管理器实现示例

import { EventEmitter } from "events";
import { v4 as uuidv4 } from "uuid";
import { GameState } from "./game-state";
import { StateStorageService } from "./state-storage-service";
import { MessageRouter } from "./message-router";
import { Logger } from "./logger";
import { StateFilter } from "./state-filter";
import { GameEngine } from "./game-engine";

enum RoomStatus {
  CREATED = "created",
  WAITING = "waiting",
  IN_PROGRESS = "in_progress",
  FINISHED = "finished",
  ARCHIVED = "archived",
}

interface RoomConfig {
  maxPlayers: number;
  gameMode: string;
  isPrivate: boolean;
  password?: string;
  autoStart: boolean;
  roleConfig: any;
}

class GameRoom extends EventEmitter {
  private id: string;
  private userIds: Set<string>;
  private connectionIds: Map<string, string>; // userId -> connectionId
  private spectatorIds: Set<string>;
  private state: GameState;
  private createdAt: Date;
  private status: RoomStatus;
  private config: RoomConfig;
  private cleanupTimeout: NodeJS.Timeout | null;
  private gameEngine: GameEngine;

  constructor(id: string, config: RoomConfig) {
    super();
    this.id = id;
    this.userIds = new Set();
    this.connectionIds = new Map();
    this.spectatorIds = new Set();
    this.state = new GameState();
    this.createdAt = new Date();
    this.status = RoomStatus.CREATED;
    this.config = config;
    this.cleanupTimeout = null;
    this.gameEngine = new GameEngine(this.state, config);

    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.gameEngine.on("stateChanged", (changes) => {
      this.emit("stateChanged", changes);
    });

    this.gameEngine.on("gamePhaseChanged", (newPhase) => {
      this.emit("gamePhaseChanged", newPhase);
    });

    this.gameEngine.on("gameEnded", (result) => {
      this.status = RoomStatus.FINISHED;
      this.emit("gameEnded", result);
    });
  }

  public getId(): string {
    return this.id;
  }

  public getState(): GameState {
    return this.state;
  }

  public getStatus(): RoomStatus {
    return this.status;
  }

  public addUser(
    userId: string,
    connectionId: string,
    isSpectator: boolean = false
  ): boolean {
    if (isSpectator) {
      this.spectatorIds.add(userId);
      this.connectionIds.set(userId, connectionId);
      return true;
    }

    // 检查玩家数量是否达到上限
    if (
      this.userIds.size >= this.config.maxPlayers &&
      !this.userIds.has(userId)
    ) {
      return false;
    }

    this.userIds.add(userId);
    this.connectionIds.set(userId, connectionId);

    // 如果是等待状态且自动开始，检查是否满足开始条件
    if (this.status === RoomStatus.WAITING && this.config.autoStart) {
      if (this.userIds.size >= this.config.maxPlayers) {
        this.startGame();
      }
    }

    return true;
  }

  public removeUser(userId: string): boolean {
    const wasPlayer = this.userIds.delete(userId);
    const wasSpectator = this.spectatorIds.delete(userId);
    this.connectionIds.delete(userId);

    // 处理游戏中玩家离开的情况
    if (wasPlayer && this.status === RoomStatus.IN_PROGRESS) {
      // 标记玩家为断线状态
      this.gameEngine.handlePlayerDisconnect(userId);
    }

    // 如果所有玩家都离开，考虑清理房间
    if (this.userIds.size === 0 && this.spectatorIds.size === 0) {
      this.scheduleCleanup();
    }

    return wasPlayer || wasSpectator;
  }

  public reconnectUser(userId: string, connectionId: string): boolean {
    if (!this.userIds.has(userId) && !this.spectatorIds.has(userId)) {
      return false;
    }

    this.connectionIds.set(userId, connectionId);

    // 如果游戏进行中，标记玩家重新连接
    if (this.status === RoomStatus.IN_PROGRESS) {
      this.gameEngine.handlePlayerReconnect(userId);
    }

    // 取消可能已经安排的清理任务
    if (this.cleanupTimeout) {
      clearTimeout(this.cleanupTimeout);
      this.cleanupTimeout = null;
    }

    return true;
  }

  public startGame(): boolean {
    if (this.status !== RoomStatus.WAITING) {
      return false;
    }

    // 确保房间中有足够的玩家
    if (this.userIds.size < 5) {
      // 阿瓦隆最少需要5名玩家
      return false;
    }

    this.status = RoomStatus.IN_PROGRESS;
    this.gameEngine.startGame(Array.from(this.userIds));
    return true;
  }

  public endGame(): boolean {
    if (this.status !== RoomStatus.IN_PROGRESS) {
      return false;
    }

    this.gameEngine.endGame();
    this.status = RoomStatus.FINISHED;
    return true;
  }

  public archiveGame(): boolean {
    if (this.status !== RoomStatus.FINISHED) {
      return false;
    }

    this.status = RoomStatus.ARCHIVED;
    return true;
  }

  public handlePlayerAction(userId: string, action: any): boolean {
    if (this.status !== RoomStatus.IN_PROGRESS) {
      return false;
    }

    // 确认用户在房间中
    if (!this.userIds.has(userId)) {
      return false;
    }

    return this.gameEngine.handleAction(userId, action);
  }

  public getConnectionId(userId: string): string | undefined {
    return this.connectionIds.get(userId);
  }

  public getUserIds(): string[] {
    return Array.from(this.userIds);
  }

  public getSpectatorIds(): string[] {
    return Array.from(this.spectatorIds);
  }

  private scheduleCleanup(): void {
    // 避免重复安排清理
    if (this.cleanupTimeout) {
      clearTimeout(this.cleanupTimeout);
    }

    // 安排房间清理
    this.cleanupTimeout = setTimeout(() => {
      this.emit("cleanup", this.id);
    }, ROOM_CLEANUP_DELAY);
  }

  public cancelCleanup(): void {
    if (this.cleanupTimeout) {
      clearTimeout(this.cleanupTimeout);
      this.cleanupTimeout = null;
    }
  }
}

class RoomManager {
  private rooms: Map<string, GameRoom>;
  private stateStorage: StateStorageService;
  private messageRouter: MessageRouter;
  private stateFilter: StateFilter;

  constructor(
    stateStorage: StateStorageService,
    messageRouter: MessageRouter,
    stateFilter: StateFilter
  ) {
    this.rooms = new Map();
    this.stateStorage = stateStorage;
    this.messageRouter = messageRouter;
    this.stateFilter = stateFilter;
  }

  public createRoom(config: RoomConfig): {
    roomId: string;
    success: boolean;
    reason?: string;
  } {
    const roomId = uuidv4();

    try {
      const room = new GameRoom(roomId, config);

      // 设置房间事件监听
      this.setupRoomEventHandlers(room);

      // 保存房间
      this.rooms.set(roomId, room);

      // 创建初始状态并持久化
      const initialState = room.getState();
      this.stateStorage.saveState(roomId, initialState);

      return { roomId, success: true };
    } catch (error) {
      Logger.error(`Failed to create room: ${error.message}`, error);
      return { roomId, success: false, reason: "Internal server error" };
    }
  }

  private setupRoomEventHandlers(room: GameRoom): void {
    const roomId = room.getId();

    room.on("stateChanged", (changes) => {
      // 保存事件和状态变更
      this.stateStorage.saveEvent({
        roomId,
        changes,
        timestamp: Date.now(),
      });

      // 向房间内的玩家广播状态更新
      this.broadcastStateUpdate(roomId, changes);
    });

    room.on("gamePhaseChanged", (newPhase) => {
      // 广播游戏阶段变化
      this.messageRouter.broadcastToRoom(roomId, {
        header: {
          msgId: uuidv4(),
          msgType: "GAME_PHASE_CHANGED",
          timestamp: Date.now(),
        },
        payload: {
          phase: newPhase,
        },
      });

      // 关键阶段进行状态快照
      if (this.isKeyPhase(newPhase)) {
        const state = room.getState();
        this.stateStorage.saveSnapshot(roomId, state);
      }
    });

    room.on("gameEnded", (result) => {
      // 保存游戏结果
      this.stateStorage.saveGameResult(roomId, result);

      // 广播游戏结束消息
      this.messageRouter.broadcastToRoom(roomId, {
        header: {
          msgId: uuidv4(),
          msgType: "GAME_ENDED",
          timestamp: Date.now(),
        },
        payload: result,
      });

      // 最终状态快照
      const finalState = room.getState();
      this.stateStorage.saveSnapshot(roomId, finalState);
    });

    room.on("cleanup", (roomId) => {
      this.rooms.delete(roomId);
      Logger.info(`Room ${roomId} has been cleaned up due to inactivity`);
    });
  }

  private isKeyPhase(phase: string): boolean {
    // 阿瓦隆关键游戏阶段
    const keyPhases = [
      "TEAM_SELECTION",
      "TEAM_VOTING",
      "QUEST_VOTING",
      "ASSASSIN_GUESS",
    ];
    return keyPhases.includes(phase);
  }

  private broadcastStateUpdate(roomId: string, changes: any[]): void {
    const room = this.rooms.get(roomId);
    if (!room) return;

    const state = room.getState();
    const userIds = [...room.getUserIds(), ...room.getSpectatorIds()];

    // 为每个玩家过滤状态变更并发送
    for (const userId of userIds) {
      const connectionId = room.getConnectionId(userId);
      if (!connectionId) continue;

      // 根据玩家角色过滤状态变更
      const filteredChanges = this.stateFilter.filterChangesForUser(
        changes,
        userId,
        state
      );

      if (filteredChanges.length > 0) {
        this.messageRouter.sendToConnection(connectionId, {
          header: {
            msgId: uuidv4(),
            msgType: "STATE_UPDATE",
            gameId: state.gameId,
            roomId,
            timestamp: Date.now(),
            sequence: state.version,
          },
          payload: {
            stateVersion: state.version,
            baseVersion: state.version - 1,
            updateType: "incremental",
            changes: filteredChanges,
          },
        });
      }
    }
  }

  public getRoom(roomId: string): GameRoom | undefined {
    return this.rooms.get(roomId);
  }

  public removeRoom(roomId: string): boolean {
    const room = this.rooms.get(roomId);
    if (!room) return false;

    // 标记为已归档
    room.archiveGame();
    room.cancelCleanup();

    // 从管理器中移除
    this.rooms.delete(roomId);

    return true;
  }

  public addUserToRoom(
    roomId: string,
    userId: string,
    connectionId: string,
    isSpectator: boolean = false
  ): { success: boolean; reason?: string } {
    const room = this.rooms.get(roomId);
    if (!room) {
      return { success: false, reason: "Room not found" };
    }

    const added = room.addUser(userId, connectionId, isSpectator);
    if (!added) {
      return { success: false, reason: "Room is full" };
    }

    // 通知房间内其他玩家
    this.messageRouter.broadcastToRoom(
      roomId,
      {
        header: {
          msgId: uuidv4(),
          msgType: "USER_JOINED",
          timestamp: Date.now(),
        },
        payload: {
          userId,
          isSpectator,
        },
      },
      (conn) => conn.getUserId() !== userId
    ); // 不发送给刚加入的玩家

    return { success: true };
  }

  public removeUserFromRoom(roomId: string, userId: string): boolean {
    const room = this.rooms.get(roomId);
    if (!room) return false;

    const removed = room.removeUser(userId);
    if (removed) {
      // 通知房间内其他玩家
      this.messageRouter.broadcastToRoom(roomId, {
        header: {
          msgId: uuidv4(),
          msgType: "USER_LEFT",
          timestamp: Date.now(),
        },
        payload: {
          userId,
        },
      });
    }

    return removed;
  }

  public reconnectUserToRoom(
    roomId: string,
    userId: string,
    connectionId: string
  ): boolean {
    const room = this.rooms.get(roomId);
    if (!room) return false;

    const reconnected = room.reconnectUser(userId, connectionId);
    if (reconnected) {
      // 通知房间内其他玩家
      this.messageRouter.broadcastToRoom(
        roomId,
        {
          header: {
            msgId: uuidv4(),
            msgType: "USER_RECONNECTED",
            timestamp: Date.now(),
          },
          payload: {
            userId,
          },
        },
        (conn) => conn.getUserId() !== userId
      ); // 不发送给重连的玩家
    }

    return reconnected;
  }

  public notifyUserDisconnected(roomId: string, userId: string): void {
    // 通知房间内其他玩家
    this.messageRouter.broadcastToRoom(
      roomId,
      {
        header: {
          msgId: uuidv4(),
          msgType: "USER_DISCONNECTED",
          timestamp: Date.now(),
        },
        payload: {
          userId,
        },
      },
      (conn) => conn.getUserId() !== userId
    ); // 不发送给断开连接的玩家
  }

  public handlePlayerAction(
    roomId: string,
    userId: string,
    action: any
  ): boolean {
    const room = this.rooms.get(roomId);
    if (!room) return false;

    return room.handlePlayerAction(userId, action);
  }

  public getGameState(roomId: string): GameState | null {
    const room = this.rooms.get(roomId);
    if (!room) return null;

    return room.getState();
  }

  public async getCurrentStateVersion(roomId: string): Promise<number> {
    const room = this.rooms.get(roomId);
    if (room) {
      return room.getState().version;
    }

    // 如果房间不在内存中，尝试从存储中获取
    try {
      const state = await this.stateStorage.loadLatestState(roomId);
      return state ? state.version : 0;
    } catch (error) {
      Logger.error(
        `Failed to get current state version for room ${roomId}`,
        error
      );
      return 0;
    }
  }

  public async getNearestSnapshotId(
    roomId: string,
    targetVersion: number
  ): Promise<string> {
    try {
      return await this.stateStorage.findNearestSnapshotId(
        roomId,
        targetVersion
      );
    } catch (error) {
      Logger.error(`Failed to get nearest snapshot for room ${roomId}`, error);
      throw error;
    }
  }

  public async getStateSnapshot(snapshotId: string): Promise<GameState | null> {
    try {
      return await this.stateStorage.loadSnapshot(snapshotId);
    } catch (error) {
      Logger.error(`Failed to load snapshot ${snapshotId}`, error);
      return null;
    }
  }

  public async getEventsSince(
    roomId: string,
    fromVersion: number
  ): Promise<any[]> {
    try {
      return await this.stateStorage.getEventsSince(roomId, fromVersion);
    } catch (error) {
      Logger.error(
        `Failed to get events since version ${fromVersion} for room ${roomId}`,
        error
      );
      return [];
    }
  }

  public getRoomCount(): number {
    return this.rooms.size;
  }

  public getActiveRoomCount(): number {
    let count = 0;
    for (const room of this.rooms.values()) {
      if (room.getStatus() === RoomStatus.IN_PROGRESS) {
        count++;
      }
    }
    return count;
  }
}

// 常量定义
const ROOM_CLEANUP_DELAY = 3600000; // 1小时

export { RoomManager, GameRoom, RoomStatus, RoomConfig };
```

#### 4.1.4 安全性与防护措施

- **连接认证与授权**：

  - 多因素认证：结合 JWT、游戏房间码和玩家凭证
  - 细粒度权限控制：基于玩家角色的操作权限
  - 令牌刷新机制：长会话的令牌自动刷新
  - 异常行为检测：识别可疑的认证请求模式

- **通信安全保障**：

  - 传输加密：TLS 1.3 加密所有通信内容
  - 消息完整性：消息摘要和签名验证
  - 防重放攻击：nonce 值和时间戳结合验证
  - 敏感数据处理：关键游戏信息的特殊加密

- **DoS 攻击防护**：

  - 连接限速：单 IP 连接建立速率限制
  - 消息限流：单连接消息发送频率控制
  - 异常连接监测：识别和阻断异常连接模式
  - 优雅降级：高负载时的服务保护机制

- **异常处理与审计**：

  - 安全日志：记录所有安全相关事件
  - 异常报警：严重安全事件的实时告警
  - 操作审计：敏感操作的完整审计跟踪
  - 取证支持：保留关键事件的详细上下文信息

- **安全措施实现示例**：

```typescript
// 安全措施实现示例

import * as jwt from "jsonwebtoken";
import * as crypto from "crypto";
import { Request, Response, NextFunction } from "express";
import { RateLimiterMemory } from "rate-limiter-flexible";
import { Logger } from "./logger";

// 配置常量
const JWT_SECRET = process.env.JWT_SECRET || "your-secret-key";
const JWT_EXPIRY = "1h";
const REFRESH_TOKEN_EXPIRY = "7d";
const MAX_CONNECTIONS_PER_IP = 10;
const MAX_CONNECTION_RATE = 5; // 每5秒最多5个连接请求
const MAX_MESSAGE_RATE = 30; // 每秒最多30条消息

// 速率限制器
const ipLimiter = new RateLimiterMemory({
  points: MAX_CONNECTION_RATE,
  duration: 5,
});

const messageLimiter = new RateLimiterMemory({
  points: MAX_MESSAGE_RATE,
  duration: 1,
});

// 保存使用过的nonce，防止重放攻击
const usedNonces = new Map<string, number>();

// 定期清理过期nonce
setInterval(() => {
  const now = Date.now();
  for (const [nonce, timestamp] of usedNonces.entries()) {
    if (now - timestamp > 3600000) {
      // 1小时后过期
      usedNonces.delete(nonce);
    }
  }
}, 300000); // 每5分钟清理一次

class SecurityService {
  /**
   * 生成JWT令牌
   */
  public static generateToken(
    userId: string,
    role: string,
    roomId?: string
  ): string {
    const payload = {
      userId,
      role,
      roomId,
      iat: Math.floor(Date.now() / 1000),
    };

    return jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRY });
  }

  /**
   * 验证JWT令牌
   */
  public static verifyToken(token: string): any {
    try {
      return jwt.verify(token, JWT_SECRET);
    } catch (error) {
      Logger.warn(`Token verification failed: ${error.message}`);
      return null;
    }
  }

  /**
   * 生成刷新令牌
   */
  public static generateRefreshToken(userId: string): string {
    const payload = {
      userId,
      type: "refresh",
      iat: Math.floor(Date.now() / 1000),
    };

    return jwt.sign(payload, JWT_SECRET, { expiresIn: REFRESH_TOKEN_EXPIRY });
  }

  /**
   * 续签JWT令牌
   */
  public static refreshAccessToken(refreshToken: string): string | null {
    try {
      const decoded = jwt.verify(refreshToken, JWT_SECRET) as any;

      if (decoded.type !== "refresh") {
        return null;
      }

      // 从数据库获取用户信息
      // const user = await userService.getUser(decoded.userId);

      // 生成新的访问令牌
      return this.generateToken(decoded.userId, "player"); // 角色应从用户信息中获取
    } catch (error) {
      Logger.warn(`Token refresh failed: ${error.message}`);
      return null;
    }
  }

  /**
   * IP速率限制中间件
   */
  public static ipRateLimiter(
    req: Request,
    res: Response,
    next: NextFunction
  ): void {
    const ip = req.ip;

    ipLimiter
      .consume(ip)
      .then(() => {
        next();
      })
      .catch(() => {
        Logger.warn(`Rate limit exceeded for IP: ${ip}`);
        res.status(429).send("Too Many Requests");
      });
  }

  /**
   * 连接数量限制中间件
   */
  public static connectionLimiter(
    req: Request,
    res: Response,
    next: NextFunction
  ): void {
    const ip = req.ip;

    // 检查IP的连接数量
    const connectionCount = getConnectionCountByIp(ip); // 需要实现此函数

    if (connectionCount >= MAX_CONNECTIONS_PER_IP) {
      Logger.warn(`Max connections reached for IP: ${ip}`);
      res.status(429).send("Too Many Connections");
      return;
    }

    next();
  }

  /**
   * 验证消息完整性和防重放
   */
  public static validateMessage(message: any): boolean {
    try {
      // 检查必要字段
      if (
        !message.header ||
        !message.header.msgId ||
        !message.header.timestamp ||
        !message.meta ||
        !message.meta.nonce
      ) {
        return false;
      }

      // 检查时间戳是否在合理范围内
      const msgTime = message.header.timestamp;
      const now = Date.now();
      if (Math.abs(now - msgTime) > 300000) {
        // 允许5分钟的时间偏差
        Logger.warn(`Message timestamp out of range: ${message.header.msgId}`);
        return false;
      }

      // 检查nonce是否被使用过
      const nonce = message.meta.nonce;
      if (usedNonces.has(nonce)) {
        Logger.warn(`Nonce already used: ${nonce}`);
        return false;
      }

      // 记录使用过的nonce
      usedNonces.set(nonce, now);

      // 验证消息签名（如果有）
      if (message.meta.signature) {
        return this.verifySignature(message);
      }

      return true;
    } catch (error) {
      Logger.error(`Message validation error: ${error.message}`, error);
      return false;
    }
  }

  /**
   * 验证消息签名
   */
  private static verifySignature(message: any): boolean {
    try {
      const signature = message.meta.signature;
      message.meta.signature = undefined; // 计算签名时需要移除签名字段

      const data = JSON.stringify(message);

      // 使用公钥验证签名
      // const publicKey = getPublicKey(message.header.senderId); // 需要实现此函数
      // const verify = crypto.createVerify('SHA256');
      // verify.update(data);
      // const isValid = verify.verify(publicKey, Buffer.from(signature, 'base64'));

      // 重新添加签名字段
      message.meta.signature = signature;

      // return isValid;
      return true; // 简化实现
    } catch (error) {
      Logger.error(`Signature verification error: ${error.message}`, error);
      return false;
    }
  }

  /**
   * 生成消息签名
   */
  public static signMessage(message: any, privateKey: string): string {
    try {
      message.meta.signature = undefined; // 计算签名时需要移除签名字段

      const data = JSON.stringify(message);

      const sign = crypto.createSign("SHA256");
      sign.update(data);
      const signature = sign.sign(privateKey, "base64");

      return signature;
    } catch (error) {
      Logger.error(`Message signing error: ${error.message}`, error);
      return "";
    }
  }

  /**
   * 敏感数据加密
   */
  public static encryptSensitiveData(data: any, key: Buffer): string {
    try {
      const iv = crypto.randomBytes(16);
      const cipher = crypto.createCipheriv("aes-256-gcm", key, iv);

      let encrypted = cipher.update(JSON.stringify(data), "utf8", "base64");
      encrypted += cipher.final("base64");

      const authTag = cipher.getAuthTag();

      // 返回格式: base64(iv):base64(authTag):base64(encryptedData)
      return `${iv.toString("base64")}:${authTag.toString(
        "base64"
      )}:${encrypted}`;
    } catch (error) {
      Logger.error(`Data encryption error: ${error.message}`, error);
      return "";
    }
  }

  /**
   * 解密敏感数据
   */
  public static decryptSensitiveData(encryptedData: string, key: Buffer): any {
    try {
      const parts = encryptedData.split(":");
      if (parts.length !== 3) {
        throw new Error("Invalid encrypted data format");
      }

      const iv = Buffer.from(parts[0], "base64");
      const authTag = Buffer.from(parts[1], "base64");
      const ciphertext = parts[2];

      const decipher = crypto.createDecipheriv("aes-256-gcm", key, iv);
      decipher.setAuthTag(authTag);

      let decrypted = decipher.update(ciphertext, "base64", "utf8");
      decrypted += decipher.final("utf8");

      return JSON.parse(decrypted);
    } catch (error) {
      Logger.error(`Data decryption error: ${error.message}`, error);
      return null;
    }
  }

  /**
   * 消息速率限制
   */
  public static async messageRateLimiter(
    connectionId: string
  ): Promise<boolean> {
    try {
      await messageLimiter.consume(connectionId);
      return true;
    } catch (error) {
      Logger.warn(
        `Message rate limit exceeded for connection: ${connectionId}`
      );
      return false;
    }
  }

  /**
   * 检测异常行为
   */
  public static detectAnomalies(connection: any, message: any): boolean {
    // 异常检测逻辑，如：
    // 1. 连接建立后立即发送大量消息
    // 2. 短时间内发送大量无效消息
    // 3. 频繁尝试访问无权限资源
    // 4. 尝试操作其他玩家的游戏状态

    // 简化实现
    return true;
  }

  /**
   * 记录安全日志
   */
  public static logSecurityEvent(eventType: string, details: any): void {
    Logger.info(`SECURITY_EVENT: ${eventType}`, {
      ...details,
      timestamp: new Date().toISOString(),
    });

    // 对于严重事件，触发告警
    if (isHighSeverityEvent(eventType)) {
      // alertService.sendAlert(eventType, details);
    }
  }
}

// 辅助函数
function getConnectionCountByIp(ip: string): number {
  // 实现获取指定IP的当前连接数
  return 0; // 简化实现
}

function isHighSeverityEvent(eventType: string): boolean {
  const highSeverityEvents = [
    "AUTHENTICATION_BRUTE_FORCE",
    "MULTIPLE_FAILED_LOGINS",
    "UNAUTHORIZED_ACCESS_ATTEMPT",
    "POSSIBLE_REPLAY_ATTACK",
    "ABNORMAL_MESSAGE_PATTERN",
  ];

  return highSeverityEvents.includes(eventType);
}

export { SecurityService };
```

### 4.2 状态同步消息处理

- **消息路由器实现示例**：

```typescript
// 消息路由器实现示例

import { v4 as uuidv4 } from "uuid";
import { WebSocketConnection } from "./websocket-server";
import { Logger } from "./logger";

interface MessageHandler {
  handle(message: any, connection: WebSocketConnection): Promise<void>;
}

class MessageRouter {
  private handlers: Map<string, MessageHandler>;
  private connectionManager: any; // 用于访问所有连接的管理器

  constructor(connectionManager: any) {
    this.handlers = new Map();
    this.connectionManager = connectionManager;
  }

  public registerHandler(messageType: string, handler: MessageHandler): void {
    this.handlers.set(messageType, handler);
    Logger.info(`Registered handler for message type: ${messageType}`);
  }

  public async route(
    message: any,
    connection: WebSocketConnection
  ): Promise<void> {
    try {
      const messageType = message.header?.msgType;

      if (!messageType) {
        connection.sendError("Message type not specified");
        return;
      }

      const handler = this.handlers.get(messageType);
      if (!handler) {
        connection.sendError(
          `No handler registered for message type: ${messageType}`
        );
        return;
      }

      // 记录消息路由
      Logger.debug(
        `Routing message ${message.header?.msgId} of type ${messageType}`
      );

      // 处理消息
      await handler.handle(message, connection);
    } catch (error) {
      Logger.error(`Error routing message: ${error.message}`, error);
      connection.sendError("Internal error processing message");
    }
  }

  public broadcastToRoom(
    roomId: string,
    message: any,
    filter?: (conn: WebSocketConnection) => boolean
  ): void {
    try {
      // 使用连接管理器广播到房间
      this.connectionManager.broadcast(message, (conn: WebSocketConnection) => {
        return conn.getRoomId() === roomId && (!filter || filter(conn));
      });
    } catch (error) {
      Logger.error(
        `Error broadcasting to room ${roomId}: ${error.message}`,
        error
      );
    }
  }

  public sendToUser(userId: string, message: any): boolean {
    try {
      // 查找用户的连接
      for (const connection of this.connectionManager.getConnections()) {
        if (connection.isAlive() && connection.getUserId() === userId) {
          connection.send(message);
          return true;
        }
      }
      Logger.warn(`User ${userId} not found or not connected`);
      return false;
    } catch (error) {
      Logger.error(`Error sending to user ${userId}: ${error.message}`, error);
      return false;
    }
  }

  public sendToConnection(connectionId: string, message: any): boolean {
    try {
      const connection = this.connectionManager.getConnection(connectionId);
      if (connection && connection.isAlive()) {
        return connection.send(message);
      }
      Logger.warn(`Connection ${connectionId} not found or not active`);
      return false;
    } catch (error) {
      Logger.error(
        `Error sending to connection ${connectionId}: ${error.message}`,
        error
      );
      return false;
    }
  }

  public sendAcknowledgement(
    originalMessage: any,
    connection: WebSocketConnection,
    status: string = "SUCCESS",
    data: any = null
  ): void {
    const ackMessage = {
      header: {
        msgId: uuidv4(),
        msgType: "ACK",
        refMsgId: originalMessage.header?.msgId,
        timestamp: Date.now(),
      },
      payload: {
        status,
        data,
      },
      meta: {
        compression: "none",
        encryption: "none",
        priority: 9,
      },
    };

    connection.send(ackMessage);
  }

  // 创建一组典型的消息处理器
  public setupDefaultHandlers(): void {
    // 心跳处理器
    this.registerHandler("HEARTBEAT", {
      handle: async (message, connection) => {
        this.sendAcknowledgement(message, connection);
      },
    });

    // 状态请求处理器
    this.registerHandler("STATE_REQUEST", {
      handle: async (message, connection) => {
        const roomId = connection.getRoomId();
        if (!roomId) {
          this.sendAcknowledgement(message, connection, "ERROR", {
            reason: "Not in a room",
          });
          return;
        }

        try {
          // 这里需要注入RoomManager或GameStateManager的实例
          // const state = await roomManager.getGameState(roomId);
          // const filteredState = stateFilter.filterForUser(state, connection.getUserId());

          // TODO: 获取并发送状态

          this.sendAcknowledgement(message, connection);
        } catch (error) {
          Logger.error(
            `Error processing state request: ${error.message}`,
            error
          );
          this.sendAcknowledgement(message, connection, "ERROR", {
            reason: "Failed to retrieve state",
          });
        }
      },
    });

    // 玩家操作处理器
    this.registerHandler("PLAYER_ACTION", {
      handle: async (message, connection) => {
        const roomId = connection.getRoomId();
        const userId = connection.getUserId();

        if (!roomId || !userId) {
          this.sendAcknowledgement(message, connection, "ERROR", {
            reason: "Not in a room or not authenticated",
          });
          return;
        }

        try {
          // 这里需要注入RoomManager的实例
          // const result = await roomManager.handlePlayerAction(roomId, userId, message.payload);

          // TODO: 处理玩家操作

          this.sendAcknowledgement(message, connection);
        } catch (error) {
          Logger.error(
            `Error processing player action: ${error.message}`,
            error
          );
          this.sendAcknowledgement(message, connection, "ERROR", {
            reason: "Failed to process action",
          });
        }
      },
    });

    // TODO: 注册更多消息处理器
  }
}

export { MessageRouter, MessageHandler };
```

#### 4.2.1 状态同步消息类型

- **完整状态同步 (FULL_STATE)**：

  - 目的：提供游戏完整状态快照
  - 触发时机：玩家加入、重连、检查点同步
  - 处理优先级：高
  - 传输策略：可选压缩、角色敏感信息过滤

- **增量状态更新 (STATE_UPDATE)**：

  - 目的：高效传输状态变化
  - 触发时机：游戏状态变化后立即发送
  - 处理优先级：中高
  - 传输策略：批量更新、变更记录跟踪

- **游戏事件 (GAME_EVENT)**：

  - 目的：通知游戏中的重要事件发生
  - 触发时机：特定游戏阶段转换、关键行为执行
  - 处理优先级：中
  - 传输策略：即时推送、事件 ID 关联

- **玩家操作 (PLAYER_ACTION)**：

  - 目的：传递玩家意图给服务器
  - 触发时机：玩家执行游戏操作
  - 处理优先级：高
  - 传输策略：操作验证、幂等性支持

- **状态同步请求 (SYNC_REQUEST)**：
  - 目的：主动请求状态同步
  - 触发时机：客户端检测到潜在不一致
  - 处理优先级：中
  - 传输策略：请求可包含当前版本号

#### 4.2.2 状态更新处理流程

- **状态处理器实现示例**：

```typescript
// 状态处理器实现示例

import { v4 as uuidv4 } from "uuid";
import { WebSocketConnection } from "./websocket-server";
import { Logger } from "./logger";
import { GameState } from "./game-state";
import { DiffCalculator } from "./state-diff";

class StateProcessor {
  private gameStateCache: Map<string, GameState>;
  private stateVersions: Map<string, number>;
  private diffCalculator: DiffCalculator;

  constructor() {
    this.gameStateCache = new Map();
    this.stateVersions = new Map();
    this.diffCalculator = new DiffCalculator();
  }

  /**
   * 处理完整状态消息
   */
  public async handleFullState(
    roomId: string,
    stateData: any
  ): Promise<GameState> {
    try {
      const gameState = new GameState(stateData);

      // 存储状态到缓存
      this.gameStateCache.set(roomId, gameState);

      // 更新状态版本
      const version = stateData.header?.version || 1;
      this.stateVersions.set(roomId, version);

      Logger.info(`Full state updated for room ${roomId}, version: ${version}`);

      return gameState;
    } catch (error) {
      Logger.error(
        `Error processing full state for room ${roomId}: ${error.message}`,
        error
      );
      throw error;
    }
  }

  /**
   * 处理增量状态更新
   */
  public async handleStateUpdate(
    roomId: string,
    updateData: any
  ): Promise<GameState | null> {
    try {
      // 获取当前状态
      const currentState = this.gameStateCache.get(roomId);
      if (!currentState) {
        Logger.warn(
          `Cannot apply state update for room ${roomId}: no base state found`
        );
        return null;
      }

      // 检查更新基础版本是否匹配
      const currentVersion = this.stateVersions.get(roomId) || 1;
      const updateBaseVersion = updateData.payload?.baseVersion;

      if (updateBaseVersion !== currentVersion) {
        Logger.warn(
          `State update version mismatch for room ${roomId}. Current: ${currentVersion}, Update base: ${updateBaseVersion}`
        );
        return currentState; // 返回当前状态，客户端需要重新同步
      }

      // 应用状态更新
      const newState = currentState.clone();
      newState.applyChanges(updateData.payload.changes);

      // 更新缓存和版本
      this.gameStateCache.set(roomId, newState);
      this.stateVersions.set(roomId, updateData.payload.stateVersion);

      Logger.debug(
        `State updated for room ${roomId}, new version: ${updateData.payload.stateVersion}`
      );

      return newState;
    } catch (error) {
      Logger.error(
        `Error processing state update for room ${roomId}: ${error.message}`,
        error
      );
      return this.gameStateCache.get(roomId); // 出错时返回当前状态
    }
  }

  /**
   * 生成完整状态消息
   */
  public generateFullStateMessage(roomId: string, recipientId?: string): any {
    const state = this.gameStateCache.get(roomId);
    if (!state) {
      Logger.warn(
        `Cannot generate full state message for room ${roomId}: no state found`
      );
      return null;
    }

    const version = this.stateVersions.get(roomId) || 1;

    // 生成完整状态消息
    return {
      header: {
        msgId: uuidv4(),
        msgType: "FULL_STATE",
        gameId: "avalon",
        roomId: roomId,
        timestamp: Date.now(),
        version: version,
      },
      payload: {
        state: recipientId
          ? state.filterForPlayer(recipientId)
          : state.getPublicState(),
        stateVersion: version,
      },
      meta: {
        compression: state.getSize() > 10240 ? "gzip" : "none",
        encryption: "none",
        priority: 8,
      },
    };
  }

  /**
   * 生成增量状态更新消息
   */
  public generateStateUpdateMessage(
    roomId: string,
    prevState: GameState,
    newState: GameState
  ): any {
    try {
      const currentVersion = this.stateVersions.get(roomId) || 1;
      const newVersion = currentVersion + 1;

      // 计算状态差异
      const diff = this.diffCalculator.calculateDiff(prevState, newState);

      // 生成状态更新消息
      const updateMessage = {
        header: {
          msgId: uuidv4(),
          msgType: "STATE_UPDATE",
          gameId: "avalon",
          roomId: roomId,
          timestamp: Date.now(),
          version: newVersion,
        },
        payload: {
          baseVersion: currentVersion,
          stateVersion: newVersion,
          updateType: "incremental",
          changes: diff,
        },
        meta: {
          compression: "none",
          encryption: "none",
          priority: 7,
        },
      };

      // 更新状态版本
      this.stateVersions.set(roomId, newVersion);

      return updateMessage;
    } catch (error) {
      Logger.error(
        `Error generating state update for room ${roomId}: ${error.message}`,
        error
      );
      return null;
    }
  }

  /**
   * 检查客户端状态是否与服务器状态一致
   */
  public checkStateConsistency(
    roomId: string,
    clientStateVersion: number
  ): boolean {
    const serverVersion = this.stateVersions.get(roomId) || 1;
    return clientStateVersion === serverVersion;
  }

  /**
   * 处理玩家对游戏状态的操作
   */
  public async processPlayerAction(
    roomId: string,
    playerId: string,
    action: any
  ): Promise<{ success: boolean; newState?: GameState; errors?: string[] }> {
    try {
      // 获取当前状态
      const currentState = this.gameStateCache.get(roomId);
      if (!currentState) {
        return { success: false, errors: ["Game state not found"] };
      }

      // 校验操作合法性
      if (!this.validateAction(action, currentState, playerId)) {
        return {
          success: false,
          errors: ["Invalid action for current game state"],
        };
      }

      // 克隆当前状态并应用操作
      const newState = currentState.clone();
      const actionResult = newState.applyAction(action, playerId);

      if (!actionResult.success) {
        return { success: false, errors: actionResult.errors };
      }

      // 更新缓存的状态
      this.gameStateCache.set(roomId, newState);

      // 增加状态版本
      const currentVersion = this.stateVersions.get(roomId) || 1;
      this.stateVersions.set(roomId, currentVersion + 1);

      return { success: true, newState };
    } catch (error) {
      Logger.error(
        `Error processing player action for room ${roomId}: ${error.message}`,
        error
      );
      return { success: false, errors: ["Internal server error"] };
    }
  }

  /**
   * 验证玩家操作的合法性
   */
  private validateAction(
    action: any,
    state: GameState,
    playerId: string
  ): boolean {
    // 基本验证
    if (!action || !action.type) {
      return false;
    }

    // 检查玩家是否在游戏中
    if (!state.hasPlayer(playerId)) {
      return false;
    }

    // 检查是否当前玩家的回合
    if (!state.isPlayerTurn(playerId) && action.requiresTurn !== false) {
      return false;
    }

    // 检查操作在当前游戏阶段是否允许
    if (!state.isActionAllowedInPhase(action.type)) {
      return false;
    }

    // 其他游戏逻辑特定的验证...

    return true; // 默认通过验证
  }

  /**
   * 获取当前游戏状态
   */
  public getGameState(roomId: string): GameState | null {
    return this.gameStateCache.get(roomId) || null;
  }

  /**
   * 获取当前状态版本
   */
  public getStateVersion(roomId: string): number {
    return this.stateVersions.get(roomId) || 1;
  }

  /**
   * 创建状态检查点
   */
  public createStateCheckpoint(
    roomId: string
  ): { state: GameState; version: number } | null {
    const state = this.gameStateCache.get(roomId);
    const version = this.stateVersions.get(roomId) || 1;

    if (!state) {
      return null;
    }

    // 这里可以持久化状态快照到数据库

    return { state, version };
  }
}

export { StateProcessor };
```

### 4.3 冲突检测与解决机制

#### 4.3.1 冲突类型与检测方案

- **操作冲突检测**：

```typescript
// 冲突检测与解决机制实现示例

import { GameState } from "./game-state";
import { Logger } from "./logger";

enum ConflictType {
  NONE = "NONE",
  VERSION_MISMATCH = "VERSION_MISMATCH",
  CONCURRENT_MODIFICATION = "CONCURRENT_MODIFICATION",
  INVALID_OPERATION = "INVALID_OPERATION",
  STATE_DIVERGENCE = "STATE_DIVERGENCE",
  TEMPORAL_VIOLATION = "TEMPORAL_VIOLATION",
}

interface Conflict {
  type: ConflictType;
  description: string;
  severity: "LOW" | "MEDIUM" | "HIGH";
  affectedArea: string[];
  affectedPlayers: string[];
  timestamp: number;
  originalState?: any;
  conflictingState?: any;
  resolution?: string;
}

class ConflictDetector {
  /**
   * 检测版本不匹配冲突
   */
  public detectVersionMismatch(
    expectedVersion: number,
    actualVersion: number
  ): Conflict | null {
    if (expectedVersion !== actualVersion) {
      return {
        type: ConflictType.VERSION_MISMATCH,
        description: `Expected state version ${expectedVersion}, but found ${actualVersion}`,
        severity: "MEDIUM",
        affectedArea: ["gameState"],
        affectedPlayers: [], // 未知受影响玩家
        timestamp: Date.now(),
      };
    }
    return null;
  }

  /**
   * 检测状态差异冲突
   */
  public detectStateDivergence(
    serverState: GameState,
    clientState: any,
    playerId: string
  ): Conflict | null {
    // 专注于检测关键状态差异，而不是完全比较
    const criticalDifferences = this.detectCriticalStateDifferences(
      serverState,
      clientState
    );

    if (criticalDifferences.length > 0) {
      return {
        type: ConflictType.STATE_DIVERGENCE,
        description: `Client state diverges from server state in critical areas: ${criticalDifferences.join(
          ", "
        )}`,
        severity: "HIGH",
        affectedArea: criticalDifferences,
        affectedPlayers: [playerId],
        timestamp: Date.now(),
        originalState: serverState.getPublicState(),
        conflictingState: clientState,
      };
    }

    return null;
  }

  /**
   * 检测时间顺序冲突
   */
  public detectTemporalViolation(
    action: any,
    gameState: GameState
  ): Conflict | null {
    // 检查操作的时间戳是否合理
    const actionTime = action.timestamp;
    const serverTime = Date.now();

    // 检查操作是否"来自未来"（客户端时钟可能快于服务器）
    if (actionTime > serverTime + 5000) {
      // 允许5秒的时钟差异
      return {
        type: ConflictType.TEMPORAL_VIOLATION,
        description: `Action timestamp ${actionTime} is too far in the future`,
        severity: "MEDIUM",
        affectedArea: ["gameFlow"],
        affectedPlayers: [action.playerId],
        timestamp: serverTime,
      };
    }

    // 检查操作是否过时
    if (actionTime < serverTime - 30000) {
      // 30秒前的操作视为过时
      return {
        type: ConflictType.TEMPORAL_VIOLATION,
        description: `Action timestamp ${actionTime} is too old`,
        severity: "LOW",
        affectedArea: ["gameFlow"],
        affectedPlayers: [action.playerId],
        timestamp: serverTime,
      };
    }

    return null;
  }

  /**
   * 检测并发修改冲突
   */
  public detectConcurrentModification(
    action1: any,
    action2: any
  ): Conflict | null {
    // 如果两个操作的目标相同，但操作来源不同，且时间接近，视为并发修改
    if (
      action1.target === action2.target &&
      action1.playerId !== action2.playerId &&
      Math.abs(action1.timestamp - action2.timestamp) < 1000
    ) {
      return {
        type: ConflictType.CONCURRENT_MODIFICATION,
        description: `Concurrent modification of ${action1.target} by players ${action1.playerId} and ${action2.playerId}`,
        severity: "HIGH",
        affectedArea: [action1.target],
        affectedPlayers: [action1.playerId, action2.playerId],
        timestamp: Date.now(),
      };
    }

    return null;
  }

  /**
   * 检测关键状态差异
   */
  private detectCriticalStateDifferences(
    serverState: GameState,
    clientState: any
  ): string[] {
    const differences: string[] = [];

    // 检查游戏阶段
    if (serverState.getPhase() !== clientState.phase) {
      differences.push("gamePhase");
    }

    // 检查当前玩家回合
    if (serverState.getCurrentTurn() !== clientState.currentTurn) {
      differences.push("currentTurn");
    }

    // 检查游戏关键状态
    // 例如任务成功/失败数量、投票状态等
    if (
      serverState.getMissionSuccessCount() !== clientState.missionSuccessCount
    ) {
      differences.push("missionSuccessCount");
    }

    if (serverState.getMissionFailCount() !== clientState.missionFailCount) {
      differences.push("missionFailCount");
    }

    // 检查玩家状态
    const serverPlayers = serverState.getPlayers();
    const clientPlayers = clientState.players || {};

    for (const [playerId, serverPlayerData] of Object.entries(serverPlayers)) {
      const clientPlayerData = clientPlayers[playerId];

      if (!clientPlayerData) {
        differences.push(`player_${playerId}_missing`);
        continue;
      }

      // 检查玩家角色
      if (serverPlayerData.role !== clientPlayerData.role) {
        differences.push(`player_${playerId}_role`);
      }

      // 检查玩家状态
      if (serverPlayerData.status !== clientPlayerData.status) {
        differences.push(`player_${playerId}_status`);
      }
    }

    return differences;
  }
}

class ConflictResolver {
  /**
   * 解决版本不匹配冲突
   */
  public resolveVersionMismatch(
    conflict: Conflict,
    currentState: GameState
  ): {
    resolution: string;
    resolvedState?: GameState;
  } {
    // 版本不匹配通常通过发送完整状态来解决
    return {
      resolution: "SEND_FULL_STATE",
      resolvedState: currentState,
    };
  }

  /**
   * 解决状态差异冲突
   */
  public resolveStateDivergence(
    conflict: Conflict,
    serverState: GameState
  ): {
    resolution: string;
    resolvedState?: GameState;
    clientAction?: string;
  } {
    // 记录严重的状态差异
    if (conflict.severity === "HIGH") {
      Logger.warn(`Severe state divergence detected: ${conflict.description}`, {
        affectedArea: conflict.affectedArea,
        affectedPlayers: conflict.affectedPlayers,
      });
    }

    // 对于大多数情况，服务器状态作为权威
    return {
      resolution: "SERVER_AUTHORITY",
      resolvedState: serverState,
      clientAction: "FORCE_SYNC",
    };
  }

  /**
   * 解决时间顺序冲突
   */
  public resolveTemporalViolation(
    conflict: Conflict,
    action: any,
    gameState: GameState
  ): {
    resolution: string;
    accepted: boolean;
    adjustedAction?: any;
  } {
    if (conflict.type !== ConflictType.TEMPORAL_VIOLATION) {
      throw new Error("Wrong conflict type passed to resolver");
    }

    // 对于"来自未来"的操作，调整其时间戳为服务器当前时间
    if (conflict.description.includes("in the future")) {
      const adjustedAction = { ...action, timestamp: Date.now() };

      return {
        resolution: "ADJUST_TIMESTAMP",
        accepted: true,
        adjustedAction,
      };
    }

    // 对于过时操作，根据其对游戏状态的潜在影响决定是拒绝还是接受
    if (conflict.description.includes("too old")) {
      const isHarmless = this.isActionHarmlessWhenLate(action, gameState);

      if (isHarmless) {
        return {
          resolution: "ACCEPT_LATE_HARMLESS",
          accepted: true,
        };
      } else {
        return {
          resolution: "REJECT_TOO_OLD",
          accepted: false,
        };
      }
    }

    // 默认拒绝其他时间违规
    return {
      resolution: "REJECT_TEMPORAL_VIOLATION",
      accepted: false,
    };
  }

  /**
   * 解决并发修改冲突
   */
  public resolveConcurrentModification(
    conflict: Conflict,
    action1: any,
    action2: any,
    gameState: GameState
  ): {
    resolution: string;
    acceptedAction: any | null;
    rejectedAction: any | null;
  } {
    if (conflict.type !== ConflictType.CONCURRENT_MODIFICATION) {
      throw new Error("Wrong conflict type passed to resolver");
    }

    // 记录冲突
    Logger.info(
      `Resolving concurrent modification conflict: ${conflict.description}`
    );

    // 根据时间戳优先原则解决冲突
    if (action1.timestamp < action2.timestamp) {
      return {
        resolution: "TIME_PRIORITY",
        acceptedAction: action1,
        rejectedAction: action2,
      };
    } else {
      return {
        resolution: "TIME_PRIORITY",
        acceptedAction: action2,
        rejectedAction: action1,
      };
    }

    // 也可以实现其他解决策略，例如：
    // - 基于角色权重
    // - 基于操作类型优先级
    // - 合并两个操作的效果（如果可能）
  }

  /**
   * 判断一个延迟操作是否无害
   */
  private isActionHarmlessWhenLate(action: any, gameState: GameState): boolean {
    // 分析操作类型和游戏当前状态，判断延迟操作是否会产生不良后果
    const harmlessLateActions = [
      "CHAT_MESSAGE",
      "PLAYER_READY",
      "REQUEST_GAME_STATE",
      "HEARTBEAT",
    ];

    if (harmlessLateActions.includes(action.type)) {
      return true;
    }

    // 其他情况需要具体分析
    return false;
  }
}

export { ConflictDetector, ConflictResolver, ConflictType, Conflict };
```

#### 4.3.2 冲突解决策略

- **分布式日志重放机制**：

  - 在解决严重冲突时，可利用操作日志重放功能
  - 从最近的一致检查点开始，按顺序重放操作
  - 支持选择性跳过冲突操作的能力
  - 重放时保持严格的时间顺序

- **冲突解决优先级**：

  - 时间优先：较早的操作优先级更高
  - 角色权重：特定角色（如主持人）具有更高决策权
  - 操作类型：游戏关键操作优先于非关键操作
  - 保守原则：在无法解决的冲突中维持当前状态

- **用户感知最小化**：

  - 自动冲突解决应尽量透明化
  - 仅当需要玩家参与决策时进行提示
  - 使用动画和过渡效果掩盖状态纠正
  - 提供简洁友好的冲突解释

- **异常冲突处理策略**：

  - 设置冲突解决超时机制
  - 对于无法自动解决的冲突，提供手动干预接口
  - 支持将房间状态回滚到最近稳定点
  - 建立冲突修复后的一致性验证机制

- **冲突日志与分析**：
  - 记录所有检测到的冲突及解决方案
  - 识别频繁出现的冲突模式
  - 统计冲突发生率和解决成功率
  - 为系统优化提供数据支持

### 4.4 多设备状态一致性

#### 4.4.1 跨设备状态表示

- **设备差异适配**：

  - 统一状态模型：核心游戏状态在不同设备间保持一致的数据结构
  - 设备特定渲染：基于统一状态，各设备按自身能力和特点渲染 UI
  - 响应式布局：状态展示会根据屏幕尺寸和方向自动适配
  - 输入方式归一化：触摸、点击、键盘等不同输入归一为标准游戏操作

- **状态表达最小化**：

  - 传输体积优化：只传输必要的游戏状态数据
  - 渲染分离：将状态数据与渲染表达严格分离
  - 增量更新机制：精确传输变化的部分，减少冗余数据
  - 视图状态本地化：非关键 UI 状态（如动画、提示等）由客户端本地管理

- **多设备一致性**：
  - 设备资源校准：检测并适应设备的渲染能力和响应速度
  - 同步关键节点：在游戏关键时刻强制同步所有设备
  - 差异化表现策略：低性能设备可简化非核心视觉效果
  - 多端数据验证：定期对比不同设备显示的游戏状态是否一致

#### 4.4.2 多设备同步管理器

```typescript
// 多设备同步管理器实现示例

import { Logger } from "./logger";
import { GameState } from "./game-state";
import { ClientDevice, DeviceCapability } from "./device-models";

class MultiDeviceSyncManager {
  private roomDevices: Map<string, Map<string, ClientDevice>>;
  private deviceStates: Map<string, { version: number; lastSync: number }>;
  private syncCheckpoints: Set<string>; // 游戏阶段强制同步检查点

  constructor() {
    this.roomDevices = new Map();
    this.deviceStates = new Map();
    this.syncCheckpoints = new Set([
      "GAME_START",
      "TEAM_PROPOSAL",
      "VOTE_PHASE",
      "MISSION_EXECUTION",
      "GAME_END",
    ]);
  }

  /**
   * 注册设备连接
   */
  public registerDevice(
    roomId: string,
    playerId: string,
    deviceInfo: any
  ): string {
    const deviceId = `${playerId}_${deviceInfo.type}_${Date.now()}`;

    // 创建设备记录
    const device = {
      id: deviceId,
      playerId,
      type: deviceInfo.type,
      capabilities: this.determineCapabilities(deviceInfo),
      connectionTime: Date.now(),
      lastActivity: Date.now(),
      screenSize: deviceInfo.screenSize || { width: 0, height: 0 },
    };

    // 获取或创建房间设备集合
    let roomDevices = this.roomDevices.get(roomId);
    if (!roomDevices) {
      roomDevices = new Map();
      this.roomDevices.set(roomId, roomDevices);
    }

    // 添加设备到房间
    roomDevices.set(deviceId, device);

    // 初始化设备状态记录
    this.deviceStates.set(deviceId, { version: 0, lastSync: Date.now() });

    Logger.info(
      `Device registered: ${deviceId} for player ${playerId} in room ${roomId}`
    );

    return deviceId;
  }

  /**
   * 确定设备能力
   */
  private determineCapabilities(deviceInfo: any): DeviceCapability[] {
    const capabilities: DeviceCapability[] = [];

    // 基本能力检测
    if (deviceInfo.type === "MOBILE") {
      capabilities.push(DeviceCapability.TOUCH);
    } else {
      capabilities.push(DeviceCapability.MOUSE);
    }

    // 屏幕尺寸能力
    if (deviceInfo.screenSize) {
      if (
        deviceInfo.screenSize.width >= 1024 &&
        deviceInfo.screenSize.height >= 768
      ) {
        capabilities.push(DeviceCapability.HIGH_RESOLUTION);
      }
    }

    // 设备性能能力
    if (deviceInfo.performance === "HIGH") {
      capabilities.push(DeviceCapability.ANIMATION);
      capabilities.push(DeviceCapability.ADVANCED_EFFECTS);
    }

    // 网络能力
    if (deviceInfo.network === "WIFI" || deviceInfo.network === "ETHERNET") {
      capabilities.push(DeviceCapability.HIGH_BANDWIDTH);
    }

    return capabilities;
  }

  /**
   * 更新设备状态
   */
  public updateDeviceState(deviceId: string, stateVersion: number): void {
    const deviceState = this.deviceStates.get(deviceId);
    if (deviceState) {
      deviceState.version = stateVersion;
      deviceState.lastSync = Date.now();
    }
  }

  /**
   * 检查玩家多设备状态一致性
   */
  public checkPlayerDeviceConsistency(
    roomId: string,
    playerId: string
  ): {
    consistent: boolean;
    devices: { deviceId: string; version: number; lastSync: number }[];
  } {
    const result = {
      consistent: true,
      devices: [],
    };

    const roomDevices = this.roomDevices.get(roomId);
    if (!roomDevices) {
      return result;
    }

    let baseVersion: number | null = null;

    // 收集该玩家的所有设备状态
    for (const [deviceId, device] of roomDevices.entries()) {
      if (device.playerId === playerId) {
        const deviceState = this.deviceStates.get(deviceId);
        if (deviceState) {
          result.devices.push({
            deviceId,
            version: deviceState.version,
            lastSync: deviceState.lastSync,
          });

          // 设置基准版本
          if (baseVersion === null) {
            baseVersion = deviceState.version;
          }
          // 检查一致性
          else if (baseVersion !== deviceState.version) {
            result.consistent = false;
          }
        }
      }
    }

    return result;
  }

  /**
   * 检查是否需要强制同步
   */
  public shouldForceSynchronization(gamePhase: string): boolean {
    return this.syncCheckpoints.has(gamePhase);
  }

  /**
   * 识别需要同步的设备
   */
  public identifyOutOfSyncDevices(
    roomId: string,
    currentVersion: number,
    maxLagMs: number = 5000
  ): string[] {
    const outOfSyncDevices: string[] = [];
    const now = Date.now();

    const roomDevices = this.roomDevices.get(roomId);
    if (!roomDevices) {
      return outOfSyncDevices;
    }

    for (const [deviceId, device] of roomDevices.entries()) {
      const deviceState = this.deviceStates.get(deviceId);
      if (deviceState) {
        // 检查版本是否匹配
        if (deviceState.version !== currentVersion) {
          outOfSyncDevices.push(deviceId);
        }
        // 检查最后同步时间是否过长
        else if (now - deviceState.lastSync > maxLagMs) {
          outOfSyncDevices.push(deviceId);
        }
      }
    }

    return outOfSyncDevices;
  }

  /**
   * 生成设备特定的状态
   */
  public generateDeviceSpecificState(
    baseState: GameState,
    deviceId: string
  ): any {
    const deviceInfo = this.getDeviceById(deviceId);
    if (!deviceInfo) {
      return baseState.getPublicState();
    }

    // 克隆基本状态
    const deviceState = baseState.clone();

    // 根据设备能力过滤或增强状态
    if (!deviceInfo.capabilities.includes(DeviceCapability.HIGH_RESOLUTION)) {
      // 简化图形资源引用
      deviceState.setLowResolutionMode(true);
    }

    if (!deviceInfo.capabilities.includes(DeviceCapability.ANIMATION)) {
      // 禁用动画相关状态
      deviceState.setAnimationEnabled(false);
    }

    // 根据设备类型调整布局相关状态
    if (deviceInfo.type === "MOBILE") {
      deviceState.setLayoutMode("MOBILE");
    } else if (deviceInfo.type === "TABLET") {
      deviceState.setLayoutMode("TABLET");
    } else {
      deviceState.setLayoutMode("DESKTOP");
    }

    return deviceState.filterForPlayer(deviceInfo.playerId);
  }

  /**
   * 获取设备信息
   */
  private getDeviceById(deviceId: string): ClientDevice | null {
    for (const roomDevices of this.roomDevices.values()) {
      const device = roomDevices.get(deviceId);
      if (device) {
        return device;
      }
    }
    return null;
  }

  /**
   * 移除设备
   */
  public removeDevice(deviceId: string): boolean {
    for (const [roomId, roomDevices] of this.roomDevices.entries()) {
      if (roomDevices.has(deviceId)) {
        roomDevices.delete(deviceId);
        this.deviceStates.delete(deviceId);

        // 如果房间没有设备了，清理房间记录
        if (roomDevices.size === 0) {
          this.roomDevices.delete(roomId);
        }

        Logger.info(`Device removed: ${deviceId} from room ${roomId}`);
        return true;
      }
    }

    return false;
  }

  /**
   * 获取房间中的设备数量
   */
  public getDeviceCount(roomId: string): number {
    const roomDevices = this.roomDevices.get(roomId);
    return roomDevices ? roomDevices.size : 0;
  }

  /**
   * 更新设备活动时间
   */
  public updateDeviceActivity(deviceId: string): void {
    const device = this.getDeviceById(deviceId);
    if (device) {
      device.lastActivity = Date.now();
    }
  }

  /**
   * 识别不活跃设备
   */
  public identifyInactiveDevices(
    inactiveThresholdMs: number = 60000
  ): string[] {
    const inactiveDevices: string[] = [];
    const now = Date.now();

    for (const roomDevices of this.roomDevices.values()) {
      for (const [deviceId, device] of roomDevices.entries()) {
        if (now - device.lastActivity > inactiveThresholdMs) {
          inactiveDevices.push(deviceId);
        }
      }
    }

    return inactiveDevices;
  }
}

enum DeviceType {
  DESKTOP = "DESKTOP",
  MOBILE = "MOBILE",
  TABLET = "TABLET",
}

export { MultiDeviceSyncManager, DeviceType };
```

#### 4.4.3 多设备一致性保障策略

- **设备同步周期**：

  - 低延迟设备：50ms 同步周期，实时性要求高
  - 标准设备：100ms 同步周期，平衡性能和流畅度
  - 网络受限设备：200ms 同步周期，减少带宽占用
  - 关键时刻强制同步：游戏节点转换时所有设备强制同步

- **版本控制与冲突处理**：

  - 单一玩家多设备版本统一：确保同一玩家的多个设备状态一致
  - 设备切换无缝衔接：在不同设备间切换时保持状态连续性
  - 设备间冲突优先级：主设备操作优先于辅助设备
  - 状态校验与恢复：定期验证多设备状态一致性，不一致时进行恢复

- **资源适配策略**：

  - 渐进式资源加载：根据设备性能调整资源加载优先级
  - 动态图形质量：根据设备性能和网络条件调整图形细节
  - 输入方式适配：根据设备类型提供最适合的操作方式
  - 布局自适应：根据屏幕尺寸和方向调整 UI 布局

- **网络条件适应**：
  - 弱网络模式：在网络条件差时减少同步频率和数据量
  - 预测性缓存：预先缓存可能的游戏状态减少等待时间
  - 批量更新策略：在低带宽条件下合并多个小更新
  - 优先级队列：在网络拥塞时按重要性传输游戏状态

## 5. 测试计划

### 5.1 单元测试

- **WebSocket 服务测试**：

  - 测试连接建立与断开的正确处理
  - 验证消息解析和路由机制
  - 测试心跳机制和连接保活功能
  - 模拟异常情况下的服务行为

- **状态同步组件测试**：

  - 验证增量状态更新的正确应用
  - 测试不同类型消息的处理逻辑
  - 验证版本控制和检查点机制
  - 测试状态缓存与检索功能

- **冲突检测与解决测试**：

  - 模拟各类型冲突场景
  - 验证冲突检测的准确性
  - 测试不同冲突解决策略的效果
  - 验证冲突解决后的状态一致性

- **多设备同步测试**：
  - 测试设备注册和状态跟踪
  - 验证不同设备类型的状态表示
  - 测试设备间的状态一致性检查
  - 验证不同网络条件下的同步表现

### 5.2 集成测试

- **完整通信流程测试**：

  - 测试客户端-服务器完整通信流程
  - 验证多客户端之间的状态同步
  - 测试游戏事件传播和处理
  - 验证在复杂交互场景下的系统行为

- **状态一致性验证**：

  - 测试长时间运行下的状态一致性
  - 验证高频状态变化下的同步准确性
  - 测试网络波动环境下的状态恢复
  - 验证断线重连后的状态恢复机制

- **异常处理与恢复**：
  - 测试不同类型的网络故障恢复
  - 验证服务器节点故障时的行为
  - 测试客户端崩溃后的状态恢复
  - 验证非法操作处理机制

### 5.3 性能测试

- **负载测试**：

  - 测试不同并发连接数下的服务性能
  - 验证单房间最大支持玩家数的服务质量
  - 测试高频消息环境下的系统吞吐量
  - 验证大型状态对象传输时的性能表现

- **延迟测试**：

  - 测量端到端消息传递延迟
  - 验证不同网络条件下的同步延迟
  - 测试状态变化响应时间
  - 分析不同同步策略对延迟的影响

- **资源使用测试**：

  - 测量服务器 CPU 和内存使用情况
  - 验证客户端资源消耗水平
  - 测试长时间运行时的资源稳定性
  - 分析网络带宽使用情况

- **扩展性测试**：
  - 测试水平扩展服务的性能增益
  - 验证多服务节点协同工作能力
  - 测试动态扩容时的服务连续性
  - 分析不同规模下的系统瓶颈

### 5.4 兼容性测试

- **客户端兼容性测试**：

  - 测试不同微信版本的兼容性
  - 验证各种移动设备型号的适配性
  - 测试不同操作系统版本下的表现
  - 分析不同硬件配置下的性能差异

- **网络环境测试**：

  - 测试不同网络类型下的同步质量
  - 验证网络波动环境下的稳定性
  - 测试跨地域通信的延迟影响
  - 分析弱网络条件下的用户体验

- **版本兼容性测试**：
  - 测试客户端版本差异的处理机制
  - 验证协议升级的向后兼容性
  - 测试状态格式演进的支持情况
  - 分析版本混合环境下的系统行为

## 6. 交付标准

### 6.1 代码质量要求

- **可读性与一致性**：

  - 遵循 TypeScript/JavaScript 代码规范
  - 一致的命名约定和代码风格
  - 清晰的模块和文件组织
  - 适当的注释和文档字符串

- **健壮性与异常处理**：

  - 全面的错误处理策略
  - 优雅的降级机制
  - 边缘情况的考虑和处理
  - 防御性编程实践

- **代码审查与测试覆盖**：
  - 核心模块的单元测试覆盖率 ≥80%
  - 通过所有集成测试场景
  - 完成代码审查流程
  - 无严重 Bug 和安全漏洞

### 6.2 性能与可靠性标准

- **性能指标**：

  - 服务器支持每房间并发连接：10 个以上
  - 消息端到端传输延迟：<100ms (良好网络)
  - 客户端每秒消息处理量：≥50 条
  - 断线重连恢复时间：<3 秒

- **资源使用限制**：

  - 客户端内存占用：<50MB
  - 单房间网络带宽：<100KB/s
  - 手机电池消耗：中等（不明显快于普通应用）
  - 服务端每连接资源消耗可控

- **稳定性指标**：
  - 服务连续运行时间：≥7 天无重启
  - 消息丢失率：<0.1%
  - 状态冲突解决成功率：≥99%
  - 异常状态恢复成功率：≥95%

### 6.3 文档与交付物清单

- **设计文档**：

  - 系统架构图及说明
  - 通信协议规范
  - 状态同步机制详细设计
  - 安全与性能设计文档

- **源代码与构建物**：

  - WebSocket 服务器代码
  - 状态同步核心组件
  - 冲突检测与解决模块
  - 多设备同步支持库

- **测试资料**：

  - 单元测试套件
  - 集成测试场景
  - 性能测试报告
  - 兼容性测试结果

- **部署与运维文档**：
  - 部署指南与配置说明
  - 监控指标与告警设置
  - 常见问题排查手册
  - 性能调优建议

## 7. 工作量估算

### 7.1 开发阶段工作量

| 阶段     | 工作项               | 工作天数  | 人员配置       |
| -------- | -------------------- | --------- | -------------- |
| 准备阶段 | 技术选型与环境搭建   | 2 天      | 1 名高级工程师 |
| 基础架构 | WebSocket 服务器框架 | 3 天      | 1 名高级工程师 |
| 基础架构 | 状态同步基础协议     | 2 天      | 1 名高级工程师 |
| 核心功能 | 状态更新处理器       | 4 天      | 1 名高级工程师 |
| 核心功能 | 冲突检测与解决       | 5 天      | 1 名高级工程师 |
| 核心功能 | 多设备同步机制       | 3 天      | 1 名工程师     |
| 安全机制 | 认证与加密逻辑       | 2 天      | 1 名工程师     |
| 性能优化 | 数据压缩与传输优化   | 2 天      | 1 名工程师     |
| 日志监控 | 监控与告警机制       | 2 天      | 1 名工程师     |
| **小计** |                      | **25 天** |                |

### 7.2 测试阶段工作量

| 阶段       | 工作项               | 工作天数  | 人员配置                   |
| ---------- | -------------------- | --------- | -------------------------- |
| 单元测试   | 编写自动化测试用例   | 4 天      | 1 名测试工程师             |
| 集成测试   | 搭建测试环境与场景   | 2 天      | 1 名测试工程师             |
| 集成测试   | 执行测试与问题修复   | 5 天      | 1 名测试工程师、1 名工程师 |
| 性能测试   | 性能测试用例与执行   | 3 天      | 1 名性能测试专家           |
| 兼容性测试 | 多设备兼容性验证     | 3 天      | 1 名测试工程师             |
| 压力测试   | 极限条件下稳定性测试 | 2 天      | 1 名性能测试专家           |
| **小计**   |                      | **19 天** |                            |

### 7.3 部署与优化阶段

| 阶段     | 工作项           | 工作天数 | 人员配置           |
| -------- | ---------------- | -------- | ------------------ |
| 部署准备 | 部署文档与脚本   | 2 天     | 1 名 DevOps 工程师 |
| 部署执行 | 生产环境部署     | 1 天     | 1 名 DevOps 工程师 |
| 性能调优 | 生产环境性能优化 | 3 天     | 1 名高级工程师     |
| 监控配置 | 监控系统配置     | 1 天     | 1 名 DevOps 工程师 |
| **小计** |                  | **7 天** |                    |

### 7.4 总工作量

| 阶段       | 工作天数                   | 人员需求                                     |
| ---------- | -------------------------- | -------------------------------------------- |
| 开发阶段   | 25 天                      | 1 名高级工程师，1 名工程师                   |
| 测试阶段   | 19 天                      | 1 名测试工程师，1 名工程师，1 名性能测试专家 |
| 部署与优化 | 7 天                       | 1 名 DevOps 工程师，1 名高级工程师           |
| **总计**   | **51 个工作日** (约 10 周) | 4-5 人团队                                   |

### 7.5 风险与缓冲

- 技术风险缓冲：+5 个工作日
- 需求变更缓冲：+3 个工作日
- 团队协作缓冲：+2 个工作日

**最终工作量预估**：61 个工作日 (约 12 周)

## 8. 总结

本文档详细定义了阿瓦隆微信小游戏的游戏状态同步机制实现方案，包括功能需求、技术规格、实现细节和测试计划等关键内容。该方案基于 WebSocket 技术构建实时通信架构，采用增量同步和冲突解决机制，确保多端游戏状态的一致性。

### 8.1 主要特点

- **高效的状态同步协议**：基于增量更新和完整状态检查点的混合同步策略，在保证数据一致性的同时最小化传输量
- **健壮的冲突检测与解决**：能够识别和处理多种类型的状态冲突，自动或手动方式解决不一致问题
- **灵活的多设备支持**：针对不同设备类型和能力提供适应性的状态表示和同步策略
- **可扩展的服务架构**：支持服务水平扩展，满足不同规模的游戏房间和用户并发需求
- **完善的安全与防护**：包含认证、加密、防重放等多层安全防护措施

### 8.2 技术挑战与解决方案

| 挑战               | 解决方案                                             |
| ------------------ | ---------------------------------------------------- |
| 实时性与一致性平衡 | 基于增量更新的状态同步，配合关键节点的强制同步检查点 |
| 网络不稳定环境     | 断线重连机制、状态恢复策略和弱网络适应模式           |
| 多设备状态表示     | 核心状态与渲染分离，设备适应性的 UI 表达             |
| 高并发连接处理     | 非阻塞 IO、连接池和消息队列优化                      |
| 冲突检测与解决     | 多层次冲突类型识别和优先级策略解决方案               |

### 8.3 未来扩展方向

- **机器学习预测**：引入 ML 模型预测可能的游戏状态变化，进一步降低感知延迟
- **P2P 辅助通信**：探索点对点通信辅助服务器同步，减轻中心服务器负担
- **自适应同步策略**：基于实时网络和设备状况动态调整同步参数和策略
- **多区域部署**：支持不同地理位置的服务节点协同工作，降低特定区域的访问延迟
- **混合存储策略**：结合内存数据库和持久化存储优化状态管理效率

## 9. 依赖关系

### 9.1 前置依赖

| 依赖项                      | 说明                                   | 依赖类型 |
| --------------------------- | -------------------------------------- | -------- |
| Task6.2.1：游戏数据模型设计 | 提供基础的游戏状态数据结构和序列化规范 | 强依赖   |
| Task6.2.2：数据持久化方案   | 提供状态存储和恢复所需的数据库接口     | 中度依赖 |
| Task6.1.3：用户身份认证服务 | 提供 WebSocket 连接的用户认证机制      | 强依赖   |
| Task4.3.2：游戏逻辑引擎     | 提供游戏规则验证和状态转换逻辑         | 强依赖   |

### 9.2 后续依赖

| 依赖项                  | 说明                                 | 依赖类型 |
| ----------------------- | ------------------------------------ | -------- |
| Task6.2.4：游戏回放功能 | 依赖本任务提供的状态记录和变更历史   | 直接依赖 |
| Task6.2.5：数据分析接口 | 使用本任务收集的游戏状态数据进行分析 | 间接依赖 |
| Task7.1.2：多人游戏大厅 | 依赖本任务的房间状态同步机制         | 直接依赖 |
| Task3.4.1：游戏 UI 组件 | 需适配本任务提供的状态数据格式       | 间接依赖 |

### 9.3 协作需求

- **前端团队**：需要与前端团队紧密协作，确保客户端正确实现状态同步协议
- **运维团队**：需要运维团队参与制定部署策略和监控方案
- **QA 团队**：需要测试团队设计针对性的测试用例验证同步机制
- **产品团队**：需要产品经理确认同步策略对用户体验的影响

### 9.4 风险与缓解

| 风险                     | 可能性 | 影响 | 缓解措施                         |
| ------------------------ | ------ | ---- | -------------------------------- |
| WebSocket 服务稳定性问题 | 中     | 高   | 完善的监控告警和自动恢复机制     |
| 移动网络环境不稳定       | 高     | 中   | 增强弱网络适应能力和断线重连机制 |
| 游戏状态复杂度超预期     | 中     | 高   | 模块化设计，分级同步策略         |
| 开发进度延误             | 中     | 中   | 预留缓冲时间，明确优先级         |
| 性能瓶颈                 | 低     | 高   | 早期性能测试，预留优化阶段       |
