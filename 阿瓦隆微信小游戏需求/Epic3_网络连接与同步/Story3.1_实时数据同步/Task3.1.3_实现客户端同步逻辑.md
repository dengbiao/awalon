# Task 3.1.3: 实现客户端同步逻辑

## 任务描述

实现阿瓦隆微信小游戏的客户端同步逻辑，负责与服务器建立和维护连接，处理接收的游戏状态更新，并在本地应用这些更新。客户端同步系统还需要实现状态预测和校正机制，确保在网络条件不佳的情况下仍能提供流畅的游戏体验。

## 功能需求

1. **网络连接管理**：

   - 建立和维护与服务器的 WebSocket 连接
   - 实现心跳检测和连接状态监控
   - 处理断线和重连逻辑

2. **消息处理**：

   - 解析和处理接收到的消息
   - 发送玩家操作和确认消息
   - 处理消息序列化和反序列化

3. **状态同步**：

   - 接收并应用完整状态快照
   - 处理增量状态更新
   - 维护本地游戏状态

4. **预测与校正**：

   - 实现客户端预测机制，预测操作的即时结果
   - 当服务器响应到达时进行状态校正
   - 平滑处理状态校正，避免视觉跳跃

5. **错误处理**：
   - 检测和处理状态不一致情况
   - 实现错误恢复机制
   - 记录和上报同步异常

## 技术需求

1. 客户端必须使用 TypeScript 开发，兼容微信小游戏环境
2. 网络连接平均延迟不影响用户体验，延迟超过 500ms 时有视觉提示
3. 状态更新的应用时间不超过 16ms（保持 60 帧的流畅度）
4. 预测与校正机制能够处理 200ms-800ms 的网络延迟
5. 内存占用合理，不发生内存泄漏
6. 支持错误恢复和异常情况处理

## 实现步骤

1. **设计客户端架构**：

   - 定义模块结构和交互方式
   - 设计状态管理模式
   - 规划错误处理策略

2. **实现网络层**：

   - 开发 WebSocket 连接管理器
   - 实现消息发送和接收机制
   - 开发心跳和重连逻辑

3. **实现状态管理**：

   - 开发本地状态存储
   - 实现状态更新机制
   - 构建状态访问接口

4. **实现预测系统**：

   - 开发操作预测机制
   - 实现状态校正逻辑
   - 开发视觉平滑过渡

5. **性能优化**：
   - 优化状态更新性能
   - 减少不必要的重渲染
   - 优化内存使用

## 示例代码

### 网络连接管理器

