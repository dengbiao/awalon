# Task 2.1.3: 实现角色特殊信息展示

## 描述

实现阿瓦隆游戏中角色特殊信息的展示功能，包括特殊角色（如梅林、派西维尔等）看到的其他玩家信息，以及这些信息的视觉呈现方式。

## 详细需求

### 功能需求

1. 根据角色类型，向玩家展示特定的其他玩家信息：
   - 梅林可以看到所有坏人（除莫德雷德）
   - 派西维尔可以看到梅林和莫甘娜（但不知道谁是谁）
   - 坏人（除奥伯伦）互相可见
   - 特殊规则配置下的其他信息展示
2. 特殊信息的展示需要在游戏开始时进行，使用特殊效果强调
3. 在游戏过程中提供随时查看特殊信息的方式
4. 特殊信息展示时提供适当的文字提示，帮助理解规则
5. 支持不同玩家数量和角色配置下的信息展示

### 界面要求

1. 特殊信息展示使用与普通角色信息区分的视觉样式
2. 对特殊角色关系使用直观的视觉效果（如连线、特殊标记等）
3. 关键信息使用动画或高亮效果强调
4. 信息展示界面支持进一步查看详情
5. 界面美观且符合游戏整体风格

### 技术要求

1. 基于角色和游戏规则，精确计算每个玩家应该看到的信息
2. 实现安全的信息展示机制，防止信息泄露
3. 优化特殊效果的性能，确保流畅运行
4. 支持信息的动态更新（如某些特殊技能使用后）

## 实现步骤

1. 设计特殊信息展示的 UI 界面
2. 实现特殊角色关系的计算逻辑
3. 实现特殊信息的视觉呈现组件
4. 实现特殊效果和动画
5. 添加交互功能，支持详细查看
6. 进行用户体验测试和优化

## 代码示例

### 特殊信息展示组件

