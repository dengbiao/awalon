# Task 2.1.2: 实现角色信息展示

## 描述

实现阿瓦隆游戏中的角色信息展示功能，在游戏开始时向玩家展示其角色身份信息，并提供游戏中随时查看角色信息的功能。

## 详细需求

### 功能需求

1. 游戏开始时展示玩家的角色信息：
   - 角色名称
   - 角色图标
   - 角色描述
   - 所属阵营（好人/坏人）
   - 特殊能力说明
2. 提供查看角色信息的按钮，允许玩家在游戏中随时查看自己的角色
3. 展示角色之间的关系（如梅林看到所有坏人）
4. 适当提供游戏提示，帮助玩家理解角色
5. 支持角色信息的本地缓存，避免断线重连后需要重新获取

### 界面要求

1. 角色信息卡片设计精美，符合角色特点和游戏风格
2. 使用动画效果展示角色信息，增强沉浸感
3. 角色信息展示界面清晰易读，文字大小合适
4. 角色信息按钮位置醒目但不影响游戏操作
5. 角色关系展示直观，使用图标或特殊标记区分
6. 支持不同屏幕尺寸的适配

### 技术要求

1. 使用 PIXI.js 实现角色信息展示界面
2. 实现角色信息的动画展示效果
3. 优化资源加载，确保角色图片加载迅速
4. 实现本地存储，避免重复加载角色信息
5. 确保界面交互灵敏，无卡顿现象

## 实现步骤

1. 设计角色信息展示界面 UI
2. 实现角色信息卡片组件
3. 实现角色关系展示逻辑
4. 实现角色信息查看按钮
5. 实现本地缓存逻辑
6. 添加动画和过渡效果
7. 进行测试和调优

## 代码示例

### 角色信息卡片组件

```typescript
class RoleInfoCard extends PIXI.Container {
  private background: PIXI.Sprite;
  private roleIcon: PIXI.Sprite;
  private roleName: PIXI.Text;
  private faction: PIXI.Text;
  private description: PIXI.Text;
  private abilityText: PIXI.Text;

  constructor(roleInfo: RoleInfo) {
    super();

    // 创建背景
    this.background = new PIXI.Sprite(PIXI.Texture.from("role_card_bg"));
    this.background.width = 300;
    this.background.height = 450;
    this.addChild(this.background);

    // 创建角色图标
    this.roleIcon = new PIXI.Sprite(PIXI.Texture.from(`role_${roleInfo.type}`));
    this.roleIcon.width = 120;
    this.roleIcon.height = 120;
    this.roleIcon.anchor.set(0.5);
    this.roleIcon.x = this.background.width / 2;
    this.roleIcon.y = 100;
    this.addChild(this.roleIcon);

    // 创建角色名称
    this.roleName = new PIXI.Text(roleInfo.name, {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
      align: "center",
      fontWeight: "bold",
    });
    this.roleName.anchor.set(0.5, 0);
    this.roleName.x = this.background.width / 2;
    this.roleName.y = 180;
    this.addChild(this.roleName);

    // 创建阵营信息
    const factionColor = roleInfo.faction === "good" ? 0x4a90e2 : 0xe74c3c;
    const factionText = roleInfo.faction === "good" ? "好人阵营" : "坏人阵营";
    this.faction = new PIXI.Text(factionText, {
      fontFamily: "Arial",
      fontSize: 18,
      fill: factionColor,
      align: "center",
    });
    this.faction.anchor.set(0.5, 0);
    this.faction.x = this.background.width / 2;
    this.faction.y = 215;
    this.addChild(this.faction);

    // 创建角色描述
    this.description = new PIXI.Text(roleInfo.description, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
      align: "center",
      wordWrap: true,
      wordWrapWidth: 260,
    });
    this.description.anchor.set(0.5, 0);
    this.description.x = this.background.width / 2;
    this.description.y = 250;
    this.addChild(this.description);

    // 创建特殊能力文本
    if (roleInfo.specialAbility) {
      this.abilityText = new PIXI.Text(`特殊能力: ${roleInfo.specialAbility}`, {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xf1c40f,
        align: "center",
        wordWrap: true,
        wordWrapWidth: 260,
      });
      this.abilityText.anchor.set(0.5, 0);
      this.abilityText.x = this.background.width / 2;
      this.abilityText.y = 340;
      this.addChild(this.abilityText);
    }
  }

  // 显示动画
  show(): Promise<void> {
    this.alpha = 0;
    this.scale.set(0.8);

    return new Promise((resolve) => {
      gsap.to(this, {
        alpha: 1,
        scale: 1,
        duration: 0.5,
        ease: "back.out(1.7)",
        onComplete: resolve,
      });
    });
  }

  // 隐藏动画
  hide(): Promise<void> {
    return new Promise((resolve) => {
      gsap.to(this, {
        alpha: 0,
        scale: 0.8,
        duration: 0.3,
        onComplete: resolve,
      });
    });
  }
}
```

