# Task 1.2.1: 设计并实现房间基本信息展示

## 描述

设计并实现游戏房间界面中的基本信息展示部分，包括房间号、房间名称、游戏模式、人数限制、当前人数等基本信息。确保这些信息清晰可见，并能实时更新。

## 详细需求

### 功能需求

1. 展示房间基本信息，包括：

   - 房间号（可复制）
   - 房间名称
   - 游戏模式（标准模式/自定义模式）
   - 人数限制（5-10 人）
   - 当前玩家数/总人数
   - 房主昵称
   - 创建时间

2. 实现信息复制功能

   - 点击房间号可复制到剪贴板，方便分享
   - 复制成功时显示提示

3. 支持信息更新
   - 当房间状态变化时（如人数变化、设置变化）能够实时更新显示
   - 更新时有简单的视觉提示

### 界面要求

1. 信息区域位于房间界面顶部，占据约 15-20% 的高度
2. 使用卡片式设计，与整体界面风格一致
3. 使用合适的字体大小和颜色，确保可读性
4. 关键信息（如房间号、当前人数）要醒目显示
5. 为可交互元素（如可复制的房间号）提供明确的视觉提示
6. 支持横竖屏适配，在不同尺寸设备上能正常显示

### 技术要求

1. 使用 PIXI.js 进行界面元素的渲染
2. 实现组件化设计，便于后续维护和扩展
3. 优化渲染性能，避免不必要的重绘
4. 实现响应式布局，适配不同分辨率设备
5. 使用事件机制接收并处理房间状态更新

## 实现步骤

1. 设计房间信息展示组件的视觉样式
2. 实现基础 UI 组件（文本、图标、背景等）
3. 创建房间信息容器类，管理各个信息项
4. 实现数据绑定机制，将房间数据映射到 UI 展示
5. 实现复制房间号功能
6. 添加状态更新监听和处理逻辑
7. 优化布局适配不同设备
8. 添加动画和过渡效果

## 代码示例

