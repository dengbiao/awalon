# Task 2.2.3: 实现任务执行机制

## 描述

实现阿瓦隆游戏中的任务执行机制，允许被选中的队员执行任务，并根据任务结果决定任务成功或失败。

## 详细需求

### 功能需求

1. 任务执行机制：

   - 只有被选中的队员可以执行任务
   - 好人阵营的队员只能选择"成功"
   - 坏人阵营的队员可以选择"成功"或"失败"
   - 任务结果根据失败票数决定（通常一张失败票即导致任务失败，第四轮特殊规则除外）
   - 保密处理队员的选择，只公布最终结果和成功/失败票数

2. 第四轮特殊规则：

   - 在 7 人及以上游戏中，第四轮任务需要至少两张失败票才会失败
   - 清晰提示玩家当前任务的成功/失败条件

3. 任务结果处理：
   - 记录任务结果（成功/失败）
   - 更新任务计分板
   - 检查游戏是否结束（3 次成功或 3 次失败）
   - 如果游戏未结束，进入下一轮任务

### 界面要求

1. 任务卡片设计：

   - 提供明显区分的成功/失败任务卡片
   - 卡片设计美观，符合游戏风格
   - 选择后有明确的视觉反馈
   - 对于好人阵营，失败卡片应显示为不可选

2. 任务状态展示：

   - 显示当前任务轮次和队员信息
   - 显示任务执行倒计时
   - 显示已执行和未执行任务的队员（不显示具体选择）

3. 任务结果展示：
   - 清晰展示任务结果（成功/失败）
   - 显示成功和失败的票数（不显示具体是谁选择的）
   - 提供适当的动画效果
   - 更新任务计分板

### 技术要求

1. 实现安全的任务执行机制，防止作弊
2. 确保只有队员能够执行任务
3. 确保好人阵营只能选择成功
4. 保密处理队员的选择，保护游戏体验
5. 处理玩家断线或超时情况

## 实现步骤

1. 设计任务执行数据结构和状态管理
2. 实现任务执行界面组件
3. 实现任务结果收集和统计逻辑
4. 实现任务结果展示
5. 实现任务计分板更新
6. 实现游戏结束检查
7. 进行测试和优化

## 代码示例

### 任务执行逻辑

```typescript
// 队员执行任务
function executeMission(
  game: Game,
  playerId: string,
  result: MissionResultType
): void {
  // 验证游戏状态
  if (game.currentMission.status !== MissionStatus.EXECUTING) {
    throw new Error("当前不是任务执行阶段");
  }

  // 验证玩家是否是队员
  if (!game.currentMission.currentTeam.memberIds.includes(playerId)) {
    throw new Error("只有队员才能执行任务");
  }

  // 验证玩家是否已执行任务
  if (game.currentMission.executionResults.has(playerId)) {
    throw new Error("玩家已经执行过任务");
  }

  // 验证好人阵营只能选择成功
  const playerRole = game.playerRoles.get(playerId);
  if (isGoodRole(playerRole.roleType) && result === MissionResultType.FAIL) {
    throw new Error("好人阵营只能选择成功");
  }

  // 记录玩家任务结果
  game.currentMission.executionResults.set(playerId, result);

  // 检查是否所有队员都已执行任务
  if (
    game.currentMission.executionResults.size ===
    game.currentMission.currentTeam.memberIds.length
  ) {
    // 计算任务最终结果
    calculateMissionResult(game);
  }
}
```

### 任务结果计算

