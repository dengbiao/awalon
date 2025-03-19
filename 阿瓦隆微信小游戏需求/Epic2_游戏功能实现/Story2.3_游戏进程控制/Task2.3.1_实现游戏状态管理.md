# Task 2.3.1: 实现游戏状态管理

## 描述

实现阿瓦隆游戏的状态管理系统，负责维护游戏的全局状态、处理状态更新和变化通知，以及确保游戏状态的一致性和可恢复性。

## 详细需求

### 功能需求

1. 游戏状态维护：

   - 创建和初始化游戏状态
   - 存储游戏的当前阶段、玩家信息、角色分配、任务状态等核心数据
   - 提供状态查询接口，允许获取完整或部分游戏状态
   - 支持状态更新，确保状态变更的原子性和一致性

2. 状态变更通知：

   - 实现发布-订阅模式，允许其他模块订阅状态变更事件
   - 在状态变更时触发相应的事件通知
   - 支持针对特定状态变更的订阅（如阶段变化、任务状态变更等）
   - 提供事件过滤和优先级机制

3. 状态持久化和恢复：
   - 定期将游戏状态持久化到存储系统
   - 支持从持久化数据恢复游戏状态
   - 处理玩家断线重连时的状态同步
   - 提供游戏状态快照和回滚机制

### 技术要求

1. 实现高效的状态管理器，确保状态更新的性能
2. 设计合理的数据结构，支持游戏所有阶段的状态表示
3. 实现可靠的事件系统，确保事件通知的及时性和可靠性
4. 确保状态管理的线程安全和并发控制
5. 实现状态验证机制，防止非法状态更新
6. 支持状态的增量更新和差异计算，优化网络传输

## 实现步骤

1. 设计游戏状态数据结构
2. 实现状态管理器核心类
3. 实现事件发布-订阅系统
4. 实现状态持久化和恢复机制
5. 实现状态验证和安全控制
6. 实现状态差异计算和增量更新
7. 进行单元测试和性能测试

## 代码示例

### 游戏状态数据结构

```typescript
// 游戏阶段枚举
enum GamePhase {
  WAITING = "waiting", // 等待开始
  ROLE_ASSIGNMENT = "role_assignment", // 角色分配阶段
  ROLE_REVEAL = "role_reveal", // 角色揭示阶段
  MISSION = "mission", // 任务阶段
  ASSASSINATION = "assassination", // 刺杀阶段
  RESULT = "result", // 结果展示阶段
  ENDED = "ended", // 游戏结束
}

// 游戏状态
interface GameState {
  gameId: string; // 游戏ID
  status: GamePhase; // 当前游戏阶段
  players: Player[]; // 玩家列表
  playerRoles: Map<string, PlayerRole>; // 玩家角色映射
  currentMissionNumber: number; // 当前任务编号
  missions: MissionState[]; // 任务状态列表
  missionResults: MissionResult[]; // 任务结果列表
  captainHistory: Captain[]; // 队长历史
  consecutiveRejections: number; // 连续拒绝次数
  assassinationTarget: string | null; // 刺杀目标
  gameResult: GameResult | null; // 游戏结果
  createdAt: Date; // 创建时间
  updatedAt: Date; // 更新时间
}

// 玩家信息
interface Player {
  id: string; // 玩家ID
  name: string; // 玩家名称
  avatar: string; // 玩家头像
  isReady: boolean; // 是否准备就绪
  isConnected: boolean; // 是否连接
  roleConfirmed: boolean; // 是否确认角色
}

// 玩家角色
interface PlayerRole {
  playerId: string; // 玩家ID
  roleType: RoleType; // 角色类型
  visiblePlayers: VisiblePlayer[]; // 可见的其他玩家
}

// 可见玩家信息
interface VisiblePlayer {
  playerId: string; // 玩家ID
  name: string; // 玩家名称
  hint?: string; // 提示信息（如"可能是梅林"）
}
```

### 状态管理器实现

