# Task 3.3.1: 实现游戏邀请分享

## 描述

本任务负责实现阿瓦隆微信小游戏的游戏房间邀请分享功能，包括开发分享逻辑、生成房间邀请卡片以及处理分享相关的用户界面交互，以便玩家能够便捷地邀请好友加入自己的游戏房间。

## 目标

1. 实现游戏房间的分享功能，生成包含房间信息的分享卡片
2. 开发分享触发逻辑和用户交互界面
3. 实现分享数据的统计和跟踪功能
4. 确保分享功能在各种设备和微信版本上兼容性良好

## 前置条件

1. 房间创建功能已实现
2. 微信小游戏 API 已经配置并可用
3. 游戏基础 UI 框架已完成
4. 服务端 API 支持房间信息查询

## 实现步骤

### 1. 分享管理器实现

```javascript
// services/shareManager.js
class ShareManager {
  constructor() {
    this.api = API_BASE_URL;
  }

  /**
   * 分享房间邀请
   * @param {Object} roomInfo 房间信息
   * @returns {Promise<void>}
   */
  shareRoomInvite(roomInfo) {
    return new Promise(async (resolve, reject) => {
      try {
        // 1. 获取分享图片
        const shareImage = await this.generateShareImage(roomInfo);

        // 2. 构建分享参数
        const query = this.buildShareQuery(roomInfo);

        // 3. 调用微信分享API
        wx.shareAppMessage({
          title: `${roomInfo.creatorName}邀请你加入阿瓦隆游戏!`,
          imageUrl: shareImage,
          query: query,
          success: (res) => {
            console.log("分享成功:", res);
            this.logShareEvent(roomInfo.roomId, "success");
            resolve(res);
          },
          fail: (err) => {
            console.error("分享失败:", err);
            this.logShareEvent(roomInfo.roomId, "fail", err);
            reject(err);
          },
        });
      } catch (error) {
        console.error("准备分享时发生错误:", error);
        reject(error);
      }
    });
  }

  /**
   * 生成分享图片
   * @param {Object} roomInfo 房间信息
   * @returns {Promise<string>} 图片临时路径
   */
  async generateShareImage(roomInfo) {
    return new Promise((resolve, reject) => {
      const canvas = wx.createCanvas();
      const ctx = canvas.getContext("2d");
      canvas.width = 500;
      canvas.height = 400;

      // 绘制背景
      ctx.fillStyle = "#1a237e";
      ctx.fillRect(0, 0, 500, 400);

      // 加载和绘制游戏标志
      const logo = wx.createImage();
      logo.onload = () => {
        ctx.drawImage(logo, 200, 50, 100, 100);

        // 绘制文本信息
        ctx.fillStyle = "#ffffff";
        ctx.font = "bold 24px Arial";
        ctx.textAlign = "center";
        ctx.fillText("阿瓦隆", 250, 180);

        ctx.font = "18px Arial";
        ctx.fillText(`房间号: ${roomInfo.roomCode}`, 250, 220);
        ctx.fillText(`创建者: ${roomInfo.creatorName}`, 250, 250);
        ctx.fillText("点击加入游戏!", 250, 300);

        // 绘制玩家数量信息
        ctx.font = "16px Arial";
        ctx.fillText(
          `当前玩家: ${roomInfo.currentPlayers}/${roomInfo.maxPlayers}`,
          250,
          340
        );

        // 转换为图片
        wx.canvasToTempFilePath({
          canvas,
          success: (res) => resolve(res.tempFilePath),
          fail: reject,
        });
      };
      logo.src = "/images/avalon_logo.png";
      logo.onerror = () => reject(new Error("加载游戏标志失败"));
    });
  }

  /**
   * 构建分享查询参数
   * @param {Object} roomInfo 房间信息
   * @returns {string} 查询参数字符串
   */
  buildShareQuery(roomInfo) {
    const params = {
      roomId: roomInfo.roomId,
      shareFrom: getApp().globalData.userInfo.playerId,
      timestamp: Date.now(),
      scene: "room_invite",
    };

    // 将对象转换为查询字符串
    return Object.keys(params)
      .map((key) => `${key}=${encodeURIComponent(params[key])}`)
      .join("&");
  }

  /**
   * 记录分享事件
   * @param {string} roomId 房间ID
   * @param {string} result 分享结果
   * @param {Object} error 错误信息（如果有）
   */
  logShareEvent(roomId, result, error = null) {
    wx.request({
      url: `${this.api}/api/share/log`,
      method: "POST",
      data: {
        roomId,
        playerId: getApp().globalData.userInfo.playerId,
        result,
        timestamp: Date.now(),
        error: error ? error.message : null,
      },
      fail: (err) => {
        console.error("记录分享事件失败:", err);
      },
    });
  }
}

export default new ShareManager();
```

### 2. 房间界面分享按钮实现

#### WXML 模板:

```html
<!-- pages/room/room.wxml -->
<view class="room-container">
  <view class="room-header">
    <text class="room-title">房间: {{roomInfo.roomCode}}</text>
    <text class="room-status">{{getStatusText()}}</text>
  </view>

  <view class="room-players">
    <!-- 玩家列表 -->
  </view>

  <view class="room-actions">
    <button
      class="action-button primary"
      wx:if="{{roomInfo.status === 'waiting'}}"
      bindtap="startGame"
    >
      开始游戏
    </button>
    <button class="action-button secondary" bindtap="shareRoom">
      邀请好友
    </button>
    <button class="action-button danger" bindtap="leaveRoom">退出房间</button>
  </view>

  <!-- 分享成功提示 -->
  <view
    class="share-toast {{showShareToast ? 'visible' : ''}}"
    bindtap="hideShareToast"
  >
    <icon type="success" size="24"></icon>
    <text>邀请已发送</text>
  </view>
</view>
```

#### WXSS 样式:

```css
/* pages/room/room.wxss */
.room-actions {
  display: flex;
  flex-direction: column;
  margin-top: 20px;
}

.action-button {
  margin: 6px 0;
  padding: 12px 0;
  border-radius: 8px;
  font-size: 16px;
}

.action-button.primary {
  background-color: #4caf50;
  color: white;
}

.action-button.secondary {
  background-color: #2196f3;
  color: white;
}

.action-button.danger {
  background-color: #f44336;
  color: white;
}

.share-toast {
  position: fixed;
  bottom: -60px;
  left: 50%;
  transform: translateX(-50%);
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 8px 16px;
  border-radius: 20px;
  display: flex;
  align-items: center;
  transition: bottom 0.3s ease;
}

.share-toast.visible {
  bottom: 40px;
}

.share-toast icon {
  margin-right: 6px;
}
```

#### JS 逻辑:

```javascript
// pages/room/room.js
import ShareManager from "../../services/shareManager";
import RoomService from "../../services/roomService";

Page({
  data: {
    roomInfo: null,
    showShareToast: false,
  },

  onLoad(options) {
    // 从参数或全局数据获取房间ID
    const roomId = options.roomId || getApp().globalData.currentRoomId;
    if (roomId) {
      this.fetchRoomInfo(roomId);
    } else {
      wx.showToast({
        title: "无效的房间",
        icon: "none",
      });
      setTimeout(() => {
        wx.navigateBack();
      }, 1500);
    }
  },

  async fetchRoomInfo(roomId) {
    try {
      wx.showLoading({ title: "加载中..." });
      const roomInfo = await RoomService.getRoomInfo(roomId);
      this.setData({ roomInfo });
      wx.hideLoading();
    } catch (error) {
      console.error("获取房间信息失败:", error);
      wx.hideLoading();
      wx.showToast({
        title: "获取房间信息失败",
        icon: "none",
      });
    }
  },

  getStatusText() {
    if (!this.data.roomInfo) return "";
    const status = this.data.roomInfo.status;

    switch (status) {
      case "waiting":
        return "等待中";
      case "playing":
        return "游戏中";
      case "finished":
        return "已结束";
      default:
        return status;
    }
  },

  async shareRoom() {
    if (!this.data.roomInfo) return;

    try {
      await ShareManager.shareRoomInvite(this.data.roomInfo);

      // 显示分享成功提示
      this.setData({ showShareToast: true });
      setTimeout(() => {
        this.setData({ showShareToast: false });
      }, 2000);
    } catch (error) {
      console.error("分享失败:", error);
      wx.showToast({
        title: "分享失败",
        icon: "none",
      });
    }
  },

  hideShareToast() {
    this.setData({ showShareToast: false });
  },

  startGame() {
    // 实现开始游戏逻辑
  },

  leaveRoom() {
    // 实现离开房间逻辑
  },

  onShareAppMessage() {
    if (!this.data.roomInfo) {
      return {
        title: "加入阿瓦隆游戏",
        path: "/pages/index/index",
      };
    }

    return {
      title: `${this.data.roomInfo.creatorName}邀请你加入阿瓦隆游戏!`,
      path: `/pages/index/index?roomId=${this.data.roomInfo.roomId}&shareFrom=${
        getApp().globalData.userInfo.playerId
      }&scene=room_invite`,
      imageUrl: "/images/share_default.png", // 默认图片，将被自定义分享图替换
    };
  },
});
```

### 3. 服务端分享事件记录实现

```javascript
// server/controllers/shareController.js
const ShareEvent = require("../models/shareEvent");

exports.logShareEvent = async (req, res) => {
  try {
    const { roomId, playerId, result, timestamp, error } = req.body;

    const shareEvent = new ShareEvent({
      roomId,
      playerId,
      result,
      timestamp: new Date(timestamp),
      error,
      type: "share",
    });

    await shareEvent.save();

    res.status(201).json({ success: true });
  } catch (err) {
    console.error("记录分享事件失败:", err);
    res.status(500).json({ error: "Internal server error" });
  }
};

// server/models/shareEvent.js
const mongoose = require("mongoose");

const shareEventSchema = new mongoose.Schema({
  roomId: {
    type: String,
    required: true,
    index: true,
  },
  playerId: {
    type: String,
    required: true,
    index: true,
  },
  result: {
    type: String,
    enum: ["success", "fail"],
    required: true,
  },
  timestamp: {
    type: Date,
    default: Date.now,
    index: true,
  },
  error: String,
  type: {
    type: String,
    enum: ["share", "conversion"],
    required: true,
  },
});

module.exports = mongoose.model("ShareEvent", shareEventSchema);

// server/routes/api.js
const shareController = require("../controllers/shareController");

router.post("/share/log", shareController.logShareEvent);
```

