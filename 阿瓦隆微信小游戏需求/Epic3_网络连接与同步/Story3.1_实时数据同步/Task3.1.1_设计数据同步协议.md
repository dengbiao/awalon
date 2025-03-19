# Task 3.1.1: 设计数据同步协议

## 任务描述

设计一套高效、可靠的数据同步协议，用于阿瓦隆微信小游戏中客户端与服务器之间的实时通信。该协议将定义消息格式、通信规则、序列化方式和数据压缩策略，确保游戏状态在所有玩家之间保持一致，同时最小化网络开销和延迟。

## 功能需求

1. **消息类型定义**：

   - 定义完整的消息类型集合，包括握手、心跳、状态快照、状态增量、玩家动作、动作结果等
   - 为每种消息类型设计适当的数据结构和字段

2. **序列化格式**：

   - 选择并实现高效的序列化方案（如 MessagePack、Protobuf 等）
   - 确保序列化和反序列化的性能满足实时游戏需求

3. **数据压缩**：

   - 设计增量更新机制，仅传输变更的游戏状态
   - 实现数据压缩策略，减少传输数据量

4. **可靠性机制**：

   - 设计消息确认和重传机制
   - 实现消息排序和去重策略
   - 定义错误处理流程

5. **安全策略**：
   - 设计消息验证机制，防止非法消息注入
   - 定义数据加密要求

## 技术需求

1. 协议必须支持 WebSocket 作为主要传输层
2. 最大消息大小不超过 100KB
3. 序列化/反序列化操作在客户端执行时间不超过 10ms
4. 协议必须向后兼容，支持客户端版本升级
5. 协议规范必须使用 TypeScript 接口进行定义
6. 协议必须能够处理网络波动和临时断连情况

## 实现步骤

1. **研究和选型**：

   - 调研适合微信小游戏环境的序列化库
   - 评估不同压缩算法的效率和兼容性
   - 确定最合适的消息格式和通信模式

2. **协议设计**：

   - 设计消息头部结构
   - 定义各类消息的具体格式
   - 制定消息处理流程和规则

3. **接口定义**：

   - 使用 TypeScript 定义所有消息类型的接口
   - 定义序列化/反序列化 API
   - 设计消息处理器接口

4. **验证与测试**：
   - 实现协议验证工具
   - 编写单元测试验证协议功能
   - 执行性能测试评估协议效率

## 示例代码

### 消息类型定义

```typescript
// 消息类型枚举
enum MessageType {
  HANDSHAKE = 0,
  HANDSHAKE_ACK = 1,
  PING = 2,
  PONG = 3,
  STATE_SNAPSHOT = 4,
  STATE_DELTA = 5,
  PLAYER_ACTION = 6,
  ACTION_RESULT = 7,
  ERROR = 8,
  SYSTEM_NOTIFICATION = 9,
}

// 基础消息接口
interface Message {
  // 消息头
  header: {
    messageId: string; // 唯一消息ID
    type: MessageType; // 消息类型
    timestamp: number; // 发送时间戳
    roomId: string; // 游戏房间ID
    sequenceNumber: number; // 序列号，用于排序和去重
    compression: boolean; // 是否启用压缩
  };
  // 消息体（根据消息类型不同而不同）
  payload: any;
}

// 状态快照消息
interface StateSnapshotMessage extends Message {
  header: {
    type: MessageType.STATE_SNAPSHOT;
    // ...其他头部字段
  };
  payload: {
    gameState: GameState; // 完整游戏状态
    versionId: string; // 状态版本ID
  };
}

// 状态增量消息
interface StateDeltaMessage extends Message {
  header: {
    type: MessageType.STATE_DELTA;
    // ...其他头部字段
  };
  payload: {
    baseVersionId: string; // 基础状态版本ID
    operations: Operation[]; // 状态操作列表
    resultVersionId: string; // 结果状态版本ID
  };
}

// 玩家动作消息
interface PlayerActionMessage extends Message {
  header: {
    type: MessageType.PLAYER_ACTION;
    // ...其他头部字段
  };
  payload: {
    playerId: string; // 玩家ID
    actionType: string; // 动作类型
    actionData: any; // 动作数据
    clientTimestamp: number; // 客户端时间戳
  };
}

// 动作结果消息
interface ActionResultMessage extends Message {
  header: {
    type: MessageType.ACTION_RESULT;
    // ...其他头部字段
  };
  payload: {
    actionId: string; // 对应的动作ID
    success: boolean; // 是否成功
    resultData?: any; // 结果数据
    errorCode?: number; // 错误代码（如果失败）
    errorMessage?: string; // 错误消息（如果失败）
  };
}
```