### 角色关系展示组件

```typescript
class RelationshipView extends PIXI.Container {
  private visiblePlayers: VisiblePlayer[];
  private playerAvatars: Map<string, PIXI.Container> = new Map();

  constructor(visiblePlayers: VisiblePlayer[]) {
    super();
    this.visiblePlayers = visiblePlayers;
    this.initialize();
  }

  private initialize(): void {
    const title = new PIXI.Text("你能看到的玩家:", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
      align: "center",
    });
    title.x = 10;
    title.y = 10;
    this.addChild(title);

    // 如果没有可见玩家
    if (this.visiblePlayers.length === 0) {
      const noVisibleText = new PIXI.Text("无特殊信息", {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xcccccc,
        align: "center",
      });
      noVisibleText.x = 10;
      noVisibleText.y = 40;
      this.addChild(noVisibleText);
      return;
    }

    // 显示可见玩家
    this.visiblePlayers.forEach((player, index) => {
      const container = new PIXI.Container();

      // 玩家头像
      const avatar = new PIXI.Sprite(
        PIXI.Texture.from(player.avatar || "default_avatar")
      );
      avatar.width = 50;
      avatar.height = 50;
      avatar.x = 0;
      avatar.y = 0;
      container.addChild(avatar);

      // 玩家名称
      const nameText = new PIXI.Text(player.name, {
        fontFamily: "Arial",
        fontSize: 14,
        fill: 0xffffff,
        align: "center",
      });
      nameText.anchor.set(0.5, 0);
      nameText.x = avatar.width / 2;
      nameText.y = avatar.height + 5;
      container.addChild(nameText);

      // 提示文本
      if (player.hint) {
        const hintText = new PIXI.Text(player.hint, {
          fontFamily: "Arial",
          fontSize: 12,
          fill: 0xf1c40f,
          align: "center",
        });
        hintText.anchor.set(0.5, 0);
        hintText.x = avatar.width / 2;
        hintText.y = avatar.height + 25;
        container.addChild(hintText);
      }

      // 设置位置
      container.x = 10 + (index % 3) * 70;
      container.y = 40 + Math.floor(index / 3) * 90;

      this.addChild(container);
      this.playerAvatars.set(player.playerId, container);
    });
  }

  // 高亮显示某个玩家
  highlightPlayer(playerId: string): void {
    const container = this.playerAvatars.get(playerId);
    if (container) {
      gsap.to(container.scale, {
        x: 1.1,
        y: 1.1,
        duration: 0.3,
        repeat: 1,
        yoyo: true,
      });
    }
  }
}
```

### 游戏中的角色信息按钮和查看逻辑