```typescript
// 计算任务结果
function calculateMissionResult(game: Game): void {
  const results = game.currentMission.executionResults;
  let successCount = 0;
  let failCount = 0;

  // 统计成功和失败票数
  results.forEach((result) => {
    if (result === MissionResultType.SUCCESS) successCount++;
    else failCount++;
  });

  // 获取当前任务配置
  const missionConfig = getMissionConfig(
    game.currentMission.missionNumber,
    game.players.length
  );

  // 判断任务结果
  const isSuccess = failCount < missionConfig.requiredFailures;

  // 创建任务结果对象
  const missionResult: MissionResult = {
    missionNumber: game.currentMission.missionNumber,
    teamMemberIds: game.currentMission.currentTeam.memberIds,
    captainId: game.currentMission.currentTeam.captainId,
    result: isSuccess ? MissionResultType.SUCCESS : MissionResultType.FAIL,
    successCount: successCount,
    failCount: failCount,
  };

  // 更新任务状态
  game.currentMission.status = isSuccess
    ? MissionStatus.SUCCEEDED
    : MissionStatus.FAILED;
  game.currentMission.result = missionResult;

  // 记录任务结果
  game.missionResults.push(missionResult);

  // 检查游戏是否结束
  checkGameEnd(game);
}
```

### 游戏结束检查

```typescript
// 检查游戏是否结束
function checkGameEnd(game: Game): void {
  // 统计成功和失败的任务数
  let successCount = 0;
  let failCount = 0;

  game.missionResults.forEach((result) => {
    if (result.result === MissionResultType.SUCCESS) successCount++;
    else failCount++;
  });

  // 判断游戏是否结束
  if (successCount >= 3) {
    // 好人阵营暂时获胜，进入刺杀阶段
    game.status = GameStatus.ASSASSINATION;
    // 触发刺杀阶段事件
    emitAssassinationPhaseEvent(game);
  } else if (failCount >= 3) {
    // 坏人阵营获胜
    game.status = GameStatus.ENDED;
    game.result = {
      winner: Faction.EVIL,
      reason: "三次任务失败",
    };
    // 触发游戏结束事件
    emitGameEndEvent(game);
  } else {
    // 游戏继续，准备下一个任务
    prepareNextMission(game);
  }
}
```

### 任务执行界面组件

