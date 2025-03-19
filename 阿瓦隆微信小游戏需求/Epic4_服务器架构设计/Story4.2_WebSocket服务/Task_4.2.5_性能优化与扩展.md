# Task 4.2.5: 性能优化与扩展

## 描述

针对阿瓦隆微信小游戏的 WebSocket 服务进行性能优化和扩展性增强，确保服务在高并发场景下的稳定运行，并能够根据用户量增长进行水平扩展，同时降低资源消耗和网络延迟。

## 验收标准

1. 实现 Socket.IO 服务器集群，支持水平扩展
2. 优化 WebSocket 消息传输效率，减少带宽使用
3. 实现连接速率限制和负载控制
4. 设计并实现性能监控指标收集
5. 优化 Redis 连接池和查询效率
6. 实现高效的消息广播和分发机制
7. 支持至少 300 个并发连接
8. 平均消息延迟控制在 100ms 以内
9. 实现服务器优雅启动和关闭
10. 编写负载测试和性能基准测试

## 详细内容

### 集群与扩展

1. **Socket.IO 集群配置**

   ```typescript
   // 简化的Socket.IO集群配置
   @Module({
     imports: [ConfigModule, RedisModule],
     providers: [
       {
         provide: "SOCKET_IO_SERVER",
         useFactory: (configService: ConfigService, redisClient: Redis) => {
           const io = new Server({
             cors: {
               origin: configService.get("CORS_ORIGIN", "*"),
               methods: ["GET", "POST"],
               credentials: true,
             },
             // 提高传输效率的配置
             transports: ["websocket", "polling"],
             pingInterval: 25000,
             pingTimeout: 20000,
             connectTimeout: 10000,
           });

           // 配置Redis适配器实现集群
           const pubClient = redisClient;
           const subClient = pubClient.duplicate();

           io.adapter(
             createAdapter(pubClient, subClient, {
               key: "avalon-game",
               publishOnSpecificResponseChannel: true,
             })
           );

           return io;
         },
         inject: [ConfigService, "REDIS_CLIENT"],
       },
     ],
     exports: ["SOCKET_IO_SERVER"],
   })
   export class SocketIoModule {}
   ```

2. **水平扩展支持**

   ```typescript
   // 配置文件示例 - 简化版
   export default {
     // 常规配置
     app: {
       port: process.env.PORT || 3000,
       environment: process.env.NODE_ENV || "development",
     },

     // 集群配置
     cluster: {
       enabled: process.env.ENABLE_CLUSTER === "true",
       workers: process.env.WORKER_COUNT || "auto", // 'auto'表示基于CPU核心数
       sticky: true, // 使用粘性会话
     },

     // 负载均衡配置
     loadBalancing: {
       maxConnPerInstance: parseInt(
         process.env.MAX_CONN_PER_INSTANCE || "1000"
       ),
       connectionRedistribution: true,
     },
   };
   ```

### 性能优化

1. **消息压缩与优化**

   ```typescript
   @Injectable()
   export class MessageOptimizer {
     // 压缩消息负载(简化版)
     compressPayload(payload: any): Buffer {
       const serialized = JSON.stringify(payload);
       return zlib.deflateSync(serialized);
     }

     // 解压消息负载
     decompressPayload(compressed: Buffer): any {
       const decompressed = zlib.inflateSync(compressed).toString();
       return JSON.parse(decompressed);
     }

     // 优化消息结构，移除冗余字段
     optimizeMessage(message: any): any {
       // 创建消息的副本
       const optimized = { ...message };

       // 移除不必要的字段
       delete optimized.debug;

       // 简化长字段
       if (optimized.payload && optimized.payload.fullState) {
         // 只保留增量更新
         optimized.payload = this.createDelta(optimized.payload);
       }

       return optimized;
     }

     // 创建增量更新(非完整实现)
     private createDelta(payload: any): any {
       // 这里应该有实际的增量计算逻辑
       return {
         isDelta: true,
         changes: {
           /* 变更的字段 */
         },
       };
     }
   }
   ```