```typescript
class SpecialRoleInfoView extends PIXI.Container {
  private playerRelationships: PIXI.Container;
  private relationshipLines: PIXI.Graphics;
  private playerSprites: Map<string, PIXI.Container> = new Map();
  private centralPlayer: PIXI.Container;
  private visiblePlayers: VisiblePlayer[];
  private roleType: RoleType;

  constructor(roleType: RoleType, visiblePlayers: VisiblePlayer[]) {
    super();
    this.roleType = roleType;
    this.visiblePlayers = visiblePlayers;

    // 创建关系图容器
    this.playerRelationships = new PIXI.Container();
    this.addChild(this.playerRelationships);

    // 创建连线图形
    this.relationshipLines = new PIXI.Graphics();
    this.playerRelationships.addChild(this.relationshipLines);

    // 初始化视图
    this.initialize();
  }

  private initialize(): void {
    // 创建中央玩家（当前玩家）
    this.centralPlayer = this.createPlayerSprite({
      playerId: "current",
      name: "你",
      avatar: "your_avatar.png",
      roleType: this.roleType,
    });
    this.centralPlayer.x = 200;
    this.centralPlayer.y = 200;
    this.playerRelationships.addChild(this.centralPlayer);

    // 如果没有可见玩家，显示提示信息
    if (this.visiblePlayers.length === 0) {
      const noInfoText = new PIXI.Text("没有特殊可见信息", {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xcccccc,
      });
      noInfoText.anchor.set(0.5);
      noInfoText.x = this.centralPlayer.x;
      noInfoText.y = this.centralPlayer.y + 80;
      this.playerRelationships.addChild(noInfoText);
      return;
    }

    // 计算可见玩家的位置（围绕中央玩家放置）
    const radius = 150;
    const angleStep = (2 * Math.PI) / this.visiblePlayers.length;

    // 创建并放置可见玩家
    this.visiblePlayers.forEach((player, index) => {
      const angle = index * angleStep;
      const x = this.centralPlayer.x + radius * Math.cos(angle);
      const y = this.centralPlayer.y + radius * Math.sin(angle);

      const playerSprite = this.createPlayerSprite(player);
      playerSprite.x = x;
      playerSprite.y = y;
      this.playerRelationships.addChild(playerSprite);
      this.playerSprites.set(player.playerId, playerSprite);
    });

    // 绘制关系连线
    this.drawRelationshipLines();

    // 创建提示文本
    this.createInfoText();
  }

  // 创建玩家精灵（头像、名称、标记）
  private createPlayerSprite(player: {
    playerId: string;
    name: string;
    avatar: string;
    roleType?: RoleType;
    hint?: string;
  }): PIXI.Container {
    const container = new PIXI.Container();

    // 头像背景
    const background = new PIXI.Graphics();
    background.beginFill(0x333333);
    background.drawCircle(0, 0, 30);
    background.endFill();
    container.addChild(background);

    // 头像
    const avatar = new PIXI.Sprite(
      PIXI.Texture.from(player.avatar || "default_avatar.png")
    );
    avatar.width = 60;
    avatar.height = 60;
    avatar.anchor.set(0.5);
    container.addChild(avatar);

    // 角色标记（如果有）
    if (player.roleType) {
      const roleBadge = new PIXI.Sprite(
        PIXI.Texture.from(`role_badge_${player.roleType}.png`)
      );
      roleBadge.width = 25;
      roleBadge.height = 25;
      roleBadge.anchor.set(0.5);
      roleBadge.x = 25;
      roleBadge.y = -25;
      container.addChild(roleBadge);
    }

    // 名称
    const nameText = new PIXI.Text(player.name, {
      fontFamily: "Arial",
      fontSize: 14,
      fill: 0xffffff,
      align: "center",
    });
    nameText.anchor.set(0.5, 0);
    nameText.y = 35;
    container.addChild(nameText);

    // 提示文本（如果有）
    if (player.hint) {
      const hintText = new PIXI.Text(player.hint, {
        fontFamily: "Arial",
        fontSize: 12,
        fill: 0xf1c40f,
        align: "center",
      });
      hintText.anchor.set(0.5, 0);
      hintText.y = 55;
      container.addChild(hintText);
    }

    // 添加交互
    container.interactive = true;
    container.buttonMode = true;
    container.on("pointerover", () => this.onPlayerHover(player.playerId));
    container.on("pointerout", () => this.onPlayerOut(player.playerId));
    container.on("pointerdown", () => this.onPlayerClick(player.playerId));

    return container;
  }

  // 绘制关系连线
  private drawRelationshipLines(): void {
    this.relationshipLines.clear();

    // 根据角色类型选择线条颜色
    let lineColor: number;

    switch (this.roleType) {
      case RoleType.MERLIN:
        lineColor = 0x3498db; // 蓝色
        break;
      case RoleType.PERCIVAL:
        lineColor = 0x9b59b6; // 紫色
        break;
      case RoleType.ASSASSIN:
      case RoleType.MORGANA:
      case RoleType.MORDRED:
      case RoleType.MINION:
        lineColor = 0xe74c3c; // 红色
        break;
      default:
        lineColor = 0x95a5a6; // 灰色
    }

    // 绘制从中央玩家到每个可见玩家的连线
    this.relationshipLines.lineStyle(2, lineColor, 0.6);

    for (const [playerId, sprite] of this.playerSprites.entries()) {
      this.relationshipLines.moveTo(this.centralPlayer.x, this.centralPlayer.y);
      this.relationshipLines.lineTo(sprite.x, sprite.y);
    }
  }

  // 创建信息提示文本
  private createInfoText(): void {
    let infoText: string;

    switch (this.roleType) {
      case RoleType.MERLIN:
        infoText = "你能看到所有坏人（除了莫德雷德）";
        break;
      case RoleType.PERCIVAL:
        infoText = "你能看到梅林和莫甘娜，但不知道谁是谁";
        break;
      case RoleType.ASSASSIN:
      case RoleType.MORGANA:
      case RoleType.MORDRED:
      case RoleType.MINION:
        infoText = "你能看到所有其他坏人（除了奥伯伦）";
        break;
      case RoleType.OBERON:
        infoText = "你是奥伯伦，无法看到其他坏人";
        break;
      default:
        infoText = "你是忠臣，没有特殊信息";
    }

    const text = new PIXI.Text(infoText, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
      align: "center",
      wordWrap: true,
      wordWrapWidth: 400,
    });
    text.anchor.set(0.5, 0);
    text.x = 200;
    text.y = 360;
    this.addChild(text);
  }

  // 玩家悬停事件
  private onPlayerHover(playerId: string): void {
    const sprite = this.playerSprites.get(playerId);
    if (sprite) {
      gsap.to(sprite.scale, {
        x: 1.1,
        y: 1.1,
        duration: 0.2,
      });
    }
  }

  // 玩家离开事件
  private onPlayerOut(playerId: string): void {
    const sprite = this.playerSprites.get(playerId);
    if (sprite) {
      gsap.to(sprite.scale, {
        x: 1,
        y: 1,
        duration: 0.2,
      });
    }
  }

  // 玩家点击事件
  private onPlayerClick(playerId: string): void {
    // 在这里可以显示更详细的玩家信息弹窗
    console.log(`Player clicked: ${playerId}`);
  }

  // 播放角色关系展示动画
  playRevealAnimation(): Promise<void> {
    return new Promise((resolve) => {
      // 初始设置
      this.alpha = 0;
      for (const sprite of this.playerSprites.values()) {
        sprite.alpha = 0;
      }
      this.relationshipLines.alpha = 0;

      // 淡入整个容器
      gsap.to(this, {
        alpha: 1,
        duration: 0.5,
        onComplete: () => {
          // 依次显示每个玩家
          const timeline = gsap.timeline({
            onComplete: resolve,
          });

          // 添加中央玩家出现动画
          timeline.from(this.centralPlayer.scale, {
            x: 0,
            y: 0,
            duration: 0.5,
            ease: "back.out",
          });

          // 添加连线动画
          timeline.to(this.relationshipLines, {
            alpha: 1,
            duration: 0.5,
          });

          // 添加每个可见玩家的动画
          let delay = 0;
          for (const sprite of this.playerSprites.values()) {
            timeline.to(sprite, {
              alpha: 1,
              duration: 0.3,
              delay: delay,
            });
            delay += 0.1;
          }
        },
      });
    });
  }
}
```

