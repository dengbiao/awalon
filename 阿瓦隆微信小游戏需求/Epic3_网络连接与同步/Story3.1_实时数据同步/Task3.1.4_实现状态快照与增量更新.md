# Task 3.1.4: 实现状态快照与增量更新

## 任务描述

设计并实现阿瓦隆微信小游戏的状态快照和增量更新机制，优化网络传输效率，并确保游戏状态在各客户端之间保持一致。该机制将负责创建完整状态快照、计算状态变化增量、应用增量更新，以及处理状态一致性校验和恢复。

## 功能需求

1. **状态快照管理**：

   - 定期创建完整游戏状态快照
   - 在关键游戏阶段转换点生成快照
   - 管理快照版本和历史记录

2. **增量更新计算**：

   - 计算两个状态版本之间的差异
   - 生成精简的增量更新数据
   - 优化增量表示，减小数据体积

3. **数据传输优化**：

   - 实现数据压缩，减少传输量
   - 根据网络条件调整数据发送策略
   - 支持增量或完整快照的智能选择

4. **状态一致性验证**：
   - 验证客户端状态与服务器状态的一致性
   - 检测状态偏差并触发修复机制
   - 跟踪和记录状态同步情况

## 技术需求

1. 增量更新计算时间不超过 10ms
2. 完整状态快照数据大小不超过 50KB
3. 增量更新数据大小平均减少 70%以上（相对于完整快照）
4. 支持至少 20 个版本的状态历史记录
5. 状态恢复时间不超过 500ms
6. 内存占用合理，不发生内存泄漏

## 实现步骤

1. **设计状态版本管理**：

   - 定义版本标识机制
   - 设计快照存储结构
   - 规划快照轮换策略

2. **实现增量计算算法**：

   - 开发状态差异比较算法
   - 设计增量表示格式
   - 优化增量计算性能

3. **实现数据传输优化**：

   - 集成数据压缩功能
   - 开发传输策略选择逻辑
   - 实现自适应传输控制

4. **实现状态一致性机制**：

   - 开发状态校验算法
   - 实现状态恢复流程
   - 添加状态监控和日志

5. **测试和优化**：
   - 验证各种游戏场景下的状态同步
   - 测试网络条件对同步的影响
   - 优化性能瓶颈

## 示例代码

### 状态快照管理

```typescript
// 简化的状态快照管理器
class SnapshotManager {
  private snapshots: Map<string, any> = new Map(); // 版本ID -> 状态快照
  private maxSnapshots: number = 20; // 最大保留快照数
  private currentVersionId: string | null = null;

  // 创建新快照
  public createSnapshot(gameState: any): string {
    // 生成版本ID
    const versionId = this.generateVersionId();

    // 克隆状态以确保快照独立
    const snapshot = this.cloneState(gameState);

    // 添加元数据
    snapshot._metadata = {
      versionId,
      timestamp: Date.now(),
      snapshotType: "full",
    };

    // 保存快照
    this.addSnapshot(versionId, snapshot);
    this.currentVersionId = versionId;

    return versionId;
  }

  // 获取快照
  public getSnapshot(versionId: string): any | null {
    return this.snapshots.get(versionId) || null;
  }

  // 获取当前版本ID
  public getCurrentVersionId(): string | null {
    return this.currentVersionId;
  }

  // 添加快照并管理历史记录大小
  private addSnapshot(versionId: string, snapshot: any): void {
    // 添加新快照
    this.snapshots.set(versionId, snapshot);

    // 如果快照数量超过限制，移除最旧的
    if (this.snapshots.size > this.maxSnapshots) {
      const oldestKey = this.getOldestSnapshotKey();
      if (oldestKey) {
        this.snapshots.delete(oldestKey);
      }
    }
  }

  // 获取最旧的快照键
  private getOldestSnapshotKey(): string | null {
    if (this.snapshots.size === 0) {
      return null;
    }

    // 找出时间戳最小的快照
    let oldestKey: string | null = null;
    let oldestTime = Infinity;

    for (const [key, snapshot] of this.snapshots.entries()) {
      const timestamp = snapshot._metadata?.timestamp || 0;
      if (timestamp < oldestTime) {
        oldestTime = timestamp;
        oldestKey = key;
      }
    }

    return oldestKey;
  }

  // 生成版本ID
  private generateVersionId(): string {
    return `v_${Date.now()}_${Math.floor(Math.random() * 10000)}`;
  }

  // 克隆状态对象
  private cloneState(state: any): any {
    return JSON.parse(JSON.stringify(state));
  }
}
```

### 增量更新计算