2. **Redis 优化**

   ```typescript
   @Injectable()
   export class OptimizedRedisService {
     private readonly connectionPool: Redis[] = [];
     private currentIndex = 0;

     constructor(
       private readonly configService: ConfigService,
       private readonly logger: LoggerService
     ) {
       this.initPool();
     }

     // 初始化连接池
     private initPool(): void {
       const poolSize = this.configService.get("REDIS_POOL_SIZE", 5);

       for (let i = 0; i < poolSize; i++) {
         this.connectionPool.push(
           new Redis({
             host: this.configService.get("REDIS_HOST", "localhost"),
             port: this.configService.get("REDIS_PORT", 6379),
             password: this.configService.get("REDIS_PASSWORD", ""),
             db: this.configService.get("REDIS_DB", 0),
             maxRetriesPerRequest: 3,
             enableOfflineQueue: false,
             connectTimeout: 5000,
           })
         );
       }

       this.logger.log(`Redis连接池已初始化，大小: ${poolSize}`);
     }

     // 获取连接(简单的循环分配)
     private getConnection(): Redis {
       const conn = this.connectionPool[this.currentIndex];
       this.currentIndex = (this.currentIndex + 1) % this.connectionPool.length;
       return conn;
     }

     // 优化的键查询，使用缓存减少Redis访问(简化实现)
     async getCached(key: string, ttl: number = 60): Promise<any> {
       // 使用本地内存缓存实现
       // 实际应使用如node-cache等库

       // 模拟从缓存获取
       const cachedValue = null; // 本地缓存中的值(此处简化)

       if (cachedValue) {
         return cachedValue;
       }

       // 缓存未命中，从Redis获取
       const result = await this.getConnection().get(key);

       // 存入本地缓存(此处简化)
       // setLocalCache(key, result, ttl);

       return result ? JSON.parse(result) : null;
     }
   }
   ```

### 负载控制与限流

1. **连接限流**

   ```typescript
   @Injectable()
   export class ConnectionRateLimiter {
     // 每IP的连接计数
     private ipConnections = new Map<
       string,
       {
         count: number;
         lastReset: number;
       }
     >();

     // 每用户的连接计数
     private userConnections = new Map<
       string,
       {
         count: number;
         connections: Set<string>;
       }
     >();

     constructor(private readonly configService: ConfigService) {
       // 定期清理过期记录
       setInterval(() => this.cleanup(), 60000);
     }

     // 检查IP是否可以建立新连接
     canConnect(ip: string): boolean {
       const now = Date.now();
       const windowSize = 60000; // 1分钟窗口
       const maxConnPerMinute = this.configService.get(
         "MAX_CONN_PER_MINUTE_PER_IP",
         20
       );

       let record = this.ipConnections.get(ip);

       if (!record) {
         record = { count: 0, lastReset: now };
         this.ipConnections.set(ip, record);
       }

       // 检查是否需要重置计数
       if (now - record.lastReset > windowSize) {
         record.count = 0;
         record.lastReset = now;
       }

       // 检查是否超过限制
       if (record.count >= maxConnPerMinute) {
         return false;
       }

       // 增加计数
       record.count++;
       return true;
     }

     // 跟踪用户连接
     trackUserConnection(userId: string, socketId: string): void {
       let record = this.userConnections.get(userId);

       if (!record) {
         record = { count: 0, connections: new Set() };
         this.userConnections.set(userId, record);
       }

       record.connections.add(socketId);
       record.count = record.connections.size;
     }

     // 移除用户连接
     removeUserConnection(userId: string, socketId: string): void {
       const record = this.userConnections.get(userId);
       if (record) {
         record.connections.delete(socketId);
         record.count = record.connections.size;

         if (record.count === 0) {
           this.userConnections.delete(userId);
         }
       }
     }

     // 清理过期记录
     private cleanup(): void {
       const now = Date.now();
       const expireTime = 3600000; // 1小时

       // 清理IP记录
       for (const [ip, record] of this.ipConnections.entries()) {
         if (now - record.lastReset > expireTime) {
           this.ipConnections.delete(ip);
         }
       }
     }
   }
   ```

