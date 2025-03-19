# Task 3.3.3: 开发邀请响应逻辑

## 描述

本任务负责实现阿瓦隆微信小游戏的邀请响应逻辑，处理玩家通过分享卡片进入游戏的流程，包括参数解析、房间验证和加入流程，确保被邀请玩家能够顺利加入游戏房间。

## 目标

1. 实现分享参数解析逻辑，正确提取房间 ID 和分享来源
2. 开发房间状态验证功能，判断房间是否可加入
3. 实现邀请进入的错误处理和提示
4. 统计分享转化数据，跟踪邀请成功率

## 前置条件

1. 分享功能已实现并能生成包含参数的分享卡片
2. 房间创建和加入的基础功能已完成
3. 服务端 API 支持房间状态查询
4. 用户识别和登录系统可用

## 实现步骤

### 1. 分享参数解析器

```javascript
// utils/shareParamParser.js
class ShareParamParser {
  /**
   * 解析分享链接参数
   * @param {Object} options 小游戏启动参数
   * @returns {Object|null} 解析后的邀请信息或null
   */
  parseShareParams(options) {
    // 检查是否包含分享参数
    if (!options || !options.query) {
      return null;
    }

    const { roomId, shareFrom, timestamp, scene, sign } = options.query;

    // 验证必要参数
    if (!roomId || !shareFrom || !timestamp || scene !== "room_invite") {
      return null;
    }

    // 验证时间戳合法性
    const parsedTimestamp = parseInt(timestamp, 10);
    if (isNaN(parsedTimestamp)) {
      return null;
    }

    // 返回解析结果
    return {
      roomId,
      shareFrom,
      timestamp: parsedTimestamp,
      scene,
    };
  }

  /**
   * 验证邀请有效期
   * @param {Object} inviteInfo 邀请信息
   * @param {number} validityPeriod 有效期（毫秒）
   * @returns {boolean} 是否在有效期内
   */
  isInviteValid(inviteInfo, validityPeriod = 24 * 60 * 60 * 1000) {
    if (!inviteInfo || !inviteInfo.timestamp) {
      return false;
    }

    return Date.now() - inviteInfo.timestamp <= validityPeriod;
  }
}

export default new ShareParamParser();
```

### 2. 房间验证服务

```javascript
// services/roomValidationService.js
import http from "../utils/http";

class RoomValidationService {
  /**
   * 验证房间状态
   * @param {string} roomId 房间ID
   * @returns {Promise<Object>} 房间状态信息
   */
  async validateRoom(roomId) {
    try {
      const response = await http.get(`/api/rooms/${roomId}/status`);
      return response.data;
    } catch (error) {
      console.error("验证房间状态失败:", error);
      return {
        isValid: false,
        canJoin: false,
        message: "验证房间失败，请稍后重试",
      };
    }
  }

  /**
   * 处理不可加入的情况
   * @param {Object} roomStatus 房间状态
   * @returns {Object} 处理结果
   */
  handleNonJoinableRoom(roomStatus) {
    // 不同情况的处理策略
    if (!roomStatus.isValid) {
      return {
        title: "房间不存在",
        content: "该房间可能已解散，请创建新房间或加入其他房间",
        showNewRoomButton: true,
      };
    }

    if (roomStatus.isFull) {
      return {
        title: "房间已满",
        content: "该房间玩家已满，请等待有玩家离开或加入其他房间",
        showRefreshButton: true,
      };
    }

    if (roomStatus.isPlaying) {
      return {
        title: "游戏已开始",
        content: "该房间游戏已经开始，请等待游戏结束或加入其他房间",
        showNewRoomButton: true,
      };
    }

    return {
      title: "无法加入房间",
      content: roomStatus.message || "无法加入该房间，请稍后重试",
      showNewRoomButton: true,
    };
  }
}

export default new RoomValidationService();
```

### 3. 邀请响应处理逻辑

