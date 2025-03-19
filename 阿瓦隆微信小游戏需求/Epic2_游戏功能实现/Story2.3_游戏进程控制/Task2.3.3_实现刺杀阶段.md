# Task 2.3.3: 实现刺杀阶段

## 描述

实现阿瓦隆游戏的刺杀阶段，该阶段是游戏的关键转折点，当好人阵营成功完成 3 次任务后，坏人阵营的刺客有一次机会刺杀梅林。如果刺杀成功，坏人阵营将逆转局势获胜；如果刺杀失败，好人阵营最终获胜。

## 详细需求

### 功能需求

1. 刺杀阶段触发：

   - 当好人阵营成功完成 3 次任务后，自动进入刺杀阶段
   - 向所有玩家展示进入刺杀阶段的提示
   - 确保只有在满足条件时才能进入该阶段

2. 刺客操作：

   - 识别并启用刺客角色的特殊能力
   - 刺客可以从好人阵营中选择一名玩家作为刺杀目标
   - 提供确认机制，防止误操作
   - 限制刺杀时间，超时自动选择（可选功能）

3. 刺杀结果处理：
   - 判断刺杀结果（是否击中梅林）
   - 根据刺杀结果确定最终胜利方
   - 展示所有玩家的角色信息
   - 提供游戏结果的详细说明

### 界面要求

1. 刺杀阶段提示：

   - 向所有玩家展示刺杀阶段的开始
   - 提供阶段说明，解释刺杀规则
   - 为非刺客玩家提供等待界面

2. 刺客选择界面：

   - 为刺客提供可选择的目标列表（好人阵营玩家）
   - 清晰显示每个可选玩家的信息
   - 提供选择和确认机制
   - 包含计时器显示（如适用）

3. 结果展示界面：
   - 展示刺杀的结果（成功/失败）
   - 显示所有玩家的真实角色
   - 突出显示关键角色（如梅林、刺客）
   - 清晰标明胜利方

### 技术要求

1. 实现刺杀阶段的状态管理
2. 确保刺客选择的安全性和正确性
3. 实现角色信息的合理展示
4. 保证刺杀过程的流畅性和可靠性
5. 支持断线重连时的状态恢复

## 实现步骤

1. 设计刺杀阶段的数据结构和状态
2. 实现刺杀阶段的触发逻辑
3. 开发刺客选择界面和交互
4. 实现刺杀结果的判定和处理
5. 开发结果展示界面
6. 进行单元测试和集成测试

## 代码示例

### 刺杀管理器

```typescript
// 简化的刺杀管理器
class AssassinationManager {
  private gameStateManager: GameStateManager;
  private eventEmitter: EventEmitter;

  constructor(gameStateManager: GameStateManager) {
    this.gameStateManager = gameStateManager;
    this.eventEmitter = new EventEmitter();
  }

  // 开始刺杀阶段
  public startAssassination(): void {
    const gameState = this.gameStateManager.getGameState();

    // 检查是否满足进入刺杀阶段的条件
    const successCount = gameState.missionResults.filter(
      (r) => r.result === "SUCCESS"
    ).length;

    if (successCount < 3) {
      throw new Error("不满足进入刺杀阶段的条件");
    }

    // 转换到刺杀阶段
    this.gameStateManager.updateGameState({
      status: "assassination",
    });

    // 触发刺杀阶段开始事件
    this.eventEmitter.emit("assassinationStarted");
  }

  // 获取刺客玩家ID
  public getAssassinId(): string | null {
    const gameState = this.gameStateManager.getGameState();

    // 遍历所有玩家角色，找到刺客
    for (const [playerId, role] of gameState.playerRoles.entries()) {
      if (role.roleType === "assassin") {
        return playerId;
      }
    }

    return null;
  }

  // 获取可刺杀目标列表
  public getAssassinationTargets(): string[] {
    const gameState = this.gameStateManager.getGameState();
    const targets: string[] = [];

    // 遍历所有玩家角色，找到好人阵营玩家
    for (const [playerId, role] of gameState.playerRoles.entries()) {
      if (role.faction === "good") {
        targets.push(playerId);
      }
    }

    return targets;
  }

  // 执行刺杀
  public executeAssassination(targetId: string): boolean {
    const gameState = this.gameStateManager.getGameState();
    const targetRole = gameState.playerRoles.get(targetId);

    if (!targetRole) {
      throw new Error("目标玩家不存在");
    }

    // 判断是否刺中梅林
    const isMerlin = targetRole.roleType === "merlin";

    // 更新游戏状态
    this.gameStateManager.updateGameState({
      assassinationTarget: targetId,
    });

    // 创建游戏结果
    const gameResult = {
      winner: isMerlin ? "evil" : "good",
      reason: isMerlin ? "刺客成功刺杀梅林" : "刺客刺杀失败，好人阵营获胜",
      assassinationSuccess: isMerlin,
      assassinationTarget: targetId,
    };

    // 更新游戏结果
    this.gameStateManager.updateGameState({
      gameResult: gameResult,
      status: "result",
    });

    // 触发刺杀完成事件
    this.eventEmitter.emit("assassinationCompleted", {
      targetId,
      isMerlin,
      gameResult,
    });

    return isMerlin;
  }

  // 订阅刺杀事件
  public onAssassinationStart(callback: Function): void {
    this.eventEmitter.on("assassinationStarted", callback);
  }

  public onAssassinationComplete(callback: Function): void {
    this.eventEmitter.on("assassinationCompleted", callback);
  }
}
```