```typescript
class MissionExecutionView extends PIXI.Container {
  private game: Game;
  private successCard: PIXI.Container;
  private failCard: PIXI.Container;
  private selectedResult: MissionResultType | null = null;
  private countdownText: PIXI.Text;
  private countdownTimer: number;

  constructor(game: Game) {
    super();
    this.game = game;

    // 初始化界面
    this.initView();

    // 开始倒计时
    this.startCountdown(20); // 20秒倒计时
  }

  private initView(): void {
    // 创建标题
    const title = new PIXI.Text("任务执行阶段", {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
    });
    title.x = 10;
    title.y = 10;
    this.addChild(title);

    // 创建任务信息
    this.createMissionInfo();

    // 检查当前玩家是否是队员
    const isTeamMember =
      this.game.currentMission.currentTeam.memberIds.includes(
        this.game.currentPlayerId
      );

    if (isTeamMember) {
      // 创建任务卡片
      this.createMissionCards();
    } else {
      // 创建等待提示
      this.createWaitingPrompt();
    }

    // 创建倒计时文本
    this.countdownText = new PIXI.Text("20", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    this.countdownText.x = 200;
    this.countdownText.y = 350;
    this.addChild(this.countdownText);

    // 创建任务执行状态显示
    this.createExecutionStatus();
  }

  private createMissionInfo(): void {
    // 创建任务信息容器
    const container = new PIXI.Container();
    container.x = 10;
    container.y = 50;
    this.addChild(container);

    // 添加任务编号信息
    const missionText = new PIXI.Text(
      `任务 ${this.game.currentMission.missionNumber}`,
      {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xffffff,
      }
    );
    container.addChild(missionText);

    // 添加队员信息
    const teamText = new PIXI.Text("队员: ", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    teamText.y = 25;
    container.addChild(teamText);

    const memberIds = this.game.currentMission.currentTeam.memberIds;
    const memberNames = memberIds.map((id) => {
      const player = this.game.players.find((p) => p.id === id);
      return player.name;
    });

    const membersText = new PIXI.Text(memberNames.join(", "), {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    membersText.x = 50;
    membersText.y = 25;
    container.addChild(membersText);

    // 添加任务成功条件
    const missionConfig = getMissionConfig(
      this.game.currentMission.missionNumber,
      this.game.players.length
    );

    const conditionText = new PIXI.Text(
      `任务成功条件: 失败票数少于 ${missionConfig.requiredFailures}`,
      {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xffffff,
      }
    );
    conditionText.y = 50;
    container.addChild(conditionText);
  }

  private createMissionCards(): void {
    // 创建成功卡片
    this.successCard = this.createMissionCard(MissionResultType.SUCCESS);
    this.successCard.x = 100;
    this.successCard.y = 150;
    this.addChild(this.successCard);

    // 检查当前玩家是否是好人阵营
    const playerRole = this.game.playerRoles.get(this.game.currentPlayerId);
    const isGoodPlayer = isGoodRole(playerRole.roleType);

    // 创建失败卡片（如果是坏人阵营）
    this.failCard = this.createMissionCard(MissionResultType.FAIL);
    this.failCard.x = 300;
    this.failCard.y = 150;
    this.failCard.interactive = !isGoodPlayer; // 好人不能选择失败
    this.failCard.alpha = isGoodPlayer ? 0.5 : 1.0; // 好人的失败卡片显示为半透明
    this.addChild(this.failCard);

    // 如果是好人，添加提示
    if (isGoodPlayer) {
      const hintText = new PIXI.Text("作为好人阵营，你只能选择成功", {
        fontFamily: "Arial",
        fontSize: 14,
        fill: 0xff0000,
      });
      hintText.x = 100;
      hintText.y = 280;
      this.addChild(hintText);
    }
  }

  private createMissionCard(resultType: MissionResultType): PIXI.Container {
    // 创建任务卡片容器
    const container = new PIXI.Container();

    // 添加卡片背景
    const background = new PIXI.Graphics();
    background.beginFill(
      resultType === MissionResultType.SUCCESS ? 0x2ecc71 : 0xe74c3c
    );
    background.drawRoundedRect(0, 0, 150, 100, 10);
    background.endFill();
    container.addChild(background);

    // 添加卡片图标
    const icon = new PIXI.Sprite(
      PIXI.Texture.from(
        resultType === MissionResultType.SUCCESS ? "success_icon" : "fail_icon"
      )
    );
    icon.width = 50;
    icon.height = 50;
    icon.x = 50;
    icon.y = 10;
    icon.anchor.set(0.5, 0);
    container.addChild(icon);

    // 添加卡片文本
    const text = new PIXI.Text(
      resultType === MissionResultType.SUCCESS ? "成功" : "失败",
      {
        fontFamily: "Arial",
        fontSize: 20,
        fill: 0xffffff,
      }
    );
    text.x = 75;
    text.y = 70;
    text.anchor.set(0.5, 0);
    container.addChild(text);

    // 添加交互
    container.interactive = true;
    container.buttonMode = true;
    container.on("pointerdown", () => this.selectResult(resultType));

    return container;
  }

  private createWaitingPrompt(): void {
    // 创建等待提示容器
    const container = new PIXI.Container();
    container.x = 100;
    container.y = 150;
    this.addChild(container);

    // 添加等待提示文本
    const waitingText = new PIXI.Text("队员正在执行任务，请等待...", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    container.addChild(waitingText);

    // 添加加载动画
    const loadingAnimation = new PIXI.Graphics();
    loadingAnimation.beginFill(0x3498db);
    loadingAnimation.drawCircle(0, 0, 10);
    loadingAnimation.endFill();
    loadingAnimation.x = waitingText.width / 2;
    loadingAnimation.y = 50;
    container.addChild(loadingAnimation);

    // 创建加载动画
    gsap.to(loadingAnimation, {
      alpha: 0.2,
      duration: 1,
      repeat: -1,
      yoyo: true,
    });
  }

  private createExecutionStatus(): void {
    // 创建任务执行状态容器
    const container = new PIXI.Container();
    container.x = 10;
    container.y = 400;
    this.addChild(container);

    // 添加标题
    const title = new PIXI.Text("任务执行状态:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    container.addChild(title);

    // 添加队员执行状态
    const memberIds = this.game.currentMission.currentTeam.memberIds;
    memberIds.forEach((memberId, index) => {
      const player = this.game.players.find((p) => p.id === memberId);
      const hasExecuted =
        this.game.currentMission.executionResults.has(memberId);

      const statusText = new PIXI.Text(
        `${player.name}: ${hasExecuted ? "已执行" : "未执行"}`,
        {
          fontFamily: "Arial",
          fontSize: 14,
          fill: hasExecuted ? 0x2ecc71 : 0xe74c3c,
        }
      );
      statusText.x = (index % 2) * 200;
      statusText.y = 25 + Math.floor(index / 2) * 20;
      container.addChild(statusText);
    });
  }

  private selectResult(resultType: MissionResultType): void {
    // 如果已经选择，则不允许再次选择
    if (this.selectedResult !== null) {
      return;
    }

    // 检查当前玩家是否是好人阵营
    const playerRole = this.game.playerRoles.get(this.game.currentPlayerId);
    const isGoodPlayer = isGoodRole(playerRole.roleType);

    // 好人不能选择失败
    if (isGoodPlayer && resultType === MissionResultType.FAIL) {
      this.showMessage("作为好人阵营，你只能选择成功");
      return;
    }

    // 记录选择
    this.selectedResult = resultType;

    // 更新卡片视觉效果
    this.successCard.alpha =
      resultType === MissionResultType.SUCCESS ? 1.0 : 0.5;
    if (this.failCard) {
      this.failCard.alpha = resultType === MissionResultType.FAIL ? 1.0 : 0.5;
    }

    // 添加选中效果
    const selectedCard =
      resultType === MissionResultType.SUCCESS
        ? this.successCard
        : this.failCard;
    const glow = new PIXI.filters.GlowFilter();
    glow.color = 0xffffff;
    glow.distance = 15;
    glow.outerStrength = 2;
    selectedCard.filters = [glow];

    // 发送任务执行结果
    try {
      executeMission(this.game, this.game.currentPlayerId, resultType);

      // 更新任务执行状态显示
      this.updateExecutionStatus();
    } catch (error) {
      // 显示错误信息
      this.showMessage(error.message);
    }
  }

  private updateExecutionStatus(): void {
    // 移除旧的任务执行状态显示
    this.removeChild(this.getChildAt(this.children.length - 1));

    // 创建新的任务执行状态显示
    this.createExecutionStatus();
  }

  private startCountdown(seconds: number): void {
    // 设置初始倒计时
    this.countdownText.text = seconds.toString();

    // 启动倒计时定时器
    this.countdownTimer = setInterval(() => {
      const currentTime = parseInt(this.countdownText.text);

      if (currentTime <= 1) {
        // 倒计时结束
        clearInterval(this.countdownTimer);
        this.countdownText.text = "0";

        // 如果是队员且还没选择，自动选择
        const isTeamMember =
          this.game.currentMission.currentTeam.memberIds.includes(
            this.game.currentPlayerId
          );
        if (isTeamMember && this.selectedResult === null) {
          this.handleTimeoutResult();
        }
      } else {
        // 更新倒计时
        this.countdownText.text = (currentTime - 1).toString();
      }
    }, 1000);
  }

  private handleTimeoutResult(): void {
    // 超时自动选择（好人选成功，坏人随机）
    const playerRole = this.game.playerRoles.get(this.game.currentPlayerId);
    const isGoodPlayer = isGoodRole(playerRole.roleType);

    if (isGoodPlayer) {
      // 好人只能选择成功
      this.selectResult(MissionResultType.SUCCESS);
    } else {
      // 坏人随机选择
      const randomResult =
        Math.random() < 0.5
          ? MissionResultType.SUCCESS
          : MissionResultType.FAIL;
      this.selectResult(randomResult);
    }
  }

  private showMessage(message: string): void {
    // 显示消息
    console.log(message);
    // 实际实现中应该显示一个消息弹窗
  }

  public destroy(): void {
    // 清除倒计时定时器
    if (this.countdownTimer) {
      clearInterval(this.countdownTimer);
    }

    super.destroy();
  }
}
```

