# Task 3.2.3: 开发客户端重连逻辑

## 描述

本任务负责实现阿瓦隆微信小游戏的客户端重连逻辑，包括自动重连尝试、重连流程处理、重连界面设计和交互逻辑，确保玩家在网络中断后能够获得流畅的重连体验，减少因网络问题导致的游戏中断。

## 目标

1. 实现客户端自动重连尝试机制，支持至少 5 次重连尝试
2. 设计并实现重连过程中的用户界面，提供状态反馈
3. 开发重连状态管理逻辑，包括成功、失败和超时处理
4. 实现手动重连功能，允许用户主动触发重连
5. 确保重连过程中的用户体验流畅，提供清晰的状态指示

## 前置条件

1. 断线检测机制已实现完成
2. 会话恢复机制已设计并实现
3. 网络通信基础设施已搭建完成

## 实现步骤

### 1. 重连管理器实现

```javascript
// client/reconnect/reconnectManager.js
import SessionManager from "../services/sessionManager";
import NetworkManager from "../services/networkManager";
import EventEmitter from "../utils/eventEmitter";

class ReconnectManager extends EventEmitter {
  constructor() {
    super();

    this.isReconnecting = false;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.baseReconnectDelay = 1000; // 1秒
    this.reconnectTimer = null;
    this.exponentialFactor = 2; // 指数退避因子

    // 重连状态
    this.state = {
      status: "idle", // idle, reconnecting, success, failed
      attempt: 0,
      maxAttempts: this.maxReconnectAttempts,
      message: "",
    };

    // 绑定网络状态变化事件
    NetworkManager.on("disconnect", () => this.onNetworkDisconnect());
    NetworkManager.on("reconnect", () => this.onNetworkReconnect());
  }

  /**
   * 网络断开处理
   * @private
   */
  onNetworkDisconnect() {
    // 如果有会话且不在重连中，则开始重连
    if (SessionManager.hasSession() && !this.isReconnecting) {
      this.startReconnect();
    }
  }

  /**
   * 网络恢复处理
   * @private
   */
  onNetworkReconnect() {
    // 如果有会话且不在重连中，则开始重连
    if (SessionManager.hasSession() && !this.isReconnecting) {
      this.startReconnect();
    }
  }

  /**
   * 开始重连流程
   * @public
   */
  startReconnect() {
    if (this.isReconnecting) return;

    this.isReconnecting = true;
    this.reconnectAttempts = 0;

    this.setState({
      status: "reconnecting",
      attempt: 1,
      message: "正在重新连接...",
    });

    this.executeReconnect();
  }

  /**
   * 执行重连尝试
   * @private
   */
  executeReconnect() {
    // 检查是否达到最大尝试次数
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      this.handleReconnectFailed("达到最大重试次数");
      return;
    }

    // 计算当前尝试的延迟时间（指数退避）
    const delay =
      this.baseReconnectDelay *
      Math.pow(this.exponentialFactor, this.reconnectAttempts);

    // 更新状态
    this.setState({
      attempt: this.reconnectAttempts + 1,
      message: `正在尝试第 ${this.reconnectAttempts + 1}/${
        this.maxReconnectAttempts
      } 次重连...`,
    });

    // 设置重连计时器
    this.reconnectTimer = setTimeout(() => {
      this.performReconnectAttempt();
    }, delay);
  }

  /**
   * 执行单次重连尝试
   * @private
   */
  async performReconnectAttempt() {
    try {
      // 尝试连接网络
      const connected = await NetworkManager.connect();

      if (!connected) {
        this.handleReconnectAttemptFailed("无法连接到服务器");
        return;
      }

      // 尝试恢复会话
      const token = SessionManager.getToken();
      if (!token) {
        this.handleReconnectFailed("无法获取会话信息");
        return;
      }

      // 发送会话恢复请求
      const result = await NetworkManager.sendRequest("reconnect", { token });

      if (result.success) {
        this.handleReconnectSuccess(result.data);
      } else {
        // 会话过期或其他错误
        if (result.error === "session_expired") {
          this.handleSessionExpired();
        } else {
          this.handleReconnectAttemptFailed(result.error || "重连请求失败");
        }
      }
    } catch (error) {
      console.error("Reconnect attempt error:", error);
      this.handleReconnectAttemptFailed("重连过程发生错误");
    }
  }

  /**
   * 处理单次重连尝试失败
   * @private
   * @param {string} reason 失败原因
   */
  handleReconnectAttemptFailed(reason) {
    this.reconnectAttempts++;

    // 更新状态消息
    this.setState({
      message: `重连失败: ${reason}，将在稍后重试...`,
    });

    // 继续尝试下一次重连
    this.executeReconnect();
  }

  /**
   * 处理重连彻底失败
   * @private
   * @param {string} reason 失败原因
   */
  handleReconnectFailed(reason) {
    this.isReconnecting = false;

    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    // 更新状态
    this.setState({
      status: "failed",
      message: `重连失败: ${reason}`,
    });

    // 触发失败事件
    this.emit("reconnect_failed", { reason });
  }

  /**
   * 处理会话过期
   * @private
   */
  handleSessionExpired() {
    this.isReconnecting = false;

    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    // 清除会话信息
    SessionManager.clearSession();

    // 更新状态
    this.setState({
      status: "failed",
      message: "会话已过期，请重新登录",
    });

    // 触发会话过期事件
    this.emit("session_expired");
  }

  /**
   * 处理重连成功
   * @private
   * @param {Object} data 服务器返回的数据
   */
  handleReconnectSuccess(data) {
    this.isReconnecting = false;

    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    // 更新会话信息
    if (data.session) {
      SessionManager.saveSession(data.session);
    }

    // 更新状态
    this.setState({
      status: "success",
      message: "重连成功",
    });

    // 触发成功事件
    this.emit("reconnect_success", data);
  }

  /**
   * 更新状态
   * @private
   * @param {Object} stateUpdate 状态更新
   */
  setState(stateUpdate) {
    this.state = { ...this.state, ...stateUpdate };
    this.emit("state_change", this.state);
  }

  /**
   * 获取当前重连状态
   * @public
   * @returns {Object} 当前状态
   */
  getState() {
    return this.state;
  }

  /**
   * 取消重连
   * @public
   */
  cancelReconnect() {
    if (!this.isReconnecting) return;

    this.isReconnecting = false;

    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }

    // 更新状态
    this.setState({
      status: "idle",
      message: "重连已取消",
    });

    this.emit("reconnect_cancelled");
  }

  /**
   * 手动触发重连
   * @public
   */
  manualReconnect() {
    // 取消现有重连
    this.cancelReconnect();

    // 重新开始重连流程
    this.startReconnect();
  }
}

export default new ReconnectManager();
```

