# Task 4.3.1: 微信登录与授权

## 任务描述

设计并实现阿瓦隆微信小游戏的登录与授权功能，包括小游戏登录流程、会话管理、token 生成与验证等，确保用户能够安全、便捷地登录游戏并授权使用其微信账号信息。

## 详细需求

### 登录流程设计

1. **客户端登录**

   - 实现微信小游戏客户端登录功能，调用 `wx.login()` 获取临时登录凭证（code）
   - 设计登录界面和登录状态管理
   - 处理登录过程中可能出现的异常情况

2. **服务端认证**

   - 开发服务端接口接收客户端发送的 code
   - 调用微信服务器的 `code2Session` 接口，获取用户的 `openid` 和 `session_key`
   - 实现 `unionid` 的获取和处理（若应用已关联到微信开放平台账号）

3. **会话管理**

   - 设计和实现基于 Redis 的会话存储机制
   - 生成安全的会话标识（token）并返回给客户端
   - 实现会话过期和自动续期机制
   - 处理多设备登录冲突问题

4. **身份验证中间件**
   - 开发用于验证请求合法性的中间件
   - 解析请求中的 token 并验证有效性
   - 在请求上下文中附加用户身份信息

### 授权管理

1. **用户授权界面**

   - 设计用户友好的授权界面
   - 实现用户授权状态管理
   - 支持授权范围的自定义配置

2. **授权信息获取**

   - 实现获取用户基本信息的授权流程
   - 支持按需获取用户昵称、头像等信息
   - 处理用户拒绝授权的情况

3. **授权状态管理**
   - 记录和管理用户的授权状态
   - 提供授权状态查询接口
   - 实现授权信息的更新和刷新机制

### 安全措施

1. **数据传输安全**

   - 实现 HTTPS 安全传输
   - 实现请求签名验证机制

2. **防刷和风控**

   - 实现登录频率限制
   - 设计异常登录检测机制
   - 实现 IP 黑名单功能

3. **敏感数据保护**
   - 安全存储 session_key
   - 对 openid 等敏感数据进行加密处理

## 接口设计

### 客户端接口

```typescript
// 登录接口
POST /api/wx/login
Request: { code: string }
Response: {
  token: string,
  expires: number,
  userInfo?: UserInfo
}

// 检查登录状态接口
GET /api/wx/checkLoginStatus
Headers: { Authorization: "Bearer {token}" }
Response: {
  isValid: boolean,
  expiresIn?: number
}

// 刷新token接口
POST /api/wx/refreshToken
Headers: { Authorization: "Bearer {token}" }
Response: {
  token: string,
  expires: number
}

// 登出接口
POST /api/wx/logout
Headers: { Authorization: "Bearer {token}" }
Response: {
  success: boolean
}
```

### 内部服务接口

```typescript
interface AuthService {
  // 验证用户登录凭证
  verifyLoginCode(code: string): Promise<WxSession>;

  // 生成用户token
  generateToken(openId: string, sessionKey: string): Promise<TokenInfo>;

  // 验证token有效性
  verifyToken(token: string): Promise<TokenVerificationResult>;

  // 刷新用户token
  refreshToken(oldToken: string): Promise<TokenInfo>;

  // 使token失效
  invalidateToken(token: string): Promise<boolean>;
}

interface TokenInfo {
  token: string;
  expires: number;
}

interface TokenVerificationResult {
  isValid: boolean;
  openId?: string;
  expiresIn?: number;
}

interface WxSession {
  openId: string;
  sessionKey: string;
  unionId?: string;
}
```

## 数据模型

### 会话数据模型（Redis）

```typescript
// Redis 会话存储结构
interface SessionRecord {
  openId: string;
  sessionKey: string;
  unionId?: string;
  expires: number;
  loginTime: number;
  loginIP: string;
  deviceInfo: string;
}

// Redis 键格式: `session:{token}`
```

### 用户授权数据模型（MongoDB）

```typescript
interface UserAuth {
  _id: string;
  openId: string;
  unionId?: string;
  authScope: string[]; // 用户授权的权限范围
  lastAuthTime: Date; // 最后授权时间
  authExpires?: Date; // 授权过期时间（如果适用）
  createdAt: Date;
  updatedAt: Date;
}
```

