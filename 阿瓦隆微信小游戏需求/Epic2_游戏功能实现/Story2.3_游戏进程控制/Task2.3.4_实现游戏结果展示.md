# Task 2.3.4: 实现游戏结果展示

## 描述

实现阿瓦隆游戏的结果展示功能，包括游戏胜负判定、角色揭示、游戏统计数据展示和结束处理，为玩家提供清晰直观的游戏结果呈现。

## 详细需求

### 功能需求

1. 游戏结果判定：

   - 判断游戏胜负情况（好人阵营或坏人阵营获胜）
   - 确定获胜原因（完成 3 次任务、5 次连续拒绝队伍、刺杀成功等）
   - 收集游戏过程的关键数据（任务成功/失败次数、刺杀结果等）
   - 生成游戏结果摘要

2. 角色揭示：

   - 展示所有玩家的真实角色和阵营
   - 突出显示关键角色（如梅林、刺客、莫德雷德等）
   - 显示每个玩家在游戏中的关键行为（如投票情况、任务执行情况）
   - 提供角色之间的关系展示（如谁能看到谁）

3. 游戏统计：
   - 统计游戏时长、回合数
   - 统计每位玩家的队长次数、出任务次数
   - 统计任务成功/失败数据、投票数据
   - 提供游戏回放或游戏历史保存功能（可选）

### 界面要求

1. 结果主界面：

   - 清晰展示胜利阵营和胜利原因
   - 使用醒目的视觉效果区分胜利和失败
   - 提供游戏结果的简要说明
   - 包含继续按钮以查看详细信息

2. 角色揭示界面：

   - 以直观的方式展示所有玩家的角色信息
   - 使用不同颜色区分好人阵营和坏人阵营
   - 提供角色介绍和能力说明
   - 可按阵营分组查看

3. 统计数据界面：
   - 使用图表或列表展示游戏统计数据
   - 提供筛选和排序功能
   - 支持数据导出或分享功能（可选）
   - 包含返回大厅或再来一局的选项

### 技术要求

1. 实现结果数据的收集和处理
2. 确保结果显示的准确性和完整性
3. 优化界面动画和过渡效果
4. 支持统计数据的存储和加载
5. 确保界面在不同设备上的适配性

## 实现步骤

1. 设计游戏结果数据结构
2. 实现结果判定逻辑
3. 开发结果主界面
4. 实现角色揭示界面
5. 开发统计数据界面
6. 实现界面之间的导航
7. 添加动画和音效
8. 进行测试和优化

## 代码示例

### 游戏结果管理器