```typescript
/**
 * 房间信息展示组件
 */
class RoomInfoDisplay extends PIXI.Container {
  private roomIdText: PIXI.Text;
  private roomNameText: PIXI.Text;
  private gameModeText: PIXI.Text;
  private playerCountText: PIXI.Text;
  private hostNameText: PIXI.Text;
  private copyIcon: PIXI.Sprite;
  private copyTip: PIXI.Container;

  constructor() {
    super();
    this.init();
  }

  private init(): void {
    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(0x2c3e50, 0.8);
    background.drawRoundedRect(0, 0, 600, 120, 10);
    background.endFill();
    this.addChild(background);

    // 创建标题和内容
    this.createLabels();

    // 创建复制按钮
    this.createCopyButton();

    // 创建复制提示（初始隐藏）
    this.createCopyTip();
  }

  private createLabels(): void {
    // 房间号
    const roomIdLabel = new PIXI.Text("房间号:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    roomIdLabel.position.set(20, 20);
    this.addChild(roomIdLabel);

    this.roomIdText = new PIXI.Text("------", {
      fontFamily: "Arial",
      fontSize: 18,
      fontWeight: "bold",
      fill: 0xffffff,
    });
    this.roomIdText.position.set(90, 20);
    this.addChild(this.roomIdText);

    // 房间名称
    const roomNameLabel = new PIXI.Text("房间名:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    roomNameLabel.position.set(20, 50);
    this.addChild(roomNameLabel);

    this.roomNameText = new PIXI.Text("------", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    this.roomNameText.position.set(90, 50);
    this.addChild(this.roomNameText);

    // 游戏模式
    const gameModeLabel = new PIXI.Text("模式:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    gameModeLabel.position.set(20, 80);
    this.addChild(gameModeLabel);

    this.gameModeText = new PIXI.Text("标准模式", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    this.gameModeText.position.set(90, 80);
    this.addChild(this.gameModeText);

    // 玩家人数
    const playerCountLabel = new PIXI.Text("人数:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    playerCountLabel.position.set(300, 20);
    this.addChild(playerCountLabel);

    this.playerCountText = new PIXI.Text("0/8", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    this.playerCountText.position.set(360, 20);
    this.addChild(this.playerCountText);

    // 房主
    const hostLabel = new PIXI.Text("房主:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    hostLabel.position.set(300, 50);
    this.addChild(hostLabel);

    this.hostNameText = new PIXI.Text("------", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    this.hostNameText.position.set(360, 50);
    this.addChild(this.hostNameText);
  }

  private createCopyButton(): void {
    this.copyIcon = new PIXI.Sprite(PIXI.Texture.from("copy_icon"));
    this.copyIcon.width = 20;
    this.copyIcon.height = 20;
    this.copyIcon.position.set(200, 20);
    this.copyIcon.interactive = true;
    this.copyIcon.buttonMode = true;
    this.copyIcon.on("pointerdown", this.copyRoomId.bind(this));
    this.addChild(this.copyIcon);
  }

  private createCopyTip(): void {
    this.copyTip = new PIXI.Container();

    const tipBackground = new PIXI.Graphics();
    tipBackground.beginFill(0x000000, 0.7);
    tipBackground.drawRoundedRect(0, 0, 120, 30, 5);
    tipBackground.endFill();
    this.copyTip.addChild(tipBackground);

    const tipText = new PIXI.Text("已复制房间号", {
      fontFamily: "Arial",
      fontSize: 14,
      fill: 0xffffff,
    });
    tipText.position.set(10, 8);
    this.copyTip.addChild(tipText);

    this.copyTip.position.set(90, 45);
    this.copyTip.visible = false;
    this.addChild(this.copyTip);
  }

  private copyRoomId(): void {
    const roomId = this.roomIdText.text;

    // 调用微信API复制文本
    wx.setClipboardData({
      data: roomId,
      success: () => {
        // 显示复制成功提示
        this.showCopyTip();
      },
    });
  }

  private showCopyTip(): void {
    this.copyTip.visible = true;
    this.copyTip.alpha = 0;

    // 使用GSAP或其他动画库实现淡入淡出效果
    gsap.to(this.copyTip, {
      alpha: 1,
      duration: 0.2,
      onComplete: () => {
        gsap.to(this.copyTip, {
          alpha: 0,
          delay: 1.5,
          duration: 0.3,
          onComplete: () => {
            this.copyTip.visible = false;
          },
        });
      },
    });
  }

  /**
   * 更新房间信息
   */
  public updateRoomInfo(roomInfo: RoomInfo): void {
    // 更新房间号
    if (this.roomIdText.text !== roomInfo.roomId) {
      this.roomIdText.text = roomInfo.roomId;
      this.animateTextUpdate(this.roomIdText);
    }

    // 更新房间名
    if (this.roomNameText.text !== roomInfo.roomName) {
      this.roomNameText.text = roomInfo.roomName;
      this.animateTextUpdate(this.roomNameText);
    }

    // 更新游戏模式
    const modeText = roomInfo.isCustomMode ? "自定义模式" : "标准模式";
    if (this.gameModeText.text !== modeText) {
      this.gameModeText.text = modeText;
      this.animateTextUpdate(this.gameModeText);
    }

    // 更新人数
    const playerCountText = `${roomInfo.currentPlayers}/${roomInfo.maxPlayers}`;
    if (this.playerCountText.text !== playerCountText) {
      this.playerCountText.text = playerCountText;
      this.animateTextUpdate(this.playerCountText);
    }

    // 更新房主
    if (this.hostNameText.text !== roomInfo.hostName) {
      this.hostNameText.text = roomInfo.hostName;
      this.animateTextUpdate(this.hostNameText);
    }
  }

  /**
   * 文本更新时的动画效果
   */
  private animateTextUpdate(textObject: PIXI.Text): void {
    const originalScale = textObject.scale.x;

    // 使用GSAP或其他动画库实现缩放动画
    gsap.fromTo(
      textObject.scale,
      { x: originalScale * 1.2, y: originalScale * 1.2 },
      {
        x: originalScale,
        y: originalScale,
        duration: 0.3,
        ease: "back.out(1.7)",
      }
    );
  }

  /**
   * 调整组件大小以适应屏幕
   */
  public resize(width: number, height: number): void {
    // 调整背景大小
    const background = this.getChildAt(0) as PIXI.Graphics;
    background.clear();
    background.beginFill(0x2c3e50, 0.8);
    background.drawRoundedRect(0, 0, width - 40, 120, 10);
    background.endFill();

    // 根据需要调整其他元素位置
    // ...
  }
}
```