```typescript
class GameStateManager {
  private gameState: GameState;
  private eventEmitter: EventEmitter;
  private persistenceManager: PersistenceManager;

  constructor(gameId: string, players: Player[]) {
    // 初始化游戏状态
    this.gameState = {
      gameId,
      status: GamePhase.WAITING,
      players,
      playerRoles: new Map(),
      currentMissionNumber: 0,
      missions: [],
      missionResults: [],
      captainHistory: [],
      consecutiveRejections: 0,
      assassinationTarget: null,
      gameResult: null,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.eventEmitter = new EventEmitter();
    this.persistenceManager = new PersistenceManager();

    // 初始化后立即持久化
    this.persistState();
  }

  // 获取完整游戏状态
  public getGameState(): GameState {
    return { ...this.gameState };
  }

  // 获取特定玩家可见的游戏状态
  public getPlayerVisibleState(playerId: string): PlayerVisibleState {
    const gameState = this.gameState;
    const playerRole = gameState.playerRoles.get(playerId);

    // 创建玩家可见状态
    const visibleState: PlayerVisibleState = {
      gameId: gameState.gameId,
      status: gameState.status,
      players: gameState.players.map((p) => ({
        id: p.id,
        name: p.name,
        avatar: p.avatar,
        isReady: p.isReady,
        isConnected: p.isConnected,
      })),
      currentMissionNumber: gameState.currentMissionNumber,
      missionResults: gameState.missionResults,
      consecutiveRejections: gameState.consecutiveRejections,
      gameResult: gameState.gameResult,

      // 只包含当前玩家的角色信息
      playerRole: playerRole || null,

      // 任务信息（移除敏感信息）
      missions: this.filterMissionsForPlayer(gameState.missions, playerId),
    };

    return visibleState;
  }

  // 更新游戏状态
  public updateGameState(updates: Partial<GameState>): void {
    // 验证状态更新的合法性
    this.validateStateUpdate(updates);

    const previousState = { ...this.gameState };

    // 应用更新
    this.gameState = {
      ...this.gameState,
      ...updates,
      updatedAt: new Date(),
    };

    // 计算状态差异
    const stateDiff = this.calculateStateDiff(previousState, this.gameState);

    // 触发状态更新事件
    this.eventEmitter.emit("stateUpdated", {
      previousState,
      currentState: this.gameState,
      diff: stateDiff,
    });

    // 如果阶段发生变化，触发阶段转换事件
    if (previousState.status !== this.gameState.status) {
      this.eventEmitter.emit("phaseTransition", {
        previousPhase: previousState.status,
        currentPhase: this.gameState.status,
        reason: "状态更新",
      });
    }

    // 持久化更新后的状态
    this.persistState();
  }

  // 订阅状态变化
  public subscribeToStateChanges(callback: Function): () => void {
    this.eventEmitter.on("stateUpdated", callback);

    // 返回取消订阅的函数
    return () => {
      this.eventEmitter.off("stateUpdated", callback);
    };
  }

  // 订阅阶段转换
  public subscribeToPhaseTransition(callback: Function): () => void {
    this.eventEmitter.on("phaseTransition", callback);

    // 返回取消订阅的函数
    return () => {
      this.eventEmitter.off("phaseTransition", callback);
    };
  }

  // 处理玩家断线
  public handlePlayerDisconnect(playerId: string): void {
    // 标记玩家断线
    const players = this.gameState.players.map((p) =>
      p.id === playerId ? { ...p, isConnected: false } : p
    );

    this.updateGameState({ players });

    // 触发玩家断线事件
    this.eventEmitter.emit("playerDisconnected", { playerId });

    // 检查是否需要暂停游戏
    this.checkGamePause();
  }

  // 处理玩家重连
  public handlePlayerReconnect(playerId: string): void {
    // 标记玩家重新连接
    const players = this.gameState.players.map((p) =>
      p.id === playerId ? { ...p, isConnected: true } : p
    );

    this.updateGameState({ players });

    // 触发玩家重连事件
    this.eventEmitter.emit("playerReconnected", { playerId });

    // 检查是否可以恢复游戏
    this.checkGameResume();
  }

  // 从持久化存储恢复状态
  public async restoreState(gameId: string): Promise<boolean> {
    try {
      const savedState = await this.persistenceManager.loadGameState(gameId);

      if (savedState) {
        this.gameState = savedState;

        // 触发状态恢复事件
        this.eventEmitter.emit("stateRestored", { gameState: this.gameState });
        return true;
      }

      return false;
    } catch (error) {
      console.error("Failed to restore game state:", error);
      return false;
    }
  }

  // 创建状态快照
  public createSnapshot(): GameStateSnapshot {
    return {
      gameState: { ...this.gameState },
      timestamp: new Date(),
      snapshotId: generateUniqueId(),
    };
  }

  // 回滚到指定快照
  public rollbackToSnapshot(snapshot: GameStateSnapshot): void {
    this.gameState = { ...snapshot.gameState };

    // 触发状态回滚事件
    this.eventEmitter.emit("stateRollback", {
      currentState: this.gameState,
      snapshotTimestamp: snapshot.timestamp,
    });

    // 持久化回滚后的状态
    this.persistState();
  }

  // 持久化当前状态
  private async persistState(): Promise<void> {
    try {
      await this.persistenceManager.saveGameState(this.gameState);
    } catch (error) {
      console.error("Failed to persist game state:", error);

      // 触发持久化失败事件
      this.eventEmitter.emit("persistenceFailed", { error });
    }
  }

  // 验证状态更新的合法性
  private validateStateUpdate(updates: Partial<GameState>): void {
    // 实现状态验证逻辑
    // 例如：检查阶段转换是否合法，检查数据完整性等
    // 如果验证失败，抛出异常
    // throw new Error("Invalid state update");
  }

  // 计算状态差异
  private calculateStateDiff(
    previousState: GameState,
    currentState: GameState
  ): StateDiff {
    // 实现状态差异计算逻辑
    // 返回变更的字段和值

    return {
      changedFields: [], // 变更的字段列表
      changes: {}, // 具体变更内容
    };
  }

  // 过滤任务信息，移除对特定玩家不可见的信息
  private filterMissionsForPlayer(
    missions: MissionState[],
    playerId: string
  ): MissionState[] {
    return missions.map((mission) => {
      // 创建任务副本
      const filteredMission = { ...mission };

      // 如果任务正在执行中，且玩家不是队员，隐藏执行结果
      if (
        mission.status === MissionStatus.EXECUTING &&
        !mission.currentTeam?.memberIds.includes(playerId)
      ) {
        filteredMission.executionResults = new Map();
      }

      return filteredMission;
    });
  }

  // 检查是否需要暂停游戏
  private checkGamePause(): void {
    const disconnectedCount = this.gameState.players.filter(
      (p) => !p.isConnected
    ).length;

    // 如果断线玩家超过一定比例，暂停游戏
    if (disconnectedCount > this.gameState.players.length / 3) {
      this.eventEmitter.emit("gamePaused", {
        reason: "过多玩家断线",
        disconnectedCount,
      });
    }
  }

  // 检查是否可以恢复游戏
  private checkGameResume(): void {
    const disconnectedCount = this.gameState.players.filter(
      (p) => !p.isConnected
    ).length;

    // 如果断线玩家数量降低到阈值以下，恢复游戏
    if (disconnectedCount <= this.gameState.players.length / 4) {
      this.eventEmitter.emit("gameResumed", {
        reason: "大多数玩家已重新连接",
      });
    }
  }
}
```

