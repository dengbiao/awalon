# Task 3.2.1: 设计断线检测机制

## 描述

本任务主要负责设计并实现阿瓦隆微信小游戏的断线检测机制，包括客户端和服务端的心跳包机制、网络状态监控和断线判定逻辑，以确保系统能够及时发现并应对网络中断情况。

## 目标

1. 设计一套可靠的断线检测机制，能在 3 秒内检测到网络异常
2. 实现客户端网络状态监控，及时感知网络变化
3. 实现服务端连接状态追踪，能够区分正常断开和异常断开
4. 提供不同网络环境下的断线检测策略优化

## 前置条件

1. WebSocket 通信基础设施已搭建完成
2. 微信小游戏网络相关 API 可用
3. 服务端已具备基本的会话管理能力

## 实现步骤

### 1. 实现客户端心跳机制

```javascript
// network/heartbeat.js
class HeartbeatManager {
  constructor(websocket, options = {}) {
    this.ws = websocket;
    this.interval = options.interval || 15000; // 15秒发送一次心跳
    this.timeout = options.timeout || 45000; // 45秒无响应判定为断线
    this.timer = null;
    this.lastResponseTime = Date.now();
    this.onTimeout = options.onTimeout || (() => {});
    this.onNetworkResumed = options.onNetworkResumed || (() => {});
    this.disconnected = false;
  }

  start() {
    if (this.timer) this.stop();

    this.timer = setInterval(() => {
      this.sendHeartbeat();
      this.checkTimeout();
    }, this.interval);

    // 初始状态设置
    this.lastResponseTime = Date.now();
    this.disconnected = false;
  }

  sendHeartbeat() {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(
        JSON.stringify({ type: "heartbeat", timestamp: Date.now() })
      );
    }
  }

  receiveHeartbeatResponse(data) {
    this.lastResponseTime = Date.now();

    // 如果之前断线了，现在恢复了连接
    if (this.disconnected) {
      this.disconnected = false;
      this.onNetworkResumed();
    }
  }

  checkTimeout() {
    const elapsed = Date.now() - this.lastResponseTime;

    if (elapsed > this.timeout && !this.disconnected) {
      this.disconnected = true;
      this.onTimeout();
    }
  }

  stop() {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }
}

export default HeartbeatManager;
```

### 2. 监听微信网络状态变化

```javascript
// network/networkMonitor.js
class NetworkMonitor {
  constructor(options = {}) {
    this.onNetworkChange = options.onNetworkChange || (() => {});
    this.onNetworkDisconnect = options.onNetworkDisconnect || (() => {});
    this.onNetworkResume = options.onNetworkResume || (() => {});
    this.networkType = "unknown";
    this.isConnected = true;
    this.initialize();
  }

  initialize() {
    // 获取当前网络状态
    wx.getNetworkType({
      success: (res) => {
        this.networkType = res.networkType;
        this.isConnected = res.networkType !== "none";
      },
    });

    // 监听网络状态变化
    wx.onNetworkStatusChange((res) => {
      const wasConnected = this.isConnected;
      this.networkType = res.networkType;
      this.isConnected = res.isConnected;

      // 触发网络变化事件
      this.onNetworkChange({
        networkType: this.networkType,
        isConnected: this.isConnected,
      });

      // 断网事件
      if (wasConnected && !this.isConnected) {
        this.onNetworkDisconnect();
      }

      // 网络恢复事件
      if (!wasConnected && this.isConnected) {
        this.onNetworkResume();
      }
    });
  }

  getCurrentNetworkType() {
    return this.networkType;
  }

  isNetworkConnected() {
    return this.isConnected;
  }
}

export default NetworkMonitor;
```

### 3. 实现服务端心跳响应

