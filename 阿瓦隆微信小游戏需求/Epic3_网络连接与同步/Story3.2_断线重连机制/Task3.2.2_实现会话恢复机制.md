# Task 3.2.2: 实现会话恢复机制

## 描述

本任务负责实现阿瓦隆微信小游戏的会话恢复机制，包括设计会话标识系统、存储游戏状态信息、实现客户端和服务端状态恢复流程，确保玩家断线后能够回到正确的游戏房间，并恢复游戏状态。

## 目标

1. 设计会话标识系统，用于唯一识别玩家会话
2. 实现会话状态的存储和序列化
3. 实现服务端的会话恢复和验证逻辑
4. 实现客户端的会话恢复请求和状态同步
5. 确保玩家在 3 分钟内重连可以恢复到原有游戏

## 前置条件

1. 断线检测机制已实现完成
2. 服务端数据存储基础设施（Redis）已就绪
3. 游戏状态管理系统已完成基本设计

## 实现步骤

### 1. 会话标识系统设计

```javascript
// server/utils/sessionId.js
const { v4: uuidv4 } = require("uuid");
const jwt = require("jsonwebtoken");
const config = require("../config");

class SessionIdManager {
  /**
   * 生成新的会话ID
   * @returns {string} UUID格式的会话ID
   */
  static generateSessionId() {
    return uuidv4();
  }

  /**
   * 生成会话令牌
   * @param {Object} payload 令牌内容
   * @returns {string} JWT格式的会话令牌
   */
  static generateToken(payload) {
    return jwt.sign(payload, config.JWT_SECRET, {
      expiresIn: "1d", // 令牌有效期1天
    });
  }

  /**
   * 验证会话令牌
   * @param {string} token JWT令牌
   * @returns {Object|null} 解析后的令牌内容或null
   */
  static verifyToken(token) {
    try {
      return jwt.verify(token, config.JWT_SECRET);
    } catch (error) {
      console.error("Token verification failed:", error.message);
      return null;
    }
  }
}

module.exports = SessionIdManager;
```

### 2. 服务端会话存储实现

```javascript
// server/services/sessionService.js
const redis = require("../database/redis");
const SessionIdManager = require("../utils/sessionId");

class SessionService {
  constructor() {
    this.sessionPrefix = "session:";
    this.roomPrefix = "room:";
    this.sessionTTL = 3 * 60; // 会话保留3分钟（秒）
  }

  /**
   * 创建新会话
   * @param {Object} userData 用户数据
   * @param {string} roomId 房间ID
   * @returns {Object} 会话信息
   */
  async createSession(userData, roomId = null) {
    const sessionId = SessionIdManager.generateSessionId();
    const now = Date.now();

    const sessionData = {
      sessionId,
      playerId: userData.playerId,
      nickname: userData.nickname,
      avatarUrl: userData.avatarUrl,
      roomId,
      gameState: null,
      connectionStatus: "connected",
      createTime: now,
      lastActiveTime: now,
      disconnectTime: null,
      expiryTime: null,
    };

    // 存储会话数据
    await redis.hset(`${this.sessionPrefix}${sessionId}`, sessionData);

    // 生成令牌
    const token = SessionIdManager.generateToken({
      sessionId,
      playerId: userData.playerId,
    });

    return {
      sessionId,
      token,
      ...sessionData,
    };
  }

  /**
   * 更新会话状态
   * @param {string} sessionId 会话ID
   * @param {Object} updateData 更新数据
   */
  async updateSession(sessionId, updateData) {
    const key = `${this.sessionPrefix}${sessionId}`;

    // 更新最后活跃时间
    updateData.lastActiveTime = Date.now();

    await redis.hset(key, updateData);
  }

  /**
   * 标记会话为断线状态
   * @param {string} sessionId 会话ID
   */
  async markDisconnected(sessionId) {
    const now = Date.now();
    const expiryTime = now + this.sessionTTL * 1000;

    await this.updateSession(sessionId, {
      connectionStatus: "disconnected",
      disconnectTime: now,
      expiryTime,
    });

    // 设置会话过期
    await redis.expire(`${this.sessionPrefix}${sessionId}`, this.sessionTTL);

    // 获取会话信息
    const sessionData = await this.getSession(sessionId);

    // 如果在房间中，通知房间其他玩家
    if (sessionData.roomId) {
      await this.notifyRoomPlayerDisconnected(
        sessionData.roomId,
        sessionData.playerId
      );
    }
  }

  /**
   * 恢复会话
   * @param {string} sessionId 会话ID
   * @returns {Object|null} 会话数据或null
   */
  async restoreSession(sessionId) {
    const key = `${this.sessionPrefix}${sessionId}`;

    // 获取会话数据
    const sessionData = await redis.hgetall(key);

    if (!sessionData || Object.keys(sessionData).length === 0) {
      return null;
    }

    // 检查会话是否过期
    if (
      sessionData.expiryTime &&
      Date.now() > parseInt(sessionData.expiryTime, 10)
    ) {
      await redis.del(key);
      return null;
    }

    // 恢复连接状态
    await this.updateSession(sessionId, {
      connectionStatus: "connected",
      disconnectTime: null,
      expiryTime: null,
    });

    // 取消过期时间
    await redis.persist(key);

    // 如果在房间中，通知房间其他玩家
    if (sessionData.roomId) {
      await this.notifyRoomPlayerReconnected(
        sessionData.roomId,
        sessionData.playerId
      );
    }

    return sessionData;
  }

  /**
   * 获取会话信息
   * @param {string} sessionId 会话ID
   * @returns {Object|null} 会话数据或null
   */
  async getSession(sessionId) {
    const data = await redis.hgetall(`${this.sessionPrefix}${sessionId}`);
    return Object.keys(data).length > 0 ? data : null;
  }

  /**
   * 保存玩家游戏状态
   * @param {string} sessionId 会话ID
   * @param {Object} gameState 游戏状态
   */
  async saveGameState(sessionId, gameState) {
    await this.updateSession(sessionId, {
      gameState: JSON.stringify(gameState),
    });
  }

  /**
   * 通知房间玩家断线
   * @param {string} roomId 房间ID
   * @param {string} playerId 玩家ID
   */
  async notifyRoomPlayerDisconnected(roomId, playerId) {
    // 实际实现中会通过Socket.IO广播到房间中的其他玩家
    console.log(`通知房间 ${roomId} 玩家 ${playerId} 断线`);
  }

  /**
   * 通知房间玩家重连
   * @param {string} roomId 房间ID
   * @param {string} playerId 玩家ID
   */
  async notifyRoomPlayerReconnected(roomId, playerId) {
    // 实际实现中会通过Socket.IO广播到房间中的其他玩家
    console.log(`通知房间 ${roomId} 玩家 ${playerId} 重连`);
  }
}

module.exports = new SessionService();
```