### 特殊信息展示场景

```typescript
class SpecialInfoScene extends PIXI.Container {
  private background: PIXI.Graphics;
  private specialInfoView: SpecialRoleInfoView;
  private nextButton: PIXI.Container;
  private roleInfo: RoleInfo;
  private visiblePlayers: VisiblePlayer[];

  constructor(roleInfo: RoleInfo, visiblePlayers: VisiblePlayer[]) {
    super();
    this.roleInfo = roleInfo;
    this.visiblePlayers = visiblePlayers;

    // 创建背景
    this.background = new PIXI.Graphics();
    this.background.beginFill(0x000000, 0.8);
    this.background.drawRect(0, 0, 800, 600);
    this.background.endFill();
    this.addChild(this.background);

    // 创建标题
    const title = new PIXI.Text("你的特殊信息", {
      fontFamily: "Arial",
      fontSize: 28,
      fontWeight: "bold",
      fill: 0xffffff,
      align: "center",
    });
    title.anchor.set(0.5, 0);
    title.x = 400;
    title.y = 30;
    this.addChild(title);

    // 创建特殊信息视图
    this.specialInfoView = new SpecialRoleInfoView(
      roleInfo.type,
      visiblePlayers
    );
    this.specialInfoView.x = 200;
    this.specialInfoView.y = 80;
    this.addChild(this.specialInfoView);

    // 创建下一步按钮
    this.nextButton = this.createNextButton();
    this.nextButton.x = 400;
    this.nextButton.y = 500;
    this.nextButton.alpha = 0; // 初始隐藏
    this.addChild(this.nextButton);

    // 播放展示动画
    this.playRevealAnimation();
  }

  private createNextButton(): PIXI.Container {
    const container = new PIXI.Container();

    // 按钮背景
    const background = new PIXI.Graphics();
    background.beginFill(0x3498db);
    background.drawRoundedRect(0, 0, 180, 50, 10);
    background.endFill();
    container.addChild(background);

    // 按钮文本
    const text = new PIXI.Text("了解！开始游戏", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
      align: "center",
    });
    text.anchor.set(0.5);
    text.x = background.width / 2;
    text.y = background.height / 2;
    container.addChild(text);

    // 添加交互
    container.interactive = true;
    container.buttonMode = true;
    container.on("pointerdown", this.onNextButtonClick.bind(this));

    return container;
  }

  private async playRevealAnimation(): Promise<void> {
    // 播放特殊信息展示动画
    await this.specialInfoView.playRevealAnimation();

    // 显示下一步按钮
    gsap.to(this.nextButton, {
      alpha: 1,
      duration: 0.5,
      ease: "power1.out",
    });
  }

  private onNextButtonClick(): void {
    // 触发关闭事件
    this.emit("complete");

    // 播放退出动画
    gsap.to(this, {
      alpha: 0,
      duration: 0.5,
      onComplete: () => {
        this.destroy();
      },
    });
  }
}
```