```javascript
// server/heartbeat.js
class ServerHeartbeatManager {
  constructor(io) {
    this.io = io;
    this.clientLastHeartbeat = new Map(); // 记录每个客户端的最后心跳时间
    this.timeout = 60000; // 60秒无心跳判定为断线
    this.checkInterval = 30000; // 30秒检查一次
    this.timer = null;
  }

  start() {
    // 设置心跳响应处理
    this.io.on("connection", (socket) => {
      // 记录初始连接时间
      this.clientLastHeartbeat.set(socket.id, Date.now());

      // 处理心跳请求
      socket.on("heartbeat", (data) => {
        // 更新最后心跳时间
        this.clientLastHeartbeat.set(socket.id, Date.now());

        // 发送响应
        socket.emit("heartbeat_response", { timestamp: Date.now() });
      });

      // 处理断开连接
      socket.on("disconnect", () => {
        this.clientLastHeartbeat.delete(socket.id);
      });
    });

    // 定时检查超时客户端
    this.timer = setInterval(() => {
      this.checkTimeouts();
    }, this.checkInterval);
  }

  checkTimeouts() {
    const now = Date.now();

    for (const [socketId, lastTime] of this.clientLastHeartbeat.entries()) {
      if (now - lastTime > this.timeout) {
        // 获取socket实例
        const socket = this.io.sockets.sockets.get(socketId);

        if (socket) {
          // 标记为超时断线
          this.handleTimeout(socket);
        } else {
          // 客户端已经断开但没有正确清理
          this.clientLastHeartbeat.delete(socketId);
        }
      }
    }
  }

  handleTimeout(socket) {
    // 记录断线事件
    console.log(`Client ${socket.id} timed out`);

    // 获取关联的玩家和游戏信息
    const playerId = socket.data.playerId;
    const roomId = socket.data.roomId;

    if (playerId && roomId) {
      // 通知房间其他玩家
      socket.to(roomId).emit("player_disconnected", { playerId });

      // 标记玩家为断线状态
      // 这里调用会话管理服务
      global.sessionManager.markDisconnected(playerId);
    }

    // 强制断开超时连接
    socket.disconnect(true);
    this.clientLastHeartbeat.delete(socket.id);
  }

  stop() {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }
}

module.exports = ServerHeartbeatManager;
```

### 4. 实现连接状态管理

```javascript
// network/connectionManager.js
import HeartbeatManager from "./heartbeat";
import NetworkMonitor from "./networkMonitor";

class ConnectionManager {
  constructor(options = {}) {
    this.ws = null;
    this.serverUrl = options.serverUrl;
    this.gameId = null;
    this.playerId = null;

    this.heartbeatManager = null;
    this.networkMonitor = null;

    this.onDisconnect = options.onDisconnect || (() => {});
    this.onReconnect = options.onReconnect || (() => {});
    this.onError = options.onError || (() => {});

    this.isConnected = false;
    this.isConnecting = false;
    this.manualDisconnect = false; // 标记是否为主动断开

    this.initialize();
  }

  initialize() {
    // 创建网络监控
    this.networkMonitor = new NetworkMonitor({
      onNetworkDisconnect: () => {
        console.log("Network disconnected");
        // 网络断开，触发断线事件
        if (this.isConnected) {
          this.handleDisconnect(false);
        }
      },
      onNetworkResume: () => {
        console.log("Network resumed");
        // 网络恢复，尝试重连
        if (!this.isConnected && !this.manualDisconnect) {
          this.tryReconnect();
        }
      },
    });
  }

  connect() {
    if (this.isConnected || this.isConnecting) return;

    this.isConnecting = true;
    this.manualDisconnect = false;

    try {
      // 创建WebSocket连接
      this.ws = new WebSocket(this.serverUrl);

      this.ws.onopen = () => {
        console.log("WebSocket connected");
        this.isConnected = true;
        this.isConnecting = false;

        // 创建并启动心跳管理
        this.heartbeatManager = new HeartbeatManager(this.ws, {
          onTimeout: () => {
            console.log("Heart beat timeout");
            this.handleDisconnect(false);
          },
          onNetworkResumed: () => {
            console.log("Heart beat resumed");
            // 心跳恢复，说明连接已经恢复
            if (!this.isConnected) {
              this.isConnected = true;
              this.onReconnect();
            }
          },
        });
        this.heartbeatManager.start();
      };

      this.ws.onclose = (event) => {
        console.log(`WebSocket closed: ${event.code} ${event.reason}`);
        this.handleDisconnect(this.manualDisconnect);
      };

      this.ws.onerror = (error) => {
        console.error("WebSocket error:", error);
        this.onError(error);
        if (this.isConnecting) {
          this.isConnecting = false;
        }
      };

      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);

        // 处理心跳响应
        if (data.type === "heartbeat_response") {
          this.heartbeatManager.receiveHeartbeatResponse(data);
        }
      };
    } catch (error) {
      console.error("Failed to establish WebSocket connection:", error);
      this.isConnecting = false;
      this.onError(error);
    }
  }

  disconnect() {
    this.manualDisconnect = true;

    if (this.heartbeatManager) {
      this.heartbeatManager.stop();
    }

    if (this.ws) {
      this.ws.close();
    }

    this.isConnected = false;
  }

  handleDisconnect(isManual) {
    if (!this.isConnected) return;

    this.isConnected = false;

    if (this.heartbeatManager) {
      this.heartbeatManager.stop();
      this.heartbeatManager = null;
    }

    if (!isManual) {
      // 非主动断开，触发断线事件
      this.onDisconnect();
    }
  }

  tryReconnect() {
    if (this.isConnecting || this.manualDisconnect) return;

    if (this.networkMonitor.isNetworkConnected()) {
      this.connect();
    }
  }

  isSocketConnected() {
    return this.isConnected;
  }
}

export default ConnectionManager;
```

