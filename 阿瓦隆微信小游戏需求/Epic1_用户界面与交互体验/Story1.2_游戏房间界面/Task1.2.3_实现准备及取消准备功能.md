# Task 1.2.3: 实现准备及取消准备功能

## 描述

实现游戏房间中的玩家准备及取消准备功能，包括准备按钮、取消准备按钮、准备状态同步、全员准备状态判断等。确保玩家能够通过简单的交互表明自己是否已准备好开始游戏，并实时向其他玩家展示准备状态。

## 详细需求

### 功能需求

1. 准备操作

   - 玩家可通过点击按钮表明已准备好开始游戏
   - 准备操作需实时反馈到界面上
   - 准备状态需同步给房间内所有其他玩家
   - 玩家准备后，按钮变为"取消准备"

2. 取消准备操作

   - 已准备的玩家可通过点击按钮取消准备状态
   - 取消操作需实时反馈到界面上
   - 取消状态需同步给房间内所有其他玩家
   - 取消准备后，按钮变回"准备"

3. 状态展示

   - 玩家准备状态在玩家列表中清晰展示
   - 房间内已准备人数/总人数统计显示
   - 当所有玩家都准备好后，向房主显示可以开始游戏的提示

4. 特殊规则
   - 房主可以在未准备状态下开始游戏（可选）
   - 玩家准备后自动锁定角色选择（如果有角色选择功能）
   - 房间玩家人数变化时可能需要重置部分玩家的准备状态

### 界面要求

1. 准备/取消准备按钮位于房间界面底部，醒目且易于点击
2. 按钮状态变化有明显的视觉区分（颜色、图标、文字等）
3. 按钮点击有适当的交互反馈（动画、音效等）
4. 准备状态变化时，玩家列表中对应玩家项有状态更新动画
5. 当全员准备就绪时，可以有特殊的视觉/音频提示

### 技术要求

1. 使用 WebSocket 实现准备状态的实时同步
2. 状态变化需要服务端验证，防止作弊
3. 优化网络传输，减少不必要的状态同步
4. 实现状态变化的平滑动画过渡
5. 处理网络延迟或断线情况下的状态一致性

## 实现步骤

1. 设计准备/取消准备按钮的视觉样式
2. 实现按钮组件及其状态切换逻辑
3. 实现准备/取消准备的网络请求发送
4. 实现服务端验证及状态广播
5. 实现客户端接收状态更新并更新界面
6. 添加状态变化动画和交互反馈
7. 实现全员准备状态检测及提示
8. 处理异常情况和错误处理

## 代码示例

### 准备按钮组件

```typescript
/**
 * 准备按钮组件
 */
class ReadyButton extends PIXI.Container {
  private background: PIXI.Graphics;
  private label: PIXI.Text;
  private icon: PIXI.Sprite;
  private isReady: boolean = false;

  constructor(width: number = 200, height: number = 60) {
    super();

    // 创建背景
    this.background = new PIXI.Graphics();
    this.addChild(this.background);

    // 创建文本标签
    this.label = new PIXI.Text("准备", {
      fontFamily: "Arial",
      fontSize: 20,
      fontWeight: "bold",
      fill: 0xffffff,
    });
    this.label.anchor.set(0.5);
    this.label.position.set(width / 2, height / 2);
    this.addChild(this.label);

    // 创建图标
    this.icon = new PIXI.Sprite(PIXI.Texture.from("ready_icon"));
    this.icon.width = 24;
    this.icon.height = 24;
    this.icon.anchor.set(0.5);
    this.icon.position.set(width / 2 - 50, height / 2);
    this.addChild(this.icon);

    // 设置按钮样式
    this.updateButtonStyle();

    // 添加交互
    this.interactive = true;
    this.buttonMode = true;
    this.on("pointerdown", this.onPointerDown.bind(this));
    this.on("pointerup", this.onPointerUp.bind(this));
    this.on("pointerupoutside", this.onPointerUp.bind(this));
  }

  /**
   * 更新按钮样式
   */
  private updateButtonStyle(): void {
    this.background.clear();

    // 根据准备状态设置不同样式
    if (this.isReady) {
      // 已准备状态 - 红色取消准备按钮
      this.background.beginFill(0xe74c3c);
      this.label.text = "取消准备";
      this.icon.texture = PIXI.Texture.from("cancel_icon");
    } else {
      // 未准备状态 - 绿色准备按钮
      this.background.beginFill(0x2ecc71);
      this.label.text = "准备";
      this.icon.texture = PIXI.Texture.from("ready_icon");
    }

    this.background.drawRoundedRect(0, 0, 200, 60, 10);
    this.background.endFill();
  }

  /**
   * 设置准备状态
   */
  public setReady(isReady: boolean): void {
    if (this.isReady !== isReady) {
      this.isReady = isReady;
      this.updateButtonStyle();

      // 播放状态变化动画
      this.playStateChangeAnimation();
    }
  }

  /**
   * 获取当前准备状态
   */
  public getReadyState(): boolean {
    return this.isReady;
  }

  /**
   * 播放状态变化动画
   */
  private playStateChangeAnimation(): void {
    // 缩放动画
    gsap.fromTo(
      this.scale,
      { x: 0.95, y: 0.95 },
      { x: 1, y: 1, duration: 0.3, ease: "elastic.out(1.2)" }
    );

    // 图标旋转动画
    gsap.fromTo(
      this.icon,
      { rotation: 0 },
      { rotation: Math.PI * 2, duration: 0.5, ease: "power1.out" }
    );
  }

  /**
   * 指针按下事件
   */
  private onPointerDown(event: PIXI.InteractionEvent): void {
    gsap.to(this.scale, { x: 0.95, y: 0.95, duration: 0.1 });
  }

  /**
   * 指针抬起事件
   */
  private onPointerUp(event: PIXI.InteractionEvent): void {
    gsap.to(this.scale, { x: 1, y: 1, duration: 0.1 });

    // 仅在指针抬起在按钮内时触发点击
    if (event.currentTarget === this) {
      this.emit("readyButtonClicked");
    }
  }

  /**
   * 禁用按钮
   */
  public disable(): void {
    this.interactive = false;
    this.buttonMode = false;
    this.alpha = 0.5;
  }

  /**
   * 启用按钮
   */
  public enable(): void {
    this.interactive = true;
    this.buttonMode = true;
    this.alpha = 1;
  }
}
```

