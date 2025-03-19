# Task 3.2.4: 实现服务端会话管理

## 描述

本任务负责实现阿瓦隆微信小游戏服务端的会话管理机制，包括处理玩家断线、占位和重连过程中的状态管理，确保游戏进程的稳定性和玩家体验的连续性。

## 目标

1. 实现服务端会话状态管理，包括创建、更新和清理会话
2. 实现断线玩家的占位机制，维护游戏进程
3. 实现会话恢复和验证逻辑
4. 实现游戏状态同步机制
5. 确保断线玩家在 3 分钟内重连可以恢复游戏状态

## 前置条件

1. 断线检测机制已实现完成
2. 会话恢复机制已设计并实现
3. Redis 数据库已配置完成
4. 游戏状态管理系统已完成基本设计

## 实现步骤

### 1. 会话管理服务实现

```javascript
// server/services/sessionManager.js
const redis = require("../database/redis");
const { v4: uuidv4 } = require("uuid");

class SessionManager {
  constructor() {
    this.sessionPrefix = "session:";
    this.sessionTTL = 180; // 3分钟过期
  }

  // 创建新会话
  async createSession(playerData) {
    const sessionId = uuidv4();
    const sessionData = {
      sessionId,
      playerId: playerData.playerId,
      nickname: playerData.nickname,
      roomId: playerData.roomId,
      gameState: null,
      status: "connected",
      createTime: Date.now(),
      lastActiveTime: Date.now(),
    };

    await redis.hset(`${this.sessionPrefix}${sessionId}`, sessionData);
    await redis.expire(`${this.sessionPrefix}${sessionId}`, this.sessionTTL);

    return sessionData;
  }

  // 更新会话状态
  async updateSession(sessionId, updateData) {
    const key = `${this.sessionPrefix}${sessionId}`;
    updateData.lastActiveTime = Date.now();
    await redis.hset(key, updateData);
  }

  // 标记会话为断线状态
  async markDisconnected(sessionId) {
    const now = Date.now();
    await this.updateSession(sessionId, {
      status: "disconnected",
      disconnectTime: now,
      expiryTime: now + this.sessionTTL * 1000,
    });
  }

  // 恢复会话
  async restoreSession(sessionId) {
    const key = `${this.sessionPrefix}${sessionId}`;
    const sessionData = await redis.hgetall(key);

    if (!sessionData || Object.keys(sessionData).length === 0) {
      return null;
    }

    // 检查会话是否过期
    if (
      sessionData.expiryTime &&
      Date.now() > parseInt(sessionData.expiryTime)
    ) {
      await redis.del(key);
      return null;
    }

    // 恢复连接状态
    await this.updateSession(sessionId, {
      status: "connected",
      disconnectTime: null,
      expiryTime: null,
    });

    return sessionData;
  }
}

module.exports = new SessionManager();
```

### 2. 玩家占位管理实现

```javascript
// server/services/playerPlaceholderManager.js
class PlayerPlaceholderManager {
  constructor() {
    this.placeholderPrefix = "placeholder:";
    this.placeholderTTL = 180; // 3分钟过期
  }

  // 创建玩家占位
  async createPlaceholder(sessionId, playerData) {
    const placeholderData = {
      sessionId,
      playerId: playerData.playerId,
      nickname: playerData.nickname,
      roomId: playerData.roomId,
      createTime: Date.now(),
      expiryTime: Date.now() + this.placeholderTTL * 1000,
    };

    await redis.hset(`${this.placeholderPrefix}${sessionId}`, placeholderData);
    await redis.expire(
      `${this.placeholderPrefix}${sessionId}`,
      this.placeholderTTL
    );

    return placeholderData;
  }

  // 移除玩家占位
  async removePlaceholder(sessionId) {
    await redis.del(`${this.placeholderPrefix}${sessionId}`);
  }

  // 检查占位是否有效
  async isValidPlaceholder(sessionId) {
    const key = `${this.placeholderPrefix}${sessionId}`;
    const placeholderData = await redis.hgetall(key);

    if (!placeholderData || Object.keys(placeholderData).length === 0) {
      return false;
    }

    if (Date.now() > parseInt(placeholderData.expiryTime)) {
      await redis.del(key);
      return false;
    }

    return true;
  }
}

module.exports = new PlayerPlaceholderManager();
```