### 刺杀界面组件

```typescript
// 简化的刺杀界面组件
class AssassinationView {
  private app: PIXI.Application;
  private assassinationManager: AssassinationManager;
  private container: PIXI.Container;
  private playerContainers: Map<string, PIXI.Container> = new Map();
  private isAssassin: boolean = false;
  private selectedTarget: string | null = null;

  constructor(
    app: PIXI.Application,
    assassinationManager: AssassinationManager
  ) {
    this.app = app;
    this.assassinationManager = assassinationManager;
    this.container = new PIXI.Container();

    // 初始化界面
    this.initialize();
  }

  // 初始化刺杀界面
  private initialize(): void {
    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(0x000000, 0.7);
    background.drawRect(0, 0, 800, 600);
    background.endFill();
    this.container.addChild(background);

    // 创建标题
    const title = new PIXI.Text("刺杀阶段", {
      fontFamily: "Arial",
      fontSize: 32,
      fill: 0xff0000,
      fontWeight: "bold",
    });
    title.x = 400;
    title.y = 50;
    title.anchor.set(0.5);
    this.container.addChild(title);

    // 判断当前玩家是否是刺客
    const currentPlayerId = "current_player_id"; // 实际中从会话中获取
    this.isAssassin =
      currentPlayerId === this.assassinationManager.getAssassinId();

    // 创建说明文本
    const description = new PIXI.Text(
      this.isAssassin
        ? "你是刺客，请选择一名你认为是梅林的玩家进行刺杀"
        : "刺客正在选择目标，请等待...",
      {
        fontFamily: "Arial",
        fontSize: 20,
        fill: 0xffffff,
        align: "center",
      }
    );
    description.x = 400;
    description.y = 100;
    description.anchor.set(0.5, 0);
    this.container.addChild(description);

    // 如果是刺客，显示可选目标
    if (this.isAssassin) {
      this.createTargetSelection();
    } else {
      this.createWaitingView();
    }
  }

  // 创建目标选择界面
  private createTargetSelection(): void {
    const targets = this.assassinationManager.getAssassinationTargets();
    const targetsContainer = new PIXI.Container();
    targetsContainer.x = 100;
    targetsContainer.y = 150;
    this.container.addChild(targetsContainer);

    // 创建玩家选择网格
    targets.forEach((playerId, index) => {
      const player = this.createPlayerCard(playerId, index);
      player.x = (index % 3) * 200;
      player.y = Math.floor(index / 3) * 150;
      targetsContainer.addChild(player);
      this.playerContainers.set(playerId, player);

      // 添加点击事件
      player.interactive = true;
      player.buttonMode = true;
      player.on("pointerdown", () => this.selectTarget(playerId));
    });

    // 创建确认按钮
    const confirmButton = new PIXI.Container();
    confirmButton.x = 400;
    confirmButton.y = 500;
    this.container.addChild(confirmButton);

    const buttonBg = new PIXI.Graphics();
    buttonBg.beginFill(0xff0000);
    buttonBg.drawRoundedRect(-100, -25, 200, 50, 10);
    buttonBg.endFill();
    confirmButton.addChild(buttonBg);

    const buttonText = new PIXI.Text("确认刺杀", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    buttonText.anchor.set(0.5);
    confirmButton.addChild(buttonText);

    // 添加确认按钮点击事件
    confirmButton.interactive = true;
    confirmButton.buttonMode = true;
    confirmButton.on("pointerdown", () => this.confirmAssassination());
  }

  // 创建等待界面
  private createWaitingView(): void {
    const waitingText = new PIXI.Text("刺客正在选择目标...", {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
    });
    waitingText.x = 400;
    waitingText.y = 300;
    waitingText.anchor.set(0.5);
    this.container.addChild(waitingText);
  }

  // 创建玩家卡片
  private createPlayerCard(playerId: string, index: number): PIXI.Container {
    const container = new PIXI.Container();

    // 卡片背景
    const background = new PIXI.Graphics();
    background.beginFill(0x333333);
    background.drawRoundedRect(0, 0, 180, 120, 10);
    background.endFill();
    container.addChild(background);

    // 玩家名称
    const playerName = `玩家${index + 1}`; // 实际中从玩家数据获取
    const nameText = new PIXI.Text(playerName, {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    nameText.x = 90;
    nameText.y = 20;
    nameText.anchor.set(0.5, 0);
    container.addChild(nameText);

    // 玩家头像
    const avatar = new PIXI.Graphics();
    avatar.beginFill(0x666666);
    avatar.drawCircle(90, 70, 30);
    avatar.endFill();
    container.addChild(avatar);

    return container;
  }

  // 选择目标
  private selectTarget(playerId: string): void {
    // 取消之前的选择
    if (this.selectedTarget) {
      const prevSelected = this.playerContainers.get(this.selectedTarget);
      if (prevSelected) {
        const background = prevSelected.getChildAt(0) as PIXI.Graphics;
        background.clear();
        background.beginFill(0x333333);
        background.drawRoundedRect(0, 0, 180, 120, 10);
        background.endFill();
      }
    }

    // 设置新选择
    this.selectedTarget = playerId;
    const selected = this.playerContainers.get(playerId);
    if (selected) {
      const background = selected.getChildAt(0) as PIXI.Graphics;
      background.clear();
      background.beginFill(0xff0000);
      background.drawRoundedRect(0, 0, 180, 120, 10);
      background.endFill();
    }
  }

  // 确认刺杀
  private confirmAssassination(): void {
    if (!this.selectedTarget) {
      alert("请选择一个目标");
      return;
    }

    // 创建确认对话框
    this.showConfirmationDialog();
  }

  // 显示确认对话框
  private showConfirmationDialog(): void {
    const dialog = new PIXI.Container();
    dialog.x = 400;
    dialog.y = 300;
    this.container.addChild(dialog);

    // 对话框背景
    const background = new PIXI.Graphics();
    background.beginFill(0x000000, 0.9);
    background.drawRect(-200, -150, 400, 300);
    background.endFill();
    dialog.addChild(background);

    // 确认文本
    const confirmText = new PIXI.Text("确定要刺杀这名玩家吗？", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
      align: "center",
    });
    confirmText.anchor.set(0.5, 0);
    confirmText.y = -100;
    dialog.addChild(confirmText);

    // 警告文本
    const warningText = new PIXI.Text("此操作无法撤销，将决定游戏最终结果", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xff0000,
      align: "center",
    });
    warningText.anchor.set(0.5, 0);
    warningText.y = -50;
    dialog.addChild(warningText);

    // 确认按钮
    const confirmButton = new PIXI.Container();
    confirmButton.x = -100;
    confirmButton.y = 50;
    dialog.addChild(confirmButton);

    const confirmBg = new PIXI.Graphics();
    confirmBg.beginFill(0xff0000);
    confirmBg.drawRoundedRect(-75, -20, 150, 40, 10);
    confirmBg.endFill();
    confirmButton.addChild(confirmBg);

    const confirmBtnText = new PIXI.Text("确认", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    confirmBtnText.anchor.set(0.5);
    confirmButton.addChild(confirmBtnText);

    // 取消按钮
    const cancelButton = new PIXI.Container();
    cancelButton.x = 100;
    cancelButton.y = 50;
    dialog.addChild(cancelButton);

    const cancelBg = new PIXI.Graphics();
    cancelBg.beginFill(0x666666);
    cancelBg.drawRoundedRect(-75, -20, 150, 40, 10);
    cancelBg.endFill();
    cancelButton.addChild(cancelBg);

    const cancelBtnText = new PIXI.Text("取消", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    cancelBtnText.anchor.set(0.5);
    cancelButton.addChild(cancelBtnText);

    // 添加按钮事件
    confirmButton.interactive = true;
    confirmButton.buttonMode = true;
    confirmButton.on("pointerdown", () => {
      // 执行刺杀
      if (this.selectedTarget) {
        this.executeAssassination(this.selectedTarget);
      }
      this.container.removeChild(dialog);
    });

    cancelButton.interactive = true;
    cancelButton.buttonMode = true;
    cancelButton.on("pointerdown", () => {
      this.container.removeChild(dialog);
    });
  }

  // 执行刺杀
  private executeAssassination(targetId: string): void {
    // 调用刺杀管理器执行刺杀
    const isMerlin = this.assassinationManager.executeAssassination(targetId);

    // 显示结果会由游戏控制器处理，这里不需要额外操作
  }

  // 显示界面
  public show(): void {
    this.app.stage.addChild(this.container);
  }

  // 隐藏界面
  public hide(): void {
    if (this.container.parent) {
      this.container.parent.removeChild(this.container);
    }
  }
}
```