### 2. 重连 UI 组件实现

```javascript
// client/components/reconnectDialog.js
import ReconnectManager from "../reconnect/reconnectManager";

Component({
  data: {
    visible: false,
    status: "reconnecting", // reconnecting, success, failed
    attempt: 0,
    maxAttempts: 5,
    message: "",
    progress: 0,
    showManualButton: false,
  },

  lifetimes: {
    attached() {
      // 监听重连状态变化
      ReconnectManager.on("state_change", this.handleStateChange.bind(this));

      // 监听重连结果事件
      ReconnectManager.on(
        "reconnect_success",
        this.handleReconnectSuccess.bind(this)
      );
      ReconnectManager.on(
        "reconnect_failed",
        this.handleReconnectFailed.bind(this)
      );
      ReconnectManager.on(
        "session_expired",
        this.handleSessionExpired.bind(this)
      );
    },

    detached() {
      // 清理事件监听
      ReconnectManager.off("state_change", this.handleStateChange);
      ReconnectManager.off("reconnect_success", this.handleReconnectSuccess);
      ReconnectManager.off("reconnect_failed", this.handleReconnectFailed);
      ReconnectManager.off("session_expired", this.handleSessionExpired);
    },
  },

  methods: {
    /**
     * 处理重连状态变化
     * @param {Object} state 新状态
     */
    handleStateChange(state) {
      // 计算进度百分比
      const progress =
        state.status === "reconnecting"
          ? (state.attempt / state.maxAttempts) * 100
          : state.status === "success"
          ? 100
          : 0;

      // 显示手动重连按钮的条件
      const showManualButton =
        state.status === "failed" ||
        (state.status === "reconnecting" && state.attempt > 2);

      this.setData({
        visible: state.status !== "idle",
        status: state.status,
        attempt: state.attempt,
        maxAttempts: state.maxAttempts,
        message: state.message,
        progress,
        showManualButton,
      });

      // 如果重连成功，延迟关闭对话框
      if (state.status === "success") {
        setTimeout(() => {
          this.hide();
        }, 1500);
      }
    },

    /**
     * 处理重连成功
     * @param {Object} data 服务器返回数据
     */
    handleReconnectSuccess(data) {
      // 触发成功事件，通知页面处理重连后的操作
      this.triggerEvent("reconnectSuccess", { data });
    },

    /**
     * 处理重连失败
     * @param {Object} error 错误信息
     */
    handleReconnectFailed(error) {
      // 触发失败事件
      this.triggerEvent("reconnectFailed", { error });
    },

    /**
     * 处理会话过期
     */
    handleSessionExpired() {
      // 触发会话过期事件
      this.triggerEvent("sessionExpired");

      // 延迟关闭对话框并跳转到登录页
      setTimeout(() => {
        this.hide();
        wx.redirectTo({
          url: "/pages/login/login",
        });
      }, 2000);
    },

    /**
     * 点击手动重连按钮
     */
    onManualReconnect() {
      ReconnectManager.manualReconnect();
    },

    /**
     * 点击取消按钮
     */
    onCancel() {
      ReconnectManager.cancelReconnect();
      this.hide();

      // 触发取消事件
      this.triggerEvent("reconnectCancelled");
    },

    /**
     * 隐藏对话框
     */
    hide() {
      this.setData({
        visible: false,
      });
    },

    /**
     * 显示对话框
     */
    show() {
      this.setData({
        visible: true,
      });
    },
  },
});
```