```typescript
// 简化的游戏结果管理器
class GameResultManager {
  private gameStateManager: GameStateManager;

  constructor(gameStateManager: GameStateManager) {
    this.gameStateManager = gameStateManager;
  }

  // 生成游戏结果
  public generateGameResult(): GameResult {
    const gameState = this.gameStateManager.getGameState();

    // 判断游戏结果
    let winner: Faction;
    let reason: string;

    // 检查刺杀结果
    if (gameState.status === "result" && gameState.assassinationTarget) {
      const targetRole = gameState.playerRoles.get(
        gameState.assassinationTarget
      );
      if (targetRole && targetRole.roleType === "merlin") {
        winner = "evil";
        reason = "刺客成功刺杀梅林";
      } else {
        winner = "good";
        reason = "刺客刺杀失败，好人阵营获胜";
      }
    }
    // 检查任务结果
    else {
      const successCount = gameState.missionResults.filter(
        (r) => r.result === "SUCCESS"
      ).length;
      const failCount = gameState.missionResults.filter(
        (r) => r.result === "FAIL"
      ).length;

      if (successCount >= 3) {
        winner = "good";
        reason = "好人阵营成功完成3次任务";
      } else if (failCount >= 3) {
        winner = "evil";
        reason = "坏人阵营成功破坏3次任务";
      } else if (gameState.consecutiveRejections >= 5) {
        winner = "evil";
        reason = "连续5次拒绝队伍，坏人阵营获胜";
      } else {
        winner = "unknown";
        reason = "游戏未正常结束";
      }
    }

    // 创建游戏结果对象
    return {
      winner,
      reason,
      missionSuccessCount: gameState.missionResults.filter(
        (r) => r.result === "SUCCESS"
      ).length,
      missionFailCount: gameState.missionResults.filter(
        (r) => r.result === "FAIL"
      ).length,
      assassinationTarget: gameState.assassinationTarget,
      playerRoles: gameState.playerRoles,
      gameDuration: this.calculateGameDuration(gameState),
      timestamp: new Date(),
    };
  }

  // 生成玩家统计数据
  public generatePlayerStats(): PlayerStats[] {
    const gameState = this.gameStateManager.getGameState();
    const stats: PlayerStats[] = [];

    // 为每个玩家生成统计数据
    gameState.players.forEach((player) => {
      const role = gameState.playerRoles.get(player.id);
      if (!role) return;

      // 计算该玩家担任队长的次数
      const captainCount = gameState.captainHistory.filter(
        (c) => c.playerId === player.id
      ).length;

      // 计算该玩家参与任务的次数
      const missionParticipation = gameState.missionResults.reduce(
        (count, mission) => {
          return count + (mission.teamMemberIds.includes(player.id) ? 1 : 0);
        },
        0
      );

      // 计算投票统计
      const voteStats = this.calculateVoteStats(player.id);

      // 创建玩家统计对象
      stats.push({
        playerId: player.id,
        playerName: player.name,
        role: role.roleType,
        faction: this.getRoleFaction(role.roleType),
        captainCount,
        missionParticipation,
        voteStats,
        isAssassin: role.roleType === "assassin",
        isMerlin: role.roleType === "merlin",
        wasAssassinated: player.id === gameState.assassinationTarget,
      });
    });

    return stats;
  }

  // 计算玩家投票统计
  private calculateVoteStats(playerId: string): VoteStats {
    const gameState = this.gameStateManager.getGameState();
    let approveCount = 0;
    let rejectCount = 0;

    // 遍历所有任务
    gameState.missions.forEach((mission) => {
      // 遍历所有被拒绝的队伍
      mission.rejectedTeams.forEach((team) => {
        const vote = team.votes.get(playerId);
        if (vote === "approve") approveCount++;
        else if (vote === "reject") rejectCount++;
      });

      // 检查当前队伍的投票
      if (mission.currentTeam && mission.currentTeam.votes) {
        const vote = mission.currentTeam.votes.get(playerId);
        if (vote === "approve") approveCount++;
        else if (vote === "reject") rejectCount++;
      }
    });

    return {
      approveCount,
      rejectCount,
      totalVotes: approveCount + rejectCount,
    };
  }

  // 获取角色所属阵营
  private getRoleFaction(roleType: string): Faction {
    const goodRoles = ["merlin", "percival", "loyal"];
    return goodRoles.includes(roleType) ? "good" : "evil";
  }

  // 计算游戏时长（秒）
  private calculateGameDuration(gameState: GameState): number {
    const startTime = gameState.createdAt.getTime();
    const endTime = gameState.updatedAt.getTime();
    return Math.floor((endTime - startTime) / 1000);
  }
}
```

### 结果展示界面

