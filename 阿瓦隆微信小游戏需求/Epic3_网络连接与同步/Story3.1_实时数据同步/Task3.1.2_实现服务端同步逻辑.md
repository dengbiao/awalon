# Task 3.1.2: 实现服务端同步逻辑

## 任务描述

实现阿瓦隆微信小游戏的服务端同步逻辑，作为游戏状态的权威来源，负责验证和处理客户端的动作请求，维护一致的游戏状态，并将状态变更可靠地广播给所有玩家。服务端同步系统还需要处理并发操作、解决冲突，以及处理网络异常情况。

## 功能需求

1. **游戏状态管理**：

   - 创建和维护权威游戏状态
   - 实现状态存储和持久化
   - 支持游戏状态回滚和恢复

2. **客户端动作处理**：

   - 接收并验证客户端动作请求
   - 应用有效动作到游戏状态
   - 返回动作处理结果给客户端

3. **状态同步广播**：

   - 将状态变更广播给房间内所有玩家
   - 实现状态快照和增量更新机制
   - 支持选择性广播和优先级排序

4. **并发控制**：

   - 实现操作排序和冲突解决
   - 防止状态不一致和竞态条件
   - 处理多客户端同时操作的场景

5. **异常处理**：
   - 处理客户端断连和重连
   - 实现错误恢复和状态修复
   - 监控和记录同步异常

## 技术需求

1. 服务端必须使用 Node.js 实现，支持 WebSocket 连接
2. 支持每个游戏房间至少 10 个并发连接
3. 动作验证和处理的平均延迟必须小于 50ms
4. 状态广播延迟必须小于 100ms
5. 服务端必须能处理每秒至少 100 个动作请求
6. 实现基于 Redis 的分布式状态存储
7. 支持水平扩展，允许多个服务器实例协同工作

## 实现步骤

1. **设计服务端架构**：

   - 规划服务器组件和交互模式
   - 设计数据流和处理模型
   - 确定状态存储和访问模式

2. **实现核心组件**：

   - 开发 WebSocket 连接管理器
   - 实现游戏状态服务
   - 开发动作处理器
   - 构建状态广播系统

3. **实现并发控制**：

   - 开发操作队列和排序机制
   - 实现冲突检测和解决策略
   - 添加事务支持和原子操作

4. **开发容错机制**：

   - 实现错误处理和恢复
   - 开发状态备份和回滚功能
   - 实现健康监控和报警

5. **性能优化**：
   - 优化状态处理和广播性能
   - 实现缓存和预处理机制
   - 优化数据传输效率

## 示例代码

### 服务端架构设计

```typescript
// 服务端应用入口
import * as http from "http";
import * as WebSocket from "ws";
import { GameStateService } from "./services/GameStateService";
import { ActionProcessor } from "./services/ActionProcessor";
import { ConnectionManager } from "./services/ConnectionManager";
import { BroadcastService } from "./services/BroadcastService";
import { RedisStateStore } from "./storage/RedisStateStore";

// 创建HTTP服务器
const server = http.createServer();
const wss = new WebSocket.Server({ server });

// 初始化服务
const stateStore = new RedisStateStore({
  host: process.env.REDIS_HOST || "localhost",
  port: parseInt(process.env.REDIS_PORT || "6379"),
});

const gameStateService = new GameStateService(stateStore);
const actionProcessor = new ActionProcessor(gameStateService);
const connectionManager = new ConnectionManager(wss);
const broadcastService = new BroadcastService(connectionManager);

// 注册广播监听器
gameStateService.on("stateChanged", (roomId, state, changes) => {
  broadcastService.broadcastState(roomId, state, changes);
});

// 启动服务器
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`游戏服务器运行在端口 ${PORT}`);
});

// 处理连接
connectionManager.on("connection", (client, request) => {
  console.log("新客户端连接");

  client.on("message", async (message: WebSocket.Data) => {
    try {
      // 解析消息
      const parsedMessage = JSON.parse(message.toString());

      // 处理不同类型的消息
      switch (parsedMessage.header.type) {
        case "PLAYER_ACTION":
          // 处理玩家动作
          const result = await actionProcessor.processAction(parsedMessage);
          client.send(JSON.stringify(result));
          break;

        case "JOIN_ROOM":
          // 处理加入房间请求
          const roomState = await gameStateService.getOrCreateRoom(
            parsedMessage.payload.roomId
          );
          connectionManager.addClientToRoom(
            client,
            parsedMessage.payload.roomId
          );
          // 发送完整状态快照
          client.send(
            JSON.stringify({
              header: {
                type: "STATE_SNAPSHOT",
                timestamp: Date.now(),
                roomId: parsedMessage.payload.roomId,
                messageId: `snapshot_${Date.now()}`,
                sequenceNumber: 0,
              },
              payload: {
                gameState: roomState,
                versionId: roomState.versionId,
              },
            })
          );
          break;

        // 处理其他消息类型...
      }
    } catch (error) {
      console.error("处理消息时出错:", error);
      // 发送错误响应
      client.send(
        JSON.stringify({
          header: {
            type: "ERROR",
            timestamp: Date.now(),
          },
          payload: {
            errorCode: 500,
            message: "服务器内部错误",
          },
        })
      );
    }
  });

  client.on("close", () => {
    console.log("客户端断开连接");
    connectionManager.removeClient(client);
  });
});
```