```typescript
// 简化的WebSocket连接管理器
class NetworkManager {
  private ws: WebSocket | null = null;
  private serverUrl: string;
  private isConnecting: boolean = false;
  private reconnectAttempts: number = 0;
  private maxReconnectAttempts: number = 5;
  private reconnectDelay: number = 1000;
  private heartbeatInterval: number = 30000;
  private heartbeatTimer: number | null = null;
  private eventEmitter = new EventEmitter();

  constructor(serverUrl: string) {
    this.serverUrl = serverUrl;
  }

  // 连接到服务器
  public connect(): void {
    if (
      this.ws &&
      (this.ws.readyState === WebSocket.OPEN ||
        this.ws.readyState === WebSocket.CONNECTING)
    ) {
      return;
    }

    if (this.isConnecting) {
      return;
    }

    this.isConnecting = true;

    try {
      this.ws = wx.connectSocket({
        url: this.serverUrl,
        header: {
          "content-type": "application/json",
        },
      });

      this.ws.onOpen(this.handleOpen.bind(this));
      this.ws.onMessage(this.handleMessage.bind(this));
      this.ws.onError(this.handleError.bind(this));
      this.ws.onClose(this.handleClose.bind(this));
    } catch (error) {
      console.error("连接服务器失败:", error);
      this.isConnecting = false;
      this.scheduleReconnect();
    }
  }

  // 断开连接
  public disconnect(): void {
    this.stopHeartbeat();

    if (this.ws) {
      try {
        this.ws.close();
      } catch (error) {
        console.error("关闭连接失败:", error);
      }
      this.ws = null;
    }
  }

  // 发送消息
  public send(message: any): boolean {
    if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
      console.warn("尝试在连接关闭时发送消息");
      return false;
    }

    try {
      const messageStr = JSON.stringify(message);
      this.ws.send({
        data: messageStr,
      });
      return true;
    } catch (error) {
      console.error("发送消息失败:", error);
      return false;
    }
  }

  // 添加事件监听器
  public on(event: string, callback: Function): void {
    this.eventEmitter.on(event, callback);
  }

  // 处理连接打开
  private handleOpen(): void {
    console.log("WebSocket连接已建立");
    this.isConnecting = false;
    this.reconnectAttempts = 0;
    this.startHeartbeat();
    this.eventEmitter.emit("connected");
  }

  // 处理收到消息
  private handleMessage(event: any): void {
    try {
      const message = JSON.parse(event.data);
      this.eventEmitter.emit("message", message);
    } catch (error) {
      console.error("解析消息失败:", error);
    }
  }

  // 处理错误
  private handleError(error: any): void {
    console.error("WebSocket错误:", error);
    this.eventEmitter.emit("error", error);
  }

  // 处理连接关闭
  private handleClose(event: any): void {
    console.log("WebSocket连接已关闭", event);
    this.isConnecting = false;
    this.stopHeartbeat();
    this.eventEmitter.emit("disconnected", event);
    this.scheduleReconnect();
  }

  // 开始心跳检测
  private startHeartbeat(): void {
    this.stopHeartbeat();
    this.heartbeatTimer = setInterval(() => {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.send({
          header: {
            type: "PING",
            timestamp: Date.now(),
          },
        });
      }
    }, this.heartbeatInterval);
  }

  // 停止心跳检测
  private stopHeartbeat(): void {
    if (this.heartbeatTimer !== null) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  // 安排重新连接
  private scheduleReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.log("已达到最大重连次数");
      this.eventEmitter.emit("reconnect_failed");
      return;
    }

    this.reconnectAttempts++;
    const delay =
      this.reconnectDelay * Math.pow(1.5, this.reconnectAttempts - 1);

    console.log(
      `计划 ${delay}ms 后重新连接，尝试次数: ${this.reconnectAttempts}`
    );

    setTimeout(() => {
      console.log("尝试重新连接...");
      this.connect();
    }, delay);
  }
}
```

### 游戏状态管理器

```typescript
// 简化的游戏状态管理器
class GameStateManager {
  private currentState: any = null;
  private lastSnapshotState: any = null;
  private stateVersion: string | null = null;
  private eventEmitter = new EventEmitter();

  // 初始化状态
  public init(initialState: any): void {
    this.currentState = initialState;
    this.lastSnapshotState = initialState;
    this.stateVersion = initialState.versionId;
    this.eventEmitter.emit("state_updated", this.currentState);
  }

  // 应用状态快照
  public applySnapshot(snapshot: any): void {
    this.lastSnapshotState = snapshot.gameState;
    this.currentState = { ...snapshot.gameState };
    this.stateVersion = snapshot.versionId;
    this.eventEmitter.emit("state_updated", this.currentState);
  }

  // 应用状态增量更新
  public applyDelta(delta: any): boolean {
    // 检查版本是否匹配
    if (delta.baseVersionId !== this.stateVersion) {
      console.error("状态版本不匹配，无法应用增量更新");
      return false;
    }

    // 复制当前状态
    const newState = JSON.parse(JSON.stringify(this.currentState));

    // 应用所有操作
    for (const operation of delta.operations) {
      this.applyOperation(newState, operation);
    }

    // 更新状态和版本
    this.currentState = newState;
    this.stateVersion = delta.resultVersionId;
    this.eventEmitter.emit("state_updated", this.currentState);
    return true;
  }

  // 获取当前状态
  public getState(): any {
    return this.currentState;
  }

  // 获取当前状态版本
  public getStateVersion(): string | null {
    return this.stateVersion;
  }

  // 监听状态更新
  public onStateUpdated(callback: (state: any) => void): void {
    this.eventEmitter.on("state_updated", callback);
  }

  // 应用单个操作到状态
  private applyOperation(state: any, operation: any): void {
    const { type, path, value } = operation;
    const pathParts = path.split(".");
    const targetPath = pathParts.slice(0, -1);
    const targetProperty = pathParts[pathParts.length - 1];

    // 找到目标对象
    let target = state;
    for (const part of targetPath) {
      if (target[part] === undefined) {
        target[part] = {};
      }
      target = target[part];
    }

    // 根据操作类型应用更改
    switch (type) {
      case "ADD":
      case "UPDATE":
        target[targetProperty] = value;
        break;
      case "REMOVE":
        delete target[targetProperty];
        break;
      case "REPLACE":
        target[targetProperty] = value;
        break;
    }
  }
}
```

