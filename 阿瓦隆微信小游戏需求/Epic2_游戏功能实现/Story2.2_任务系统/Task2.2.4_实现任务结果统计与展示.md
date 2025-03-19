# Task 2.2.4: 实现任务结果统计与展示

## 描述

实现阿瓦隆游戏中的任务结果统计与展示功能，包括任务成功/失败的判定、任务计分板更新、游戏进度展示和游戏结束判断。

## 详细需求

### 功能需求

1. 任务结果统计：

   - 根据任务执行结果，统计成功和失败票数
   - 根据任务规则判定任务成功或失败（通常一张失败票即导致任务失败，第四轮特殊规则除外）
   - 记录每次任务的详细结果，包括队长、队员和投票情况
   - 更新游戏总体进度（好人/坏人阵营的任务完成情况）

2. 任务计分板：

   - 直观展示已完成任务的成功/失败状态
   - 显示当前任务进度（第几轮任务）
   - 显示好人/坏人阵营的任务完成情况（例如：好人 2 次，坏人 1 次）
   - 提供任务历史记录查看功能

3. 游戏结束判断：
   - 检测 3 次任务成功（好人阵营暂时获胜，进入刺杀阶段）
   - 检测 3 次任务失败（坏人阵营直接获胜）
   - 检测 5 次连续拒绝队伍（坏人阵营直接获胜）
   - 在适当时机触发游戏结束或进入下一阶段

### 界面要求

1. 任务结果展示：

   - 任务结果公布时有明显的视觉效果（成功/失败不同效果）
   - 显示任务成功/失败的票数统计（不显示具体是谁投的）
   - 提供适当的动画和音效增强体验
   - 结果展示后有明确的继续游戏按钮

2. 任务计分板设计：

   - 在游戏界面中始终可见
   - 使用明确的视觉标识区分成功/失败任务
   - 当前任务有特殊标记
   - 设计美观，符合游戏风格

3. 游戏进度提示：
   - 在关键节点提供游戏进度提示（例如："好人阵营已完成 2 次任务，还需 1 次即可进入刺杀阶段"）
   - 当游戏接近结束时提供紧张感提示
   - 游戏结束时有明确的结果展示

### 技术要求

1. 实现任务结果计算和统计逻辑
2. 实现任务计分板更新机制
3. 实现游戏结束判断逻辑
4. 确保结果展示的动画流畅
5. 支持断线重连后恢复当前游戏进度

## 实现步骤

1. 设计任务结果数据结构
2. 实现任务结果计算逻辑
3. 实现任务计分板界面
4. 实现任务结果展示界面
5. 实现游戏结束判断逻辑
6. 实现任务历史记录功能
7. 进行测试和优化

## 代码示例

### 任务结果计算

```typescript
// 计算任务结果
function calculateMissionResult(game: Game): MissionResult {
  const mission = game.currentMission;
  const executionResults = mission.executionResults;

  // 统计成功和失败票数
  let successCount = 0;
  let failCount = 0;

  executionResults.forEach((result) => {
    if (result === MissionResultType.SUCCESS) {
      successCount++;
    } else {
      failCount++;
    }
  });

  // 获取任务配置
  const missionConfig = getMissionConfig(
    mission.missionNumber,
    game.players.length
  );

  // 判断任务结果
  const isSuccess = failCount < missionConfig.requiredFailures;

  // 创建任务结果对象
  const result: MissionResult = {
    missionNumber: mission.missionNumber,
    teamMemberIds: mission.currentTeam.memberIds,
    captainId: mission.currentTeam.captainId,
    result: isSuccess ? MissionResultType.SUCCESS : MissionResultType.FAIL,
    successCount: successCount,
    failCount: failCount,
  };

  return result;
}
```

### 游戏结束判断

```typescript
// 检查游戏是否结束
function checkGameEnd(game: Game): GameEndResult | null {
  // 统计成功和失败的任务数
  let successCount = 0;
  let failCount = 0;

  game.missionResults.forEach((result) => {
    if (result.result === MissionResultType.SUCCESS) {
      successCount++;
    } else {
      failCount++;
    }
  });

  // 检查连续拒绝次数
  if (game.consecutiveRejections >= 5) {
    return {
      isEnded: true,
      winner: Faction.EVIL,
      reason: "连续5次拒绝队伍",
      nextPhase: GamePhase.ENDED,
    };
  }

  // 检查任务成功/失败次数
  if (successCount >= 3) {
    return {
      isEnded: false, // 游戏还没有真正结束
      winner: null,
      reason: "好人阵营完成3次任务",
      nextPhase: GamePhase.ASSASSINATION, // 进入刺杀阶段
    };
  }

  if (failCount >= 3) {
    return {
      isEnded: true,
      winner: Faction.EVIL,
      reason: "坏人阵营破坏3次任务",
      nextPhase: GamePhase.ENDED,
    };
  }

  // 游戏继续
  return null;
}
```