### 游戏状态服务实现

```typescript
// GameStateService.ts
import { EventEmitter } from "events";
import { StateStore } from "../storage/StateStore";
import { generateStateDelta } from "../utils/deltaUtils";

export class GameStateService extends EventEmitter {
  private stateVersions: Map<string, string[]> = new Map();
  private pendingActions: Map<string, Set<string>> = new Map();

  constructor(private stateStore: StateStore) {
    super();
  }

  // 获取或创建游戏房间
  public async getOrCreateRoom(roomId: string): Promise<any> {
    try {
      // 尝试获取现有房间
      const roomState = await this.stateStore.getState(roomId);
      if (roomState) {
        return roomState;
      }

      // 创建新房间
      const initialState = this.createInitialState(roomId);
      await this.stateStore.setState(roomId, initialState);

      // 初始化版本历史
      this.stateVersions.set(roomId, [initialState.versionId]);

      return initialState;
    } catch (error) {
      console.error(`获取或创建房间失败: ${roomId}`, error);
      throw new Error(`无法访问房间: ${roomId}`);
    }
  }

  // 更新游戏状态
  public async updateState(
    roomId: string,
    actionId: string,
    updateFn: (state: any) => any
  ): Promise<{ newState: any; delta: any }> {
    // 获取锁以确保串行处理
    const lock = await this.stateStore.acquireLock(roomId, 5000);
    try {
      // 添加到待处理队列
      this.addPendingAction(roomId, actionId);

      // 获取当前状态
      const currentState = await this.stateStore.getState(roomId);
      if (!currentState) {
        throw new Error(`房间不存在: ${roomId}`);
      }

      // 应用更新函数
      const newState = updateFn(JSON.parse(JSON.stringify(currentState)));

      // 更新版本ID
      newState.versionId = this.generateVersionId(roomId);

      // 生成增量更新
      const delta = generateStateDelta(currentState, newState);

      // 保存新状态
      await this.stateStore.setState(roomId, newState);

      // 更新版本历史
      this.updateVersionHistory(roomId, newState.versionId);

      // 移除已处理的动作
      this.removePendingAction(roomId, actionId);

      // 触发状态变更事件
      this.emit("stateChanged", roomId, newState, delta);

      return { newState, delta };
    } finally {
      // 释放锁
      await this.stateStore.releaseLock(roomId, lock);
    }
  }

  // 验证动作的合法性
  public async validateAction(roomId: string, action: any): Promise<boolean> {
    const state = await this.stateStore.getState(roomId);
    if (!state) {
      return false;
    }

    // 根据游戏规则验证动作
    // 这里是简化的示例，实际实现会更复杂
    switch (action.payload.actionType) {
      case "SELECT_TEAM":
        // 检查是否是队长的回合
        return state.currentLeader === action.payload.playerId;

      case "VOTE_FOR_TEAM":
        // 检查是否在投票阶段且玩家未投票
        return (
          state.phase === "TEAM_VOTING" &&
          !state.votes.some((v: any) => v.playerId === action.payload.playerId)
        );

      case "VOTE_FOR_MISSION":
        // 检查玩家是否在任务团队中且未投票
        const inTeam = state.currentTeam.includes(action.payload.playerId);
        const hasVoted = state.missionVotes.some(
          (v: any) => v.playerId === action.payload.playerId
        );
        return state.phase === "MISSION_VOTING" && inTeam && !hasVoted;

      // ... 其他动作类型的验证

      default:
        return false;
    }
  }

  // 创建初始游戏状态
  private createInitialState(roomId: string): any {
    return {
      roomId,
      versionId: this.generateVersionId(roomId),
      createdAt: Date.now(),
      updatedAt: Date.now(),
      players: [],
      gameStarted: false,
      phase: "WAITING_FOR_PLAYERS",
      round: 0,
      mission: 0,
      rejectedTeams: 0,
      currentLeader: null,
      currentTeam: [],
      votes: [],
      missionVotes: [],
      missions: [
        { required: 2, result: null },
        { required: 3, result: null },
        { required: 2, result: null },
        { required: 3, result: null },
        { required: 3, result: null },
      ],
      roles: {},
    };
  }

  // 生成新的版本ID
  private generateVersionId(roomId: string): string {
    return `${roomId}_${Date.now()}_${Math.floor(Math.random() * 10000)}`;
  }

  // 更新版本历史
  private updateVersionHistory(roomId: string, versionId: string): void {
    const versions = this.stateVersions.get(roomId) || [];
    versions.push(versionId);

    // 只保留最近的20个版本
    if (versions.length > 20) {
      versions.shift();
    }

    this.stateVersions.set(roomId, versions);
  }

  // 添加待处理动作
  private addPendingAction(roomId: string, actionId: string): void {
    const actions = this.pendingActions.get(roomId) || new Set();
    actions.add(actionId);
    this.pendingActions.set(roomId, actions);
  }

  // 移除已处理动作
  private removePendingAction(roomId: string, actionId: string): void {
    const actions = this.pendingActions.get(roomId);
    if (actions) {
      actions.delete(actionId);
    }
  }
}
```

