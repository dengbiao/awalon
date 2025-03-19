# Task 1.2.2: 实现玩家列表及状态展示

## 描述

设计并实现游戏房间界面中的玩家列表和状态展示功能，包括展示所有房间内玩家、玩家准备状态、房主标识等。确保玩家信息清晰可见，并能实时更新。

## 详细需求

### 功能需求

1. 玩家列表展示

   - 展示房间内所有玩家信息（头像、昵称）
   - 突出显示当前玩家（自己）
   - 标识房主身份（如皇冠图标）
   - 显示玩家加入顺序
   - 支持最多 10 名玩家的显示

2. 准备状态展示

   - 显示每位玩家的准备状态（已准备/未准备）
   - 使用直观的视觉标识（如对勾/问号图标）
   - 已准备玩家与未准备玩家有明显视觉区分

3. 实时更新功能

   - 玩家加入房间时实时更新列表
   - 玩家离开房间时实时更新列表
   - 玩家准备状态变化时实时更新
   - 房主变更时实时更新标识

4. 交互功能
   - 点击玩家头像可查看玩家简要信息
   - 长按玩家可进行操作（仅房主可用，如踢出玩家）

### 界面要求

1. 玩家列表位于房间界面中部，占据主要视觉区域
2. 每个玩家项包含头像、昵称、准备状态、特殊标识（如房主）
3. 已准备玩家和未准备玩家有明显的视觉差异
4. 使用流畅的加入/离开动画，增强用户体验
5. 支持横竖屏适配，在不同尺寸设备上能正常显示
6. 玩家过多时支持滚动浏览

### 技术要求

1. 使用 PIXI.js 进行界面元素的渲染
2. 实现组件化设计，便于后续维护和扩展
3. 优化渲染性能，特别是在多人同时加入/离开时
4. 使用事件机制接收并处理玩家列表更新
5. 实现平滑的动画效果，包括玩家加入、离开、状态变更等

## 实现步骤

1. 设计玩家列表项的视觉样式
2. 实现基础的玩家列表容器和布局
3. 实现单个玩家项组件，包含各种状态显示
4. 实现玩家列表的数据绑定和更新机制
5. 添加玩家加入/离开的动画效果
6. 实现玩家状态变更的实时更新和动画
7. 添加交互功能（点击、长按等）
8. 优化滚动和布局适配不同设备
9. 进行性能测试和优化

## 代码示例