### 任务计分板组件

```typescript
class MissionScoreboard extends PIXI.Container {
  private game: Game;
  private missionIndicators: PIXI.Container[] = [];

  constructor(game: Game) {
    super();
    this.game = game;

    // 初始化计分板
    this.initScoreboard();

    // 监听任务结果更新
    game.on("missionCompleted", this.updateScoreboard.bind(this));
  }

  private initScoreboard(): void {
    // 创建标题
    const title = new PIXI.Text("任务计分板", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    title.x = 10;
    title.y = 5;
    this.addChild(title);

    // 创建任务指示器
    for (let i = 0; i < 5; i++) {
      const indicator = this.createMissionIndicator(i + 1);
      indicator.x = 20 + i * 60;
      indicator.y = 30;
      this.addChild(indicator);
      this.missionIndicators.push(indicator);
    }

    // 创建任务进度提示
    const progressText = new PIXI.Text("", {
      fontFamily: "Arial",
      fontSize: 14,
      fill: 0xffffff,
    });
    progressText.x = 10;
    progressText.y = 80;
    this.addChild(progressText);

    // 初始更新
    this.updateScoreboard();
  }

  private createMissionIndicator(missionNumber: number): PIXI.Container {
    const container = new PIXI.Container();

    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(0x333333);
    background.drawCircle(25, 25, 25);
    background.endFill();
    container.addChild(background);

    // 创建任务编号文本
    const numberText = new PIXI.Text(missionNumber.toString(), {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    numberText.x = 25;
    numberText.y = 25;
    numberText.anchor.set(0.5);
    container.addChild(numberText);

    // 添加特殊规则标记（如果是第4轮任务且玩家数>=7）
    if (missionNumber === 4 && this.game.players.length >= 7) {
      const specialMark = new PIXI.Text("*", {
        fontFamily: "Arial",
        fontSize: 20,
        fill: 0xff0000,
      });
      specialMark.x = 40;
      specialMark.y = 10;
      container.addChild(specialMark);
    }

    return container;
  }

  public updateScoreboard(): void {
    // 更新任务指示器状态
    this.game.missionResults.forEach((result) => {
      const indicator = this.missionIndicators[result.missionNumber - 1];

      // 清除之前的状态
      if (indicator.children.length > 2) {
        indicator.removeChildAt(2);
      }

      // 添加结果标记
      const resultMark = new PIXI.Graphics();
      if (result.result === MissionResultType.SUCCESS) {
        // 成功标记（蓝色对勾）
        resultMark.beginFill(0x3498db);
        resultMark.drawCircle(25, 25, 25);
        resultMark.endFill();
      } else {
        // 失败标记（红色叉号）
        resultMark.beginFill(0xe74c3c);
        resultMark.drawCircle(25, 25, 25);
        resultMark.endFill();
      }

      // 插入到背景和文本之间
      indicator.addChildAt(resultMark, 1);
    });

    // 高亮当前任务
    this.missionIndicators.forEach((indicator, index) => {
      if (index + 1 === this.game.currentMissionNumber) {
        indicator.scale.set(1.1);
        indicator.alpha = 1.0;
      } else if (index + 1 > this.game.missionResults.length) {
        indicator.scale.set(1.0);
        indicator.alpha = 0.7;
      } else {
        indicator.scale.set(1.0);
        indicator.alpha = 1.0;
      }
    });

    // 更新进度提示
    const successCount = this.game.missionResults.filter(
      (r) => r.result === MissionResultType.SUCCESS
    ).length;
    const failCount = this.game.missionResults.filter(
      (r) => r.result === MissionResultType.FAIL
    ).length;

    const progressText = this.getChildAt(this.children.length - 1) as PIXI.Text;
    progressText.text = `好人: ${successCount}/3  坏人: ${failCount}/3`;
  }
}
```

### 任务结果展示界面

