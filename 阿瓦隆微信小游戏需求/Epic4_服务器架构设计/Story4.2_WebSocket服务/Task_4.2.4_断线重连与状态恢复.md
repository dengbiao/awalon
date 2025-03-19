# Task 4.2.4: 断线重连与状态恢复

## 描述

设计并实现阿瓦隆微信小游戏的断线重连和游戏状态恢复机制，确保玩家在网络波动或短暂断开连接后能够无缝重新加入游戏，恢复游戏状态，提升用户体验和游戏的可靠性。

## 验收标准

1. 实现连接状态跟踪和断线检测机制
2. 设计并实现客户端重连协议和服务端处理流程
3. 完成游戏会话保持和状态缓存机制
4. 支持玩家重连后的游戏状态快速恢复
5. 实现断线玩家的临时 AI 代替（如适用）
6. 处理长时间离线的特殊策略
7. 设计重连失败的补偿机制
8. 完成重连过程的错误处理和日志记录
9. 编写断线重连功能的单元测试和集成测试

## 详细内容

### 连接状态管理

1. **连接状态跟踪**

   ```typescript
   @Injectable()
   export class ConnectionTracker {
     // 玩家连接状态映射
     private playerConnections = new Map<
       string,
       {
         socketId: string;
         roomId: string;
         lastActive: number;
         status: ConnectionStatus;
       }
     >();

     constructor(
       private readonly redisService: RedisService,
       private readonly logger: LoggerService
     ) {
       // 定期检查超时连接
       setInterval(() => this.checkTimeoutConnections(), 30000);
     }

     // 记录新连接
     async trackConnection(
       userId: string,
       socketId: string,
       roomId?: string
     ): Promise<void> {
       const connectionInfo = {
         socketId,
         roomId,
         lastActive: Date.now(),
         status: ConnectionStatus.CONNECTED,
       };

       this.playerConnections.set(userId, connectionInfo);

       // 同时存储到Redis以支持集群环境
       await this.redisService.hset(`connections:${userId}`, connectionInfo);

       this.logger.debug(`用户 ${userId} 连接成功，SocketID: ${socketId}`);
     }

     // 更新连接活跃时间
     async updateActivity(userId: string): Promise<void> {
       const connection = this.playerConnections.get(userId);
       if (connection) {
         connection.lastActive = Date.now();
         this.playerConnections.set(userId, connection);

         // 更新Redis
         await this.redisService.hset(
           `connections:${userId}`,
           "lastActive",
           connection.lastActive
         );
       }
     }

     // 标记连接断开
     async markDisconnected(userId: string, reason?: string): Promise<void> {
       const connection = this.playerConnections.get(userId);
       if (connection) {
         connection.status = ConnectionStatus.DISCONNECTED;
         this.playerConnections.set(userId, connection);

         // 更新Redis
         await this.redisService.hset(
           `connections:${userId}`,
           "status",
           ConnectionStatus.DISCONNECTED
         );

         this.logger.debug(
           `用户 ${userId} 断开连接，原因: ${reason || "未知"}`
         );
       }
     }

     // 检查玩家是否连接
     async isPlayerConnected(userId: string): Promise<boolean> {
       const connection = this.playerConnections.get(userId);

       if (connection && connection.status === ConnectionStatus.CONNECTED) {
         return true;
       }

       // 检查Redis（支持集群）
       const redisConnection = await this.redisService.hgetall(
         `connections:${userId}`
       );
       return (
         redisConnection &&
         redisConnection.status === ConnectionStatus.CONNECTED
       );
     }

     // 检查超时连接
     private async checkTimeoutConnections(): Promise<void> {
       const now = Date.now();
       const timeout = 60000; // 60秒无活动视为超时

       for (const [userId, connection] of this.playerConnections.entries()) {
         if (
           connection.status === ConnectionStatus.CONNECTED &&
           now - connection.lastActive > timeout
         ) {
           // 标记为断开连接
           await this.markDisconnected(userId, "连接超时");
         }
       }
     }
   }

   export enum ConnectionStatus {
     CONNECTED = "connected",
     DISCONNECTED = "disconnected",
     RECONNECTING = "reconnecting",
   }
   ```

### 断线重连机制