### 3. 游戏状态管理实现

```javascript
// server/services/gameStateManager.js
class GameStateManager {
  constructor() {
    this.gameStatePrefix = "game:";
  }

  // 保存游戏状态
  async saveGameState(roomId, gameState) {
    const key = `${this.gameStatePrefix}${roomId}`;
    await redis.set(key, JSON.stringify(gameState));
  }

  // 获取游戏状态
  async getGameState(roomId) {
    const key = `${this.gameStatePrefix}${roomId}`;
    const gameState = await redis.get(key);
    return gameState ? JSON.parse(gameState) : null;
  }

  // 更新玩家状态
  async updatePlayerState(roomId, playerId, playerState) {
    const gameState = await this.getGameState(roomId);
    if (!gameState) return null;

    gameState.players[playerId] = {
      ...gameState.players[playerId],
      ...playerState,
    };

    await this.saveGameState(roomId, gameState);
    return gameState;
  }
}

module.exports = new GameStateManager();
```

### 4. 重连控制器实现

```javascript
// server/controllers/reconnectionController.js
const sessionManager = require("../services/sessionManager");
const playerPlaceholderManager = require("../services/playerPlaceholderManager");
const gameStateManager = require("../services/gameStateManager");

class ReconnectionController {
  // 处理重连请求
  async handleReconnection(socket, data) {
    try {
      const { sessionId } = data;

      // 验证会话
      const sessionData = await sessionManager.restoreSession(sessionId);
      if (!sessionData) {
        return this.sendError(socket, "会话已过期或不存在");
      }

      // 检查占位是否有效
      const isValidPlaceholder =
        await playerPlaceholderManager.isValidPlaceholder(sessionId);
      if (!isValidPlaceholder) {
        return this.sendError(socket, "占位已过期，请重新加入游戏");
      }

      // 获取游戏状态
      const gameState = await gameStateManager.getGameState(sessionData.roomId);
      if (!gameState) {
        return this.sendError(socket, "游戏状态不存在");
      }

      // 更新玩家状态
      await gameStateManager.updatePlayerState(
        sessionData.roomId,
        sessionData.playerId,
        {
          status: "connected",
          lastActiveTime: Date.now(),
        }
      );

      // 发送重连成功响应
      socket.emit("reconnection_success", {
        session: sessionData,
        game: gameState,
      });

      // 通知房间其他玩家
      socket.to(sessionData.roomId).emit("player_reconnected", {
        playerId: sessionData.playerId,
        nickname: sessionData.nickname,
      });

      return true;
    } catch (error) {
      console.error("重连处理错误:", error);
      return this.sendError(socket, "重连失败");
    }
  }

  // 发送错误响应
  sendError(socket, message) {
    socket.emit("reconnection_error", { message });
    return false;
  }
}

module.exports = new ReconnectionController();
```

## 输出成果

1. 会话管理服务：处理会话的创建、更新和恢复
2. 玩家占位管理：处理断线玩家的占位和恢复
3. 游戏状态管理：维护和同步游戏状态
4. 重连控制器：处理重连请求和状态恢复

## 验收标准

1. 断线玩家在 3 分钟内重连可以恢复游戏状态
2. 服务端能够正确维护断线玩家的占位
3. 重连后游戏状态能够准确同步
4. 其他玩家能够看到断线玩家的状态变化
5. 会话过期后能够正确清理相关数据

## 注意事项

1. 会话数据需要设置合理的过期时间
2. 需要处理并发重连请求
3. 游戏状态同步需要考虑数据一致性
4. 需要处理异常情况和错误恢复
5. 监控会话管理的性能指标

## 相关文档

- [Redis 数据存储最佳实践](https://redis.io/topics/data-types-intro)
- [Socket.IO 服务器 API](https://socket.io/docs/v4/server-api/)
- [断线重连技术方案](../技术方案.md)