### 5. 集成到游戏主流程

```javascript
// game/gameNetwork.js
import ConnectionManager from "../network/connectionManager";

class GameNetwork {
  constructor(game) {
    this.game = game;
    this.connectionManager = new ConnectionManager({
      serverUrl: CONFIG.WS_SERVER_URL,
      onDisconnect: () => this.handleDisconnect(),
      onReconnect: () => this.handleReconnect(),
      onError: (error) => this.handleError(error),
    });
  }

  initialize() {
    // 连接到游戏服务器
    this.connectionManager.connect();

    // 监听UI上的网络状态指示
    this.game.ui.setNetworkStatusIndicator({
      onManualReconnect: () => this.connectionManager.tryReconnect(),
    });
  }

  handleDisconnect() {
    console.log("Game network disconnected");

    // 更新UI，显示断线提示
    this.game.ui.showDisconnectNotification();

    // 通知游戏逻辑层处理断线
    this.game.onNetworkDisconnect();
  }

  handleReconnect() {
    console.log("Game network reconnected");

    // 更新UI，隐藏断线提示
    this.game.ui.hideDisconnectNotification();

    // 通知游戏逻辑层处理重连
    this.game.onNetworkReconnect();
  }

  handleError(error) {
    console.error("Game network error:", error);

    // 显示错误提示
    this.game.ui.showErrorNotification(`网络错误: ${error.message}`);
  }

  shutdown() {
    // 主动断开连接
    this.connectionManager.disconnect();
  }
}

export default GameNetwork;
```

## 输出成果

1. 客户端心跳包机制
2. 微信小游戏网络状态监控
3. 服务端心跳响应处理
4. 连接状态管理模块
5. 游戏网络层集成

## 验收标准

1. 客户端在网络断开 3 秒内能检测到并触发相应事件
2. 服务端能在 60 秒内检测到心跳超时的客户端
3. 系统能够正确区分网络断线和主动断开连接
4. 断线检测机制能适应不同的网络环境（WiFi、4G、弱网等）
5. 资源占用合理，不造成明显性能影响

## 注意事项

1. 心跳频率设置需平衡及时性和资源消耗
2. 微信小游戏在切换到后台时可能会导致 WebSocket 连接异常
3. 不同网络类型下的心跳超时设置可能需要调整
4. 需考虑服务端大量断线重连时的并发处理能力

## 相关文档

- [微信小游戏 WebSocket API 文档](https://developers.weixin.qq.com/minigame/dev/api/network/websocket/wx.connectSocket.html)
- [网络状态监听 API 文档](https://developers.weixin.qq.com/minigame/dev/api/device/network/wx.onNetworkStatusChange.html)
- [技术方案详细设计](../技术方案.md)