```javascript
// app.js 中的邀请响应处理函数
App({
  // 其他代码...

  onLaunch(options) {
    // 初始化应用
    this.initApp();

    // 处理启动参数
    this.handleLaunchOptions(options);
  },

  onShow(options) {
    // 从后台进入前台时，也可能带有分享参数
    this.handleLaunchOptions(options);
  },

  // 处理启动参数
  handleLaunchOptions(options) {
    console.log("App启动参数:", options);

    // 解析分享参数
    const inviteInfo = shareParamParser.parseShareParams(options);
    if (!inviteInfo) {
      return;
    }

    // 保存邀请信息到全局状态
    this.globalData.pendingInvite = inviteInfo;

    // 如果用户已登录，处理邀请
    if (this.globalData.isLoggedIn) {
      this.handlePendingInvite();
    } else {
      // 标记为待处理，等用户登录后再处理
      this.globalData.hasInvitePending = true;
    }
  },

  // 处理待处理的邀请
  async handlePendingInvite() {
    const invite = this.globalData.pendingInvite;
    if (!invite) {
      return;
    }

    // 清除待处理邀请
    this.globalData.pendingInvite = null;
    this.globalData.hasInvitePending = false;

    // 检查邀请有效期
    if (!shareParamParser.isInviteValid(invite)) {
      this.showToast("邀请已过期");
      return;
    }

    try {
      // 显示加载提示
      wx.showLoading({ title: "正在加入房间..." });

      // 验证房间状态
      const roomStatus = await roomValidationService.validateRoom(
        invite.roomId
      );
      wx.hideLoading();

      if (roomStatus.isValid && roomStatus.canJoin) {
        // 记录邀请转化
        this.logInviteConversion(invite);

        // 加入房间
        this.joinRoom(invite.roomId);
      } else {
        // 处理不可加入的情况
        const errorInfo =
          roomValidationService.handleNonJoinableRoom(roomStatus);
        this.showRoomJoinError(errorInfo);
      }
    } catch (error) {
      console.error("处理邀请失败:", error);
      wx.hideLoading();

      this.showToast("加入房间失败，请稍后重试");
    }
  },

  // 记录邀请转化
  logInviteConversion(invite) {
    try {
      // 发送邀请转化数据到服务器
      http.post("/api/share/conversion", {
        roomId: invite.roomId,
        shareFrom: invite.shareFrom,
        joinedBy: this.globalData.userInfo.id,
        timestamp: Date.now(),
      });
    } catch (error) {
      console.error("记录邀请转化失败:", error);
      // 非关键错误，不阻止流程继续
    }
  },

  // 显示房间加入错误
  showRoomJoinError(errorInfo) {
    wx.showModal({
      title: errorInfo.title,
      content: errorInfo.content,
      showCancel: errorInfo.showNewRoomButton,
      cancelText: "创建房间",
      confirmText: errorInfo.showRefreshButton ? "刷新状态" : "返回",
      success: (res) => {
        if (res.cancel && errorInfo.showNewRoomButton) {
          // 跳转到创建房间页面
          wx.navigateTo({ url: "/pages/createRoom/createRoom" });
        } else if (res.confirm && errorInfo.showRefreshButton) {
          // 重新检查房间状态
          this.handlePendingInvite();
        }
      },
    });
  },

  // 加入房间
  joinRoom(roomId) {
    wx.navigateTo({
      url: `/pages/room/room?roomId=${roomId}`,
      fail: (err) => {
        console.error("跳转到房间页面失败:", err);
        this.showToast("加入房间失败，请稍后重试");
      },
    });
  },

  // 显示提示
  showToast(message) {
    wx.showToast({
      title: message,
      icon: "none",
      duration: 2000,
    });
  },

  // 其他全局方法和数据...

  globalData: {
    userInfo: null,
    isLoggedIn: false,
    pendingInvite: null,
    hasInvitePending: false,
  },
});
```

### 4. 分享转化数据收集

