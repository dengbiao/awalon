# Task 4.3.2: 用户信息处理

## 任务描述

设计并实现阿瓦隆微信小游戏的用户信息处理功能，包括微信用户信息的获取、解密、存储和管理，确保游戏能够安全地获取和使用用户信息，提供个性化的游戏体验。

## 详细需求

### 用户信息获取

1. **基础信息获取**

   - 实现从微信获取用户基础信息的功能
   - 支持获取用户昵称、头像、性别等基本信息
   - 处理微信隐私保护新规下的用户信息获取限制

2. **敏感信息处理**

   - 实现微信加密数据的解密功能，使用 session_key 解密
   - 支持获取用户手机号等敏感信息（如游戏需要）
   - 确保敏感信息处理符合微信平台规范和隐私政策

3. **用户标识管理**
   - 管理用户的 OpenID 和 UnionID（如有）
   - 实现用户标识与游戏内用户账号的绑定
   - 处理跨平台用户标识的统一问题

### 用户信息存储

1. **数据模型设计**

   - 设计合理的用户数据模型，包含必要的微信用户信息
   - 支持用户信息的增删改查操作
   - 实现用户数据的版本控制

2. **数据持久化**

   - 实现用户信息在 MongoDB 中的存储与检索
   - 设计高效的用户数据索引
   - 实现数据一致性保证机制

3. **缓存策略**
   - 设计用户信息缓存策略，减少数据库访问
   - 实现基于 Redis 的用户信息缓存
   - 处理缓存与数据库同步问题

### 用户信息更新

1. **信息更新机制**

   - 实现用户信息定期更新机制
   - 支持用户主动更新个人信息
   - 处理微信用户信息变更的同步问题

2. **数据迁移与合并**
   - 设计用户数据迁移机制
   - 支持不同设备上用户数据的合并
   - 处理用户数据冲突解决策略

### 隐私保护

1. **隐私政策实现**

   - 确保用户信息处理符合相关法律法规
   - 实现用户数据脱敏和隐私保护机制
   - 设计用户数据使用授权流程

2. **数据安全措施**
   - 实现用户数据加密存储
   - 设计数据访问控制策略
   - 实现敏感操作的日志审计

## 接口设计

### 客户端接口

```typescript
// 更新用户信息接口
POST /api/wx/updateUserInfo
Headers: { Authorization: "Bearer {token}" }
Request: {
  encryptedData?: string,
  iv?: string,
  userInfo?: WxUserInfo  // 明文用户信息（新版微信API）
}
Response: {
  success: boolean,
  userInfo: UserInfo
}

// 获取用户信息接口
GET /api/wx/getUserInfo
Headers: { Authorization: "Bearer {token}" }
Response: {
  userInfo: UserInfo
}

// 用户手机号获取接口
POST /api/wx/getPhoneNumber
Headers: { Authorization: "Bearer {token}" }
Request: {
  encryptedData: string,
  iv: string
}
Response: {
  success: boolean,
  phoneNumber?: string
}
```

### 内部服务接口

```typescript
interface UserService {
  // 通过OpenID获取用户
  getUserByOpenId(openId: string): Promise<User>;

  // 创建或更新用户信息
  createOrUpdateUser(
    openId: string,
    userInfo: Partial<UserInfo>
  ): Promise<User>;

  // 解密微信加密数据
  decryptWxData(
    sessionKey: string,
    encryptedData: string,
    iv: string
  ): Promise<any>;

  // 获取用户手机号
  getPhoneNumber(
    openId: string,
    encryptedData: string,
    iv: string
  ): Promise<string>;

  // 删除用户数据
  deleteUserData(openId: string): Promise<boolean>;
}
```

## 数据模型

### 用户基本信息模型

```typescript
interface User {
  _id: string;
  openId: string; // 微信OpenID
  unionId?: string; // 微信UnionID（如有）
  gameId?: string; // 游戏内用户ID
  nickname?: string; // 用户昵称
  avatarUrl?: string; // 用户头像URL
  gender?: number; // 性别：0-未知，1-男，2-女
  country?: string; // 国家
  province?: string; // 省份
  city?: string; // 城市
  language?: string; // 语言
  phoneNumber?: string; // 手机号（加密存储）
  registrationDate: Date; // 注册日期
  lastLoginDate: Date; // 最后登录日期
  userSettings: UserSettings; // 用户设置
  gameStats: GameStats; // 游戏统计数据
  createdAt: Date;
  updatedAt: Date;
}

interface UserSettings {
  notification: boolean; // 通知开关
  sound: boolean; // 声音开关
  vibration: boolean; // 震动开关
  theme?: string; // 主题设置
  language?: string; // 语言设置
}

interface GameStats {
  totalGames: number; // 总游戏局数
  winGames: number; // 胜利局数
  goodRoleGames: number; // 扮演好人角色局数
  evilRoleGames: number; // 扮演坏人角色局数
  favoriteRole?: string; // 最常扮演角色
  achievementIds: string[]; // 已解锁成就ID列表
}
```