## 房间信息数据结构

```typescript
/**
 * 房间信息数据结构
 */
interface RoomInfo {
  roomId: string; // 房间ID
  roomName: string; // 房间名称
  hostId: string; // 房主ID
  hostName: string; // 房主昵称
  createTime: number; // 创建时间戳
  isCustomMode: boolean; // 是否自定义模式
  currentPlayers: number; // 当前玩家数
  maxPlayers: number; // 最大玩家数
  gameStatus: GameStatus; // 游戏状态
}

/**
 * 游戏状态枚举
 */
enum GameStatus {
  WAITING = "waiting", // 等待中
  READY = "ready", // 准备中
  PLAYING = "playing", // 游戏中
  ENDED = "ended", // 已结束
}
```

## 房间信息展示功能（文本描述）

房间信息展示功能是游戏房间界面的重要组成部分，它向玩家提供当前房间的关键信息。设计应遵循清晰、直观、实用的原则：

1. **布局设计**：

   - 房间信息区位于界面顶部，使用卡片式设计，有轻微的阴影效果增强层次感
   - 左侧展示房间标识信息（房间号、名称、模式），右侧展示状态信息（人数、房主）
   - 使用垂直方向的分组，每组包含标签和具体内容，标签使用较浅的颜色，内容使用醒目的白色

2. **交互设计**：

   - 房间号旁添加复制图标，点击后可复制房间号到剪贴板
   - 复制成功时，显示简洁的"已复制"提示，提示会在短时间内自动消失
   - 当房间信息更新时，相应的文本会有轻微的缩放动画，提示用户信息已更新

3. **状态展示**：

   - 当前人数/最大人数使用分数形式展示（如"6/8"）
   - 当房间接近满员时（如只剩 1-2 个位置），人数显示颜色变为黄色提醒
   - 当房间满员时，人数显示为红色
   - 游戏模式根据房主设置显示为"标准模式"或"自定义模式"

4. **适配机制**：
   - 在不同尺寸的设备上，保持合理的间距和字体大小
   - 超长文本（如特别长的房间名）会自动截断并显示省略号
   - 在横屏模式下，信息排列更加水平化，充分利用水平空间

## 验收标准

1. 房间信息展示完整，包括房间号、房间名称、游戏模式、人数限制、当前人数、房主昵称
2. 点击复制图标能成功复制房间号并显示提示
3. 当房间状态变化（如人数变化）时，界面能实时更新并有视觉提示
4. 界面在不同尺寸和方向的设备上都能正常显示
5. 文本和图标清晰可见，没有模糊或像素化现象
6. 组件性能良好，不影响整体界面的流畅度
7. 布局和样式符合整体游戏风格

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库（或类似工具）
- 微信小游戏 API（用于剪贴板操作）
- WebSocket 客户端（接收房间更新）

## 工作量估计

1.5 人天

## 相关文档

- [游戏房间界面设计稿](待补充)
- [微信小游戏 API 文档 - 剪贴板](https://developers.weixin.qq.com/minigame/dev/api/device/clipboard/wx.setClipboardData.html)
- [PIXI.js 文档](https://pixijs.download/dev/docs/index.html)