### 性能监控

1. **WebSocket 指标收集**

   ```typescript
   @Injectable()
   export class WebSocketMetrics {
     // 基础指标
     private metrics = {
       connections: {
         current: 0,
         total: 0,
         rejected: 0,
       },
       messages: {
         sent: 0,
         received: 0,
         errors: 0,
       },
       rooms: {
         count: 0,
       },
       latency: {
         sum: 0,
         count: 0,
         max: 0,
       },
     };

     constructor(
       private readonly metricsRegistry: PrometheusRegistry,
       private readonly logger: LoggerService
     ) {
       this.setupPrometheusMetrics();

       // 定期记录指标
       setInterval(() => this.logMetrics(), 60000);
     }

     // 连接建立
     connectionCreated(): void {
       this.metrics.connections.current++;
       this.metrics.connections.total++;
     }

     // 连接关闭
     connectionClosed(): void {
       this.metrics.connections.current = Math.max(
         0,
         this.metrics.connections.current - 1
       );
     }

     // 消息接收
     messageReceived(): void {
       this.metrics.messages.received++;
     }

     // 消息发送
     messageSent(): void {
       this.metrics.messages.sent++;
     }

     // 记录消息延迟
     recordLatency(latencyMs: number): void {
       this.metrics.latency.sum += latencyMs;
       this.metrics.latency.count++;
       this.metrics.latency.max = Math.max(this.metrics.latency.max, latencyMs);
     }

     // 设置房间数量
     setRoomCount(count: number): void {
       this.metrics.rooms.count = count;
     }

     // 获取当前指标
     getMetrics(): any {
       return {
         ...this.metrics,
         latency: {
           ...this.metrics.latency,
           avg:
             this.metrics.latency.count > 0
               ? this.metrics.latency.sum / this.metrics.latency.count
               : 0,
         },
       };
     }

     // 记录指标到日志
     private logMetrics(): void {
       const metrics = this.getMetrics();
       this.logger.log(
         `WebSocket指标 - 连接数: ${metrics.connections.current}, 消息: ${
           metrics.messages.sent
         }发/${
           metrics.messages.received
         }收, 平均延迟: ${metrics.latency.avg.toFixed(2)}ms`
       );
     }

     // 设置Prometheus指标(简化版)
     private setupPrometheusMetrics(): void {
       // 实际实现中使用prom-client库
       // this.connectionsGauge = new Gauge({...});
       // this.messagesCounter = new Counter({...});
     }
   }
   ```

### 优雅启动与关闭

```typescript
@Injectable()
export class GracefulShutdownService {
  private isShuttingDown = false;

  constructor(
    private readonly socketServer: Server,
    private readonly logger: LoggerService
  ) {
    this.setupShutdownHooks();
  }

  private setupShutdownHooks(): void {
    // 处理进程信号
    process.on("SIGTERM", () => this.shutdown("SIGTERM"));
    process.on("SIGINT", () => this.shutdown("SIGINT"));

    // 处理未捕获的异常
    process.on("uncaughtException", (error) => {
      this.logger.error(`未捕获的异常: ${error.message}`, error.stack);
      this.shutdown("uncaughtException");
    });
  }

  private async shutdown(signal: string): Promise<void> {
    if (this.isShuttingDown) return;

    this.isShuttingDown = true;
    this.logger.log(`收到${signal}信号，开始优雅关闭...`);

    try {
      // 1. 停止接受新连接
      this.logger.log("停止接受新连接...");

      // 2. 通知客户端服务器即将关闭
      this.logger.log("通知客户端服务器即将关闭...");
      const connectedSockets = await this.getConnectedSockets();
      for (const socket of connectedSockets) {
        socket.emit("server:shutdown", {
          message: "服务器即将维护，请稍后重连",
          reconnectAfter: 30000,
        });
      }

      // 3. 等待一段时间让客户端处理
      this.logger.log("等待客户端处理关闭通知...");
      await new Promise((resolve) => setTimeout(resolve, 2000));

      // 4. 关闭Socket.IO服务器
      this.logger.log("关闭WebSocket服务器...");
      await this.closeSocketServer();

      // 5. 完成关闭
      this.logger.log("服务器已成功关闭");
      process.exit(0);
    } catch (error) {
      this.logger.error(`关闭过程中出错: ${error.message}`);
      process.exit(1);
    }
  }

  // 获取所有已连接的客户端(简化)
  private async getConnectedSockets(): Promise<any[]> {
    const sockets = await this.socketServer.fetchSockets();
    return sockets;
  }

  // 关闭Socket.IO服务器
  private closeSocketServer(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.socketServer.close((err) => {
        if (err) {
          reject(err);
        } else {
          resolve();
        }
      });
    });
  }
}
```