1. **会话保持服务**

   ```typescript
   @Injectable()
   export class SessionManager {
     // 会话超时时间（毫秒）
     private readonly sessionTtl = 300000; // 5分钟

     constructor(
       private readonly redisService: RedisService,
       private readonly logger: LoggerService
     ) {}

     // 创建会话
     async createSession(
       userId: string,
       roomId: string,
       gameId?: string
     ): Promise<string> {
       const sessionId = uuid();
       const session = {
         userId,
         roomId,
         gameId,
         createdAt: Date.now(),
         expiresAt: Date.now() + this.sessionTtl,
       };

       // 存储会话信息
       await this.redisService.set(
         `session:${sessionId}`,
         JSON.stringify(session),
         this.sessionTtl / 1000
       );

       // 关联用户和会话
       await this.redisService.set(
         `user_session:${userId}`,
         sessionId,
         this.sessionTtl / 1000
       );

       return sessionId;
     }

     // 获取会话
     async getSession(sessionId: string): Promise<any> {
       const sessionData = await this.redisService.get(`session:${sessionId}`);
       if (!sessionData) {
         return null;
       }

       try {
         return JSON.parse(sessionData);
       } catch (error) {
         this.logger.error(`解析会话数据失败: ${error.message}`);
         return null;
       }
     }

     // 获取用户会话
     async getUserSession(userId: string): Promise<any> {
       const sessionId = await this.redisService.get(`user_session:${userId}`);
       if (!sessionId) {
         return null;
       }

       return this.getSession(sessionId);
     }

     // 延长会话时间
     async extendSession(sessionId: string): Promise<boolean> {
       const session = await this.getSession(sessionId);
       if (!session) {
         return false;
       }

       session.expiresAt = Date.now() + this.sessionTtl;

       await this.redisService.set(
         `session:${sessionId}`,
         JSON.stringify(session),
         this.sessionTtl / 1000
       );

       await this.redisService.expire(
         `user_session:${session.userId}`,
         this.sessionTtl / 1000
       );

       return true;
     }

     // 销毁会话
     async destroySession(sessionId: string): Promise<void> {
       const session = await this.getSession(sessionId);
       if (session) {
         await this.redisService.del(`session:${sessionId}`);
         await this.redisService.del(`user_session:${session.userId}`);
       }
     }
   }
   ```

2. **重连处理器**

   ```typescript
   @Injectable()
   export class ReconnectionService {
     constructor(
       private readonly sessionManager: SessionManager,
       private readonly gameStateService: GameStateService,
       private readonly connectionTracker: ConnectionTracker,
       private readonly messageService: MessageService,
       private readonly socketGateway: GameSocketGateway,
       private readonly logger: LoggerService
     ) {}

     // 处理重连请求
     async handleReconnection(
       socketId: string,
       userId: string,
       sessionId: string
     ): Promise<boolean> {
       try {
         // 验证会话有效性
         const session = await this.sessionManager.getSession(sessionId);
         if (!session || session.userId !== userId) {
           this.logger.warn(`用户 ${userId} 重连失败: 会话无效`);
           return false;
         }

         // 更新连接跟踪
         await this.connectionTracker.trackConnection(
           userId,
           socketId,
           session.roomId
         );

         // 加入socket房间
         this.socketGateway.joinRoom(socketId, session.roomId);

         // 恢复游戏状态
         await this.restoreGameState(socketId, userId, session);

         // 通知房间内其他玩家
         await this.notifyPlayerReconnected(userId, session.roomId);

         // 延长会话
         await this.sessionManager.extendSession(sessionId);

         this.logger.debug(`用户 ${userId} 重连成功`);
         return true;
       } catch (error) {
         this.logger.error(`处理重连请求失败: ${error.message}`);
         return false;
       }
     }

     // 恢复游戏状态
     private async restoreGameState(
       socketId: string,
       userId: string,
       session: any
     ): Promise<void> {
       if (!session.gameId) {
         // 不在游戏中，只需恢复房间状态
         return;
       }

       // 获取个性化游戏状态
       const gameState = await this.gameStateService.getPersonalizedGameState(
         session.gameId,
         userId
       );

       if (!gameState) {
         this.logger.warn(
           `无法恢复游戏状态，游戏可能已结束: ${session.gameId}`
         );
         return;
       }

       // 发送游戏状态恢复消息
       const socket = this.socketGateway.getSocket(socketId);
       if (socket) {
         socket.emit(MessageType.GAME_STATE, {
           id: uuid(),
           type: MessageType.GAME_STATE,
           timestamp: Date.now(),
           payload: {
             type: "restore",
             state: gameState,
           },
         });
       }
     }

     // 通知玩家重连
     private async notifyPlayerReconnected(
       userId: string,
       roomId: string
     ): Promise<void> {
       const message = {
         id: uuid(),
         type: MessageType.PLAYER_RECONNECTED,
         timestamp: Date.now(),
         payload: { userId },
       };

       await this.messageService.sendToRoom(roomId, message);
     }

     // 创建重连令牌
     async createReconnectionToken(userId: string): Promise<string> {
       const token = uuid();

       // 存储令牌，有效期5分钟
       await this.redisService.set(`reconnect_token:${token}`, userId, 300);

       return token;
     }

     // 验证重连令牌
     async validateReconnectionToken(token: string): Promise<string | null> {
       const userId = await this.redisService.get(`reconnect_token:${token}`);
       if (userId) {
         // 使用后立即删除令牌
         await this.redisService.del(`reconnect_token:${token}`);
       }
       return userId;
     }
   }
   ```