```typescript
// 简化的增量更新计算器
class DeltaCalculator {
  // 计算两个状态之间的差异
  public calculateDelta(baseState: any, currentState: any): any {
    // 开始计时（用于性能监控）
    const startTime = Date.now();

    // 初始化增量对象
    const delta = {
      baseVersionId: baseState._metadata?.versionId,
      resultVersionId: currentState._metadata?.versionId,
      operations: [] as Array<any>,
      timestamp: Date.now(),
    };

    // 使用路径比较计算差异
    this.compareObjects(baseState, currentState, "", delta.operations);

    // 计算处理时间
    const processingTime = Date.now() - startTime;
    console.log(
      `增量计算耗时: ${processingTime}ms，操作数: ${delta.operations.length}`
    );

    return delta;
  }

  // 递归比较对象差异
  private compareObjects(
    base: any,
    current: any,
    path: string,
    operations: Array<any>
  ): void {
    // 如果类型不同，直接替换
    if (typeof base !== typeof current) {
      operations.push({
        type: "REPLACE",
        path,
        value: current,
      });
      return;
    }

    // 跳过元数据比较
    if (path === "_metadata") {
      return;
    }

    // 如果是数组，采用简化处理
    if (Array.isArray(base) && Array.isArray(current)) {
      // 如果数组长度或内容有变化，整体替换
      // 注意：实际生产环境应该使用更复杂的数组差异算法
      if (JSON.stringify(base) !== JSON.stringify(current)) {
        operations.push({
          type: "REPLACE",
          path,
          value: current,
        });
      }
      return;
    }

    // 如果是对象，递归比较各属性
    if (
      typeof base === "object" &&
      base !== null &&
      typeof current === "object" &&
      current !== null
    ) {
      // 检查删除的属性
      for (const key in base) {
        if (!Object.prototype.hasOwnProperty.call(current, key)) {
          const currentPath = path ? `${path}.${key}` : key;
          operations.push({
            type: "REMOVE",
            path: currentPath,
          });
        }
      }

      // 检查新增或修改的属性
      for (const key in current) {
        if (Object.prototype.hasOwnProperty.call(current, key)) {
          const currentPath = path ? `${path}.${key}` : key;

          if (!Object.prototype.hasOwnProperty.call(base, key)) {
            // 新增属性
            operations.push({
              type: "ADD",
              path: currentPath,
              value: current[key],
            });
          } else if (
            JSON.stringify(base[key]) !== JSON.stringify(current[key])
          ) {
            // 修改的属性，递归比较
            if (
              typeof base[key] === "object" &&
              base[key] !== null &&
              typeof current[key] === "object" &&
              current[key] !== null
            ) {
              this.compareObjects(
                base[key],
                current[key],
                currentPath,
                operations
              );
            } else {
              operations.push({
                type: "UPDATE",
                path: currentPath,
                value: current[key],
              });
            }
          }
        }
      }
      return;
    }

    // 基本类型值比较
    if (base !== current) {
      operations.push({
        type: "UPDATE",
        path,
        value: current,
      });
    }
  }
}
```

### 状态应用器

```typescript
// 简化的状态应用器
class StateApplier {
  // 应用增量更新到状态
  public applyDelta(baseState: any, delta: any): any {
    // 验证版本
    if (baseState._metadata?.versionId !== delta.baseVersionId) {
      throw new Error("状态版本不匹配，无法应用增量更新");
    }

    // 克隆基础状态
    const newState = this.cloneState(baseState);

    // 应用所有操作
    for (const operation of delta.operations) {
      this.applyOperation(newState, operation);
    }

    // 更新元数据
    newState._metadata = {
      ...(newState._metadata || {}),
      versionId: delta.resultVersionId,
      lastUpdated: Date.now(),
    };

    return newState;
  }

  // 应用单个操作
  private applyOperation(state: any, operation: any): void {
    const { type, path, value } = operation;

    // 空路径表示整个状态
    if (!path) {
      // 只处理REPLACE类型，因为其他类型对根对象无意义
      if (type === "REPLACE") {
        Object.keys(state).forEach((key) => {
          if (key !== "_metadata") {
            delete state[key];
          }
        });

        Object.keys(value).forEach((key) => {
          if (key !== "_metadata") {
            state[key] = value[key];
          }
        });
      }
      return;
    }

    // 解析路径
    const pathParts = path.split(".");
    const targetProperty = pathParts.pop() as string;

    // 找到目标对象
    let target = state;
    for (const part of pathParts) {
      if (target[part] === undefined) {
        // 如果路径不存在，根据操作类型可能需要创建
        if (type === "ADD" || type === "UPDATE" || type === "REPLACE") {
          target[part] = {};
        } else {
          // 对于REMOVE操作，如果路径不存在则忽略
          return;
        }
      }
      target = target[part];
    }

    // 根据操作类型应用更改
    switch (type) {
      case "ADD":
      case "UPDATE":
      case "REPLACE":
        target[targetProperty] = value;
        break;

      case "REMOVE":
        delete target[targetProperty];
        break;
    }
  }

  // 克隆状态对象
  private cloneState(state: any): any {
    return JSON.parse(JSON.stringify(state));
  }
}
```

