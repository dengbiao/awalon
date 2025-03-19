# Task 6.1.3: 用户数据访问 API

## 任务描述

为阿瓦隆微信小游戏设计和实现一套完整的用户数据访问 API，提供标准化的 REST 接口，用于用户个人资料管理、游戏统计数据查询和用户设置管理，确保数据访问的安全性、高效性和一致性。

## 详细要求

### 1. 用户个人资料 API

1. **基本信息管理**

   - 获取用户个人资料 (GET)
   - 更新用户基本信息 (PATCH)
   - 上传/更新用户头像 (POST)
   - 删除用户账号 (DELETE)

2. **用户状态管理**

   - 获取用户在线状态
   - 更新用户状态 (在线/离线/游戏中)
   - 用户登录历史查询
   - 设备管理与绑定

3. **账号验证功能**
   - 验证用户信息
   - 绑定/解绑手机号
   - 更改用户标识符
   - 账号恢复功能 API

### 2. 游戏统计数据 API

1. **个人游戏数据**

   - 获取游戏总览统计
   - 查询特定角色使用数据
   - 获取胜率和游戏场次信息
   - 查询游戏趋势数据

2. **排行榜数据**

   - 获取总体排行榜数据
   - 获取好友排行榜数据
   - 获取特定指标的排名数据
   - 获取历史最佳记录

3. **游戏历史记录**
   - 获取最近游戏记录
   - 查询历史游戏详情
   - 获取游戏记录汇总分析
   - 筛选特定条件的游戏记录

### 3. 用户设置管理 API

1. **游戏偏好设置**

   - 获取用户设置配置
   - 更新游戏设置
   - 重置默认设置
   - 同步设置到新设备

2. **隐私设置**

   - 获取隐私设置状态
   - 更新隐私选项
   - 数据分享偏好设置
   - 社交功能开关设置

3. **通知设置**
   - 获取通知配置
   - 更新通知偏好
   - 设置免打扰时间段
   - 通知渠道管理

### 4. 社交关系 API

1. **好友管理**

   - 获取好友列表
   - 添加/删除好友
   - 查询好友状态
   - 好友分组管理

2. **社交互动**
   - 获取社交动态
   - 发送/接收游戏邀请
   - 获取最近互动玩家
   - 社交关系推荐

## 技术细节

### 1. API 设计规范

1. **RESTful API 设计原则**

   - 使用 HTTP 方法表示操作类型(GET/POST/PATCH/DELETE)
   - 资源路径使用名词而非动词
   - 使用复数形式表示资源集合
   - 支持 HATEOAS 原则，通过链接导航 API

2. **请求/响应格式**

   - 请求参数：URL 参数、Query 参数、Body 参数规范
   - 响应格式：统一的 JSON 响应结构
   - 错误处理：标准 HTTP 状态码和错误信息格式
   - 分页、排序、过滤的标准实现方式

3. **API 版本控制**
   - URL 路径版本控制(如/api/v1/users)
   - 请求头版本控制
   - 版本兼容性策略
   - API 弃用和迁移策略

### 2. API 文档示例

以下是一些核心 API 的文档示例：

#### 获取用户个人资料

```
GET /api/v1/users/{userId}

描述: 获取指定用户的详细个人资料信息

请求参数:
- 路径参数:
  - userId: string (必需) - 用户ID

响应:
200 OK
{
  "id": "user_12345",
  "nickname": "游戏玩家",
  "avatarUrl": "https://example.com/avatars/user_12345.jpg",
  "level": 8,
  "reputation": 92,
  "createdAt": "2023-01-15T08:30:00Z",
  "lastLoginAt": "2023-05-20T14:45:23Z",
  "region": "北京",
  "status": "online",
  "gameStats": {
    "totalGames": 75,
    "winRate": 0.56,
    "goodSideRate": 0.6,
    "evilSideRate": 0.4
  }
}

错误:
- 401 Unauthorized: 未经授权访问
- 403 Forbidden: 无权访问此用户资料
- 404 Not Found: 指定用户不存在
```

#### 更新用户游戏设置