### 3. 客户端会话管理实现

```javascript
// client/services/sessionManager.js
class SessionManager {
  constructor() {
    this.storageKey = "avalon_session";
    this.session = null;
    this.loadSession();
  }

  /**
   * 保存会话信息到本地
   * @param {Object} sessionData 会话数据
   */
  saveSession(sessionData) {
    this.session = sessionData;

    try {
      wx.setStorageSync(this.storageKey, JSON.stringify(sessionData));
    } catch (e) {
      console.error("Failed to save session to storage:", e);
    }
  }

  /**
   * 从本地加载会话信息
   */
  loadSession() {
    try {
      const sessionData = wx.getStorageSync(this.storageKey);
      if (sessionData) {
        this.session = JSON.parse(sessionData);
      }
    } catch (e) {
      console.error("Failed to load session from storage:", e);
      this.session = null;
    }
  }

  /**
   * 清除会话信息
   */
  clearSession() {
    this.session = null;

    try {
      wx.removeStorageSync(this.storageKey);
    } catch (e) {
      console.error("Failed to clear session from storage:", e);
    }
  }

  /**
   * 获取当前会话信息
   * @returns {Object|null} 会话信息
   */
  getSession() {
    return this.session;
  }

  /**
   * 检查是否有有效会话
   * @returns {boolean} 是否有会话
   */
  hasSession() {
    return !!this.session && !!this.session.token;
  }

  /**
   * 获取会话令牌
   * @returns {string|null} 令牌
   */
  getToken() {
    return this.session ? this.session.token : null;
  }

  /**
   * 获取会话ID
   * @returns {string|null} 会话ID
   */
  getSessionId() {
    return this.session ? this.session.sessionId : null;
  }

  /**
   * 获取玩家ID
   * @returns {string|null} 玩家ID
   */
  getPlayerId() {
    return this.session ? this.session.playerId : null;
  }

  /**
   * 更新会话游戏状态
   * @param {Object} gameState 游戏状态
   */
  updateGameState(gameState) {
    if (this.session) {
      this.session.gameState = gameState;
      this.saveSession(this.session);
    }
  }
}

export default new SessionManager();
```

### 4. 实现重连请求处理