### 事件发布-订阅系统

```typescript
class EventEmitter {
  private events: Map<string, Set<Function>>;
  private priorityEvents: Map<string, Map<number, Set<Function>>>;

  constructor() {
    this.events = new Map();
    this.priorityEvents = new Map();
  }

  // 注册事件监听器
  public on(eventName: string, callback: Function): void {
    if (!this.events.has(eventName)) {
      this.events.set(eventName, new Set());
    }

    this.events.get(eventName)!.add(callback);
  }

  // 注册带优先级的事件监听器
  public onWithPriority(
    eventName: string,
    callback: Function,
    priority: number
  ): void {
    if (!this.priorityEvents.has(eventName)) {
      this.priorityEvents.set(eventName, new Map());
    }

    const priorityMap = this.priorityEvents.get(eventName)!;

    if (!priorityMap.has(priority)) {
      priorityMap.set(priority, new Set());
    }

    priorityMap.get(priority)!.add(callback);
  }

  // 移除事件监听器
  public off(eventName: string, callback: Function): void {
    if (this.events.has(eventName)) {
      this.events.get(eventName)!.delete(callback);
    }

    if (this.priorityEvents.has(eventName)) {
      const priorityMap = this.priorityEvents.get(eventName)!;

      for (const [priority, callbacks] of priorityMap.entries()) {
        if (callbacks.delete(callback) && callbacks.size === 0) {
          priorityMap.delete(priority);
        }
      }

      if (priorityMap.size === 0) {
        this.priorityEvents.delete(eventName);
      }
    }
  }

  // 触发事件
  public emit(eventName: string, data: any): void {
    // 先触发优先级事件
    if (this.priorityEvents.has(eventName)) {
      const priorityMap = this.priorityEvents.get(eventName)!;
      const priorities = Array.from(priorityMap.keys()).sort((a, b) => b - a); // 降序排列

      for (const priority of priorities) {
        const callbacks = priorityMap.get(priority)!;

        for (const callback of callbacks) {
          try {
            callback(data);
          } catch (error) {
            console.error(`Error in event handler for ${eventName}:`, error);
          }
        }
      }
    }

    // 再触发普通事件
    if (this.events.has(eventName)) {
      const callbacks = this.events.get(eventName)!;

      for (const callback of callbacks) {
        try {
          callback(data);
        } catch (error) {
          console.error(`Error in event handler for ${eventName}:`, error);
        }
      }
    }
  }

  // 清除所有事件监听器
  public clear(): void {
    this.events.clear();
    this.priorityEvents.clear();
  }

  // 获取事件监听器数量
  public listenerCount(eventName: string): number {
    let count = 0;

    if (this.events.has(eventName)) {
      count += this.events.get(eventName)!.size;
    }

    if (this.priorityEvents.has(eventName)) {
      const priorityMap = this.priorityEvents.get(eventName)!;

      for (const callbacks of priorityMap.values()) {
        count += callbacks.size;
      }
    }

    return count;
  }
}
```