```javascript
// server/controllers/shareConversionController.js
const ShareConversion = require("../models/shareConversion");
const UserStats = require("../models/userStats");

exports.logConversion = async (req, res) => {
  try {
    const { roomId, shareFrom, joinedBy, timestamp } = req.body;

    // 创建转化记录
    const conversion = new ShareConversion({
      roomId,
      shareFrom,
      joinedBy,
      timestamp: new Date(timestamp || Date.now()),
    });

    await conversion.save();

    // 更新分享者的统计数据
    await UserStats.findOneAndUpdate(
      { userId: shareFrom },
      {
        $inc: {
          totalSuccessfulInvites: 1,
          weeklySuccessfulInvites: 1,
        },
        $set: { lastInviteSuccess: new Date() },
      },
      { upsert: true }
    );

    res.status(201).json({ success: true });
  } catch (error) {
    console.error("记录分享转化失败:", error);
    res.status(500).json({ success: false, message: "服务器错误" });
  }
};

// 获取用户分享统计
exports.getUserShareStats = async (req, res) => {
  try {
    const { userId } = req.params;

    const stats = await UserStats.findOne({ userId });

    if (!stats) {
      return res.json({
        totalSuccessfulInvites: 0,
        weeklySuccessfulInvites: 0,
      });
    }

    res.json({
      totalSuccessfulInvites: stats.totalSuccessfulInvites || 0,
      weeklySuccessfulInvites: stats.weeklySuccessfulInvites || 0,
      lastInviteSuccess: stats.lastInviteSuccess,
    });
  } catch (error) {
    console.error("获取用户分享统计失败:", error);
    res.status(500).json({ success: false, message: "服务器错误" });
  }
};
```

### 5. 分享转化统计页面

```html
<!-- pages/shareStats/shareStats.wxml -->
<view class="container">
  <view class="header">
    <text class="title">我的邀请成绩</text>
  </view>

  <view class="stats-card">
    <view class="stat-item">
      <text class="stat-value">{{stats.totalSuccessfulInvites}}</text>
      <text class="stat-label">总成功邀请</text>
    </view>

    <view class="stat-item">
      <text class="stat-value">{{stats.weeklySuccessfulInvites}}</text>
      <text class="stat-label">本周成功邀请</text>
    </view>
  </view>

  <view class="action-section">
    <button class="share-button" bindtap="shareGame">邀请更多好友</button>
  </view>

  <view class="tips-section">
    <text class="tips-title">邀请提示</text>
    <text class="tips-content"
      >成功邀请好友加入游戏可以获得额外积分奖励！每成功邀请一位好友加入游戏，可获得10积分。</text
    >
  </view>
</view>
```

```javascript
// pages/shareStats/shareStats.js
import ShareManager from "../../services/shareManager";
import http from "../../utils/http";

Page({
  data: {
    stats: {
      totalSuccessfulInvites: 0,
      weeklySuccessfulInvites: 0,
      lastInviteSuccess: null,
    },
  },

  onLoad() {
    this.fetchShareStats();
  },

  async fetchShareStats() {
    try {
      wx.showLoading({ title: "加载中..." });

      const userId = getApp().globalData.userInfo.id;
      const response = await http.get(`/api/users/${userId}/share-stats`);

      this.setData({
        stats: response.data,
      });

      wx.hideLoading();
    } catch (error) {
      console.error("获取分享统计失败:", error);
      wx.hideLoading();

      wx.showToast({
        title: "获取数据失败",
        icon: "none",
      });
    }
  },

  shareGame() {
    // 使用通用游戏分享，而非特定房间分享
    ShareManager.shareGame()
      .then(() => {
        wx.showToast({
          title: "分享成功",
          icon: "success",
        });
      })
      .catch((error) => {
        console.error("分享失败:", error);
      });
  },
});
```

## 输出成果

1. 分享参数解析器模块
2. 房间验证服务模块
3. 邀请响应处理逻辑
4. 分享转化数据收集 API
5. 分享统计展示页面

## 验收标准

1. 用户通过分享链接能够正确进入对应房间
2. 房间状态验证能正确识别不可加入的情况并给出友好提示
3. 分享转化数据能够准确记录和统计
4. 不同场景下的错误处理逻辑正确，提示信息友好
5. 邀请流程顺畅，不会出现卡顿或异常跳转

## 注意事项

1. 处理参数时需注意安全性，防止参数篡改导致的安全问题
2. 房间加入逻辑需处理各种边界情况，如房间已满、游戏已开始等
3. 用户未登录时收到邀请的处理需特别注意
4. 分享统计需定期清理过期数据，避免数据库占用过大
5. 邀请响应逻辑需要做好兼容处理，适应不同版本的微信小游戏环境

## 相关文档

- [微信小游戏场景值文档](https://developers.weixin.qq.com/minigame/dev/reference/scene-list.html)
- [微信小游戏启动参数文档](https://developers.weixin.qq.com/minigame/dev/api/base/app/life-cycle/wx.getLaunchOptionsSync.html)
- [技术方案设计文档](../技术方案.md)
- [分享内容设计文档](./Task3.3.2_设计分享内容与样式.md)