### 压缩与传输优化

```typescript
// 简化的压缩与传输优化器
class TransportOptimizer {
  // 压缩数据
  public static compressData(data: any): Uint8Array {
    // 在实际项目中，这里会使用实际的压缩库，如pako
    // 这里是简化的模拟实现
    console.log("模拟压缩数据...");
    const jsonStr = JSON.stringify(data);

    // 假设压缩率为50%
    const originalSize = jsonStr.length;
    const compressedSize = Math.floor(originalSize * 0.5);

    console.log(`数据大小: ${originalSize} -> ${compressedSize} (模拟压缩)`);

    // 返回一个假的Uint8Array，实际项目会返回真实压缩后的数据
    return new Uint8Array(compressedSize);
  }

  // 解压数据
  public static decompressData(
    compressedData: Uint8Array,
    originalData: any
  ): any {
    // 在实际项目中，这里会使用实际的解压缩逻辑
    // 简化实现，直接返回原始数据
    console.log("模拟解压数据...");
    return originalData;
  }

  // 根据网络条件和数据大小选择最佳传输方式
  public static selectTransportStrategy(
    snapshot: any,
    delta: any | null,
    networkQuality: "good" | "medium" | "poor"
  ): { type: "snapshot" | "delta"; data: any } {
    // 如果没有增量或网络状况非常好，发送完整快照
    if (!delta || networkQuality === "good") {
      return {
        type: "snapshot",
        data: snapshot,
      };
    }

    // 计算数据大小（简化）
    const snapshotSize = JSON.stringify(snapshot).length;
    const deltaSize = JSON.stringify(delta).length;

    // 如果增量小于快照的30%，发送增量，否则发送快照
    if (deltaSize < snapshotSize * 0.3 || networkQuality === "poor") {
      return {
        type: "delta",
        data: delta,
      };
    } else {
      return {
        type: "snapshot",
        data: snapshot,
      };
    }
  }
}
```

### 状态一致性验证

```typescript
// 简化的状态一致性验证器
class ConsistencyValidator {
  // 验证状态一致性
  public static validateConsistency(
    clientState: any,
    serverState: any
  ): boolean {
    // 比较关键状态字段（简化）
    // 实际项目会根据游戏规则选择关键字段
    const keyFields = ["phase", "players.length", "currentLeader", "missions"];

    for (const field of keyFields) {
      const clientValue = this.getNestedValue(clientState, field);
      const serverValue = this.getNestedValue(serverState, field);

      if (JSON.stringify(clientValue) !== JSON.stringify(serverValue)) {
        console.warn(
          `状态不一致: ${field}, 客户端=${clientValue}, 服务器=${serverValue}`
        );
        return false;
      }
    }

    return true;
  }

  // 获取嵌套对象属性值
  private static getNestedValue(obj: any, path: string): any {
    const parts = path.split(".");
    let current = obj;

    for (const part of parts) {
      if (current === undefined || current === null) {
        return undefined;
      }

      if (part.endsWith("length")) {
        const arrayName = part.slice(0, -7);
        if (Array.isArray(current[arrayName])) {
          return current[arrayName].length;
        }
        return undefined;
      }

      current = current[part];
    }

    return current;
  }

  // 计算状态的哈希值（简化）
  public static calculateStateHash(state: any): string {
    // 实际项目会使用真正的哈希算法
    // 这里使用JSON字符串截取模拟
    const json = JSON.stringify(state);
    return `hash_${json.length}_${Date.now() % 10000}`;
  }
}
```

### 集成示例