```typescript
class GameScene extends PIXI.Container {
  private roleInfoButton: PIXI.Container;
  private roleInfoPanel: PIXI.Container;
  private roleInfoVisible: boolean = false;
  private roleInfo: RoleInfo;
  private visiblePlayers: VisiblePlayer[];

  constructor(roleInfo: RoleInfo, visiblePlayers: VisiblePlayer[]) {
    super();
    this.roleInfo = roleInfo;
    this.visiblePlayers = visiblePlayers;

    // 创建角色信息按钮
    this.roleInfoButton = this.createRoleInfoButton();
    this.addChild(this.roleInfoButton);

    // 创建角色信息面板（初始隐藏）
    this.roleInfoPanel = this.createRoleInfoPanel();
    this.roleInfoPanel.visible = false;
    this.addChild(this.roleInfoPanel);

    // 添加按钮点击事件
    this.roleInfoButton.interactive = true;
    this.roleInfoButton.buttonMode = true;
    this.roleInfoButton.on("pointerdown", this.toggleRoleInfo.bind(this));
  }

  private createRoleInfoButton(): PIXI.Container {
    const container = new PIXI.Container();

    // 按钮背景
    const bg = new PIXI.Graphics();
    bg.beginFill(0x333333, 0.8);
    bg.drawRoundedRect(0, 0, 50, 50, 10);
    bg.endFill();
    container.addChild(bg);

    // 按钮图标
    const icon = new PIXI.Sprite(PIXI.Texture.from("role_info_icon"));
    icon.width = 30;
    icon.height = 30;
    icon.anchor.set(0.5);
    icon.x = 25;
    icon.y = 25;
    container.addChild(icon);

    // 设置按钮位置（靠近屏幕右上角）
    container.x = app.screen.width - 60;
    container.y = 10;

    return container;
  }

  private createRoleInfoPanel(): PIXI.Container {
    const container = new PIXI.Container();

    // 面板背景
    const bg = new PIXI.Graphics();
    bg.beginFill(0x000000, 0.85);
    bg.drawRoundedRect(0, 0, 320, 480, 10);
    bg.endFill();
    container.addChild(bg);

    // 添加角色信息卡片
    const roleCard = new RoleInfoCard(this.roleInfo);
    roleCard.x = 10;
    roleCard.y = 20;
    container.addChild(roleCard);

    // 添加角色关系显示
    const relationView = new RelationshipView(this.visiblePlayers);
    relationView.x = 10;
    relationView.y = 280;
    container.addChild(relationView);

    // 添加关闭按钮
    const closeButton = new PIXI.Graphics();
    closeButton.beginFill(0xe74c3c);
    closeButton.drawCircle(0, 0, 15);
    closeButton.endFill();

    const closeIcon = new PIXI.Text("×", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
      align: "center",
    });
    closeIcon.anchor.set(0.5);
    closeButton.addChild(closeIcon);

    closeButton.x = container.width - 20;
    closeButton.y = 20;
    closeButton.interactive = true;
    closeButton.buttonMode = true;
    closeButton.on("pointerdown", this.toggleRoleInfo.bind(this));
    container.addChild(closeButton);

    // 设置面板位置（居中）
    container.x = (app.screen.width - container.width) / 2;
    container.y = (app.screen.height - container.height) / 2;

    return container;
  }

  private toggleRoleInfo(): void {
    this.roleInfoVisible = !this.roleInfoVisible;

    if (this.roleInfoVisible) {
      // 显示角色信息面板
      this.roleInfoPanel.visible = true;
      gsap.fromTo(
        this.roleInfoPanel,
        { alpha: 0, scale: 0.9 },
        { alpha: 1, scale: 1, duration: 0.3, ease: "back.out" }
      );
    } else {
      // 隐藏角色信息面板
      gsap.to(this.roleInfoPanel, {
        alpha: 0,
        scale: 0.9,
        duration: 0.2,
        onComplete: () => {
          this.roleInfoPanel.visible = false;
        },
      });
    }
  }
}
```

## 界面原型