### 序列化和压缩实现

```typescript
// 消息序列化服务
class MessageSerializer {
  // 序列化消息
  public serialize(message: Message): Uint8Array {
    // 1. 转换为JSON
    const jsonStr = JSON.stringify(message);

    // 2. 根据消息头决定是否压缩
    if (message.header.compression) {
      return this.compress(jsonStr);
    } else {
      // 直接使用MessagePack序列化
      return msgpack.encode(jsonStr);
    }
  }

  // 反序列化消息
  public deserialize(data: Uint8Array): Message {
    // 1. 先解析消息头以确定是否需要解压
    const headerBytes = data.slice(0, HEADER_SIZE);
    const header = msgpack.decode(headerBytes);

    // 2. 根据头部信息决定是否需要解压
    let jsonStr: string;
    if (header.compression) {
      jsonStr = this.decompress(data.slice(HEADER_SIZE));
    } else {
      jsonStr = msgpack.decode(data.slice(HEADER_SIZE));
    }

    // 3. 解析JSON并返回完整消息
    const fullMessage = JSON.parse(jsonStr);
    fullMessage.header = header;
    return fullMessage;
  }

  // 压缩数据
  private compress(jsonStr: string): Uint8Array {
    // 使用适合微信小游戏的压缩库，如pako
    return pako.deflate(jsonStr);
  }

  // 解压数据
  private decompress(data: Uint8Array): string {
    return pako.inflate(data, { to: "string" });
  }
}
```

### 增量更新操作定义

```typescript
// 操作类型
enum OperationType {
  ADD = "add",
  UPDATE = "update",
  REMOVE = "remove",
  REPLACE = "replace",
}

// 操作定义
interface Operation {
  // 操作类型
  type: OperationType;

  // 目标路径（使用点分隔符表示嵌套属性）
  path: string;

  // 操作值（根据操作类型不同而不同）
  value?: any;
}

// 生成增量更新
function generateDelta(
  previousState: GameState,
  currentState: GameState
): Operation[] {
  const operations: Operation[] = [];

  // 递归比较对象差异并生成操作
  function compareObjects(prev: any, curr: any, path: string = "") {
    // 如果类型不同，直接替换
    if (typeof prev !== typeof curr) {
      operations.push({
        type: OperationType.REPLACE,
        path: path,
        value: curr,
      });
      return;
    }

    // 如果是数组类型
    if (Array.isArray(prev) && Array.isArray(curr)) {
      // 简化处理：如果数组不同，直接替换
      // 注意：实际实现可能会使用更复杂的数组差异算法
      if (JSON.stringify(prev) !== JSON.stringify(curr)) {
        operations.push({
          type: OperationType.REPLACE,
          path: path,
          value: curr,
        });
      }
      return;
    }

    // 如果是对象类型，递归比较属性
    if (
      typeof prev === "object" &&
      prev !== null &&
      typeof curr === "object" &&
      curr !== null
    ) {
      // 检查移除的属性
      Object.keys(prev).forEach((key) => {
        if (!(key in curr)) {
          operations.push({
            type: OperationType.REMOVE,
            path: path ? `${path}.${key}` : key,
          });
        }
      });

      // 检查新增或修改的属性
      Object.keys(curr).forEach((key) => {
        const currPath = path ? `${path}.${key}` : key;
        if (!(key in prev)) {
          operations.push({
            type: OperationType.ADD,
            path: currPath,
            value: curr[key],
          });
        } else if (JSON.stringify(prev[key]) !== JSON.stringify(curr[key])) {
          // 如果值是对象，递归比较
          if (
            typeof prev[key] === "object" &&
            prev[key] !== null &&
            typeof curr[key] === "object" &&
            curr[key] !== null
          ) {
            compareObjects(prev[key], curr[key], currPath);
          } else {
            operations.push({
              type: OperationType.UPDATE,
              path: currPath,
              value: curr[key],
            });
          }
        }
      });
      return;
    }

    // 对于基本类型，如果不同则更新
    if (prev !== curr) {
      operations.push({
        type: OperationType.UPDATE,
        path: path,
        value: curr,
      });
    }
  }

  compareObjects(previousState, currentState);
  return operations;
}
```

### 可靠性机制实现

