# Task 4.2.3: 房间与游戏状态同步

## 描述

设计并实现阿瓦隆微信小游戏的房间和游戏状态同步机制，确保所有玩家能够实时获取房间状态变化和游戏进程更新，保证游戏体验的一致性和实时性。

## 验收标准

1. 实现房间状态的实时同步和广播
2. 设计并实现游戏状态模型和状态变更机制
3. 完成游戏状态的增量更新和实时推送
4. 保证多玩家环境下的状态一致性
5. 实现状态同步的冲突解决策略
6. 优化状态数据的传输大小
7. 确保状态同步的低延迟
8. 完成状态同步的单元测试和集成测试

## 详细内容

### 房间状态管理

1. **房间状态模型**

   ```typescript
   // 简化的房间状态模型
   export interface RoomState {
     roomId: string;
     roomCode: string;
     status: RoomStatus;
     players: PlayerInfo[];
     owner: string;
     settings: GameSettings;
     createdAt: number;
     updatedAt: number;
   }

   export enum RoomStatus {
     WAITING = "waiting",
     PLAYING = "playing",
     FINISHED = "finished",
   }

   export interface PlayerInfo {
     userId: string;
     nickname: string;
     avatarUrl: string;
     isReady: boolean;
     isOnline: boolean;
     joinedAt: number;
   }

   export interface GameSettings {
     playerCount: number;
     roleConfig: RoleConfig;
     enableVoiceChat: boolean;
   }
   ```

2. **房间状态服务**

   ```typescript
   @Injectable()
   export class RoomStateService {
     constructor(
       private readonly messageService: MessageService,
       private readonly redisService: RedisService,
       private readonly logger: LoggerService
     ) {}

     // 获取房间状态
     async getRoomState(roomId: string): Promise<RoomState | null> {
       try {
         const roomData = await this.redisService.get(`room:${roomId}`);
         return roomData ? JSON.parse(roomData) : null;
       } catch (error) {
         this.logger.error(`获取房间状态失败: ${error.message}`);
         return null;
       }
     }

     // 更新房间状态
     async updateRoomState(
       roomId: string,
       updates: Partial<RoomState>
     ): Promise<RoomState> {
       try {
         // 获取当前状态
         const currentState = await this.getRoomState(roomId);
         if (!currentState) {
           throw new Error(`房间不存在: ${roomId}`);
         }

         // 更新状态
         const newState = {
           ...currentState,
           ...updates,
           updatedAt: Date.now(),
         };

         // 保存状态
         await this.redisService.set(
           `room:${roomId}`,
           JSON.stringify(newState)
         );

         // 广播更新
         await this.broadcastRoomUpdate(newState);
         return newState;
       } catch (error) {
         this.logger.error(`更新房间状态失败: ${error.message}`);
         throw error;
       }
     }

     // 广播房间状态更新
     private async broadcastRoomUpdate(roomState: RoomState): Promise<void> {
       const message = {
         id: uuid(),
         type: MessageType.ROOM_UPDATE,
         timestamp: Date.now(),
         payload: roomState,
       };
       await this.messageService.sendToRoom(roomState.roomId, message);
     }

     // 玩家加入房间
     async playerJoinRoom(
       roomId: string,
       player: PlayerInfo
     ): Promise<RoomState> {
       const room = await this.getRoomState(roomId);
       if (!room) {
         throw new Error(`房间不存在: ${roomId}`);
       }

       // 判断玩家是否已在房间内
       const existingPlayer = room.players.find(
         (p) => p.userId === player.userId
       );
       if (existingPlayer) {
         // 更新玩家状态
         existingPlayer.isOnline = true;
         return this.updateRoomState(roomId, { players: room.players });
       }

       // 添加新玩家
       room.players.push(player);
       return this.updateRoomState(roomId, { players: room.players });
     }

     // 玩家离开房间
     async playerLeaveRoom(roomId: string, userId: string): Promise<RoomState> {
       const room = await this.getRoomState(roomId);
       if (!room) {
         throw new Error(`房间不存在: ${roomId}`);
       }

       // 更新玩家列表
       const updatedPlayers = room.players.filter((p) => p.userId !== userId);

       // 如果房主离开，转移房主权限
       let updatedOwner = room.owner;
       if (room.owner === userId && updatedPlayers.length > 0) {
         updatedOwner = updatedPlayers[0].userId;
       }

       return this.updateRoomState(roomId, {
         players: updatedPlayers,
         owner: updatedOwner,
       });
     }
   }
   ```

### 游戏状态同步

1. **游戏状态模型**

   ```typescript
   // 简化的游戏状态模型
   export interface GameState {
     gameId: string;
     roomId: string;
     status: GameStatus;
     currentRound: number;
     currentPhase: GamePhase;
     players: GamePlayer[];
     missions: Mission[];
     currentTeam: string[];
     voteResults: VoteResult[];
     phaseEndTime: number;
     version: number; // 用于状态版本控制
   }

   export enum GameStatus {
     INITIALIZING = "initializing",
     ACTIVE = "active",
     PAUSED = "paused",
     COMPLETED = "completed",
   }

   export enum GamePhase {
     ROLE_ASSIGNMENT = "roleAssignment",
     TEAM_BUILDING = "teamBuilding",
     TEAM_VOTING = "teamVoting",
     MISSION_EXECUTION = "missionExecution",
     ASSASSINATION = "assassination",
     GAME_END = "gameEnd",
   }

   export interface GamePlayer {
     userId: string;
     role?: GameRole;
     isLeader: boolean;
     isOnTeam: boolean;
   }

   export interface Mission {
     round: number;
     requiredPlayers: number;
     teamMembers: string[];
     votes: MissionVote[];
     status: MissionStatus;
     result?: boolean;
   }
   ```

