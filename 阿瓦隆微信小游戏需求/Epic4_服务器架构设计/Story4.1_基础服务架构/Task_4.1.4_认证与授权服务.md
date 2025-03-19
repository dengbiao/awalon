# Task 4.1.4: 认证与授权服务

## 描述

设计并实现阿瓦隆微信小游戏的认证与授权系统，包括微信用户登录认证、JWT 令牌管理、会话处理、权限控制，以及用户身份验证相关的安全措施。

## 验收标准

1. 完成微信登录认证服务的实现
2. 实现 JWT 令牌生成、验证和刷新功能
3. 完成基于 Redis 的会话管理系统
4. 实现适用于 HTTP 和 WebSocket 的认证中间件
5. 开发基于角色的访问控制(RBAC)机制
6. 实现防止 CSRF、会话劫持等安全措施
7. 添加速率限制机制防止暴力攻击
8. 完成认证相关的日志记录和安全审计
9. 提供用户会话管理和退出功能

## 详细内容

### 微信认证服务

1. **微信小游戏登录流程**

   - 实现基于 code 的临时登录码验证
   - 请求微信服务器获取 openId 和会话密钥
   - 处理登录态校验和信息解密
   - 支持静默登录和用户信息授权

   ```typescript
   // wechat-auth.service.ts 示例
   @Injectable()
   export class WechatAuthService {
     constructor(
       private readonly httpService: HttpService,
       private readonly configService: ConfigService,
       private readonly userService: UserService
     ) {}

     async login(
       code: string
     ): Promise<{ openId: string; session_key: string }> {
       const appId = this.configService.get("WECHAT_APPID");
       const secret = this.configService.get("WECHAT_SECRET");
       const url = `https://api.weixin.qq.com/sns/jscode2session?appid=${appId}&secret=${secret}&js_code=${code}&grant_type=authorization_code`;

       const response = await this.httpService.get(url).toPromise();
       const { openid, session_key, errcode, errmsg } = response.data;

       if (errcode) {
         throw new UnauthorizedException(`微信认证失败: ${errmsg}`);
       }

       return { openId: openid, session_key };
     }

     async decryptUserInfo(
       encryptedData: string,
       iv: string,
       sessionKey: string
     ): Promise<any> {
       // 实现微信数据解密逻辑
       // ...
     }
   }
   ```

2. **用户管理与注册**
   - 基于 openId 查询或创建用户记录
   - 更新用户基本信息
   - 处理用户首次登录的初始化

### JWT 认证系统

1. **JWT 策略实现**

   - 使用@nestjs/jwt 和 Passport 实现 JWT 认证
   - 设计 JWT 载荷结构和签名算法
   - 配置令牌过期时间和刷新策略

   ```typescript
   // jwt.strategy.ts 示例
   @Injectable()
   export class JwtStrategy extends PassportStrategy(Strategy) {
     constructor(private readonly configService: ConfigService) {
       super({
         jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
         ignoreExpiration: false,
         secretOrKey: configService.get("JWT_SECRET"),
       });
     }

     async validate(payload: any) {
       return { userId: payload.sub, openId: payload.openId };
     }
   }
   ```

2. **令牌生成与管理**

   - 实现令牌生成服务
   - 处理令牌刷新逻辑
   - 实现令牌撤销机制

   ```typescript
   // auth.service.ts 示例
   @Injectable()
   export class AuthService {
     constructor(
       private readonly jwtService: JwtService,
       private readonly redisService: RedisService
     ) {}

     async generateTokens(
       userId: string,
       openId: string
     ): Promise<{ accessToken: string; refreshToken: string }> {
       const payload = { sub: userId, openId };

       const accessToken = this.jwtService.sign(payload, {
         expiresIn: "1h",
       });

       const refreshToken = this.jwtService.sign(
         { ...payload, type: "refresh" },
         { expiresIn: "7d" }
       );

       // 存储刷新令牌到Redis
       await this.redisService.set(
         `refresh_token:${userId}`,
         refreshToken,
         60 * 60 * 24 * 7
       );

       return { accessToken, refreshToken };
     }

     async refreshToken(
       refreshToken: string
     ): Promise<{ accessToken: string }> {
       try {
         // 验证刷新令牌
         const payload = this.jwtService.verify(refreshToken);
         const storedToken = await this.redisService.get(
           `refresh_token:${payload.sub}`
         );

         if (!storedToken || storedToken !== refreshToken) {
           throw new UnauthorizedException("刷新令牌无效");
         }

         // 生成新的访问令牌
         const newAccessToken = this.jwtService.sign(
           { sub: payload.sub, openId: payload.openId },
           { expiresIn: "1h" }
         );

         return { accessToken: newAccessToken };
       } catch (e) {
         throw new UnauthorizedException("刷新令牌失效或已过期");
       }
     }

     async revokeTokens(userId: string): Promise<void> {
       await this.redisService.del(`refresh_token:${userId}`);
     }
   }
   ```

### 会话管理

1. **基于 Redis 的会话存储**

   - 设计会话数据结构
   - 实现会话创建、验证和过期处理
   - 支持多设备登录管理

2. **会话中间件**
   - 开发认证和会话处理中间件
   - 在请求处理管道中集成会话验证
   - 实现会话状态的传递和使用

### WebSocket 认证

1. **WebSocket 连接认证**

   - 实现基于 JWT 的 WebSocket 连接认证
   - 处理 Socket.IO 握手阶段的令牌验证
   - 将用户身份与 WebSocket 连接关联

   ```typescript
   // websocket-auth.guard.ts 示例
   @Injectable()
   export class WsAuthGuard implements CanActivate {
     constructor(private readonly jwtService: JwtService) {}

     canActivate(context: ExecutionContext): boolean | Promise<boolean> {
       const client = context.switchToWs().getClient();
       const handshake = client.handshake || {};
       const { headers = {}, query = {} } = handshake;

       // 从请求中提取令牌
       const token = headers.authorization || query.token;

       if (!token) {
         return false;
       }

       try {
         // 验证JWT令牌
         const payload = this.jwtService.verify(token);
         // 将用户信息附加到socket对象
         client.user = { userId: payload.sub, openId: payload.openId };
         return true;
       } catch (e) {
         return false;
       }
     }
   }
   ```

2. **连接状态管理**
   - 跟踪用户的 WebSocket 连接状态
   - 处理断线重连的身份验证
   - 实现会话超时和自动断开逻辑

### 安全措施

1. **防 CSRF 保护**

   - 实现 CSRF 令牌验证
   - 为敏感操作添加额外验证

2. **速率限制**

   - 实现基于 IP 和用户 ID 的请求限流
   - 为敏感 API 添加特定的速率限制策略
   - 配置限制失败处理和封禁机制

   ```typescript
   // rate-limit.guard.ts 示例
   @Injectable()
   export class RateLimitGuard implements CanActivate {
     constructor(
       private readonly redisService: RedisService,
       private readonly configService: ConfigService
     ) {}

     async canActivate(context: ExecutionContext): Promise<boolean> {
       const request = context.switchToHttp().getRequest();
       const ip = request.ip;
       const userId = request.user?.userId || "anonymous";

       const ipKey = `ratelimit:ip:${ip}`;
       const userKey = `ratelimit:user:${userId}`;

       // 检查IP限制
       const ipHits = await this.redisService.incr(ipKey);
       if (ipHits === 1) {
         await this.redisService.expire(ipKey, 60); // 60秒过期
       }

       // 检查用户限制
       const userHits = await this.redisService.incr(userKey);
       if (userHits === 1) {
         await this.redisService.expire(userKey, 60); // 60秒过期
       }

       // 检查是否超出限制
       const ipLimit = this.configService.get("RATE_LIMIT_IP", 60);
       const userLimit = this.configService.get("RATE_LIMIT_USER", 30);

       if (ipHits > ipLimit || userHits > userLimit) {
         throw new HttpException(
           "请求过于频繁，请稍后再试",
           HttpStatus.TOO_MANY_REQUESTS
         );
       }

       return true;
     }
   }
   ```

3. **安全头部配置**

   - 集成 Helmet 中间件
   - 配置内容安全策略(CSP)
   - 设置 XSS 防护头部

4. **安全日志和审计**
   - 记录敏感操作的审计日志
   - 实现安全事件通知机制
   - 开发安全仪表板用于监控

## 工作量估计

3 人天

## 技术关键点

1. 确保微信登录认证流程的安全性和稳定性
2. 设计合理的 JWT 策略，平衡安全性和用户体验
3. 实现高效的 Redis 会话管理系统
4. 添加适当的安全防护措施，不过度影响性能
5. 保证 WebSocket 连接的安全认证机制

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [微信小游戏登录文档](https://developers.weixin.qq.com/minigame/dev/guide/open-ability/login.html)
- [NestJS 认证文档](https://docs.nestjs.com/security/authentication)
- [JWT 认证最佳实践](https://auth0.com/blog/a-complete-guide-to-jwt-authentication/)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
  - Task 4.1.2: 核心服务框架搭建
  - Task 4.1.3: 数据库与缓存配置
- 下游依赖:
  - Task 4.1.5: 日志与监控系统
  - Task 4.1.6: 服务部署与配置
  - Story 4.3: 微信接入服务