### 3. 重连对话框 WXML 模板

```html
<!-- components/reconnect-dialog/reconnect-dialog.wxml -->
<view class="reconnect-dialog {{visible ? 'visible' : ''}}" catch:tap="prevent">
  <view class="reconnect-content">
    <view class="reconnect-header">
      <text class="reconnect-title">连接已断开</text>
    </view>

    <view class="reconnect-body">
      <view class="reconnect-icon {{status}}"></view>
      <text class="reconnect-message">{{message}}</text>

      <view
        class="reconnect-progress-container"
        wx:if="{{status === 'reconnecting'}}"
      >
        <progress
          class="reconnect-progress"
          percent="{{progress}}"
          active
          stroke-width="3"
        />
        <text class="reconnect-attempts">尝试 {{attempt}}/{{maxAttempts}}</text>
      </view>
    </view>

    <view class="reconnect-footer">
      <button
        class="reconnect-button cancel"
        bindtap="onCancel"
        wx:if="{{status !== 'success'}}"
      >
        取消
      </button>

      <button
        class="reconnect-button retry"
        bindtap="onManualReconnect"
        wx:if="{{showManualButton}}"
      >
        重试
      </button>
    </view>
  </view>
</view>
```

### 4. 重连对话框样式

```css
/* components/reconnect-dialog/reconnect-dialog.wxss */
.reconnect-dialog {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
}

.reconnect-dialog.visible {
  opacity: 1;
  pointer-events: auto;
}

.reconnect-content {
  width: 80%;
  max-width: 300px;
  background-color: #fff;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.reconnect-header {
  padding: 16px;
  text-align: center;
  border-bottom: 1px solid #f0f0f0;
}

.reconnect-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.reconnect-body {
  padding: 20px 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.reconnect-icon {
  width: 48px;
  height: 48px;
  margin-bottom: 16px;
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
}

.reconnect-icon.reconnecting {
  background-image: url("data:image/svg+xml;base64,..."); /* 加载动画SVG */
  animation: spin 1.5s infinite linear;
}

.reconnect-icon.success {
  background-image: url("data:image/svg+xml;base64,..."); /* 成功图标SVG */
}

.reconnect-icon.failed {
  background-image: url("data:image/svg+xml;base64,..."); /* 失败图标SVG */
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.reconnect-message {
  font-size: 14px;
  color: #666;
  text-align: center;
  margin-bottom: 16px;
}

.reconnect-progress-container {
  width: 100%;
  margin-bottom: 8px;
}

.reconnect-progress {
  width: 100%;
}

.reconnect-attempts {
  font-size: 12px;
  color: #999;
  text-align: right;
  margin-top: 4px;
}

.reconnect-footer {
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  border-top: 1px solid #f0f0f0;
}

.reconnect-button {
  flex: 1;
  margin: 0 8px;
  font-size: 14px;
  border-radius: 4px;
}

.reconnect-button.cancel {
  background-color: #f5f5f5;
  color: #666;
}

.reconnect-button.retry {
  background-color: #4caf50;
  color: white;
}
```