```typescript
// 状态同步服务示例
class StateSyncService {
  private snapshotManager: SnapshotManager;
  private deltaCalculator: DeltaCalculator;
  private stateApplier: StateApplier;
  private currentState: any;
  private lastBroadcastTime: number = 0;
  private broadcastInterval: number = 5000; // 5秒
  private snapshotInterval: number = 30000; // 30秒
  private lastSnapshotTime: number = 0;

  constructor() {
    this.snapshotManager = new SnapshotManager();
    this.deltaCalculator = new DeltaCalculator();
    this.stateApplier = new StateApplier();

    // 初始化状态
    this.currentState = {
      phase: "WAITING_FOR_PLAYERS",
      players: [],
      missions: [],
      _metadata: {
        versionId: "initial_state",
        timestamp: Date.now(),
      },
    };

    // 创建初始快照
    this.createAndStoreSnapshot();
  }

  // 更新状态
  public updateState(newState: any): void {
    // 更新当前状态
    this.currentState = newState;

    // 检查是否需要广播
    const now = Date.now();
    if (now - this.lastBroadcastTime >= this.broadcastInterval) {
      this.broadcastStateUpdates();
    }

    // 检查是否需要创建快照
    if (now - this.lastSnapshotTime >= this.snapshotInterval) {
      this.createAndStoreSnapshot();
    }
  }

  // 创建并存储快照
  private createAndStoreSnapshot(): void {
    const versionId = this.snapshotManager.createSnapshot(this.currentState);
    this.lastSnapshotTime = Date.now();
    console.log(`创建快照: ${versionId}`);

    // 更新当前状态的版本ID
    this.currentState._metadata = {
      ...this.currentState._metadata,
      versionId,
    };
  }

  // 广播状态更新
  private broadcastStateUpdates(): void {
    // 获取最新快照
    const latestVersionId = this.snapshotManager.getCurrentVersionId();
    if (!latestVersionId) {
      console.warn("没有可用的基础状态快照");
      return;
    }

    const baseSnapshot = this.snapshotManager.getSnapshot(latestVersionId);
    if (!baseSnapshot) {
      console.warn(`无法获取基础快照: ${latestVersionId}`);
      return;
    }

    // 计算状态增量
    const delta = this.deltaCalculator.calculateDelta(
      baseSnapshot,
      this.currentState
    );

    // 获取网络状况（简化）
    const networkQuality = this.getNetworkQuality();

    // 根据网络状况选择传输策略
    const transportStrategy = TransportOptimizer.selectTransportStrategy(
      this.currentState,
      delta,
      networkQuality
    );

    // 发送数据
    if (transportStrategy.type === "snapshot") {
      this.sendStateSnapshot(transportStrategy.data);
    } else {
      this.sendStateDelta(transportStrategy.data);
    }

    this.lastBroadcastTime = Date.now();
  }

  // 发送状态快照
  private sendStateSnapshot(snapshot: any): void {
    // 压缩数据
    const compressedData = TransportOptimizer.compressData(snapshot);

    // 实际项目中会通过WebSocket发送
    console.log(
      `发送完整状态快照, 版本: ${snapshot._metadata?.versionId}, 大小: ${compressedData.length} 字节`
    );
  }

  // 发送状态增量
  private sendStateDelta(delta: any): void {
    // 压缩数据
    const compressedData = TransportOptimizer.compressData(delta);

    // 实际项目中会通过WebSocket发送
    console.log(
      `发送状态增量, 基础版本: ${delta.baseVersionId}, 结果版本: ${delta.resultVersionId}, 大小: ${compressedData.length} 字节`
    );
  }

  // 获取网络质量（简化）
  private getNetworkQuality(): "good" | "medium" | "poor" {
    // 实际项目会根据网络测量结果返回
    const qualities = ["good", "medium", "poor"] as const;
    const randomIndex = Math.floor(Math.random() * qualities.length);
    return qualities[randomIndex];
  }

  // 接收方：处理状态更新（客户端或服务器）
  public processStateUpdate(data: any, type: "snapshot" | "delta"): void {
    if (type === "snapshot") {
      // 直接应用快照
      this.currentState = data;
      console.log(`应用状态快照, 版本: ${data._metadata?.versionId}`);
    } else {
      // 应用增量
      try {
        const newState = this.stateApplier.applyDelta(this.currentState, data);

        // 验证状态一致性
        const isConsistent = ConsistencyValidator.validateConsistency(
          newState,
          this.snapshotManager.getSnapshot(data.resultVersionId) || {}
        );

        if (isConsistent) {
          this.currentState = newState;
          console.log(`应用状态增量, 结果版本: ${data.resultVersionId}`);
        } else {
          console.warn("状态不一致，请求完整快照");
          this.requestStateSnapshot();
        }
      } catch (error) {
        console.error("应用增量失败:", error);
        this.requestStateSnapshot();
      }
    }
  }

  // 请求状态快照
  private requestStateSnapshot(): void {
    // 实际项目中会发送请求到服务器
    console.log("发送状态快照请求");
  }
}
```

## 验收标准

1. 增量更新平均体积比完整状态小 70%以上
2. 增量计算时间不超过 10ms
3. 增量应用时间不超过 5ms
4. 定期快照和增量更新机制正常工作
5. 在模拟的网络波动条件下，状态能正确同步
6. 当检测到状态不一致时，能自动恢复
7. 内存使用稳定，不出现内存泄漏
8. 通过高负载测试，每秒能处理至少 100 次状态更新

## 技术依赖

1. JSON 处理库
2. 数据压缩库（如 pako 或 zlib）
3. 状态差异计算工具
4. WebSocket 连接库
5. 性能监测工具

## 工作量估计

预计工作量：1 人/天