### 预测与校正系统

```typescript
// 简化的预测与校正系统
class PredictionSystem {
  private gameStateManager: GameStateManager;
  private pendingActions: Map<string, any> = new Map();

  constructor(gameStateManager: GameStateManager) {
    this.gameStateManager = gameStateManager;
  }

  // 添加预测动作
  public addPrediction(action: any, predictedState: any): void {
    const actionId = action.header.messageId;
    this.pendingActions.set(actionId, {
      action,
      predictedState,
      timestamp: Date.now(),
    });

    // 应用预测状态到游戏状态
    this.gameStateManager.applyPredictedState(predictedState);
  }

  // 处理服务器确认
  public handleServerConfirmation(result: any): void {
    const actionId = result.payload.actionId;
    const pendingAction = this.pendingActions.get(actionId);

    if (!pendingAction) {
      console.warn("收到未知动作的确认:", actionId);
      return;
    }

    // 移除待确认的动作
    this.pendingActions.delete(actionId);

    // 如果动作成功，不需要特殊处理，状态更新会通过增量或快照更新
    if (result.payload.success) {
      return;
    }

    // 如果动作失败，需要回滚预测
    this.rollbackPrediction(actionId);
  }

  // 回滚预测
  private rollbackPrediction(actionId: string): void {
    // 清理所有待确认的动作，因为可能有依赖关系
    this.pendingActions.clear();

    // 请求完整状态快照以重置状态
    // 实际实现会触发一个事件，由网络管理器处理
    this.requestStateSnapshot();
  }

  // 请求状态快照
  private requestStateSnapshot(): void {
    // 发出请求快照的事件
    eventEmitter.emit("request_state_snapshot");
  }
}
```

### 同步管理器

