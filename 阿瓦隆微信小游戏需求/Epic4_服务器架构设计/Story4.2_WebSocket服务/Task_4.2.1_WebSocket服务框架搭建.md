# Task 4.2.1: WebSocket 服务框架搭建

## 描述

设计并实现阿瓦隆微信小游戏的 WebSocket 服务基础框架，包括 Socket.IO 服务器配置、集群支持、事件网关、连接管理、认证机制和基本消息处理功能。

## 验收标准

1. 完成基于 NestJS 和 Socket.IO 的 WebSocket 服务器搭建
2. 实现 Socket.IO 的 Redis 适配器，支持集群化部署
3. 设计并实现基础的 WebSocket 网关和事件处理器
4. 完成连接建立、断开和心跳机制
5. 实现基于 JWT 的 WebSocket 连接认证
6. 创建基础的消息发送和广播机制
7. 编写 WebSocket 服务的单元测试
8. 配置合适的 Socket.IO 服务器参数，优化性能
9. 实现连接状态监控和日志记录

## 详细内容

### WebSocket 服务器搭建

1. **NestJS WebSocket 网关配置**

   - 创建主 WebSocket 网关
   - 配置 Socket.IO 选项
   - 设置命名空间和路径

   ```typescript
   // game.gateway.ts 示例
   import {
     WebSocketGateway,
     WebSocketServer,
     OnGatewayInit,
     OnGatewayConnection,
     OnGatewayDisconnect,
   } from "@nestjs/websockets";
   import { Server, Socket } from "socket.io";
   import { Logger } from "@nestjs/common";

   @WebSocketGateway({
     cors: {
       origin: "*", // 生产环境应限制为特定域名
     },
     namespace: "/game",
     transports: ["websocket", "polling"],
   })
   export class GameGateway
     implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
   {
     @WebSocketServer() server: Server;
     private readonly logger = new Logger(GameGateway.name);

     afterInit(server: Server) {
       this.logger.log("WebSocket服务器初始化完成");
     }

     handleConnection(client: Socket, ...args: any[]) {
       this.logger.log(`客户端连接: ${client.id}`);
     }

     handleDisconnect(client: Socket) {
       this.logger.log(`客户端断开连接: ${client.id}`);
     }
   }
   ```

2. **Socket.IO 服务器配置**

   - 设置 Socket.IO 选项
   - 配置心跳间隔和超时
   - 优化连接参数

   ```typescript
   // main.ts 中配置Socket.IO
   import { NestFactory } from "@nestjs/core";
   import { AppModule } from "./app.module";
   import { RedisIoAdapter } from "./adapters/redis-io.adapter";
   import { ConfigService } from "@nestjs/config";

   async function bootstrap() {
     const app = await NestFactory.create(AppModule);
     const configService = app.get(ConfigService);

     // 创建并配置Redis适配器
     const redisIoAdapter = new RedisIoAdapter(app, configService);
     await redisIoAdapter.connectToRedis();
     app.useWebSocketAdapter(redisIoAdapter);

     await app.listen(3001);
     console.log(`WebSocket服务器运行在: ${await app.getUrl()}`);
   }
   bootstrap();
   ```

### Redis 适配器实现

1. **Redis 适配器开发**

   - 实现 Socket.IO Redis 适配器
   - 配置 Redis 连接和选项
   - 设置适配器选项

   ```typescript
   // redis-io.adapter.ts 示例
   import { IoAdapter } from "@nestjs/platform-socket.io";
   import { createAdapter } from "@socket.io/redis-adapter";
   import { Redis } from "ioredis";
   import { ConfigService } from "@nestjs/config";
   import { INestApplication } from "@nestjs/common";
   import { ServerOptions } from "socket.io";

   export class RedisIoAdapter extends IoAdapter {
     private adapterConstructor: ReturnType<typeof createAdapter>;

     constructor(
       app: INestApplication,
       private readonly configService: ConfigService
     ) {
       super(app);
     }

     async connectToRedis(): Promise<void> {
       const pubClient = new Redis({
         host: this.configService.get("REDIS_HOST"),
         port: this.configService.get("REDIS_PORT"),
         password: this.configService.get("REDIS_PASSWORD"),
         db: Number(this.configService.get("REDIS_DB", 0)),
       });

       const subClient = pubClient.duplicate();

       this.adapterConstructor = createAdapter(pubClient, subClient, {
         publishOnSpecificResponseChannel: true,
         requestsTimeout: 5000,
       });
     }

     createIOServer(port: number, options?: ServerOptions): any {
       const server = super.createIOServer(port, {
         ...options,
         cors: {
           origin: this.configService.get("CORS_ORIGINS", "*"),
           methods: ["GET", "POST"],
           credentials: true,
         },
         pingInterval: 25000, // 25秒心跳间隔
         pingTimeout: 10000, // 10秒超时
         connectTimeout: 10000, // 连接超时
         maxHttpBufferSize: 1e6, // 1MB最大消息大小
         transports: ["websocket", "polling"], // 支持的传输方式
       });

       server.adapter(this.adapterConstructor);
       return server;
     }
   }
   ```