```typescript
// 简化的结果展示界面
class GameResultView {
  private app: PIXI.Application;
  private gameResult: GameResult;
  private playerStats: PlayerStats[];
  private container: PIXI.Container;

  constructor(
    app: PIXI.Application,
    gameResult: GameResult,
    playerStats: PlayerStats[]
  ) {
    this.app = app;
    this.gameResult = gameResult;
    this.playerStats = playerStats;
    this.container = new PIXI.Container();

    // 初始化结果界面
    this.initializeResultView();
  }

  // 初始化结果界面
  private initializeResultView(): void {
    // 创建背景
    const background = new PIXI.Graphics();
    background.beginFill(
      this.gameResult.winner === "good" ? 0x3498db : 0xe74c3c,
      0.9
    );
    background.drawRect(0, 0, 800, 600);
    background.endFill();
    this.container.addChild(background);

    // 创建标题
    const titleText =
      this.gameResult.winner === "good" ? "好人阵营获胜!" : "坏人阵营获胜!";
    const title = new PIXI.Text(titleText, {
      fontFamily: "Arial",
      fontSize: 48,
      fill: 0xffffff,
      fontWeight: "bold",
    });
    title.x = 400;
    title.y = 50;
    title.anchor.set(0.5);
    this.container.addChild(title);

    // 创建原因文本
    const reason = new PIXI.Text(this.gameResult.reason, {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
    });
    reason.x = 400;
    reason.y = 120;
    reason.anchor.set(0.5);
    this.container.addChild(reason);

    // 创建任务统计
    this.createMissionStats();

    // 创建角色列表
    this.createRolesList();

    // 创建导航按钮
    this.createNavigationButtons();
  }

  // 创建任务统计
  private createMissionStats(): void {
    const statsContainer = new PIXI.Container();
    statsContainer.x = 400;
    statsContainer.y = 180;
    this.container.addChild(statsContainer);

    // 任务统计标题
    const title = new PIXI.Text("任务统计", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
      fontWeight: "bold",
    });
    title.anchor.set(0.5, 0);
    statsContainer.addChild(title);

    // 任务成功统计
    const successText = new PIXI.Text(
      `任务成功: ${this.gameResult.missionSuccessCount}`,
      {
        fontFamily: "Arial",
        fontSize: 18,
        fill: 0xffffff,
      }
    );
    successText.x = 0;
    successText.y = 30;
    successText.anchor.set(0.5, 0);
    statsContainer.addChild(successText);

    // 任务失败统计
    const failText = new PIXI.Text(
      `任务失败: ${this.gameResult.missionFailCount}`,
      {
        fontFamily: "Arial",
        fontSize: 18,
        fill: 0xffffff,
      }
    );
    failText.x = 0;
    failText.y = 60;
    failText.anchor.set(0.5, 0);
    statsContainer.addChild(failText);
  }

  // 创建角色列表
  private createRolesList(): void {
    const rolesContainer = new PIXI.Container();
    rolesContainer.x = 400;
    rolesContainer.y = 280;
    this.container.addChild(rolesContainer);

    // 角色列表标题
    const title = new PIXI.Text("所有角色", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
      fontWeight: "bold",
    });
    title.anchor.set(0.5, 0);
    rolesContainer.addChild(title);

    // 创建角色列表
    const rolesList = new PIXI.Container();
    rolesList.y = 40;
    rolesContainer.addChild(rolesList);

    // 按阵营分组
    const goodPlayers = this.playerStats.filter((p) => p.faction === "good");
    const evilPlayers = this.playerStats.filter((p) => p.faction === "evil");

    // 展示好人阵营
    let yOffset = 0;
    goodPlayers.forEach((player, index) => {
      const playerRow = this.createPlayerRoleRow(player, 0x3498db);
      playerRow.y = yOffset;
      rolesList.addChild(playerRow);
      yOffset += 30;
    });

    // 添加分隔
    yOffset += 10;

    // 展示坏人阵营
    evilPlayers.forEach((player, index) => {
      const playerRow = this.createPlayerRoleRow(player, 0xe74c3c);
      playerRow.y = yOffset;
      rolesList.addChild(playerRow);
      yOffset += 30;
    });
  }

  // 创建单个玩家角色行
  private createPlayerRoleRow(
    player: PlayerStats,
    color: number
  ): PIXI.Container {
    const container = new PIXI.Container();

    // 背景
    if (player.wasAssassinated || player.isAssassin) {
      const bg = new PIXI.Graphics();
      bg.beginFill(0xffff00, 0.3);
      bg.drawRect(-200, -2, 400, 24);
      bg.endFill();
      container.addChild(bg);
    }

    // 玩家名称
    const nameText = new PIXI.Text(player.playerName, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    nameText.x = -150;
    container.addChild(nameText);

    // 角色名称
    const roleText = new PIXI.Text(this.getRoleDisplayName(player.role), {
      fontFamily: "Arial",
      fontSize: 16,
      fill: color,
    });
    roleText.x = 50;
    container.addChild(roleText);

    // 特殊标记
    if (player.wasAssassinated) {
      const marker = new PIXI.Text("(被刺杀)", {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xff0000,
      });
      marker.x = 150;
      container.addChild(marker);
    } else if (player.isAssassin) {
      const marker = new PIXI.Text("(刺客)", {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xff0000,
      });
      marker.x = 150;
      container.addChild(marker);
    }

    return container;
  }

  // 获取角色显示名称
  private getRoleDisplayName(roleType: string): string {
    const roleNames = {
      merlin: "梅林",
      percival: "派西维尔",
      loyal: "忠臣",
      assassin: "刺客",
      morgana: "莫甘娜",
      mordred: "莫德雷德",
      oberon: "奥伯伦",
      minion: "爪牙",
    };

    return roleNames[roleType] || roleType;
  }

  // 创建导航按钮
  private createNavigationButtons(): void {
    const buttonsContainer = new PIXI.Container();
    buttonsContainer.x = 400;
    buttonsContainer.y = 520;
    this.container.addChild(buttonsContainer);

    // 查看详细统计按钮
    const statsButton = this.createButton("查看详细统计", 0x3498db);
    statsButton.x = -150;
    buttonsContainer.addChild(statsButton);

    // 添加点击事件
    statsButton.interactive = true;
    statsButton.buttonMode = true;
    statsButton.on("pointerdown", () => {
      this.showStatsView();
    });

    // 返回大厅按钮
    const lobbyButton = this.createButton("返回大厅", 0x2c3e50);
    lobbyButton.x = 150;
    buttonsContainer.addChild(lobbyButton);

    // 添加点击事件
    lobbyButton.interactive = true;
    lobbyButton.buttonMode = true;
    lobbyButton.on("pointerdown", () => {
      this.returnToLobby();
    });
  }

  // 创建按钮
  private createButton(text: string, color: number): PIXI.Container {
    const button = new PIXI.Container();

    // 按钮背景
    const bg = new PIXI.Graphics();
    bg.beginFill(color);
    bg.drawRoundedRect(-100, -25, 200, 50, 10);
    bg.endFill();
    button.addChild(bg);

    // 按钮文本
    const buttonText = new PIXI.Text(text, {
      fontFamily: "Arial",
      fontSize: 18,
      fill: 0xffffff,
    });
    buttonText.anchor.set(0.5);
    button.addChild(buttonText);

    return button;
  }

  // 显示详细统计视图
  private showStatsView(): void {
    // 这里会创建并显示详细统计视图
    // 由于代码简化，这部分实现略过
    console.log("显示详细统计");
  }

  // 返回大厅
  private returnToLobby(): void {
    // 返回大厅的逻辑
    console.log("返回大厅");
  }

  // 显示结果界面
  public show(): void {
    this.app.stage.addChild(this.container);
  }

  // 隐藏结果界面
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
|           游戏结果 - 好人获胜             |
+------------------------------------------+
|                                          |
|           好人阵营获胜!                   |
|                                          |
|      刺客刺杀失败，好人阵营获胜           |
|                                          |
|              任务统计                     |
|             任务成功: 3                   |
|             任务失败: 2                   |
|                                          |
|              所有角色                     |
|                                          |
|  玩家1    梅林                            |
|  玩家2    派西维尔                        |
|  玩家3    忠臣                            |
|  玩家4    忠臣      (被刺杀)              |
|                                          |
|  玩家5    刺客      (刺客)                |
|  玩家6    莫甘娜                          |
|  玩家7    莫德雷德                        |
|                                          |
|  [查看详细统计]         [返回大厅]        |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           游戏结果 - 坏人获胜             |
+------------------------------------------+
|                                          |
|           坏人阵营获胜!                   |
|                                          |
|         刺客成功刺杀梅林                  |
|                                          |
|              任务统计                     |
|             任务成功: 3                   |
|             任务失败: 1                   |
|                                          |
|              所有角色                     |
|                                          |
|  玩家1    梅林      (被刺杀)              |
|  玩家2    派西维尔                        |
|  玩家3    忠臣                            |
|  玩家4    忠臣                            |
|                                          |
|  玩家5    刺客      (刺客)                |
|  玩家6    莫甘娜                          |
|  玩家7    莫德雷德                        |
|                                          |
|  [查看详细统计]         [返回大厅]        |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           详细游戏统计                    |
+------------------------------------------+
|                                          |
|  游戏时长: 35分钟                         |
|  任务总数: 5                              |
|  投票总数: 12                             |
|                                          |
|  玩家数据:                               |
|                                          |
|  玩家名称  角色    队长次数  任务次数  投票情况 |
|  玩家1    梅林       1        2      5赞成/2反对 |
|  玩家2    派西维尔   0        2      4赞成/3反对 |
|  玩家3    忠臣       1        3      6赞成/1反对 |
|  玩家4    忠臣       1        1      3赞成/4反对 |
|  玩家5    刺客       1        1      2赞成/5反对 |
|  玩家6    莫甘娜     0        1      1赞成/6反对 |
|  玩家7    莫德雷德   1        2      3赞成/4反对 |
|                                          |
|                [返回]                    |
|                                          |
+------------------------------------------+
```

## 验收标准

1. 游戏结果正确判定，显示正确的胜利阵营和原因
2. 所有玩家角色信息完整展示，并明确区分好人与坏人阵营
3. 突出显示关键角色（如梅林、被刺杀的玩家、刺客等）
4. 游戏统计数据准确完整
5. 界面美观，使用合适的颜色和动画效果
6. 提供清晰的导航选项（查看详细统计、返回大厅等）
7. 结果界面能适应不同屏幕尺寸
8. 所有按钮和交互元素响应正常

## 技术依赖

- TypeScript/JavaScript
- PIXI.js 渲染引擎
- 游戏状态管理系统
- 事件发布-订阅系统
- 数据可视化库（可选，用于统计图表）

## 工作量估计

1 人天

## 相关文档

- [游戏进程控制技术方案](./技术方案.md)
- [刺杀阶段实现](./Task2.3.3_实现刺杀阶段.md)
- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