### 动作处理器实现

```typescript
// ActionProcessor.ts
import { GameStateService } from "./GameStateService";

export class ActionProcessor {
  constructor(private gameStateService: GameStateService) {}

  // 处理玩家动作
  public async processAction(actionMessage: any): Promise<any> {
    const { roomId, playerId, actionType, actionData } = actionMessage.payload;
    const actionId = actionMessage.header.messageId;

    try {
      // 验证动作合法性
      const isValid = await this.gameStateService.validateAction(
        roomId,
        actionMessage
      );
      if (!isValid) {
        return this.createErrorResponse(
          actionId,
          "INVALID_ACTION",
          "无效的动作请求"
        );
      }

      // 根据动作类型更新游戏状态
      const { newState, delta } = await this.gameStateService.updateState(
        roomId,
        actionId,
        (state) => {
          // 复制状态以便进行修改
          const newState = { ...state };

          // 根据动作类型进行更新
          switch (actionType) {
            case "SELECT_TEAM":
              newState.currentTeam = actionData.team;
              newState.phase = "TEAM_VOTING";
              newState.votes = [];
              break;

            case "VOTE_FOR_TEAM":
              newState.votes.push({
                playerId,
                vote: actionData.vote,
                timestamp: Date.now(),
              });

              // 检查是否所有玩家都已投票
              if (newState.votes.length === newState.players.length) {
                // 计算投票结果
                const approvals = newState.votes.filter(
                  (v: any) => v.vote === "APPROVE"
                ).length;
                const majority = Math.floor(newState.players.length / 2) + 1;

                if (approvals >= majority) {
                  // 团队被批准
                  newState.phase = "MISSION_VOTING";
                  newState.missionVotes = [];
                } else {
                  // 团队被拒绝
                  newState.rejectedTeams++;

                  // 检查是否达到连续拒绝上限
                  if (newState.rejectedTeams >= 5) {
                    newState.phase = "EVIL_WINS";
                    newState.gameOver = true;
                    newState.winner = "EVIL";
                  } else {
                    // 移动到下一个队长
                    const currentLeaderIndex = newState.players.indexOf(
                      newState.currentLeader
                    );
                    const nextLeaderIndex =
                      (currentLeaderIndex + 1) % newState.players.length;
                    newState.currentLeader = newState.players[nextLeaderIndex];
                    newState.phase = "TEAM_SELECTION";
                    newState.currentTeam = [];
                  }
                }
              }
              break;

            case "VOTE_FOR_MISSION":
              newState.missionVotes.push({
                playerId,
                vote: actionData.vote,
                timestamp: Date.now(),
              });

              // 检查是否所有团队成员都已投票
              if (
                newState.missionVotes.length === newState.currentTeam.length
              ) {
                // 计算任务结果
                const fails = newState.missionVotes.filter(
                  (v: any) => v.vote === "FAIL"
                ).length;
                const missionSuccess =
                  fails < newState.missions[newState.mission].required;

                // 更新任务结果
                newState.missions[newState.mission].result = missionSuccess
                  ? "SUCCESS"
                  : "FAIL";

                // 检查游戏是否结束
                const successMissions = newState.missions.filter(
                  (m: any) => m.result === "SUCCESS"
                ).length;
                const failMissions = newState.missions.filter(
                  (m: any) => m.result === "FAIL"
                ).length;

                if (successMissions >= 3) {
                  // 善良阵营可能获胜，但需要等待刺客刺杀阶段
                  newState.phase = "ASSASSINATION";
                } else if (failMissions >= 3) {
                  // 邪恶阵营获胜
                  newState.phase = "EVIL_WINS";
                  newState.gameOver = true;
                  newState.winner = "EVIL";
                } else {
                  // 游戏继续
                  newState.mission++;

                  // 移动到下一个队长
                  const currentLeaderIndex = newState.players.indexOf(
                    newState.currentLeader
                  );
                  const nextLeaderIndex =
                    (currentLeaderIndex + 1) % newState.players.length;
                  newState.currentLeader = newState.players[nextLeaderIndex];

                  newState.phase = "TEAM_SELECTION";
                  newState.currentTeam = [];
                  newState.rejectedTeams = 0;
                }
              }
              break;

            // ... 处理其他动作类型

            default:
              // 未知动作类型，不做任何处理
              break;
          }

          // 更新时间戳
          newState.updatedAt = Date.now();

          return newState;
        }
      );

      // 创建成功响应
      return {
        header: {
          type: "ACTION_RESULT",
          messageId: `result_${actionId}`,
          timestamp: Date.now(),
          roomId,
        },
        payload: {
          actionId,
          success: true,
          resultData: {
            stateVersion: newState.versionId,
            stateChanged: delta.operations.length > 0,
          },
        },
      };
    } catch (error) {
      console.error(`处理动作失败: ${actionId}`, error);
      return this.createErrorResponse(
        actionId,
        "PROCESSING_ERROR",
        "处理动作时发生错误"
      );
    }
  }

  // 创建错误响应
  private createErrorResponse(
    actionId: string,
    errorCode: string,
    errorMessage: string
  ): any {
    return {
      header: {
        type: "ACTION_RESULT",
        messageId: `error_${actionId}`,
        timestamp: Date.now(),
      },
      payload: {
        actionId,
        success: false,
        errorCode,
        errorMessage,
      },
    };
  }
}
```

