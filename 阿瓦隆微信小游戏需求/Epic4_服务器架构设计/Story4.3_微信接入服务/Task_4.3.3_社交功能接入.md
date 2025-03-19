# Task 4.3.3: 社交功能接入

## 任务描述

设计并实现阿瓦隆微信小游戏的社交功能接入，包括微信好友互动、邀请分享、排行榜、开放数据域等功能，最大化利用微信平台的社交能力，提升游戏的社交性和传播性。

## 详细需求

### 开放数据域实现

1. **开放数据域设计**

   - 设计与实现微信开放数据域，用于展示好友信息和排行榜
   - 确保开放数据域与主域的隔离性和安全性
   - 实现开放数据域的页面设计和交互逻辑

2. **数据通信机制**

   - 实现主域向开放数据域发送消息的机制
   - 设计开放数据域向主域反馈的通信方式
   - 处理数据通信的异常和超时情况

3. **开放数据管理**
   - 实现开放数据的存储和更新机制
   - 设计数据缓存策略，提高访问性能
   - 处理数据一致性和同步问题

### 好友互动功能

1. **好友列表**

   - 实现获取微信好友列表功能
   - 设计好友数据的展示格式和交互方式
   - 支持好友状态查看和筛选

2. **好友互动**

   - 设计好友间的互动机制，如送礼物、挑战等
   - 实现好友间数据比较功能
   - 开发好友动态通知机制

3. **组队游戏**
   - 实现好友间直接邀请加入游戏的功能
   - 设计多人组队游戏流程
   - 处理组队过程中的状态同步和异常情况

### 分享与邀请

1. **游戏内分享**

   - 实现多场景分享功能，支持分享到好友、群聊等
   - 设计分享内容模板，提高分享吸引力
   - 开发自定义分享图片生成功能

2. **邀请奖励机制**

   - 设计邀请新用户的奖励机制
   - 实现邀请追踪和验证
   - 开发邀请奖励的发放和管理功能

3. **游戏房间分享**
   - 实现游戏房间链接分享功能
   - 设计通过分享链接直接加入游戏的流程
   - 处理分享链接的有效期和安全性

### 排行榜系统

1. **好友排行榜**

   - 实现好友间的游戏数据排行功能
   - 设计排行榜界面和交互方式
   - 支持多种排行维度，如胜率、局数等

2. **全球排行榜**

   - 实现全球玩家的排行榜功能
   - 设计数据更新和缓存策略
   - 处理大数据量下的性能问题

3. **成就系统**
   - 设计游戏成就体系
   - 实现成就解锁和展示功能
   - 支持成就分享到社交平台

## 接口设计

### 客户端接口

```typescript
// 获取好友列表接口
GET /api/wx/getFriendList
Headers: { Authorization: "Bearer {token}" }
Response: {
  friends: FriendInfo[]
}

// 获取排行榜接口
GET /api/wx/getRankingList
Headers: { Authorization: "Bearer {token}" }
Query: { type: 'global' | 'friends', dimension: string, pageSize: number, pageIndex: number }
Response: {
  rankings: RankingItem[],
  totalCount: number
}

// 分享游戏接口
POST /api/wx/createShareInfo
Headers: { Authorization: "Bearer {token}" }
Request: {
  shareType: 'invite' | 'room' | 'achievement',
  targetId?: string,
  customMessage?: string
}
Response: {
  shareId: string,
  shareUrl: string,
  shareImageUrl?: string
}

// 提交分数接口
POST /api/wx/submitScore
Headers: { Authorization: "Bearer {token}" }
Request: {
  score: number,
  dimension: string,
  gameId?: string
}
Response: {
  success: boolean,
  newRanking?: number
}
```

### 内部服务接口

```typescript
interface SocialService {
  // 获取用户好友列表
  getUserFriends(openId: string): Promise<FriendInfo[]>;

  // 创建分享信息
  createShareInfo(
    openId: string,
    shareType: ShareType,
    options?: ShareOptions
  ): Promise<ShareInfo>;

  // 验证分享邀请
  verifyInvitation(
    inviteId: string,
    inviteeOpenId: string
  ): Promise<VerificationResult>;

  // 提交用户分数
  submitUserScore(
    openId: string,
    dimension: string,
    score: number,
    metadata?: any
  ): Promise<ScoreSubmitResult>;

  // 获取排行榜
  getRankingList(
    openId: string,
    rankType: RankType,
    dimension: string,
    options?: RankingOptions
  ): Promise<RankingResult>;
}

interface FriendInfo {
  avatarUrl: string;
  nickname: string;
  openId: string; // 经过加密的openId
  status?: "online" | "offline" | "gaming";
  lastPlayTime?: Date;
}

interface ShareInfo {
  shareId: string;
  shareUrl: string;
  shareImageUrl?: string;
  expireTime?: Date;
}

interface VerificationResult {
  isValid: boolean;
  inviterInfo?: {
    openId: string;
    nickname?: string;
  };
  rewardIssued?: boolean;
}

interface ScoreSubmitResult {
  success: boolean;
  newRanking?: number;
  previousBest?: number;
  rankChange?: number;
}

interface RankingResult {
  items: RankingItem[];
  totalCount: number;
  userRanking?: number;
}

interface RankingItem {
  rank: number;
  nickname: string;
  avatarUrl: string;
  score: number;
  isFriend: boolean;
  isPlayer: boolean;
}
```

## 数据模型

### 排行榜数据模型

```typescript
interface RankingRecord {
  _id: string;
  openId: string;
  dimension: string; // 排名维度：totalWins, winRate, bestStreak等
  score: number;
  metadata?: any;
  timestamp: Date;
  updateCount: number;
}

// MongoDB索引
// { openId: 1, dimension: 1 }
// { dimension: 1, score: -1 }
```