### 4. 微信小游戏 App.js 中添加分享支持

```javascript
// app.js
App({
  onLaunch(options) {
    // 初始化流程
    this.initUserData();

    // 检查是否通过分享进入
    this.checkShareEntry(options);
  },

  onShow(options) {
    // 当小游戏从后台进入前台时，也可能带有分享参数
    this.checkShareEntry(options);
  },

  checkShareEntry(options) {
    console.log("启动参数:", options);

    // 检查是否包含分享参数
    if (options.query && options.query.scene === "room_invite") {
      // 保存邀请参数到全局数据
      this.globalData.pendingInvite = {
        roomId: options.query.roomId,
        shareFrom: options.query.shareFrom,
        timestamp: parseInt(options.query.timestamp),
      };

      console.log("检测到房间邀请:", this.globalData.pendingInvite);

      // 如果用户数据已加载，处理邀请
      if (this.globalData.userInfo) {
        this.handlePendingInvite();
      }
    }
  },

  async handlePendingInvite() {
    const invite = this.globalData.pendingInvite;
    if (!invite) return;

    // 清除待处理邀请
    this.globalData.pendingInvite = null;

    // 验证邀请有效期（24小时）
    const now = Date.now();
    const validityPeriod = 24 * 60 * 60 * 1000;

    if (now - invite.timestamp > validityPeriod) {
      wx.showToast({
        title: "邀请已过期",
        icon: "none",
      });
      return;
    }

    try {
      // 验证房间状态
      wx.showLoading({ title: "正在加入房间..." });
      const response = await this.request({
        url: `/api/rooms/${invite.roomId}/status`,
        method: "GET",
      });
      wx.hideLoading();

      if (response.isValid && response.canJoin) {
        // 记录分享转化
        this.logInviteConversion(invite.roomId, invite.shareFrom);

        // 跳转到房间
        wx.navigateTo({
          url: `/pages/room/room?roomId=${invite.roomId}`,
        });
      } else {
        // 显示无法加入的原因
        wx.showModal({
          title: "无法加入房间",
          content: response.message || "房间不可用",
          showCancel: false,
        });
      }
    } catch (error) {
      console.error("验证房间状态失败:", error);
      wx.hideLoading();
      wx.showToast({
        title: "连接服务器失败",
        icon: "none",
      });
    }
  },

  logInviteConversion(roomId, shareFrom) {
    this.request({
      url: "/api/share/conversion",
      method: "POST",
      data: {
        roomId,
        sharedBy: shareFrom,
        joinedBy: this.globalData.userInfo.playerId,
      },
    }).catch((err) => {
      console.error("记录分享转化失败:", err);
    });
  },

  request(options) {
    return new Promise((resolve, reject) => {
      wx.request({
        url: this.globalData.apiBaseUrl + options.url,
        method: options.method || "GET",
        data: options.data,
        header: {
          Authorization: this.globalData.token
            ? `Bearer ${this.globalData.token}`
            : "",
          "Content-Type": "application/json",
        },
        success: (res) => {
          if (res.statusCode >= 200 && res.statusCode < 300) {
            resolve(res.data);
          } else {
            reject(res);
          }
        },
        fail: reject,
      });
    });
  },

  initUserData() {
    // 实现用户数据初始化
  },

  globalData: {
    userInfo: null,
    token: null,
    apiBaseUrl: "https://api.avalon-game.com",
    pendingInvite: null,
  },
});
```

## 输出成果

1. 分享管理器模块：处理分享逻辑和图片生成
2. 房间界面分享按钮：触发分享的 UI 组件
3. 分享事件记录：服务端分享数据统计功能
4. 分享进入处理：处理通过分享链接进入游戏的逻辑

## 验收标准

1. 房间界面能显示"邀请好友"按钮，点击后能成功调起分享
2. 分享卡片显示正确的房间信息和创建者信息
3. 通过分享链接能成功加入对应房间
4. 分享事件和转化数据能正确记录到服务器
5. 处理房间已满、已开始等异常情况，提供友好提示
6. 分享功能在 Android 和 iOS 设备上表现一致

## 注意事项

1. 微信小游戏的分享功能有调用频率限制，需要合理设计用户引导
2. 生成分享图片时注意不同设备的性能差异，确保流畅体验
3. 分享链接有效期设置应合理，避免长期过期链接带来的不良体验
4. 需要注意处理用户未授权用户信息的情况

## 相关文档

- [微信小游戏分享 API 文档](https://developers.weixin.qq.com/minigame/dev/api/share/wx.shareAppMessage.html)
- [Canvas 绘图 API 文档](https://developers.weixin.qq.com/minigame/dev/api/render/canvas/Canvas.html)
- [技术方案设计文档](../技术方案.md)