```typescript
class MissionResultView extends PIXI.Container {
  private game: Game;
  private missionResult: MissionResult;

  constructor(game: Game, missionResult: MissionResult) {
    super();
    this.game = game;
    this.missionResult = missionResult;

    // 初始化界面
    this.initView();
  }

  private initView(): void {
    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(0x000000, 0.8);
    background.drawRect(0, 0, 500, 400);
    background.endFill();
    this.addChild(background);

    // 创建标题
    const titleText = `任务${this.missionResult.missionNumber}结果`;
    const title = new PIXI.Text(titleText, {
      fontFamily: "Arial",
      fontSize: 28,
      fill: 0xffffff,
      fontWeight: "bold",
    });
    title.x = 250;
    title.y = 30;
    title.anchor.set(0.5, 0);
    this.addChild(title);

    // 创建结果图标和文本
    this.createResultDisplay();

    // 创建票数统计
    this.createVoteStatistics();

    // 创建队员信息
    this.createTeamInfo();

    // 创建进度提示
    this.createProgressInfo();

    // 创建继续按钮
    this.createContinueButton();
  }

  private createResultDisplay(): void {
    // 创建结果容器
    const container = new PIXI.Container();
    container.x = 250;
    container.y = 100;
    this.addChild(container);

    // 创建结果图标
    const iconTexture =
      this.missionResult.result === MissionResultType.SUCCESS
        ? "success_icon"
        : "fail_icon";
    const icon = new PIXI.Sprite(PIXI.Texture.from(iconTexture));
    icon.width = 80;
    icon.height = 80;
    icon.anchor.set(0.5);
    container.addChild(icon);

    // 创建结果文本
    const resultText =
      this.missionResult.result === MissionResultType.SUCCESS
        ? "任务成功"
        : "任务失败";
    const textColor =
      this.missionResult.result === MissionResultType.SUCCESS
        ? 0x3498db
        : 0xe74c3c;
    const text = new PIXI.Text(resultText, {
      fontFamily: "Arial",
      fontSize: 32,
      fill: textColor,
      fontWeight: "bold",
    });
    text.x = 0;
    text.y = 100;
    text.anchor.set(0.5, 0);
    container.addChild(text);

    // 添加动画效果
    gsap.from(icon.scale, {
      x: 0,
      y: 0,
      duration: 0.5,
      ease: "back.out(1.7)",
    });
    gsap.from(text, {
      alpha: 0,
      y: text.y + 20,
      duration: 0.5,
      delay: 0.3,
    });
  }

  private createVoteStatistics(): void {
    // 创建票数统计容器
    const container = new PIXI.Container();
    container.x = 250;
    container.y = 200;
    this.addChild(container);

    // 创建成功票数文本
    const successText = new PIXI.Text(
      `成功票数: ${this.missionResult.successCount}`,
      {
        fontFamily: "Arial",
        fontSize: 18,
        fill: 0x3498db,
      }
    );
    successText.anchor.set(0.5, 0);
    container.addChild(successText);

    // 创建失败票数文本
    const failText = new PIXI.Text(
      `失败票数: ${this.missionResult.failCount}`,
      {
        fontFamily: "Arial",
        fontSize: 18,
        fill: 0xe74c3c,
      }
    );
    failText.y = 30;
    failText.anchor.set(0.5, 0);
    container.addChild(failText);

    // 添加动画效果
    gsap.from(container, {
      alpha: 0,
      duration: 0.5,
      delay: 0.6,
    });
  }

  private createTeamInfo(): void {
    // 创建队员信息容器
    const container = new PIXI.Container();
    container.x = 50;
    container.y = 250;
    this.addChild(container);

    // 创建标题
    const title = new PIXI.Text("执行任务的队员:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    container.addChild(title);

    // 获取队员名称
    const teamMembers = this.missionResult.teamMemberIds.map((id) => {
      const player = this.game.players.find((p) => p.id === id);
      return player ? player.name : "未知玩家";
    });

    // 创建队员列表
    const membersList = new PIXI.Text(teamMembers.join(", "), {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xcccccc,
    });
    membersList.y = 25;
    container.addChild(membersList);

    // 创建队长信息
    const captain = this.game.players.find(
      (p) => p.id === this.missionResult.captainId
    );
    const captainText = new PIXI.Text(
      `队长: ${captain ? captain.name : "未知队长"}`,
      {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xf1c40f,
      }
    );
    captainText.y = 50;
    container.addChild(captainText);

    // 添加动画效果
    gsap.from(container, {
      alpha: 0,
      x: container.x - 20,
      duration: 0.5,
      delay: 0.8,
    });
  }

  private createProgressInfo(): void {
    // 统计成功和失败的任务数
    const successCount = this.game.missionResults.filter(
      (r) => r.result === MissionResultType.SUCCESS
    ).length;
    const failCount = this.game.missionResults.filter(
      (r) => r.result === MissionResultType.FAIL
    ).length;

    // 创建进度提示容器
    const container = new PIXI.Container();
    container.x = 250;
    container.y = 320;
    this.addChild(container);

    // 创建进度文本
    const progressText = new PIXI.Text(
      `好人阵营已完成${successCount}个任务，坏人阵营已完成${failCount}个任务`,
      {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xffffff,
        align: "center",
      }
    );
    progressText.anchor.set(0.5, 0);
    container.addChild(progressText);

    // 添加动画效果
    gsap.from(container, {
      alpha: 0,
      duration: 0.5,
      delay: 1.0,
    });
  }

  private createContinueButton(): void {
    // 创建继续按钮
    const button = new PIXI.Container();
    button.x = 250;
    button.y = 360;
    this.addChild(button);

    // 创建按钮背景
    const background = new PIXI.Graphics();
    background.beginFill(0x3498db);
    background.drawRoundedRect(-75, 0, 150, 40, 10);
    background.endFill();
    button.addChild(background);

    // 创建按钮文本
    const text = new PIXI.Text("继续", {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    text.anchor.set(0.5);
    text.x = 0;
    text.y = 20;
    button.addChild(text);

    // 添加交互
    button.interactive = true;
    button.buttonMode = true;
    button.on("pointerdown", () => {
      this.emit("continue");
    });

    // 添加动画效果
    gsap.from(button, {
      alpha: 0,
      y: button.y + 20,
      duration: 0.5,
      delay: 1.2,
    });
  }
}
```