### 分享数据模型

```typescript
interface ShareRecord {
  _id: string;
  shareId: string;
  creatorOpenId: string;
  shareType: "invite" | "room" | "achievement";
  targetId?: string; // 房间ID或成就ID
  customMessage?: string;
  createdAt: Date;
  expireAt: Date;
  clickCount: number;
  conversionCount: number;
  status: "active" | "expired" | "deleted";
}

// MongoDB索引
// { shareId: 1 }
// { creatorOpenId: 1 }
// { expireAt: 1 }
```

### 成就数据模型

```typescript
interface Achievement {
  _id: string;
  achievementId: string;
  title: string;
  description: string;
  icon: string;
  condition: string; // 成就解锁条件的描述
  rarity: "common" | "rare" | "epic" | "legendary";
  rewardType?: string;
  rewardAmount?: number;
  hidden: boolean; // 是否为隐藏成就
}

interface UserAchievement {
  _id: string;
  openId: string;
  achievementId: string;
  unlockedAt: Date;
  progress?: number; // 对于未解锁的成就，记录进度
  shared: boolean;
}

// MongoDB索引
// { openId: 1, achievementId: 1 }
// { achievementId: 1 }
```

## 实现步骤

1. **设置开放数据域**

   - 配置微信开放数据域项目
   - 实现开放数据域与主域通信
   - 开发开放数据域界面和逻辑

2. **开发好友功能**

   - 实现好友列表获取
   - 开发好友数据展示界面
   - 实现好友互动功能

3. **实现分享功能**

   - 开发多场景分享接口
   - 实现自定义分享图片生成
   - 设计分享跟踪和转化统计

4. **开发排行榜系统**

   - 实现排行榜数据存储和更新
   - 开发排行榜界面和交互
   - 实现多维度排行功能

5. **设计成就系统**

   - 定义游戏成就体系
   - 实现成就解锁条件检测
   - 开发成就展示和分享功能

6. **构建邀请奖励机制**

   - 设计邀请流程和验证逻辑
   - 实现奖励发放和管理
   - 开发邀请统计和分析功能

7. **进行测试**
   - 编写单元测试
   - 进行集成测试
   - 执行社交功能压力测试

## 技术选型

1. **客户端技术**

   - 微信小游戏开放数据域 API
   - Canvas 绘图（自定义分享图）
   - TypeScript

2. **服务端技术**

   - NestJS 框架
   - MongoDB（数据存储）
   - Redis（排行榜缓存）
   - Node Canvas（服务端图片生成）

3. **社交工具**
   - 微信开放 API
   - 社交分享 SDK

## 测试要点

1. **功能测试**

   - 开放数据域功能测试
   - 好友列表和互动测试
   - 分享和邀请功能测试
   - 排行榜和成就系统测试

2. **异常测试**

   - 无好友情况处理测试
   - 分享失败情况测试
   - 数据通信异常测试
   - 并发访问压力测试

3. **性能测试**

   - 排行榜大数据量性能测试
   - 开放数据域渲染性能测试
   - 分享图片生成性能测试

4. **兼容性测试**
   - 不同微信版本兼容性测试
   - 不同设备兼容性测试
   - 网络环境适应性测试

## 验收标准

1. 开放数据域正常运行，能够展示好友信息和排行榜
2. 好友列表功能正常，支持好友互动和状态查看
3. 分享功能完善，支持多场景分享，分享内容美观
4. 排行榜系统运行稳定，支持多维度排行和实时更新
5. 成就系统设计合理，解锁条件和奖励明确
6. 邀请奖励机制有效，能够正确追踪和验证邀请
7. 全部测试用例通过，包括功能、异常和性能测试
8. 代码符合项目编码规范，并通过代码审查

## 工作量估计

- 开放数据域设计与实现：2 人天
- 好友功能开发：1.5 人天
- 分享功能实现：1.5 人天
- 排行榜系统开发：1.5 人天
- 成就系统设计与实现：1.5 人天
- 邀请奖励机制构建：1 人天
- 测试与调优：2 人天

总计：**11 人天**

## 依赖关系

- 依赖 Task 4.3.1（微信登录与授权）
- 依赖 Task 4.3.2（用户信息处理）
- 依赖 Task 5.1.1（房间创建与管理）

## 风险与应对

| 风险                     | 可能性 | 影响 | 应对措施                                     |
| ------------------------ | ------ | ---- | -------------------------------------------- |
| 微信开放数据域限制调整   | 中     | 高   | 密切关注微信官方文档更新，设计可适应的架构   |
| 社交功能滥用             | 低     | 中   | 实现防刷机制，限制频繁操作，监控异常行为     |
| 排行榜数据量过大性能问题 | 中     | 中   | 实现分页加载，采用缓存策略，定期清理过期数据 |
| 用户隐私保护问题         | 低     | 高   | 严格遵守微信隐私规范，明确用户数据使用说明   |

## 相关文档

- [微信开放数据域文档](https://developers.weixin.qq.com/minigame/dev/guide/open-ability/open-data.html)
- [微信分享接口文档](https://developers.weixin.qq.com/minigame/dev/api/share/wx.updateShareMenu.html)
- [微信排行榜实现最佳实践](https://developers.weixin.qq.com/community/develop/article/doc/0006a059b54d78c3e3618544d5c13e)
- [MongoDB 索引设计](https://docs.mongodb.com/manual/indexes/)
- [Redis 排行榜实现](https://redis.io/commands/zrevrank)