```typescript
// 消息管理器
class MessageManager {
  private pendingMessages: Map<string, Message> = new Map();
  private retryInterval: number = 1000; // 重试间隔（毫秒）
  private maxRetries: number = 3; // 最大重试次数
  private sequenceNumber: number = 0; // 当前序列号

  // 发送消息并等待确认
  public async sendWithAck(message: Message): Promise<ActionResultMessage> {
    // 设置消息ID和序列号
    message.header.messageId = this.generateMessageId();
    message.header.sequenceNumber = this.nextSequenceNumber();

    // 存储待确认的消息
    this.pendingMessages.set(message.header.messageId, message);

    // 发送消息
    this.sendMessage(message);

    // 返回一个Promise，等待消息确认
    return new Promise((resolve, reject) => {
      let retries = 0;

      const retryTimer = setInterval(() => {
        // 检查消息是否已被确认（从待确认列表中移除）
        if (!this.pendingMessages.has(message.header.messageId)) {
          clearInterval(retryTimer);
          // 此处假设有一个消息结果缓存
          const result = this.getResultForMessage(message.header.messageId);
          resolve(result);
          return;
        }

        // 超出最大重试次数
        if (++retries > this.maxRetries) {
          clearInterval(retryTimer);
          this.pendingMessages.delete(message.header.messageId);
          reject(new Error(`消息发送失败，已重试${this.maxRetries}次`));
          return;
        }

        // 重新发送消息
        console.log(
          `重新发送消息：${message.header.messageId}，重试次数：${retries}`
        );
        this.sendMessage(message);
      }, this.retryInterval);
    });
  }

  // 处理收到的确认消息
  public handleAcknowledgement(ackMessage: ActionResultMessage): void {
    // 获取确认的消息ID
    const messageId = ackMessage.payload.actionId;

    // 从待确认列表中移除
    if (this.pendingMessages.has(messageId)) {
      this.pendingMessages.delete(messageId);
      // 存储结果（实际实现会更复杂）
      this.storeResult(messageId, ackMessage);
    }
  }

  // 获取下一个序列号
  private nextSequenceNumber(): number {
    return this.sequenceNumber++;
  }

  // 生成唯一消息ID
  private generateMessageId(): string {
    return `msg_${Date.now()}_${Math.floor(Math.random() * 1000)}`;
  }

  // 实际发送消息的方法（将与WebSocket集成）
  private sendMessage(message: Message): void {
    // 序列化消息
    const serializer = new MessageSerializer();
    const data = serializer.serialize(message);

    // 通过WebSocket发送
    // websocket.send(data);
    console.log(
      `发送消息：${message.header.messageId}，类型：${
        MessageType[message.header.type]
      }`
    );
  }

  // 存储消息结果（简化实现）
  private storeResult(messageId: string, result: ActionResultMessage): void {
    // 实际实现会将结果存储在缓存或状态管理器中
    console.log(
      `存储消息${messageId}的结果：${result.payload.success ? "成功" : "失败"}`
    );
  }

  // 获取消息结果（简化实现）
  private getResultForMessage(messageId: string): ActionResultMessage {
    // 实际实现会从缓存或状态管理器中获取结果
    return {
      header: {
        messageId: `ack_${messageId}`,
        type: MessageType.ACTION_RESULT,
        timestamp: Date.now(),
        roomId: "模拟房间ID",
        sequenceNumber: 0,
        compression: false,
      },
      payload: {
        actionId: messageId,
        success: true,
        resultData: { message: "模拟结果数据" },
      },
    } as ActionResultMessage;
  }
}
```

## 验收标准

1. 协议设计文档完整，包含所有必要的消息类型和处理流程
2. TypeScript 接口定义清晰，能够支持完整的游戏状态同步需求
3. 序列化/反序列化性能满足要求，在客户端设备上测试延迟不超过 10ms
4. 增量更新机制有效减少数据传输量，比完整状态传输减少至少 60%的数据量
5. 消息可靠性机制能够处理网络波动场景，确保消息不丢失
6. 所有安全机制设计完善，能够防止基本的消息注入和篡改攻击
7. 协议文档经技术评审通过，并交付开发团队实现

## 技术依赖

1. WebSocket 连接库（微信小游戏 API）
2. 序列化库：MessagePack 或 Protobuf
3. 数据压缩库：pako 或 zlib
4. UUID 生成库
5. TypeScript 编译环境

## 工作量估计

预计工作量：1 人/天