### 持久化管理器

```typescript
class PersistenceManager {
  private storage: Storage;

  constructor() {
    this.storage = new Storage(); // 假设这是一个存储接口的实现
  }

  // 保存游戏状态
  public async saveGameState(gameState: GameState): Promise<void> {
    const key = `game:${gameState.gameId}`;
    const serializedState = this.serializeGameState(gameState);

    await this.storage.set(key, serializedState);
  }

  // 加载游戏状态
  public async loadGameState(gameId: string): Promise<GameState | null> {
    const key = `game:${gameId}`;
    const serializedState = await this.storage.get(key);

    if (!serializedState) {
      return null;
    }

    return this.deserializeGameState(serializedState);
  }

  // 删除游戏状态
  public async deleteGameState(gameId: string): Promise<void> {
    const key = `game:${gameId}`;
    await this.storage.delete(key);
  }

  // 序列化游戏状态
  private serializeGameState(gameState: GameState): string {
    // 处理Map等特殊数据结构的序列化
    const processedState = {
      ...gameState,
      playerRoles: Array.from(gameState.playerRoles.entries()),
      missions: gameState.missions.map((mission) => ({
        ...mission,
        executionResults: mission.executionResults
          ? Array.from(mission.executionResults.entries())
          : undefined,
        currentTeam: mission.currentTeam
          ? {
              ...mission.currentTeam,
              votes: mission.currentTeam.votes
                ? Array.from(mission.currentTeam.votes.entries())
                : undefined,
            }
          : null,
      })),
    };

    return JSON.stringify(processedState);
  }

  // 反序列化游戏状态
  private deserializeGameState(serializedState: string): GameState {
    const parsedState = JSON.parse(serializedState);

    // 处理Map等特殊数据结构的反序列化
    return {
      ...parsedState,
      playerRoles: new Map(parsedState.playerRoles),
      missions: parsedState.missions.map((mission) => ({
        ...mission,
        executionResults: mission.executionResults
          ? new Map(mission.executionResults)
          : new Map(),
        currentTeam: mission.currentTeam
          ? {
              ...mission.currentTeam,
              votes: mission.currentTeam.votes
                ? new Map(mission.currentTeam.votes)
                : new Map(),
            }
          : null,
      })),
      createdAt: new Date(parsedState.createdAt),
      updatedAt: new Date(parsedState.updatedAt),
    };
  }
}
```