### 5. 页面级重连处理

```javascript
// client/pages/game/game.js
import ReconnectManager from "../../reconnect/reconnectManager";
import GameManager from "../../game/gameManager";

Page({
  data: {
    roomId: "",
    gameData: null,
    isLoading: true,
    error: null,
  },

  onLoad(options) {
    // 获取房间ID
    const roomId = options.roomId;
    if (!roomId) {
      this.setError("房间ID无效");
      return;
    }

    this.setData({ roomId });

    // 加载游戏数据
    this.loadGameData();

    // 设置重连事件监听
    this.setupReconnectionHandlers();
  },

  onUnload() {
    // 清理事件监听
    this.cleanupReconnectionHandlers();
  },

  /**
   * 设置重连事件处理
   */
  setupReconnectionHandlers() {
    // 监听重连成功事件
    ReconnectManager.on(
      "reconnect_success",
      this.handleReconnectSuccess.bind(this)
    );

    // 监听重连失败事件
    ReconnectManager.on(
      "reconnect_failed",
      this.handleReconnectFailed.bind(this)
    );

    // 监听会话过期事件
    ReconnectManager.on(
      "session_expired",
      this.handleSessionExpired.bind(this)
    );
  },

  /**
   * 清理重连事件监听
   */
  cleanupReconnectionHandlers() {
    ReconnectManager.off("reconnect_success", this.handleReconnectSuccess);
    ReconnectManager.off("reconnect_failed", this.handleReconnectFailed);
    ReconnectManager.off("session_expired", this.handleSessionExpired);
  },

  /**
   * 处理重连成功
   * @param {Object} data 服务器返回数据
   */
  handleReconnectSuccess(data) {
    console.log("Game page: reconnection successful", data);

    // 如果返回了游戏数据，刷新游戏状态
    if (data.game) {
      this.setData({
        gameData: data.game,
        isLoading: false,
        error: null,
      });

      // 通知游戏管理器更新状态
      GameManager.updateGameState(data.game);
    }

    // 如果返回的房间ID与当前不同，跳转到正确的房间
    if (data.room && data.room.roomId !== this.data.roomId) {
      wx.redirectTo({
        url: `/pages/game/game?roomId=${data.room.roomId}`,
      });
    }
  },

  /**
   * 处理重连失败
   */
  handleReconnectFailed() {
    console.log("Game page: reconnection failed");

    // 显示重连失败提示
    wx.showToast({
      title: "重连失败，游戏可能无法正常进行",
      icon: "none",
      duration: 2000,
    });
  },

  /**
   * 处理会话过期
   */
  handleSessionExpired() {
    console.log("Game page: session expired");

    // 显示会话过期提示
    wx.showModal({
      title: "会话已过期",
      content: "您的游戏会话已过期，需要重新登录",
      showCancel: false,
      success: () => {
        // 跳转到登录页
        wx.redirectTo({
          url: "/pages/login/login",
        });
      },
    });
  },

  /**
   * 加载游戏数据
   */
  async loadGameData() {
    try {
      this.setData({ isLoading: true, error: null });

      // 加载游戏数据
      const gameData = await GameManager.loadGame(this.data.roomId);

      this.setData({
        gameData,
        isLoading: false,
      });
    } catch (error) {
      console.error("Failed to load game data:", error);
      this.setError("加载游戏数据失败");
    }
  },

  /**
   * 设置错误状态
   * @param {string} message 错误消息
   */
  setError(message) {
    this.setData({
      isLoading: false,
      error: message,
    });
  },

  /**
   * 重连对话框事件：重连成功
   */
  onReconnectSuccess(e) {
    const { data } = e.detail;
    console.log("Reconnect dialog: success", data);
  },

  /**
   * 重连对话框事件：重连失败
   */
  onReconnectFailed(e) {
    const { error } = e.detail;
    console.log("Reconnect dialog: failed", error);

    // 显示错误提示
    wx.showToast({
      title: "无法重连到游戏，请稍后重试",
      icon: "none",
      duration: 2000,
    });
  },

  /**
   * 重连对话框事件：会话过期
   */
  onSessionExpired() {
    console.log("Reconnect dialog: session expired");
  },

  /**
   * 重连对话框事件：重连取消
   */
  onReconnectCancelled() {
    console.log("Reconnect dialog: cancelled");

    // 显示提示
    wx.showModal({
      title: "已取消重连",
      content: "您已取消重连，游戏可能无法正常进行。是否返回大厅？",
      confirmText: "返回大厅",
      cancelText: "留在游戏",
      success: (res) => {
        if (res.confirm) {
          // 返回大厅
          wx.redirectTo({
            url: "/pages/lobby/lobby",
          });
        }
      },
    });
  },

  /**
   * 手动触发重连
   */
  triggerManualReconnect() {
    ReconnectManager.manualReconnect();
  },
});
```