## 界面原型

```
+------------------------------------------+
|             任务计分板                    |
+------------------------------------------+
|                                          |
|  [成功] [失败] [成功] [ * ] [ ]          |
|   #1     #2     #3     #4    #5          |
|                                          |
|  好人: 2/3       坏人: 1/3               |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           任务结果 - 任务成功            |
+------------------------------------------+
|                                          |
|  任务3                                   |
|                                          |
|           [成功图标]                     |
|                                          |
|           任务成功                       |
|                                          |
|  成功票数: 4                             |
|  失败票数: 0                             |
|                                          |
|  执行任务的队员:                         |
|  玩家1, 玩家3, 玩家5, 玩家8              |
|  队长: 玩家3                             |
|                                          |
|  好人阵营已完成2个任务，坏人阵营已完成1个任务 |
|                                          |
|              [继续]                      |
+------------------------------------------+
```

```
+------------------------------------------+
|           任务结果 - 任务失败            |
+------------------------------------------+
|                                          |
|  任务2                                   |
|                                          |
|           [失败图标]                     |
|                                          |
|           任务失败                       |
|                                          |
|  成功票数: 2                             |
|  失败票数: 1                             |
|                                          |
|  执行任务的队员:                         |
|  玩家1, 玩家3, 玩家6                     |
|  队长: 玩家1                             |
|                                          |
|  好人阵营已完成1个任务，坏人阵营已完成1个任务 |
|                                          |
|              [继续]                      |
+------------------------------------------+
```

```
+------------------------------------------+
|           游戏进度提示                   |
+------------------------------------------+
|                                          |
|  好人阵营已完成2次任务                    |
|  再完成1次任务即可进入刺杀阶段            |
|                                          |
|  坏人阵营已完成1次任务                    |
|  再完成2次任务即可获胜                    |
|                                          |
|  当前是第4轮任务                          |
|  需要2张失败票才会导致任务失败            |
|                                          |
|              [了解]                      |
+------------------------------------------+
```

```
+------------------------------------------+
|           游戏结束 - 好人获胜            |
+------------------------------------------+
|                                          |
|           [胜利图标]                     |
|                                          |
|        好人阵营暂时获胜!                 |
|                                          |
|  好人阵营成功完成了3次任务                |
|  但游戏尚未结束...                       |
|                                          |
|  刺客将有一次机会刺杀梅林                 |
|  如果成功，坏人阵营将最终获胜             |
|                                          |
|        [进入刺杀阶段]                    |
+------------------------------------------+
```

```
+------------------------------------------+
|           游戏结束 - 坏人获胜            |
+------------------------------------------+
|                                          |
|           [失败图标]                     |
|                                          |
|        坏人阵营获胜!                     |
|                                          |
|  坏人阵营成功破坏了3次任务                |
|  邪恶势力统治了卡美洛特                   |
|                                          |
|  任务结果:                               |
|  [成功] [失败] [失败] [失败] [ ]         |
|                                          |
|        [查看角色]      [返回大厅]        |
+------------------------------------------+
```

## 验收标准

1. 任务结果计算准确，正确判定任务成功或失败
2. 任务计分板清晰展示任务状态和游戏进度
3. 任务结果展示界面美观，提供足够的信息
4. 游戏结束判断逻辑正确，能够识别所有结束条件
5. 任务历史记录功能正常工作
6. 断线重连后能够恢复当前游戏进度
7. 界面动画流畅，增强游戏体验

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库
- 游戏状态管理系统
- 网络同步模块
- 用户界面组件库

## 工作量估计

1 人天

## 相关文档

- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
- [任务系统技术方案](./技术方案.md)