### 游戏中的查看功能

```typescript
// 在游戏场景中实现特殊信息查看按钮
class GameScene extends PIXI.Container {
  private roleInfoButton: PIXI.Container;

  constructor() {
    super();
    // 初始化场景
    // ...

    // 创建角色信息查看按钮
    this.roleInfoButton = this.createRoleInfoButton();
    this.addChild(this.roleInfoButton);
  }

  private createRoleInfoButton(): PIXI.Container {
    const button = new PIXI.Container();
    // 创建按钮背景和图标
    // ...

    // 添加点击事件
    button.interactive = true;
    button.buttonMode = true;
    button.on("pointerdown", this.showRoleInfo.bind(this));

    return button;
  }

  private async showRoleInfo(): Promise<void> {
    // 获取角色信息和可见玩家信息
    const gameId = GameManager.getInstance().getCurrentGameId();
    const roleInfoManager = RoleInfoManager.getInstance();
    const { role, visiblePlayers } = await roleInfoManager.fetchRoleInfo(
      gameId
    );

    // 创建特殊信息场景
    const specialInfoScene = new SpecialInfoScene(role, visiblePlayers);
    this.addChild(specialInfoScene);

    // 监听完成事件
    specialInfoScene.once("complete", () => {
      // 继续游戏
    });
  }
}
```

## 界面原型

```
+------------------------------------------+
|        特殊信息展示 - 梅林视角            |
+------------------------------------------+
|                                          |
|  你是梅林，你能看到以下邪恶阵营成员:       |
|                                          |
|                 [你]                     |
|                  |                       |
|                  |                       |
|                  ▼                       |
|  +------+     +------+     +------+     |
|  |玩家2  |<--->|玩家1  |<--->|玩家5  |     |
|  |刺客   |     |梅林   |     |爪牙   |     |
|  +------+     +------+     +------+     |
|                  ^                       |
|                  |                       |
|                  |                       |
|               +------+                   |
|               |玩家7  |                   |
|               |莫甘娜 |                   |
|               +------+                   |
|                                          |
|  注意: 莫德雷德对你隐身，你无法看到他      |
|                                          |
|          [我已了解，继续游戏]             |
+------------------------------------------+
```

```
+------------------------------------------+
|       特殊信息展示 - 派西维尔视角         |
+------------------------------------------+
|                                          |
|  你是派西维尔，你能看到以下玩家:           |
|  (但你无法分辨谁是梅林，谁是莫甘娜)        |
|                                          |
|                 [你]                     |
|                  |                       |
|                  |                       |
|          +-------+-------+               |
|          |               |               |
|          ▼               ▼               |
|      +------+         +------+           |
|      |玩家1  |         |玩家7  |           |
|      |梅林？ |         |莫甘娜？|           |
|      +------+         +------+           |
|                                          |
|  提示: 仔细观察这两位玩家的行为，          |
|  判断谁是真正的梅林！                     |
|                                          |
|          [我已了解，继续游戏]             |
+------------------------------------------+
```