```
PATCH /api/v1/users/{userId}/settings/game

描述: 更新用户的游戏偏好设置

请求参数:
- 路径参数:
  - userId: string (必需) - 用户ID
- 请求体:
  {
    "soundEnabled": true,
    "musicVolume": 0.8,
    "sfxVolume": 0.7,
    "language": "zh_CN",
    "themeMode": "dark",
    "autoReady": false,
    "showTutorialTips": true
  }

响应:
200 OK
{
  "id": "settings_89012",
  "userId": "user_12345",
  "soundEnabled": true,
  "musicVolume": 0.8,
  "sfxVolume": 0.7,
  "language": "zh_CN",
  "themeMode": "dark",
  "autoReady": false,
  "showTutorialTips": true,
  "updatedAt": "2023-06-10T09:15:28Z"
}

错误:
- 400 Bad Request: 设置参数无效
- 401 Unauthorized: 未经授权操作
- 403 Forbidden: 无权修改此用户设置
- 404 Not Found: 用户不存在
```

#### 获取用户游戏统计数据

```
GET /api/v1/users/{userId}/game-stats

描述: 获取用户的详细游戏统计数据

请求参数:
- 路径参数:
  - userId: string (必需) - 用户ID
- 查询参数:
  - period: string (可选) - 时间段筛选 (all, last7days, last30days, thisMonth)
  - roleType: string (可选) - 角色类型筛选 (good, evil, specific)
  - roleName: string (可选) - 特定角色名称

响应:
200 OK
{
  "userId": "user_12345",
  "totalGames": 75,
  "winCount": 42,
  "winRate": 0.56,
  "goodSideGames": {
    "count": 45,
    "wins": 28,
    "winRate": 0.62
  },
  "evilSideGames": {
    "count": 30,
    "wins": 14,
    "winRate": 0.47
  },
  "roleStats": [
    {
      "role": "merlin",
      "count": 12,
      "wins": 9,
      "winRate": 0.75
    },
    {
      "role": "assassin",
      "count": 8,
      "wins": 5,
      "winRate": 0.63
    },
    // 其他角色统计
  ],
  "missionStats": {
    "success": 35,
    "fail": 25,
    "successRate": 0.58
  },
  "voteAccuracy": 0.82,
  "avgGameDuration": 22.5
}

错误:
- 401 Unauthorized: 未经授权访问
- 403 Forbidden: 无权访问此用户统计数据
- 404 Not Found: 用户不存在
```

### 3. API 实现示例

以下是一个用户数据访问 API 的 Express 路由实现示例：

```typescript
// 用户数据访问路由
import { Router } from "express";
import { AuthMiddleware } from "../middlewares/auth.middleware";
import { UserController } from "../controllers/user.controller";
import { UserStatsController } from "../controllers/user-stats.controller";
import { UserSettingsController } from "../controllers/user-settings.controller";
import { FriendController } from "../controllers/friend.controller";
import { validate } from "../middlewares/validation.middleware";
import {
  getUserProfileSchema,
  updateUserProfileSchema,
  updateGameSettingsSchema,
  updatePrivacySettingsSchema,
} from "../schemas/user.schema";

const router = Router();
const userController = new UserController();
const userStatsController = new UserStatsController();
const userSettingsController = new UserSettingsController();
const friendController = new FriendController();

// 用户个人资料API
router.get(
  "/users/me",
  AuthMiddleware.verifyToken,
  userController.getCurrentUserProfile
);

router.get(
  "/users/:userId",
  AuthMiddleware.verifyToken,
  validate(getUserProfileSchema),
  userController.getUserProfile
);

router.patch(
  "/users/me",
  AuthMiddleware.verifyToken,
  validate(updateUserProfileSchema),
  userController.updateCurrentUserProfile
);

router.post(
  "/users/me/avatar",
  AuthMiddleware.verifyToken,
  userController.uploadAvatar
);

// 游戏统计数据API
router.get(
  "/users/:userId/game-stats",
  AuthMiddleware.verifyToken,
  userStatsController.getUserGameStats
);

router.get(
  "/users/:userId/game-stats/roles/:roleName",
  AuthMiddleware.verifyToken,
  userStatsController.getRoleSpecificStats
);

router.get(
  "/users/:userId/game-history",
  AuthMiddleware.verifyToken,
  userStatsController.getGameHistory
);

router.get(
  "/rankings",
  AuthMiddleware.verifyToken,
  userStatsController.getGlobalRankings
);

router.get(
  "/rankings/friends",
  AuthMiddleware.verifyToken,
  userStatsController.getFriendsRankings
);

// 用户设置管理API
router.get(
  "/users/:userId/settings",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  userSettingsController.getAllSettings
);

router.get(
  "/users/:userId/settings/game",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  userSettingsController.getGameSettings
);

router.patch(
  "/users/:userId/settings/game",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  validate(updateGameSettingsSchema),
  userSettingsController.updateGameSettings
);

router.patch(
  "/users/:userId/settings/privacy",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  validate(updatePrivacySettingsSchema),
  userSettingsController.updatePrivacySettings
);

router.post(
  "/users/:userId/settings/reset",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  userSettingsController.resetToDefaultSettings
);

// 好友管理API
router.get(
  "/users/:userId/friends",
  AuthMiddleware.verifyToken,
  friendController.getFriendsList
);

router.post(
  "/users/:userId/friends/:friendId",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  friendController.addFriend
);

router.delete(
  "/users/:userId/friends/:friendId",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  friendController.removeFriend
);

router.get(
  "/users/:userId/friend-requests",
  AuthMiddleware.verifyToken,
  AuthMiddleware.isResourceOwner("userId"),
  friendController.getFriendRequests
);

export default router;
```