```
+------------------------------------------+
|        角色信息展示 - 梅林视角            |
+------------------------------------------+
|                                          |
|                [光效]                    |
|          +----------------+              |
|          |                |              |
|          |    [角色图像]   |              |
|          |                |              |
|          |     梅林       |              |
|          |    好人阵营     |              |
|          |                |              |
|          +----------------+              |
|                                          |
|  角色能力:                               |
|  你能看到所有邪恶阵营的成员               |
|  (除了莫德雷德)                          |
|                                          |
|  你看到的邪恶阵营成员:                    |
|  +------+  +------+  +------+           |
|  |玩家2  |  |玩家5  |  |玩家7  |           |
|  |邪恶   |  |邪恶   |  |邪恶   |           |
|  +------+  +------+  +------+           |
|                                          |
|  提示: 保持你的身份秘密，不要被刺客发现！   |
|                                          |
|          [我已了解，开始游戏]             |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|        角色信息展示 - 忠臣视角            |
+------------------------------------------+
|                                          |
|                [光效]                    |
|          +----------------+              |
|          |                |              |
|          |    [角色图像]   |              |
|          |                |              |
|          |     忠臣       |              |
|          |    好人阵营     |              |
|          |                |              |
|          +----------------+              |
|                                          |
|  角色能力:                               |
|  你是亚瑟王的忠实追随者                   |
|  你没有特殊能力，但你的使命是帮助好人阵营   |
|  找出邪恶势力并完成任务                   |
|                                          |
|  提示: 仔细观察其他玩家的行为，寻找线索！   |
|                                          |
|          [我已了解，开始游戏]             |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|        角色信息展示 - 游戏中查看          |
+------------------------------------------+
|                              [关闭]      |
|                                          |
|          +----------------+              |
|          |                |              |
|          |    [角色图像]   |              |
|          |                |              |
|          |     梅林       |              |
|          |    好人阵营     |              |
|          |                |              |
|          +----------------+              |
|                                          |
|  角色能力:                               |
|  你能看到所有邪恶阵营的成员               |
|  (除了莫德雷德)                          |
|                                          |
|  你看到的邪恶阵营成员:                    |
|  +------+  +------+  +------+           |
|  |玩家2  |  |玩家5  |  |玩家7  |           |
|  |邪恶   |  |邪恶   |  |邪恶   |           |
|  +------+  +------+  +------+           |
|                                          |
|  提示: 保持你的身份秘密，不要被刺客发现！   |
|                                          |
+------------------------------------------+
```

## 游戏中的角色信息查看功能（文本描述）

游戏中的角色信息查看功能允许玩家通过点击界面上的专用按钮，随时查看自己的角色信息和关系网络。此功能设计如下：

1. **按钮设计**：

   - 在游戏界面右上角放置一个醒目但不干扰游戏进行的角色信息按钮
   - 按钮使用半透明背景和角色图标，明确其功能
   - 按钮支持点击效果，提供用户反馈

2. **信息面板**：

   - 点击按钮后，屏幕中央弹出角色信息面板
   - 面板使用半透明黑色背景，确保内容清晰可见
   - 包含角色卡片和可见玩家关系两部分内容
   - 面板出现时有平滑的动画效果

3. **交互方式**：

   - 点击按钮打开/关闭面板
   - 面板右上角有关闭按钮
   - 面板打开时，游戏主界面变暗但仍可见
   - 点击面板外区域也可关闭面板

4. **信息更新**：

   - 角色信息从游戏开始后保持不变
   - 可见玩家信息随游戏进程可能更新（例如，某些特殊角色能力触发后）
   - 面板每次打开时自动刷新最新信息

5. **离线支持**：
   - 角色信息会缓存在本地存储中
   - 即使断线重连，也能立即显示玩家角色信息
   - 重新连接后自动验证信息正确性

## 验收标准

1. 游戏开始时正确展示玩家角色信息，包含所有必要内容
2. 角色信息展示动画流畅，提升游戏体验
3. 游戏过程中随时可以查看角色信息
4. 角色关系显示准确，符合游戏规则
5. 界面在不同设备上自适应显示
6. 断线重连后能快速恢复角色信息
7. 界面美观，符合游戏整体风格

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库
- 本地存储 API
- 后端角色信息 API

## 工作量估计

1 人天

## 相关文档

- [UI 设计稿链接](待补充)
- [阿瓦隆角色设定](./角色设定.md)
- [PIXI.js 文档](https://pixijs.io/guides/)