### 状态广播服务实现

```typescript
// BroadcastService.ts
import { ConnectionManager } from "./ConnectionManager";

export class BroadcastService {
  constructor(private connectionManager: ConnectionManager) {}

  // 广播状态更新
  public broadcastState(roomId: string, state: any, delta: any): void {
    const clients = this.connectionManager.getClientsInRoom(roomId);
    if (!clients || clients.length === 0) {
      return;
    }

    // 创建状态增量消息
    const deltaMessage = {
      header: {
        type: "STATE_DELTA",
        timestamp: Date.now(),
        roomId,
        messageId: `delta_${Date.now()}`,
        sequenceNumber: 0,
        compression: delta.operations.length > 10, // 只有大的增量才进行压缩
      },
      payload: {
        baseVersionId: delta.baseVersionId,
        operations: delta.operations,
        resultVersionId: delta.resultVersionId,
      },
    };

    // 序列化消息
    const messageStr = JSON.stringify(deltaMessage);

    // 广播给所有客户端
    for (const client of clients) {
      if (client.readyState === client.OPEN) {
        client.send(messageStr);
      }
    }

    // 每30秒或状态变化较大时发送完整快照
    const shouldSendSnapshot = this.shouldSendSnapshot(roomId, delta);
    if (shouldSendSnapshot) {
      this.sendStateSnapshot(roomId, state, clients);
    }
  }

  // 发送状态快照
  private sendStateSnapshot(roomId: string, state: any, clients: any[]): void {
    // 创建快照消息
    const snapshotMessage = {
      header: {
        type: "STATE_SNAPSHOT",
        timestamp: Date.now(),
        roomId,
        messageId: `snapshot_${Date.now()}`,
        sequenceNumber: 0,
        compression: true, // 快照通常较大，总是压缩
      },
      payload: {
        gameState: state,
        versionId: state.versionId,
      },
    };

    // 序列化消息
    const messageStr = JSON.stringify(snapshotMessage);

    // 广播给所有客户端
    for (const client of clients) {
      if (client.readyState === client.OPEN) {
        client.send(messageStr);
      }
    }
  }

  // 判断是否应该发送完整快照
  private shouldSendSnapshot(roomId: string, delta: any): boolean {
    // 获取上次快照时间
    const lastSnapshotTime = this.getLastSnapshotTime(roomId);
    const now = Date.now();

    // 如果更改较大或距离上次快照超过30秒，则发送快照
    return delta.operations.length > 20 || now - lastSnapshotTime > 30000;
  }

  // 获取上次快照时间（简化实现）
  private lastSnapshotTimes: Map<string, number> = new Map();

  private getLastSnapshotTime(roomId: string): number {
    const time = this.lastSnapshotTimes.get(roomId) || 0;
    this.lastSnapshotTimes.set(roomId, Date.now());
    return time;
  }
}
```