### 负载测试

1. **k6 负载测试脚本示例**

   ```javascript
   // loadtest.js - k6测试脚本示例
   import { check } from "k6";
   import { SharedArray } from "k6/data";
   import { Socket } from "k6/experimental/websockets";

   // 配置
   export const options = {
     stages: [
       { duration: "30s", target: 100 }, // 逐步增加到100个用户
       { duration: "1m", target: 100 }, // 保持100个用户1分钟
       { duration: "30s", target: 200 }, // 增加到200个用户
       { duration: "1m", target: 200 }, // 保持200个用户1分钟
       { duration: "30s", target: 0 }, // 逐步减少到0个用户
     ],
     thresholds: {
       ws_connecting: ["p(95)<1000"], // 95%的连接应在1s内完成
       ws_msgs_rx: ["rate>100"], // 每秒接收至少100条消息
     },
   };

   // 主测试函数
   export default function () {
     // 建立连接
     const url = "ws://localhost:3000/socket.io/?EIO=4&transport=websocket";
     const socket = new Socket(url);

     socket.on("open", () => {
       console.log("连接已打开");

       // 模拟认证
       socket.send(
         JSON.stringify({
           type: "auth",
           token: "test-token",
         })
       );
     });

     socket.on("message", (data) => {
       console.log(`收到消息: ${data}`);

       // 验证响应
       const parsed = JSON.parse(data);
       if (parsed.type === "auth_response") {
         check(parsed, {
           "Auth success": (obj) => obj.success === true,
         });

         // 加入房间
         socket.send(
           JSON.stringify({
             type: "room:join",
             roomId: "test-room",
           })
         );
       }
     });

     socket.on("close", () => console.log("连接已关闭"));
     socket.on("error", (e) => console.log("错误:", e));

     // 在连接中等待30秒
     socket.setTimeout(function () {
       socket.close();
     }, 30000);
   }
   ```

## 工作量估计

3 人天

## 技术关键点

1. 配置 Socket.IO 服务器集群，实现水平扩展
2. 优化 WebSocket 消息格式和传输效率
3. 实现有效的连接限流和负载控制机制
4. 设计合理的监控指标体系
5. 优化 Redis 连接和查询性能
6. 实现优雅的服务启动和关闭流程

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.2: WebSocket 服务](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Socket.IO 集群文档](https://socket.io/docs/v4/cluster-adapter/)
- [Redis 性能优化](https://redis.io/topics/benchmarks)
- [k6 负载测试工具](https://k6.io/docs/)
- [PM2 进程管理](https://pm2.keymetrics.io/docs/usage/cluster-mode/)

## 依赖关系

- 上游依赖:
  - Task 4.2.1: WebSocket 服务框架搭建
  - Task 4.2.2: 实时消息系统实现
  - Task 4.2.3: 房间与游戏状态同步
  - Task 4.2.4: 断线重连与状态恢复
- 下游依赖:
  - 无