2. **集群支持配置**
   - 设计跨实例消息传递机制
   - 配置 Redis 发布/订阅
   - 处理跨实例会话管理

### 连接管理实现

1. **连接处理器**

   - 实现连接建立处理
   - 处理断开连接逻辑
   - 实现心跳和连接状态监控

   ```typescript
   // connection.service.ts 示例
   import { Injectable, Logger } from "@nestjs/common";
   import { Socket } from "socket.io";
   import { Redis } from "ioredis";
   import { InjectRedis } from "@nestjs-modules/ioredis";

   @Injectable()
   export class ConnectionService {
     private readonly logger = new Logger(ConnectionService.name);

     constructor(@InjectRedis() private readonly redis: Redis) {}

     async handleConnection(client: Socket): Promise<void> {
       const { id: socketId } = client;

       try {
         // 记录连接信息到Redis
         await this.redis.hset(`socket:${socketId}`, {
           connected_at: Date.now(),
           ip: client.handshake.address,
           user_agent: client.handshake.headers["user-agent"] || "",
         });

         // 设置连接过期时间（为了处理可能的孤立连接）
         await this.redis.expire(`socket:${socketId}`, 86400); // 24小时

         this.logger.log(`Socket连接成功: ${socketId}`);
       } catch (error) {
         this.logger.error(`处理连接时出错: ${error.message}`);
       }
     }

     async handleDisconnect(client: Socket): Promise<void> {
       const { id: socketId } = client;

       try {
         // 获取连接关联的用户ID（如果有）
         const userId = await this.redis.hget(`socket:${socketId}`, "user_id");

         // 清理连接数据
         await this.redis.del(`socket:${socketId}`);

         // 如果连接已认证，更新用户状态
         if (userId) {
           await this.redis.hset(`user:${userId}`, {
             is_online: 0,
             last_disconnect: Date.now(),
           });
           this.logger.log(`用户下线: ${userId}`);
         }

         this.logger.log(`Socket断开连接: ${socketId}`);
       } catch (error) {
         this.logger.error(`处理断开连接时出错: ${error.message}`);
       }
     }

     async trackActiveSockets(): Promise<number> {
       // 统计当前活跃连接数
       const keys = await this.redis.keys("socket:*");
       return keys.length;
     }
   }
   ```

2. **心跳机制**
   - 配置 Socket.IO 心跳参数
   - 监控连接健康状态
   - 处理僵尸连接

### 认证机制实现

1. **WebSocket 认证守卫**

   - 实现 JWT 验证守卫
   - 处理握手阶段认证
   - 关联用户身份和 Socket 连接

   ```typescript
   // ws-auth.guard.ts 示例
   import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
   import { JwtService } from "@nestjs/jwt";
   import { Socket } from "socket.io";
   import { WsException } from "@nestjs/websockets";
   import { ConnectionService } from "./connection.service";

   @Injectable()
   export class WsAuthGuard implements CanActivate {
     constructor(
       private readonly jwtService: JwtService,
       private readonly connectionService: ConnectionService
     ) {}

     async canActivate(context: ExecutionContext): Promise<boolean> {
       const client: Socket = context.switchToWs().getClient();
       const token = this.extractToken(client);

       if (!token) {
         throw new WsException("未提供认证令牌");
       }

       try {
         const payload = this.jwtService.verify(token);
         const { sub: userId, openId } = payload;

         // 在socket对象上保存用户信息
         client.data.user = { userId, openId };

         // 记录用户会话信息
         await this.updateUserSession(client, userId);

         return true;
       } catch (error) {
         throw new WsException("认证失败");
       }
     }

     private extractToken(client: Socket): string | null {
       const { handshake } = client;
       const { auth, query } = handshake;

       // 优先从auth对象中获取
       if (auth?.token) {
         return auth.token;
       }

       // 其次从query参数中获取
       if (query?.token) {
         return query.token as string;
       }

       // 最后从headers中获取
       const authorization = handshake.headers.authorization;
       if (authorization && authorization.startsWith("Bearer ")) {
         return authorization.substring(7);
       }

       return null;
     }

     private async updateUserSession(
       client: Socket,
       userId: string
     ): Promise<void> {
       const socketId = client.id;

       // 更新Socket记录，关联用户ID
       await client.to(`socket:${socketId}`).emit("user_id", userId);

       // 更新用户状态为在线
       await client.to(`user:${userId}`).emit({
         socket_id: socketId,
         is_online: 1,
         last_connect: Date.now(),
       });
     }
   }
   ```