### Redis 状态存储实现

```typescript
// RedisStateStore.ts
import * as Redis from "ioredis";
import { StateStore } from "./StateStore";

export class RedisStateStore implements StateStore {
  private redis: Redis.Redis;

  constructor(options: Redis.RedisOptions) {
    this.redis = new Redis(options);
  }

  // 获取游戏状态
  public async getState(roomId: string): Promise<any> {
    const key = this.getStateKey(roomId);
    const state = await this.redis.get(key);

    if (!state) {
      return null;
    }

    try {
      return JSON.parse(state);
    } catch (error) {
      console.error(`解析状态失败: ${roomId}`, error);
      return null;
    }
  }

  // 保存游戏状态
  public async setState(roomId: string, state: any): Promise<boolean> {
    const key = this.getStateKey(roomId);

    try {
      const stateStr = JSON.stringify(state);
      await this.redis.set(key, stateStr);

      // 设置过期时间（24小时），防止无用数据积累
      await this.redis.expire(key, 86400);

      return true;
    } catch (error) {
      console.error(`保存状态失败: ${roomId}`, error);
      return false;
    }
  }

  // 获取状态存储键
  private getStateKey(roomId: string): string {
    return `avalon:gamestate:${roomId}`;
  }

  // 获取分布式锁
  public async acquireLock(roomId: string, ttl: number): Promise<string> {
    const lockKey = this.getLockKey(roomId);
    const lockValue = Date.now().toString();

    // 尝试获取锁，使用NX（仅在键不存在时设置）和PX（以毫秒为单位设置过期时间）
    const result = await this.redis.set(lockKey, lockValue, "PX", ttl, "NX");

    if (result === "OK") {
      return lockValue;
    }

    // 如果锁已存在，等待一段时间后重试
    await new Promise((resolve) => setTimeout(resolve, 50));
    return this.acquireLock(roomId, ttl);
  }

  // 释放分布式锁
  public async releaseLock(
    roomId: string,
    lockValue: string
  ): Promise<boolean> {
    const lockKey = this.getLockKey(roomId);

    // 使用Lua脚本确保原子操作
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;

    const result = await this.redis.eval(script, 1, lockKey, lockValue);
    return result === 1;
  }

  // 获取锁的键
  private getLockKey(roomId: string): string {
    return `avalon:gamelock:${roomId}`;
  }
}
```

## 验收标准

1. 服务端能够成功接收、验证和处理客户端的所有游戏动作
2. 动作处理的平均响应时间小于 50ms，状态广播延迟小于 100ms
3. 状态变更能够正确广播给房间内所有玩家，包括增量更新和周期性快照
4. 并发操作能够正确排序和处理，不会导致游戏状态不一致
5. 服务端能够处理客户端异常断开和重连，正确恢复状态
6. 服务端能够处理每个房间每秒至少 10 个动作请求，总体支持每秒至少 100 个请求
7. 状态存储能够可靠地保存游戏状态，并在需要时恢复
8. 服务端在高负载和网络波动情况下保持稳定运行

## 技术依赖

1. Node.js 环境（v12+）
2. WebSocket 服务器（如 ws 库）
3. Redis 服务器（用于状态存储和分布式锁）
4. TypeScript 编译环境
5. 消息序列化库（如 MessagePack）
6. 状态差异计算库
7. 服务器监控工具

## 工作量估计

预计工作量：1 人/天