```
+------------------------------------------+
|        特殊信息展示 - 坏人视角            |
+------------------------------------------+
|                                          |
|  你是刺客，你能看到以下邪恶阵营成员:       |
|                                          |
|                 [你]                     |
|                  |                       |
|          +-------+-------+               |
|          |       |       |               |
|          ▼       ▼       ▼               |
|      +------+ +------+ +------+          |
|      |玩家5  | |玩家6  | |玩家7  |          |
|      |爪牙   | |莫德雷德| |莫甘娜 |          |
|      +------+ +------+ +------+          |
|                                          |
|  注意: 奥伯伦不会出现在此列表中            |
|  梅林能看到你，小心隐藏你的身份！          |
|                                          |
|          [我已了解，继续游戏]             |
+------------------------------------------+
```

```
+------------------------------------------+
|        特殊信息展示 - 游戏中查看          |
+------------------------------------------+
|                              [关闭]      |
|                                          |
|  角色关系网络:                           |
|                                          |
|                 [你]                     |
|                  |                       |
|                  |                       |
|                  ▼                       |
|  +------+     +------+     +------+     |
|  |玩家2  |<--->|玩家1  |<--->|玩家5  |     |
|  |刺客   |     |梅林   |     |爪牙   |     |
|  +------+     +------+     +------+     |
|                  ^                       |
|                  |                       |
|                  |                       |
|               +------+                   |
|               |玩家7  |                   |
|               |莫甘娜 |                   |
|               +------+                   |
|                                          |
|  [显示角色信息]        [显示游戏规则]      |
+------------------------------------------+
```

## 游戏中特殊信息查看功能的文本描述

游戏中的特殊信息查看功能是阿瓦隆游戏中至关重要的一部分，它展示了不同角色之间的知情关系，这是游戏推理的基础。具体设计如下：

1. **初始展示**：

   - 游戏开始时，在角色分配后立即展示特殊信息
   - 使用引人注目的动画效果，突出重要关系
   - 采用关系图的形式，中央是玩家自己，周围是可见的其他玩家
   - 使用连线表示关系，不同角色关系使用不同颜色的连线

2. **视觉呈现**：

   - 梅林看到的坏人以红色标记
   - 派西维尔看到的梅林和莫甘娜以紫色标记，并特别注明无法区分
   - 坏人之间的关系以暗红色连线表示
   - 使用玩家头像、名称和特殊标记（如问号、眼睛图标等）增强可视性

3. **交互体验**：

   - 玩家可以点击关系图中的任何头像获取更详细信息
   - 悬停在头像上会显示特殊提示（如"莫德雷德的爪牙"）
   - 提供明确的文字说明，解释当前玩家能看到的信息原因
   - 在首次展示后提供"了解！开始游戏"按钮，让玩家确认已理解

4. **游戏中查看**：

   - 通过右上角的"眼睛"图标按钮随时查看特殊信息
   - 再次查看时，关系图可能会根据游戏进程更新（如特殊能力使用后）
   - 查看面板使用半透明背景覆盖游戏界面，但不完全中断游戏
   - 提供简单的关闭方式（点击按钮或面板外区域）

5. **信息安全**：
   - 所有特殊信息计算在服务端完成，客户端只接收当前玩家可见的信息
   - 网络传输过程加密，防止数据包分析
   - 每次展示前验证信息的有效性
   - 设计防止截屏、录屏泄露信息的措施（如关键信息动态变化）

## 验收标准

1. 特殊信息展示符合阿瓦隆规则，准确展示各角色应知道的信息
2. 关系图直观清晰，玩家容易理解自己能看到的信息
3. 展示动画流畅，增强游戏体验
4. 信息展示安全，不会泄露给不应知道的玩家
5. 交互响应及时，无明显延迟
6. 在不同设备上正确显示，适应各种屏幕尺寸
7. 文本提示清晰易懂，帮助玩家理解游戏规则

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库
- 关系图绘制工具
- WebSocket 实时通信
- 后端角色信息 API

## 工作量估计

1.5 人天

## 相关文档

- [UI 设计稿链接](待补充)
- [阿瓦隆角色能力关系图](./角色能力关系图.md)
- [角色信息安全策略](待补充)