```javascript
// server/controllers/reconnectionController.js
const sessionService = require("../services/sessionService");
const gameService = require("../services/gameService");
const roomService = require("../services/roomService");
const SessionIdManager = require("../utils/sessionId");

class ReconnectionController {
  /**
   * 处理重连请求
   * @param {Object} socket Socket.IO socket对象
   * @param {Object} data 请求数据
   */
  async handleReconnection(socket, data) {
    try {
      const { token } = data;

      // 验证令牌
      const decoded = SessionIdManager.verifyToken(token);
      if (!decoded) {
        return this.sendError(socket, "Invalid or expired token");
      }

      const { sessionId } = decoded;

      // 恢复会话
      const sessionData = await sessionService.restoreSession(sessionId);
      if (!sessionData) {
        return this.sendError(socket, "Session expired or not found");
      }

      // 关联Socket和会话
      socket.data.sessionId = sessionId;
      socket.data.playerId = sessionData.playerId;

      // 如果在房间中，重新加入房间
      if (sessionData.roomId) {
        socket.data.roomId = sessionData.roomId;
        socket.join(sessionData.roomId);

        // 获取房间和游戏状态
        const roomData = await roomService.getRoomById(sessionData.roomId);
        const gameData = await gameService.getGameByRoomId(sessionData.roomId);

        // 发送重连成功响应
        socket.emit("reconnection_success", {
          session: sessionData,
          room: roomData,
          game: gameData,
        });

        // 广播玩家重连消息
        socket.to(sessionData.roomId).emit("player_reconnected", {
          playerId: sessionData.playerId,
          nickname: sessionData.nickname,
        });
      } else {
        // 不在房间中，只恢复会话
        socket.emit("reconnection_success", {
          session: sessionData,
          room: null,
          game: null,
        });
      }

      return true;
    } catch (error) {
      console.error("Reconnection error:", error);
      this.sendError(socket, "Reconnection failed");
      return false;
    }
  }

  /**
   * 发送错误响应
   * @param {Object} socket Socket对象
   * @param {string} message 错误消息
   */
  sendError(socket, message) {
    socket.emit("reconnection_error", { message });
    return false;
  }
}

module.exports = new ReconnectionController();
```

### 5. 客户端重连流程实现

```javascript
// client/services/reconnectionService.js
import SessionManager from "./sessionManager";
import NetworkManager from "./networkManager";
import GameStateManager from "./gameStateManager";
import UIManager from "./uiManager";

class ReconnectionService {
  constructor() {
    this.isReconnecting = false;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.baseReconnectDelay = 1000; // 1秒
    this.reconnectTimer = null;

    // 注册网络恢复事件
    NetworkManager.onReconnect(() => {
      this.tryReconnect();
    });
  }

  /**
   * 尝试重连
   * @returns {Promise<boolean>} 重连是否成功
   */
  async tryReconnect() {
    if (this.isReconnecting) return false;
    if (!SessionManager.hasSession()) return false;

    this.isReconnecting = true;
    this.reconnectAttempts = 0;

    // 显示重连提示
    UIManager.showReconnecting();

    return this.executeReconnect();
  }

  /**
   * 执行重连
   * @returns {Promise<boolean>} 重连是否成功
   */
  async executeReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      this.isReconnecting = false;
      UIManager.showReconnectFailed();
      return false;
    }

    const delay = this.baseReconnectDelay * Math.pow(2, this.reconnectAttempts);
    this.reconnectAttempts++;

    // 更新重连UI
    UIManager.updateReconnectingStatus(
      this.reconnectAttempts,
      this.maxReconnectAttempts
    );

    // 等待延迟
    await new Promise((resolve) => setTimeout(resolve, delay));

    try {
      // 连接服务器
      await NetworkManager.connect();

      // 发送重连请求
      const result = await this.sendReconnectRequest();

      if (result.success) {
        this.handleReconnectSuccess(result.data);
        return true;
      } else {
        // 处理特定错误
        if (result.error === "Session expired or not found") {
          this.handleSessionExpired();
          return false;
        }

        // 继续尝试
        return this.executeReconnect();
      }
    } catch (error) {
      console.error("Reconnection attempt failed:", error);

      // 继续尝试
      return this.executeReconnect();
    }
  }

  /**
   * 发送重连请求
   * @returns {Promise<Object>} 请求结果
   */
  async sendReconnectRequest() {
    const token = SessionManager.getToken();

    return new Promise((resolve) => {
      // 超时处理
      const timeoutId = setTimeout(() => {
        NetworkManager.off("reconnection_success");
        NetworkManager.off("reconnection_error");
        resolve({ success: false, error: "Request timeout" });
      }, 5000);

      // 成功响应
      NetworkManager.once("reconnection_success", (data) => {
        clearTimeout(timeoutId);
        resolve({ success: true, data });
      });

      // 错误响应
      NetworkManager.once("reconnection_error", (error) => {
        clearTimeout(timeoutId);
        resolve({ success: false, error: error.message });
      });

      // 发送请求
      NetworkManager.emit("reconnect", { token });
    });
  }

  /**
   * 处理重连成功
   * @param {Object} data 服务器返回数据
   */
  handleReconnectSuccess(data) {
    this.isReconnecting = false;

    // 更新会话信息
    SessionManager.saveSession(data.session);

    // 恢复游戏状态
    if (data.game) {
      GameStateManager.restoreGameState(data.game);
    }

    // 更新UI
    UIManager.hideReconnecting();
    UIManager.showReconnectSuccess();

    // 恢复到正确的游戏场景
    if (data.room && data.game) {
      // 回到游戏场景
      wx.redirectTo({
        url: `/pages/game/game?roomId=${data.room.roomId}`,
      });
    } else if (data.room) {
      // 回到房间场景
      wx.redirectTo({
        url: `/pages/room/room?roomId=${data.room.roomId}`,
      });
    } else {
      // 回到大厅
      wx.redirectTo({
        url: "/pages/lobby/lobby",
      });
    }
  }

  /**
   * 处理会话过期
   */
  handleSessionExpired() {
    this.isReconnecting = false;

    // 清除会话
    SessionManager.clearSession();

    // 更新UI
    UIManager.hideReconnecting();
    UIManager.showSessionExpired();

    // 回到登录页
    setTimeout(() => {
      wx.redirectTo({
        url: "/pages/login/login",
      });
    }, 2000);
  }

  /**
   * 取消重连
   */
  cancelReconnect() {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    this.isReconnecting = false;
    UIManager.hideReconnecting();
  }

  /**
   * 手动触发重连
   * @returns {Promise<boolean>} 重连是否成功
   */
  manualReconnect() {
    this.reconnectAttempts = 0;
    return this.tryReconnect();
  }
}

export default new ReconnectionService();
```