2. **游戏状态服务**

   ```typescript
   @Injectable()
   export class GameStateService {
     constructor(
       private readonly messageService: MessageService,
       private readonly redisService: RedisService,
       private readonly logger: LoggerService
     ) {}

     // 获取游戏状态
     async getGameState(gameId: string): Promise<GameState | null> {
       try {
         const gameData = await this.redisService.get(`game:${gameId}`);
         return gameData ? JSON.parse(gameData) : null;
       } catch (error) {
         this.logger.error(`获取游戏状态失败: ${error.message}`);
         return null;
       }
     }

     // 更新游戏状态
     async updateGameState(
       gameId: string,
       updates: Partial<GameState>
     ): Promise<GameState> {
       try {
         // 获取当前状态
         const currentState = await this.getGameState(gameId);
         if (!currentState) {
           throw new Error(`游戏不存在: ${gameId}`);
         }

         // 版本递增，用于并发控制
         const newVersion = currentState.version + 1;

         // 更新状态
         const newState = {
           ...currentState,
           ...updates,
           version: newVersion,
         };

         // 保存状态
         await this.redisService.set(
           `game:${gameId}`,
           JSON.stringify(newState)
         );

         // 广播更新
         await this.broadcastGameUpdate(newState);
         return newState;
       } catch (error) {
         this.logger.error(`更新游戏状态失败: ${error.message}`);
         throw error;
       }
     }

     // 广播游戏状态更新
     private async broadcastGameUpdate(gameState: GameState): Promise<void> {
       const message = {
         id: uuid(),
         type: MessageType.GAME_STATE,
         timestamp: Date.now(),
         payload: this.sanitizeGameState(gameState),
       };
       await this.messageService.sendToRoom(gameState.roomId, message);
     }

     // 净化游戏状态，移除敏感信息
     private sanitizeGameState(gameState: GameState): GameState {
       // 创建状态副本，避免修改原始对象
       const sanitized = JSON.parse(JSON.stringify(gameState));

       // 移除角色信息，每个玩家只能看到自己的角色和游戏规则允许的其他玩家角色
       sanitized.players.forEach((player) => {
         delete player.role;
       });

       return sanitized;
     }

     // 为特定玩家准备个性化游戏状态
     async getPersonalizedGameState(
       gameId: string,
       userId: string
     ): Promise<GameState | null> {
       const gameState = await this.getGameState(gameId);
       if (!gameState) return null;

       // 创建状态副本
       const personalState = JSON.parse(JSON.stringify(gameState));

       // 添加玩家可见的信息
       this.addVisibleRoleInfo(personalState, userId);

       return personalState;
     }

     // 添加玩家可见的角色信息
     private addVisibleRoleInfo(gameState: GameState, userId: string): void {
       // 获取当前玩家
       const currentPlayer = gameState.players.find((p) => p.userId === userId);
       if (!currentPlayer || !currentPlayer.role) return;

       // 根据游戏规则，为不同角色添加可见信息
       // 这里仅简单实现，实际逻辑会更复杂
       switch (currentPlayer.role) {
         case GameRole.MERLIN:
           // 梅林可以看到所有坏人(除莫德雷德)
           gameState.players.forEach((p) => {
             if (
               p.role === GameRole.ASSASSIN ||
               p.role === GameRole.MORGANA ||
               p.role === GameRole.OBERON
             ) {
               p.role = p.role;
             }
           });
           break;
         // 其他角色的可见逻辑...
       }
     }
   }
   ```

### 增量状态更新

1. **状态差异计算与传输**

   ```typescript
   export class StateDiffUtil {
     // 计算状态差异
     static calculateDiff(oldState: any, newState: any): any {
       const diff = {};

       // 简单深度比较生成差异对象
       for (const key in newState) {
         if (typeof newState[key] === "object" && newState[key] !== null) {
           if (
             JSON.stringify(oldState[key]) !== JSON.stringify(newState[key])
           ) {
             diff[key] = newState[key];
           }
         } else if (oldState[key] !== newState[key]) {
           diff[key] = newState[key];
         }
       }

       return Object.keys(diff).length > 0 ? diff : null;
     }

     // 应用差异到状态
     static applyDiff(state: any, diff: any): any {
       if (!diff) return state;

       const result = { ...state };

       for (const key in diff) {
         if (typeof diff[key] === "object" && diff[key] !== null) {
           result[key] = { ...result[key], ...diff[key] };
         } else {
           result[key] = diff[key];
         }
       }

       return result;
     }
   }
   ```

2. **状态冲突解决**

   ```typescript
   @Injectable()
   export class StateConflictResolver {
     // 解决并发状态更新冲突
     resolveConflict(
       serverState: GameState,
       clientState: GameState
     ): GameState {
       // 优先使用版本号高的状态
       if (clientState.version > serverState.version) {
         return clientState;
       }

       if (serverState.version > clientState.version) {
         return serverState;
       }

       // 版本相同，合并状态（以服务器为准）
       return {
         ...clientState,
         ...serverState,
       };
     }
   }
   ```

## 工作量估计

3 人天

## 技术关键点

1. 设计高效的状态更新和同步机制
2. 确保状态一致性和冲突解决策略
3. 优化状态数据传输大小，减少网络负载
4. 处理网络延迟和状态同步的时序问题
5. 实现不同角色的信息可见性控制

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.2: WebSocket 服务](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Socket.IO 文档](https://socket.io/docs/)
- [Redis 文档](https://redis.io/documentation)

## 依赖关系

- 上游依赖:
  - Task 4.2.1: WebSocket 服务框架搭建
  - Task 4.2.2: 实时消息系统实现
- 下游依赖:
  - Task 4.2.4: 断线重连与状态恢复
  - Task 4.2.5: 性能优化与扩展