## 实现步骤

1. **设置微信小游戏开发环境**

   - 注册微信小游戏账号
   - 配置小游戏基本信息和域名白名单
   - 获取 AppID 和 AppSecret

2. **实现客户端登录**

   - 编写 wx.login() 调用代码
   - 实现登录状态和错误处理
   - 开发发送 code 到服务器的功能

3. **开发服务端登录接口**

   - 创建登录接口路由和控制器
   - 实现 code2Session 接口调用
   - 开发会话创建和 token 生成逻辑

4. **实现会话管理**

   - 配置 Redis 连接和操作封装
   - 实现会话存储和检索功能
   - 开发会话清理和过期处理机制

5. **开发身份验证中间件**

   - 编写 token 验证中间件
   - 实现用户信息注入请求上下文
   - 集成中间件到 API 路由系统

6. **实现授权功能**

   - 开发用户信息授权流程
   - 实现授权信息存储和查询
   - 编写授权状态管理逻辑

7. **添加安全措施**

   - 实现请求签名验证
   - 开发登录频率限制
   - 编写异常检测和防护代码

8. **进行测试**
   - 编写单元测试用例
   - 进行集成测试
   - 执行安全测试和压力测试

## 技术选型

1. **客户端技术**

   - 微信小游戏 API
   - TypeScript

2. **服务端技术**

   - NestJS 框架
   - TypeScript
   - JWT（JSON Web Token）
   - Redis（会话存储）
   - MongoDB（用户授权数据）

3. **安全工具**
   - bcrypt（密码哈希）
   - helmet（HTTP 安全头）
   - rate-limiter（请求限流）

## 测试要点

1. **功能测试**

   - 正常登录流程测试
   - 授权流程测试
   - 会话过期和续期测试
   - 多设备登录测试

2. **异常测试**

   - 无效 code 测试
   - 过期 token 测试
   - 网络异常情况测试
   - 微信服务不可用情况测试

3. **安全测试**
   - 请求伪造测试
   - 会话劫持尝试测试
   - 高频请求防护测试

## 验收标准

1. 用户能够通过微信快速登录游戏，无需额外注册流程
2. 登录过程安全可靠，采用 HTTPS 传输和数据加密
3. 会话管理正确实现，支持过期和自动续期
4. 能够正确处理用户授权，包括拒绝授权的情况
5. 支持多设备登录冲突的合理处理
6. 全部测试用例通过，包括功能测试、异常测试和安全测试
7. 代码符合项目编码规范，并通过代码审查

## 工作量估计

- 客户端登录实现：1 人天
- 服务端登录接口：1 人天
- 会话管理系统：1.5 人天
- 身份验证中间件：1 人天
- 授权功能：1 人天
- 安全措施：1.5 人天
- 测试和调优：1 人天

总计：**8 人天**

## 依赖关系

- 依赖 Task 4.1.1（架构设计与规划）
- 依赖 Task 4.1.2（核心服务框架搭建）
- 依赖 Task 4.1.3（数据库与缓存配置）
- 依赖 Task 4.1.4（认证与授权服务）

## 风险与应对

| 风险             | 可能性 | 影响 | 应对措施                                    |
| ---------------- | ------ | ---- | ------------------------------------------- |
| 微信登录接口变更 | 低     | 高   | 定期关注微信开发文档，设计松耦合架构        |
| 高并发登录压力   | 中     | 中   | 实现登录请求队列和限流机制，优化 Redis 访问 |
| 会话安全问题     | 低     | 高   | 实施严格的安全措施，定期进行安全审计        |
| 用户授权体验不佳 | 中     | 中   | 优化 UI 设计，提供明确的引导和说明          |

## 相关文档

- [微信小游戏登录流程官方文档](https://developers.weixin.qq.com/minigame/dev/guide/open-ability/login.html)
- [微信用户信息接口文档](https://developers.weixin.qq.com/minigame/dev/api/open-api/user-info/wx.getUserInfo.html)
- [技术方案文档](./技术方案.md)
- [NestJS 身份验证文档](https://docs.nestjs.com/security/authentication)