### 6. 会话恢复流程集成

```javascript
// client/app.js
import SessionManager from "./services/sessionManager";
import NetworkManager from "./services/networkManager";
import ReconnectionService from "./services/reconnectionService";

App({
  globalData: {
    userInfo: null,
    sessionReady: false,
  },

  onLaunch() {
    // 初始化网络管理
    NetworkManager.initialize();

    // 检查是否有会话信息
    if (SessionManager.hasSession()) {
      // 尝试恢复会话
      this.tryRestoreSession();
    }
  },

  async tryRestoreSession() {
    try {
      // 尝试重连
      const success = await ReconnectionService.tryReconnect();

      this.globalData.sessionReady = true;

      if (!success) {
        // 重连失败，清除会话
        SessionManager.clearSession();
      }
    } catch (error) {
      console.error("Failed to restore session:", error);
      SessionManager.clearSession();
      this.globalData.sessionReady = true;
    }
  },

  onHide() {
    // 应用进入后台时记录状态
    if (NetworkManager.isConnected()) {
      NetworkManager.saveLastActiveState();
    }
  },

  onShow() {
    // 应用恢复前台时检查连接状态
    if (SessionManager.hasSession() && !NetworkManager.isConnected()) {
      // 检查是否需要重连
      if (NetworkManager.wasActiveBeforeHide()) {
        ReconnectionService.tryReconnect();
      }
    }
  },
});
```

## 输出成果

1. 会话标识系统：生成和验证会话 ID 和令牌
2. 服务端会话存储：管理会话创建、更新和恢复
3. 客户端会话管理：本地存储和管理会话信息
4. 重连请求处理：服务端处理重连请求并恢复状态
5. 客户端重连流程：处理断线重连和状态恢复
6. 应用级集成：在应用生命周期中集成会话恢复

## 验收标准

1. 断线后 3 分钟内玩家可以重新连接并恢复游戏状态
2. 客户端能够存储和恢复会话信息
3. 服务端能够识别和验证重连请求，恢复正确的会话
4. 重连成功后玩家能够回到正确的游戏房间
5. 游戏状态能够准确恢复，包括角色、游戏阶段等
6. 其他玩家能够看到断线玩家的状态变化（断线和重连）
7. 会话过期或失效时能够提供清晰的错误提示

## 注意事项

1. 会话令牌需要设计合理的有效期，避免长期使用
2. 会话数据需要设置适当的过期时间，减少无效数据占用
3. 重连流程需要考虑网络异常情况，设计退避策略
4. 敏感游戏信息（如角色身份）需在重连时验证权限
5. 会话恢复需确保游戏状态一致性，避免状态不同步

## 相关文档

- [JWT 使用指南](https://jwt.io/introduction/)
- [Redis 数据存储最佳实践](https://redis.io/topics/data-types-intro)
- [微信小游戏存储 API](https://developers.weixin.qq.com/minigame/dev/api/storage/wx.setStorage.html)
- [断线重连技术方案](../技术方案.md)