### 6. 页面 WXML 集成重连组件

```html
<!-- pages/game/game.wxml -->
<view class="game-container">
  <!-- 游戏内容 -->
  <view class="game-content" wx:if="{{!isLoading && !error}}">
    <!-- 游戏UI组件 -->
  </view>

  <!-- 加载状态 -->
  <view class="loading-container" wx:if="{{isLoading}}">
    <view class="loading-spinner"></view>
    <text class="loading-text">加载中...</text>
  </view>

  <!-- 错误状态 -->
  <view class="error-container" wx:if="{{error}}">
    <icon type="warn" size="64"></icon>
    <text class="error-text">{{error}}</text>
    <button class="error-button" bindtap="loadGameData">重试</button>
  </view>

  <!-- 网络状态指示器 -->
  <view
    class="network-indicator {{isNetworkConnected ? 'connected' : 'disconnected'}}"
    bindtap="triggerManualReconnect"
  >
    <icon type="{{isNetworkConnected ? 'success' : 'warn'}}" size="14"></icon>
    <text>{{isNetworkConnected ? '已连接' : '连接已断开'}}</text>
  </view>

  <!-- 重连对话框组件 -->
  <reconnect-dialog
    bind:reconnectSuccess="onReconnectSuccess"
    bind:reconnectFailed="onReconnectFailed"
    bind:sessionExpired="onSessionExpired"
    bind:reconnectCancelled="onReconnectCancelled"
  />
</view>
```

## 输出成果

1. 重连管理器：处理自动重连、重连状态和事件
2. 重连 UI 组件：提供重连过程的视觉反馈
3. 页面级重连处理：集成重连逻辑到游戏流程
4. 网络状态指示器：显示当前网络连接状态
5. 手动重连功能：允许用户主动触发重连

## 验收标准

1. 断线后能够自动检测并启动重连流程
2. 重连尝试至少进行 5 次，每次间隔时间按指数退避增加
3. 重连对话框提供清晰的视觉反馈，包括进度条和状态
4. 重连成功后能够正确恢复游戏状态和 UI
5. 提供手动重连按钮，允许用户主动触发重连
6. 重连失败时提供合理的错误提示和后续操作选项
7. 会话过期时能够引导用户回到登录页面
8. 重连过程不影响游戏的其他 UI 元素和操作

## 注意事项

1. 重连对话框需要适应不同屏幕大小和方向
2. 避免频繁尝试重连导致服务器压力增大
3. 重连状态保持与其他页面的一致性
4. 需处理微信小游戏前后台切换时的重连逻辑
5. UI 设计需符合微信小游戏的设计规范

## 相关文档

- [微信小游戏 UI 组件文档](https://developers.weixin.qq.com/minigame/dev/guide/base-ability/custom-component.html)
- [事件处理文档](https://developers.weixin.qq.com/minigame/dev/guide/base-ability/event.html)
- [网络状态 API](https://developers.weixin.qq.com/minigame/dev/api/device/network/wx.getNetworkType.html)
- [断线重连技术方案](../技术方案.md)