### 状态恢复机制

1. **状态快照与恢复**

   ```typescript
   @Injectable()
   export class StateSnapshotService {
     constructor(
       private readonly redisService: RedisService,
       private readonly logger: LoggerService
     ) {}

     // 保存游戏状态快照
     async saveStateSnapshot(gameId: string, state: GameState): Promise<void> {
       try {
         // 添加版本标记和时间戳
         const snapshot = {
           ...state,
           snapshotTime: Date.now(),
           snapshotVersion: state.version,
         };

         // 保存到Redis，使用版本号作为标识
         await this.redisService.set(
           `game_snapshot:${gameId}:${state.version}`,
           JSON.stringify(snapshot),
           3600 // 保存1小时
         );

         // 保存最新快照引用
         await this.redisService.set(
           `game_snapshot:${gameId}:latest`,
           state.version.toString(),
           7200 // 保存2小时
         );

         this.logger.debug(
           `游戏 ${gameId} 状态快照已保存，版本: ${state.version}`
         );
       } catch (error) {
         this.logger.error(`保存状态快照失败: ${error.message}`);
       }
     }

     // 获取最新快照
     async getLatestSnapshot(gameId: string): Promise<GameState | null> {
       try {
         // 获取最新版本号
         const latestVersion = await this.redisService.get(
           `game_snapshot:${gameId}:latest`
         );
         if (!latestVersion) {
           return null;
         }

         // 获取对应版本的快照
         const snapshotData = await this.redisService.get(
           `game_snapshot:${gameId}:${latestVersion}`
         );

         if (!snapshotData) {
           return null;
         }

         return JSON.parse(snapshotData);
       } catch (error) {
         this.logger.error(`获取状态快照失败: ${error.message}`);
         return null;
       }
     }

     // 获取指定版本快照
     async getSnapshotByVersion(
       gameId: string,
       version: number
     ): Promise<GameState | null> {
       try {
         const snapshotData = await this.redisService.get(
           `game_snapshot:${gameId}:${version}`
         );

         if (!snapshotData) {
           return null;
         }

         return JSON.parse(snapshotData);
       } catch (error) {
         this.logger.error(`获取状态快照失败: ${error.message}`);
         return null;
       }
     }

     // 清理旧快照
     async cleanupOldSnapshots(
       gameId: string,
       keepVersions: number[] = []
     ): Promise<void> {
       try {
         // 获取所有快照键
         const keys = await this.redisService.keys(`game_snapshot:${gameId}:*`);

         // 过滤掉需要保留的版本和latest标记
         const keysToDelete = keys.filter((key) => {
           const versionMatch = key.match(/game_snapshot:.*:(\d+)$/);
           if (!versionMatch) return false;

           const version = parseInt(versionMatch[1], 10);
           return (
             !keepVersions.includes(version) &&
             key !== `game_snapshot:${gameId}:latest`
           );
         });

         if (keysToDelete.length > 0) {
           await this.redisService.del(...keysToDelete);
           this.logger.debug(`已清理 ${keysToDelete.length} 个旧快照`);
         }
       } catch (error) {
         this.logger.error(`清理旧快照失败: ${error.message}`);
       }
     }
   }
   ```

### 断线玩家处理