## 界面原型

```
+------------------------------------------+
|           游戏状态管理控制台              |
+------------------------------------------+
|                                          |
|  游戏ID: game_12345                      |
|  当前阶段: 任务阶段                       |
|  当前任务: 第2轮任务                      |
|                                          |
|  玩家状态:                               |
|  +------+  +------+  +------+  +------+  |
|  |玩家1  |  |玩家2  |  |玩家3  |  |玩家4  |  |
|  |已连接 |  |已连接 |  |断线   |  |已连接 |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  +------+  +------+  +------+  +------+  |
|  |玩家5  |  |玩家6  |  |玩家7  |  |玩家8  |  |
|  |已连接 |  |已连接 |  |已连接 |  |已连接 |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  任务状态:                               |
|  [成功] [进行中] [ ] [ ] [ ]             |
|                                          |
|  系统事件:                               |
|  - 玩家3断线 (12:45:30)                  |
|  - 阶段转换: 投票阶段 -> 任务执行阶段 (12:44:15) |
|  - 状态更新: 队长选择完成 (12:43:02)      |
|                                          |
|  [创建快照]  [恢复状态]  [强制同步]       |
+------------------------------------------+
```

```
+------------------------------------------+
|           玩家状态同步界面                |
+------------------------------------------+
|                                          |
|  正在同步游戏状态...                      |
|                                          |
|  [进度条: 75%]                           |
|                                          |
|  同步内容:                               |
|  - 玩家信息 ✓                            |
|  - 角色分配 ✓                            |
|  - 任务状态 ✓                            |
|  - 投票记录 ⟳                            |
|  - 游戏进度 ⟳                            |
|                                          |
|  请稍候，游戏即将恢复                     |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           断线重连处理                    |
+------------------------------------------+
|                                          |
|  检测到您已断线重连                       |
|                                          |
|  正在恢复游戏状态...                      |
|                                          |
|  游戏当前处于: 任务执行阶段               |
|  当前任务: 第2轮任务                      |
|                                          |
|  您是否需要查看角色信息?                  |
|                                          |
|  [查看角色]            [继续游戏]         |
|                                          |
+------------------------------------------+
```

## 验收标准

1. 游戏状态管理器能够正确维护游戏的全局状态
2. 状态更新操作是原子的，确保状态一致性
3. 事件系统能够及时通知状态变更
4. 状态持久化和恢复机制正常工作
5. 断线重连时能够正确恢复玩家状态
6. 状态验证机制能够防止非法状态更新
7. 状态差异计算正确，支持增量更新
8. 系统能够处理并发操作和竞态条件

## 技术依赖

- TypeScript/JavaScript
- 事件发布-订阅库
- 存储系统（如 Redis、MongoDB 等）
- WebSocket 或长轮询技术（用于实时通知）
- JSON 序列化/反序列化工具

## 工作量估计

1 人天

## 相关文档

- [游戏进程控制技术方案](./技术方案.md)
- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