## 界面原型

```
+------------------------------------------+
|           刺杀阶段 - 刺客视角             |
+------------------------------------------+
|                                          |
|  你是刺客，请选择一名你认为是梅林的玩家    |
|                                          |
|  +--------+  +--------+  +--------+      |
|  | 玩家1   |  | 玩家2   |  | 玩家3   |     |
|  |        |  |        |  |        |      |
|  |   👤   |  |   👤   |  |   👤   |      |
|  +--------+  +--------+  +--------+      |
|                                          |
|  +--------+  +--------+  +--------+      |
|  | 玩家4   |  | 玩家5   |  | 玩家6   |     |
|  |        |  |        |  |        |      |
|  |   👤   |  |   👤   |  |   👤   |      |
|  +--------+  +--------+  +--------+      |
|                                          |
|          [    确认刺杀    ]              |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           刺杀确认                       |
|                                          |
|  确定要刺杀这名玩家吗？                   |
|                                          |
|  此操作无法撤销，将决定游戏最终结果        |
|                                          |
|                                          |
|      [  确认  ]        [  取消  ]        |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           刺杀阶段 - 其他玩家视角         |
+------------------------------------------+
|                                          |
|  刺杀阶段                                |
|                                          |
|  刺客正在选择目标，请等待...              |
|                                          |
|  好人阵营已完成3次任务                    |
|  如果刺客成功刺杀梅林，坏人阵营将获胜      |
|  如果刺客刺杀失败，好人阵营将获胜          |
|                                          |
|                                          |
|  ⌛ 等待刺客做出选择...                  |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           刺杀结果 - 刺杀成功             |
+------------------------------------------+
|                                          |
|  刺杀成功！坏人阵营获胜                   |
|                                          |
|  刺客成功找出了梅林:                      |
|  [玩家2] 梅林                            |
|                                          |
|  所有角色:                               |
|  [玩家1] 派西维尔                        |
|  [玩家2] 梅林                            |
|  [玩家3] 忠臣                            |
|  [玩家4] 莫甘娜                          |
|  [玩家5] 刺客                            |
|  [玩家6] 莫德雷德                        |
|  [玩家7] 忠臣                            |
|  [玩家8] 忠臣                            |
|                                          |
|  [查看游戏统计]      [返回大厅]           |
+------------------------------------------+
```

## 验收标准

1. 当好人阵营完成 3 次任务后，游戏成功进入刺杀阶段
2. 刺客能够正确看到并选择可刺杀的目标
3. 刺杀的结果被正确判定和显示
4. 所有玩家能够看到明确的刺杀阶段提示和说明
5. 刺杀确认机制能够防止误操作
6. 结果展示界面清晰显示所有角色和胜利方
7. 断线重连后能够恢复到正确的刺杀状态
8. 界面操作流畅，视觉效果符合游戏风格

## 技术依赖

- TypeScript/JavaScript
- PIXI.js 渲染引擎
- 游戏状态管理系统
- 事件发布-订阅系统
- GSAP 动画库（可选）

## 工作量估计

1 人天

## 相关文档

- [游戏进程控制技术方案](./技术方案.md)
- [游戏阶段转换](./Task2.3.2_实现游戏阶段转换.md)
- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
