# Task 4.2.2: 实时消息系统实现

## 描述

设计并实现阿瓦隆微信小游戏的实时消息系统，包括事件定义、消息发送与接收、消息序列化、消息队列和消息处理机制，确保游戏中的实时通信可靠高效。

## 验收标准

1. 定义完整的消息类型和事件枚举
2. 实现消息的发送、广播和接收机制
3. 设计并实现消息队列和处理器
4. 创建消息的序列化和反序列化工具
5. 实现消息确认和重试机制
6. 添加消息过滤和优先级处理
7. 完成消息处理的错误处理和日志记录
8. 编写消息系统的单元测试
9. 实现基于 Redis 的消息广播机制，支持集群部署

## 详细内容

### 消息与事件定义

1. **消息类型定义**

   ```typescript
   // 消息基础接口
   export interface Message {
     id: string; // 消息唯一ID
     type: MessageType; // 消息类型
     timestamp: number; // 时间戳
     payload: any; // 消息内容
     roomId?: string; // 房间ID（如适用）
   }

   // 消息类型枚举
   export enum MessageType {
     // 系统消息
     CONNECT = "connect",
     DISCONNECT = "disconnect",
     ERROR = "error",

     // 房间消息
     ROOM_JOIN = "room:join",
     ROOM_LEAVE = "room:leave",
     ROOM_UPDATE = "room:update",

     // 游戏消息
     GAME_START = "game:start",
     GAME_STATE = "game:state",
     GAME_ACTION = "game:action",
     GAME_END = "game:end",

     // 玩家消息
     PLAYER_ACTION = "player:action",
     PLAYER_VOTE = "player:vote",
     PLAYER_CHAT = "player:chat",

     // 其他
     PING = "ping",
     PONG = "pong",
     ACK = "ack",
   }
   ```

2. **游戏事件定义**

   ```typescript
   // 游戏事件枚举
   export enum GameEvent {
     // 角色分配
     ROLE_ASSIGNED = "role:assigned",

     // 任务相关
     TEAM_PROPOSAL = "team:proposal",
     TEAM_VOTE = "team:vote",
     TEAM_APPROVED = "team:approved",
     TEAM_REJECTED = "team:rejected",

     // 任务执行
     MISSION_START = "mission:start",
     MISSION_VOTE = "mission:vote",
     MISSION_COMPLETE = "mission:complete",

     // 游戏状态
     GAME_PHASE_CHANGE = "game:phaseChange",
     GAME_OVER = "game:over",
   }

   // 游戏角色枚举
   export enum GameRole {
     MERLIN = "merlin",
     PERCIVAL = "percival",
     LOYAL_SERVANT = "loyalServant",
     ASSASSIN = "assassin",
     MORGANA = "morgana",
     MORDRED = "mordred",
     OBERON = "oberon",
   }
   ```

### 消息服务实现

1. **消息服务**

   ```typescript
   @Injectable()
   export class MessageService {
     private readonly logger = new Logger(MessageService.name);

     constructor(
       private readonly socketGateway: GameSocketGateway,
       @Inject("REDIS_PUB") private readonly redisPub: Redis,
       @Inject("REDIS_SUB") private readonly redisSub: Redis
     ) {
       this.setupRedisSubscription();
     }

     // 发送消息给指定用户
     async sendToUser(userId: string, message: Message): Promise<void> {
       try {
         const socket = this.socketGateway.getUserSocket(userId);
         if (socket) {
           socket.emit(message.type, message);
           this.logger.debug(`消息发送至用户 ${userId}: ${message.type}`);
         } else {
           this.logger.warn(`用户 ${userId} 不在线，消息未送达`);
         }
       } catch (error) {
         this.logger.error(`发送消息到用户 ${userId} 失败: ${error.message}`);
         throw new Error(`消息发送失败: ${error.message}`);
       }
     }

     // 发送消息给房间内所有用户
     async sendToRoom(roomId: string, message: Message): Promise<void> {
       try {
         // 通过Redis发布，确保集群环境下所有服务实例都能接收到
         message.roomId = roomId;
         await this.redisPub.publish("room-messages", JSON.stringify(message));
         this.logger.debug(`消息发送至房间 ${roomId}: ${message.type}`);
       } catch (error) {
         this.logger.error(`发送消息到房间 ${roomId} 失败: ${error.message}`);
         throw new Error(`房间消息发送失败: ${error.message}`);
       }
     }

     // 设置Redis订阅
     private setupRedisSubscription(): void {
       this.redisSub.subscribe("room-messages");
       this.redisSub.on("message", (channel, message) => {
         if (channel === "room-messages") {
           try {
             const parsedMessage: Message = JSON.parse(message);
             if (parsedMessage.roomId) {
               this.socketGateway.server
                 .to(parsedMessage.roomId)
                 .emit(parsedMessage.type, parsedMessage);
             }
           } catch (error) {
             this.logger.error(`处理Redis消息失败: ${error.message}`);
           }
         }
       });
     }

     // 生成唯一消息ID
     generateMessageId(): string {
       return uuid();
     }
   }
   ```

### 消息确认与重试机制