```typescript
/**
 * 玩家列表组件
 */
class PlayerListView extends PIXI.Container {
  private players: Map<string, PlayerItemView> = new Map();
  private scrollContainer: PIXI.Container;
  private scrollMask: PIXI.Graphics;
  private currentUserId: string;
  private contentHeight: number = 0;

  /**
   * 构造函数
   * @param width 列表宽度
   * @param height 列表高度
   * @param currentUserId 当前用户ID
   */
  constructor(width: number, height: number, currentUserId: string) {
    super();
    this.currentUserId = currentUserId;

    // 创建列表容器
    this.initContainer(width, height);
  }

  /**
   * 初始化列表容器
   */
  private initContainer(width: number, height: number): void {
    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(0x2c3e50, 0.5);
    background.drawRoundedRect(0, 0, width, height, 10);
    background.endFill();
    this.addChild(background);

    // 创建标题
    const title = new PIXI.Text("玩家列表", {
      fontFamily: "Arial",
      fontSize: 18,
      fontWeight: "bold",
      fill: 0xffffff,
    });
    title.position.set(20, 15);
    this.addChild(title);

    // 创建滚动容器和蒙版
    this.scrollContainer = new PIXI.Container();
    this.scrollContainer.position.set(10, 50);

    this.scrollMask = new PIXI.Graphics();
    this.scrollMask.beginFill(0xffffff);
    this.scrollMask.drawRect(0, 0, width - 20, height - 60);
    this.scrollMask.endFill();
    this.addChild(this.scrollMask);

    this.scrollContainer.mask = this.scrollMask;
    this.addChild(this.scrollContainer);

    // 添加交互支持
    this.enableScrolling();
  }

  /**
   * 启用滚动功能
   */
  private enableScrolling(): void {
    // 简化起见，这里只展示基本思路
    // 实际实现需要处理触摸/鼠标滚动等细节

    let isDragging = false;
    let lastY = 0;

    this.scrollContainer.interactive = true;

    this.scrollContainer.on("pointerdown", (event) => {
      isDragging = true;
      lastY = event.data.global.y;
    });

    this.scrollContainer.on("pointermove", (event) => {
      if (isDragging) {
        const newY = event.data.global.y;
        const deltaY = newY - lastY;

        // 更新容器位置
        let newPosY = this.scrollContainer.y + deltaY;

        // 边界检查
        const minY = 50;
        const maxY = Math.min(
          50,
          50 - (this.contentHeight - this.scrollMask.height)
        );

        newPosY = Math.min(Math.max(newPosY, maxY), minY);
        this.scrollContainer.y = newPosY;

        lastY = newY;
      }
    });

    this.scrollContainer.on("pointerup", () => {
      isDragging = false;
    });

    this.scrollContainer.on("pointerupoutside", () => {
      isDragging = false;
    });
  }

  /**
   * 更新玩家列表
   * @param playerInfos 玩家信息数组
   */
  public updatePlayerList(playerInfos: PlayerInfo[]): void {
    // 记录现有玩家ID
    const existingPlayerIds = new Set(this.players.keys());
    const newPlayerIds = new Set<string>();

    // 添加/更新玩家
    for (let i = 0; i < playerInfos.length; i++) {
      const playerInfo = playerInfos[i];
      newPlayerIds.add(playerInfo.id);

      if (this.players.has(playerInfo.id)) {
        // 更新已存在的玩家
        this.players.get(playerInfo.id)?.updateInfo(playerInfo);
      } else {
        // 添加新玩家
        this.addPlayer(playerInfo, i);
      }
    }

    // 移除已离开的玩家
    for (const playerId of existingPlayerIds) {
      if (!newPlayerIds.has(playerId)) {
        this.removePlayer(playerId);
      }
    }

    // 重新排列玩家列表
    this.rearrangePlayers();
  }

  /**
   * 添加玩家到列表
   * @param playerInfo 玩家信息
   * @param index 排序索引
   */
  private addPlayer(playerInfo: PlayerInfo, index: number): void {
    // 创建玩家项
    const isCurrentUser = playerInfo.id === this.currentUserId;
    const playerItem = new PlayerItemView(playerInfo, isCurrentUser);

    // 设置位置
    playerItem.y = index * (playerItem.height + 10);
    playerItem.alpha = 0;

    // 添加到容器
    this.scrollContainer.addChild(playerItem);
    this.players.set(playerInfo.id, playerItem);

    // 播放添加动画
    gsap.to(playerItem, {
      alpha: 1,
      duration: 0.3,
      ease: "power1.out",
    });

    // 更新内容高度
    this.contentHeight = Math.max(
      this.contentHeight,
      (index + 1) * (playerItem.height + 10)
    );
  }

  /**
   * 从列表中移除玩家
   * @param playerId 玩家ID
   */
  private removePlayer(playerId: string): void {
    const playerItem = this.players.get(playerId);
    if (playerItem) {
      // 播放移除动画
      gsap.to(playerItem, {
        alpha: 0,
        x: -50,
        duration: 0.3,
        ease: "power1.in",
        onComplete: () => {
          this.scrollContainer.removeChild(playerItem);
          this.players.delete(playerId);
        },
      });
    }
  }

  /**
   * 重新排列玩家位置
   */
  private rearrangePlayers(): void {
    let index = 0;

    // 按照加入顺序重新排列
    const sortedPlayers = Array.from(this.players.values()).sort((a, b) => {
      return a.getJoinTime() - b.getJoinTime();
    });

    // 更新位置
    for (const playerItem of sortedPlayers) {
      const targetY = index * (playerItem.height + 10);

      // 如果位置不同，则添加动画
      if (playerItem.y !== targetY) {
        gsap.to(playerItem, {
          y: targetY,
          duration: 0.3,
          ease: "power1.out",
        });
      }

      index++;
    }

    // 更新内容高度
    this.contentHeight =
      sortedPlayers.length * (sortedPlayers[0]?.height || 0 + 10);
  }

  /**
   * 调整组件大小
   */
  public resize(width: number, height: number): void {
    // 调整背景
    const background = this.getChildAt(0) as PIXI.Graphics;
    background.clear();
    background.beginFill(0x2c3e50, 0.5);
    background.drawRoundedRect(0, 0, width, height, 10);
    background.endFill();

    // 调整蒙版
    this.scrollMask.clear();
    this.scrollMask.beginFill(0xffffff);
    this.scrollMask.drawRect(0, 0, width - 20, height - 60);
    this.scrollMask.endFill();
  }
}

/**
 * 单个玩家项组件
 */
class PlayerItemView extends PIXI.Container {
  private playerInfo: PlayerInfo;
  private isCurrentUser: boolean;
  private avatar: PIXI.Sprite;
  private nameText: PIXI.Text;
  private readyIcon: PIXI.Sprite;
  private hostIcon: PIXI.Sprite;
  private background: PIXI.Graphics;

  /**
   * 构造函数
   * @param playerInfo 玩家信息
   * @param isCurrentUser 是否为当前用户
   */
  constructor(playerInfo: PlayerInfo, isCurrentUser: boolean) {
    super();
    this.playerInfo = playerInfo;
    this.isCurrentUser = isCurrentUser;

    this.init();
  }

  /**
   * 初始化组件
   */
  private init(): void {
    // 创建背景
    this.background = new PIXI.Graphics();
    this.updateBackground();
    this.addChild(this.background);

    // 创建头像
    this.avatar = new PIXI.Sprite(
      PIXI.Texture.from(this.playerInfo.avatar || "default_avatar")
    );
    this.avatar.width = 50;
    this.avatar.height = 50;
    this.avatar.position.set(10, 10);
    this.avatar.mask = new PIXI.Graphics()
      .beginFill(0xffffff)
      .drawCircle(this.avatar.x + 25, this.avatar.y + 25, 25)
      .endFill();
    this.addChild(this.avatar);
    this.addChild(this.avatar.mask);

    // 创建名称文本
    this.nameText = new PIXI.Text(this.playerInfo.nickname, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: this.isCurrentUser ? 0x3498db : 0xffffff,
    });
    this.nameText.position.set(70, 25);
    this.addChild(this.nameText);

    // 创建准备图标
    const readyTexture = this.playerInfo.isReady
      ? "ready_icon"
      : "not_ready_icon";
    this.readyIcon = new PIXI.Sprite(PIXI.Texture.from(readyTexture));
    this.readyIcon.width = 24;
    this.readyIcon.height = 24;
    this.readyIcon.position.set(230, 23);
    this.addChild(this.readyIcon);

    // 如果是房主，添加房主图标
    if (this.playerInfo.isHost) {
      this.hostIcon = new PIXI.Sprite(PIXI.Texture.from("host_icon"));
      this.hostIcon.width = 20;
      this.hostIcon.height = 20;
      this.hostIcon.position.set(20, 5);
      this.addChild(this.hostIcon);
    }

    // 添加交互
    this.interactive = true;
    this.buttonMode = true;
    this.on("pointerdown", this.onPointerDown.bind(this));
    this.on("pointerup", this.onPointerUp.bind(this));
  }

  /**
   * 更新背景样式
   */
  private updateBackground(): void {
    this.background.clear();

    const bgColor = this.isCurrentUser
      ? 0x34495e
      : this.playerInfo.isReady
      ? 0x27ae60
      : 0x2c3e50;
    const alpha = this.playerInfo.isReady ? 0.8 : 0.5;

    this.background.beginFill(bgColor, alpha);
    this.background.drawRoundedRect(0, 0, 270, 70, 10);
    this.background.endFill();
  }

  /**
   * 更新玩家信息
   * @param playerInfo 玩家信息
   */
  public updateInfo(playerInfo: PlayerInfo): void {
    // 检查准备状态是否变化
    const readyChanged = this.playerInfo.isReady !== playerInfo.isReady;
    const hostChanged = this.playerInfo.isHost !== playerInfo.isHost;

    // 更新数据
    this.playerInfo = playerInfo;

    // 更新名称
    this.nameText.text = playerInfo.nickname;

    // 更新准备状态
    if (readyChanged) {
      const readyTexture = playerInfo.isReady ? "ready_icon" : "not_ready_icon";
      this.readyIcon.texture = PIXI.Texture.from(readyTexture);

      // 播放准备状态变化动画
      gsap.fromTo(
        this.readyIcon.scale,
        { x: 0.5, y: 0.5 },
        { x: 1, y: 1, duration: 0.3, ease: "back.out(1.7)" }
      );

      // 更新背景
      this.updateBackground();
    }

    // 更新房主状态
    if (hostChanged) {
      if (playerInfo.isHost && !this.hostIcon) {
        // 添加房主图标
        this.hostIcon = new PIXI.Sprite(PIXI.Texture.from("host_icon"));
        this.hostIcon.width = 20;
        this.hostIcon.height = 20;
        this.hostIcon.position.set(20, 5);
        this.addChild(this.hostIcon);

        // 播放房主图标出现动画
        this.hostIcon.scale.set(0);
        gsap.to(this.hostIcon.scale, {
          x: 1,
          y: 1,
          duration: 0.3,
          ease: "back.out(1.7)",
        });
      } else if (!playerInfo.isHost && this.hostIcon) {
        // 移除房主图标
        gsap.to(this.hostIcon, {
          alpha: 0,
          duration: 0.2,
          onComplete: () => {
            if (this.hostIcon) {
              this.removeChild(this.hostIcon);
              this.hostIcon = null;
            }
          },
        });
      }
    }
  }

  /**
   * 获取玩家加入时间
   */
  public getJoinTime(): number {
    return this.playerInfo.joinTime;
  }

  /**
   * 指针按下事件
   */
  private onPointerDown(event: PIXI.InteractionEvent): void {
    gsap.to(this.scale, { x: 0.97, y: 0.97, duration: 0.1 });
  }

  /**
   * 指针抬起事件
   */
  private onPointerUp(event: PIXI.InteractionEvent): void {
    gsap.to(this.scale, { x: 1, y: 1, duration: 0.1 });

    // 触发点击事件
    this.emit("playerItemClick", this.playerInfo);
  }
}
```