2. **事件级别认证**
   - 实现装饰器和守卫
   - 对特定事件进行权限控制
   - 处理未认证请求

### 基础消息处理

1. **消息服务**

   - 实现基础消息发送功能
   - 创建房间消息广播机制
   - 设计消息格式和规范

   ```typescript
   // message.service.ts 示例
   import { Injectable, Logger } from "@nestjs/common";
   import { Server, Socket } from "socket.io";

   @Injectable()
   export class MessageService {
     private readonly logger = new Logger(MessageService.name);
     private server: Server;

     setServer(server: Server) {
       this.server = server;
     }

     sendToClient(clientId: string, event: string, data: any): boolean {
       try {
         this.server.to(clientId).emit(event, data);
         return true;
       } catch (error) {
         this.logger.error(`发送消息到客户端失败: ${error.message}`);
         return false;
       }
     }

     sendToUser(userId: string, event: string, data: any): boolean {
       try {
         this.server.to(`user:${userId}`).emit(event, data);
         return true;
       } catch (error) {
         this.logger.error(`发送消息到用户失败: ${error.message}`);
         return false;
       }
     }

     sendToRoom(
       roomId: string,
       event: string,
       data: any,
       except?: string
     ): boolean {
       try {
         if (except) {
           this.server.to(`room:${roomId}`).except(except).emit(event, data);
         } else {
           this.server.to(`room:${roomId}`).emit(event, data);
         }
         return true;
       } catch (error) {
         this.logger.error(`发送消息到房间失败: ${error.message}`);
         return false;
       }
     }

     broadcastAll(event: string, data: any): boolean {
       try {
         this.server.emit(event, data);
         return true;
       } catch (error) {
         this.logger.error(`广播消息失败: ${error.message}`);
         return false;
       }
     }

     // 加入房间
     joinRoom(client: Socket, roomId: string): void {
       const roomName = `room:${roomId}`;
       client.join(roomName);
       this.logger.log(`客户端 ${client.id} 加入房间 ${roomName}`);
     }

     // 离开房间
     leaveRoom(client: Socket, roomId: string): void {
       const roomName = `room:${roomId}`;
       client.leave(roomName);
       this.logger.log(`客户端 ${client.id} 离开房间 ${roomName}`);
     }
   }
   ```

2. **事件处理器**
   - 创建事件监听装饰器
   - 实现基础事件路由
   - 处理异常和错误

### 监控与日志

1. **连接监控**

   - 实现连接统计和指标收集
   - 记录连接和断开事件
   - 设计异常连接检测

2. **性能指标**
   - 收集 WebSocket 服务性能指标
   - 监控消息吞吐量
   - 跟踪延迟和错误率

## 工作量估计

3 人天

## 技术关键点

1. 正确配置 Socket.IO 以优化性能和可靠性
2. 实现健壮的认证和会话管理机制
3. 设计可扩展的事件处理架构
4. 确保 Redis 适配器正确支持集群部署
5. 优化连接管理以支持大量并发连接

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.2: WebSocket 服务](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Socket.IO 官方文档](https://socket.io/docs/)
- [NestJS WebSocket 文档](https://docs.nestjs.com/websockets/gateways)
- [Socket.IO Redis 适配器文档](https://socket.io/docs/v4/redis-adapter/)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
  - Task 4.1.3: 数据库与缓存配置
  - Task 4.1.4: 认证与授权服务
- 下游依赖:
  - Task 4.2.2: 实时消息系统实现
  - Task 4.2.3: 房间与游戏状态同步
  - Task 4.2.4: 断线重连与状态恢复