```typescript
// 简化的同步管理器，集成网络、状态和预测系统
class SynchronizationManager {
  private networkManager: NetworkManager;
  private gameStateManager: GameStateManager;
  private predictionSystem: PredictionSystem;
  private userId: string;
  private roomId: string;
  private isConnected: boolean = false;
  private eventEmitter = new EventEmitter();

  constructor(serverUrl: string, userId: string, roomId: string) {
    this.userId = userId;
    this.roomId = roomId;

    // 初始化组件
    this.networkManager = new NetworkManager(serverUrl);
    this.gameStateManager = new GameStateManager();
    this.predictionSystem = new PredictionSystem(this.gameStateManager);

    // 设置事件监听
    this.setupEventListeners();
  }

  // 设置事件监听器
  private setupEventListeners(): void {
    // 网络事件
    this.networkManager.on("connected", this.handleConnected.bind(this));
    this.networkManager.on("disconnected", this.handleDisconnected.bind(this));
    this.networkManager.on("message", this.handleMessage.bind(this));
    this.networkManager.on("error", this.handleNetworkError.bind(this));

    // 预测系统事件
    this.predictionSystem.on(
      "request_state_snapshot",
      this.requestStateSnapshot.bind(this)
    );
  }

  // 启动同步
  public start(): void {
    this.networkManager.connect();
  }

  // 停止同步
  public stop(): void {
    this.networkManager.disconnect();
  }

  // 发送玩家动作
  public sendPlayerAction(actionType: string, actionData: any): void {
    if (!this.isConnected) {
      console.warn("未连接服务器，无法发送动作");
      return;
    }

    // 创建动作消息
    const action = {
      header: {
        type: "PLAYER_ACTION",
        messageId: `action_${Date.now()}_${Math.floor(Math.random() * 1000)}`,
        timestamp: Date.now(),
        roomId: this.roomId,
      },
      payload: {
        playerId: this.userId,
        actionType,
        actionData,
        clientTimestamp: Date.now(),
      },
    };

    // 生成预测状态
    const currentState = this.gameStateManager.getState();
    const predictedState = this.generatePredictedState(currentState, action);

    // 添加预测
    this.predictionSystem.addPrediction(action, predictedState);

    // 发送动作
    this.networkManager.send(action);
  }

  // 监听状态更新
  public onStateUpdated(callback: (state: any) => void): void {
    this.gameStateManager.onStateUpdated(callback);
  }

  // 处理连接成功
  private handleConnected(): void {
    console.log("已连接到游戏服务器");
    this.isConnected = true;

    // 发送加入房间消息
    this.sendJoinRoom();

    this.eventEmitter.emit("sync_connected");
  }

  // 处理连接断开
  private handleDisconnected(): void {
    console.log("与游戏服务器的连接已断开");
    this.isConnected = false;
    this.eventEmitter.emit("sync_disconnected");
  }

  // 处理收到消息
  private handleMessage(message: any): void {
    const messageType = message.header.type;

    switch (messageType) {
      case "STATE_SNAPSHOT":
        this.handleStateSnapshot(message.payload);
        break;

      case "STATE_DELTA":
        this.handleStateDelta(message.payload);
        break;

      case "ACTION_RESULT":
        this.handleActionResult(message);
        break;

      case "PONG":
        // 心跳响应，忽略
        break;

      case "ERROR":
        this.handleError(message.payload);
        break;

      default:
        console.warn("收到未知类型的消息:", messageType);
        break;
    }
  }

  // 处理网络错误
  private handleNetworkError(error: any): void {
    console.error("网络错误:", error);
    this.eventEmitter.emit("sync_error", error);
  }

  // 处理状态快照
  private handleStateSnapshot(payload: any): void {
    console.log("收到状态快照，版本:", payload.versionId);
    this.gameStateManager.applySnapshot(payload);
  }

  // 处理状态增量
  private handleStateDelta(payload: any): void {
    console.log(
      "收到状态增量，基础版本:",
      payload.baseVersionId,
      "结果版本:",
      payload.resultVersionId
    );
    const success = this.gameStateManager.applyDelta(payload);

    if (!success) {
      // 增量应用失败，请求完整快照
      this.requestStateSnapshot();
    }
  }

  // 处理动作结果
  private handleActionResult(message: any): void {
    console.log(
      "收到动作结果:",
      message.payload.actionId,
      message.payload.success
    );
    this.predictionSystem.handleServerConfirmation(message);
  }

  // 处理错误消息
  private handleError(payload: any): void {
    console.error("服务器错误:", payload.errorCode, payload.message);
    this.eventEmitter.emit("server_error", payload);
  }

  // 发送加入房间消息
  private sendJoinRoom(): void {
    const joinMessage = {
      header: {
        type: "JOIN_ROOM",
        messageId: `join_${Date.now()}`,
        timestamp: Date.now(),
        roomId: this.roomId,
      },
      payload: {
        playerId: this.userId,
        roomId: this.roomId,
      },
    };

    this.networkManager.send(joinMessage);
  }

  // 请求状态快照
  private requestStateSnapshot(): void {
    const requestMessage = {
      header: {
        type: "REQUEST_SNAPSHOT",
        messageId: `req_snapshot_${Date.now()}`,
        timestamp: Date.now(),
        roomId: this.roomId,
      },
      payload: {
        playerId: this.userId,
        currentVersion: this.gameStateManager.getStateVersion(),
      },
    };

    this.networkManager.send(requestMessage);
  }

  // 生成预测状态（简化版）
  private generatePredictedState(currentState: any, action: any): any {
    // 实际实现会根据游戏规则进行复杂的预测
    // 这里使用简化版本，只进行基本修改

    const state = JSON.parse(JSON.stringify(currentState));
    const actionType = action.payload.actionType;
    const actionData = action.payload.actionData;

    switch (actionType) {
      case "SELECT_TEAM":
        state.currentTeam = actionData.team;
        state.phase = "TEAM_VOTING";
        break;

      case "VOTE_FOR_TEAM":
        // 添加新投票
        if (!state.votes) {
          state.votes = [];
        }
        state.votes.push({
          playerId: this.userId,
          vote: actionData.vote,
          timestamp: Date.now(),
        });
        break;

      case "VOTE_FOR_MISSION":
        // 添加任务投票
        if (!state.missionVotes) {
          state.missionVotes = [];
        }
        state.missionVotes.push({
          playerId: this.userId,
          vote: actionData.vote,
          timestamp: Date.now(),
        });
        break;
    }

    return state;
  }
}
```

### 使用示例

