# Task 4.1.3: 数据库与缓存配置

## 描述

配置和实现阿瓦隆微信小游戏的数据库和缓存系统，包括 MongoDB 数据库连接、模式定义和索引优化，以及 Redis 缓存服务的配置和数据存储策略。

## 验收标准

1. 完成 MongoDB 连接配置和集成
2. 实现主要数据模型和模式定义(Schema)
3. 设计并创建必要的索引
4. 实现 MongoDB 数据访问服务
5. 配置 Redis 连接和会话存储
6. 实现 Redis 缓存服务
7. 实现数据库和缓存的健康检查
8. 编写数据迁移和种子脚本
9. 完成数据备份和恢复策略文档

## 详细内容

### MongoDB 配置与模型定义

1. **数据库连接配置**

   - 使用 Mongoose 集成 MongoDB
   - 配置连接池和超时设置
   - 实现重试和失败处理逻辑
   - 支持 MongoDB 副本集连接

   ```typescript
   // database.module.ts 示例
   @Module({
     imports: [
       MongooseModule.forRootAsync({
         imports: [ConfigModule],
         inject: [ConfigService],
         useFactory: async (configService: ConfigService) => ({
           uri: configService.get<string>("MONGODB_URI"),
           useNewUrlParser: true,
           useUnifiedTopology: true,
           maxPoolSize: 10,
           socketTimeoutMS: 45000,
           serverSelectionTimeoutMS: 5000,
           retryWrites: true,
         }),
       }),
     ],
     exports: [MongooseModule],
   })
   export class DatabaseModule {}
   ```

2. **核心数据模型定义**

   - **用户模型**

     ```typescript
     // user.schema.ts
     @Schema({ timestamps: true })
     export class User {
       @Prop({ required: true, unique: true })
       openId: string;

       @Prop({ required: true })
       nickname: string;

       @Prop()
       avatarUrl: string;

       @Prop({ default: 0 })
       gender: number;

       @Prop({ default: Date.now })
       lastLogin: Date;

       @Prop({ type: Object, default: {} })
       gameStats: Record<string, any>;
     }

     export const UserSchema = SchemaFactory.createForClass(User);
     ```

   - **房间模型**

     ```typescript
     // room.schema.ts
     @Schema({ timestamps: true })
     export class Room {
       @Prop({ required: true, unique: true })
       roomCode: string;

       @Prop({
         type: mongoose.Schema.Types.ObjectId,
         ref: "User",
         required: true,
       })
       creatorId: User;

       @Prop({ enum: ["waiting", "playing", "finished"], default: "waiting" })
       status: string;

       @Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: "User" }] })
       players: User[];

       @Prop({ type: Object, default: {} })
       settings: Record<string, any>;

       @Prop({ type: mongoose.Schema.Types.ObjectId, ref: "Game" })
       currentGame: Game;
     }

     export const RoomSchema = SchemaFactory.createForClass(Room);
     ```

   - **游戏模型**

     ```typescript
     // game.schema.ts
     @Schema({ timestamps: true })
     export class Game {
       @Prop({
         type: mongoose.Schema.Types.ObjectId,
         ref: "Room",
         required: true,
       })
       roomId: Room;

       @Prop({ type: [Object], required: true })
       players: Record<string, any>[];

       @Prop({ type: Object, required: true })
       roles: Record<string, any>;

       @Prop({ type: [Object], default: [] })
       missions: Record<string, any>[];

       @Prop({ type: [Object], default: [] })
       votes: Record<string, any>[];

       @Prop({ type: Object })
       result: Record<string, any>;

       @Prop()
       startTime: Date;

       @Prop()
       endTime: Date;
     }

     export const GameSchema = SchemaFactory.createForClass(Game);
     ```

3. **索引设计与创建**

   - 为频繁查询的字段创建索引
   - 为关联查询创建复合索引
   - 设置 TTL 索引用于过期数据自动清理

   ```typescript
   // 示例索引配置
   UserSchema.index({ openId: 1 });
   RoomSchema.index({ roomCode: 1 });
   RoomSchema.index({ status: 1, createdAt: -1 });
   GameSchema.index({ roomId: 1, startTime: -1 });
   ```

4. **数据访问服务**
   - 实现基础 CRUD 操作服务
   - 实现针对特定业务场景的数据查询方法
   - 实现事务管理

### Redis 配置与缓存策略

1. **Redis 连接配置**

   - 使用 IORedis 客户端
   - 配置连接池和重试策略
   - 支持 Redis 集群或哨兵模式连接

   ```typescript
   // redis.module.ts 示例
   @Module({
     imports: [ConfigModule],
     providers: [
       {
         provide: "REDIS_CLIENT",
         useFactory: (configService: ConfigService) => {
           return new Redis({
             host: configService.get("REDIS_HOST"),
             port: configService.get("REDIS_PORT"),
             password: configService.get("REDIS_PASSWORD"),
             db: configService.get("REDIS_DB"),
             retryStrategy: (times) => Math.min(times * 50, 2000),
           });
         },
         inject: [ConfigService],
       },
       RedisService,
     ],
     exports: ["REDIS_CLIENT", RedisService],
   })
   export class RedisModule {}
   ```

2. **会话管理配置**

   - 配置基于 Redis 的会话存储
   - 设置会话过期和清理策略
   - 实现会话状态管理服务

3. **缓存服务实现**

   - 开发通用缓存工具服务
   - 实现缓存键管理和命名空间
   - 实现数据序列化和反序列化
   - 实现 TTL 和缓存失效策略

   ```typescript
   // cache.service.ts 示例
   @Injectable()
   export class CacheService {
     constructor(
       @Inject("REDIS_CLIENT")
       private readonly redisClient: Redis
     ) {}

     async get<T>(key: string): Promise<T | null> {
       const data = await this.redisClient.get(key);
       if (!data) return null;
       return JSON.parse(data);
     }

     async set(key: string, value: any, ttl?: number): Promise<void> {
       const serialized = JSON.stringify(value);
       if (ttl) {
         await this.redisClient.set(key, serialized, "EX", ttl);
       } else {
         await this.redisClient.set(key, serialized);
       }
     }

     // 其他方法: del, exists, ttl, keys, etc.
   }
   ```

4. **游戏状态缓存实现**
   - 设计游戏状态在 Redis 中的存储结构
   - 实现实时游戏状态更新
   - 实现客户端订阅和状态同步机制

### 数据库管理工具

1. **数据迁移脚本**

   - 使用 Mongoose Migrations 或自定义脚本
   - 实现版本控制和前向/后向迁移
   - 集成到部署流程

2. **数据种子脚本**

   - 创建初始数据填充脚本
   - 实现测试和开发环境数据准备

3. **数据备份和恢复**
   - 设计自动备份策略
   - 实现增量备份机制
   - 记录备份恢复流程文档

## 工作量估计

3 人天

## 技术关键点

1. 确保数据库连接的稳定性和弹性
2. 设计高效的索引策略以优化查询性能
3. 实现合理的缓存策略以减轻数据库负载
4. 确保数据一致性和完整性
5. 设计高效的游戏状态存储结构

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [MongoDB 文档](https://docs.mongodb.com/)
- [Mongoose 文档](https://mongoosejs.com/docs/)
- [Redis 文档](https://redis.io/documentation)
- [IORedis 文档](https://github.com/luin/ioredis#readme)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
  - Task 4.1.2: 核心服务框架搭建
- 下游依赖:
  - Task 4.1.4: 认证与授权服务
  - Task 4.1.5: 日志与监控系统
  - Task 4.1.6: 服务部署与配置