### 准备状态管理器

```typescript
/**
 * 准备状态管理器
 */
class ReadyStateManager {
  private socket: SocketIOClient.Socket;
  private roomId: string;
  private playerId: string;
  private readyButton: ReadyButton;
  private playerListView: PlayerListView;
  private onAllPlayersReady: () => void;

  constructor(
    socket: SocketIOClient.Socket,
    roomId: string,
    playerId: string,
    readyButton: ReadyButton,
    playerListView: PlayerListView,
    onAllPlayersReady: () => void
  ) {
    this.socket = socket;
    this.roomId = roomId;
    this.playerId = playerId;
    this.readyButton = readyButton;
    this.playerListView = playerListView;
    this.onAllPlayersReady = onAllPlayersReady;

    this.init();
  }

  /**
   * 初始化管理器
   */
  private init(): void {
    // 监听准备按钮点击
    this.readyButton.on("readyButtonClicked", this.toggleReadyState.bind(this));

    // 监听玩家准备状态变化事件
    this.socket.on("player:ready", this.handlePlayerReadyUpdate.bind(this));

    // 监听所有玩家准备就绪事件
    this.socket.on("room:allReady", this.handleAllPlayersReady.bind(this));

    // 监听房间玩家变化事件
    this.socket.on(
      "room:playerChanged",
      this.handlePlayerListChanged.bind(this)
    );
  }

  /**
   * 切换准备状态
   */
  private toggleReadyState(): void {
    const newReadyState = !this.readyButton.getReadyState();

    // 禁用按钮，防止重复点击
    this.readyButton.disable();

    // 发送准备状态变更请求
    this.socket.emit(
      "player:setReady",
      {
        roomId: this.roomId,
        ready: newReadyState,
      },
      (response) => {
        // 收到服务器响应后重新启用按钮
        this.readyButton.enable();

        if (response.success) {
          // 更新按钮状态
          this.readyButton.setReady(newReadyState);

          // 播放音效
          playSound(newReadyState ? "ready_sound" : "cancel_sound");
        } else {
          // 显示错误提示
          showToast(response.error || "操作失败，请重试");
        }
      }
    );
  }

  /**
   * 处理玩家准备状态更新
   */
  private handlePlayerReadyUpdate(data: {
    playerId: string;
    isReady: boolean;
  }): void {
    // 如果是当前玩家，更新按钮状态
    if (data.playerId === this.playerId) {
      this.readyButton.setReady(data.isReady);
    }

    // 更新玩家列表中的准备状态
    this.playerListView.updatePlayerReadyState(data.playerId, data.isReady);
  }

  /**
   * 处理所有玩家准备就绪
   */
  private handleAllPlayersReady(): void {
    // 播放全员准备音效
    playSound("all_ready_sound");

    // 调用回调函数
    this.onAllPlayersReady();
  }

  /**
   * 处理玩家列表变化
   */
  private handlePlayerListChanged(playerList: PlayerInfo[]): void {
    // 检查当前玩家是否在列表中
    const currentPlayer = playerList.find((p) => p.id === this.playerId);

    // 如果存在当前玩家，则更新准备按钮状态
    if (currentPlayer) {
      this.readyButton.setReady(currentPlayer.isReady);
    }
  }

  /**
   * 清理资源
   */
  public destroy(): void {
    // 移除事件监听
    this.readyButton.off("readyButtonClicked");
    this.socket.off("player:ready");
    this.socket.off("room:allReady");
    this.socket.off("room:playerChanged");
  }
}

/**
 * 播放音效
 */
function playSound(soundName: string): void {
  // 实际实现会使用音频管理器
  console.log(`Play sound: ${soundName}`);
}

/**
 * 显示提示
 */
function showToast(message: string): void {
  // 实际实现会使用UI提示组件
  console.log(`Toast: ${message}`);
}
```