## 玩家信息数据结构

```typescript
/**
 * 玩家信息数据结构
 */
interface PlayerInfo {
  id: string; // 玩家ID
  nickname: string; // 昵称
  avatar: string; // 头像URL
  isReady: boolean; // 准备状态
  isHost: boolean; // 是否房主
  joinTime: number; // 加入时间戳
}
```

## 玩家列表展示功能（文本描述）

玩家列表展示功能是游戏房间界面的核心部分，它直观地向用户展示房间中的所有玩家及其状态：

1. **布局与排列**：

   - 玩家列表位于房间界面的中央部分，占据主要视觉区域
   - 每个玩家项包含头像、昵称、准备状态和特殊标识
   - 玩家按加入房间的时间顺序从上到下排列
   - 列表项使用卡片式设计，有明确的视觉边界

2. **状态表示**：

   - 已准备玩家的卡片背景为深绿色，未准备玩家为深蓝灰色
   - 已准备玩家显示绿色对勾图标，未准备玩家显示灰色问号图标
   - 当前用户（自己）的卡片有特殊颜色或边框，使其易于识别
   - 房主玩家在头像旁显示皇冠图标，表示其特殊身份

3. **动态效果**：

   - 新玩家加入时，其卡片从透明渐变显示，引起注意
   - 玩家离开时，卡片向左滑出并淡出，给予明确的离开提示
   - 玩家准备状态变化时，准备图标有放大缩小的动画效果
   - 房主变更时，皇冠图标有平滑的转移动画