```typescript
// 在游戏场景中使用同步管理器
class GameScene {
  private synchronizationManager: SynchronizationManager;
  private gameState: any = null;

  constructor() {
    // 从游戏配置或登录信息获取
    const serverUrl = "wss://game-server.example.com/avalon";
    const userId = "user123";
    const roomId = "room456";

    // 创建同步管理器
    this.synchronizationManager = new SynchronizationManager(
      serverUrl,
      userId,
      roomId
    );

    // 监听状态更新
    this.synchronizationManager.onStateUpdated(
      this.handleStateUpdated.bind(this)
    );

    // 启动同步
    this.synchronizationManager.start();
  }

  // 处理状态更新
  private handleStateUpdated(state: any): void {
    this.gameState = state;

    // 根据新状态更新UI
    this.updateUI();
  }

  // 更新UI
  private updateUI(): void {
    // 根据游戏状态更新UI组件
    if (!this.gameState) {
      return;
    }

    // 更新玩家列表
    this.updatePlayerList(this.gameState.players);

    // 更新游戏阶段
    this.updateGamePhase(this.gameState.phase);

    // 更新任务状态
    this.updateMissions(this.gameState.missions);

    // 根据当前游戏阶段显示/隐藏相关UI
    switch (this.gameState.phase) {
      case "TEAM_SELECTION":
        this.showTeamSelectionUI();
        break;

      case "TEAM_VOTING":
        this.showTeamVotingUI();
        break;

      case "MISSION_VOTING":
        this.showMissionVotingUI();
        break;

      case "ASSASSINATION":
        this.showAssassinationUI();
        break;

      case "GOOD_WINS":
      case "EVIL_WINS":
        this.showGameResultUI();
        break;
    }
  }

  // 用户操作：选择团队成员
  public onTeamSelected(selectedPlayers: string[]): void {
    this.synchronizationManager.sendPlayerAction("SELECT_TEAM", {
      team: selectedPlayers,
    });
  }

  // 用户操作：投票
  public onVoteForTeam(approve: boolean): void {
    this.synchronizationManager.sendPlayerAction("VOTE_FOR_TEAM", {
      vote: approve ? "APPROVE" : "REJECT",
    });
  }

  // 用户操作：任务投票
  public onVoteForMission(success: boolean): void {
    this.synchronizationManager.sendPlayerAction("VOTE_FOR_MISSION", {
      vote: success ? "SUCCESS" : "FAIL",
    });
  }

  // 清理
  public destroy(): void {
    this.synchronizationManager.stop();
  }

  // UI更新方法（示例）
  private updatePlayerList(players: any[]): void {
    // 更新玩家列表UI
    console.log("更新玩家列表:", players);
  }

  private updateGamePhase(phase: string): void {
    // 更新游戏阶段UI
    console.log("更新游戏阶段:", phase);
  }

  private updateMissions(missions: any[]): void {
    // 更新任务状态UI
    console.log("更新任务状态:", missions);
  }

  private showTeamSelectionUI(): void {
    // 显示队伍选择UI
    console.log("显示队伍选择UI");
  }

  private showTeamVotingUI(): void {
    // 显示队伍投票UI
    console.log("显示队伍投票UI");
  }

  private showMissionVotingUI(): void {
    // 显示任务投票UI
    console.log("显示任务投票UI");
  }

  private showAssassinationUI(): void {
    // 显示刺杀UI
    console.log("显示刺杀UI");
  }

  private showGameResultUI(): void {
    // 显示游戏结果UI
    console.log("显示游戏结果UI");
  }
}
```

## 验收标准

1. 客户端能够成功连接到服务器，并处理网络连接的建立、断开和重连
2. 客户端能够接收和应用状态快照和增量更新，保持本地状态与服务器状态一致
3. 通过预测与校正机制，玩家操作在网络延迟条件下有即时响应
4. 在网络延迟 200ms-800ms 的情况下，游戏操作流畅，无明显卡顿
5. 在网络波动或断线情况下，客户端能够适当处理，显示视觉提示并尝试恢复
6. 在极端情况下（如状态不一致），客户端能够检测并自动恢复
7. 内存使用稳定，长时间运行不出现内存泄漏
8. 所有玩家操作都能正确发送到服务器并得到处理

## 技术依赖

1. 微信小游戏 API（WebSocket、本地存储等）
2. TypeScript 开发环境
3. 事件发射器（EventEmitter）
4. JSON 序列化/反序列化工具
5. 状态管理库或自定义状态管理工具
6. 性能监控工具

## 工作量估计

预计工作量：1 人/天