1. **离线玩家管理**

   ```typescript
   @Injectable()
   export class OfflinePlayerHandler {
     // 允许离线的最大时间（毫秒）
     private readonly maxOfflineTime = 180000; // 3分钟

     constructor(
       private readonly gameStateService: GameStateService,
       private readonly roomStateService: RoomStateService,
       private readonly messageService: MessageService,
       private readonly logger: LoggerService
     ) {}

     // 处理玩家断线
     async handlePlayerDisconnection(
       userId: string,
       roomId: string,
       gameId?: string
     ): Promise<void> {
       try {
         // 更新房间中玩家状态为离线
         const roomState = await this.roomStateService.getRoomState(roomId);
         if (roomState) {
           const player = roomState.players.find((p) => p.userId === userId);
           if (player) {
             player.isOnline = false;
             await this.roomStateService.updateRoomState(roomId, {
               players: roomState.players,
             });
           }
         }

         // 如果在游戏中，设置离线定时器
         if (gameId) {
           // 设置定时器处理长时间离线
           setTimeout(async () => {
             await this.handleLongOffline(userId, roomId, gameId);
           }, this.maxOfflineTime);

           // 通知其他玩家
           const message = {
             id: uuid(),
             type: MessageType.PLAYER_DISCONNECTED,
             timestamp: Date.now(),
             payload: {
               userId,
               reason: "connection_lost",
               canReconnect: true,
               timeout: this.maxOfflineTime,
             },
           };

           await this.messageService.sendToRoom(roomId, message);
         }
       } catch (error) {
         this.logger.error(`处理玩家断线失败: ${error.message}`);
       }
     }

     // 处理长时间离线
     private async handleLongOffline(
       userId: string,
       roomId: string,
       gameId: string
     ): Promise<void> {
       try {
         // 检查玩家是否已重连
         const roomState = await this.roomStateService.getRoomState(roomId);
         if (!roomState) return;

         const player = roomState.players.find((p) => p.userId === userId);
         if (!player || player.isOnline) {
           // 已重连或不在房间中
           return;
         }

         // 判断游戏状态
         const gameState = await this.gameStateService.getGameState(gameId);
         if (!gameState || gameState.status === GameStatus.COMPLETED) {
           // 游戏已结束
           return;
         }

         // 根据游戏配置决定处理方式
         if (this.shouldAutoPlayForOfflinePlayer(gameState)) {
           // 启用AI代替
           await this.enableAutoPlayForPlayer(userId, gameId);
         } else {
           // 终止游戏
           await this.terminateGameDueToOffline(userId, roomId, gameId);
         }
       } catch (error) {
         this.logger.error(`处理长时间离线失败: ${error.message}`);
       }
     }

     // 判断是否应该为离线玩家启用AI代替
     private shouldAutoPlayForOfflinePlayer(gameState: GameState): boolean {
       // 根据游戏规则和当前阶段判断
       // 这里只是简单示例，实际逻辑可能更复杂
       return gameState.currentPhase !== GamePhase.TEAM_BUILDING;
     }

     // 为离线玩家启用AI代替
     private async enableAutoPlayForPlayer(
       userId: string,
       gameId: string
     ): Promise<void> {
       try {
         // 更新游戏状态，标记玩家由AI控制
         const gameState = await this.gameStateService.getGameState(gameId);
         if (!gameState) return;

         const playerIndex = gameState.players.findIndex(
           (p) => p.userId === userId
         );
         if (playerIndex >= 0) {
           gameState.players[playerIndex].isAiControlled = true;

           await this.gameStateService.updateGameState(gameId, {
             players: gameState.players,
           });

           // 通知其他玩家
           const message = {
             id: uuid(),
             type: MessageType.PLAYER_AI_CONTROL,
             timestamp: Date.now(),
             payload: { userId },
           };

           await this.messageService.sendToRoom(gameState.roomId, message);
         }
       } catch (error) {
         this.logger.error(`启用AI代替失败: ${error.message}`);
       }
     }

     // 因玩家离线终止游戏
     private async terminateGameDueToOffline(
       userId: string,
       roomId: string,
       gameId: string
     ): Promise<void> {
       try {
         // 更新游戏状态为中止
         await this.gameStateService.updateGameState(gameId, {
           status: GameStatus.TERMINATED,
           terminationReason: "player_offline",
         });

         // 通知玩家
         const message = {
           id: uuid(),
           type: MessageType.GAME_TERMINATED,
           timestamp: Date.now(),
           payload: {
             reason: "player_offline",
             playerId: userId,
           },
         };

         await this.messageService.sendToRoom(roomId, message);
       } catch (error) {
         this.logger.error(`终止游戏失败: ${error.message}`);
       }
     }
   }
   ```

## 工作量估计

3 人天

## 技术关键点

1. 设计可靠的连接状态跟踪和心跳机制
2. 确保游戏会话数据的安全存储和快速恢复
3. 实现高效的状态同步算法，减少重连时的数据传输
4. 设计合理的离线玩家处理策略，平衡游戏体验和公平性
5. 处理网络不稳定场景下的各种边界情况

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.2: WebSocket 服务](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Socket.IO 重连机制文档](https://socket.io/docs/client-api/#Reconnection)
- [Redis 会话管理最佳实践](https://redis.io/topics/data-types-intro)

## 依赖关系

- 上游依赖:
  - Task 4.2.1: WebSocket 服务框架搭建
  - Task 4.2.2: 实时消息系统实现
  - Task 4.2.3: 房间与游戏状态同步
- 下游依赖:
  - Task 4.2.5: 性能优化与扩展