4. **交互功能**：

   - 点击玩家卡片时有轻微的缩放反馈
   - 点击玩家头像可查看其简要信息（如游戏记录、胜率等）
   - 房主长按其他玩家卡片时，可显示操作菜单（如踢出房间）
   - 玩家列表支持上下滚动，当玩家数量超过可视区域时

5. **适配能力**：
   - 在不同尺寸的设备上自动调整布局和大小
   - 竖屏模式下列表占据较大的垂直空间
   - 横屏模式下可能改为双列或多列布局
   - 支持触摸滑动和鼠标滚轮操作

通过这些设计，玩家列表不仅清晰展示了房间状态，还通过动画和交互增强了用户体验，使房间准备阶段更加生动有趣。

## 验收标准

1. 玩家列表能正确显示房间内所有玩家及其基本信息
2. 能明确区分已准备和未准备的玩家
3. 房主标识清晰可见
4. 当玩家加入或离开房间时，列表能实时更新并有合适的动画效果
5. 当玩家准备状态变化时，其显示状态能即时更新
6. 点击玩家项有适当的交互反馈
7. 列表在玩家较多时支持滚动，操作流畅
8. 在不同尺寸的设备上都能正常显示
9. 性能良好，即使在多人频繁加入/离开时也不卡顿
10. 整体设计风格与游戏主题一致

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库（或类似工具）
- WebSocket 客户端（接收玩家更新）
- 微信头像 API（获取玩家头像）

## 工作量估计

2 人天

## 相关文档

- [游戏房间界面设计稿](待补充)
- [玩家状态管理流程](待补充)
- [PIXI.js 文档 - 容器与遮罩](https://pixijs.download/dev/docs/PIXI.Container.html)