### 4. API 测试与文档生成

```typescript
// 使用Jest和SuperTest进行API测试示例
import request from "supertest";
import app from "../app";
import { generateUserToken } from "../test/helpers";
import { connectTestDatabase, closeTestDatabase } from "../test/db";

describe("User Profile API", () => {
  let testUserToken: string;
  let testUserId: string;

  beforeAll(async () => {
    await connectTestDatabase();
    // 创建测试用户并生成令牌
    testUserId = "user_test_12345";
    testUserToken = await generateUserToken(testUserId);
  });

  afterAll(async () => {
    await closeTestDatabase();
  });

  it("should get current user profile", async () => {
    const response = await request(app)
      .get("/api/v1/users/me")
      .set("Authorization", `Bearer ${testUserToken}`);

    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty("id");
    expect(response.body).toHaveProperty("nickname");
    expect(response.body).toHaveProperty("avatarUrl");
  });

  it("should update user profile", async () => {
    const updateData = {
      nickname: "新的昵称",
      region: "上海",
    };

    const response = await request(app)
      .patch("/api/v1/users/me")
      .set("Authorization", `Bearer ${testUserToken}`)
      .send(updateData);

    expect(response.status).toBe(200);
    expect(response.body.nickname).toBe(updateData.nickname);
    expect(response.body.region).toBe(updateData.region);
  });

  it("should return 401 without token", async () => {
    const response = await request(app).get("/api/v1/users/me");

    expect(response.status).toBe(401);
  });
});
```

## 验收标准

1. 所有 API 端点按照 RESTful 设计原则实现并满足功能需求
2. API 响应时间符合要求：GET 操作<50ms，写操作<100ms
3. 所有 API 都有完善的参数验证和错误处理
4. API 文档完整，包含每个端点的详细说明、参数要求和响应格式
5. 单元测试覆盖率>85%，所有核心功能都有测试用例
6. 功能测试确认所有 API 在实际场景中能正常工作
7. 安全测试验证所有 API 都有适当的访问控制和权限验证
8. 所有 API 遵循数据隐私保护原则，敏感信息加密传输
9. 负载测试验证 API 在高并发条件下仍能稳定运行
10. API 支持版本控制，能够平滑升级和兼容旧版本

## 依赖关系

- 依赖 Task6.1.1 的用户数据模型设计
- 依赖 Task6.1.2 的用户认证与授权服务
- 为 Task6.1.4 的用户数据同步机制提供基础 API 支持
- 为前端用户界面和游戏功能提供数据接口

## 工作量估计

- API 设计与规范制定：1 人天
- 用户个人资料 API 实现：1.5 人天
- 游戏统计数据 API 实现：2 人天
- 用户设置管理 API 实现：1.5 人天
- 社交关系 API 实现：1.5 人天
- API 测试与性能优化：2 人天
- API 文档编写与校验：1.5 人天

总计：约 11 人天