## 界面原型

```
+------------------------------------------+
|        第2轮任务 - 执行阶段 (队员视角)    |
+------------------------------------------+
|                                          |
|  任务2                                   |
|  队员: 玩家1, 玩家3, 玩家6               |
|  任务成功条件: 失败票数少于1              |
|                                          |
|  请选择任务结果:                         |
|                                          |
|  +---------------+  +---------------+    |
|  |               |  |               |    |
|  |     [图标]     |  |     [图标]     |    |
|  |               |  |               |    |
|  |     成功      |  |     失败      |    |
|  |               |  |               |    |
|  +---------------+  +---------------+    |
|                                          |
|  倒计时: 15                              |
|                                          |
|  任务执行状态:                           |
|  玩家1(你): 未执行    玩家3: 已执行      |
|  玩家6: 未执行                           |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|        第2轮任务 - 执行阶段 (旁观视角)    |
+------------------------------------------+
|                                          |
|  任务2                                   |
|  队员: 玩家1, 玩家3, 玩家6               |
|  任务成功条件: 失败票数少于1              |
|                                          |
|  队员正在执行任务，请等待...              |
|                                          |
|           [加载动画]                     |
|                                          |
|  倒计时: 15                              |
|                                          |
|  任务执行状态:                           |
|  玩家1: 未执行       玩家3: 已执行      |
|  玩家6: 未执行                           |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           任务结果 - 任务成功            |
+------------------------------------------+
|                                          |
|  任务2                                   |
|  队员: 玩家1, 玩家3, 玩家6               |
|                                          |
|  任务结果: 成功                          |
|                                          |
|  成功票数: 3                             |
|  失败票数: 0                             |
|                                          |
|  任务计分板:                             |
|  [成功] [失败] [成功] [ ] [ ]            |
|                                          |
|  好人阵营已完成2个任务                    |
|  坏人阵营已完成1个任务                    |
|                                          |
|                [继续]                    |
+------------------------------------------+
```