### 用户数据操作日志模型

```typescript
interface UserDataLog {
  _id: string;
  userId: string;
  openId: string;
  operation: "create" | "update" | "delete" | "decrypt";
  fields?: string[]; // 操作涉及的字段
  operatorId?: string; // 操作者ID（系统操作为null）
  ipAddress?: string; // 操作IP地址
  deviceInfo?: string; // 设备信息
  timestamp: Date;
}
```

## 实现步骤

1. **设计用户数据模型**

   - 定义 MongoDB 用户集合结构
   - 设计用户数据索引
   - 实现数据模型验证逻辑

2. **实现用户信息获取**

   - 开发微信用户信息获取功能
   - 实现加密数据解密功能
   - 编写用户信息处理和格式化逻辑

3. **开发数据持久化功能**

   - 实现用户数据存储和读取
   - 开发数据一致性保证机制
   - 实现数据版本控制

4. **实现缓存策略**

   - 配置 Redis 缓存
   - 开发用户数据缓存管理
   - 实现缓存更新和失效策略

5. **开发信息更新机制**

   - 实现用户信息更新接口
   - 开发定期同步机制
   - 编写数据迁移和合并逻辑

6. **实现隐私保护措施**

   - 开发数据脱敏功能
   - 实现数据访问控制
   - 建立操作日志审计系统

7. **进行测试**
   - 编写单元测试
   - 进行集成测试
   - 执行安全和性能测试

## 技术选型

1. **数据存储**

   - MongoDB（用户数据持久化）
   - Redis（用户数据缓存）

2. **后端技术**

   - NestJS 框架
   - TypeScript
   - Mongoose（MongoDB ODM）
   - Redis 客户端（ioredis）

3. **安全工具**
   - crypto（数据加密解密）
   - mongo-sanitize（防注入）
   - class-validator（数据验证）

## 测试要点

1. **功能测试**

   - 用户信息获取测试
   - 用户数据存储和检索测试
   - 信息更新和同步测试
   - 缓存机制测试

2. **异常测试**

   - 无效数据处理测试
   - 数据解密失败处理测试
   - 网络异常情况测试
   - 数据库连接失败测试

3. **安全测试**

   - 数据加密测试
   - 访问控制测试
   - 数据泄露防护测试
   - 隐私合规测试

4. **性能测试**
   - 高并发数据访问测试
   - 大数据量处理测试
   - 缓存性能测试

## 验收标准

1. 能够安全地获取和处理微信用户信息，符合微信平台规范
2. 用户数据存储和检索功能正常，支持完整的增删改查操作
3. 缓存策略有效，能够提高数据访问性能
4. 用户信息更新机制可靠，能够保持数据最新状态
5. 数据安全措施完善，符合隐私保护要求
6. 全部测试用例通过，包括功能、异常、安全和性能测试
7. 代码符合项目编码规范，并通过代码审查

## 工作量估计

- 用户数据模型设计：0.5 人天
- 用户信息获取实现：1 人天
- 数据持久化功能：1 人天
- 缓存策略实现：1 人天
- 信息更新机制：1 人天
- 隐私保护措施：1 人天
- 测试和调优：1.5 人天

总计：**7 人天**

## 依赖关系

- 依赖 Task 4.1.3（数据库与缓存配置）
- 依赖 Task 4.3.1（微信登录与授权）

## 风险与应对

| 风险                   | 可能性 | 影响 | 应对措施                                         |
| ---------------------- | ------ | ---- | ------------------------------------------------ |
| 微信用户隐私政策变更   | 中     | 高   | 密切关注微信政策更新，设计灵活的用户信息获取机制 |
| 用户数据安全问题       | 低     | 高   | 实施严格的数据加密和访问控制，定期安全审计       |
| 高并发数据访问性能问题 | 中     | 中   | 优化数据库设计和索引，实现有效的缓存策略         |
| 数据一致性问题         | 中     | 中   | 实现数据版本控制和冲突解决机制，采用事务处理     |

## 相关文档

- [微信用户信息接口文档](https://developers.weixin.qq.com/minigame/dev/api/open-api/user-info/wx.getUserInfo.html)
- [微信数据解密文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/signature.html)
- [MongoDB 数据模型设计最佳实践](https://docs.mongodb.com/manual/core/data-model-design/)
- [Redis 缓存策略](https://redis.io/topics/lru-cache)
- [NestJS 与 MongoDB 集成文档](https://docs.nestjs.com/techniques/mongodb)