### 服务端准备状态处理

```typescript
/**
 * 服务端准备状态处理伪代码
 */

// 设置玩家准备状态
async function handleSetPlayerReady(socket, data) {
  const { roomId, ready } = data;
  const playerId = socket.id;

  try {
    // 获取房间信息
    const room = await getRoomById(roomId);
    if (!room) {
      return { success: false, error: "房间不存在" };
    }

    // 检查玩家是否在房间中
    const playerIndex = room.players.findIndex((p) => p.id === playerId);
    if (playerIndex === -1) {
      return { success: false, error: "玩家不在房间中" };
    }

    // 更新玩家准备状态
    room.players[playerIndex].isReady = ready;
    await updateRoom(room);

    // 广播准备状态变化
    io.to(roomId).emit("player:ready", {
      playerId,
      isReady: ready,
    });

    // 检查是否所有玩家都已准备
    const allReady = checkAllPlayersReady(room);
    if (allReady) {
      io.to(roomId).emit("room:allReady");
    }

    return { success: true };
  } catch (error) {
    console.error("设置准备状态出错:", error);
    return { success: false, error: "服务器错误" };
  }
}

// 检查是否所有玩家都已准备
function checkAllPlayersReady(room) {
  // 房间至少需要5个玩家才能开始游戏
  if (room.players.length < 5) {
    return false;
  }

  // 检查每位玩家的准备状态
  return room.players.every((player) => player.isReady);
}
```

## 准备及取消准备功能（文本描述）

准备及取消准备功能是游戏房间中的关键操作，它允许玩家表明自己是否已准备好开始游戏。这一功能的设计遵循简洁明了、反馈即时的原则：

1. **按钮设计与位置**：

   - 准备按钮是一个醒目的绿色圆角矩形，位于房间界面底部中央
   - 按钮上同时显示"准备"文字和一个勾选图标，增强可识别性
   - 按钮大小适中，在移动设备上也便于点击，不易误触
   - 取消准备按钮使用红色，文字改为"取消准备"，图标变为叉号

2. **状态切换流程**：

   - 玩家初次进入房间默认为未准备状态
   - 点击准备按钮后，按钮立即变灰（表示处理中）
   - 服务器确认后，按钮变为红色取消准备按钮
   - 再次点击可切换回未准备状态
   - 每次状态切换都有轻微的动画和音效反馈

3. **视觉反馈机制**：

   - 玩家准备状态在玩家列表中通过颜色和图标清晰展示
   - 已准备玩家的卡片底色变为绿色，并显示对勾图标
   - 房间上方有已准备人数/总人数的统计，例如"5/8 已准备"
   - 当所有所需玩家都准备就绪时，会有全屏提示特效

4. **特殊规则处理**：

   - 房主可看到"开始游戏"按钮，当所有玩家准备就绪时高亮显示
   - 玩家人数不足时，即使全部准备也无法开始游戏，会显示提示
   - 断线重连的玩家会保持之前的准备状态
   - 新玩家加入已有大部分人准备的房间时，会收到友好提示建议准备

5. **网络交互设计**：
   - 准备状态变更立即发送到服务器
   - 服务器广播更新给房间内所有玩家
   - 网络延迟下有适当的加载提示
   - 操作失败时会显示原因并允许重试

通过这些设计，准备功能既满足了功能性需求，也通过适当的视觉和交互设计增强了用户体验，使玩家在游戏开始前的准备阶段也能感受到乐趣和参与感。

## 验收标准

1. 准备按钮正常显示，点击后能成功切换到取消准备状态
2. 取消准备按钮点击后能成功切换回准备状态
3. 状态变化有适当的视觉和音频反馈
4. 状态变化能同步到服务器并广播给所有玩家
5. 玩家列表中的准备状态图标正确更新
6. 已准备人数/总人数统计显示正确
7. 所有玩家准备就绪时，房主的开始游戏按钮高亮显示
8. 断线重连后能正确恢复准备状态
9. 网络延迟下有合适的加载提示
10. 极端情况下（如网络断开）有友好的错误提示

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库
- Socket.IO 客户端（WebSocket 通信）
- 微信小游戏音频 API（音效播放）

## 工作量估计

1.5 人天

## 相关文档

- [游戏房间界面设计稿](待补充)
- [游戏房间网络通信规范](待补充)
- [准备状态流转图](待补充)