```
+------------------------------------------+
|           任务结果 - 任务失败            |
+------------------------------------------+
|                                          |
|  任务2                                   |
|  队员: 玩家1, 玩家3, 玩家6               |
|                                          |
|  任务结果: 失败                          |
|                                          |
|  成功票数: 2                             |
|  失败票数: 1                             |
|                                          |
|  任务计分板:                             |
|  [成功] [失败] [失败] [ ] [ ]            |
|                                          |
|  好人阵营已完成1个任务                    |
|  坏人阵营已完成2个任务                    |
|                                          |
|                [继续]                    |
+------------------------------------------+
```

## 验收标准

1. 任务执行界面清晰展示任务信息和选项
2. 只有队员能够执行任务
3. 好人阵营只能选择成功
4. 坏人阵营可以选择成功或失败
5. 任务结果正确计算并展示
6. 第四轮特殊规则正确实现
7. 任务计分板正确更新
8. 游戏结束条件正确判断
9. 断线重连后能够恢复当前任务状态
10. 任务执行过程保密，不泄露队员的具体选择

## 技术依赖

- PIXI.js 渲染引擎
- 游戏状态管理系统
- 网络同步模块
- 用户界面组件库
- 定时器管理工具

## 工作量估计

1 人天

## 相关文档

- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
- [任务系统技术方案](./技术方案.md)