1. **消息确认跟踪**

   ```typescript
   @Injectable()
   export class MessageAckService {
     private pendingMessages = new Map<
       string,
       {
         message: Message;
         expires: number;
         attempts: number;
       }
     >();

     constructor(
       private readonly messageService: MessageService,
       private readonly configService: ConfigService
     ) {
       // 定期检查未确认的消息
       setInterval(() => this.checkPendingMessages(), 5000);
     }

     // 跟踪需要确认的消息
     trackMessage(message: Message, recipientId: string): void {
       const maxAttempts = this.configService.get("MESSAGE_MAX_RETRIES", 3);
       const ttl = this.configService.get("MESSAGE_ACK_TTL", 10000); // 10秒

       this.pendingMessages.set(message.id, {
         message,
         expires: Date.now() + ttl,
         attempts: 1,
       });

       // 设置确认超时处理
       setTimeout(() => {
         this.handleAckTimeout(message.id, recipientId);
       }, ttl);
     }

     // 处理消息确认
     handleAck(messageId: string): void {
       if (this.pendingMessages.has(messageId)) {
         this.pendingMessages.delete(messageId);
       }
     }

     // 处理确认超时
     private async handleAckTimeout(
       messageId: string,
       recipientId: string
     ): Promise<void> {
       const pendingItem = this.pendingMessages.get(messageId);
       if (!pendingItem) return;

       const maxAttempts = this.configService.get("MESSAGE_MAX_RETRIES", 3);

       if (pendingItem.attempts < maxAttempts) {
         // 重试发送
         pendingItem.attempts += 1;
         try {
           await this.messageService.sendToUser(
             recipientId,
             pendingItem.message
           );
         } catch (error) {
           console.error(`消息重试失败: ${error.message}`);
         }
       } else {
         // 达到最大重试次数，放弃并记录
         this.pendingMessages.delete(messageId);
         console.warn(`消息 ${messageId} 发送失败，超过最大重试次数`);
       }
     }

     // 检查所有待确认消息
     private checkPendingMessages(): void {
       const now = Date.now();
       for (const [messageId, item] of this.pendingMessages.entries()) {
         if (now > item.expires) {
           this.pendingMessages.delete(messageId);
         }
       }
     }
   }
   ```

### 消息序列化与处理

1. **消息序列化工具**

   ```typescript
   export class MessageSerializer {
     // 序列化消息对象为传输格式
     static serialize(message: Message): string {
       return JSON.stringify(message);
     }

     // 从传输格式解析消息对象
     static deserialize(data: string): Message {
       try {
         const parsed = JSON.parse(data);
         return parsed as Message;
       } catch (error) {
         throw new Error(`消息解析失败: ${error.message}`);
       }
     }

     // 验证消息格式
     static validate(message: any): boolean {
       return (
         message &&
         typeof message.id === "string" &&
         typeof message.type === "string" &&
         typeof message.timestamp === "number"
       );
     }
   }
   ```

2. **消息处理器工厂**

   ```typescript
   @Injectable()
   export class MessageHandlerFactory {
     private handlers = new Map<MessageType, MessageHandler>();

     constructor(
       private readonly gameActionHandler: GameActionHandler,
       private readonly roomHandler: RoomHandler,
       private readonly playerActionHandler: PlayerActionHandler,
       private readonly systemMessageHandler: SystemMessageHandler
     ) {
       this.registerHandlers();
     }

     // 注册所有消息处理器
     private registerHandlers(): void {
       // 游戏相关消息
       this.handlers.set(MessageType.GAME_ACTION, this.gameActionHandler);
       this.handlers.set(MessageType.GAME_START, this.gameActionHandler);
       this.handlers.set(MessageType.GAME_END, this.gameActionHandler);

       // 房间相关消息
       this.handlers.set(MessageType.ROOM_JOIN, this.roomHandler);
       this.handlers.set(MessageType.ROOM_LEAVE, this.roomHandler);
       this.handlers.set(MessageType.ROOM_UPDATE, this.roomHandler);

       // 玩家相关消息
       this.handlers.set(MessageType.PLAYER_ACTION, this.playerActionHandler);
       this.handlers.set(MessageType.PLAYER_VOTE, this.playerActionHandler);
       this.handlers.set(MessageType.PLAYER_CHAT, this.playerActionHandler);

       // 系统消息
       this.handlers.set(MessageType.ERROR, this.systemMessageHandler);
       this.handlers.set(MessageType.CONNECT, this.systemMessageHandler);
       this.handlers.set(MessageType.DISCONNECT, this.systemMessageHandler);
     }

     // 获取对应类型的消息处理器
     getHandler(type: MessageType): MessageHandler {
       const handler = this.handlers.get(type);
       if (!handler) {
         throw new Error(`没有找到类型 ${type} 的消息处理器`);
       }
       return handler;
     }

     // 处理消息
     async handleMessage(message: Message, client: Socket): Promise<void> {
       try {
         const handler = this.getHandler(message.type);
         await handler.handle(message, client);
       } catch (error) {
         console.error(`处理消息 ${message.type} 失败: ${error.message}`);
         // 发送错误消息给客户端
         client.emit(MessageType.ERROR, {
           id: uuid(),
           type: MessageType.ERROR,
           timestamp: Date.now(),
           payload: { message: "消息处理失败", originalType: message.type },
         });
       }
     }
   }
   ```

## 工作量估计

3 人天

## 技术关键点

1. 设计清晰的消息类型和事件系统，便于扩展和维护
2. 确保消息传递的可靠性，实现必要的重试和确认机制
3. 优化消息序列化过程，减少传输数据大小
4. 实现高效的消息处理机制，避免阻塞和延迟
5. 确保在分布式环境下消息的一致性和可靠传递

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.2: WebSocket 服务](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Socket.IO 文档](https://socket.io/docs/)
- [NestJS WebSockets 文档](https://docs.nestjs.com/websockets/gateways)

## 依赖关系

- 上游依赖:
  - Task 4.2.1: WebSocket 服务框架搭建
- 下游依赖:
  - Task 4.2.3: 房间与游戏状态同步
  - Task 4.2.4: 断线重连与状态恢复
